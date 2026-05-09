# Unit of Work Plan

**作成日**: 2026-05-09
**フェーズ**: AI-DLC Inception / Units Generation
**深さレベル**: Standard
**生成対象**:
- `aidlc-docs/inception/application-design/unit-of-work.md`
- `aidlc-docs/inception/application-design/unit-of-work-dependency.md`
- `aidlc-docs/inception/application-design/unit-of-work-story-map.md`

> **役割**: AI はテックリードとして、`requirements.md` / `stories.md` / `application-design.md` を入力に、システムを 4 名チーム (5/15〜5/30 実装期間) で並行開発可能な独立 Unit に分解する。

---

## 入力ソース

- `aidlc-docs/inception/requirements/requirements.md` (4 件追補済)
- `aidlc-docs/inception/user-stories/personas.md` / `stories.md` (MVP 6 + Roadmap 2)
- `aidlc-docs/inception/application-design/application-design.md` (Section 8 で Unit 候補 4 + 横断 を提示済)
- `aidlc-docs/inception/plans/execution-plan.md` (Standard 深さ確定)

---

## 全体ステップチェックリスト

### Part 1: Planning
- [x] (Step 1) 質問書 Q1〜Q6 への回答受領 (2026-05-09T15:50 / 全問 AI 推奨案フル採用)
- [x] (Step 2) 回答曖昧性分析 → 残らず
- [x] (Step 3) Plan 最終承認 (2026-05-09T15:55)

### Part 2: Generation
- [x] (Step 4) `unit-of-work.md` (5 Unit + Roadmap / Code Organization 戦略 / チーム編成選択肢 / NFR Unit 別対応マップ)
- [x] (Step 5) `unit-of-work-dependency.md` (依存マトリクス / Mermaid レイヤー図 / 通信パターン / ビルド順 / 統合テスト境界)
- [x] (Step 6) `unit-of-work-story-map.md` (8 ストーリー × Unit マトリクス / ストーリー別内訳 / デモ動線対応 / 4 名割当案 / SP 見積)
- [x] (Step 7) ストーリー網羅性検証 — 全 8 ストーリー主担当 Unit 割当 + Unit-E 横断対応確認 (story-map Section 2)
- [x] (Step 8) `aidlc-state.md` を Stage 7 ✅ Completed に更新 / ユーザー承認受領 (2026-05-09T16:45) → INCEPTION フェーズ完了

---

# 質問書 (Q1〜Q6)

> **記入方法**: 各 Q の `[Answer]:` に回答記入。「推奨で OK」と書くだけで進められる構成。

---

## Q1. Unit 数と境界

**問い**: システムを何個の Unit に分けますか?

- **A**: 4 Unit — Application Design Section 8 のまま (Unit-A フロント / Unit-B キャラメイク・マッチ / Unit-C 自己分析・爆褒め / Unit-D DM・ヤキモキ)。横断は各 Unit 内に分散
- **B**: **5 Unit** — A〜D + Unit-E 横断 (PromptGuardrail / AuditLogger / IaC / 共通型) を独立 Unit 化
- **C**: 6 Unit — B + Roadmap (Unit-RM 行動変容モード) を独立 Unit 化
- **D**: 単一 Unit (モノリス) — 4 名で全員同じ Unit に貢献
- **E**: その他 (自由記述)

**AI 推奨**: **B (5 Unit)**。理由:
- 横断機能 (Guardrail / AuditLogger / IaC) を Unit-E として独立させると、4 名で並行実装する際に「**横断は誰か 1 名が責任**、各 Unit はそれを利用」という明快な役割分担が成立
- Roadmap (Unit-RM) は **書類審査時点では設計のみ示し、実装単位として独立 Unit にはしない** (5/30 予選会の MVP に含めないため)。設計上のロードマップ枠として stories.md / application-design.md / unit-of-work.md に記述するに留める
- A (4 Unit) は横断機能の所有が曖昧化、D (モノリス) は 4 名並行実装の効率を落とす

[Answer]: 

---

## Q2. デプロイモデル / コード構成 (Greenfield)

**問い**: コード構成と CI/CD はどうしますか?

- **A**: **モノレポ + 単一 CI/CD** — 1 つの GitHub リポジトリ内に Unit ごとのディレクトリ。pnpm workspace + Turborepo (TS) で依存解決、CI は変更検知して影響範囲のみビルド
- **B**: マルチレポ + 個別 CI/CD — Unit ごとに別リポジトリ。リリース独立だがリポジトリ数 = 5
- **C**: モノレポ + Unit 別パイプライン — 1 リポジトリ内で Unit ごとに独立 CI/CD パイプライン
- **D**: その他 (自由記述)

**AI 推奨**: **A (モノレポ + 単一 CI/CD)**。理由:
- ハッカソン (5/15〜5/30) の短期間で 4 名が並行実装する場合、リポジトリは 1 つに集約した方がコードレビュー / 依存解決 / 共通型 (DTO / プロンプト) の取り回しが楽
- モノレポでも Turborepo / Nx などで Unit 別ビルド + 影響範囲ビルドが可能 → デプロイ独立性は確保
- マルチレポは権限管理・PR 連携で時間を食う

[Answer]: 

---

## Q3. ディレクトリ構造 (モノレポ前提)

**問い**: モノレポ採用時のディレクトリ構造はどうしますか?

- **A**: **`packages/` ベースのワークスペース構造** — `packages/{unit-a-frontend, unit-b-onboarding, unit-c-self-analysis, unit-d-dm, unit-e-shared, infra}/` の 6 ディレクトリ + ルート `package.json`
- **B**: `apps/` + `packages/` 二層構造 — `apps/` にデプロイ対象 (frontend / backend lambda)、`packages/` に shared types / guardrail / audit
- **C**: フラット構造 — Unit ごとのディレクトリのみ (workspace なし)
- **D**: その他 (自由記述)

**AI 推奨**: **B (apps/ + packages/ 二層)**。理由:
- `apps/frontend` (Amplify) と `apps/backend-*` (Lambda) は **デプロイ単位**、`packages/shared`(型/プロンプト共通) と `packages/guardrail`(横断) と `packages/audit-logger`(横断) は **共有ライブラリ**として明示的に分離する方が責務が明快
- pnpm workspace + Turborepo の標準パターンと整合
- `infra/` (CDK) はトップレベル別ディレクトリで分離

[Answer]: 

---

## Q4. Unit 間通信パターン

**問い**: Unit 間 (Lambda 間) の通信はどう実装しますか?

- **A**: **基本は in-process 関数呼び出し (同一 Lambda にまとめる)、跨ぐときは EventBridge 非同期** — Domain Service 群は同一 Lambda にデプロイ、AI Adapter は別 Lambda、Front から API Gateway 経由
- **B**: 全 Service を別 Lambda + REST/HTTP 同期呼び出し — Lambda 間通信のオーバーヘッド大
- **C**: 全 Service を別 Lambda + EventBridge 非同期 — 結果整合性で UX が落ちる可能性
- **D**: その他 (自由記述)

**AI 推奨**: **A (in-process + 必要時 EventBridge)**。理由:
- ハッカソン MVP の同時 10 ユーザー (NFR-PERF-02) では Lambda 間通信のレイテンシが NFR-PERF-01 (3 秒応答) を圧迫
- Onboarding → Matching → DM の連鎖は基本 in-process で速度を稼ぎ、ヤキモキ遅延配信のみ EventBridge Scheduler を使う
- AI Adapter は同一 Lambda で OK (Bedrock SDK 呼び出しがメイン)

[Answer]: 

---

## Q5. チーム編成 / Unit 担当割当 (4 名)

**問い**: 4 名 (Miura / Ito / Matsumoto / Kawano) と 5 Unit をどう紐付けますか?

- **A**: **1 人 1 Unit + 1 人が横断 (Unit-E)** を担当、Roadmap は全員でレビュー
- **B**: 1 Unit 複数人 (ペアプロ) — Unit 数 ≤ 4 になるまで統合
- **C**: スキル別 (フロント / バックエンド / インフラ) で固定 — Unit 横断
- **D**: 5/15 書類審査結果発表後に正式決定 — Plan 上は「未確定」
- **E**: その他 (自由記述)

**AI 推奨**: **D (5/15 結果発表後に確定)**。理由:
- 書類審査通過しなかった場合は実装不要のため、5/9 時点で具体的な担当を確定する必要はない
- ただし `unit-of-work.md` 内に「**4 名割当の選択肢**」と「Unit ごとの想定スキル要件 (フロント React / バックエンド TypeScript / Bedrock 知識 / IaC)」を明記し、5/15 当日にすぐ割り当て確定できる状態にしておく

[Answer]: 

---

## Q6. 共有モジュール戦略 (`packages/shared`)

**問い**: 横断 Unit (Unit-E) と各 Unit が共有する型・関数をどう構造化しますか?

- **A**: 単一 `packages/shared` に全部 — 型・プロンプト共通文・Guardrail / AuditLogger を 1 パッケージに
- **B**: **責務別に複数パッケージ** — `packages/shared-types` (DTO) / `packages/guardrail` / `packages/audit-logger` / `packages/prompts` (System Prompt 共通断片)
- **C**: 各 Unit 内に重複コピー — DRY 違反だがビルド独立性◎
- **D**: その他 (自由記述)

**AI 推奨**: **B (責務別に複数パッケージ)**。理由:
- `packages/guardrail` と `packages/audit-logger` は Unit-E の主体。複数 Unit から import される
- `packages/shared-types` は DTO (User / DM / Match / etc.) の共通定義。Unit 間の型整合を保証
- `packages/prompts` は System Prompt の共通断片 (倫理ルール / 用語禁止リスト等) を集約 → Unit-C, Unit-D, Unit-RM が利用
- これにより各 Unit はピンポイントに必要な依存だけを宣言できる

[Answer]: 

---

# 推奨案採用時の Unit 構成サマリ

| Unit ID | Unit 名 | 主担当 Story | 主要コンポーネント | デプロイ単位 | 想定スキル | 想定 SP |
| --- | --- | --- | --- | --- | --- | --- |
| **Unit-A** | Frontend Foundation + UIShell + 認証 | 全 Story の UI シェル | UIShell, 全 View, Cognito 統合, レトロ CSS | Amplify (CloudFront + S3) | React / TS / CSS / UX | M |
| **Unit-B** | Onboarding + Matching | US-COM-01 (両側マッチ含) / US-OSHI-02 候補表示 | OnboardingService, MatchingService, UserStore, MatchStore, CharacterMakerView, CandidateCardView | Lambda × 1〜2 (apps/backend-onboarding) | TS / DDB / Bedrock 軽 | M |
| **Unit-C** | Self-Analysis + Daily Praise | US-COM-02, US-OSHI-02 (日次褒め), US-RECV-02 | SelfAnalysisService, DailyPraiseService, SelfAnalysisAgent, PraiseAgent, AnalysisStore, SelfAnalysisView | Lambda × 1 (apps/backend-praise) | TS / Bedrock streaming / SSE | L |
| **Unit-D** | DM (唯一性 + 感情労働代行 + ヤキモキ) | US-OSHI-01, US-RECV-01 | DMService, DMRelayAgent, YakimokiAgent, DependencyMeterService, DMStore, DMListView | Lambda × 2 (apps/backend-dm + apps/scheduler-yakimoki) | TS / Bedrock 並列 / EventBridge | L |
| **Unit-E** | Cross-cutting + IaC | 横断 / 全 Story から利用 | PromptGuardrail, AuditLogger, AuditLogStore, Cognito 設定, CDK Stack 全部 | packages (共有) + infra/ (CDK デプロイ) | TS / IaC / Bedrock Guardrails / セキュリティ | M |
| (Roadmap) | Unit-RM 行動変容モード | US-RM-01, 02 | BehaviorChangeService, CoachingAgent, AssistantCharacter | (将来) | (将来) | XL (5/30 までは含めない) |

---

# 推奨案採用時のディレクトリ構造

```
oshi-osare/
├── package.json (workspace ルート / pnpm)
├── pnpm-workspace.yaml
├── turbo.json
├── apps/
│   ├── frontend/                    ← Unit-A
│   │   ├── package.json
│   │   ├── src/components/UIShell.tsx
│   │   ├── src/views/CharacterMakerView.tsx
│   │   ├── src/views/SelfAnalysisView.tsx
│   │   ├── src/views/DMListView.tsx
│   │   └── src/views/CandidateCardView.tsx
│   ├── backend-onboarding/          ← Unit-B
│   │   ├── package.json
│   │   └── src/handlers/{onboarding,matching}/...
│   ├── backend-praise/              ← Unit-C
│   │   ├── package.json
│   │   └── src/handlers/{self-analysis,daily-praise}/...
│   ├── backend-dm/                  ← Unit-D
│   │   ├── package.json
│   │   └── src/handlers/{dm-list,dm-relay,approve}/...
│   └── scheduler-yakimoki/          ← Unit-D の一部 (EventBridge Scheduler 起動)
│       ├── package.json
│       └── src/handlers/yakimoki.ts
├── packages/                         ← Unit-E 主体
│   ├── shared-types/
│   ├── guardrail/                   ← PromptGuardrail
│   ├── audit-logger/                ← AuditLogger
│   └── prompts/                     ← System Prompt 共通断片
├── infra/                            ← Unit-E の IaC
│   ├── package.json (CDK)
│   ├── lib/
│   │   ├── cognito-stack.ts
│   │   ├── api-gateway-stack.ts
│   │   ├── lambda-stack.ts
│   │   ├── ddb-stack.ts
│   │   ├── s3-audit-stack.ts
│   │   └── bedrock-guardrails-stack.ts
│   └── bin/oshi-osare.ts
└── .github/workflows/
    └── ci.yml                       ← 単一 CI/CD パイプライン (Turborepo 影響範囲ビルド)
```

---

# Story → Unit マッピング (推奨案ベース)

| Story | Unit |
| --- | --- |
| US-COM-01 オンボーディング & 褒めキャラメイク + 両側マッチング起点 | **Unit-A (UI) + Unit-B (Service)** + Unit-E (Cognito 横断) |
| US-COM-02 誘導尋問自己分析 + 爆褒め | **Unit-A (UI) + Unit-C (Service+AI)** + Unit-E (Guardrail/Audit) |
| US-OSHI-01 唯一性 DM + ヤキモキ (ハード推し) | **Unit-A (UI) + Unit-D (Service+AI+Scheduler)** + Unit-E (Guardrail/Audit) |
| US-OSHI-02 候補提示 + 日次褒め (ソフト推し) | **Unit-A (UI) + Unit-B (候補) + Unit-C (日次褒め)** |
| US-RECV-01 感情労働全代行 (ハード推され) | **Unit-A (UI) + Unit-D (Service+AI)** + Unit-E (Guardrail/Audit) |
| US-RECV-02 副業推され + 釣れる行動 (ソフト推され) | **Unit-A (UI) + Unit-C (Service+AI)** |
| US-RM-01 行動変容 — 推され OS 再インストール (Roadmap) | (将来) Unit-RM |
| US-RM-02 行動変容 — 推し OS 再インストール (Roadmap) | (将来) Unit-RM |

---

# 完了基準

- 全 `[Answer]:` タグが埋まっている
- AI が回答曖昧性分析 → 残らないことを確認
- ユーザーが Plan 承認
- audit.md に Q&A + 承認応答を記録
