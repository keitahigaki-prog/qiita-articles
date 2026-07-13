# qiita-articles

Qiita 記事のソースリポジトリ。[Qiita CLI](https://github.com/increments/qiita-cli) 互換のディレクトリ構成で管理する。

## 構成

```
qiita-articles/
├── public/        # 記事本体（Markdown + Qiita フロントマター）
│   └── *.md
├── images/        # 記事内で使う画像・SVG
└── README.md
```

## 記事一覧

| ファイル | タイトル |
|---|---|
| [public/ecs-vs-kubernetes.md](public/ecs-vs-kubernetes.md) | ECS と Kubernetes の違いを多方面から徹底比較 ― 技術・ユースケース・障害耐性・運用・セキュリティ |

## 執筆・公開フロー（Qiita CLI）

```bash
# 初回のみ
npm install @qiita/qiita-cli --save-dev
npx qiita login
npx qiita init      # 既存 public/ を維持する場合は不要

# プレビュー（http://localhost:8888）
npx qiita preview

# 公開・更新
npx qiita publish <記事ファイル名（拡張子なし）>
```

## 図について

構成図・比較図は Qiita ネイティブでレンダリングされる **Mermaid** を中心に作成。
配色は Claude Code の `dataviz` skill の検証済みカテゴリカルパレットを採用し、
`classDef` で全図に統一している（ECS/AWS = ブルー系、Kubernetes = アクア系）。
