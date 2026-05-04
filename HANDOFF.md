# Claude Code 引き継ぎドキュメント

このリポジトリは AWS Summit Japan 2026 AI-DLC ハッカソン参加用です。
このファイルを読んだら、下記の「やってほしいこと」を順番に実行してください。

## プロジェクト概要

- **チーム名**: t4g.lazy.4xlarge
- **メンバー (4 名)**: Miura Kazuki / Ito Nari / Matsumoto Shogo / Kawano Shinichiro
- **テーマ**: 「人をダメにする」サービス
- **使用方法論**: AI-DLC (AI-Driven Development Lifecycle)

## 重要な期日

|日付           |内容                                       |
|-------------|-----------------------------------------|
|2026/05/10   |応募締切 + 公開 Git リポジトリに Inception フェーズ成果物を公開|
|2026/05/12 正午|運営に Git リポジトリ URL を連絡                    |
|2026/05/15   |書類審査結果発表                                 |
|2026/05/30   |予選会 @麻布台ヒルズ AWS オフィス (MVP デモ + プレゼン)     |
|2026/06/26   |決勝 @幕張メッセ AWS Summit Japan 2026          |

## 書類審査の観点

- ビジネス意図 (Intent) の明確さ
- 創造性とテーマ適合性
- Unit 分解の適切さ
- ドキュメント品質

## やってほしいこと

### Step 1: AI-DLC ワークフローのセットアップ

`awslabs/aidlc-workflows` の最新リリースから ai-dlc-rules を取得して、Claude Code 用に配置してください。

1. GitHub API で最新リリースを取得し、`ai-dlc-rules-vX.X.X.zip` をダウンロード
   
   ```bash
   curl -sL https://api.github.com/repos/awslabs/aidlc-workflows/releases/latest \
     | grep "browser_download_url.*ai-dlc-rules.*zip" \
     | cut -d '"' -f 4 \
     | xargs curl -sL -o /tmp/ai-dlc-rules.zip
   ```
1. zip を一時ディレクトリに展開
1. Claude Code 用にプロジェクトルートに配置:
- `CLAUDE.md` (なければ `AGENTS.md` から作成、もしくは公式の指示に従う)
- `.aidlc-rule-details/` ディレクトリ
1. `.gitignore` に必要なら `.claude/settings.local.json` を追加
1. 配置後、ディレクトリ構造を確認

参考リンク:

- リポジトリ: https://github.com/awslabs/aidlc-workflows
- セットアップガイド: https://github.com/awslabs/aidlc-workflows/blob/main/README.md
- 使い方ガイド: https://github.com/awslabs/aidlc-workflows/blob/main/docs/WORKING-WITH-AIDLC.md

### Step 2: README.md の作成

リポジトリのトップに README.md を作成してください。最低限:

- チーム名 t4g.lazy.4xlarge とメンバー紹介
- ハッカソン参加用リポジトリである旨
- AI-DLC を使っていること
- `aidlc-docs/` に成果物が入る旨

### Step 3: Inception フェーズの開始

セットアップが完了したら、AI-DLC の Inception フェーズを開始してください。

テーマは「人をダメにする」サービスで、まだアイデアは決まっていません。Inception フェーズの中でアイデア出しから一緒にやりたいです。提示されたアイデア例:

- 今日の服・ご飯・行動を全部決定してくれる AI エージェント
- 無限先延ばしくん: タスクを先延ばしする言い訳を生成し続ける
- 仕事中に定期的に猫の画像を出してきて癒してくれるアプリ
- 長時間仕事をしていると作業を止めてゲームを開始する Kiro Powers

これらに引っ張られず、4 名のチームでオリジナルのアイデアを出したいので、Inception フェーズの最初で複数案をブレストするところから始めてください。

### Step 4: Inception 成果物を `aidlc-docs/` に生成

書類審査の対象になるのは以下です (全部含む必要はないが揃っているほど良い):

- 要件分析
- ユーザーストーリー
- アプリケーション設計
- Unit of Work 計画

`aidlc-state.md` で Inception フェーズ完了状態にしてください。

### Step 5: コミット & プッシュ

5/10 までに公開リポジトリに push 完了が必須です。コミットメッセージは Conventional Commits 形式 (feat:, docs:, chore: 等)、日本語で記述してください。

## 制約・注意事項

- AWS アカウント・利用費は参加者持ち (クレジット提供なし)
- AI エージェント実装は必須ではない (4/24 規約改定で削除)
- 外部 API 利用 OK、ただし AWS に同等サービスがあればそれを推奨
- Kiro 利用は推奨だが必須ではない (Claude Code で OK)
- 「人をダメにする」は公序良俗の範囲内で

## 開発ルール (組織方針)

- ドキュメント・コミットメッセージ・PR は日本語
- ブランチ名: 単一文字の前にハイフン禁止 (例: `fix-japanese-encoding` は OK、`fix-f-encoding` は NG)
- TDD 基本 (Red → Green → Refactor)
- セキュリティ: シークレットをハードコードしない、XSS/SQLi 等を作り込まない
- 動作確認・型チェック・テストを通してから完了報告

## 参考リンク

- ハッカソン応募ページ: https://bit.ly/4mPkzy8
- 説明会動画: https://youtu.be/1G5DJQwxTig
- AWS Summit Japan 2026: https://aws.amazon.com/jp/summits/japan/
