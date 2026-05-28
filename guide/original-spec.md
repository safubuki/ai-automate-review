# GitHub環境におけるAI活用PRレビュー自動化 仕様書

**作成日**: 2026年5月  
**バージョン**: 1.0  
**目的**: GitHubの公式機能・ソリューションを最大限活用した、AIによるPull Requestレビュー自動化の設計と仕様の整理

---

## 1. 目的と背景

### 1.1 目的
- 開発チームのコードレビュー負担を軽減し、レビュー品質と速度を向上させる。
- 人間のレビューアがより本質的・設計レベルの議論に集中できるようにする。
- GitHubのエコシステム内で完結し、外部APIキーの管理負担やベンダーロックインを最小化する。

### 1.2 背景
- 近年、GitHub CopilotやGitHub Modelsの進化により、PRレビュー領域でのAI活用が現実的になった。
- 従来のサードパーティツール（CodeRabbit, PR-Agent等）は高機能だが、GitHub外のキー管理やデータ送信が発生し、企業ガバナンス上ハードルが高いケースがある。
- GitHubは2025〜2026年にかけて「GitHub Models」と「Copilot Code Review」を公式に強化し、Actionsとの親和性も高まっている。

---

## 2. 実現方針（優先順位付き）

GitHub公式ソリューションを優先し、以下の順で検討する。

| 優先度 | 方式 | 概要 | 推奨シーン | 工数 |
|--------|------|------|------------|------|
| **1** | **GitHub Copilot Code Review** | GitHub公式のAIレビュー機能。PRに`@copilot`をレビュアーとして追加 | 最も手軽に始めたいチーム | 極小 |
| **2** | **GitHub Models + GitHub Actions** | 公式の`actions/ai-inference`アクションとGitHub Models推論APIを組み合わせたカスタムレビュー | レビュー観点の細かい制御が必要なチーム | 中 |
| **3** | **ハイブリッド** | Copilot + カスタムActions + CodeQL/ESLint等の静的解析の組み合わせ | セキュリティ・品質を強く担保したいチーム | 中〜大 |
| 4 | カスタムGitHub App | Octokit + GitHub Modelsでフルスクラッチ | 特殊要件（特定ドメイン知識の深掘り等）がある場合 | 大 |

**基本戦略**: まずは方式1を導入し、不足があれば方式2で補完する。

---

## 3. アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────────────┐
│                        GitHub Repository                         │
├─────────────────────────────────────────────────────────────────┤
│  .github/                                                        │
│   ├── workflows/                                                 │
│   │    └── ai-pr-review.yml          ← 方式2で使用               │
│   ├── prompts/                                                  │
│   │    └── code-review.prompt.yml    ← 方式2で使用（バージョン管理）│
│   ├── copilot-instructions.md       ← 方式1で使用               │
│   └── CODEOWNERS                                                  │
│                                                                  │
│  PRイベント (opened, synchronize, reopened)                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    GitHub公式AIレイヤー                           │
├─────────────────────────────────────────────────────────────────┤
│  方式1: GitHub Copilot Code Review                               │
│         - 自動レビュアー割り当て（Rulesetsで制御）                │
│         - カスタム指示: copilot-instructions.md                 │
│                                                                  │
│  方式2: GitHub Models                                            │
│         - モデル: openai/gpt-4o, anthropic/claude-3-5-sonnet 等  │
│         - 推論API: https://models.github.ai/inference/...        │
│         - 公式Action: actions/ai-inference@v1                   │
│         - gh models CLI拡張                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    GitHub Reviews / Checks API                   │
│  - レビューコメント投稿（サマリ / インライン）                     │
│  - Suggested Changes                                              │
│  - Checks API（Annotations）                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. 主要方式の詳細仕様

### 4.1 方式1: GitHub Copilot Code Review 自動化（推奨ファーストステップ）

#### 4.1.1 概要
GitHubが提供する公式AIコードレビュー機能。PRに対して直接コメントと変更提案を投稿する。

#### 4.1.2 自動化の方法
1. **手動**: PR画面でReviewersに「Copilot」を追加。
2. **自動化（推奨）**: Repository Rulesets または Branch protection rules で以下を設定。
   - トリガー: PR作成時、プッシュ時、Draft → Ready for review時。
   - アクション: 「Request Copilot review」または「Add Copilot as reviewer」。

#### 4.1.3 カスタマイズ
- リポジトリルートまたは`.github/`配下に **`copilot-instructions.md`**（または `AGENTS.md`）を作成。
- 内容例:
  ```markdown
  ## コードレビューの観点
  - セキュリティ: 認証・認可、入力検証、機密情報の扱い
  - パフォーマンス: N+1問題、不要なメモリ確保
  - テスト: 境界値、異常系、モック使用の適切性
  - 保守性: 命名、責務分離、ドキュメント
  - プロジェクト固有ルール: 詳細は社内コーディング規約を参照
  ```

#### 4.1.4 制限・注意点
- 2026年6月1日以降、一部の利用でGitHub Actions分が消費される場合あり。
- モデル選択は不可（GitHubが最適化したミックスモデルを使用）。
- 大規模リポジトリや特定言語で精度が変動する可能性。

---

### 4.2 方式2: GitHub Models + GitHub Actions（高制御・ガバナンス重視）

#### 4.2.1 概要
GitHub Modelsの推論APIをGitHub Actionsから呼び出し、完全カスタマイズ可能なレビュー処理を実現。

#### 4.2.2 必須権限
```yaml
permissions:
  contents: read
  pull-requests: write
  models: read          # ← これが重要
```

#### 4.2.3 推奨アクション
- **公式推奨**: `actions/ai-inference@v1`
- 代替: `gh models` CLI拡張（`gh extension install github/gh-models`）

#### 4.2.4 Prompt管理の仕様（重要）
- GitHubの **Modelsタブ** でプロンプトを視覚的に作成・テスト可能。
- 完成したプロンプトは **`.prompt.yml`** 形式で保存し、PR経由でバージョン管理。
- ファイル配置推奨: `.github/prompts/code-review.prompt.yml`

**`.prompt.yml` の例**:
```yaml
model: openai/gpt-4o
system: |
  あなたは経験豊富なシニアソフトウェアエンジニア兼セキュリティエンジニアです。
  以下のGit diffを徹底的にレビューしてください。

  重点確認項目:
  1. バグ・論理的誤り
  2. セキュリティ脆弱性（インジェクション、認証不備、シークレット露出等）
  3. パフォーマンス・スケーラビリティ
  4. 可読性・保守性・ベストプラクティス
  5. テスト不足

  出力は以下の構造で日本語で記述してください:
  ## サマリ
  ## 重大な問題
  ## 改善提案
  ## 良い点
temperature: 0.2
max_tokens: 4000
```

#### 4.2.5 ワークフロー例（基本形）
`.github/workflows/ai-pr-review.yml`

```yaml
name: AI PR Review (GitHub Models)

on:
  pull_request:
    types: [opened, synchronize, reopened]
  workflow_dispatch:

permissions:
  contents: read
  pull-requests: write
  models: read

jobs:
  ai-review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get diff
        id: diff
        run: |
          gh pr diff ${{ github.event.pull_request.number }} --patch > pr.diff
          echo "size=$(stat -c%s pr.diff)" >> $GITHUB_OUTPUT
        env:
          GH_TOKEN: ${{ github.token }}

      - name: AI Review
        id: review
        uses: actions/ai-inference@v1
        with:
          prompt-file: .github/prompts/code-review.prompt.yml
          input: |
            PRタイトル: ${{ github.event.pull_request.title }}
            PR本文:
            ${{ github.event.pull_request.body }}

            変更差分:
            ${{ steps.diff.outputs.content || '差分が取得できませんでした' }}
          # model は prompt.yml で指定済みの場合省略可

      - name: Post review summary
        uses: peter-evans/create-or-update-comment@v4
        with:
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            ## 🤖 AIコードレビュー結果（GitHub Models）

            ${{ steps.review.outputs.response }}

            ---
            *使用モデル: openai/gpt-4o / 自動生成レビューです。人間による最終確認を推奨します。*
```

#### 4.2.6 高度な機能要件（将来拡張）
- **インラインコメント対応**: モデル出力にファイル名・行番号・提案コードを含め、GitHub Reviews APIで投稿。
- **大規模PR対策**:
  - 一定行数超過時は「要約→詳細レビュー」の2段階処理。
  - ファイル単位で分割処理。
- **コンテキスト拡張**: MCP（Model Context Protocol）対応のinferenceアクションを利用し、リポジトリ全体の情報（Issues, 関連PR, ドキュメント）をモデルに渡す。
- **構造化出力**: JSON Schemaをpromptに定義し、確実にパース可能な形式で出力。

#### 4.2.7 コスト・レートリミット
- GitHub Modelsの利用は別途課金（Copilotプランによる）。
- 大量PRが発生するリポジトリでは、以下を検討:
  - 初回は安価モデル（例: meta/llama-3.1-70b）でトリアージ。
  - 重要なPRのみ高性能モデル使用。
  - ワークフローに `if: github.event.pull_request.draft == false` を追加。

---

## 5. セキュリティ・ガバナンス

### 5.1 トークン管理
- 常に `${{ github.token }}` を使用。長期PATの発行は極力避ける。
- ワークフロー外（ローカル実行等）でGitHub Modelsを使う場合は、**fine-grained PAT** に `models: read` スコープを付与。

### 5.2 データ漏洩対策
- GitHub ModelsはGitHubのインフラ内で推論が完結（2025年以降の推奨エンドポイント）。
- ただし、プロンプトに機密情報（本番DB接続情報等）が含まれないよう、**除外パターン**を定義すべき。
- 推奨: `.github/prompts/` にセキュリティレビュー用プロンプトとは別に「機密情報フィルタリング」ステップを設ける。

### 5.3 プロンプトのガバナンス
- すべてのプロンプト変更はPRレビュー必須とする。
- 定期的に「評価用データセット（golden set）」を用意し、プロンプト改修時の品質劣化を検知。

---

## 6. 運用・メンテナンス

### 6.1 推奨運用フロー
1. 開発者がPRを作成。
2. AIレビューが自動実行（サマリコメント or インライン）。
3. 人間レビューアがAIコメントを参照しつつレビュー。
4. 定期的にAIレビューの精度を振り返り、prompt.ymlを改善（PRで）。

### 6.2 監視すべきメトリクス
- AIレビュー実行回数・トークン使用量（GitHubのUsageページ）。
- 人間レビューアがAIコメントを「解決済み」にする割合。
- AIが指摘した重大バグが本番に流出した件数（0を目指す）。

### 6.3 メンテナンスポイント
- 3ヶ月ごとに主要モデルの性能変化を確認（GitHub ModelsのCompare機能使用）。
- プロジェクトの技術スタック変更時にpromptを更新。

---

## 7. 代替案・比較

| 方式 | GitHubネイティブ度 | カスタマイズ性 | インラインコメント | コスト透明性 | 導入難易度 | 推奨度 |
|------|------------------|----------------|--------------------|--------------|------------|--------|
| GitHub Copilot Code Review | ★★★★★ | ★☆☆☆☆ | ◎ | ◎ | 非常に低い | ★★★★★ |
| GitHub Models + Actions | ★★★★★ | ★★★★★ | △（追加実装必要） | ◎ | 低〜中 | ★★★★☆ |
| PR-Agent (オープンソース) | ★★☆☆☆ | ★★★★★ | ◎ | △ | 中 | ★★★☆☆ |
| CodeRabbit | ★☆☆☆☆ | ★★★☆☆ | ◎ | △ | 低 | ★★☆☆☆ |
| フルスクラッチ GitHub App | ★★★★☆ | ★★★★★ | ◎ | ◎ | 非常に高い | ★★☆☆☆ |

---

## 8. 実装ロードマップ（提案）

### Phase 1: クイックスタート（1〜2週間）
- GitHub Copilot Code Review を有効化。
- `copilot-instructions.md` をチームで作成・合意。
- Rulesetsで自動リクエストを設定。

### Phase 2: カスタムレビュー基盤構築（3〜6週間）
- `actions/ai-inference` を用いたワークフローを導入。
- 基本的なレビュー用prompt.ymlを作成。
- レビューサマリコメントの運用を開始。

### Phase 3: 高度化（任意）
- インラインコメント投稿機能の実装。
- 構造化出力 + 自動Suggested Changes。
- 静的解析ツール（CodeQL, ESLint, tsc等）との組み合わせ。
- 大規模PR向けの分割処理ロジック。

### Phase 4: ガバナンス強化
- プロンプト変更の必須レビュー運用。
- 評価用データセットの整備と定期評価。
- 利用量ダッシュボードの作成。

---

## 9. 参考資料

- [GitHub Models クイックスタート](https://docs.github.com/en/github-models/quickstart)
- [GitHub Models at scale](https://docs.github.com/en/github-models/github-models-at-scale/use-models-at-scale)
- [actions/ai-inference アクション](https://github.com/actions/ai-inference)
- [GitHub Copilot コードレビュー](https://docs.github.com/en/copilot/using-github-copilot/code-review/using-copilot-code-review)
- [REST API - Models Inference](https://docs.github.com/en/rest/models/inference)

---

## 付録A: すぐに試せる最小構成（コピペ用）

1. `.github/copilot-instructions.md` を作成（方式1）
2. Repository Settings → Rules → Rulesets で「Request Copilot review」を追加
3. または、`.github/workflows/ai-pr-review.yml` + `.github/prompts/code-review.prompt.yml` を作成（方式2）

---

*本仕様書は、GitHubの2026年時点の機能に基づいて作成。機能は随時更新されるため、最新の公式ドキュメントを参照のこと。*