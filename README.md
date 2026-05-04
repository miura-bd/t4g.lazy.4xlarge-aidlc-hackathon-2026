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

## 5/10 提出物 (書類審査用 Inception 成果物)

5/10 までに、本リポジトリの `aidlc-docs/` 配下に AI-DLC Inception フェーズの成果物一式を公開する必要があります。

### 書類審査の観点

| 観点               | 該当する成果物                                   |
| ---------------- | ----------------------------------------- |
| ビジネス意図 (Intent) の明確さ | `requirements/requirements.md`            |
| 創造性とテーマ適合性       | `requirements/requirements.md` (Intent 章) / `user-stories/personas.md` |
| Unit 分解の適切さ      | `application-design/unit-of-work.md` 他    |
| ドキュメント品質         | 全成果物                                      |

### 提出物ファイル一覧

AI-DLC ワークフロー (`.aidlc-rule-details/inception/`) で規定された出力ファイルです。

#### 全体管理

| パス                            | 内容                                |
| ----------------------------- | --------------------------------- |
| `aidlc-docs/aidlc-state.md`   | 各フェーズ進行状態 / Extension 設定          |
| `aidlc-docs/audit.md`         | 各ステージの承認ログ・タイムスタンプ                |

#### 1. 要件分析 (Requirements Analysis)

| パス                                                          | 内容                       |
| ----------------------------------------------------------- | ------------------------ |
| `aidlc-docs/inception/requirements/requirements.md`         | Intent / 機能要件 / 非機能要件    |
| `aidlc-docs/inception/requirements/requirement-verification-questions.md` | 要件曖昧性の検証質問          |

#### 2. ユーザーストーリー (User Stories)

| パス                                                       | 内容                  |
| -------------------------------------------------------- | ------------------- |
| `aidlc-docs/inception/user-stories/personas.md`          | ペルソナ定義              |
| `aidlc-docs/inception/user-stories/stories.md`           | ユーザーストーリー           |
| `aidlc-docs/inception/plans/story-generation-plan.md`    | ストーリー生成計画           |
| `aidlc-docs/inception/plans/user-stories-assessment.md`  | ストーリー品質評価           |

#### 3. アプリケーション設計 (Application Design)

| パス                                                              | 内容              |
| --------------------------------------------------------------- | --------------- |
| `aidlc-docs/inception/application-design/application-design.md` | アプリケーション全体設計    |
| `aidlc-docs/inception/application-design/services.md`           | サービス分割          |
| `aidlc-docs/inception/application-design/components.md`         | コンポーネント定義       |
| `aidlc-docs/inception/application-design/component-methods.md`  | コンポーネント間メソッド    |
| `aidlc-docs/inception/application-design/component-dependency.md` | コンポーネント依存関係   |
| `aidlc-docs/inception/plans/application-design-plan.md`         | アプリ設計計画         |

#### 4. Unit of Work 計画

| パス                                                                | 内容               |
| ----------------------------------------------------------------- | ---------------- |
| `aidlc-docs/inception/application-design/unit-of-work.md`         | Unit of Work 一覧  |
| `aidlc-docs/inception/application-design/unit-of-work-dependency.md` | Unit 間依存関係     |
| `aidlc-docs/inception/application-design/unit-of-work-story-map.md` | Unit ↔ ストーリー対応 |
| `aidlc-docs/inception/plans/unit-of-work-plan.md`                 | Unit 分解計画        |

#### 5. 実行計画

| パス                                                  | 内容                |
| --------------------------------------------------- | ----------------- |
| `aidlc-docs/inception/plans/execution-plan.md`      | Inception 全体の実行計画 |

### 提出までのステップ

1. テーマ「人をダメにする」のアイデアをチーム 4 名で確定
2. AI-DLC Inception フェーズを順に実行し各 `.md` を埋める
3. `aidlc-state.md` で **Inception フェーズ完了** に更新
4. `main` ブランチに merge し、5/10 までに公開状態にする
5. 5/12 正午までに運営にリポジトリ URL を連絡

## 開発ルール

- ドキュメント・コミットメッセージ・PR は日本語で記述
- コミット規約: [Conventional Commits](https://www.conventionalcommits.org/)
- TDD 基本 (Red → Green → Refactor)
- ブランチ名: 単一文字の前にハイフン禁止 (例 `fix-japanese-encoding` は OK、`fix-f-encoding` は NG)
