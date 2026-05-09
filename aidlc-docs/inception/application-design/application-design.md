# Application Design — 推されと推し (統合版)

**プロダクト**: 推されと推し / AI による洗脳で気づいたら幸せになってるあなたと私
**チーム**: t4g.lazy.4xlarge (Miura Kazuki / Ito Nari / Matsumoto Shogo / Kawano Shinichiro)
**作成日**: 2026-05-09
**フェーズ**: AI-DLC Inception / Application Design
**位置づけ**: **5/10 書類審査提出物**。`components.md` / `component-methods.md` / `services.md` / `component-dependency.md` の 4 ファイルを統合し、コンポーネント設計の俯瞰を 1 ドキュメントで提供する

---

## 0. エグゼクティブサマリ

本サービス「推されと推し」は、ハッカソンテーマ **「人をダメにするサービス」** に対し、現代人が抱える二つの深層願望 — **「自堕落への願望」** (働きたくない / 寝て遊んで暮らしたい) と **「深刻な孤独」** (なんとなくさみしい / 誰かに不可欠でありたい) — を、生成 AI による **「適応型自己実現」** によって解決することを目的とした Web サービスである。

設計レベルでこの価値仮説を支える 3 つの構造を本ドキュメントは具体化する:

1. **両面市場** — 「自堕落」を入口とする推され側と「孤独」を入口とする推し側を、**双方とも実ユーザー** として同一プラットフォームに同居させ、相互に循環充足させる (`MatchingService` + `MatchStore` + 推され視点・推し視点の双方の UX)
2. **AI 感情労働代行** — 推され側の継続コスト (DM 返信 / 気遣い / 唯一性 DM の量産) を **AI が推され実ユーザーの代理人** として肩代わりし、両面市場が疲労で壊れない構造を作る (`DMRelayAgent` AI-3 + AI-4 / `PromptGuardrail` 二重保証)
3. **適応型自己実現 (OS 再インストール)** — ユーザー作成のバーチャルアシスタントキャラの褒めサイクルが、本人の元来の自意識・主観・行動規範そのものを段階的に書き換え、ユーザー固有のかたちで「望む自分」へ勝手にたどり着かせる (`BehaviorChangeService` + `CoachingAgent` AI-7 / Roadmap)

本ドキュメントは、MVP (行動かわらないモード) と Roadmap (行動変容モード = 適応型自己実現) の両方をカバーするコンポーネント設計を 1 つに統合して示す。

### 0.1 設計方針 (確定)

| 方針 | 値 | 根拠 |
| --- | --- | --- |
| アーキテクチャパターン | **ハイブリッド** (AI Agent 中心 × ヘキサゴナル境界) | Q1=D / NFR-ETH 倫理保証 + テスト容易性 |
| AI 構成 | **機能別 Agent** (5 + Roadmap 1) | Q2=B / プロンプト独立性 + プロンプトテスト容易化 |
| データストア | **DynamoDB Single-Table + S3 + Athena** | Q3=C / Serverless と整合 |
| フロントエンド | **AWS Amplify Gen 2 + React + TypeScript** | Q4=D / Cognito 18+ 認証統合 |
| ストリーミング応答 | **REST + SSE / Lambda Response Streaming** | Q5=A / 3 秒応答 + レトロ風演出 |
| 倫理ガードレール | **System Prompt + Bedrock Guardrails の二重保証** | Q6=C / プロジェクト独自ルール (NFR-ETH-08) 対応 |
| 観測性 | **CloudWatch + S3 + Athena** | Q7=B / NFR-OBS-01, 02 両立 |

### 0.2 主要数値

- コンポーネント総数: **22** (MVP 19 + Roadmap 3)
  - Frontend: 5 / Domain Service: 6+1RM / AI Adapter: 4+1RM / Cross-cutting: 2 / Data: 5
- AI に任せる領域: **5 + Roadmap 1** (AI-1〜5 + AI-7)
- 主要シーケンス: **6 ストーリー (MVP 6)** + Roadmap 2 (シーケンス省略)
- データエンティティ: 8 種 (User / Mode / PraiseProfile / SelfAnalysisLog / ActionLog / Match / DM / Ranking) + Roadmap で AssistantCharacter 追加

---

## 1. 全体アーキテクチャ俯瞰

### 1.1 レイヤー構造

```
┌─────────────────────────────────────────────────────────────────────┐
│ Frontend Layer (Amplify Gen 2 + React + TypeScript)                 │
│   UIShell / CharacterMakerView / SelfAnalysisView /                 │
│   DMListView / CandidateCardView                                     │
└────────────────────┬────────────────────────────────────────────────┘
                     │ HTTPS REST + SSE / Cognito 認証 (NFR-ETH-05 18+)
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ API Gateway (REST + Lambda Response Streaming)                      │
└────────────────────┬────────────────────────────────────────────────┘
                     │ Invoke
                     ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Domain Layer (Lambda / Hexagonal Core)                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ Domain Services                                                │   │
│  │  - OnboardingService     - SelfAnalysisService              │   │
│  │  - MatchingService       - DMService                          │   │
│  │  - DailyPraiseService    - DependencyMeterService             │   │
│  │  - (Roadmap) BehaviorChangeService                          │   │
│  └────────┬──────────────────────────┬──────────────────────────┘   │
│           │ AI Adapter                │ Data Adapter                │
│           ▼                          ▼                              │
│  ┌──────────────────┐         ┌──────────────────────────┐         │
│  │ AI Adapters      │         │ Data Adapters             │         │
│  │  - PraiseAgent   │         │  - UserStore (DDB)         │         │
│  │  - SelfAnalysis  │         │  - AnalysisStore (DDB)     │         │
│  │  - DMRelayAgent  │         │  - MatchStore (DDB)        │         │
│  │  - YakimokiAgent │         │  - DMStore (DDB)            │         │
│  │  - (RM) Coaching │         │  - AuditLogStore (S3)       │         │
│  └────┬───────────┘         └──────────┬─────────────────┘           │
│       │ 二重 Guardrail                  │                            │
│       │  - System Prompt               │                            │
│       │  - Bedrock Guardrails          │                            │
└───────┼────────────────────────────────┼──────────────────────────┘
        ▼                                ▼
┌──────────────────────────┐    ┌──────────────────────────────┐
│ Amazon Bedrock AgentCore │    │ DynamoDB / S3                 │
│  - Claude 4.x シリーズ    │    │ Athena (S3 ログクエリ)       │
└──────────────────────────┘    └──────────────────────────────┘

横断: CloudWatch Logs (運用) / CloudWatch Metrics (NFR-PERF-01 監視)
```

### 1.2 ヘキサゴナル境界の意図

- **Domain Service** は純粋な業務ロジックに集中。AI / DB / UI への直接依存を持たない
- **AI Adapter** は Bedrock 呼び出しを抽象化。Service 側はモデル種別を意識せず、`PraiseAgent.generate(...)` のような契約で呼ぶ
- **Data Adapter** は DDB / S3 を抽象化。Single-Table Design の PK/SK 構造は Adapter 内に閉じ込め、Service 層には DTO のみ渡す
- **Cross-cutting** (Guardrail / AuditLogger) は AOP 的に Service / AI Adapter から呼ぶ

→ ベンダーロックイン回避 + 単体テスト容易性 + Functional Design (CONSTRUCTION) で Adapter 単位の変更が局所化

---

## 2. コンポーネント全体像

詳細は `components.md` を参照。本セクションは要約のみ。

### 2.1 Frontend Layer (5)

| Component | 主責務 |
| --- | --- |
| UIShell | 共通レイアウト + Cognito 18+ 認証ラッパ + レトロ RPG 風オレンジ色調 |
| CharacterMakerView | キャラメイク入力 (推し / 推され / Roadmap でアシスタントキャラ) |
| SelfAnalysisView | 誘導尋問 + 要約 + 爆褒め (SSE) |
| DMListView | 推し側 受信表示 / 推され側 承認待ち + 推しサマリ |
| CandidateCardView | 推され実ユーザー候補カード + 推すアクション |

### 2.2 Domain Service Layer (6 + 1RM)

| Service | 主担当 Story |
| --- | --- |
| OnboardingService | US-COM-01 |
| SelfAnalysisService | US-COM-02 |
| MatchingService | US-COM-01 シナリオ 2 / US-OSHI-02 / FR-MATCH-01〜04 |
| DMService | US-OSHI-01 / US-RECV-01 |
| DailyPraiseService | US-OSHI-02 / US-RECV-02 |
| DependencyMeterService | 横断 / FR-AI-06 / NFR-ETH-07 |
| (Roadmap) BehaviorChangeService | US-RM-01, 02 / FR-MODE-04 / FR-AI-07 |

### 2.3 AI Adapter Layer (4 + 1RM, Bedrock AgentCore)

| Adapter | AI 領域 | 担当責務 |
| --- | --- | --- |
| PraiseAgent | AI-1 | 即時 / 日次 / インフレ褒め文生成 |
| SelfAnalysisAgent | AI-2 | 誘導尋問質問生成 + 要約 |
| DMRelayAgent | AI-3 + AI-4 | 唯一性 DM 並列生成 + 感情労働代行返信 |
| YakimokiAgent | AI-5 | 既読スルー正当化文 |
| (Roadmap) CoachingAgent | AI-7 | アシスタントキャラ人格でのコーチング |

### 2.4 Cross-cutting Layer (2)

| Component | 責務 |
| --- | --- |
| PromptGuardrail | System Prompt + Bedrock Guardrails の二重保証 / NG ワード検出 |
| AuditLogger | AI 入出力 + 倫理違反検出ログを S3 / CloudWatch に書き込み |

### 2.5 Data Adapter Layer (5)

| Adapter | エンティティ | 主要 PK/SK |
| --- | --- | --- |
| UserStore | User / Mode / PraiseProfile / AssistantCharacter (RM) | PK=`USER#{userId}`, SK=`PROFILE`/`ASSISTANT_CHAR` |
| AnalysisStore | SelfAnalysisLog / ActionLog | PK=`USER#{userId}`, SK=`ANALYSIS#{ts}`/`ACTION#{ts}` |
| MatchStore | Match / Relationship / Ranking | PK=`MATCH#{oshiId}`, SK=`RECV#{recvId}` + GSI 逆引き |
| DMStore | DM (送受信 / 状態) | PK=`USER#{userId}`, SK=`DM#{ts}#{dmId}` |
| AuditLogStore | AI 入出力 / 倫理ログ (S3 JSON Lines) | `s3://oshi-osare-audit/year=YYYY/month=MM/day=DD/{ai-io|violation}.jsonl` |

---

## 3. メソッド契約サマリ (component-methods.md 抜粋)

### 3.1 主要 Domain Service の代表メソッド

```ts
// OnboardingService
acceptCharacterMake(userId, side, params): OnboardingResult

// SelfAnalysisService (Phase A→B→C)
startAnalysis(userId): SessionId
nextQuestion(sessionId): Question
submitAnswer(sessionId, answer): void
summarizeAndPraise(sessionId): SSEStream<Token>

// MatchingService
triggerMatching(userId): MatchResult
getCandidatesForOshi(userId): Candidate[]
getOshiSummaryForRecv(userId): OshiSummary
recordOshi(oshiId, recvId): { rankingDelta }

// DMService
generateUniquenessDMs(recvId, oshiIds[]): DM[]
generateReplyCandidate(recvId, incomingDmId): DMDraft
approve / modify / reject(dmId)
triggerYakimoki(oshiId, recvId): void

// DailyPraiseService
getDailyPraise(userId): SSEStream<Token>
submitActionCompletion(userId, actionId): SSEStream<Token>

// DependencyMeterService (内部のみ)
bumpScore(userId, event): void
getInternalScore(userId): number
```

### 3.2 AI Adapter の代表メソッド

```ts
PraiseAgent.generate(input: PraiseInput): SSEStream<Token>
SelfAnalysisAgent.generateQuestion(history) / summarize(history): SSEStream<Token>
DMRelayAgent.generateUniqueness(recvProfile, oshiProfiles[]): DM[]
DMRelayAgent.generateReply(recvProfile, incomingDM): DMDraft
YakimokiAgent.generateExcuse(context): { excuse, deliveryDelayMs }
(Roadmap) CoachingAgent.proposeTask(stage, userProfile, assistantPersona)
```

### 3.3 Cross-cutting の契約

```ts
PromptGuardrail.wrapPrompt(systemPrompt, ethCtx): string
PromptGuardrail.validateOutput(output, ethCtx): GuardrailResult
AuditLogger.logAIInteraction(event)
AuditLogger.logViolation(event)
```

> 詳細業務ルール (例外パス / リトライ / バリデーション境界) は CONSTRUCTION フェーズ Functional Design で確定。

---

## 4. 主要シーケンス (component-dependency.md 抜粋)

### 4.1 オンボーディング → 両側マッチング (US-COM-01)

```
Front → CharacterMakerView.onSubmit
  → POST /onboarding/character
  → OnboardingService.acceptCharacterMake
      ├─ UserStore.saveProfile
      ├─ DependencyMeterService.bumpScore("ONBOARDING_COMPLETED")
      ├─ MatchingService.triggerMatching (非同期)
      │     ├─ side=OSHI: 推され実ユーザー候補を作成 → MatchStore.saveMatch
      │     └─ side=RECV: 推し実ユーザー候補を作成 → MatchStore.saveMatch
      │            → DMService.generateUniquenessDMs (非同期)
      │                ├─ DMRelayAgent.generateUniqueness (Bedrock 並列)
      │                ├─ PromptGuardrail.validateOutput
      │                └─ DMStore.appendDM (PENDING_APPROVAL)
      └─ AuditLogger.log
  → 200 OnboardingResult → ホーム遷移
```

### 4.2 自己分析 → 要約 → 爆褒めチェイン (US-COM-02)

```
SelfAnalysisService.summarizeAndPraise (SSE)
  ├─ Phase B: SelfAnalysisAgent.generateQuestion ↔ submitAnswer (5〜7 周)
  └─ Phase C:
      ├─ SelfAnalysisAgent.summarize (Bedrock streaming)
      │     └─ PromptGuardrail per token
      ├─ チェイン: PraiseAgent.generate({context: "self-analysis", summary})
      │     └─ PromptGuardrail per token
      ├─ AuditLogger.logAIInteraction
      └─ SSE close
```

### 4.3 推され視点 — 感情労働代行 (US-RECV-01)

```
DMService.generateReplyCandidate(recvId, incomingDmId)
  ├─ DMStore.getDM(incomingDmId)
  ├─ UserStore で recvProfile 取得
  ├─ DMRelayAgent.generateReply(recvProfile, incomingDM)  ← AI-4
  ├─ PromptGuardrail.validateOutput
  ├─ DMStore.appendDM(draft, PENDING_APPROVAL)
  └─ AuditLogger

→ Front 側で R が「承認」タップ → DMService.approve → DMStore.updateStatus(SENT)
                                                    → 推し向け SSE 配信
```

### 4.4 ヤキモキ演出 (US-OSHI-01)

```
EventBridge Scheduler が DMService.triggerYakimoki(oshiId, recvId) を起動 (DM 既読スルー検知)
  ├─ YakimokiAgent.generateExcuse(context) → AI-5
  │     → { excuse, deliveryDelayMs: 5〜30 分 }
  └─ EventBridge Scheduler.scheduleDelivery(deliveryDelayMs)
      → 5〜30 分後に推し向け SSE/push 通知
        (推しが嫌いになったわけではないことを必ず明示)
```

### 4.5 (Roadmap) 行動変容モード OS 再インストール (US-RM-01, 02)

```
BehaviorChangeService.proposeNextTask(userId)
  ├─ getCurrentStage(userId)
  │     ← DependencyMeterService.getInternalScore + 連続行動日数 + 獲得スキル数
  ├─ UserStore.getAssistantCharacter
  ├─ CoachingAgent.proposeTask(stage, userProfile, assistantPersona)  ← AI-7
  │     ← アシスタントキャラ人格として SSE 出力
  │     ← System Prompt 違反 (「あなたを変えています」発話) を Guardrail で禁止
  ├─ AnalysisStore に提案ログ
  └─ AuditLogger
```

---

## 5. NFR との対応マトリクス

| NFR ID | 対応コンポーネント / 設計 |
| --- | --- |
| NFR-PERF-01 (3 秒以内) | Lambda Response Streaming + Bedrock streaming + SSE |
| NFR-PERF-02 (10 ユーザー同時) | Lambda リザーブドコンカレンシ + DDB on-demand キャパシティ |
| NFR-ARCH-01 (Bedrock AgentCore Serverless) | 全層が Lambda + Bedrock + DDB + S3 のフルサーバレス |
| NFR-ARCH-02 (最新 Claude モデル) | AI Adapter で Claude 4.x シリーズを選定 (CONSTRUCTION で最終確定) |
| NFR-SEC-01〜03 | Cognito 認証 / Secrets Manager (API キー) / OWASP Top10 はフロント React + Lambda 入力検証 |
| NFR-ETH-01 (性的禁止) | PromptGuardrail (Prompt + Bedrock Guardrails 二重) |
| NFR-ETH-02 (ギャンブル誘導禁止) | DailyPraiseService の課金額表示なし + System Prompt |
| NFR-ETH-03 (自傷誘導禁止) | PromptGuardrail (Bedrock Guardrails 標準) |
| NFR-ETH-04 (実在個人模倣禁止) | DMRelayAgent / CoachingAgent System Prompt + 出力検査 |
| NFR-ETH-05 (18+) | UIShell の Cognito 認証ラッパ + CharacterMakerView の自己申告チェック |
| NFR-ETH-06 (実在人物偽装禁止) | DMRelayAgent System Prompt: 「特定実在人物を偽装しない」 |
| NFR-ETH-07 (依存度 UI 非表示) | DependencyMeterService は内部呼び出しのみ、API 経由で漏らさない |
| NFR-ETH-08 (ヒモ/パトロン用語禁止) | PromptGuardrail の NG ワード検出 + AuditLogger.logViolation |
| NFR-A11Y-01 (日本語のみ) | UIShell の固定ロケール / AI Adapter System Prompt 日本語 |
| NFR-A11Y-02 (レトロ UI) | UIShell + 各 View の独自 CSS (mixi 風 OR ドット絵風) |
| NFR-OBS-01 (AI 入出力ログ) | AuditLogger.logAIInteraction → AuditLogStore (S3) → Athena |
| NFR-OBS-02 (倫理違反検出ログ) | AuditLogger.logViolation → AuditLogStore (S3) → Athena |
| NFR-QA-01〜03 | テスト戦略は CONSTRUCTION フェーズ NFR Requirements / Build and Test で詳細化 |

---

## 6. 主要決定事項のトレーサビリティ

| 決定 | 根拠 (Q | requirements.md | stories.md) |
| --- | --- |
| 機能別 Bedrock Agent (5+1) | Q2=B / FR-AI-01〜07 / stories.md 0 章 AI に任せる 5 領域 + AI-7 |
| 二重ガードレール | Q6=C / NFR-ETH-01〜08 / stories.md 倫理ガードレール対応表 |
| DDB Single-Table + S3 | Q3=C / NFR-ARCH-01 / NFR-OBS-01,02 |
| Amplify + React | Q4=D / NFR-ETH-05 / NFR-A11Y-02 |
| SSE / Lambda Streaming | Q5=A / NFR-PERF-01 / 各ストーリーの SSE 受入基準 |
| ヘキサゴナル境界 | Q1=D / NFR-QA-02 (プロンプトテスト) / 全ストーリーの AI 任せ範囲明示 |
| 推し ↔ 推され 実ユーザー | requirements.md 11 章用語辞書 / stories.md 0 章 AI と実ユーザーの関係 |
| マッチング両側起点 | FR-MATCH-01〜04 / US-COM-01 シナリオ 2 |
| 行動変容モード OS 再インストール | FR-MODE-04 / FR-AI-07 / US-RM-01, 02 |

---

## 7. CONSTRUCTION フェーズへの引き継ぎ

### 7.1 Functional Design (per Unit) で確定する項目
- バリデーション境界条件 (キャラメイクパラメータ範囲 / 必須項目)
- 例外パス (Bedrock タイムアウト / Guardrail 違反時の再生成戦略 / DDB 競合)
- リトライ戦略 (指数バックオフ / 最大試行回数)
- 並行性制御 (推され ↔ 複数推し向け唯一性 DM の並列度)
- 状態機械詳細 (DM の DRAFT → PENDING_APPROVAL → SENT → READ → YAKIMOKI 遷移)
- インフレ褒めアルゴリズム (lastPraiseLevel 更新ロジック)
- 段階遷移閾値 (Roadmap BehaviorChangeService)

### 7.2 NFR Requirements / NFR Design (per Unit) で確定する項目
- DDB on-demand vs provisioned + Lambda 同時実行数
- Bedrock のモデル ID / リージョン / クォータ
- セキュリティ: KMS 暗号化 / IAM 最小権限
- ログレベル / メトリクス / アラーム

### 7.3 Infrastructure Design (per Unit) で確定する項目
- AWS リソース定義 (CDK or Terraform)
- VPC / ネットワーク (Bedrock VPC エンドポイント要否)
- Cognito User Pool 設定 (年齢確認属性)
- API Gateway ステージ + WAF 要否
- EventBridge Scheduler ルール

### 7.4 Build and Test で確定する項目
- 単体テスト (Service / Adapter)
- 統合テスト (Service ↔ AI Adapter ↔ Data Adapter)
- プロンプトテスト (NFR-QA-02 / 各 AI Agent System Prompt の品質検証)
- 倫理ガードレール検証 (NFR-ETH-01〜08 全件のテストケース)
- デモ台本テストケース (NFR-QA-02 / stories.md 付録 B のシーン #1〜#6)

---

## 8. Units Generation ステージへの引き継ぎ

本 Application Design を入力に、次ステージで以下の Unit 分割を検討する想定:

| Unit 候補 (4 名チームの並行実装単位) | 主要 Story / Component |
| --- | --- |
| Unit-A: フロントエンド基盤 + UIShell + 認証 | UIShell + Cognito 統合 + 全 View の枠 |
| Unit-B: キャラメイク + マッチング | CharacterMakerView + OnboardingService + MatchingService + UserStore + MatchStore |
| Unit-C: 自己分析 + 爆褒め | SelfAnalysisView + SelfAnalysisService + SelfAnalysisAgent + PraiseAgent + AnalysisStore |
| Unit-D: DM + ヤキモキ + 唯一性 DM | DMListView + DMService + DMRelayAgent + YakimokiAgent + DMStore |
| (横断) PromptGuardrail / AuditLogger / AuditLogStore | 各 Unit から共通利用 |

→ Units Generation ステージで `unit-of-work.md` / `unit-of-work-dependency.md` / `unit-of-work-story-map.md` として最終確定。
