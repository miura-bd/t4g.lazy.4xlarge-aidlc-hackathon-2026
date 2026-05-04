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

## AI-DLC Inception フェーズ概要

5/10 の応募締切までに **Inception フェーズを完了** させる必要があります。AI-DLC は 3 フェーズ構造 (Inception → Construction → Operations) ですが、書類審査の対象になるのは Inception フェーズのみです。

### Inception フェーズの目的

**「何を / なぜ」作るか** を明確化するフェーズ。実装前に、ビジネス意図・要件・ユーザー像・全体設計・実装単位 (Unit of Work) をドキュメント化します。

### Inception フェーズのステージ

| #  | ステージ                     | 区分          | 内容                                       | 本プロジェクトでの実行 |
| -- | ------------------------ | ----------- | ---------------------------------------- | ----------- |
| 1  | Workspace Detection      | ALWAYS      | プロジェクト状態判定 (新規 / 既存)                     | 新規 (Greenfield) として実行 |
| 2  | Reverse Engineering      | CONDITIONAL | 既存コード解析                                  | スキップ (新規開発のため) |
| 3  | Requirements Analysis    | ALWAYS      | Intent / 機能要件 / 非機能要件                    | 実行           |
| 4  | User Stories             | CONDITIONAL | ペルソナ + ユーザーストーリー                         | 実行 (ユーザー向けサービスのため) |
| 5  | Workflow Planning        | ALWAYS      | 後続フェーズの実行計画                              | 実行           |
| 6  | Application Design       | CONDITIONAL | コンポーネント / サービス分割                         | 実行 (新規アーキ設計のため) |
| 7  | Units Generation         | CONDITIONAL | 実装単位 (Unit of Work) 分解                   | 実行 (Unit 分解は審査観点のため) |

### フロー (本プロジェクト Greenfield 版)

```
Workspace Detection (新規確認)
        ↓
Requirements Analysis (Intent + 機能要件 + 非機能要件)
        ↓
User Stories (ペルソナ + ストーリー)
        ↓
Workflow Planning (後続実行計画)
        ↓
Application Design (コンポーネント / サービス分割)
        ↓
Units Generation (Unit of Work 分解 + ストーリーマップ)
        ↓
=== 5/10 提出ライン ===
        ↓
Construction (5/30 予選会 MVP に向けて)
```

### 進行ルール

- **各ステージ終了時にチーム承認**: 自動で次に進まない。「Request Changes」または「Continue」の 2 択
- **質問形式**: AI が聞く質問は専用 `.md` に記載、**A / B / C / D / E** の選択式で `[Answer]:` タグに回答 (E = Other で自由記述可)
- **状態管理**: `aidlc-docs/aidlc-state.md` でフェーズ進行状態を管理し、最終的に **Inception 完了** に更新する
- **監査ログ**: `aidlc-docs/audit.md` に全ユーザー入力・AI 応答を ISO8601 タイムスタンプ付で記録
- **チーム作業前提**: 関係者を巻き込み、各フェーズのレビュー・承認を実施

### 各ステージで具体的に作るもの

#### Requirements Analysis
- サービスのコア Intent (なぜ作るのか) を 1〜2 文で明確化
- 機能要件 / 非機能要件 (性能・コスト・セキュリティ・倫理境界)
- 曖昧な点は `requirement-verification-questions.md` で確認

#### User Stories
- ペルソナ (どんな人が使うか)
- "As a 〜 / I want 〜 / So that 〜" 形式のストーリー
- ストーリーごとの受け入れ基準

#### Application Design
- 全体アーキ図 (どんなコンポーネントが何個)
- サービス分割 (Bedrock / Lambda / S3 等の AWS マッピング)
- コンポーネント間 API / 依存関係

#### Units Generation
- 開発単位 (Unit of Work) 分解
- Unit ↔ ストーリー対応マップ
- Unit 間の依存・実装順

### 詳細ルール

各ステージの詳細手順は [`.aidlc-rule-details/inception/`](./.aidlc-rule-details/inception/) を参照。

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
