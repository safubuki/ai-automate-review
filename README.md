# ai-automate-review

**GitHub公式機能による「ドキュメントレビュー」自動化のサンプル環境**

このリポジトリは、運用マニュアル・手順書などのドキュメントをチームで作成・メンテナンスするプロジェクトを想定した、**AIを活用したPRレビュー自動化**の完全に動作するサンプルです。

## 目的

- 人間のレビューアが「内容の正確性」や「本質的な判断」に集中できるようにする
- ドキュメント特有の品質問題（用語の揺れ、構造の崩れ、曖昧な手順表現など）をAIが自動で指摘
- GitHubの公式機能（Copilot Code Review / GitHub Models）だけで完結し、外部ツールのガバナンス負担を排除

## 提供している2つの方式

### 方式1: GitHub Copilot Code Review（最も簡単）

- `copilot-instructions.md` でドキュメント特化のレビュー指示を定義
- Repository Rulesets でPR作成時にCopilotを自動リクエスト
- 導入工数: 極小

### 方式2: GitHub Models + GitHub Actions（高制御）

- ドキュメントレビューに特化した `.prompt.yml` を用意
- PRイベントでActionsが起動し、構造化されたレビューコメントを自動投稿
- 用語集との整合性チェックなど、細かい制御が可能

**推奨**: まずは方式1を導入し、必要に応じて方式2を追加してください。

## クイックスタート

### 1. ドキュメントサイトをローカルで確認（仮想環境推奨）

ホスト環境を汚染しないよう、プロジェクト直下に作成済みの `venv` を使用してください。

```powershell
# 仮想環境を有効化（PowerShell）
.\venv\Scripts\Activate.ps1

mkdocs serve
```

ブラウザで http://localhost:8000 を開くと、整形されたサンプルマニュアルが表示されます。

> 初回のみ `pip install mkdocs mkdocs-material` を仮想環境内で実行してください（すでにインストール済みです）。

### 2. AIレビューを体験する

詳細は **[セットアップガイド](guide/SETUP_GUIDE.md)** を参照してください。

- 方式1: Rulesetを設定するだけでCopilotが自動レビュー
- 方式2: 2ファイルをコピーするだけでActionsがドキュメント特化レビューを実行

## リポジトリ構成

このリポジトリでは、**公開するマニュアル**と**このプロジェクト自体の説明ドキュメント**を明確に分離しています。

```
docs/                          # MkDocsでHTML化する「サンプルマニュアル」専用
├── 用語集.md                  # AIレビューで最重要参照ファイル
├── 運用手順/
├── 開発ガイド/
└── ...

guide/                         # このワークスペース自体の説明ドキュメント
├── SETUP_GUIDE.md             # ★最も重要なドキュメント
│                              #   （このAIレビュー自動化を自分のリポジトリに導入する方法）
└── original-spec.md           # 設計仕様書（参考）

.github/
├── copilot-instructions.md    # 方式1用（ドキュメント特化指示）
├── prompts/
│   └── docs-review.prompt.yml # 方式2用
└── workflows/
    ├── ai-docs-review.yml     # 方式2 メイン
    └── deploy-docs.yml        # GitHub Pages自動デプロイ
```

## 特徴的なポイント

- **用語集中心設計**: `docs/用語集.md` をAIが常に参照する構成
- **意図的な悪い例**: `incident-response.md` に品質問題を埋め込んでおり、AIレビューの効果を実際に確認可能
- **再現性重視**: `guide/SETUP_GUIDE.md` は「30分〜1時間で自分の環境に導入できる」ことを最優先に執筆。`docs/` にはサンプルマニュアルのみ、`guide/` にはこのプロジェクト自体の説明を分離して配置しています。

## 関連ドキュメント

- **[セットアップガイド](guide/SETUP_GUIDE.md)** — この仕組みを自分のリポジトリに導入する完全ガイド
- [設計仕様書（参考）](guide/original-spec.md) — 元となったアーキテクチャ設計

## 注意事項

- 2026年6月以降、Copilot Code ReviewはGitHub Actions分を消費する可能性があります。
- 利用には適切なGitHub Copilot / GitHub Modelsのプランが必要です。
- 最終的な判断は必ず人間のレビューアが行ってください。

---

**このリポジトリは「実際に動くこと」を最優先に作成されています。**
改善提案・Pull Request大歓迎です。