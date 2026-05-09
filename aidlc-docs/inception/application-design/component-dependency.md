# Component Dependency — 推されと推し

**プロダクト**: 推されと推し
**フェーズ**: AI-DLC Inception / Application Design
**作成日**: 2026-05-09
**スコープ**: コンポーネント間の依存関係 / 通信パターン / データフロー
**準拠**: `components.md` / `component-methods.md` / `services.md`

---

## 1. 全体依存マトリクス

「○」は呼び出す側 (行) → 呼び出される側 (列) を示す。

|  | UIShell | CharacterMakerView | SelfAnalysisView | DMListView | CandidateCardView | OnboardingService | SelfAnalysisService | MatchingService | DMService | DailyPraiseService | DependencyMeterService | BehaviorChangeService (RM) | PraiseAgent | SelfAnalysisAgent | DMRelayAgent | YakimokiAgent | CoachingAgent (RM) | PromptGuardrail | AuditLogger | UserStore | AnalysisStore | MatchStore | DMStore | AuditLogStore |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **CharacterMakerView** | — | — | — | — | — | ○ | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — |
| **SelfAnalysisView** | — | — | — | — | — | — | ○ | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — |
| **DMListView** | — | — | — | — | — | — | — | — | ○ | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — |
| **CandidateCardView** | — | — | — | — | — | — | — | ○ | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — |
| **OnboardingService** | — | — | — | — | — | — | — | ○ | — | — | ○ | — | — | — | — | — | — | — | ○ | ○ | — | — | — | — |
| **SelfAnalysisService** | — | — | — | — | — | — | — | — | — | — | ○ | — | ○ | ○ | — | — | — | ○ | ○ | — | ○ | — | — | ○ |
| **MatchingService** | — | — | — | — | — | — | — | — | ○ | — | — | — | — | — | — | — | — | — | ○ | ○ | — | ○ | — | ○ |
| **DMService** | — | — | — | — | — | — | — | ○ | — | — | ○ | — | — | — | ○ | ○ | — | ○ | ○ | ○ | — | ○ | ○ | ○ |
| **DailyPraiseService** | — | — | — | — | — | — | — | — | — | — | ○ | — | ○ | — | — | — | — | ○ | ○ | ○ | ○ | — | — | ○ |
| **DependencyMeterService** | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | ○ | ○ | — | — | — | ○ |
| **BehaviorChangeService (RM)** | — | — | — | — | — | — | — | — | — | — | ○ | — | ○ | — | — | — | ○ | ○ | ○ | ○ | ○ | — | — | ○ |
| **PraiseAgent / SelfAnalysisAgent / DMRelayAgent / YakimokiAgent / CoachingAgent** | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | ○ (System Prompt 注入) | — | — | — | — | — | — |
| **PromptGuardrail** | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | ○ | — | — | — | — | ○ |
| **AuditLogger** | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | — | ○ |

### 依存ルール (ヘキサゴナル境界)

1. **Frontend → Service** のみ。Frontend は AI Adapter / Data Adapter に直接依存しない (API Gateway 経由)
2. **Service → AI Adapter / Data Adapter / Cross-cutting** は許容。Service 同士の呼び出しは許容
3. **AI Adapter / Data Adapter は Service に依存しない** (一方向)
4. **Cross-cutting (Guardrail / AuditLogger) は単独で完結**。各 Service / AI Adapter から呼ばれる
5. **Service 同士の呼び出し**: OnboardingService → MatchingService → DMService の連鎖が中心。逆向きはない (循環依存なし)

---

## 2. レイヤード依存図 (Mermaid)

```mermaid
flowchart TB
    subgraph FE["Frontend (Amplify + React)"]
        UISHELL[UIShell]
        CMV[CharacterMakerView]
        SAV[SelfAnalysisView]
        DLV[DMListView]
        CCV[CandidateCardView]
    end

    subgraph APIGW["API Gateway + Lambda Response Streaming"]
        APIGW_NODE[REST + SSE<br/>Cognito Authorizer]
    end

    subgraph DOMAIN["Domain Services (Lambda / Hexagonal Core)"]
        ONB[OnboardingService]
        SAS[SelfAnalysisService]
        MS[MatchingService]
        DMS[DMService]
        DPS[DailyPraiseService]
        DMR[DependencyMeterService]
        BCS["BehaviorChangeService<br/>(Roadmap)"]
    end

    subgraph AIADP["AI Adapters (Bedrock AgentCore)"]
        PA[PraiseAgent<br/>AI-1]
        SAA[SelfAnalysisAgent<br/>AI-2]
        DMRA[DMRelayAgent<br/>AI-3 + AI-4]
        YA[YakimokiAgent<br/>AI-5]
        CA["CoachingAgent<br/>AI-7 (Roadmap)"]
    end

    subgraph CROSS["Cross-cutting"]
        PG[PromptGuardrail<br/>二重保証]
        AL[AuditLogger]
    end

    subgraph DATAADP["Data Adapters"]
        US[UserStore<br/>DDB]
        AS[AnalysisStore<br/>DDB]
        MST[MatchStore<br/>DDB]
        DMST[DMStore<br/>DDB]
        ALS[AuditLogStore<br/>S3]
    end

    UISHELL --> CMV
    UISHELL --> SAV
    UISHELL --> DLV
    UISHELL --> CCV
    CMV --> APIGW_NODE
    SAV --> APIGW_NODE
    DLV --> APIGW_NODE
    CCV --> APIGW_NODE

    APIGW_NODE --> ONB
    APIGW_NODE --> SAS
    APIGW_NODE --> MS
    APIGW_NODE --> DMS
    APIGW_NODE --> DPS
    APIGW_NODE --> BCS

    ONB --> MS
    ONB --> DMR
    ONB --> US
    ONB --> AS
    ONB --> AL

    SAS --> SAA
    SAS --> PA
    SAS --> PG
    SAS --> AL
    SAS --> AS
    SAS --> ALS
    SAS --> DMR

    MS --> DMS
    MS --> US
    MS --> AS
    MS --> MST
    MS --> ALS

    DMS --> DMRA
    DMS --> YA
    DMS --> PG
    DMS --> AL
    DMS --> US
    DMS --> MST
    DMS --> DMST
    DMS --> ALS
    DMS --> DMR

    DPS --> PA
    DPS --> PG
    DPS --> AL
    DPS --> US
    DPS --> AS
    DPS --> ALS
    DPS --> DMR

    BCS --> CA
    BCS --> PA
    BCS --> PG
    BCS --> AL
    BCS --> US
    BCS --> AS
    BCS --> ALS
    BCS --> DMR

    PA --> PG
    SAA --> PG
    DMRA --> PG
    YA --> PG
    CA --> PG

    DMR --> US
    DMR --> AL
    PG --> AL
    AL --> ALS

    style FE fill:#E3F2FD,stroke:#1565C0
    style APIGW fill:#FFF3E0,stroke:#E65100
    style DOMAIN fill:#E8F5E9,stroke:#2E7D32
    style AIADP fill:#FCE4EC,stroke:#AD1457
    style CROSS fill:#F3E5F5,stroke:#6A1B9A
    style DATAADP fill:#FFFDE7,stroke:#F9A825

    style BCS fill:#FFCCBC,stroke:#BF360C,stroke-dasharray: 5 5
    style CA fill:#FFCCBC,stroke:#BF360C,stroke-dasharray: 5 5
```

> Roadmap (BehaviorChangeService / CoachingAgent) は MVP 範囲外のため点線で示す。

---

## 3. 主要シーケンス図

### 3.1 US-COM-01 オンボーディング & 両側マッチング起動 (Mermaid)

```mermaid
sequenceDiagram
    actor U as ユーザー
    participant CMV as CharacterMakerView
    participant API as API Gateway
    participant ONB as OnboardingService
    participant US as UserStore
    participant DMR as DependencyMeter
    participant MS as MatchingService
    participant DMS as DMService
    participant MST as MatchStore
    participant DMRA as DMRelayAgent
    participant DMST as DMStore
    participant AL as AuditLogger

    U->>CMV: パラメータ入力 → 完了ボタン
    CMV->>API: POST /onboarding/character (params)
    API->>ONB: acceptCharacterMake
    ONB->>US: saveProfile(side, params)
    ONB->>DMR: bumpScore("ONBOARDING_COMPLETED")
    ONB-->>MS: triggerMatching (非同期)
    ONB->>AL: log
    ONB-->>API: 200 OnboardingResult
    API-->>CMV: ホーム遷移
    CMV-->>U: ホーム表示

    Note over MS: 非同期でマッチング処理
    MS->>US: getUser
    MS->>MST: listCandidates / saveMatch
    alt side=RECV
        MS-->>DMS: generateUniquenessDMs (非同期)
        DMS->>DMRA: generateUniqueness (Bedrock 並列)
        DMRA-->>DMS: DM[] (PromptGuardrail validate)
        DMS->>DMST: appendDM (PENDING_APPROVAL)
        DMS->>AL: logAIInteraction
    end
    MS->>AL: log
```

### 3.2 US-COM-02 自己分析 → 要約 → 爆褒めチェイン (Mermaid)

```mermaid
sequenceDiagram
    actor U as ユーザー
    participant SAV as SelfAnalysisView
    participant API as API Gateway
    participant SAS as SelfAnalysisService
    participant SAA as SelfAnalysisAgent
    participant PA as PraiseAgent
    participant PG as PromptGuardrail
    participant AS as AnalysisStore
    participant AL as AuditLogger

    U->>SAV: 「自己分析する」タップ
    SAV->>API: POST /self-analysis/start
    API->>SAS: startAnalysis
    SAS->>AS: appendLog(START)
    SAS-->>SAV: sessionId

    loop 5〜7 周
        SAV->>API: GET /self-analysis/next-question
        API->>SAS: nextQuestion
        SAS->>AS: get history
        SAS->>SAA: generateQuestion(history)
        SAA-->>SAS: Question
        SAS->>PG: validateOutput
        SAS-->>SAV: Question
        U->>SAV: 回答
        SAV->>API: POST /self-analysis/answer
        API->>SAS: submitAnswer
        SAS->>AS: appendLog
    end

    SAV->>API: GET /self-analysis/summary (SSE)
    API->>SAS: summarizeAndPraise
    SAS->>SAA: summarize (streaming)
    activate SAA
    SAA-->>SAS: token (loop)
    SAS->>PG: validate snapshot
    SAS-->>SAV: SSE token
    deactivate SAA

    SAS->>PA: generate({context: "self-analysis", summary})
    activate PA
    PA-->>SAS: token (loop)
    SAS->>PG: validate snapshot
    SAS-->>SAV: SSE token (爆褒め)
    deactivate PA

    SAS->>AL: logAIInteraction
    SAS-->>SAV: SSE close
    SAV-->>U: 要約 + 「自己分析して偉い」表示
```

### 3.3 US-RECV-01 推され側 — AI が DM 返信案を代理生成 → 承認 (Mermaid)

```mermaid
sequenceDiagram
    actor R as 推され (真城)
    participant DLV as DMListView
    participant API as API Gateway
    participant DMS as DMService
    participant DMRA as DMRelayAgent
    participant PG as PromptGuardrail
    participant US as UserStore
    participant DMST as DMStore
    participant AL as AuditLogger
    participant O as 推し (七瀬)

    Note over O,DMST: 推しから真城へ DM が届いている (status=READ_BY_RECV)

    R->>DLV: 承認待ち枠を開く
    DLV->>API: GET /dm?status=PENDING_APPROVAL
    API->>DMS: listIncoming
    DMS->>DMST: listIncoming
    DMST-->>DMS: DM[]
    DMS-->>DLV: DM[]

    Note over DMS: 各 DM に対して AI-4 返信案生成
    DLV->>API: POST /dm/{dmId}/generate-reply
    API->>DMS: generateReplyCandidate
    DMS->>DMST: getDM(incomingDmId)
    DMS->>US: getUser(recv)
    DMS->>DMRA: generateReply(recvProfile, incomingDM)
    DMRA-->>DMS: DMDraft
    DMS->>PG: validateOutput
    PG-->>DMS: allowed
    DMS->>DMST: appendDM(draft, PENDING_APPROVAL)
    DMS->>AL: logAIInteraction
    DMS-->>DLV: DMDraft

    R->>DLV: 「承認」をタップ
    DLV->>API: POST /dm/{dmId}/approve
    API->>DMS: approve
    DMS->>DMST: updateStatus(SENT)
    DMS->>O: SSE/push 配信
    DMS->>AL: log
    DMS-->>DLV: DM (SENT)
```

### 3.4 US-OSHI-01 唯一性 DM 受信 + ヤキモキ演出 (Mermaid)

```mermaid
sequenceDiagram
    participant Sched as EventBridge Scheduler
    participant DMS as DMService
    participant YA as YakimokiAgent
    participant DMST as DMStore
    actor O as 推し (七瀬)

    Note over Sched,DMS: 唯一性 DM 配信完了後、一定時間 DM 既読/返信が無いことを検知
    Sched->>DMS: triggerYakimoki(oshiId, recvId)
    DMS->>YA: generateExcuse(context)
    YA-->>DMS: { excuse: "今ちょっとお風呂入ってて…", deliveryDelayMs: 12min }
    DMS->>Sched: scheduleDelivery(deliveryDelayMs)
    Note over Sched: 12 分待機
    Sched->>O: SSE/push「仕方ない理由」通知 (推しが嫌いになったわけではないことを必ず明示)
```

---

## 4. データフロー図

### 4.1 マスター情報フロー

```
キャラメイク入力 → CharacterMakerView → OnboardingService → UserStore (PROFILE)
                                                          → MatchingService → MatchStore (MATCH)
                                                                            → DMService (RECV 側のみ)
                                                                              → DMStore (PENDING_APPROVAL)
```

### 4.2 AI 対話フロー (推し側 DM 受信)

```
[推され実ユーザー (真城)] のキャラメイク + マッチング成立
        ↓
DMRelayAgent.generateUniqueness (AI-3) — System Prompt + Bedrock Guardrails の二重保証
        ↓
PromptGuardrail.validateOutput — NG ワード検査 + 違反検出は AuditLogger へ
        ↓
DMStore に PENDING_APPROVAL で保存
        ↓
[推され実ユーザー本人] が承認 → DMService.approve → status=SENT
        ↓
[推し実ユーザー (七瀬)] へ配信 (SSE)
```

### 4.3 監査ログフロー (NFR-OBS-01, 02)

```
全 AI 対話 → AuditLogger.logAIInteraction → CloudWatch Logs (運用)
                                          → AuditLogStore (S3 JSON Lines)
                                              → Athena でクエリ可能
                                                  → プロンプト品質後評価 (NFR-OBS-01)
                                                  → 倫理違反集計 (NFR-OBS-02)

PromptGuardrail 違反検出 → AuditLogger.logViolation → 同上
```

---

## 5. 通信パターンサマリ

| 通信 | プロトコル | パターン | 例 |
| --- | --- | --- | --- |
| Front ↔ API Gateway | HTTPS REST + SSE | 同期 / ストリーミング | キャラメイク POST / 自己分析 SSE |
| API Gateway ↔ Lambda | AWS Lambda 統合 | Lambda Response Streaming 対応 | summarizeAndPraise |
| Lambda ↔ Lambda (内部 Service) | EventBridge / 非同期 invoke | 非同期 fire-and-forget | OnboardingService → MatchingService.triggerMatching |
| Lambda ↔ Bedrock | Bedrock SDK | 同期 (streaming あり) | PraiseAgent.generate |
| Lambda ↔ DDB | DDB SDK | 同期 KV / Query | UserStore / MatchStore / DMStore |
| Lambda ↔ S3 | S3 SDK | 非同期 (PutObject) | AuditLogStore |
| 遅延配信 | EventBridge Scheduler | 非同期スケジュール | ヤキモキ演出 (5〜30 分後通知) |

---

## 6. 循環依存検証

主要呼び出し方向:
```
Frontend → API Gateway → Service → (AI Adapter | Data Adapter | Cross-cutting | 他 Service)
```

Service 間の連鎖:
- OnboardingService → MatchingService → DMService (連鎖、逆向き無し)
- DMService → DependencyMeterService (横断、逆向き無し)
- BehaviorChangeService → DependencyMeterService (横断、逆向き無し)

→ **循環依存なし**。各 Service / Adapter は独立に単体テスト可能。

---

## 7. NFR との整合性

| NFR | 依存パターンでの対応 |
| --- | --- |
| NFR-PERF-01 (3 秒応答) | フロント → API Gateway → Lambda Response Streaming → Bedrock streaming の経路を最短化、初動 1〜2 秒目標 |
| NFR-ARCH-01 (Bedrock AgentCore Serverless) | 全層が Lambda + Bedrock + DDB + S3 のフルサーバレス、依存関係は SDK 呼び出しのみ |
| NFR-ETH-01〜08 | PromptGuardrail を AI Adapter のラッパとして強制注入 (System Prompt + Bedrock Guardrails 二重) |
| NFR-OBS-01, 02 | AuditLogger を Service 層から非同期で呼ぶ + AuditLogStore で 90 日アーカイブ |
| NFR-PERF-01 (10 ユーザー同時接続) | Lambda 同時実行数とリザーブドコンカレンシ設定 (CONSTRUCTION で詳細化) |
