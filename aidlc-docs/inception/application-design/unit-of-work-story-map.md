# Unit of Work ↔ Story Map — 推されと推し

**プロダクト**: 推されと推し
**フェーズ**: AI-DLC Inception / Units Generation
**作成日**: 2026-05-09
**スコープ**: 全 8 ストーリー (MVP 6 + Roadmap 2) を 5 Unit + Roadmap Unit に完全マッピングする
**準拠**: `stories.md` (US-COM/OSHI/RECV × MVP 6 + RM 2) / `unit-of-work.md` / `unit-of-work-dependency.md`

---

## 1. ストーリー × Unit マトリクス

凡例:
- **★** = 主担当 Unit (Service / AI Adapter の主実装責任)
- ○ = 副担当 (UI 担当 / 横断機能利用 等)
- — = 関与なし

| Story ID | Story タイトル | Unit-A Frontend | Unit-B Onboarding | Unit-C Praise | Unit-D DM | Unit-E Cross/IaC | Unit-RM (RM) |
| --- | --- | --- | --- | --- | --- | --- | --- |
| US-COM-01 | オンボーディング & 褒めキャラメイク + 両側マッチング起点 | ★ (CharacterMakerView) | **★** (Onboarding + Matching) | — | ○ (RECV 側時に唯一性 DM 起動) | ○ (Cognito + IaC) | — |
| US-COM-02 | 誘導尋問自己分析と爆褒め | ★ (SelfAnalysisView) | — | **★** (SelfAnalysis + Praise) | — | ○ (Guardrail / Audit / prompts) | — |
| US-OSHI-01 | 唯一性 DM とヤキモキ演出 (ハード推し) | ★ (DMListView 推し受信) | — | — | **★** (DMService + DMRelayAgent + YakimokiAgent) | ○ (Guardrail / Audit) | — |
| US-OSHI-02 | 候補提示と日次褒めで日常の彩り (ソフト推し) | ★ (CandidateCardView) | **★** (候補表示) | **★** (日次褒め) | — | ○ (Audit) | — |
| US-RECV-01 | 感情労働全代行とゲーム化承認作業 (ハード推され) | ★ (DMListView 推され承認待ち) | — | — | **★** (DMService + DMRelayAgent AI-4) | ○ (Guardrail / Audit) | — |
| US-RECV-02 | 副業推されと釣れる行動の AI 提示 (ソフト推され) | ★ (UIShell + 行動表示) | — | **★** (DailyPraiseService 行動提示 + 爆褒め) | — | ○ (Audit) | — |
| US-RM-01 (RM) | 行動変容モード — 推され OS 再インストール | (将来 ★ アシスタントキャラ UI) | — | — | — | (将来 ○ prompts/coaching) | **★** (BehaviorChangeService + CoachingAgent) |
| US-RM-02 (RM) | 行動変容モード — 推し OS 再インストール | (将来 ★) | — | — | — | (将来 ○) | **★** |

---

## 2. ストーリー網羅性検証

### 2.1 全 8 ストーリーの主担当 Unit が決定済か?

| Story | 主担当 Unit | 確認 |
| --- | --- | --- |
| US-COM-01 | Unit-B (Service) + Unit-A (UI) | ✅ |
| US-COM-02 | Unit-C (Service+AI) + Unit-A (UI) | ✅ |
| US-OSHI-01 | Unit-D (Service+AI) + Unit-A (UI) | ✅ |
| US-OSHI-02 | Unit-B (候補) + Unit-C (日次) + Unit-A (UI) | ✅ (複数 Unit 共同) |
| US-RECV-01 | Unit-D (Service+AI) + Unit-A (UI) | ✅ |
| US-RECV-02 | Unit-C (Service+AI) + Unit-A (UI) | ✅ |
| US-RM-01 | Unit-RM (将来) | ✅ (Roadmap) |
| US-RM-02 | Unit-RM (将来) | ✅ (Roadmap) |

→ **全 8 ストーリーが主担当 Unit に割り当てられている**。網羅性 OK。

### 2.2 横断 Unit-E の関与確認

| Unit-E 機能 | 利用ストーリー |
| --- | --- |
| PromptGuardrail (二重保証) | US-COM-02 / US-OSHI-01 / US-OSHI-02 / US-RECV-01 / US-RECV-02 / US-RM (将来) — 全 AI 利用ストーリー |
| AuditLogger | 全ストーリー (NFR-OBS-01, 02 のため例外なし) |
| shared-types DTO | 全ストーリー |
| prompts (System Prompt 共通断片) | AI 利用ストーリー全件 |
| Cognito 認証 | 全ストーリー (NFR-ETH-05 18+ のため必須) |
| CDK / IaC | 全ストーリー (デプロイ時) |

→ **Unit-E が全ストーリーに横断対応**を確認。

### 2.3 INVEST 原則 × Unit 視点の再評価

`stories.md` 付録 A INVEST チェックリストでは Story 視点で評価したが、Unit 視点でも再評価:

| Unit | Independent (他 Unit から独立) | Negotiable (機能スコープ調整可能) | Valuable (価値提供) | Estimable (見積可能) | Small (短期完了可) | Testable (テスト可能) |
| --- | --- | --- | --- | --- | --- | --- |
| Unit-A | ⚠ Unit-E 共有型に依存 | ✅ レトロ UI 装飾は調整可 | ✅ ユーザー接点 | ✅ M | ✅ 5〜10 日想定 | ✅ E2E + Visual テスト |
| Unit-B | ⚠ Unit-D Async invoke 連動 | ✅ 候補数調整可 | ✅ 両側マッチ起点 | ✅ M | ✅ 5〜7 日 | ✅ Service 単体 + 連動 |
| Unit-C | ⚠ Bedrock 連携重 | ✅ 質問数 / 褒めトーン調整可 | ✅ 褒めたおすループ | ✅ L | ⚠ SSE 実装が時間要 | ✅ プロンプトテスト |
| Unit-D | ⚠ Bedrock 並列 + EventBridge 重 | ✅ ヤキモキ閾値調整可 | ✅ DM 中核 | ✅ L | ⚠ 並列処理 + 状態機械 | ✅ プロンプト + 状態遷移 |
| Unit-E | ✅ 完全独立 | ✅ Guardrail 厳しさ調整可 | ✅ 全 Unit が利用 | ✅ M | ✅ 3〜5 日 | ✅ Guardrail テスト + IaC dry-run |

⚠ マークの Unit (A, B, C, D) は「他 Unit に依存する」「実装ボリュームが大きい」項目があり、開発初日に Unit-E (横断) を整備してから他 Unit の並行実装に入る順序が重要。

---

## 3. ストーリー詳細 × Unit 内訳

### 3.1 US-COM-01 オンボーディング & 褒めキャラメイク + 両側マッチング起点

| Unit | 担当部分 | 主要シナリオ |
| --- | --- | --- |
| **Unit-A Frontend** | CharacterMakerView (推し / 推され パラメータ入力 / Cognito 18+ チェック) | シナリオ 1 (UI) |
| **Unit-B Onboarding** | OnboardingService.acceptCharacterMake / MatchingService.triggerMatching / UserStore.saveProfile / MatchStore.saveMatch / 推し → 候補 / 推され → サマリ + Lambda Async 起動 | シナリオ 1 (Service) + シナリオ 2 (両側マッチング) |
| **Unit-D DM** | (RECV 側マッチ成立時に) DMService.generateUniquenessDMs を Unit-B から非同期起動 → 唯一性 DM 並列生成 | シナリオ 2 (推され側 → 推し向け唯一性 DM の初動) |
| **Unit-E Cross-cutting** | Cognito User Pool / shared-types (OshiCharacterParams / RecvCharacterParams) / AuditLogger | シナリオ 1, 2 共通 |

### 3.2 US-COM-02 誘導尋問自己分析と爆褒め

| Unit | 担当部分 | 主要シナリオ |
| --- | --- | --- |
| **Unit-A Frontend** | SelfAnalysisView (誘導尋問 → 要約 → 爆褒め SSE 受信) | シナリオ 1, 2 (UI) |
| **Unit-C Praise** | SelfAnalysisService (Phase A→B→C オーケストレーション) / SelfAnalysisAgent (AI-2) / PraiseAgent (AI-1) チェイン / AnalysisStore | シナリオ 1, 2, 3 (AI 出力品質保証 + Guardrail) |
| **Unit-E** | PromptGuardrail (二重保証) / packages/prompts (誘導尋問 + 爆褒め System Prompt) / AuditLogger / AuditLogStore | シナリオ 3 (NG ワード非混入) |

### 3.3 US-OSHI-01 唯一性 DM とヤキモキ演出 (ハード推し)

| Unit | 担当部分 | 主要シナリオ |
| --- | --- | --- |
| **Unit-A Frontend** | DMListView (受信表示 + 「💭」ヤキモキアイコン + 「仕方ない理由」通知 SSE 受信) | シナリオ 1, 2, 3 (UI) |
| **Unit-D DM** | DMService.listIncoming / DMRelayAgent.generateUniqueness (AI-3 並列生成) / DMService.triggerYakimoki / YakimokiAgent.generateExcuse (AI-5) / EventBridge Scheduler ルール作成 / DMStore | シナリオ 1, 2, 3 (Service+AI+Scheduler) |
| **Unit-E** | PromptGuardrail (NFR-ETH-04 実在個人模倣禁止 / NFR-ETH-06 偽装禁止) / AuditLogger / packages/prompts (DM relay + yakimoki) | シナリオ 1, 2 倫理ガードレール |

### 3.4 US-OSHI-02 候補提示と日次褒めで日常の彩り (ソフト推し)

| Unit | 担当部分 | 主要シナリオ |
| --- | --- | --- |
| **Unit-A Frontend** | CandidateCardView (3〜5 枚カード + 「推す」ボタン) | シナリオ 1 (UI) |
| **Unit-B Onboarding** | MatchingService.getCandidatesForOshi / recordOshi / MatchStore.incrementRanking | シナリオ 1 (候補提示 + 推す → ランキング) |
| **Unit-C Praise** | DailyPraiseService.getDailyPraise / PraiseAgent (AI-1 / インフレ褒め / ソフト推し向けトーン制御) | シナリオ 2 (日次褒め配信) |
| **Unit-E** | PromptGuardrail / AuditLogger / packages/prompts (daily praise トーン分岐) | 通常 |

### 3.5 US-RECV-01 感情労働全代行とゲーム化承認作業 (ハード推され)

| Unit | 担当部分 | 主要シナリオ |
| --- | --- | --- |
| **Unit-A Frontend** | DMListView (推され側: 「あなたを推している推し ◯名」サマリ + 承認待ち枠 + 承認/微修正/却下ボタン + 連続承認コンボ演出) | シナリオ 1, 2, 3, 4 (UI) |
| **Unit-D DM** | DMService.generateReplyCandidate (AI-4 感情労働代行返信) / approve / modify / reject / DMRelayAgent.generateReply / DependencyMeterService.bumpScore | シナリオ 1, 2, 3, 4 (Service+AI+承認ワークフロー) |
| **Unit-E** | PromptGuardrail (NFR-ETH-01,03,06 二重保証) / AuditLogger / NG ワード検出 / packages/prompts (DM relay) | シナリオ 1 (倫理ガードレール直書き) |

### 3.6 US-RECV-02 副業推されと釣れる行動の AI 提示 (ソフト推され)

| Unit | 担当部分 | 主要シナリオ |
| --- | --- | --- |
| **Unit-A Frontend** | UIShell (帰宅後 30 分タイムスロットで開く) + 行動提案リスト UI + 完了報告ボタン + 爆褒めアニメーション | シナリオ 1, 2 (UI) |
| **Unit-C Praise** | DailyPraiseService.submitActionCompletion (AI-1) / 行動提示生成 (PraiseAgent context=daily-recv-soft) / AnalysisStore.appendActionLog / DependencyMeterService | シナリオ 1, 2, 3 (Service+AI) |
| **Unit-E** | PromptGuardrail (NFR-ETH-02 ギャンブル誘導禁止 + NFR-ETH-03 自傷誘導禁止) / packages/prompts (action proposal トーン) | 倫理 |

### 3.7 US-RM-01 行動変容モード — 推され OS 再インストール (Roadmap)

| Unit | 担当部分 (将来) | 主要シナリオ |
| --- | --- | --- |
| (将来) Unit-A Frontend 拡張 | アシスタントキャラビジュアル選択 UI / 段階別タスク提案 UI / 爆褒めアニメ | 全シナリオ |
| **(Roadmap) Unit-RM** | BehaviorChangeService (段階遷移判定 1〜6) / CoachingAgent (AI-7 アシスタントキャラ人格) / AssistantCharacter エンティティ | シナリオ 1〜6 |
| (将来) Unit-E 拡張 | packages/prompts に coaching-prompts.ts 追加 (シナリオ 6 共通: 「あなたを変えています」発話禁止) | シナリオ 6 |

### 3.8 US-RM-02 行動変容モード — 推し OS 再インストール (Roadmap)

| Unit | 担当部分 (将来) | 主要シナリオ |
| --- | --- | --- |
| (将来) Unit-A Frontend 拡張 | 推し向けアシスタントキャラ UI / 段階別タスク提示 | 全シナリオ |
| **(Roadmap) Unit-RM** | BehaviorChangeService (推し OS) / CoachingAgent (推し向けトーン) / AssistantCharacter | シナリオ 1〜6 |
| (将来) Unit-E 拡張 | NFR-ETH-02 ギャンブル誘導禁止の境界制御 (見返り無し / 養い金額表示無し) | シナリオ 4 |

---

## 4. デモ動線 × Unit 担当 (5/30 予選会 3 分)

`stories.md` 付録 B のデモ動線対応表 (#1〜#6) を Unit 担当視点で再整理:

| シーン番号 | 時間 | Story | 担当 Unit | 連動 |
| --- | --- | --- | --- | --- |
| #1 | 0:00〜0:30 | US-COM-01 オンボーディング → モード選択 → キャラメイク → **両側マッチング起動** | **Unit-A + Unit-B** | Unit-D (RECV 起動時) + Unit-E |
| #2 | 0:30〜1:00 | US-COM-02 自己分析 → 要約 → 爆褒め | **Unit-A + Unit-C** | Unit-E |
| #3 | 1:00〜1:30 | US-OSHI-01 唯一性 DM 受信 → ヤキモキ演出 → 仕方ない理由通知 | **Unit-A + Unit-D** | Unit-E |
| #4 | 1:30〜1:45 | US-OSHI-02 ソフト推し裾野説明 (静止画でも可) | Unit-A | — |
| #5 | 1:45〜2:30 | US-RECV-01 推され視点切替 → AI が DM 自動生成 → 承認 → 爆褒め | **Unit-A + Unit-D** | Unit-E |
| #6 | 2:30〜2:45 | US-RECV-02 副業推され裾野説明 (静止画でも可) | Unit-A | — |
| 締 | 2:45〜3:00 | キーメッセージ + ロードマップ示唆 | プレゼンター | Unit-RM 設計言及 |

→ **デモ完結のためにすべての Unit が一定動作している必要がある**。Unit-E (横断) は 5/15 開発初日に最優先で整備する。

---

## 5. 4 名 → Unit 担当割当の選択肢 (5/15 当日確定用)

### 5.1 案 1 「1 人 1 Unit + 1 人横断」(推奨)

```
Miura     : Unit-? (例: Unit-A Frontend)
Ito       : Unit-? (例: Unit-B Onboarding)
Matsumoto : Unit-? (例: Unit-C Praise)
Kawano    : Unit-D + Unit-E 横断 (※ Unit-D は AI 並列が重く Unit-E と同時担当は注意)
```

または:

```
Miura     : Unit-A Frontend
Ito       : Unit-B Onboarding
Matsumoto : Unit-C Praise
Kawano    : Unit-D DM (AI 中核)
全員 : Unit-E 横断 (持ち回り or リード 1 名 + 全員レビュー)
```

### 5.2 案 2 「ペアで 2 Unit ずつ」

```
ペア 1 (Miura + Ito)        : Unit-A + Unit-B
ペア 2 (Matsumoto + Kawano) : Unit-C + Unit-D
全員 : Unit-E 横断
```

### 5.3 案 3 「スキル別固定」

```
Miura     : Unit-A Frontend (React 担当)
Ito       : Unit-B + Unit-C のサーバ層
Matsumoto : Unit-D AI 並列担当
Kawano    : Unit-E 横断 + IaC + CDK
```

→ 5/15 書類審査結果発表後、4 名で話し合って確定。

---

## 6. ストーリーポイント × Unit ごとの工数見積 (粗見積もり)

| Unit | 総 SP (S=1, M=3, L=5, XL=8) | 主担当ストーリーから算出 | 備考 |
| --- | --- | --- | --- |
| Unit-A Frontend | M (3) | 全 6 ストーリーの UI 部分 | レトロ UI 実装は時間要 |
| Unit-B Onboarding | M (3) | US-COM-01 + US-OSHI-02 候補 | DDB Single-Table の習熟度次第 |
| Unit-C Praise | L (5) | US-COM-02 + US-OSHI-02 日次 + US-RECV-02 + SSE 実装 | Bedrock streaming + SSE が技術的山場 |
| Unit-D DM | L (5) | US-OSHI-01 + US-RECV-01 + EventBridge | Bedrock 並列 + 状態機械 |
| Unit-E Cross-cutting + IaC | M (3) | 横断 / IaC | CDK Stack 数が多い |
| (Roadmap) Unit-RM | XL (8) | US-RM-01, 02 | 5/30 予選後着手見込み |

**MVP 合計**: M(3) + M(3) + L(5) + L(5) + M(3) = **19 SP** ≈ 約 2 週間 (4 名並行) → 5/15〜5/29 の 14 日間でぎりぎり収まる想定。

---

## 7. CONSTRUCTION フェーズへの引き継ぎ (Per-Unit Loop の入力)

各 Unit は CONSTRUCTION フェーズで以下 5 ステージを実行する (`execution-plan.md` 準拠):

| Stage | Unit-A | Unit-B | Unit-C | Unit-D | Unit-E | (Roadmap) Unit-RM |
| --- | --- | --- | --- | --- | --- | --- |
| Functional Design | ✅ Standard | ✅ Standard | ✅ Standard | ✅ Standard | ✅ Standard | (将来) |
| NFR Requirements | ✅ Standard | ✅ Standard | ✅ Standard | ✅ Standard | ✅ Standard | (将来) |
| NFR Design | ✅ Standard | ✅ Standard | ✅ Standard | ✅ Standard | ✅ Standard | (将来) |
| Infrastructure Design | ✅ Standard (Amplify) | ✅ Standard (Lambda + DDB) | ✅ Standard (Lambda + Bedrock) | ✅ Standard (Lambda + EventBridge + Bedrock) | ✅ Standard (CDK Stack 全部) | (将来) |
| Code Generation | ✅ Always | ✅ Always | ✅ Always | ✅ Always | ✅ Always | (将来) |

→ 全 Unit 完了後 **Build and Test を 1 回実行** (統合テスト + プロンプトテスト + 倫理ガードレール検証 + デモ台本シーン #1〜#6)

---

## 8. 提出物としての位置づけ (5/10 書類審査)

要件書 8.2 で必須とされている 5/10 提出物のうち、本ドキュメントは:

```
aidlc-docs/inception/application-design/unit-of-work*.md
```

の一部として、`unit-of-work.md` / `unit-of-work-dependency.md` と合わせて 3 ファイル構成で提出される。

**審査観点との対応**:

| 審査観点 | 本ドキュメントでの対応 |
| --- | --- |
| Unit 分解の適切さ | 5 Unit + Roadmap の境界、依存方向、ビルド順、Story マッピングを明示 |
| ドキュメント品質 | Mermaid 図 + マトリクス + シナリオ別内訳 で多面的に表現 |
| ビジネス意図の明確さ | デモ動線対応表で「3 分でこう見せる」を示す |
| 創造性とテーマ適合性 | Roadmap Unit-RM (適応型自己実現 / OS 再インストール) で本サービスの本質的回答を示す |
