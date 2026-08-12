# Contextory ドキュメント

Contextoryは、PMが見た画面と聞いた会話をローカルに収集し、AIの整理とユーザー確認を通して、案件・タスク単位の信頼できる業務文脈へ育てるデスクトップアプリです。

自分だけが使う非公開ツールとして開発し、一般公開、App Store配布、複数ユーザー利用は想定しません。

通常は私用Macでビルドし、会社Macにはビルド済みアプリバイナリを直接提供して実行します。直接配布がGatekeeper、署名、互換性などで安定しない場合は、会社MacへXcodeを導入してローカルビルドするフォールバックを使用します。

## 現在の方針

- 画面キャプチャ、音声録音、画面録画によるInput収集を簡単にする。
- 初期対象OSはmacOSのみに限定する。
- macOSメニューバーに常駐し、画面を邪魔せず、すぐ開始・停止できるようにする。
- 取得したSourceを即時にAIへ渡し、要約とMarkdownを生成する。
- ユーザーが明示的に取得・取り込みしたSourceは保存後に自動処理し、失敗を状態表示する。
- 過去のSourceとの関連を調べ、案件・タスク候補へ自動グルーピングする。
- 自動グルーピングが誤っている場合はユーザーが修正できるようにする。
- 案件が文脈を持ち、タスクが対応を持ち、Sourceが根拠になる。
- 原本、AIによる解釈、ユーザー確認済み情報を分離する。
- Sourceを複数Contextで再利用し、解析目的ごとのAnalysisを上書きせず追加する。
- 画像・PDF・テキストを手動取り込みし、AIのタスク分類候補を確認可能な状態で保存する。
- 原本はローカルに保存し、Claude Codeへ即時送信する対象と範囲はユーザーが事前に許可する。
- ユーザーが最低1日1回、要約とグルーピングを確認・修正して確定する。
- 撮り溜めたコンテンツの一覧管理・詳細レビューUIは、常駐型取得エージェントとは別に設計する。
- 常駐メニューはInput専用、将来のタスク整理画面は確認・補足・関連付け・再解析専用とし、相互の操作を混在させない。
- 将来のExcel、PowerPoint、Jira、Backlog、Slack出力は確認済み文脈から生成する。

## 文書一覧

- [プロダクトビジョン](01-vision/product-vision.md)
- [MVPスコープ](02-requirements/mvp-scope.md)
- [Source・Group・Task・Output要件](02-requirements/source-group-task-output.md)
- [安全・プライバシー原則](02-requirements/safety-principles.md)
- [システム概要](03-design/system-overview.md)
- [データモデル](03-design/data-model.md)
- [開発フェーズ](04-roadmap/development-phases.md)
- [未決事項](05-decisions/open-questions.md)
- [ADR一覧](06-adr/README.md)
- [ADR-001 Local VaultのファイルとSQLite構成](06-adr/ADR-001-local-vault-storage.md)
- [ADR-002 初期対象OSをmacOSに限定](06-adr/ADR-002-macos-first.md)
- [ADR-003 macOSネイティブ・単一ユーザー構成](06-adr/ADR-003-native-single-user-stack.md)
- [ADR-004 私用Macでビルドして会社Macへバイナリ提供](06-adr/ADR-004-private-build-binary-delivery.md)
- [ADR-005 システム音声とマイク音声を別原本として保存](06-adr/ADR-005-separate-system-and-microphone-audio.md)
- [ADR-006 Source・Context・Analysisを分離し派生結果を追加保存](06-adr/ADR-006-append-only-context-analysis.md)
- [ADR-007 TaskとOutputの根拠来歴をID参照で保持](06-adr/ADR-007-task-output-lineage.md)
- [ADR-008 音声・動画をローカル前処理してからAIへ渡す](06-adr/ADR-008-local-media-preprocessing.md)
- [PoC一覧](07-poc/README.md)
- [Phase 0 配布・権限スパイク結果](07-poc/phase-0-distribution-permissions-result.md)

## リポジトリ境界

- Docs: `/Users/mikey/workspace/vault/personal-vault/20-projects/contextory/docs`
- App: `/Users/mikey/workspace/apps/contextory-app`
- 実業務データ: 会社PCのローカルアプリデータ領域。Git管理しない。

## 現在の状態

Phase 0の配布・権限スパイクとPhase 1のInput Captureは完了しました。Phase 2の自動解析Queueに続き、Inputと分離したタスク整理画面の最初の縦切りとして、Analysis確認、根拠Source、Task作成、Task来歴、失敗解析の手動再実行を実装しました。次はSourceを蓄積するGroup整理、出典URL、Analysis Sourceに紐づくAI対話を優先し、その後に派生Source生成へ進みます。
