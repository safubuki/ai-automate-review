# AI PRレビュー自動化 セットアップガイド

> **このガイドの目的**  
> このリポジトリで実際に動作している「ドキュメント特化AIレビュー自動化」の仕組みを、自分のリポジトリに再現するための完全な手順を提供します。

---

## 目次

1. [前提条件](#前提条件)
2. [全体像の理解](#全体像の理解)
3. [方式1: GitHub Copilot Code Review の導入](#方式1-github-copilot-code-review-の導入)
4. [方式2: GitHub Models + GitHub Actions の導入](#方式2-github-models--github-actions-の導入)
5. [ドキュメントサイトのセットアップ（MkDocsの場合）](#ドキュメントサイトのセットアップmkdocsの場合)
6. [自分のドメインに合わせたカスタマイズ方法](#自分のドメインに合わせたカスタマイズ方法)
7. [コスト管理と運用ベストプラクティス](#コスト管理と運用ベストプラクティス)
8. [トラブルシューティング](#トラブルシューティング)
9. [他のプラットフォームへの移行ガイド](#他のプラットフォームへの移行ガイド)

---

## 前提条件

### 必要なGitHubプラン・権限

| 項目 | 方式1 (Copilot) | 方式2 (GitHub Models) | 備考 |
|------|------------------|------------------------|------|
| GitHubアカウント | 必要 | 必要 | - |
| Copilotライセンス | **必須** | 推奨 | 方式1はCopilot Code Reviewを利用するため必須 |
| GitHub Models利用権限 | 不要 | **必須** | OrganizationのCopilotプランに含まれる場合が多い |
| Repository Admin権限 | 必要 | 必要 | Ruleset作成・Secrets設定のため |

**重要（2026年6月以降）**:
- GitHub Copilot Code Review は **GitHub Actionsの分** を消費する可能性があります。
- 大量のPRが発生するリポジトリでは事前にコスト試算を推奨します。

---

## 全体像の理解

本サンプルでは以下の2つの方式を提供しています。

```
┌─────────────────────────────┐
│  PR作成 (feature → main)     │
└──────────────┬──────────────┘
               │
       ┌───────┴───────┐
       ▼               ▼
┌──────────────┐  ┌──────────────────────────┐
│ 方式1         │  │ 方式2                     │
│ Copilot      │  │ GitHub Models + Actions  │
│ 自動レビュー │  │ カスタムpromptで高制御   │
└──────────────┘  └──────────────────────────┘
       │                     │
       └─────────┬───────────┘
                 ▼
        人間レビューアが最終判断
```

**推奨の始め方**:
- 最初は **方式1のみ** を有効化（最も簡単）
- 物足りなければ **方式2** を追加（より細かい制御が可能）

---

## 方式1: GitHub Copilot Code Review の導入

### ステップ1: copilot-instructions.md の配置

このリポジトリの `.github/copilot-instructions.md` を自分のリポジトリの **リポジトリルート直下の `.github/` 配下** にコピーしてください。

**配置場所例**:
```
your-repo/
└── .github/
    └── copilot-instructions.md   ← ここ
```

> **注意**: Copilot Code Review はこのファイルの**最初の4,000文字まで**しか読みません。重要なルールは冒頭に配置してください。

### ステップ2: Repository Ruleset で自動リクエストを設定

1. GitHubで対象リポジトリを開く
2. **Settings** → **Rules** → **Rulesets** をクリック
3. **New ruleset** → **New branch ruleset** を選択
4. 以下の設定を行う:

**基本設定**:
- **Ruleset name**: `Require Copilot review on main`
- **Enforcement status**: `Active`

**Target branches**:
- `Include default branch` を選択（または `main` を指定）

**Branch rules** で以下を有効化:
- ✅ **Automatically request Copilot code review**
  - **Review new pushes**: 推奨（ON）
  - **Review draft pull requests**: 状況による（OFF推奨）

5. **Create** をクリック

これで、mainをターゲットにしたPR作成時にCopilotが自動でレビュアーとして追加されます。

### ステップ3: 動作確認

1. 新しいブランチを作成
2. 適当なドキュメントを編集してPRを作成
3. PR画面右側の **Reviewers** に「Copilot」が表示されていることを確認
4. Copilotがレビューを投稿するまで待つ（数秒〜数十秒）

---

## 方式2: GitHub Models + GitHub Actions の導入

この方式では、GitHub Modelsの推論APIをGitHub Actionsから呼び出し、**ドキュメントに特化した高精度なレビュー**を自動実行します。

### 必要なファイル（本リポジトリからコピー）

以下の2ファイルを自分のリポジトリに配置してください。

- `.github/prompts/docs-review.prompt.yml` — レビュー用のプロンプト定義
- `.github/workflows/ai-docs-review.yml` — 実際のレビューを実行するワークフロー

### ステップ1: ファイルのコピー

```bash
# 例: 自分のリポジトリにコピーする場合
mkdir -p .github/prompts .github/workflows
cp -r path/to/ai-automate-review/.github/prompts/docs-review.prompt.yml .github/prompts/
cp -r path/to/ai-automate-review/.github/workflows/ai-docs-review.yml .github/workflows/
```

### ステップ2: 権限の確認

ワークフロー上部に以下の権限が明記されていることを確認してください（すでに記載済み）:

```yaml
permissions:
  contents: read
  pull-requests: write
  models: read     # ← これが重要
```

### ステップ3: 初回動作確認

1. ブランチを作成して軽微なドキュメント変更をコミット
2. PRを作成（**Draftではない**状態にしてください）
3. **Actions** タブで `AI Docs Review` ワークフローが起動していることを確認
4. 完了後、PRのコメント欄にAIレビュー結果が投稿される

### 高度なカスタマイズ

- 使用モデルを変更したい場合: `.prompt.yml` の `model:` を編集
  - 例: `meta/llama-3.1-70b`（安価）や `anthropic/claude-3-5-sonnet` など
- 大規模PR対策: ワークフロー内でdiffサイズを判定して要約モードに切り替えるロジックを追加可能

### コストコントロールの推奨設定

本サンプルワークフローではすでに以下を実施しています:

- `if: github.event.pull_request.draft == false` でDraft PRを除外
- 週に大量のPRが発生する場合は、重要ブランチのみトリガーするよう `paths` フィルタを追加することを推奨

---

## ドキュメントサイトのセットアップ（MkDocsの場合）

---

## ドキュメントサイトのセットアップ（MkDocsの場合）

### 推奨構成

本サンプルでは **MkDocs + Material for MkDocs** を採用しています。
理由:
- 日本語対応が非常に優秀
- 企業内の運用マニュアルで実績多数
- GitHub Pagesへのデプロイが簡単
- 検索・ナビゲーションのUXが優れている

### ローカルでの確認方法

```bash
pip install mkdocs mkdocs-material
mkdocs serve
```

ブラウザで http://localhost:8000 を開くと即座に確認できます。

### ファイル構成のポイント

- `mkdocs.yml` — ナビゲーション、テーマ、プラグイン設定
- `docs/` 配下 — 実際のMarkdownファイル群
- `docs/用語集.md` — AIレビューで最も参照される重要ファイル

### GitHub Pagesへの自動デプロイ

本リポジトリに同梱の `.github/workflows/deploy-docs.yml` をコピーすると、
mainブランチへのプッシュ時に自動でGitHub Pagesが更新されます。

1. リポジトリの **Settings → Pages** で Source を `GitHub Actions` に設定
2. `deploy-docs.yml` を配置
3. mainにマージすれば自動公開

---

## 自分のドメインに合わせたカスタマイズ方法（最重要）

---

## 自分のドメインに合わせたカスタマイズ方法（最重要）

### プロンプトを自組織向けに調整するポイント

1. **用語集の参照を強くする**
   - 自組織専用の用語集.mdを用意し、prompt内で必ず照合させる

2. **ページタイプごとの必須セクションを定義**
   - 手順書 / 概念説明 / トラブルシューティング などで異なる必須項目を明記

3. **トーンを明示**
   - 「ですます調必須」「過度に砕けた表現の禁止」など

4. **優先度を付ける**
   - 「用語統一違反は重大問題」「軽微な文言は改善提案に留める」など

### 実際にカスタマイズするファイル

- **方式1の場合**: `.github/copilot-instructions.md` を直接編集
- **方式2の場合**: `.github/prompts/docs-review.prompt.yml` の `system:` 部分を編集

### カスタマイズ時の推奨プロセス

1. まずそのままコピーして1週間運用
2. AIが指摘した内容を振り返る
3. 「指摘してほしいのに指摘されない」点をpromptに追加
4. 「指摘されすぎてノイズになる」点を緩和
5. **必ずPRでprompt変更をレビュー**してからmainにマージ

---

## コスト管理と運用ベストプラクティス

---

## コスト管理と運用ベストプラクティス

### 推奨設定

- draft PRはレビュー対象外にする（workflowに `if: github.event.pull_request.draft == false` を追加）
- 初回は安価なモデルでトリアージ → 重要なPRのみ高性能モデル
- 週に1回、AIレビューのヒット率を振り返る

### 監視すべきメトリクス

- GitHubの Usage ページでModels / Copilot Review の使用量
- 人間がAIコメントを「解決済み」にする割合
- 重大な問題の見逃し件数（0を目指す）

---

## トラブルシューティング

### Copilotがレビューを投稿しない

- Copilotライセンスが有効か確認
- Rulesetが正しくActiveになっているか
- `.github/copilot-instructions.md` の文字数が4,000文字を超えていないか

### Actionsが失敗する（方式2）

- `models: read` 権限がworkflowに付与されているか
- GitHub ModelsがOrganizationで有効化されているか
- トークンのスコープを確認

### 出力品質が低い

- promptの `temperature` を下げる（0.1〜0.2推奨）
- システムプロンプトをより具体的にする
- 出力構造をさらに厳格に指定する

---

## 他のプラットフォームへの移行ガイド

### Docusaurusへ移行する場合

1. `mkdocs.yml` の内容を参考に `docusaurus.config.ts` を作成
2. `docs/` 配下のMarkdownはほぼそのまま利用可能
3. 画像パスの調整が必要な場合あり
4. レビュー自動化部分（.github配下）は変更不要

### VitePress / その他 への移行

- 基本的にMarkdownファイルはそのまま使用可能
- ナビゲーション定義のみ各ツールの方式に置き換え
- AIレビュー用の指示ファイルは流用可能

---

## 貢献・フィードバック

このガイドの改善提案は大歓迎です。
本リポジトリに対してPull Requestをお送りください。

---

**最終更新**: 2026-05-27

> 本ガイドは「実際に動くサンプル」を最優先に作成されています。
> 最新のGitHub UIや機能は随時変更される可能性があるため、公式ドキュメントも併せて参照してください。