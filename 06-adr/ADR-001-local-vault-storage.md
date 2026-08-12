# ADR-001 Local VaultのファイルとSQLite構成

- Status: Accepted
- Date: 2026-08-10

## Context

Contextoryは画面キャプチャ、音声録音、画面録画を継続的に取得し、文字起こし、AI要約、Markdown生成、自動グルーピングを行う。原本には画像、音声、動画が含まれ、Claude Codeから参照しやすく、アプリやDB障害時にも回収可能である必要がある。

一方、ファイルだけでは次の処理が難しい。

- AI処理キューと再試行。
- 処理状態と失敗状態の管理。
- 複数Project／Taskへの関連付け。
- 未確認項目と日次レビュー対象の抽出。
- Source横断検索と重複判定。

画像、音声、動画をSQLite BLOBへ格納すると、DBが巨大になり、Claude Codeから扱いにくくなり、DB破損時の影響範囲も大きくなる。

## Decision

Local Vaultは、SourceごとのファイルBundleとSQLite Indexを組み合わせて構成する。

### Source Bundle

- Sourceごとに固有IDのフォルダを作成する。
- 原本、`manifest.json`、文字起こし、プレビュー、キーフレームを同じBundleへ保存する。Analysis／OutputはADR-009に従う派生Sourceとして同じ`sourceId`モデルへ保存する。
- 画像、音声、動画は通常ファイルとして保存し、SQLite BLOBへ格納しない。
- 原本を上書きしない。
- 原本のSHA-256をManifestへ保存する。
- ファイル名に顧客名、人物名、件名などの機密情報を含めない。
- Project／Taskとの関係が変わってもBundleを物理移動・複製しない。

### ManifestとMarkdown

- `manifest.json`を機械可読な永続メタデータとする。
- Markdownをユーザーとアプリが確認・再利用できる成果物とする。Claude Codeには選択した送信対象だけを、ADR-012の一時staging directoryからRead限定で渡す。
- Analysis Sourceの最新`summary.md`はRevision snapshotから再生成するmaterialized viewとし、Revisionの代替正本にしない。
- AI生成内容とユーザー確認済み内容を状態で区別する。
- Project／Taskとの確定した関連とレビュー結果を、SQLiteだけに閉じ込めない。
- Manifest schemaには`schemaVersion`を持たせる。

### SQLite Index

- Source検索、関連付け、処理キュー、再試行、レビュー対象抽出、全文検索にSQLiteを使用する。
- SQLiteを画像、音声、動画の保存先にしない。
- 永続的な情報は可能な限りSource Bundleから再構築できるようにする。
- 実行待ち、処理中、ロック、再試行回数などの一時的な運用状態はSQLiteのみで保持できる。

## Consequences

### Positive

- 原本とAI成果物を通常ファイルとして確認・回収できる。
- Claude Codeが一時staging directory内の選択済み画像とMarkdownを扱える。
- SQLiteにより検索、関連付け、処理キューを効率的に実装できる。
- DB破損時にSource Bundleから索引を再構築できる。
- Project／Taskの変更で大容量ファイルを移動・複製せずに済む。

### Negative

- ファイルとSQLite間の整合性管理が必要になる。
- Manifest schemaとDB schemaのmigrationが必要になる。
- 再索引と整合性検査を実装する必要がある。
- 同一Sourceを複数プロセスが更新しないための排他制御が必要になる。

## Implementation constraints

- 原本保存をSQLite更新より優先し、索引更新失敗で原本を失わない。
- ManifestとMarkdownは一時ファイルからの置き換えで更新する。
- SQLite DB、WAL、SHM、Source BundleをGit管理しない。
- Local Vaultはクラウド同期フォルダとアプリリポジトリの外へ配置する。
- SQLite削除後にSource Bundleを走査して再索引できる検証を用意する。

## Follow-up

- Manifest schemaを確定する。
- SQLiteの初期schemaとmigration方式を決定する。
- 全文検索方式を決定する。
- Source BundleとSQLiteの整合性検査・再索引PoCを行う。
