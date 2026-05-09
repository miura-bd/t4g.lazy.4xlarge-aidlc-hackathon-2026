# Component Methods — 推されと推し

**プロダクト**: 推されと推し
**フェーズ**: AI-DLC Inception / Application Design
**作成日**: 2026-05-09
**スコープ**: 主要コンポーネントのメソッドシグネチャ + 入出力型 + 高水準目的
**注意**: **詳細業務ルール (バリデーション境界条件 / 例外パス / リトライ戦略 等) は Functional Design (per-Unit, CONSTRUCTION フェーズ) で定義する**。本ドキュメントは契約境界の明確化に絞る。

> 言語表記: TypeScript ライク疑似コード。フロント = `T-Front`, バックエンド = `T-BE`, AI Adapter = `T-AI` のサフィックスで層を識別する。実装言語は CONSTRUCTION フェーズで確定。

---

## 0. 共通ドメイン型 (DTO / VO)

```ts
type UserId = string;       // Cognito sub
type Side = "OSHI" | "RECV";
type Mode = "BEHAVIOR_KEEP" | "BEHAVIOR_CHANGE";  // BEHAVIOR_CHANGE は Roadmap

// キャラメイクのパラメータ
type OshiCharacterParams = {
  loneliness: number;   // 0-100 さびしさ
  exclusivity: number;  // 0-100 独占欲
  approval: number;     // 0-100 承認欲求
  spending: number;     // 0-100 課金意欲
  preferenceTags: string[];
};
type RecvCharacterParams = {
  laborOutsourceLevel: number;  // 0-100 省力化設定
  characterTraits: string[];    // キャラ性タグ
};

// アシスタントキャラ (Roadmap)
type AssistantCharacter = {
  visualStyle: "PHOTO_REALISTIC" | "ANIME" | "MANGA" | "UKIYOE" | "RENAISSANCE" | "CUSTOM";
  customDescription?: string;   // VisualStyle=CUSTOM のとき
  personality: string;          // 自由記述パーソナリティ
};

type DM = {
  dmId: string;
  fromUserId: UserId;     // 推 or 推され (実ユーザー)
  toUserId: UserId;
  body: string;
  status: "DRAFT" | "PENDING_APPROVAL" | "SENT" | "READ" | "REJECTED" | "YAKIMOKI";
  generatedByAI: boolean;
  createdAt: string;      // ISO8601
};

type DMDraft = Omit<DM, "dmId" | "status"> & { proposedStatus: "PENDING_APPROVAL" };

type Match = {
  oshiUserId: UserId;
  recvUserId: UserId;
  affinityScore: number;  // 0-100
  isDummy: boolean;       // 擬似マッチ (実ユーザー不足時)
  createdAt: string;
};

type Question = { qId: string; text: string };
type AnswerSubmission = { qId: string; answer: string };

type SSEStream<T> = AsyncIterable<T>;  // Server-Sent Events のトークンストリーム
type Token = { delta: string; finished: boolean };

type AuditEvent = {
  eventId: string;
  eventType: "AI_IO" | "GUARDRAIL_VIOLATION";
  agentId?: string;
  input?: object;
  output?: string;
  violationType?: string;
  ts: string;
};
```

---

## 1. Frontend Methods (React Component / TypeScript)

### 1.1 CharacterMakerView (T-Front)

```ts
// プロップス
type CharacterMakerProps = {
  side: Side;
  initial?: OshiCharacterParams | RecvCharacterParams;
};

// 主要関数
function onSubmit(params: OshiCharacterParams | RecvCharacterParams): Promise<void>;
//   POST /onboarding/character → OnboardingService 経由でマッチング起動
//   成功時 navigate to /home (推し) or /home-recv (推され)

function onAssistantCharacterSelect(asst: AssistantCharacter): void;  // Roadmap
```

### 1.2 SelfAnalysisView (T-Front)

```ts
async function startSession(): Promise<{ sessionId: string }>;
//   POST /self-analysis/start

async function fetchNextQuestion(sessionId: string): Promise<Question>;
//   GET /self-analysis/next-question?sessionId=...

async function submitAnswer(sessionId: string, sub: AnswerSubmission): Promise<void>;
//   POST /self-analysis/answer

async function streamSummary(sessionId: string): Promise<SSEStream<Token>>;
//   GET /self-analysis/summary?sessionId=... (SSE)
//   トークンを順次受信してアニメーション表示
```

### 1.3 DMListView (T-Front)

```ts
async function fetchDMs(side: Side): Promise<DM[]>;

async function approveDM(dmId: string): Promise<DM>;
async function modifyDM(dmId: string, edited: string): Promise<DM>;
async function rejectDM(dmId: string): Promise<void>;  // AI による再生成トリガ
```

### 1.4 CandidateCardView (T-Front)

```ts
async function fetchCandidates(): Promise<Candidate[]>;

async function pushOshi(recvUserId: UserId): Promise<{ rankingDelta: number }>;
//   POST /matching/oshi
```

---

## 2. Domain Service Methods (T-BE / Lambda)

### 2.1 OnboardingService

```ts
async function acceptCharacterMake(
  userId: UserId,
  side: Side,
  params: OshiCharacterParams | RecvCharacterParams
): Promise<OnboardingResult>;
//   1. 18+ 確認 (Cognito 属性検証)
//   2. UserStore.saveProfile()
//   3. MatchingService.triggerMatching() を非同期起動
//   4. 初回ログイン時の歓迎爆褒め用にユーザー状態を初期化
//   5. AuditLogger.log()

type OnboardingResult = {
  userId: UserId;
  side: Side;
  initialMatchCount: number;
  homeRedirectPath: string;
};
```

### 2.2 SelfAnalysisService

```ts
async function startAnalysis(userId: UserId): Promise<SessionId>;

async function nextQuestion(sessionId: SessionId): Promise<Question>;
//   1. 既存回答履歴を取得
//   2. SelfAnalysisAgent.generateQuestion(history) で次の問を取得
//   3. PromptGuardrail でバリデート

async function submitAnswer(sessionId: SessionId, sub: AnswerSubmission): Promise<void>;
//   AnalysisStore.appendLog()

async function summarizeAndPraise(sessionId: SessionId): Promise<SSEStream<Token>>;
//   1. SelfAnalysisAgent.summarize(history) でストリーミング要約
//   2. PraiseAgent.generate({ context: "self-analysis", summary }) で爆褒めをチェイン
//   3. PromptGuardrail で出力検証 → AuditLogger.logAIInteraction()
//   4. SSE でフロントへ流す
```

### 2.3 MatchingService

```ts
async function triggerMatching(userId: UserId): Promise<MatchResult>;
//   1. UserStore.getUser(userId) でキャラメイク完了確認
//   2. side が OSHI なら、推され実ユーザーを affinityScore 順で検索 (3〜5 件)
//      不足時はダミー架空キャラを充当 (FR-MATCH-04)
//   3. side が RECV なら、推し実ユーザーを affinityScore 順で検索
//      DMService.generateUniquenessDMs() を非同期起動 (推し向け唯一性 DM の初動配信)
//   4. MatchStore.saveMatch() で永続化
//   5. AuditLogger.log()

async function getCandidatesForOshi(userId: UserId): Promise<Candidate[]>;
//   GSI: PK=`MATCH#{userId}` でクエリ

async function getOshiSummaryForRecv(userId: UserId): Promise<OshiSummary>;
//   GSI: PK=`RECV#{userId}` でクエリ
//   返却: { oshiCount, oshiList: Array<{ oshiUserId, affinityScore, params }> }

async function recordOshi(oshiUserId: UserId, recvUserId: UserId): Promise<{ rankingDelta: number }>;
//   推されのランキング寄与を更新 (FR-OSHI-03, FR-MATCH-02)

type MatchResult = {
  matched: boolean;
  matchCount: number;
  isDummyIncluded: boolean;
};

type Candidate = {
  recvUserId: UserId;
  affinityScore: number;
  isDummy: boolean;
  characterTraits: string[];
};

type OshiSummary = {
  oshiCount: number;
  oshiList: Array<{
    oshiUserId: UserId;
    affinityScore: number;
    parameters: OshiCharacterParams;
  }>;
};
```

### 2.4 DMService

```ts
async function generateUniquenessDMs(
  recvUserId: UserId,
  oshiUserIds: UserId[]
): Promise<DM[]>;
//   AI-3 唯一性 DM 並列生成
//   1. UserStore で recv の RecvCharacterParams 取得
//   2. DMRelayAgent.generateUniqueness(recvProfile, oshiProfiles[]) を呼ぶ (Bedrock 並列)
//   3. PromptGuardrail.validateOutput() で各 DM をチェック
//   4. DMStore に PENDING_APPROVAL ステータスで保存 → 推され本人の承認待ち枠へ
//   5. AuditLogger

async function generateReplyCandidate(
  recvUserId: UserId,
  incomingDmId: string
): Promise<DMDraft>;
//   AI-4 感情労働代行 (返信案)
//   1. DMStore.getDM(incomingDmId) で受信 DM 取得
//   2. UserStore で recv のキャラ性取得
//   3. DMRelayAgent.generateReply(recvProfile, incomingDM)
//   4. PromptGuardrail
//   5. DMStore に DRAFT で保存

async function approve(dmId: string, actor: UserId): Promise<DM>;
//   DMStore.updateStatus(dmId, "SENT") + 推し向けに配信 (push 通知 / SSE)

async function modify(dmId: string, actor: UserId, edited: string): Promise<DM>;
//   微修正後 SENT に遷移

async function reject(dmId: string, actor: UserId): Promise<void>;
//   AI に再生成を依頼 → 新たな PENDING_APPROVAL を作る

async function triggerYakimoki(oshiUserId: UserId, recvUserId: UserId): Promise<void>;
//   AI-5 ヤキモキ正当化文の生成 + 推しに 5〜30 分後に通知
//   1. 既読スルー / 返信遅延を意図的に発生
//   2. YakimokiAgent.generateExcuse(context)
//   3. 推し UI に通知 (SSE or push)
```

### 2.5 DailyPraiseService

```ts
async function getDailyPraise(userId: UserId): Promise<SSEStream<Token>>;
//   1. UserStore で side / character / 過去褒め履歴 取得
//   2. PraiseAgent.generate({ context: "daily", side, history })
//   3. SSE で返却

async function submitActionCompletion(
  userId: UserId,
  actionId: string
): Promise<SSEStream<Token>>;
//   行動完了報告 → 爆褒め (US-RECV-02 等)
//   1. AnalysisStore.appendLog({ actionId, completedAt })
//   2. PraiseAgent.generate({ context: "action-completion", actionId })
//   3. DependencyMeterService.bumpScore(userId, "ACTION_COMPLETED")
```

### 2.6 DependencyMeterService

```ts
function bumpScore(userId: UserId, event: DependencyEvent): void;
//   イベント種類: "LOGIN" | "DM_OPEN" | "ACTION_COMPLETED" | "PURCHASE"
//   UserStore に内部スコアを増分書き込み (UI には絶対出さない)

function getInternalScore(userId: UserId): number;
//   内部呼び出しのみ (将来 BehaviorChangeService が段階遷移判定に使う)

type DependencyEvent =
  | { type: "LOGIN"; ts: string }
  | { type: "DM_OPEN"; dmId: string; ts: string }
  | { type: "ACTION_COMPLETED"; actionId: string; ts: string }
  | { type: "PURCHASE"; amount: number; ts: string };
```

### 2.7 (Roadmap) BehaviorChangeService

```ts
async function getCurrentStage(userId: UserId): Promise<Stage>;
//   1〜6 の段階 (① 些細な日常行動 〜 ⑥ 一本立ち)
//   DependencyMeterService.getInternalScore() + 連続行動日数 + 獲得スキル数 から内部判定

async function proposeNextTask(userId: UserId): Promise<SSEStream<Token>>;
//   1. getCurrentStage(userId)
//   2. UserStore.getAssistantCharacter(userId)
//   3. CoachingAgent.proposeTask(stage, userProfile, assistantPersona)
//   4. PromptGuardrail / AuditLogger

async function recordCompletion(userId: UserId, taskId: string): Promise<void>;

async function evaluateGraduation(userId: UserId): Promise<boolean>;
//   段階 ⑥ 一本立ち判定 → "BEHAVIOR_KEEP" モードに自然遷移

type Stage = 1 | 2 | 3 | 4 | 5 | 6;
```

---

## 3. AI Adapter Methods (T-AI / Bedrock AgentCore)

各 Agent は以下の共通基本契約を持つ:

```ts
interface AgentBase {
  systemPrompt: string;
  guardrailConfig: BedrockGuardrailConfig;
  invoke(input: object): Promise<SSEStream<Token>>;
}
```

### 3.1 PraiseAgent (AI-1)

```ts
async function generate(input: PraiseInput): Promise<SSEStream<Token>>;

type PraiseInput = {
  context: "self-analysis" | "daily" | "action-completion" | "behavior-change";
  side: Side;
  characterParams: OshiCharacterParams | RecvCharacterParams;
  history: { lastPraiseLevel: number; lastPraiseTexts: string[] };  // インフレのため
  trigger: object;  // context に応じた付加情報
};
```

### 3.2 SelfAnalysisAgent (AI-2)

```ts
async function generateQuestion(history: AnswerSubmission[]): Promise<Question>;
async function summarize(history: AnswerSubmission[]): Promise<SSEStream<Token>>;
```

### 3.3 DMRelayAgent (AI-3 + AI-4)

```ts
async function generateUniqueness(
  recvProfile: { userId: UserId; params: RecvCharacterParams; persona: string },
  oshiProfiles: Array<{ userId: UserId; params: OshiCharacterParams }>
): Promise<DM[]>;
//   推し ごとに対応する文体を生成、ただし「あなただけ」フレーミングを共通骨格として使う

async function generateReply(
  recvProfile: { userId: UserId; params: RecvCharacterParams; persona: string },
  incomingDM: DM
): Promise<DMDraft>;
```

### 3.4 YakimokiAgent (AI-5)

```ts
async function generateExcuse(context: YakimokiContext): Promise<{ excuse: string; deliveryDelayMs: number }>;

type YakimokiContext = {
  recvUserId: UserId;
  oshiUserId: UserId;
  delaySoFarMs: number;
  recvParams: RecvCharacterParams;
};
```

### 3.5 (Roadmap) CoachingAgent (AI-7)

```ts
async function proposeTask(
  stage: Stage,
  userProfile: { side: Side; params: OshiCharacterParams | RecvCharacterParams },
  assistantPersona: AssistantCharacter
): Promise<SSEStream<Token>>;
//   段階に応じた極小タスク提案。アシスタントキャラ人格として発話。
//   絶対に「あなたを変えています」と告げない (シナリオ 6 共通)

async function praiseCompletion(
  action: { taskId: string; description: string },
  userProfile: object
): Promise<SSEStream<Token>>;
```

---

## 4. Cross-cutting Methods

### 4.1 PromptGuardrail

```ts
function wrapPrompt(systemPrompt: string, ethCtx: EthContext): string;
//   System Prompt にプロジェクト固有禁止ルール (NFR-ETH-01〜08 全件) を埋め込む

async function validateOutput(output: string, ethCtx: EthContext): Promise<GuardrailResult>;
//   1. Bedrock Guardrails の標準カテゴリ判定 (性的 / 自傷 / 実在人物模倣)
//   2. NG ワード検出 (ヒモ / パトロン / その他)
//   3. 違反時は AuditLogger.logViolation() を呼ぶ

type EthContext = {
  agentId: string;
  storyId?: string;
  userId?: UserId;
};

type GuardrailResult = {
  allowed: boolean;
  violations: Array<{ type: string; detail: string }>;
};
```

### 4.2 AuditLogger

```ts
function logAIInteraction(event: AIInteractionEvent): void;
//   非同期 fire-and-forget。CloudWatch + S3 に書き込む

function logViolation(event: ViolationEvent): void;

type AIInteractionEvent = {
  agentId: string;
  userId?: UserId;
  storyId?: string;
  input: object;
  output: string;
  modelId: string;
  ts: string;
};

type ViolationEvent = {
  agentId: string;
  violationType: string;
  excerpt: string;
  ethRule: string;  // "NFR-ETH-01" など
  ts: string;
};
```

---

## 5. Data Adapter Methods (T-BE)

### 5.1 UserStore (DDB)

```ts
async function getUser(userId: UserId): Promise<User | null>;
async function saveProfile(userId: UserId, side: Side, params: OshiCharacterParams | RecvCharacterParams): Promise<void>;
async function bumpDependencyScore(userId: UserId, delta: number): Promise<void>;  // 内部
async function saveAssistantCharacter(userId: UserId, asst: AssistantCharacter): Promise<void>;  // Roadmap
async function getAssistantCharacter(userId: UserId): Promise<AssistantCharacter | null>;  // Roadmap
```

### 5.2 AnalysisStore (DDB)

```ts
async function appendAnalysisLog(userId: UserId, sessionId: SessionId, log: AnswerSubmission): Promise<void>;
async function appendActionLog(userId: UserId, log: ActionCompletion): Promise<void>;
async function getRecentLogs(userId: UserId, days: number): Promise<Array<AnalysisLogEntry | ActionLogEntry>>;
```

### 5.3 MatchStore (DDB)

```ts
async function saveMatch(oshiId: UserId, recvId: UserId, affinityScore: number, isDummy: boolean): Promise<void>;
async function listOshiCandidates(oshiId: UserId, limit: number): Promise<Candidate[]>;
async function listRecvOshis(recvId: UserId): Promise<OshiSummary["oshiList"]>;
async function incrementRanking(recvId: UserId): Promise<{ newRanking: number }>;
```

### 5.4 DMStore (DDB)

```ts
async function appendDM(dm: DM): Promise<void>;
async function getDM(dmId: string): Promise<DM | null>;
async function listIncoming(recvId: UserId, status: DM["status"]): Promise<DM[]>;
async function listOutgoing(oshiId: UserId): Promise<DM[]>;
async function updateStatus(dmId: string, status: DM["status"]): Promise<void>;
```

### 5.5 AuditLogStore (S3)

```ts
async function appendLine(prefix: string, line: object): Promise<void>;
//   prefix: "year=YYYY/month=MM/day=DD/{ai-io|violation}"
//   JSON Lines として書き込む。Athena でパーティション読み込み可能
```

---

## 6. メソッド × ストーリー紐付け俯瞰

| Story | 主要メソッドフロー (要約) |
| --- | --- |
| US-COM-01 | CharacterMakerView.onSubmit → OnboardingService.acceptCharacterMake → MatchingService.triggerMatching → (推し→候補生成 / 推され→DMService.generateUniquenessDMs) |
| US-COM-02 | SelfAnalysisView.startSession → SelfAnalysisService.startAnalysis → (loop: nextQuestion / submitAnswer) → summarizeAndPraise (SSE) |
| US-OSHI-01 | DMListView.fetchDMs → 受信表示 / DMService.triggerYakimoki → YakimokiAgent.generateExcuse → 5〜30 分後通知 |
| US-OSHI-02 | CandidateCardView.fetchCandidates → MatchingService.getCandidatesForOshi / pushOshi → recordOshi / DailyPraiseService.getDailyPraise |
| US-RECV-01 | DMListView.fetchDMs (承認待ち) → DMService.generateReplyCandidate (AI-4) / approve / modify / reject |
| US-RECV-02 | DailyPraiseService.submitActionCompletion → 爆褒め SSE |
| US-RM-01/02 (RM) | BehaviorChangeService.proposeNextTask → CoachingAgent.proposeTask (アシスタントキャラ人格で SSE) → recordCompletion → evaluateGraduation |

---

## 7. 詳細業務ルールの送り先 (Functional Design 段階で定義予定)

本ファイルでは契約境界 (シグネチャ + 入出力型) のみ。以下は CONSTRUCTION フェーズの Functional Design (per-Unit) で確定:

- バリデーション境界条件 (キャラメイクパラメータの範囲チェック / 必須項目)
- 例外パス (Bedrock タイムアウト / Guardrail 違反時の再生成戦略 / DDB 競合)
- リトライ戦略 (指数バックオフ / 最大試行回数)
- 並行性制御 (推され ↔ 複数推し向け唯一性 DM の並列度)
- 状態機械 (DM の DRAFT → PENDING_APPROVAL → SENT → READ → YAKIMOKI 遷移詳細)
- ガードレール違反検出時の UX (ユーザーに見せるエラーメッセージ / 透過的に再生成)
- インフレ褒めのアルゴリズム (lastPraiseLevel をどう更新するか)
- 段階遷移の閾値 (Roadmap BehaviorChangeService の段階判定式)
