# 開発フェーズ

## Phase 0: 基盤

- App／Docsの責務分離。
- 作業指示、checkpoint、Git除外、安全方針。
- Local Vaultのファイル＋SQLite構成をADRで確定。
- 初期対象OSをmacOSに限定するADR。
- Swift／SwiftUIによるmacOSネイティブ・単一ユーザー構成をADRで確定。
- 私用Macでビルドし会社Macへバイナリ提供する運用をADRで確定。

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

## Phase 1.5: Binary Delivery Smoke Test

- 私用MacでRelease `.app`を生成する。
- 署名状態、Bundle ID、entitlements、同梱ファイル、ハッシュを確認する。
- 会社Macへバイナリだけを提供して初回起動する。
- Gatekeeper、画面収録、マイク権限、Claude Code検出、Vault作成を確認する。
- 更新版の上書き後も権限と既存Vaultを維持できるか確認する。
- 直接配布が安定しない場合、会社MacへXcodeを導入してローカルビルドし、同じ機能を確認する。

## Phase 2: Immediate Processing

- 文字起こし。
- Claude Codeへの事前許可済み即時処理。
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
