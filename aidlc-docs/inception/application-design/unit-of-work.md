# Unit of Work — 推されと推し

**プロダクト**: 推されと推し / AI による洗脳で気づいたら幸せになってるあなたと私
**チーム**: t4g.lazy.4xlarge (Miura / Ito / Matsumoto / Kawano)
**作成日**: 2026-05-09
**フェーズ**: AI-DLC Inception / Units Generation
**位置づけ**: **5/10 書類審査提出物**。`application-design.md` を入力に、4 名で 5/15〜5/30 に並行実装可能な独立 Unit (Unit of Work) を定義する
**準拠**: `requirements.md` / `stories.md` / `application-design.md` / `unit-of-work-plan.md` (Q1=B / Q2=A / Q3=B / Q4=A / Q5=D / Q6=B)

---

## 0. Unit of Work とは

> **A unit of work is a logical grouping of stories for development purposes.**
> モノリス前提では論理モジュールの集合、マイクロサービス前提では独立デプロイ可能なサービスを指す。本プロジェクトは **モノレポ + 論理 Unit (内部に一部独立デプロイ Lambda)** のハイブリッドを採用する (Q2=A モノレポ + 単一 CI/CD)。

**呼称規約**:
- **Unit of Work**: 計画文脈での Unit (本ドキュメント)
- **Module / Package**: pnpm workspace 上のディレクトリ単位
- **Service**: 独立デプロイ可能な Lambda (apps/backend-* / scheduler-*)

---

## 1. Unit 一覧 (5 + Roadmap)

| ID | Unit 名 | 主担当 Story | デプロイ単位 | パッケージ位置 | MVP/RM | 想定 SP |
| --- | --- | --- | --- | --- | --- | --- |
| **Unit-A** | Frontend Foundation + UIShell + 認証 | 全 Story の UI | Amplify (CloudFront + S3) | `apps/frontend` | MVP | M |
| **Unit-B** | Onboarding + Matching | US-COM-01 / US-OSHI-02 候補 | Lambda × 1〜2 | `apps/backend-onboarding` | MVP | M |
| **Unit-C** | Self-Analysis + Daily Praise | US-COM-02 / US-OSHI-02 (日次) / US-RECV-02 | Lambda × 1 | `apps/backend-praise` | MVP | L |
| **Unit-D** | DM + 唯一性 + 感情労働代行 + ヤキモキ | US-OSHI-01 / US-RECV-01 | Lambda × 2 | `apps/backend-dm` + `apps/scheduler-yakimoki` | MVP | L |
| **Unit-E** | Cross-cutting + IaC | 横断 / 全 Story から利用 | packages (共有) + infra/ (CDK デプロイ) | `packages/{shared-types,guardrail,audit-logger,prompts}` + `infra/` | MVP | M |
| (Roadmap) **Unit-RM** | 行動変容モード | US-RM-01, 02 | (将来) | (将来) | Roadmap | XL |

---

## 2. 各 Unit の詳細

### 2.1 Unit-A: Frontend Foundation + UIShell + 認証

**目的**: エンドユーザーが触れる Web UI 全体。レトロ RPG 風 / 初期 mixi 風オレンジ色調。Cognito で 18+ 認証。

**含まれるコンポーネント** (`components.md` Section 1):
- UIShell (共通レイアウト + 認証ラッパ)
- CharacterMakerView
- SelfAnalysisView
- DMListView
- CandidateCardView

**含まれる Story の UI 部分**: 全 6 MVP ストーリー + Roadmap 2 (Roadmap UI 拡張は Unit-RM へ送り)

**デプロイ単位**: AWS Amplify Hosting (CloudFront + S3 + CI/CD は Amplify Console)

**主要な技術選定**:
- React 18 + TypeScript + Vite
- AWS Amplify Gen 2 (Auth / Storage / API クライアント生成)
- Tailwind CSS or 独自 CSS でレトロ UI 実装 (NFR-A11Y-02)
- SSE クライアント実装 (EventSource API + ReadableStream)

**依存**:
- Unit-B / C / D の Lambda API (REST + SSE 経由)
- Unit-E の `packages/shared-types` (DTO 共通定義)
- Cognito User Pool (Unit-E でプロビジョニング)

**想定スキル要件**: React / TypeScript / CSS (レトロ UI) / UX / SSE クライアント実装

**主要ファイル構成 (Greenfield)**:
```
apps/frontend/
├── package.json
├── vite.config.ts
├── amplify/                         (Amplify Gen 2 設定)
├── src/
│   ├── main.tsx
│   ├── components/UIShell.tsx
│   ├── views/
│   │   ├── CharacterMakerView.tsx
│   │   ├── SelfAnalysisView.tsx
│   │   ├── DMListView.tsx
│   │   └── CandidateCardView.tsx
│   ├── api/                         (Unit-B/C/D 呼び出しクライアント)
│   └── styles/retro.css             (NFR-A11Y-02)
└── public/
```

---

### 2.2 Unit-B: Onboarding + Matching

**目的**: キャラメイク受付 → 両側マッチング起動。「両面市場の起点」。

**含まれるコンポーネント**:
- OnboardingService
- MatchingService
- UserStore (DynamoDB Adapter)
- MatchStore (DynamoDB Adapter)

**含まれる Story (Service 層)**:
- US-COM-01 オンボーディング & 褒めキャラメイク + 両側マッチング起点 (主役)
- US-OSHI-02 候補提示と日次褒め — 候補表示部分のみ (日次褒めは Unit-C)

**デプロイ単位**: Lambda × 1 (`apps/backend-onboarding`)。サイズが大きくなれば 2 つに分割可

**主要 API**:
- `POST /onboarding/character` (OnboardingService.acceptCharacterMake)
- `POST /matching/trigger` (MatchingService.triggerMatching) — 内部呼び出し
- `GET /matching/candidates` (MatchingService.getCandidatesForOshi)
- `GET /matching/oshi-summary` (MatchingService.getOshiSummaryForRecv) — FR-MATCH-03
- `POST /matching/oshi` (MatchingService.recordOshi)

**依存**:
- Unit-E `packages/shared-types` / `packages/audit-logger`
- (Unit-D の DMService.generateUniquenessDMs を非同期で起動 — RECV 側のキャラメイク完了時 / Lambda Async invoke)
- DynamoDB Single-Table (Unit-E でプロビジョニング)

**想定スキル要件**: TypeScript / DynamoDB Single-Table Design / Lambda / NFR-PERF-01 (3 秒応答)

**主要ファイル構成**:
```
apps/backend-onboarding/
├── package.json
├── src/
│   ├── handlers/
│   │   ├── onboarding.ts            (POST /onboarding/character)
│   │   ├── matching-trigger.ts      (内部 Lambda Async 起動)
│   │   ├── matching-candidates.ts   (GET /matching/candidates)
│   │   ├── matching-summary.ts      (GET /matching/oshi-summary)
│   │   └── matching-oshi.ts         (POST /matching/oshi)
│   ├── services/
│   │   ├── onboarding-service.ts
│   │   └── matching-service.ts
│   └── adapters/
│       ├── user-store.ts
│       └── match-store.ts
└── tests/                           (NFR-QA-03)
```

---

### 2.3 Unit-C: Self-Analysis + Daily Praise

**目的**: 「褒めたおすループ」のエンジン。誘導尋問 → 要約 → 爆褒め と 日次褒め配信。

**含まれるコンポーネント**:
- SelfAnalysisService
- DailyPraiseService
- SelfAnalysisAgent (AI-2)
- PraiseAgent (AI-1)
- AnalysisStore (DDB Adapter)

**含まれる Story**:
- US-COM-02 誘導尋問自己分析と爆褒め (主役)
- US-OSHI-02 候補提示と日次褒めで日常の彩り — 日次褒め部分
- US-RECV-02 副業推されと釣れる行動の AI 提示

**デプロイ単位**: Lambda × 1 (`apps/backend-praise`)。SSE / Lambda Response Streaming 設定が必要

**主要 API**:
- `POST /self-analysis/start` (SelfAnalysisService.startAnalysis)
- `GET /self-analysis/next-question` (nextQuestion)
- `POST /self-analysis/answer` (submitAnswer)
- `GET /self-analysis/summary` **SSE** (summarizeAndPraise — 要約 + 爆褒めチェイン)
- `GET /daily-praise` **SSE** (DailyPraiseService.getDailyPraise)
- `POST /daily-praise/action-completion` **SSE** (submitActionCompletion)

**依存**:
- Unit-E `packages/shared-types` / `packages/guardrail` / `packages/audit-logger` / `packages/prompts`
- Bedrock (Claude 4.x)
- DynamoDB (Unit-E プロビジョニング) / S3 (AuditLogStore)

**想定スキル要件**: TypeScript / Bedrock SDK (streaming) / Lambda Response Streaming / SSE サーバ実装 / プロンプトエンジニアリング

**主要ファイル構成**:
```
apps/backend-praise/
├── package.json
├── src/
│   ├── handlers/
│   │   ├── self-analysis-start.ts
│   │   ├── self-analysis-next.ts
│   │   ├── self-analysis-answer.ts
│   │   ├── self-analysis-summary.ts (SSE / Lambda Response Streaming)
│   │   ├── daily-praise.ts          (SSE)
│   │   └── action-completion.ts     (SSE)
│   ├── services/
│   │   ├── self-analysis-service.ts
│   │   └── daily-praise-service.ts
│   ├── ai/
│   │   ├── self-analysis-agent.ts   (AI-2 / Bedrock 呼び出し)
│   │   └── praise-agent.ts          (AI-1)
│   └── adapters/analysis-store.ts
└── tests/                           (含 プロンプトテスト NFR-QA-02)
```

---

### 2.4 Unit-D: DM + 唯一性 + 感情労働代行 + ヤキモキ

**目的**: 両面市場の核となる DM 通信。AI が推され実ユーザーの代理として唯一性 DM を並列生成 + 推し → 推され DM への返信案を代理生成。ヤキモキ演出。

**含まれるコンポーネント**:
- DMService
- DMRelayAgent (AI-3 + AI-4)
- YakimokiAgent (AI-5)
- DependencyMeterService (内部)
- DMStore (DDB Adapter)

**含まれる Story**:
- US-OSHI-01 唯一性 DM とヤキモキ演出 (主役)
- US-RECV-01 感情労働全代行とゲーム化承認作業 (主役)

**デプロイ単位**: Lambda × 2:
- `apps/backend-dm` (REST API / DMService)
- `apps/scheduler-yakimoki` (EventBridge Scheduler 起動 / triggerYakimoki)

**主要 API**:
- `GET /dm` (DMService.listIncoming / listOutgoing)
- `POST /dm/{dmId}/generate-reply` (DMService.generateReplyCandidate — AI-4)
- `POST /dm/{dmId}/approve` (DMService.approve)
- `POST /dm/{dmId}/modify` (DMService.modify)
- `POST /dm/{dmId}/reject` (DMService.reject)
- (内部) `POST /dm/uniqueness` (DMService.generateUniquenessDMs — AI-3 / Unit-B から起動)
- (Scheduler) `triggerYakimoki` Lambda (EventBridge Scheduler ルール)

**依存**:
- Unit-E (Guardrail / AuditLogger / shared-types / prompts)
- Bedrock (Claude 4.x — AI-3 並列 / AI-4 / AI-5)
- DynamoDB / S3 / EventBridge Scheduler

**想定スキル要件**: TypeScript / Bedrock 並列呼び出し / EventBridge Scheduler / Lambda 間連携 / 状態機械 (DM の DRAFT → PENDING_APPROVAL → SENT → READ → YAKIMOKI)

**主要ファイル構成**:
```
apps/backend-dm/
├── package.json
├── src/
│   ├── handlers/
│   │   ├── list-dms.ts
│   │   ├── generate-reply.ts        (AI-4)
│   │   ├── approve.ts
│   │   ├── modify.ts
│   │   ├── reject.ts
│   │   └── generate-uniqueness.ts   (AI-3 / 内部 Lambda Async)
│   ├── services/dm-service.ts
│   ├── ai/
│   │   ├── dm-relay-agent.ts        (AI-3 + AI-4)
│   │   └── yakimoki-agent.ts        (AI-5)
│   ├── domain/dependency-meter.ts   (内部 / NFR-ETH-07)
│   └── adapters/dm-store.ts
└── tests/

apps/scheduler-yakimoki/
├── package.json
├── src/handlers/yakimoki.ts          (EventBridge Scheduler ターゲット)
└── tests/
```

---

### 2.5 Unit-E: Cross-cutting + IaC

**目的**: 横断機能 (倫理ガードレール / 監査ログ / 共通型 / プロンプト共通断片) と全インフラの IaC を 1 名が責任を持って整備する。

**含まれるコンポーネント**:
- PromptGuardrail (`packages/guardrail`)
- AuditLogger (`packages/audit-logger`)
- AuditLogStore (S3 Adapter — `packages/audit-logger` 内)
- 共通型 (DTO) (`packages/shared-types`)
- プロンプト共通断片 (System Prompt の倫理ルール / 用語禁止リスト) (`packages/prompts`)
- 全インフラの CDK Stack (`infra/`)

**含まれる Story**: 横断 / 全 Story から利用される

**デプロイ単位**:
- packages: 各 Unit の Lambda パッケージ時に includes
- `infra/`: CDK で AWS 環境にデプロイ (`cdk deploy`)

**主要 IaC スタック**:
```
infra/lib/
├── cognito-stack.ts                 (User Pool + 18+ 属性 / NFR-ETH-05)
├── api-gateway-stack.ts             (REST + SSE / Cognito Authorizer)
├── lambda-stack.ts                  (各 backend-* / scheduler-* の Lambda)
├── ddb-stack.ts                     (Single-Table 定義 + GSI)
├── s3-audit-stack.ts                (AuditLogStore + Athena Workgroup)
├── bedrock-guardrails-stack.ts      (Bedrock Guardrails 設定)
├── eventbridge-scheduler-stack.ts   (ヤキモキ用)
└── amplify-stack.ts                 (frontend ホスティング)
```

**主要パッケージ構成**:
```
packages/
├── shared-types/
│   ├── package.json
│   └── src/
│       ├── user.ts
│       ├── dm.ts
│       ├── match.ts
│       └── ...
├── guardrail/
│   ├── package.json
│   └── src/
│       ├── prompt-guardrail.ts      (System Prompt + Bedrock Guardrails 二重)
│       └── ng-words.ts              (ヒモ/パトロン 等 NG ワード辞書)
├── audit-logger/
│   ├── package.json
│   └── src/
│       ├── audit-logger.ts
│       └── audit-log-store.ts       (S3 Adapter)
└── prompts/
    ├── package.json
    └── src/
        ├── ethical-rules.ts         (NFR-ETH-01〜08 共通 System Prompt 断片)
        ├── praise-prompts.ts
        ├── self-analysis-prompts.ts
        ├── dm-relay-prompts.ts
        ├── yakimoki-prompts.ts
        └── coaching-prompts.ts      (Roadmap)
```

**想定スキル要件**: TypeScript / AWS CDK / Bedrock Guardrails / IAM 最小権限 / セキュリティ / プロンプトエンジニアリング (倫理ルール埋め込み)

**依存**: なし (他 Unit は Unit-E に依存する一方向)

---

### 2.6 (Roadmap) Unit-RM: 行動変容モード

**目的**: ロードマップ実装。OS 再インストール = 適応型自己実現 のコアレイヤを構築。

**含まれるコンポーネント**:
- BehaviorChangeService
- CoachingAgent (AI-7)
- AssistantCharacter エンティティ (UserStore 拡張)
- (Frontend) アシスタントキャラビジュアル選択 UI 拡張 (Unit-A 拡張として)

**含まれる Story**:
- US-RM-01 推され OS の再インストール
- US-RM-02 推し OS の再インストール

**MVP では**: 実装しない (FR-MODE-03)。書類審査 / プレゼンでは「拡張余地として描かれる」のみ。

**5/30 予選会後の実装着手見込み**: 6/26 決勝向けに段階的に実装する場合の独立 Unit。

---

## 3. Code Organization 戦略 (Greenfield 専用)

### 3.1 モノレポ全体構造 (`unit-of-work-plan.md` 推奨採用)

```
oshi-osare/                           ← GitHub リポジトリルート
├── package.json                      (workspace ルート / pnpm)
├── pnpm-workspace.yaml
├── turbo.json                        (Turborepo パイプライン定義)
├── tsconfig.base.json
├── .gitignore
├── README.md
├── apps/                             ← デプロイ対象
│   ├── frontend/                    [Unit-A]
│   ├── backend-onboarding/          [Unit-B]
│   ├── backend-praise/              [Unit-C]
│   ├── backend-dm/                  [Unit-D]
│   └── scheduler-yakimoki/          [Unit-D の一部]
├── packages/                         ← 共有ライブラリ [Unit-E]
│   ├── shared-types/
│   ├── guardrail/
│   ├── audit-logger/
│   └── prompts/
├── infra/                            ← CDK [Unit-E]
│   ├── package.json
│   ├── lib/*.ts
│   └── bin/oshi-osare.ts
├── aidlc-docs/                       ← 設計ドキュメント (本ディレクトリ)
└── .github/workflows/
    └── ci.yml                        (Turborepo 影響範囲ビルド)
```

### 3.2 ビルド戦略 (Turborepo)

- ルートで `pnpm install && pnpm turbo build` で全 Unit を依存順に並列ビルド
- 影響範囲のみビルド: `pnpm turbo build --filter=...[origin/main]` で変更検知
- TypeScript Project References で型チェックを高速化

### 3.3 依存ルール

- `apps/*` は `packages/*` に依存 OK / 他の `apps/*` には直接依存しない (Lambda 間は API or Async Invoke)
- `packages/*` は他の `packages/*` に依存 OK (例: `audit-logger` → `shared-types`)
- 循環依存禁止 (Turborepo が検出)

### 3.4 共通リソース命名規則

- AWS リソース: `oshi-osare-{env}-{component}` (例: `oshi-osare-dev-praise-lambda`)
- Lambda 環境変数: 各 Unit ごとに必要な値だけを最小権限で渡す
- Secrets: Secrets Manager (NFR-SEC-01 シークレット非ハードコード)

---

## 4. チーム編成と Unit 担当 (5/15 結果発表後に確定)

### 4.1 4 名の想定スキル要件 × Unit 適合度

| 担当者 | Unit-A (Frontend) | Unit-B (Onboarding) | Unit-C (Praise/AI) | Unit-D (DM/AI 並列) | Unit-E (横断/IaC) |
| --- | --- | --- | --- | --- | --- |
| Miura | TBD | TBD | TBD | TBD | TBD |
| Ito | TBD | TBD | TBD | TBD | TBD |
| Matsumoto | TBD | TBD | TBD | TBD | TBD |
| Kawano | TBD | TBD | TBD | TBD | TBD |

→ 5/15 書類審査結果発表後、上記表を埋めて確定する。**1 人 1 Unit + 1 人が横断 (Unit-E) を担当** または **Unit-E はチーム全員でローテーション** のいずれかで運用。

### 4.2 各 Unit のスキル要件サマリ (5/15 担当決め時のチェックリスト)

| Unit | 必須スキル | あれば望ましい |
| --- | --- | --- |
| Unit-A | React + TS / CSS (レトロ UI) | Amplify Gen 2 / SSE クライアント |
| Unit-B | TypeScript / DynamoDB / Lambda | API Gateway / NFR-PERF |
| Unit-C | TypeScript / Bedrock streaming / SSE | プロンプトエンジニアリング |
| Unit-D | TypeScript / Bedrock 並列 / EventBridge / 状態機械設計 | プロンプトエンジニアリング |
| Unit-E | TypeScript / AWS CDK / IAM / Bedrock Guardrails | セキュリティ / 倫理ルール設計 |

### 4.3 Unit-RM (Roadmap) の扱い

5/30 予選会の MVP には含めない。書類審査 / 予選プレゼンでは「拡張余地として描く」のみ。

---

## 5. 開発スケジュール (5/15〜5/30)

### 5.1 マイルストーン

| 日付 | マイルストーン |
| --- | --- |
| 2026-05-15 | 書類審査結果発表 → 4 名の Unit 担当確定 |
| 2026-05-15 | Unit-E (CDK / Cognito / Bedrock Guardrails) を最初に整備 → 開発環境セットアップ |
| 2026-05-16〜2026-05-20 | Unit-A / B / C / D を並行実装 (per Unit Functional Design + NFR Requirements + NFR Design + Infrastructure Design + Code Generation) |
| 2026-05-21〜2026-05-26 | 統合テスト + プロンプトテスト + 倫理ガードレール検証 |
| 2026-05-27〜2026-05-29 | デモ台本動作確認 + バグ修正 + 5/30 当日リハーサル |
| 2026-05-30 | 予選会 @麻布台ヒルズ AWS オフィス |
| 2026-06-26 | 決勝 @幕張メッセ AWS Summit (通過した場合) — Unit-RM 着手検討 |

### 5.2 Per-Unit Loop (CONSTRUCTION フェーズ)

各 Unit に対し、CONSTRUCTION フェーズで以下 5 ステージを実行 (`execution-plan.md` 準拠):

1. Functional Design (per Unit) — Standard
2. NFR Requirements (per Unit) — Standard
3. NFR Design (per Unit) — Standard
4. Infrastructure Design (per Unit) — Standard
5. Code Generation (Planning + Generation) (per Unit) — Standard

→ 全 Unit 完了後 Build and Test を 1 回実行 (統合テスト + プロンプトテスト + 倫理検証)

---

## 6. NFR Unit 別対応マップ

| NFR | 主担当 Unit |
| --- | --- |
| NFR-PERF-01 (3 秒応答) | Unit-A (SSE クライアント) + Unit-C/D (Bedrock streaming) + Unit-E (Lambda Response Streaming 設定) |
| NFR-PERF-02 (10 同時) | Unit-E (Lambda 同時実行数 + DDB on-demand) |
| NFR-ARCH-01 (Bedrock Serverless) | Unit-C/D (Bedrock 利用) + Unit-E (CDK) |
| NFR-SEC-01〜03 | Unit-E (IAM / Cognito / Secrets Manager) + 各 Unit (入力検証) |
| NFR-ETH-01〜08 | Unit-E (PromptGuardrail / 共通プロンプト断片) + Unit-C/D (各 AI Adapter) |
| NFR-A11Y-01, 02 | Unit-A (日本語固定 + レトロ CSS) |
| NFR-OBS-01, 02 | Unit-E (AuditLogger / AuditLogStore) + 各 AI Adapter (logAIInteraction) |
| NFR-QA-02 (プロンプトテスト) | Unit-C/D (各 AI Agent System Prompt のテスト) |
| NFR-QA-03 (TDD Red→Green→Refactor) | 全 Unit |
