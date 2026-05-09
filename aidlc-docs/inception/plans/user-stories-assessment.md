# User Stories Assessment

**作成日**: 2026-05-09
**フェーズ**: AI-DLC Inception / User Stories — Part 1 Planning Step 1
**判定**: ✅ **Execute User Stories** (実行する)

---

## 1. Request Analysis

| 項目 | 値 |
| --- | --- |
| Original Request | AWS Summit Japan 2026 AI-DLC ハッカソン参加。Greenfield で「人をダメにする」サービス (推されと推し) を MVP まで開発し、5/10 までに Inception 成果物を提出 |
| User Impact | **Direct** — エンドユーザーがフロントエンドで直接対話するゲーム体験 (推し側 / 推され側 の両モード) |
| Complexity Level | **Complex** — 両面市場 + 2 ペルソナ + AI 感情労働代行 + ヤキモキ演出 + 倫理ガードレール (NFR-ETH-01〜08) |
| Stakeholders | チーム t4g.lazy.4xlarge (4 名: Miura / Ito / Matsumoto / Kawano)、AI-DLC ハッカソン審査員、想定ユーザー (推し側 / 推され側) |

---

## 2. Assessment Criteria Met

### 2.1 High Priority Indicators (該当多数)

- [x] **New User Features**: キャラメイク / 自己分析 / 日次褒め / マッチング / 唯一性 DM / ヤキモキ演出 など、すべて新規ユーザー向け機能
- [x] **User Experience Changes**: 既存ワークフローではなく完全新規 UX (推し側 / 推され側 の双方向 UX を同居)
- [x] **Multi-Persona Systems**: 「推し側」「推され側」という構造的に異なる 2 ペルソナを同一プラットフォームで提供
- [x] **Complex Business Logic**: 「褒めたおすループ」(自己分析 → 褒め → 行動 → 褒め) + 2 モード (行動かわらない / 行動変容) + 倫理 8 制約 (NFR-ETH-01〜08) + ヤキモキ演出ロジック
- [x] **Cross-Team Projects**: 4 名チームで分担実装 → 共通理解のためにストーリー化が必須
- [x] **Customer-Facing APIs**: AI への「褒め文生成 / 誘導尋問 / DM 生成 / 感情労働代行」という顧客向けの AI API が中核

### 2.2 Medium Priority — N/A
High Priority に多数該当しているため評価不要。

### 2.3 Skip Conditions — 全て不該当
- ❌ Pure Refactoring: Greenfield のため該当しない
- ❌ Isolated Bug Fixes: 該当しない
- ❌ Infrastructure Only: ユーザー向け機能が中心のため該当しない
- ❌ Developer Tooling: 該当しない
- ❌ Documentation Only: 該当しない

---

## 3. Expected Benefits

User Stories ステージを実行することで以下の便益が得られる:

1. **要件書の解像度を上げる**: requirements.md は機能要件 (FR-MODE / FR-COM / FR-OSHI / FR-RECV / FR-AI / FR-MATCH) を機能カタログ的に列挙しているが、ペルソナ視点の物語化が未実施。ストーリー化により「誰が・なぜ・どんな状況で・どう使うか」が明示される
2. **審査観点との直結**: 審査観点「ビジネス意図 (Intent) の明確さ / 創造性とテーマ適合性」はペルソナとストーリーで最も伝わりやすい (5/10 提出物の必須項目)
3. **AI 任せ範囲の抽象化**: 各ストーリーで「AI が代行する責務 (DM 文生成 / 誘導尋問 / 感情労働代行)」を明示すると、Application Design ステージ (Bedrock AgentCore コンポーネント分割) の入力が明確になる
4. **倫理ガードレールのストーリー単位確認**: 各ストーリーの受入基準に「NFR-ETH-01〜08 のどれが効くか」を結びつけることで、ガードレールの抜け漏れ検知が早まる
5. **デモ 3 分台本の素材化**: 「キャラメイク → 自己分析 → 爆褒め → 次の一手」のデモ動線が、そのままストーリー優先順位として活用できる
6. **テスト戦略との接続**: NFR-QA-02 (プロンプトテスト / デモ台本テストケース化) と各ストーリーの受入基準を結合できる
7. **実装分担の根拠**: 4 名で 5/15〜5/30 に MVP を実装するため、Independent / Small なストーリー単位での分担が必要
8. **行動変容モード (ロードマップ) との境界明確化**: MVP に含めない「行動変容モード」のストーリーをロードマップとして残し、MVP 範囲を逆説的に締める

---

## 4. Decision

| 項目 | 判定 |
| --- | --- |
| **Execute User Stories** | **Yes** |
| **Depth Level** | **Standard** (打ち合わせメモに 9 ストーリーのドラフトが既に存在し、骨格は固まっているため) |
| **Approach Hint** | ペルソナベース + ジャーニーベースのハイブリッド (Story Plan で最終確定) |

### Reasoning

High Priority 指標 6 件すべてに該当し、Skip Condition には 1 件も該当しない。さらに 5/10 提出物の必須項目 (`stories.md` / `personas.md`) に明示されているため、本ステージのスキップは選択肢に存在しない。打ち合わせメモ Section 12.2 に 9 ストーリー (US-01〜US-09) のドラフトと、Section 13.4 に審査差別化視点のストーリー骨子があり、それを起点に Standard 深さで仕上げるのが効率的。

---

## 5. Expected Outcomes

- **`personas.md`**: 推し側 / 推され側 の 2 ペルソナ + (必要なら) サブペルソナ。年齢・属性・利用シーン・動機・不安・利用頻度・課金可能性を明示
- **`stories.md`**: 9〜12 ストーリー (共通 / 推し側 / 推され側 / ロードマップ参考)。各ストーリーに INVEST 適合、受入基準 (Given-When-Then or Done 条件)、紐付く FR / NFR ID、紐付くペルソナ ID を記載
- **デモ動線との対応表**: 3 分デモのシーン → 対応ストーリー → 対応ペルソナ の俯瞰表
- **倫理ガードレール対応表**: 各ストーリー × NFR-ETH-01〜08 のクロスチェック (リスクの高いストーリーのみピックアップ)

---

## 6. Constraints (本ステージ全体に効く既存決定)

- **用語制約**: 「ヒモ」「パトロン」は使用禁止 (NFR-ETH-08)。打ち合わせメモ 13.3.3 の `US-Patron` / `US-Himo` は本ステージでは「推し」「推され」表記で再定義する
- **MVP 境界**: 行動かわらないモードのみ実装。行動変容モードのストーリーは「ロードマップ参考」として残すが、受入基準は省略可
- **18 歳未満不可 (NFR-ETH-05)**: ペルソナの年齢設定で必ず満たす
- **アクセシビリティ**: 日本語のみ (NFR-A11Y-01)、レトロ UI 前提 (NFR-A11Y-02)
