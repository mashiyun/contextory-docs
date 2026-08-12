# 未決事項

## 技術

- 最低対応macOSバージョンをいくつにするか。会社MacのOSを確認して確定する。
- SQLite accessに標準C API、軽量wrapper、ORMのどれを使うか。
- Source Manifestの必須項目とschema migrationをどう設計するか。
- SQLite Indexの初期テーブル、全文検索、再構築処理をどう設計するか。
- URL検出用のローカルVision OCRは確定している。画面本文の一般OCRを実装するか、実装する場合の方式と保存範囲は未決とする。
- SonnetからOpusを提案する自動ルーティング条件と、実行前確認をどの範囲で必要にするか。
- 画面録画から音声とキーフレームをどのように抽出するか。一般本文OCRは未決のため、URL検出用Vision OCRと混同しない。
- 即時処理のキュー、再試行、失敗状態をどう管理するか。
- 常駐UIをmacOSメニューバーだけにするか、小型フローティングコントロールも用意するか。
- キャプチャ、録音、録画のグローバルショートカットをどう割り当てるか。
- 既存のContext／Analysis Bundleを統一SourceとGroup／Revisionモデルへ移行するschema migrationと後方互換をどう実装するか。
- Revision差分をMarkdownの行差分、構造化フィールド差分、または両方のどこまで表示するか。
- 確定したURL検出用Vision OCRについて、URL領域検出精度、マスク範囲、tokenらしさの具体的な判定規則をどう検証・調整するか。
- Slack／Teamsの対象Bundle ID一覧と、ユーザー設定での追加・変更UIをどう確定するか。

## プロダクト

- 案件とタスクの作成・関連付け候補をどの確信度から自動提示するか。
- キャプチャ直後に必要な確認操作は何か。
- 人物名を保持しながら不要な個人情報をマスクする判定ルール。
- 日次レビューの通知時刻、未確認件数、優先順位をどう表示するか。
- Library／Review Interfaceを同一アプリ内の別ウィンドウ、別アプリ、ローカルWeb UIのどれにするか。

## 運用

- 会社PCでのVault保存先とバックアップ方針。
- アプリ更新時にBundle IDと署名identityを維持し、画面収録・マイク権限が継続するか。
- 会社Macでのローカルビルドが必要になった場合のclone、build、update手順。
- ローカル監査記録の保持期間、容量上限、破損時の整合性検査をどう設計するか。

上記の会社Mac導入関連項目は機能実装の終盤まで保留せず、Phase 0の配布・権限スパイクで可能な限り解消する。

Phase 0で、GitHub Releaseによる受け渡し、SHA-256照合、ad-hoc署名アプリの起動手順、会社MacのOS／architecture、Claude Code実行ファイル検出を確認した。詳細は[Phase 0 配布・権限スパイク結果](../07-poc/phase-0-distribution-permissions-result.md)を参照する。

正規モデルでのClaude実行は、選択済み送信対象だけを置く一時staging directoryを`Read`限定で渡し、セッション非保存、構造化JSON出力とする。通常画像・テキストは選択済みならstagingへ置けるが、Source Bundle全体、音声・動画原本、`localOpenUrl`は置かない。URLはquery／fragmentを安全化する。このstaging境界は未実装であり、既存の実行経路を置き換える。ユーザーが明示的に取得・取り込みしたSourceは、その操作を処理許可として保存後に自動分析し、失敗時の自動再送は行わない。
