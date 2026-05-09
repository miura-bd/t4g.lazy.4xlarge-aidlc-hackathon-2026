# Application Design Plan

**作成日**: 2026-05-09
**フェーズ**: AI-DLC Inception / Application Design
**深さレベル**: Standard
**生成対象**: `aidlc-docs/inception/application-design/{components, component-methods, services, component-dependency, application-design}.md`

> **役割**: AI はソフトウェアアーキテクトとして、`requirements.md` (FR-MODE / FR-COM / FR-OSHI / FR-RECV / FR-AI / FR-MATCH / NFR-ARCH / NFR-PERF) と `stories.md` (AI に任せる 5 領域 + 将来 AI-7) を入力に、コンポーネント分割 / メソッドシグネチャ / サービス層 / 依存関係 を設計する。

---

## 入力ソース

- `aidlc-docs/inception/requirements/requirements.md` (4 件追補済 / NFR-ARCH-01 = Bedrock AgentCore Serverless / FR-AI-01〜07)
- `aidlc-docs/inception/user-stories/personas.md` (4 ペルソナ)
- `aidlc-docs/inception/user-stories/stories.md` (MVP 6 + Roadmap 2 / 「AI に任せる 5 領域 + AI-7」を 0 章で総説)
- `aidlc-docs/inception/plans/execution-plan.md` (Application Design EXECUTE / Standard 深さ)

---

## 全体ステップチェックリスト

### Part 1: Planning
- [x] (Step 1) 質問書 Q1〜Q7 への回答受領 (2026-05-09T14:30 / 全問 AI 推奨案フル採用)
- [x] (Step 2) 回答曖昧性分析 → 残らず (Auto モードで follow-up 省略)
- [x] (Step 3) Plan 確定 (2026-05-09T14:35)

### Part 2: Generation
- [x] (Step 4) `components.md` (22 コンポーネント定義 + 責務 + インターフェース)
- [x] (Step 5) `component-methods.md` (メソッドシグネチャ + 入出力型)
- [x] (Step 6) `services.md` (Domain Service 7 のオーケストレーション + シーケンス)
- [x] (Step 7) `component-dependency.md` (依存マトリクス + Mermaid 全体図 + 4 シーケンス図)
- [x] (Step 8) `application-design.md` (統合版 / 5/10 提出メイン)
- [x] (Step 9) NFR (NFR-ARCH-01 / NFR-PERF-01 / NFR-ETH-01〜08 / NFR-OBS-01,2) 整合性確認
- [x] (Step 10) `aidlc-state.md` を Stage 6 ✅ Completed に更新 / ユーザー承認受領 (2026-05-09T15:30)

---

# 質問書 (Q1〜Q7)

> **記入方法**: 各 Q の下にある `[Answer]:` の右側に回答を記入してください。
> **AI 推奨案について**: 全問題で AI 推奨案を提示しています。「推奨で OK」と書くだけで進められます。
> **回答完了の合図**: すべての `[Answer]:` が埋まったら「回答記入しました」と Claude に伝えてください。

---

## Q1. コンポーネント分割アプローチ

**問い**: アプリケーション全体をどのアーキテクチャパターンで構成しますか?

- **A**: 4 層レイヤード (UI / API / Domain / Data) — 古典的、変化に弱い
- **B**: AI Agent 中心 (5 領域 + 1 = 6 Agent) + 周辺サービス — AI 任せのコンセプトに直接対応
- **C**: ヘキサゴナル (Port & Adapter) — Domain 中心、AI / DB / UI は Adapter 化
- **D**: ハイブリッド (B + C のミックス) — ヘキサゴナル境界の中で AI Agent をドメインサービスとして配置
- **E**: その他 (自由記述)

**AI 推奨**: **D (ハイブリッド)**。理由: 「AI に任せる」コンセプトには AI Agent 中心の設計 (B) が直感的だが、要件 NFR-ETH-01〜08 の倫理ガードレールやテスト容易性 (NFR-QA-02 プロンプトテスト) を考えると、Domain 中心のヘキサゴナル境界 (C) で AI Agent を Adapter として隔離する方が変更耐性が高い。両方の良い面を採用する。

[Answer]:  **D (ハイブリッド)**。理由: 「AI に任せる」コンセプトには AI Agent 中心の設計 (B) が直感的だが、要件 NFR-ETH-01〜08 の倫理ガードレールやテスト容易性 (NFR-QA-02 プロンプトテスト) を考えると、Domain 中心のヘキサゴナル境界 (C) で AI Agent を Adapter として隔離する方が変更耐性が高い。両方の良い面を採用する。

---

## Q2. Bedrock AgentCore の Agent 構成

**問い**: AI に任せる 5 領域 (+ Roadmap で AI-7) をどう Bedrock AgentCore に落としますか?

- **A**: 単一 Multi-Agent — 1 つの AgentCore で全 5 領域 + AI-7 を Tool として呼ぶ
- **B**: 機能別 Agent — 5 + 1 = 6 個の個別 Agent (PraiseGen / SelfAnalysis / DMRelay / Yakimoki / Coaching / DependencyMeter)
- **C**: ハイブリッド — 主要 4 つは個別 Agent、補助 (FR-AI-06 内部依存度計測) は単一 Agent 内 Tool
- **D**: その他 (自由記述)

**AI 推奨**: **B (機能別 Agent)**。理由: 各 AI 領域で**プロンプトのトーン・長さ・倫理制約が大きく異なる** (褒め文 vs 誘導尋問 vs 唯一性 DM vs 感情労働代行 vs ヤキモキ正当化 vs コーチング)。プロンプトテスト (NFR-QA-02) を独立に行うため Agent 単位の境界を明確化。Bedrock AgentCore の Multi-Agent Collaboration で必要に応じて連携。FR-AI-06 (内部依存度計測) は AI ではないので除外し、専用 Service として実装。

[Answer]: **B (機能別 Agent)**。理由: 各 AI 領域で**プロンプトのトーン・長さ・倫理制約が大きく異なる** (褒め文 vs 誘導尋問 vs 唯一性 DM vs 感情労働代行 vs ヤキモキ正当化 vs コーチング)。プロンプトテスト (NFR-QA-02) を独立に行うため Agent 単位の境界を明確化。Bedrock AgentCore の Multi-Agent Collaboration で必要に応じて連携。FR-AI-06 (内部依存度計測) は AI ではないので除外し、専用 Service として実装。

---

## Q3. データストア選定

**問い**: ユーザー / マッチング / DM / ログ等のデータをどこに保存しますか?

- **A**: DynamoDB のみ — フルサーバレス
- **B**: Aurora Serverless v2 のみ — RDB の関係性を活かす
- **C**: **DynamoDB (運用データ) + S3 (監査ログ / AI 入出力ログ)** — シンプル + Bedrock Serverless と整合
- **D**: DynamoDB + Aurora Serverless v2 + S3 — 適材適所のハイブリッド
- **E**: その他 (自由記述)

**AI 推奨**: **C (DynamoDB + S3)**。理由: MVP のシンプル性 + NFR-ARCH-01 (Bedrock AgentCore Serverless 構成) と整合。User / Match / DM のような単純な KV / 時系列はすべて DynamoDB の Single-Table Design で十分。NFR-OBS-01, 02 (AI 入出力 / 倫理違反検出ログ) は S3 に置き Athena でクエリ可能にする。

[Answer]: C (DynamoDB + S3)。理由: MVP のシンプル性 + NFR-ARCH-01 (Bedrock AgentCore Serverless 構成) と整合。User / Match / DM のような単純な KV / 時系列はすべて DynamoDB の Single-Table Design で十分。NFR-OBS-01, 02 (AI 入出力 / 倫理違反検出ログ) は S3 に置き Athena でクエリ可能にする。

---

## Q4. フロントエンド技術スタック

**問い**: Web フロントエンドはどう構築しますか?

- **A**: React + Vite + TypeScript (純 CSR / 認証は別途実装)
- **B**: Next.js + TypeScript (SSR / SEO 対応 / 認証は NextAuth)
- **C**: Vue 3 + Vite + TypeScript
- **D**: **AWS Amplify Gen 2 + React + TypeScript** (Cognito 認証 / CloudFront 配信 / Lambda 連携が標準で揃う)
- **E**: その他 (自由記述)

**AI 推奨**: **D (AWS Amplify Gen 2 + React)**。理由: NFR-ETH-05 (18 歳以上の年齢確認) には Cognito 連携が便利、AWS 環境との統合がワンクリックで済むためハッカソン MVP の開発工数を削減。NFR-A11Y-02 (レトロ UI) は Amplify UI でなく独自 CSS / Tailwind で実装可。

[Answer]: **D (AWS Amplify Gen 2 + React)**。理由: NFR-ETH-05 (18 歳以上の年齢確認) には Cognito 連携が便利、AWS 環境との統合がワンクリックで済むためハッカソン MVP の開発工数を削減。NFR-A11Y-02 (レトロ UI) は Amplify UI でなく独自 CSS / Tailwind で実装可。

---

## Q5. AI ストリーミング応答方式

**問い**: 褒め文 / 自己分析要約 / 唯一性 DM 等の AI 出力をどうフロントに届けますか?

- **A**: **REST + SSE (Server-Sent Events) — Lambda Response Streaming** で段階的にトークンを送出
- **B**: WebSocket (API Gateway WebSocket) — 双方向通信、過剰スペック気味
- **C**: REST のみ (3 秒以内に完全レスポンス、ストリーミング無し)
- **D**: その他 (自由記述)

**AI 推奨**: **A (REST + SSE / Lambda Response Streaming)**。理由: NFR-PERF-01 (3 秒以内の初動表示) は Bedrock のストリーミング応答 + SSE でクリアできる。褒め文が 1 文字ずつ出るレトロ風演出は審査ウケに繋がる。WebSocket は推し ↔ 推され リアルタイム通知のみで限定的に検討。

[Answer]: **A (REST + SSE / Lambda Response Streaming)**。理由: NFR-PERF-01 (3 秒以内の初動表示) は Bedrock のストリーミング応答 + SSE でクリアできる。褒め文が 1 文字ずつ出るレトロ風演出は審査ウケに繋がる。WebSocket は推し ↔ 推され リアルタイム通知のみで限定的に検討。

---

## Q6. 倫理ガードレール (NFR-ETH-01〜08) の実装方法

**問い**: 性的禁止 / 自傷誘導禁止 / 実在人物模倣禁止 / ヒモパトロン用語禁止 等を AI 出力で守らせる仕組みは?

- **A**: プロンプト埋め込みのみ (System Prompt で禁止指示)
- **B**: Bedrock Guardrails のみ (AWS マネージド)
- **C**: **プロンプト埋め込み + Bedrock Guardrails の二重保証**
- **D**: A + B + アプリ層出力フィルタの三重保証
- **E**: その他 (自由記述)

**AI 推奨**: **C (二重保証)**。理由: 性的表現禁止 / 自傷誘導禁止は Bedrock Guardrails の標準カテゴリでカバーできるが、「ヒモ / パトロン用語禁止」(NFR-ETH-08) のような**プロジェクト独自ルール**は System Prompt で指示するのが効率的。三重保証 (D) はオーバースペックで MVP には過剰。stories.md 倫理対応表で「直書き + マトリクス」の二重保証を採用したのと整合。

[Answer]:  **C (二重保証)**。理由: 性的表現禁止 / 自傷誘導禁止は Bedrock Guardrails の標準カテゴリでカバーできるが、「ヒモ / パトロン用語禁止」(NFR-ETH-08) のような**プロジェクト独自ルール**は System Prompt で指示するのが効率的。三重保証 (D) はオーバースペックで MVP には過剰。stories.md 倫理対応表で「直書き + マトリクス」の二重保証を採用したのと整合。

---

## Q7. 観測性ログ実装 (NFR-OBS-01, 02)

**問い**: AI 入出力 + 倫理違反検出ログをどう保存しクエリ可能にしますか?

- **A**: CloudWatch Logs のみ (短期保存 / 検索性低)
- **B**: **CloudWatch Logs (運用) + S3 アーカイブ + Athena クエリ** (長期保存 + SQL クエリ)
- **C**: 専用ログ DB (DynamoDB Streams + OpenSearch) — オーバースペック気味
- **D**: その他 (自由記述)

**AI 推奨**: **B (CloudWatch + S3 + Athena)**。理由: NFR-OBS-01 (AI 入出力 / プロンプト品質後評価) と NFR-OBS-02 (倫理違反検出) の両方をクリア。S3 + Athena は Glue カタログ無しでも JSON Lines を直接クエリ可能でハッカソン向き。

[Answer]: **B (CloudWatch + S3 + Athena)**。理由: NFR-OBS-01 (AI 入出力 / プロンプト品質後評価) と NFR-OBS-02 (倫理違反検出) の両方をクリア。S3 + Athena は Glue カタログ無しでも JSON Lines を直接クエリ可能でハッカソン向き。

---

# 想定アーキテクチャの俯瞰 (推奨案ベース)

```
┌─────────────────────────────────────────────────────────────────────┐
│ Frontend (Amplify + React + TypeScript / レトロ RPG 風 UI)          │
│   - キャラメイク画面 / ホーム / 自己分析 / DM 一覧 / 候補カード    │
└────────────────────┬────────────────────────────────────────────────┘
                     │ HTTPS (REST + SSE)
                     │ Cognito 認証 (18+ 年齢確認)
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ API Gateway (REST + Lambda Response Streaming)                      │
└────────────────────┬────────────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Application Layer (Lambda / Hexagonal: Domain + Adapter)            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Domain Services (オーケストレーション)                        │   │
│  │  - OnboardingService    - SelfAnalysisService              │   │
│  │  - MatchingService      - DMService                          │   │
│  │  - DailyPraiseService   - DependencyMeterService             │   │
│  │  - (Roadmap) BehaviorChangeService                          │   │
│  └──────────┬──────────────────────────┬────────────────────────┘   │
│             │ AI Adapter                │ Data Adapter              │
│             ▼                          ▼                            │
│  ┌──────────────────┐         ┌──────────────────────────┐         │
│  │ AI Adapters      │         │ Data Adapters             │         │
│  │  - PraiseAgent   │         │  - UserStore (DDB)         │         │
│  │  - SelfAnalysis  │         │  - MatchStore (DDB)        │         │
│  │  - DMRelayAgent  │         │  - DMStore (DDB)            │         │
│  │  - YakimokiAgent │         │  - AuditLogStore (S3)       │         │
│  │  - (RM) Coaching │         │                            │         │
│  └──────┬───────────┘         └──────────┬─────────────────┘         │
│         │ Guardrail (二重)                │                          │
│         │  - System Prompt               │                          │
│         │  - Bedrock Guardrails          │                          │
└─────────┼────────────────────────────────┼──────────────────────────┘
          ▼                                ▼
┌──────────────────────────┐    ┌──────────────────────────────┐
│ Amazon Bedrock AgentCore │    │ DynamoDB (運用) / S3 (監査) │
│  - Claude 4.x シリーズ    │    │ Athena (S3 ログクエリ)       │
└──────────────────────────┘    └──────────────────────────────┘

横断: CloudWatch Logs (運用) / CloudWatch Metrics (NFR-PERF-01 監視)
```

---

# 想定主要コンポーネント一覧 (推奨案採用時)

| カテゴリ | コンポーネント名 | 主責務 | 紐付くストーリー |
| --- | --- | --- | --- |
| Frontend | UIShell | レトロ RPG / mixi 風 UI のシェル | 全 Story |
| Frontend | CharacterMakerView | キャラメイク画面 (推し/推され/アシスタントキャラ) | US-COM-01 / US-RM |
| Frontend | SelfAnalysisView | 誘導尋問 + 要約 + 爆褒め画面 | US-COM-02 |
| Frontend | DMListView | DM 一覧 + 承認待ち枠 | US-OSHI-01 / US-RECV-01 |
| Frontend | CandidateCardView | 推され候補カード | US-OSHI-02 |
| Domain | OnboardingService | キャラメイク受付 + マッチング起動 | US-COM-01 |
| Domain | SelfAnalysisService | 誘導尋問の進行管理 | US-COM-02 |
| Domain | MatchingService | キャラメイク完了トリガーで両側マッチング | US-COM-01 シナリオ 2 / FR-MATCH-01〜04 |
| Domain | DMService | DM 送受信 + AI 代理生成 + 承認/微修正 | US-OSHI-01 / US-RECV-01 |
| Domain | DailyPraiseService | 日次褒め配信 | US-OSHI-02 / US-RECV-02 |
| Domain | DependencyMeterService | 内部依存度計測 (UI 非表示) | FR-AI-06 |
| Domain (Roadmap) | BehaviorChangeService | 6 段階コーチング進行管理 | US-RM-01, 02 |
| AI Adapter | PraiseAgent (Bedrock) | AI-1 褒め文生成 | US-COM-02 / 全爆褒めシーン |
| AI Adapter | SelfAnalysisAgent (Bedrock) | AI-2 誘導尋問質問生成と要約 | US-COM-02 |
| AI Adapter | DMRelayAgent (Bedrock) | AI-3 唯一性 DM 並列生成 + AI-4 感情労働代行 | US-OSHI-01 / US-RECV-01 |
| AI Adapter | YakimokiAgent (Bedrock) | AI-5 既読スルー正当化文 | US-OSHI-01 |
| AI Adapter (Roadmap) | CoachingAgent (Bedrock) | AI-7 段階別コーチング | US-RM-01, 02 |
| Cross-cutting | PromptGuardrail | 二重ガードレール (Prompt + Bedrock Guardrails) | NFR-ETH-01〜08 |
| Cross-cutting | AuditLogger | AI 入出力 + 倫理違反検出ログ | NFR-OBS-01, 02 |
| Data | UserStore (DDB) | User / Mode / PraiseProfile / AssistantCharacter | 全 Story |
| Data | AnalysisStore (DDB) | SelfAnalysisLog / ActionLog | US-COM-02 |
| Data | MatchStore (DDB) | Match / Relationship / Ranking | US-COM-01 / US-OSHI / US-RECV |
| Data | DMStore (DDB) | DM (送受信) / 承認待ち枠 | US-OSHI-01 / US-RECV-01 |
| Data | AuditLogStore (S3) | AI 入出力 / 倫理ログ | NFR-OBS-01, 02 |

---

# 完了基準

- 全 `[Answer]:` タグが埋まっている
- AI が回答曖昧性を分析 → 残らないことを確認
- ユーザーが「Plan 承認 / Generation 進めて OK」と明示
- audit.md に Q&A 全文 + 承認応答を記録
