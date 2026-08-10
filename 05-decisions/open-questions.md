# 未決事項

## 技術

- 最低対応macOSバージョンをいくつにするか。会社MacのOSを確認して確定する。
- SQLite accessに標準C API、軽量wrapper、ORMのどれを使うか。
- Source Manifestの必須項目とschema migrationをどう設計するか。
- SQLite Indexの初期テーブル、全文検索、再構築処理をどう設計するか。
- ローカルOCRと文字起こしに何を使うか。
- Claude Codeをどの単位・権限・作業ディレクトリで起動するか。
- 画面録画から音声、キーフレーム、OCRをどのように抽出するか。
- 即時処理のキュー、再試行、失敗状態をどう管理するか。
- 常駐UIをmacOSメニューバーだけにするか、小型フローティングコントロールも用意するか。
- キャプチャ、録音、録画のグローバルショートカットをどう割り当てるか。

## プロダクト

- 案件とタスクの作成・関連付け候補をどの確信度から自動提示するか。
- キャプチャ直後に必要な確認操作は何か。
- 人物名を保持しながら不要な個人情報をマスクする判定ルール。
- 原本、派生物、確認済み知識の保存期間。
- 日次レビューの通知時刻、未確認件数、優先順位をどう表示するか。
- Library／Review Interfaceを同一アプリ内の別ウィンドウ、別アプリ、ローカルWeb UIのどれにするか。

## 運用

- 会社PCでのVault保存先とバックアップ方針。
- Apple Developer Programへ加入済みか。Developer ID署名・notarizationを使用できるか。
- 初回の会社Mac向けバイナリをDeveloper ID署名・notarizeするか、未署名Copy Appで限定導入するか。
- 会社Macへの`.app`受け渡し方法とハッシュ照合手順。
- 会社MacのmacOS、CPU architecture、Claude Codeバージョン確認。優先経路ではXcodeとSwiftは不要。フォールバック採用時だけXcode環境を確認する。
- アプリ更新時にBundle IDと署名identityを維持し、画面収録・マイク権限が継続するか。
- 会社Macでのローカルビルドが必要になった場合のclone、build、update手順。
- Claude Codeへ送信したSourceと時刻を監査記録するか。

上記の会社Mac導入関連項目は機能実装の終盤まで保留せず、Phase 0の配布・権限スパイクで可能な限り解消する。
