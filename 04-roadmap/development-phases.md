# 開発フェーズ

## Phase 0: 基盤

Status: 完了（2026-08-11）

- App／Docsの責務分離。
- 作業指示、checkpoint、Git除外、安全方針。
- Local Vaultのファイル＋SQLite構成をADRで確定。
- 初期対象OSをmacOSに限定するADR。
- Swift／SwiftUIによるmacOSネイティブ・単一ユーザー構成をADRで確定。
- 私用Macでビルドし会社Macへバイナリ提供する運用をADRで確定。
- 固定Bundle IDを決定する。
- 空のmacOSメニューバーアプリを作成し、Debug／Releaseビルドを確認する。
- 私用MacでRelease `.app`を生成する。
- 署名状態、Bundle ID、entitlements、同梱ファイル、SHA-256を確認する。
- 会社Macへバイナリだけを提供してGatekeeperの初回起動を確認する。
- 最小の画面収録・マイク権限要求を実装し、会社Macで許可できることを確認する。
- 会社MacでClaude Codeのパスと認証状態を確認する。
- アプリ終了後の再起動で、起動、権限、アプリデータを維持できることを確認する。
- 更新版への置き換えによる権限維持は、次回Release時に継続確認する。
- 直接配布で起動できたため、会社MacでのXcodeローカルビルドは実施しない。

実機結果と配布手順は[Phase 0 配布・権限スパイク結果](../07-poc/phase-0-distribution-permissions-result.md)を参照する。

## Phase 1: Input Capture

- macOSメニューバー常駐。
- グローバルショートカット。
- 範囲キャプチャ。
- マイクとシステム音声の録音。
- 画面録画。
- 開始、停止、経過時間、取得状態の表示。
- 原本とメタデータのローカル保存。
- Source Bundle、Manifest schema、SQLite Indexの最小実装。
- 保存、AI処理、完了、失敗の状態表示。
- 総コンテンツ数とLocal Vault使用量の表示。
- Source Bundleから再構築可能なSQLite `sources` Index。
- 直近取得の2段階確認とゴミ箱への移動。
- 明示的に有効化する保存完了通知。

## Phase 2: Immediate Processing

- 文字起こし。
- Claude Codeへの事前許可済み即時処理。
- Sonnet／Opusの手動選択とCLIへのモデル指定。
- 使用モデルとルーティング理由の処理結果への記録。
- 単純な処理をSonnet、複雑な複数Source処理をOpus候補とする自動ルーティングの検討。
- 要約とMarkdown生成。
- 処理失敗時の状態保持と手動再実行。

## Phase 3: Grouping Data

- 案件・タスク候補への自動グルーピング。
- グルーピング候補、根拠、確信度の保存。
- 後続の手動修正を可能にする関連モデル。
- Fact、Decision、Action、Question、Risk、Waiting候補の生成。
- Library／Review Interfaceが利用できる確認待ちデータの保存。

## Phase 4: Library and Daily Review

- 撮り溜めたコンテンツの一覧、検索、詳細表示。
- 最低1日1回のReview Queue確認。
- グルーピングとAI理解の修正・確定。
- 修正履歴と確認済み文脈の蓄積。

## Phase 5: Output Support

- Jira、Backlog、Slack向け下書き。
- Excel、PowerPointなどの成果物生成。
- 外部サービスへの反映はユーザー承認後に行う。
