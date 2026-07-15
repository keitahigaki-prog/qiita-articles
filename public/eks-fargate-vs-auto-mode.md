---
title: 'EKS Fargate vs EKS Auto Mode ― ランタイムセキュリティ(Sysdig)の観点で選ぶ【実機検証あり】'
tags:
  - AWS
  - EKS
  - Fargate
  - Kubernetes
  - Sysdig
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: true
---

> ⚠️ 用語注意：EKS に「Autopilot」という機能名はありません。「Autopilot」は GKE の機能です。AWS 側の同種フルマネージド機能は **EKS Auto Mode**（2024年末 GA）。本記事は EKS Fargate と EKS Auto Mode を比較します。

機能比較の記事は世にたくさんあります。この記事は **「セキュリティエージェント（Sysdig）をどう入れて、どう運用し、いくらかかるか」** の3軸で両者を比べます。

そして最初に結論の核を言います：

> **EKS でランタイムセキュリティ（Sysdig）を効かせたいなら、Fargate は実質選べません。**
> Sysdig 公式は **「AWS Fargate is not supported on EKS」** と明記しています（後述）。ノードが要る＝ **Auto Mode（or マネージドノードグループ）一択**です。

これは机上の話ではなく、**アーキテクチャの制約が製品サポート境界に直結している**という、選定で最初に踏むべき分岐です。

## 1. まず結論（比較表）

| 観点 | EKS Fargate | EKS Auto Mode |
|---|---|---|
| 計算単位 | Pod = 専用 micro-VM（サーバレス） | マネージド EC2（Bottlerocket）|
| ノードの実体 | 見えない | EC2 として存在（AWS 所有・SSH 不可）|
| スケーリング | Fargate Profile 単位、Pod ごと | Karpenter ベースで自動 |
| **DaemonSet** | ❌ 非対応 | ✅ 対応 |
| **特権 Pod / hostPath / hostPID** | ❌ 不可 | ⚠️ 制限あり（Bottlerocket, read-only rootfs, SELinux）|
| GPU | ❌ 非対応 | ✅ 対応 |
| EBS | ❌（EFS のみ）| ✅ EBS/EFS |
| OS パッチ管理 | 不要 | 不要（AWS が Bottlerocket 自動更新）|
| 料金モデル | vCPU秒 + GB秒（実行時間課金）| $0.10/h クラスタ + EC2 + Auto Mode 手数料(~12%) |
| **Sysdig ランタイム監視** | ❌ **公式サポート外（EKS Fargate）** | ✅ DaemonSet + universal eBPF |

**一言でいうと**：Fargate は「ノードを消す」、Auto Mode は「ノードは残して AWS が運用する」。ランタイムセキュリティはノード（カーネル）アクセスを前提とするため、この差が製品サポートの可否に直結します。

## 2. セキュリティの観点：なぜ EKS Fargate だと Sysdig が入らないのか

### EKS Fargate は DaemonSet も特権も使えない → サポート外

Fargate は **DaemonSet も特権コンテナも hostPath も使えません**。「ノードに1つ常駐してカーネルを観測するエージェント」という常道が構造的に封じられます。結果として **Sysdig は EKS Fargate を公式サポートしていません**（Installation Requirements に "AWS Fargate is not supported on EKS" と明記）。

> よくある誤解：「Fargate に ptrace サイドカーを注入すれば Sysdig が動く」——これは **ECS Fargate（Amazon ECS）** の話です。**EKS Fargate ではサポートされていません**。混同注意。

### ECS Fargate なら別製品（Serverless Agent）で対応できる（＝EKS の話ではない）

参考として、**ECS Fargate** では Sysdig の **Serverless Agent** が使えます。`CAP_SYS_PTRACE` を使い、タスク定義に `quay.io/sysdig/workload-agent` サイドカーを注入して syscall を捕捉する方式（`pid_mode: "task"` でサイドカー、v5.0.0 以降デフォルト）。ただしこれは ECS の話で、**本記事のテーマ（EKS）とは別レイヤー**です。

### Auto Mode — DaemonSet は動くが Bottlerocket でロックダウン

Auto Mode はマネージド EC2（Bottlerocket）なので DaemonSet が動き、Sysdig の Shield/Agent がデプロイできます。ただし Bottlerocket は read-only rootfs・SELinux enforced・**カーネルモジュールのロード不可**。

→ **`universal_ebpf` ドライバが必須**。従来のカーネルモジュール前提デプロイはそのままでは動きません。agent 12.17.0 以降は universal eBPF を同梱しており `kmodule` コンテナは不要です。

> **合言葉**：EKS Fargate は「そもそも入らない」、Auto Mode は「設定を間違えるとサイレントに動かない」。Auto Mode でイベント 0 件なら、まず driver 設定を疑う。

## 3. デプロイ手順（＝実質 Auto Mode の手順）

EKS Fargate には Sysdig を入れられないので、EKS でランタイムを見るなら Auto Mode（or マネージドノードグループ）です。

### EKS Auto Mode（DaemonSet + universal eBPF）

driver を `universal_ebpf` に固定するのが肝。

```bash
helm repo add sysdig https://charts.sysdig.com && helm repo update
helm upgrade --install sysdig-agent sysdig/sysdig-deploy \
  -n sysdig-agent --create-namespace -f automode-values.yaml
```

```yaml
# automode-values.yaml（要点）
global:
  sysdig:
    accessKey: <AGENT_ACCESS_KEY>
    region: <us1|us2|us4|eu1|au1>
  clusterConfig:
    name: <your-eks-automode-cluster>
agent:
  ebpf:
    enabled: true
    kind: universal_ebpf   # Bottlerocket はモジュール不可 → universal 一択
```

（参考：ECS Fargate 側は workload-agent サイドカー方式。EKS の議論とは別物として扱う）

### 実機で確認した（EKS Auto Mode 実クラスタ）

Auto Mode クラスタを立て、universal eBPF での検知まで通した実測。

**① universal eBPF が Bottlerocket でロードされる**
```
driver_selection: opening universal_ebpf with buf size 8388608 and 2 cpus per buf
sinsp_event_src: Successfully loaded scap driver: universal_ebpf
  driver_type: "universal_ebpf"
```
ノードは `Bottlerocket (EKS Auto, Standard)` / kernel 6.1.174。カーネルモジュール無しで syscall を観測。

**② ランタイム検知が上がる**（nginx Pod で `cat /etc/shadow` / `find / -name id_rsa`）
```
Rule: Read sensitive file untrusted  (Sysdig Runtime Notable Events)
proc chain: cat → bash → runc → containerd-shim → systemd
親 runc = /x86_64-bottlerocket-linux-gnu/.../runc     ← Bottlerocket の証跡
MITRE: T1552 credential access / T1083 discovery
```
プロセスツリーに Bottlerocket のパスが写り、「Auto Mode の管理ノード上で eBPF が効いている」ことがそのまま可視化される。**この画は EKS Fargate では原理的に撮れない**（エージェントが載らないため）。

> ハマりどころ（実測）：EIP 枠が尽きて NAT を無効化したら、**Auto Mode ノードが egress の無いプライベートサブネットに立ち上がり、エラーを出さずクラスタに join しなかった**（`Node not registered`、ワークロードは永久 Pending）。カスタム NodeClass でパブリックサブネットに寄せて解消。まさに「Auto Mode は設定ミスがサイレント」の実例。

## 4. 運用（Day 2）の視点 ← 本記事の主眼

導入は一度きり。運用は毎日続きます。EKS で Sysdig を回す＝ Auto Mode 前提での Day 2 を押さえます。

### 4-1. エージェントのライフサイクル管理

- Auto Mode は DaemonSet なので **更新は `kubectl rollout` で一撃**。バージョンドリフトは起きにくい（ノードが最新に収束）。
- ただし **Karpenter のノード churn** に注意。新ノードが立ってからエージェント DaemonSet が Ready になるまでの一瞬、ワークロードが先に載る隙がある。Pod の readiness/依存で吸収する。
- （EKS Fargate はそもそもエージェントが載らないので「更新」以前の問題）

### 4-2. OS 自動更新との付き合い方

- Auto Mode は **Bottlerocket を AWS が自動更新**。カーネルが上がってもドライバが **universal eBPF（CO-RE）なら再ビルド不要**で追従する。
- 逆に言えば、ここでカーネルモジュール方式を選ぶと自動更新のたびにサイレントに壊れる。**「Auto Mode を選ぶなら universal eBPF」はほぼ必須要件**。

### 4-3. インシデントレスポンスでできること／できないこと

「検知した後に何ができるか」。Auto Mode は SSH 不可でも、ノード常駐エージェント経由で手が打てる。

| レスポンス手段 | EKS Fargate | EKS Auto Mode |
|---|---|---|
| そもそもランタイム検知 | ❌（Sysdig 非対応）| ✅ |
| Pod の Kill / 再起動 | ✅（K8s API）| ✅ |
| ネットワーク隔離（動的 NetworkPolicy）| △（CNI 依存）| ✅（Cluster Shield）|
| ノードへ SSH してフォレンジック | ❌（ノードが無い）| ❌（Bottlerocket は SSH 不可）|
| ノードレベルのキャプチャ | ❌ | △（ノード常駐エージェント経由）|
| プロセスツリー/コマンド監査 | ❌ | ✅（eBPF フル）|

- EKS Fargate は **検知そのものが無い**ので、レスポンスは K8s API 経由の Pod 操作に限られる（Sysdig の文脈では死角）。
- Auto Mode は SSH 不可でもキャプチャ・プロセスツリー・動的ネットワーク隔離まで踏める。**対応の幅は圧倒的に Auto Mode**。

### 4-4. 障害切り分け（0 件が出たとき）

- **Auto Mode で 0 件** → まず「**driver 設定（universal_ebpf）とノード対応状況**」。Bottlerocket 更新後の互換性、DaemonSet の Ready 状態、agent key/collector の向き先を疑う。
- 検知直後（1〜2分）は API/UI に出ないことがある（propagation lag）。「観測の死角だ」と結論する前に、時間をおいて再確認し、エージェントの配置と driver をまず疑う。

### 4-5. スケール運用とキャパシティ

- Fargate：Pod = micro-VM でゼロスケール可、バーストに強い。ただし **ランタイムセキュリティは付かない**ので「セキュリティ要件のあるワークロードでは選べない」。
- Auto Mode：Karpenter が集約・Spot 混在まで面倒を見る。定常稼働で効率が出る。ノード churn 時のエージェント Ready を設計で吸収。

### 4-6. ガバナンス／可観測性

- **エージェント自身の死活監視**を必ず入れる（DaemonSet の Ready 率、collector への疎通）。「入れたつもりで見えていない」が最悪。
- コスト配賦のため NodePool にタグを付け、セキュリティ関連の EC2 消費を可視化。

## 5. コスト試算

前提：**20 Pod、各 0.5 vCPU / 1 GB、24/7、us-east-1、x86**。

「EKS ＋ ランタイムセキュリティ」という要件では、そもそも Fargate は候補から外れます。その上で **純粋な計算コスト**として両者を並べると：

| | EKS Fargate（※Sysdig 不可）| EKS Auto Mode |
|---|---|---|
| アプリ計算 | ~$360 | ~$561 (EC2) |
| クラスタ | $73 | $73 |
| マネージド手数料 | (単価に内包) | ~$67 |
| **Sysdig ランタイム** | **不可（$-）** | **~$0（DaemonSet が EC2 内包）** |
| ざっくり合計 | ~$433/月（ただし無防備）| **~$700/月（防御込み）** |

読み方：

- Fargate は計算単価だけ見ると安く見えるが、**ランタイムセキュリティが付けられない**ので「EKS で守る」要件では土俵に乗らない。
- Auto Mode は Sysdig を **DaemonSet で EC2 の空きに内包**でき、追加課金が **Pod 数に比例しない**。守りを込みで見ると割安。
- （参考：ECS Fargate で Serverless Agent を使う場合、サイドカーの vCPU/メモリが Pod 数ぶん課金に乗る。EKS の話ではないが「Fargate ＋ ランタイム＝サイドカー課金」の直感は ECS 側で成立する）

## 6. どう選ぶか

- **ランタイムセキュリティが要る EKS ワークロード** → **Auto Mode（or マネージドノードグループ）一択**。Fargate は Sysdig 非対応で選べない。**universal eBPF 前提**で構築する。
- **セキュリティ要件が薄い / バッチ・イベント駆動でノード運用を消したい** → Fargate も可。ただし Sysdig ランタイムは付かない前提で。
- **サーバレスかつランタイムを見たい** → それは EKS ではなく **ECS Fargate ＋ Serverless Agent** の world。設計を分けて考える。

## まとめ

- **入れ方**：EKS Fargate は Sysdig 公式サポート外。Auto Mode は DaemonSet + universal eBPF。
- **運用**：Auto Mode は更新が一撃・OS 自動更新に universal eBPF で追従・IR の手も広い。Karpenter churn だけ設計で吸収。
- **コスト**：計算単価は Fargate が安く見えるが、EKS で守るなら Auto Mode 一択。守りは EC2 内包で Pod 数に比例しない。

---

### 注記（正確性・一次ソース）

1. 「Autopilot」は GKE。EKS は **Auto Mode**。
2. **"AWS Fargate is not supported on EKS"**（Sysdig Installation Requirements、一次ソースで確認）。
3. Sysdig Serverless Agent は **ECS Fargate 専用**。イメージ `quay.io/sysdig/workload-agent`（v5.0.0+ サイドカー既定、`pid_mode: "task"`）。
4. Bottlerocket はカーネルモジュール不可 → driver は **universal eBPF 必須**（agent 12.17.0+ は同梱）。
5. 数値・イメージ名・values キーは改版が入りやすい。公開前に現行 docs で最終確認。

### 参考

- Sysdig Installation Requirements（"AWS Fargate is not supported on EKS"）
- Sysdig Serverless Agents / ECS on Fargate（Terraform, workload-agent イメージ）
- Sysdig Understand Agent Drivers（universal eBPF）
- AWS EKS Pricing / AWS Fargate Pricing
- AWS Builder Center: Understanding EKS Auto Mode and Fargate
