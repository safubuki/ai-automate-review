# AI活用PRレビュー自動化 サンプル環境

本リポジトリは、**GitHub公式のAI機能**を活用したPull Requestレビューの自動化を、実際に動作する形で体験・検証するためのサンプル環境です。

特に「**運用マニュアル・手順書**」のようなドキュメントをチームで作成・メンテナンスするプロジェクトを想定したレビュー自動化に特化しています。

## このリポジトリで実現していること

- **方式1**: GitHub Copilot Code Review を活用した自動レビュー
- **方式2**: GitHub Models + GitHub Actions によるカスタムレビュー（ドキュメント特化）

どちらの方式も、**コードレビューとは異なる「ドキュメント品質」の観点**（用語統一、構造の一貫性、読者への配慮など）を重視して設計されています。

## 目的

- 人間のレビューアが「内容の正確性」や「設計レベルの判断」に集中できるようにする
- ドキュメント特有の品質問題（用語の揺れ、構造の崩れ、トーンの不統一など）を早期に検知する
- GitHubの公式機能だけで完結し、外部ツールの導入コストやガバナンス負担を最小化する

## クイックスタート

### 1. ローカルでドキュメントを確認する

```bash
# MkDocs Material をインストール
pip install mkdocs mkdocs-material

# ローカルサーバーを起動
mkdocs serve
```

ブラウザで http://localhost:8000 を開くと、美しいHTMLで閲覧できます。

### 2. AIレビューを体験する（方式2）

詳細は [セットアップガイド](https://github.com/safubuki/ai-automate-review/blob/main/guide/SETUP_GUIDE.md) を参照してください。

## ドキュメント構成

- [用語集](用語集.md) — すべてのドキュメントが準拠すべき用語定義
- はじめに/
- 運用手順/
- 開発ガイド/

## 関連リンク

- [セットアップガイド](https://github.com/safubuki/ai-automate-review/blob/main/guide/SETUP_GUIDE.md) — この仕組みを自分のリポジトリに導入する方法
- [元の仕様書](https://github.com/safubuki/ai-automate-review/blob/main/guide/original-spec.md) — 設計思想の詳細

---

**注意**: 本サンプルは2026年時点のGitHub機能に基づいています。実際の利用時は最新の公式ドキュメントを確認してください。