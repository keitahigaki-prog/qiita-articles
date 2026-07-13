---
title: ECS と Kubernetes の違いを多方面から徹底比較 ― 技術・ユースケース・障害耐性・運用・セキュリティ
tags:
  - AWS
  - ECS
  - Kubernetes
  - コンテナ
  - CloudSecurity
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

コンテナオーケストレーターを選ぶとき、必ず出てくるのが「ECS か Kubernetes か」という問い。
ネット上には「K8s は難しい」「ECS は AWS ロックイン」といった断片的な話が溢れていますが、本質を一言でまとめるとこうです。

> **ECS は「AWS が運用の大半を隠してくれる薄いスケジューラ」、Kubernetes は「自分で組み立てる分散システムのフレームワーク」。**

この設計思想の違いが、技術・ユースケース・障害耐性・運用・セキュリティのすべての差に効いてきます。

```mermaid
flowchart TB
  subgraph ECS["ECS ＝ 薄いスケジューラ"]
    direction TB
    E1[アプリ] --> E2[Task / Service]
    E2 --> E3["AWS が運用する<br/>コントロールプレーン<br/>（不可視・無料）"]
  end
  subgraph K8S["Kubernetes ＝ 分散システム FW"]
    direction TB
    K1[アプリ] --> K2[Pod / Deployment]
    K2 --> K3["自分で組む抽象レイヤー<br/>RBAC / CRD / CNI / Ingress"]
    K3 --> K4["コントロールプレーン<br/>etcd / API / scheduler"]
  end
  classDef ecs fill:#cde2fb,stroke:#2a78d6,color:#0b0b0b;
  classDef k8s fill:#d3f2e6,stroke:#1baf7a,color:#0b0b0b;
  class E1,E2,E3 ecs;
  class K1,K2,K3,K4 k8s;
```

:::note info
筆者はクラウドセキュリティ（CSPM / CNAPP）が専門なので、最後に**セキュリティ運用の観点**も一節足しています。
:::

---

# TL;DR（結論だけ先に）

- **迷ったら ECS（Fargate）**。AWS 限定・小〜中規模・運用リソースが少ないなら、これで大半の要件は満たせる。
- **Kubernetes を選ぶべき**なのは、①マルチクラウド/オンプレ要件がある ②大規模・多チームで統制が要る ③Operator/CRD でプラットフォームを内製したい、のいずれか。
- 耐障害性は **K8s が上限（設計自由度）が高く、ECS が下限（デフォルトの堅牢さ）が高い**。
- 運用は **ECS ＝ AWS にアウトソース / K8s ＝ 自組織に内製**。専任チームを持てない組織が K8s を選ぶと、アップグレードと運用維持で疲弊する。

---

# 1. 技術的アーキテクチャの違い

まず全体像。両者とも「コントロールプレーン（頭脳）」と「データプレーン（コンテナが実際に動く場所）」に分かれます。違いは**どこまで自分で持つか**です。

## ECS のアーキテクチャ

```mermaid
flowchart TB
  ALB["ALB / NLB"]
  subgraph CP["コントロールプレーン（AWS 管理・不可視・無料）"]
    S["ECS Scheduler<br/>desired count を維持"]
  end
  subgraph DP["データプレーン"]
    direction TB
    F1["Fargate Task"]
    F2["Fargate Task"]
    EC2["EC2 起動タイプ Task"]
  end
  IAM["Task Role / IAM"]
  ALB --> F1 & F2
  S -. 配置・自己修復 .-> F1 & F2 & EC2
  IAM --> F1
  classDef ecs fill:#cde2fb,stroke:#2a78d6,color:#0b0b0b;
  classDef aws fill:#eef5ff,stroke:#2a78d6,color:#0b0b0b;
  classDef neutral fill:#f0efec,stroke:#898781,color:#0b0b0b;
  class S ecs;
  class F1,F2,EC2 aws;
  class ALB,IAM neutral;
```

コントロールプレーンは AWS が持っていて**見えないし課金もされない**。ユーザーは Task を定義して「3つ動かして」と言うだけ。

## Kubernetes のアーキテクチャ

```mermaid
flowchart TB
  ING["Ingress"]
  subgraph CP["コントロールプレーン（EKS は有料 / 自前は要運用）"]
    direction TB
    API["API server"]
    ETCD[("etcd")]
    SCH["scheduler"]
    CM["controller-manager"]
    API --- ETCD
    API --- SCH
    API --- CM
  end
  subgraph N1["Node（EC2 / VM / オンプレ）"]
    direction TB
    KL["kubelet"]
    CNI["CNI plugin<br/>Calico / Cilium / VPC CNI"]
    P1["Pod"]
    P2["Pod"]
    KL --> P1 & P2
  end
  SVC["Service / CoreDNS"]
  ING --> SVC --> P1 & P2
  API -. desired state 維持 .-> KL
  classDef k8s fill:#d3f2e6,stroke:#1baf7a,color:#0b0b0b;
  classDef neutral fill:#f0efec,stroke:#898781,color:#0b0b0b;
  class API,ETCD,SCH,CM,KL,CNI,P1,P2,SVC k8s;
  class ING neutral;
```

コントロールプレーンの構成要素が全部露出していて、**etcd のバックアップや証明書ローテ、CNI の選定まで自分の責任範囲**に入ってきます（EKS なら AWS が肩代わり）。

## 主要スペックの対比

| 観点 | ECS | Kubernetes |
|---|---|---|
| コントロールプレーン | AWS 管理・不可視・**無料** | etcd + API server + scheduler 等。EKS は有料（$0.10/h/クラスタ）、自前は全部運用 |
| 最小デプロイ単位 | Task | Pod |
| 抽象の階層 | Task Definition → Service（浅い） | Pod → ReplicaSet → Deployment（多層） |
| 構成管理 | JSON の Task Definition | YAML + 調整ループ（reconciliation loop）が中核 |
| データプレーン | EC2 起動タイプ / **Fargate** | ノード（EC2/VM/オンプレ）、EKS Fargate も一部 |
| ネットワーク | awsvpc で Task に ENI 直付け | CNI プラグインを選定・運用 |
| サービスディスカバリ | Cloud Map / ALB | 組み込み Service + CoreDNS + Ingress |
| 秘匿情報 | SSM / Secrets Manager | ConfigMap / Secret（+ External Secrets） |

## 抽象の階層 ― ここが「覚えることの多さ」の正体

```mermaid
flowchart TB
  subgraph E["ECS：浅い"]
    direction LR
    ED["Task Definition"] --> ES["Service"] --> ET["Task 群"]
  end
  subgraph K["Kubernetes：多層"]
    direction LR
    KD["Deployment"] --> KR["ReplicaSet"] --> KP["Pod 群"]
  end
  classDef ecs fill:#cde2fb,stroke:#2a78d6,color:#0b0b0b;
  classDef k8s fill:#d3f2e6,stroke:#1baf7a,color:#0b0b0b;
  class ED,ES,ET ecs;
  class KD,KR,KP k8s;
```

## 同じ「nginx を 3 つ動かす」を書き比べる

**ECS（Task Definition 抜粋 + `desiredCount: 3` の Service）**

```json
{
  "family": "web",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    { "name": "nginx", "image": "nginx:1.27", "portMappings": [{ "containerPort": 80 }] }
  ]
}
```

**Kubernetes（Deployment + Service）**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels: { app: web }
  template:
    metadata:
      labels: { app: web }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector: { app: web }
  ports:
    - port: 80
```

やることは同じでも、K8s は概念（Deployment / Service / selector / label）が多い。この差が学習コストに直結します。

---

# 2. ユースケース適性

```mermaid
flowchart TB
  subgraph ECSU["ECS が向く"]
    direction TB
    U1["AWS 全振り・移植性不要"] ~~~ U2["インフラ専任が薄い"] ~~~ U3["Fargate で<br/>ノード管理を消したい"] ~~~ U4["まず早く価値を出したい"]
  end
  subgraph K8SU["Kubernetes が向く"]
    direction TB
    V1["マルチクラウド / オンプレ"] ~~~ V2["多チームで細かい統制"] ~~~ V3["Operator / CRD で<br/>内製 PaaS"] ~~~ V4["カナリア / GitOps<br/>/ メッシュ"]
  end
  classDef ecs fill:#cde2fb,stroke:#2a78d6,color:#0b0b0b;
  classDef k8s fill:#d3f2e6,stroke:#1baf7a,color:#0b0b0b;
  class U1,U2,U3,U4 ecs;
  class V1,V2,V3,V4 k8s;
```

判断軸を絞るなら「**AWS だけで完結するか**」と「**プラットフォームを作る側に回るか、使うだけか**」の 2 つです。

---

# 3. 障害耐性・可用性

## 自己修復は「思想が同じ」

両者とも「宣言した数を維持する」調整ループを回します。ここは考え方が一緒。

```mermaid
sequenceDiagram
  participant U as 宣言（desired = 3）
  participant C as コントローラ
  participant R as 実体（running）
  U->>C: あるべき数 = 3
  C->>R: 現在 = 2 を検知
  C->>R: 不足分 1 を再作成
  R-->>C: 現在 = 3
  Note over C,R: 差分がゼロになるまで<br/>永久にループし続ける
```

## マルチ AZ での分散

```mermaid
flowchart LR
  LB["ロードバランサ"]
  subgraph AZa["AZ-a"]
    Ta["Task / Pod"]
  end
  subgraph AZb["AZ-b"]
    Tb["Task / Pod"]
  end
  subgraph AZc["AZ-c"]
    Tc["Task / Pod"]
  end
  LB --> Ta
  LB --> Tb
  LB --> Tc
  classDef neutral fill:#f0efec,stroke:#898781,color:#0b0b0b;
  classDef good fill:#d9f2d9,stroke:#0ca30c,color:#0b0b0b;
  class LB neutral;
  class Ta,Tb,Tc good;
```

- ECS はプレースメント戦略（spread / binpack）で AZ 分散。
- K8s は `topologySpreadConstraints` / PodAntiAffinity で**より細かく**制御可能。

## 決定的な差 ― コントロールプレーン障害

| | ECS | Kubernetes |
|---|---|---|
| コントロールプレーン障害 | AWS が冗長化・SLA 提供。**ユーザーは何もしない** | セルフマネージドだと **etcd クォーラム・バックアップ・証明書ローテ**が生命線（EKS なら AWS 肩代わり） |
| ヘルスチェック | ALB / コンテナヘルスチェック | liveness / readiness / startup probe（粒度が細かい） |
| 障害の切り分け | 層が薄く速い | 層が多く（Pod→Node→CNI→etcd→API）調査が複雑 |

> **まとめ**：作り込めば K8s の方が高い可用性を設計できる。だが「何もしなくても壊れにくい」のは ECS（特に Fargate）。
> **耐障害性は K8s が上限が高く、ECS が下限が高い。**

---

# 4. エコシステム

```mermaid
flowchart LR
  K(("Kubernetes"))
  K --> HELM["Helm"]
  K --> GITOPS["ArgoCD / Flux<br/>(GitOps)"]
  K --> MESH["Istio / Linkerd<br/>(メッシュ)"]
  K --> OBS["Prometheus / Grafana"]
  K --> SCALE["KEDA / Karpenter"]
  K --> CRD["Operator / CRD"]

  E(("ECS"))
  E --> AWSN["AWS 純正で完結<br/>ALB / App Mesh / CloudWatch<br/>CodeDeploy / ECR / Copilot"]

  classDef k8s fill:#d3f2e6,stroke:#1baf7a,color:#0b0b0b;
  classDef ecs fill:#cde2fb,stroke:#2a78d6,color:#0b0b0b;
  class K,HELM,GITOPS,MESH,OBS,SCALE,CRD k8s;
  class E,AWSN ecs;
```

- **Kubernetes**：CNCF 中心に圧倒的。求人・知見・OSS の母数が桁違いで、CRD による無限拡張が効く。スキルはどのクラウドでも通用する。
- **ECS**：基本は AWS 純正。サードパーティは少ないが「AWS の中では全部つながっていて摩擦がない」。覚える概念が少なく、スキルは AWS 特化。

---

# 5. 運用（Operations）― 実務で一番効く差

## アップグレードのモデルが根本的に違う

```mermaid
flowchart TB
  subgraph ECSU["ECS"]
    EU["コントロールプレーンは AWS 任せ<br/>＝実質意識しない"]
  end
  subgraph K8SU["Kubernetes（年 3 回・EOL が速い）"]
    direction LR
    U1["K8s 本体を上げる"] --> U2["CNI/CSI/CoreDNS<br/>互換性確認"]
    U2 --> U3["アドオン追随"]
    U3 --> U1
  end
  classDef ecs fill:#cde2fb,stroke:#2a78d6,color:#0b0b0b;
  classDef warn fill:#fdeecb,stroke:#eda100,color:#0b0b0b;
  class EU ecs;
  class U1,U2,U3 warn;
```

K8s は本体 + アドオン（CNI/CSI/CoreDNS）の互換性を追い続ける“アップグレード地獄”が**運用の定常コスト**になります。

## 運用観点まとめ

| 運用観点 | ECS | Kubernetes |
|---|---|---|
| 学習コスト | 低い（数日で本番イメージ） | 高い（習熟に数ヶ月） |
| アップグレード | AWS 任せで実質意識しない | 年 3 回・EOL 速い。本体+アドオンの互換性追随 |
| オートスケール | Service Auto Scaling + Capacity Provider（Fargate は自動） | HPA/VPA + Cluster Autoscaler/Karpenter |
| 可観測性 | CloudWatch にほぼ統合（Container Insights） | Prometheus/Grafana/OTel を自前構築 |
| コスト | コントロールプレーン無料。Fargate は割高だが工数ゼロ | EKS クラスタ課金 + ノード + アドオン運用。**TCO は人件費が支配的** |
| 運用自動化の思想 | AWS が良きに計らう（カスタマイズ余地小） | GitOps + Operator で「運用をコード化」（上限が高い） |

> **運用の本質的トレードオフ**
> ECS は**運用作業を AWS にアウトソース**するモデル。楽だが AWS の設計思想の外には出られない。
> Kubernetes は**運用能力を自組織に内製**するモデル。専任チームを持てる規模なら強力だが、体制がないと維持コストで疲弊する。

---

# 6. Fargate on ECS vs Fargate on EKS

「Fargate ＝ ノード管理が消えるサーバーレス」という点は同じですが、**ECS に載せるか EKS に載せるかで別物**です。ここを混同すると設計を誤ります。

```mermaid
flowchart TB
  subgraph ECSF["ECS on Fargate"]
    direction TB
    ECP["コントロールプレーン<br/>無料"] ~~~ ET["Task ＝ Fargate microVM"]
  end
  subgraph EKSF["EKS on Fargate"]
    direction TB
    KCP["コントロールプレーン<br/>$0.10/h"] ~~~ KProf["Fargate Profile<br/>namespace / label で選択"]
    KProf --> KP["1 Pod ＝ 1 Fargate microVM"]
  end
  FG["Fargate 基盤<br/>（サーバーレス compute）"]
  ET --> FG
  KP --> FG
  classDef ecs fill:#cde2fb,stroke:#2a78d6,color:#0b0b0b;
  classDef k8s fill:#d3f2e6,stroke:#1baf7a,color:#0b0b0b;
  classDef neutral fill:#f0efec,stroke:#898781,color:#0b0b0b;
  class ECP,ET ecs;
  class KCP,KProf,KP k8s;
  class FG neutral;
```

## 何が同じで、何が違うか

**同じ**：ノード（EC2）の管理・パッチ・スケールが不要。Task/Pod ごとに独立した microVM で隔離され、使ったぶんだけ課金。

**違う**：EKS on Fargate は「Kubernetes の API と豊富なエコシステムを保ったまま、ノードだけ消す」構成。そのぶん **K8s 特有の制約**を Fargate 側が肩代わりできず、いくつかの機能が使えません。

| 観点 | ECS on Fargate | EKS on Fargate |
|---|---|---|
| コントロールプレーン料金 | **無料** | $0.10/h/クラスタ |
| デプロイ単位 | Task（複数コンテナ可） | Pod（**1 Pod = 1 microVM**） |
| 配置指定 | 起動タイプ指定だけ | **Fargate Profile**（namespace / label で対象を選択）が必須 |
| DaemonSet | 概念なし（不要） | **非対応**。ノードが無いため node-level agent が動かせない |
| 特権コンテナ / hostPath | 不可 | **不可** |
| GPU | 不可 | 不可 |
| 永続ボリューム | EFS | **EFS のみ**（EBS 不可） |
| 起動速度 | 速い | やや遅い（Pod ごとに microVM 起動） |
| エコシステム | AWS 純正 | **Helm / CRD / Operator など K8s 資産をそのまま利用** |
| 学習コスト | 最小 | K8s の知識は必要（ノード運用だけが消える） |

## セキュリティ・可観測性での大きな落とし穴（CNAPP 視点）

EKS on Fargate で最も効くのが **DaemonSet 非対応**です。Kubernetes ではセキュリティ/可観測性エージェント（Falco/Sysdig、各種 CNAPP のランタイムセンサー、ログ収集の Fluent Bit など）を **DaemonSet で全ノードに 1 つ配る**のが定石ですが、Fargate にはノードが無いため**この方式が使えません**。

```mermaid
flowchart TB
  subgraph EC2N["EKS on EC2（ノードあり）"]
    direction TB
    DS["DaemonSet で<br/>ノードに 1 センサー"]
    N1["Pod"]
    N2["Pod"]
    DS -. 全 Pod を監視 .-> N1
    DS -. 全 Pod を監視 .-> N2
  end
  subgraph FGN["EKS on Fargate（ノードなし）"]
    direction TB
    SC["各 Pod に<br/>サイドカー注入"]
    P1["Pod + sidecar"]
    P2["Pod + sidecar"]
    SC --> P1
    SC --> P2
  end
  classDef k8s fill:#d3f2e6,stroke:#1baf7a,color:#0b0b0b;
  classDef warn fill:#fdeecb,stroke:#eda100,color:#0b0b0b;
  class DS,N1,N2 k8s;
  class SC,P1,P2 warn;
```

そのため Fargate では **サイドカー方式**（各 Pod にセンサーを注入）や、Fargate 対応のフックに頼ることになり、カバレッジ設計と運用が変わります。ランタイム保護を前提にするなら、この制約は選定段階で必ず織り込むべきポイントです。

## 使い分け

- **ECS on Fargate** … AWS 完結・最小構成でサーバーレスにしたい。コントロールプレーン無料で最も安く速い。
- **EKS on Fargate** … K8s API とエコシステムは維持しつつ、特定ワークロード（例：バースト的なジョブ、テナント隔離したい Pod）だけノード管理を消したい。全部を Fargate にするより、**EC2 ノード + Fargate Profile のハイブリッド**で「隔離したい namespace だけ Fargate」という使い方が実務では多い。

---

# 7. セキュリティ運用の観点（CNAPP 視点の補足）

ここは筆者の専門なので少し踏み込みます。ポイントは**守るべき層の数＝攻撃面の広さ**です。

```mermaid
flowchart TB
  subgraph ECSS["ECS：攻撃面が小さい"]
    ES1["Task Role / IAM<br/>アタッチするだけ"]
  end
  subgraph K8SS["Kubernetes：守る層が多い（＝構成ミスの余地が広い）"]
    direction TB
    KS1["RBAC<br/>（過剰権限）"]
    KS2["NetworkPolicy"]
    KS3["PodSecurity<br/>Admission Control"]
    KS4["Secret 管理<br/>（平文リスク）"]
    KS5["サプライチェーン<br/>/ イメージ"]
  end
  CNAPP["CNAPP<br/>CSPM・ランタイム検知<br/>アドミッション・スキャン"]
  KS1 --> CNAPP
  KS2 --> CNAPP
  KS3 --> CNAPP
  KS4 --> CNAPP
  KS5 --> CNAPP
  classDef ecs fill:#cde2fb,stroke:#2a78d6,color:#0b0b0b;
  classDef crit fill:#fbe0dd,stroke:#d03b3b,color:#0b0b0b;
  classDef good fill:#d9f2d9,stroke:#0ca30c,color:#0b0b0b;
  class ES1 ecs;
  class KS1,KS2,KS3,KS4,KS5 crit;
  class CNAPP good;
```

- **ECS**：Task Role で IAM をアタッチするだけとシンプル。抽象が薄いぶん攻撃面が小さく、ガードレールの多くが AWS 側にある。
- **Kubernetes**：RBAC・NetworkPolicy・PodSecurity・Admission・Secret・サプライチェーンと守る層が多い。裏を返すと構成ミスの余地が広く（過剰権限 RBAC、公開 API server、特権 Pod、Secret 平文、CNI 設定不備…）、**CSPM + ランタイム検知 + アドミッション制御 + イメージスキャンを束ねた CNAPP の投資対効果が ECS より明確に高い**。IAM 連携は IRSA / EKS Pod Identity で行います。

一言でいうと、**ECS は守る対象がシンプル、K8s は守る対象が多いぶんセキュリティ製品の価値が出る**。

---

# 8. 選定フローチャート

```mermaid
flowchart TD
  Q1{"マルチクラウド /<br/>オンプレ要件は?"}
  Q2{"大規模・多チーム統制 or<br/>プラットフォーム内製?"}
  Q3{"ノード管理すら<br/>消したい?"}
  K["Kubernetes（EKS）"]
  F["ECS on Fargate"]
  E["ECS on EC2"]

  Q1 -->|Yes| K
  Q1 -->|No| Q2
  Q2 -->|Yes| K
  Q2 -->|No| Q3
  Q3 -->|Yes| F
  Q3 -->|No| E

  classDef q fill:#f0efec,stroke:#898781,color:#0b0b0b;
  classDef k8s fill:#d3f2e6,stroke:#1baf7a,color:#0b0b0b;
  classDef ecs fill:#cde2fb,stroke:#2a78d6,color:#0b0b0b;
  class Q1,Q2,Q3 q;
  class K k8s;
  class F,E ecs;
```

---

# まとめ

| | ECS | Kubernetes |
|---|---|---|
| 思想 | 運用を AWS にアウトソース | 運用を自組織に内製 |
| 強み | シンプル・低学習コスト・低運用負荷 | 移植性・拡張性・エコシステム・統制 |
| 弱み | AWS ロックイン・拡張性の上限 | 高い学習/運用コスト |
| 耐障害性 | デフォルトが堅牢（下限が高い） | 作り込めば最強（上限が高い） |
| セキュリティ | 攻撃面が小さい | 守る層が多く CNAPP の価値が高い |
| 向く組織 | 小〜中規模・AWS 特化 | 大規模・マルチクラウド・専任チームあり |

**「過剰な複雑性を買わない」のが正解**です。K8s は強力ですが、それを使いこなす体制がないなら ECS(Fargate) で十分なケースが大半。逆に、移植性やプラットフォーム内製が戦略なら K8s 一択です。

---

*この記事が役に立ったら LGTM & ストックしていただけると励みになります🙌*
