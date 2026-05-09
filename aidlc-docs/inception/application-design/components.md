# Components — 推されと推し

**プロダクト**: 推されと推し / AI による洗脳で気づいたら幸せになってるあなたと私
**チーム**: t4g.lazy.4xlarge
**作成日**: 2026-05-09
**フェーズ**: AI-DLC Inception / Application Design
**準拠**: `requirements.md` (4 件追補済) / `personas.md` / `stories.md` / `application-design-plan.md` (Q1=D ハイブリッド / Q2=B 機能別 Agent / Q3=C DDB+S3 / Q4=D Amplify+React / Q5=A SSE / Q6=C 二重ガードレール / Q7=B CloudWatch+S3+Athena)

---

## 0. アーキテクチャの方針

ヘキサゴナル (Port & Adapter) の境界の中に、AI Agent をドメインサービスの依存先 (AI Adapter) として配置する。**Domain Service はビジネスルールに集中し、AI / DB / UI は Adapter として外部化される。**

5 カテゴリで合計 **22 コンポーネント** を定義 (うち Roadmap = 2):

| # | カテゴリ | コンポーネント数 (MVP / Roadmap) |
| --- | --- | --- |
| 1 | Frontend Components | 5 / 0 |
| 2 | Domain Services | 6 / 1 (RM) |
| 3 | AI Adapters (Bedrock AgentCore) | 4 / 1 (RM) |
| 4 | Cross-cutting | 2 / 0 |
| 5 | Data Adapters | 5 / 0 |

---

## 1. Frontend Components (Amplify + React + TypeScript)

### 1.1 UIShell

| 項目 | 内容 |
| --- | --- |
| **責務** | アプリ全体のシェル。レトロ RPG / 初期 mixi 風オレンジ色調の共通レイアウト / ヘッダ / フッタ / ナビゲーション。Cognito 認証ラッパ (NFR-ETH-05 18+ 確認済セッション保持) |
| **提供 IF** | `<UIShell>{children}</UIShell>` Render Prop |
| **必要 IF** | Cognito Auth (Amplify), Routing (React Router) |
| **紐付き** | 全 Story 共通 |

### 1.2 CharacterMakerView

| 項目 | 内容 |
| --- | --- |
| **責務** | キャラメイク UI。推し側 (さびしさ/独占欲/承認欲求/課金意欲/好みタグ) / 推され側 (省力化設定/キャラ性) / Roadmap でアシスタントキャラビジュアル選択 (実写風/アニメ風/浮世絵風/ルネサンス調 等) |
| **提供 IF** | `/character-maker` ルートをレンダリング |
| **必要 IF** | OnboardingService (Domain) — REST POST /onboarding/character |
| **紐付き** | US-COM-01 / US-RM-01, 02 (Roadmap) |

### 1.3 SelfAnalysisView

| 項目 | 内容 |
| --- | --- |
| **責務** | 誘導尋問の進行 UI (1 問ずつ表示 → 回答 → 次の問) + 要約表示 + 爆褒めアニメーション |
| **提供 IF** | `/self-analysis` ルート |
| **必要 IF** | SelfAnalysisService (Domain) — REST GET /self-analysis/next-question, POST /self-analysis/answer, GET /self-analysis/summary (SSE) |
| **紐付き** | US-COM-02 |

### 1.4 DMListView

| 項目 | 内容 |
| --- | --- |
| **責務** | 推し側: 受信 DM 一覧 / 推され側: 「あなたを推している推し ◯名」サマリ + 承認待ち DM 一覧 (推しごとの相性 % / パラメータをヘッダ表示 / 承認・微修正・却下ボタン) |
| **提供 IF** | `/dm` ルート |
| **必要 IF** | DMService (Domain) — REST GET /dm, POST /dm/approve, POST /dm/modify, POST /dm/reject |
| **紐付き** | US-OSHI-01 (推し受信) / US-RECV-01 (推され承認) |

### 1.5 CandidateCardView

| 項目 | 内容 |
| --- | --- |
| **責務** | 推し側ホーム画面: 推され実ユーザー候補 3〜5 枚カード (相性 % 付き) + 「推す」ボタン → 推しランキング寄与 |
| **提供 IF** | `/home` の一部としてレンダリング |
| **必要 IF** | MatchingService (Domain) — REST GET /matching/candidates, POST /matching/oshi |
| **紐付き** | US-OSHI-02 (候補提示) |

---

## 2. Domain Services (Lambda / Hexagonal Domain Core)

### 2.1 OnboardingService

| 項目 | 内容 |
| --- | --- |
| **責務** | キャラメイク受付 (推し/推され パラメータ保存) → MatchingService にマッチング起動を委譲 → ホーム画面遷移情報を返却 |
| **提供 IF** | `acceptCharacterMake(userId, side, params): OnboardingResult` |
| **必要 IF** | UserStore, MatchingService, AuditLogger |
| **紐付き要件** | FR-MODE-01, FR-COM-01, FR-OSHI-01, FR-RECV-01 |

### 2.2 SelfAnalysisService

| 項目 | 内容 |
| --- | --- |
| **責務** | 誘導尋問 5〜7 問の生成 → 進行管理 → 全回答後の要約 + 爆褒め生成オーケストレーション |
| **提供 IF** | `startAnalysis(userId): SessionId` / `nextQuestion(sessionId): Question` / `submitAnswer(sessionId, answer): void` / `summarizeAndPraise(sessionId): SSEStream<SummaryAndPraise>` |
| **必要 IF** | SelfAnalysisAgent (AI Adapter), PraiseAgent, AnalysisStore, AuditLogger, PromptGuardrail |
| **紐付き要件** | FR-COM-02, FR-COM-03, FR-COM-05, FR-AI-01, FR-AI-02 |

### 2.3 MatchingService

| 項目 | 内容 |
| --- | --- |
| **責務** | キャラメイク完了をトリガーに両側マッチング処理を実行。推し → 推され候補リスト / 推され → 「推している推しサマリ」を生成。実ユーザー不足時はダミー架空キャラを充当 |
| **提供 IF** | `triggerMatching(userId): MatchResult` / `getCandidatesForOshi(userId): Candidate[]` / `getOshiSummaryForRecv(userId): OshiSummary` / `recordOshi(oshiUserId, recvUserId)` |
| **必要 IF** | MatchStore, UserStore, AuditLogger |
| **紐付き要件** | FR-MATCH-01, 02, 03, 04 |

### 2.4 DMService

| 項目 | 内容 |
| --- | --- |
| **責務** | 推 ↔ 推され 間の DM 送受信を統括。推され実ユーザーから推し向け **唯一性 DM 並列生成** (AI-3) + 推し → 推され DM への **感情労働代行返信案生成** (AI-4)。承認・微修正・却下ワークフロー |
| **提供 IF** | `generateUniquenessDMs(recvUserId, oshiUserIds[]): DM[]` / `generateReplyCandidate(recvUserId, incomingDmId): DMDraft` / `approve(dmId)` / `modify(dmId, edited)` / `reject(dmId)` / `triggerYakimoki(oshiUserId, recvUserId): void` |
| **必要 IF** | DMRelayAgent (AI-3, 4), YakimokiAgent (AI-5), DMStore, MatchStore, UserStore, PromptGuardrail, AuditLogger |
| **紐付き要件** | FR-OSHI-04, 05, FR-RECV-04, 05, FR-AI-03, 04, 05, FR-MATCH-01 |

### 2.5 DailyPraiseService

| 項目 | 内容 |
| --- | --- |
| **責務** | 日次褒め配信 (ログイン契機 / スケジュール契機)。推し側 / 推され側 でトーン切替 (ハード/ソフト)。免罪符ワード織り込み |
| **提供 IF** | `getDailyPraise(userId): SSEStream<PraiseMessage>` / `submitActionCompletion(userId, actionId): SSEStream<PraiseMessage>` |
| **必要 IF** | PraiseAgent, UserStore, AnalysisStore, PromptGuardrail, AuditLogger |
| **紐付き要件** | FR-COM-04, FR-COM-05, FR-RECV-02, 03, FR-AI-01 |

### 2.6 DependencyMeterService (UI 非表示 / 内部のみ)

| 項目 | 内容 |
| --- | --- |
| **責務** | ユーザー単位の依存度スコアを内部計測 (連続ログイン日数 / DM 開封率 / アクション完了数 / 課金額)。スコアは UI には表示しない (NFR-ETH-07)。将来の離脱検知 → 承認爆撃機能の基盤 |
| **提供 IF** | `bumpScore(userId, event): void` / `getInternalScore(userId): float` (内部呼び出しのみ) |
| **必要 IF** | UserStore (拡張)、AuditLogger |
| **紐付き要件** | FR-AI-06, NFR-ETH-07 |

### 2.7 (Roadmap) BehaviorChangeService

| 項目 | 内容 |
| --- | --- |
| **責務** | 行動変容モード活性時、ユーザーの段階遷移 (① 些細な日常行動 → ⑥ 一本立ち) を判定し、CoachingAgent に次段階タスク生成を依頼 |
| **提供 IF** | `getCurrentStage(userId): Stage` / `proposeNextTask(userId): SSEStream<TaskProposal>` / `recordCompletion(userId, taskId)` / `evaluateGraduation(userId): bool` |
| **必要 IF** | CoachingAgent, UserStore, AnalysisStore, PromptGuardrail, AuditLogger |
| **紐付き要件** | FR-MODE-04, FR-AI-07 (Roadmap) |

---

## 3. AI Adapters (Bedrock AgentCore — 機能別 Agent)

各 Agent は独立した System Prompt + Bedrock Guardrails (二重保証) を持つ。Lambda Function URL or Bedrock AgentCore Action Group 経由で呼び出される。

### 3.1 PraiseAgent (AI-1)

| 項目 | 内容 |
| --- | --- |
| **責務** | 即時 / 日次 / インフレ褒め文生成。昨日より強い言葉。免罪符ワード織り込み |
| **提供 IF** | `generate(input: PraiseInput): SSEStream<token>` |
| **System Prompt 要点** | 「あなたはユーザーを爆褒めする AI。性的・自傷・実在人物模倣・ヒモ・パトロン用語禁止。」 |
| **モデル** | Claude (開発開始時の最新 / NFR-ARCH-02) |
| **紐付き要件** | FR-AI-01 |

### 3.2 SelfAnalysisAgent (AI-2)

| 項目 | 内容 |
| --- | --- |
| **責務** | 誘導尋問の質問生成 + 回答要約。「自己分析して偉い」フレーミング |
| **提供 IF** | `generateQuestion(history): Question` / `summarize(history): SSEStream<token>` |
| **System Prompt 要点** | 「回答ハードルが低い肯定的な誘導尋問を生成。要約には必ず『〜という観点で自分を客観視できている』を含める」 |
| **紐付き要件** | FR-AI-02 |

### 3.3 DMRelayAgent (AI-3 + AI-4)

| 項目 | 内容 |
| --- | --- |
| **責務** | (1) 推され実ユーザーの**代理として** 「あなただけ」DM を全推し向けに並列生成 (AI-3) / (2) 推 → 推され DM への返信案を本人キャラ性で代理生成 (AI-4) |
| **提供 IF** | `generateUniqueness(recvProfile, oshiProfiles[]): DM[]` / `generateReply(recvProfile, incomingDM): DMDraft` |
| **System Prompt 要点** | 「あなたは推され本人の人格代理 AI。**特定実在人物 (芸能人 / 知人) を偽装しない**。文体は推され本人のキャラメイク設定に従う」 |
| **紐付き要件** | FR-AI-03, 04 |

### 3.4 YakimokiAgent (AI-5)

| 項目 | 内容 |
| --- | --- |
| **責務** | 既読スルー / 返信遅延の「仕方ない理由」生成。推しが嫌いになったわけではないことを必ず明示 |
| **提供 IF** | `generateExcuse(context): Excuse` |
| **System Prompt 要点** | 「日常的でリアルな『仕方ない理由』を生成。例: 『今ちょっとお風呂入ってて』『友達からの相談で抜けられなくて』」 |
| **紐付き要件** | FR-AI-05 |

### 3.5 (Roadmap) CoachingAgent (AI-7)

| 項目 | 内容 |
| --- | --- |
| **責務** | 行動変容モード活性時、ユーザー作成のバーチャルアシスタントキャラ人格として、段階別の極小タスク提案 / 達成爆褒め / コーチングコメント生成 |
| **提供 IF** | `proposeTask(stage, userProfile, assistantPersona): TaskProposal` / `praiseCompletion(action, userProfile): SSEStream<token>` |
| **System Prompt 要点** | 「**ユーザー作成のアシスタントキャラ人格** として発話。段階に応じて爆褒め → 微小提案 → 関係構築練習 → 生活スキル → 一本立ち を促す。**ユーザーに『あなたを変えています』と告げない**」 |
| **紐付き要件** | FR-AI-07 (Roadmap), FR-MODE-04 |

---

## 4. Cross-cutting Components

### 4.1 PromptGuardrail (二重保証)

| 項目 | 内容 |
| --- | --- |
| **責務** | (1) System Prompt への倫理ルール埋め込み (NFR-ETH-01〜08) / (2) Bedrock Guardrails 設定 (性的/自傷/実在人物模倣カテゴリ) / 出力後 NG ワード検出 → AuditLogger に記録 |
| **提供 IF** | `wrapPrompt(systemPrompt, ethCtx): string` / `validateOutput(output): GuardrailResult` |
| **必要 IF** | Bedrock Guardrails API, AuditLogger |
| **紐付き要件** | NFR-ETH-01〜08, NFR-OBS-02 |

### 4.2 AuditLogger

| 項目 | 内容 |
| --- | --- |
| **責務** | 全 AI 入出力 + 倫理違反検出を構造化ログとして S3 (JSON Lines) に書き込み。CloudWatch Logs にも運用ログを出力 |
| **提供 IF** | `logAIInteraction(agentId, input, output, meta)` / `logViolation(violationType, content, meta)` |
| **必要 IF** | AuditLogStore (S3 Adapter), CloudWatch Logs |
| **紐付き要件** | NFR-OBS-01, 02 |

---

## 5. Data Adapters (DynamoDB Single-Table + S3)

DynamoDB の Single-Table Design で運用データを集約。Partition Key (PK) + Sort Key (SK) のパターンで論理的にエンティティを区分。

### 5.1 UserStore (DDB)

| 項目 | 内容 |
| --- | --- |
| **責務** | User / Mode / PraiseProfile (推し/推され パラメータ) / AssistantCharacter (Roadmap) のエンティティ管理 |
| **提供 IF** | `getUser(userId)` / `saveProfile(userId, profile)` / `getAssistantCharacter(userId)` |
| **PK/SK** | PK=`USER#{userId}`, SK=`PROFILE` / `ASSISTANT_CHAR` |
| **紐付き要件** | FR-COM-01, FR-OSHI-01, FR-RECV-01, FR-MODE-04 (Roadmap) |

### 5.2 AnalysisStore (DDB)

| 項目 | 内容 |
| --- | --- |
| **責務** | SelfAnalysisLog / ActionLog の時系列保存 |
| **提供 IF** | `appendLog(userId, log)` / `getLogs(userId, range)` |
| **PK/SK** | PK=`USER#{userId}`, SK=`ANALYSIS#{ts}` / `ACTION#{ts}` |
| **紐付き要件** | FR-COM-02, FR-RECV-03 |

### 5.3 MatchStore (DDB)

| 項目 | 内容 |
| --- | --- |
| **責務** | Match (推し ↔ 推され ペア + 相性 %) / Relationship (推しランキング寄与) / Ranking |
| **提供 IF** | `saveMatch(oshiId, recvId, score)` / `listOshiCandidates(oshiId)` / `listRecvOshis(recvId)` / `incrementRanking(recvId)` |
| **PK/SK** | PK=`MATCH#{oshiId}`, SK=`RECV#{recvId}` / GSI: PK=`RECV#{recvId}`, SK=`MATCH#{oshiId}` |
| **紐付き要件** | FR-MATCH-01, 02, 03, 04 |

### 5.4 DMStore (DDB)

| 項目 | 内容 |
| --- | --- |
| **責務** | DM (送受信 / 承認待ち / 承認済 / 却下) の時系列保存。読み取り側のステータス管理 (既読 / 未読 / ヤキモキ中) |
| **提供 IF** | `appendDM(dm)` / `listIncoming(recvId, status)` / `listOutgoing(oshiId)` / `updateStatus(dmId, status)` |
| **PK/SK** | PK=`USER#{userId}`, SK=`DM#{ts}#{dmId}` |
| **紐付き要件** | FR-OSHI-04, 05, FR-RECV-04, 05 |

### 5.5 AuditLogStore (S3)

| 項目 | 内容 |
| --- | --- |
| **責務** | AI 入出力ログ (NFR-OBS-01) と倫理違反検出ログ (NFR-OBS-02) を JSON Lines として S3 にパーティション分割保存 (date=YYYY-MM-DD prefix)。Athena でクエリ可能 |
| **提供 IF** | `appendLine(line)` (内部呼び出し / AuditLogger 経由) |
| **構造** | `s3://oshi-osare-audit/year=YYYY/month=MM/day=DD/{ai-io|violation}-XX.jsonl` |
| **紐付き要件** | NFR-OBS-01, 02 |

---

## 6. コンポーネント × ストーリー紐付け俯瞰

| Story | Frontend | Domain Service | AI Adapter | Cross-cutting | Data |
| --- | --- | --- | --- | --- | --- |
| US-COM-01 | UIShell, CharacterMakerView | OnboardingService, MatchingService | — | AuditLogger | UserStore, MatchStore |
| US-COM-02 | UIShell, SelfAnalysisView | SelfAnalysisService | SelfAnalysisAgent, PraiseAgent | PromptGuardrail, AuditLogger | AnalysisStore, AuditLogStore |
| US-OSHI-01 | UIShell, DMListView | DMService, DependencyMeterService | DMRelayAgent (AI-3), YakimokiAgent | PromptGuardrail, AuditLogger | DMStore, MatchStore, AuditLogStore |
| US-OSHI-02 | UIShell, CandidateCardView | MatchingService, DailyPraiseService | PraiseAgent | PromptGuardrail, AuditLogger | MatchStore, UserStore |
| US-RECV-01 | UIShell, DMListView | DMService, MatchingService | DMRelayAgent (AI-4), PraiseAgent | PromptGuardrail, AuditLogger | DMStore, MatchStore, AuditLogStore |
| US-RECV-02 | UIShell | DailyPraiseService | PraiseAgent | PromptGuardrail, AuditLogger | UserStore, AnalysisStore |
| US-RM-01 (RM) | UIShell, CharacterMakerView (拡張) | BehaviorChangeService | CoachingAgent, PraiseAgent | PromptGuardrail, AuditLogger | UserStore, AnalysisStore |
| US-RM-02 (RM) | UIShell, CharacterMakerView (拡張) | BehaviorChangeService | CoachingAgent, PraiseAgent | PromptGuardrail, AuditLogger | UserStore, AnalysisStore |

---

## 7. NFR との対応

| NFR | 関連コンポーネント |
| --- | --- |
| NFR-PERF-01 (3 秒応答) | API Gateway + Lambda Response Streaming + SSE / Bedrock ストリーミング |
| NFR-ARCH-01 (Bedrock AgentCore Serverless) | AI Adapter 全件 + Lambda + DDB + S3 (フルサーバレス) |
| NFR-ETH-01〜08 | PromptGuardrail (二重保証) + AuditLogger |
| NFR-OBS-01, 02 | AuditLogger + AuditLogStore (S3 + Athena) |
| NFR-A11Y-02 (レトロ UI) | UIShell + 各 View の独自 CSS |
