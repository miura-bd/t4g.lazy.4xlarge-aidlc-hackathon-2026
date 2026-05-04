# t4g.lazy.4xlarge — AWS Summit Japan 2026 AI-DLC ハッカソン

[AWS Summit Japan 2026 AI-DLC ハッカソン](https://aws.amazon.com/jp/summits/japan/) 参加用リポジトリです。

## チーム

**t4g.lazy.4xlarge** (4 名)

| メンバー              |
| ----------------- |
| Miura Kazuki      |
| Ito Nari          |
| Matsumoto Shogo   |
| Kawano Shinichiro |

## テーマ

「人をダメにする」サービス

## 開発方法論

[AI-DLC (AI-Driven Development Lifecycle)](https://github.com/awslabs/aidlc-workflows) に沿って開発します。Claude Code を主たる AI コーディング環境として利用します。

- ワークフロー定義: [`CLAUDE.md`](./CLAUDE.md)
- 詳細ルール: [`.aidlc-rule-details/`](./.aidlc-rule-details/)

## ディレクトリ構成

```
.
├── CLAUDE.md                # AI-DLC ワークフロー (Claude Code 用)
├── .aidlc-rule-details/     # AI-DLC ルール詳細
├── aidlc-docs/              # AI-DLC 各フェーズの成果物 (Inception 等)
├── HANDOFF.md               # Claude Code セッション間引き継ぎドキュメント
└── README.md
```

## 重要な期日

| 日付            | 内容                                       |
| ------------- | ---------------------------------------- |
| 2026/05/10    | 応募締切 + 公開 Git リポジトリに Inception フェーズ成果物を公開 |
| 2026/05/12 正午 | 運営に Git リポジトリ URL を連絡                    |
| 2026/05/15    | 書類審査結果発表                                 |
| 2026/05/30    | 予選会 @麻布台ヒルズ AWS オフィス (MVP デモ + プレゼン)     |
| 2026/06/26    | 決勝 @幕張メッセ AWS Summit Japan 2026          |

## 開発ルール

- ドキュメント・コミットメッセージ・PR は日本語で記述
- コミット規約: [Conventional Commits](https://www.conventionalcommits.org/)
- TDD 基本 (Red → Green → Refactor)
- ブランチ名: 単一文字の前にハイフン禁止 (例 `fix-japanese-encoding` は OK、`fix-f-encoding` は NG)
