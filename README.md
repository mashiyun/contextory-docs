# Contextory ドキュメント

Contextoryは、PMが見た画面と聞いた会話をローカルに収集し、AIの整理とユーザー確認を通して、案件・タスク単位の信頼できる業務文脈へ育てるデスクトップアプリです。

自分だけが使う非公開ツールとして開発し、一般公開、App Store配布、複数ユーザー利用は想定しません。

通常は私用Macでビルドし、会社Macにはビルド済みアプリバイナリを直接提供して実行します。直接配布がGatekeeper、署名、互換性などで安定しない場合は、会社MacへXcodeを導入してローカルビルドするフォールバックを使用します。

## 現在の方針

- 画面キャプチャ、音声録音、画面録画によるInput収集を簡単にする。
- 初期対象OSはmacOSのみに限定する。
- macOSメニューバーに常駐し、画面を邪魔せず、すぐ開始・停止できるようにする。
- 取得したSourceを即時にAIへ渡し、要約とMarkdownを生成する。
- 事前許可されたSourceは自動処理し、失敗時は手動で再実行できるようにする。
- 過去のSourceとの関連を調べ、案件・タスク候補へ自動グルーピングする。
- 自動グルーピングが誤っている場合はユーザーが修正できるようにする。
- 案件が文脈を持ち、タスクが対応を持ち、Sourceが根拠になる。
- 原本、AIによる解釈、ユーザー確認済み情報を分離する。
- 原本はローカルに保存し、Claude Codeへ即時送信する対象と範囲はユーザーが事前に許可する。
- ユーザーが最低1日1回、要約とグルーピングを確認・修正して確定する。
- 撮り溜めたコンテンツの一覧管理・詳細レビューUIは、常駐型取得エージェントとは別に設計する。
- 将来のExcel、PowerPoint、Jira、Backlog、Slack出力は確認済み文脈から生成する。

## 文書一覧

- [プロダクトビジョン](01-vision/product-vision.md)
- [MVPスコープ](02-requirements/mvp-scope.md)
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
- [PoC一覧](07-poc/README.md)
- [Phase 0 配布・権限スパイク結果](07-poc/phase-0-distribution-permissions-result.md)

## リポジトリ境界

- Docs: `/Users/mikey/workspace/vault/personal-vault/20-projects/contextory/docs`
- App: `/Users/mikey/workspace/apps/contextory-app`
- 実業務データ: 会社PCのローカルアプリデータ領域。Git管理しない。

## 現在の状態

Phase 0の配布・権限スパイクは完了しました。私用Macでビルドしたアプリを会社Macへ直接配布でき、必要な権限、Claude Code検出、ローカル保存、再起動後の状態維持を確認済みです。次はPhase 1のInput Captureを実装します。
