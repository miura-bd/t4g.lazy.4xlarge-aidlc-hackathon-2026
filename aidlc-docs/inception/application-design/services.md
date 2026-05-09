# Services — 推されと推し

**プロダクト**: 推されと推し
**フェーズ**: AI-DLC Inception / Application Design
**作成日**: 2026-05-09
**スコープ**: ドメインサービス層のオーケストレーション設計
**準拠**: `requirements.md` / `stories.md` / `components.md` / `component-methods.md`

> Service 層は **Domain Logic + Adapter Orchestration** に集中する。AI Adapter / Data Adapter / Cross-cutting (Guardrail / AuditLogger) を組み合わせて Story を成立させる。

---

## 0. サービス一覧 (再掲)

| # | Service | 主担当 Story | MVP/RM |
| --- | --- | --- | --- |
| S1 | OnboardingService | US-COM-01 | MVP |
| S2 | SelfAnalysisService | US-COM-02 | MVP |
| S3 | MatchingService | US-COM-01 シナリオ 2 / US-OSHI-02 | MVP |
| S4 | DMService | US-OSHI-01 / US-RECV-01 | MVP |
| S5 | DailyPraiseService | US-OSHI-02 / US-RECV-02 | MVP |
| S6 | DependencyMeterService | (横断 / FR-AI-06) | MVP |
| S7 | BehaviorChangeService | US-RM-01 / US-RM-02 | Roadmap |

---

## 1. OnboardingService — オーケストレーションパターン

### 1.1 責務境界
- **In**: フロントから REST POST /onboarding/character (推し or 推され パラメータ)
- **Out**: マッチング起動 + ホーム画面遷移情報
- **依存**: UserStore, MatchingService, AuditLogger, (Roadmap) AssistantCharacter store

### 1.2 オーケストレーションフロー

```
[1] 18+ 確認 (Cognito 属性 vs 自己申告チェック整合)
        ↓ 失敗 → 403 (NFR-ETH-05)
        ↓ 成功
[2] パラメータバリデーション (0-100 範囲 / 必須項目)
        ↓ 失敗 → 422
        ↓ 成功
[3] UserStore.saveProfile(side, params)
        ↓
[4] DependencyMeterService.bumpScore(userId, "ONBOARDING_COMPLETED")
        ↓
[5] MatchingService.triggerMatching(userId)        ← 非同期 fire-and-forget
        ↓
[6] AuditLogger.log({event: "ONBOARDING", userId, side})
        ↓
[7] OnboardingResult を返す
```

### 1.3 失敗パスのオーケストレーション
- バリデーション失敗: UserStore に書き込まず Front に 422 を返す
- MatchingService 非同期起動失敗: Onboarding 自体は成功扱いとし、後段でリトライ (DLQ パターン → CONSTRUCTION で詳細化)

---

## 2. SelfAnalysisService — オーケストレーションパターン

### 2.1 責務境界
- **In**: フロントから REST (start / next-question / answer / summary[SSE])
- **Out**: 5〜7 問の対話サイクル + 要約 + 爆褒め
- **依存**: SelfAnalysisAgent (AI-2), PraiseAgent (AI-1), AnalysisStore, PromptGuardrail, AuditLogger

### 2.2 オーケストレーションフロー

**Phase A: セッション開始**
```
startAnalysis(userId)
    ↓
[1] AnalysisStore.appendAnalysisLog(userId, sessionId, START)
    ↓
[2] return sessionId
```

**Phase B: 問答ループ (5〜7 周)**
```
nextQuestion(sessionId)                          submitAnswer(sessionId, answer)
    ↓                                                ↓
[1] AnalysisStore で history 取得                [1] AnalysisStore.appendAnalysisLog
    ↓                                                ↓
[2] SelfAnalysisAgent.generateQuestion(history)  [2] DependencyMeterService.bumpScore
    ↓                                                ↓
[3] PromptGuardrail.validateOutput()             [3] return 200
    ↓ 失敗 → 再生成 (max 2 回)
[4] AuditLogger.logAIInteraction
    ↓
[5] return Question
```

**Phase C: 要約 + 爆褒め (チェイン)**
```
summarizeAndPraise(sessionId)  → SSE で逐次返却
    ↓
[1] AnalysisStore で全回答取得
    ↓
[2] SelfAnalysisAgent.summarize() でストリーミング要約 (Bedrock streaming)
    ↓ 各 Token 受信時:
        ┌─ PromptGuardrail.validateOutput(snapshot) ← フィルタ違反時は中断+再生成
        └─ SSE 経由で Front に push
    ↓ 要約完了
[3] PraiseAgent.generate({context: "self-analysis", summary}) でチェイン
    ↓ 同様にトークンを SSE で push
[4] AuditLogger.logAIInteraction (要約 + 爆褒め両方)
    ↓
[5] SSE close
```

### 2.3 受入基準との対応 (US-COM-02)
- 各問の応答 3 秒以内 → Bedrock streaming + Lambda Response Streaming で初動 1〜2 秒
- 要約に「〜という観点で自分を客観視できている」を必ず含める → System Prompt + 出力検証
- 「自己分析して偉い」を必ず含める → PraiseAgent 用 System Prompt
- 免罪符ワードの 1 つ以上含有 → PraiseAgent 用 System Prompt

---

## 3. MatchingService — オーケストレーションパターン

### 3.1 責務境界
- **In**: OnboardingService.triggerMatching() + フロントの候補取得 / 推す API
- **Out**: 両側マッチング結果 + 推され視点サマリ
- **依存**: MatchStore, UserStore, DMService (推され成立時に唯一性 DM 配信を起動), AuditLogger

### 3.2 オーケストレーションフロー (FR-MATCH-04 トリガー)

```
triggerMatching(userId)
    ↓
[1] UserStore.getUser(userId) で side / params 取得
    ↓
[2-OSHI] side = OSHI:
    a. MatchStore.listRecvCandidates(params, limit=5)
       ↓ 不足: ダミー架空キャラを充当 (FR-MATCH-04, NFR-ETH-04 遵守)
    b. affinityScore 算出 (推し params × 推され params の重み付き内積)
    c. MatchStore.saveMatch(oshiId, recvId, score, isDummy) ×N
[2-RECV] side = RECV:
    a. MatchStore.listOshiCandidates(params, limit=5)
    b. saveMatch(...) ×N
    c. DMService.generateUniquenessDMs(recvUserId, oshiUserIds[]) を非同期起動
       ↓ DMRelayAgent.generateUniqueness() で AI-3 並列生成
    ↓
[3] AuditLogger.log({event: "MATCHING", userId, count, isDummyIncluded})
    ↓
[4] return MatchResult
```

### 3.3 候補取得 / 推す API
- `getCandidatesForOshi(userId)`: MatchStore からフロント表示用に 3〜5 件取得
- `getOshiSummaryForRecv(userId)`: 推され UI の「あなたを推している推し ◯名」サマリ用 (FR-MATCH-03)
- `recordOshi(oshiId, recvId)`: ランキング寄与 + DependencyMeterService.bumpScore("OSHI_PUSH")

### 3.4 NFR-PERF-01 遵守
- triggerMatching の 3 秒以内応答: 同期処理は MatchStore 書き込みまで、唯一性 DM 生成は非同期 fire-and-forget で別 Lambda 起動

---

## 4. DMService — オーケストレーションパターン

### 4.1 責務境界
- **In**: フロントの DM 一覧取得 / 承認 / 微修正 / 却下 + MatchingService からの非同期トリガー (唯一性 DM) + スケジューラからの ヤキモキ起動
- **Out**: 推され実ユーザーの代理として AI が生成する DM (推し向け) + 推し ↔ 推され 双方の状態管理
- **依存**: DMRelayAgent (AI-3, AI-4), YakimokiAgent (AI-5), DMStore, MatchStore, UserStore, PromptGuardrail, AuditLogger

### 4.2 オーケストレーションフロー

**4.2.1 唯一性 DM 並列生成 (AI-3)**
```
generateUniquenessDMs(recvUserId, oshiUserIds[])  ← MatchingService から非同期起動
    ↓
[1] UserStore.getUser(recvUserId) で recvProfile 取得
[2] UserStore.getUsers(oshiUserIds) で oshiProfiles[] 取得
    ↓
[3] DMRelayAgent.generateUniqueness(recvProfile, oshiProfiles[])
    ↓ Bedrock で並列 invocation (oshi 数だけ)
        各 DM 生成時: PromptGuardrail.validateOutput()
        ↓ 違反 → 再生成 (max 2 回 / それでも失敗なら除外 + AuditLogger.logViolation)
    ↓
[4] DMStore.appendDM(dm) ×N (status=PENDING_APPROVAL)
[5] 推され UI に「承認待ち DM ◯件」通知 (SSE / フロントの DMListView がポーリング)
[6] AuditLogger.logAIInteraction
```

**4.2.2 感情労働代行 — 返信案生成 (AI-4)**
```
generateReplyCandidate(recvUserId, incomingDmId)
    ↓
[1] DMStore.getDM(incomingDmId) で受信 DM 取得
[2] UserStore で recv のキャラ性 取得
    ↓
[3] DMRelayAgent.generateReply(recvProfile, incomingDM)
    ↓ PromptGuardrail.validateOutput()
[4] DMStore.appendDM(draft) (status=DRAFT or PENDING_APPROVAL)
[5] AuditLogger
[6] return DMDraft
```

**4.2.3 承認 / 微修正 / 却下 ワークフロー**
```
approve(dmId, actor)              modify(dmId, actor, edited)        reject(dmId, actor)
    ↓                                  ↓                                 ↓
[1] DMStore.updateStatus(SENT)     [1] DMStore で body 更新           [1] DMStore.updateStatus(REJECTED)
[2] 推し向け 配信 (SSE/push)       [2] updateStatus(SENT)             [2] DMRelayAgent で再生成 → PENDING_APPROVAL
[3] DependencyMeterService.bump    [3] 推し向け 配信                  [3] AuditLogger
[4] AuditLogger                     [4] AuditLogger
```

**4.2.4 ヤキモキ演出 (AI-5)**
```
triggerYakimoki(oshiUserId, recvUserId)  ← スケジューラ or DMService.detectIdleness
    ↓
[1] UserStore で recv の RecvCharacterParams 取得
[2] YakimokiAgent.generateExcuse(context)
    ↓ deliveryDelayMs (5〜30 分のランダム) を含む結果を取得
[3] EventBridge Scheduler で deliveryDelayMs 後に通知配信を予約
    ↓ (deliveryDelayMs 経過後)
[4] 推し UI に「仕方ない理由」通知 (SSE / push)
    ↓ 必須: 「推しが嫌いになったわけではない」を明示
[5] AuditLogger
```

### 4.3 NFR-ETH 遵守ポイント
- AI-3 / AI-4 ともに「**推され実ユーザー本人の代理として**」発話する System Prompt
- DMRelayAgent System Prompt に「特定実在人物 (芸能人/知人) を偽装しない」を明示 (NFR-ETH-06)
- 性的表現 / 自傷誘導 / ヒモ・パトロン用語 を Bedrock Guardrails + 出力検証で二重ブロック

---

## 5. DailyPraiseService — オーケストレーションパターン

### 5.1 責務境界
- **In**: フロントのログイン契機 + 行動完了報告 + EventBridge スケジューラ (定期送信)
- **Out**: 段階的に強くなる褒め文 (インフレ) を SSE で配信
- **依存**: PraiseAgent, UserStore, AnalysisStore, PromptGuardrail, AuditLogger

### 5.2 オーケストレーションフロー

**5.2.1 ログイン契機 (US-OSHI-02 朝の通勤シーン等)**
```
getDailyPraise(userId)
    ↓
[1] UserStore.getUser → side / params / lastPraiseLevel 取得
[2] AnalysisStore で過去 7 日の褒め履歴取得 (インフレ判定用)
    ↓
[3] PraiseAgent.generate({
       context: "daily",
       side, characterParams,
       history: { lastPraiseLevel, lastPraiseTexts },
       trigger: { triggerType: "LOGIN", ts }
    })
    ↓ Bedrock streaming
        各 token: PromptGuardrail.validateOutput
[4] SSE で Front に push
[5] UserStore で lastPraiseLevel をインクリメント
[6] DependencyMeterService.bumpScore("LOGIN")
[7] AuditLogger
```

**5.2.2 行動完了爆褒め (US-RECV-02)**
```
submitActionCompletion(userId, actionId)
    ↓
[1] AnalysisStore.appendActionLog
[2] PraiseAgent.generate({context: "action-completion", trigger: {actionId, description}})
    ↓ SSE で爆褒め配信
[3] DependencyMeterService.bumpScore("ACTION_COMPLETED")
[4] AuditLogger
```

### 5.3 トーン分岐 (ハード/ソフト)
- side=OSHI かつ subType=HARD (P-OSHI-H) → 「あなたなしじゃ生きていけない」系の濃い褒め
- side=OSHI かつ subType=SOFT (P-OSHI-S) → 「重いユーザー認定回避」、軽量な褒め
- side=RECV かつ subType=HARD (P-RECV-H) → 「働かないでいいよ、生きてるだけで偉い」濃い
- side=RECV かつ subType=SOFT (P-RECV-S) → 「やったら偉い、頑張ってないのに頑張ってる風」軽

→ subType は UserStore のキャラメイクパラメータから派生して PraiseAgent に渡す

---

## 6. DependencyMeterService — オーケストレーションパターン (横断)

### 6.1 責務境界
- **In**: 各 Service からの bumpScore() 内部呼び出し
- **Out**: 内部スコアの更新 (UI には絶対出さない)
- **依存**: UserStore (拡張カラム), AuditLogger

### 6.2 オーケストレーションフロー

```
bumpScore(userId, event)
    ↓
[1] event 種別ごとに重みを掛ける
    LOGIN: +1, DM_OPEN: +2, ACTION_COMPLETED: +5, PURCHASE: +amount/1000
[2] UserStore.bumpDependencyScore(userId, weighted)
[3] AuditLogger (内部メーター更新ログ)
```

### 6.3 重要原則 (NFR-ETH-07 遵守)
- このスコアは**絶対に UI に表示しない / API 経由でフロントに返さない**
- 将来 BehaviorChangeService の段階遷移判定 / 離脱検知 → 承認爆撃 機能の基盤として内部のみ使用

---

## 7. (Roadmap) BehaviorChangeService — オーケストレーションパターン

### 7.1 責務境界
- **In**: 行動変容モード活性化時のフロント呼び出し + 段階遷移判定タイマー
- **Out**: 段階別タスク提案 + 爆褒め + 一本立ち判定
- **依存**: CoachingAgent (AI-7), UserStore (AssistantCharacter), AnalysisStore, DependencyMeterService, PromptGuardrail, AuditLogger

### 7.2 オーケストレーションフロー (概要のみ / 詳細は将来)

```
proposeNextTask(userId)
    ↓
[1] getCurrentStage(userId) で 1〜6 の段階を内部判定
    判定式: 連続行動日数 + 獲得スキル数 + 関係構築練習回数 + DependencyMeterService.getInternalScore()
    ↓
[2] UserStore.getAssistantCharacter(userId) でアシスタントキャラ取得
[3] CoachingAgent.proposeTask(stage, userProfile, assistantPersona)
    ↓ アシスタントキャラ人格として SSE 出力
        各 token: PromptGuardrail (シナリオ 6 共通: 「あなたを変えています」を出さない)
[4] AnalysisStore に提案ログ記録
[5] AuditLogger
```

```
recordCompletion(userId, taskId)
    ↓
[1] AnalysisStore.appendActionLog
[2] CoachingAgent.praiseCompletion(action) → 爆褒め SSE
[3] DependencyMeterService.bumpScore("BC_TASK_COMPLETED")
[4] evaluateGraduation(userId) を呼び、true なら mode を BEHAVIOR_KEEP に自動切替 (US-RM シナリオ 6 = 一本立ち判定)
[5] AuditLogger
```

---

## 8. サービス間の通信パターンまとめ

| 呼び出し方向 | 同期 / 非同期 | 通信手段 | 例 |
| --- | --- | --- | --- |
| Front ↔ Service | 同期 (SSE) | API Gateway + Lambda Response Streaming | SelfAnalysisService.summarizeAndPraise |
| Service → AI Adapter | 同期 (streaming) | Bedrock SDK | PraiseAgent.generate |
| Service → Service (内部) | 非同期 fire-and-forget | EventBridge / Lambda Async invoke | OnboardingService → MatchingService.triggerMatching |
| Service → Data Adapter | 同期 | DDB SDK / S3 SDK | UserStore.saveProfile |
| Cross-cutting (ガードレール / 監査) | 非同期 (監査) / 同期 (ガードレール) | 関数呼び出し | PromptGuardrail / AuditLogger |
| ヤキモキ遅延配信 | 非同期 (遅延スケジュール) | EventBridge Scheduler | DMService.triggerYakimoki |

---

## 9. ストーリーごとの主要オーケストレーション (1 行サマリ)

| Story | オーケストレーションサマリ |
| --- | --- |
| US-COM-01 | Onboarding → Matching → (推: 候補表示) / (推され: 唯一性 DM 並列生成 + 「推されました」サマリ更新) |
| US-COM-02 | SelfAnalysis: 質問生成 (AI-2) ループ → 要約 (AI-2) → 爆褒めチェイン (AI-1) を SSE で 1 つの体験に |
| US-OSHI-01 | DM 受信 (AI-3 産) 表示 → スケジューラがヤキモキを起動 → AI-5 で正当化文 → 5〜30 分後に通知 |
| US-OSHI-02 | キャラメイク後 → MatchingService の候補表示 → 推す → ランキング寄与 → DailyPraise でログイン爆褒め |
| US-RECV-01 | 受信 DM に対し AI-4 が返信案 → 承認/微修正/却下 → 承認時 SENT → 推しに配信 / 却下時 AI 再生成 |
| US-RECV-02 | DailyPraise で釣れる行動提示 (AI-1) → 行動完了報告 → 爆褒め SSE |
| US-RM-01/02 | BehaviorChange 段階判定 → CoachingAgent 提案 → 完了報告 → 爆褒め → 一本立ち判定 |

---

## 10. 5/30 予選会 MVP デモシナリオに対するサービス利用パス

| シーン | 時間 | サービスフロー |
| --- | --- | --- |
| #1 | 0:00〜0:30 | OnboardingService → MatchingService (擬似マッチで両側成立を見せる) |
| #2 | 0:30〜1:00 | SelfAnalysisService の Phase B + Phase C を SSE 1 本で見せる |
| #3 | 1:00〜1:30 | 推し側 → DMService で受信表示 → triggerYakimoki で正当化通知 |
| #4 | 1:30〜1:45 | (静止画 or 短く) DailyPraiseService の軽量爆褒め |
| #5 | 1:45〜2:30 | 推され側へ視点切替 → DMService.generateReplyCandidate (AI-4) → 承認 → SENT |
| #6 | 2:30〜2:45 | (静止画 or 短く) DailyPraiseService の行動完了爆褒め |
| 締 | 2:45〜3:00 | キーメッセージ (AI に任せる 5 領域 + Roadmap の OS 再インストール) |
