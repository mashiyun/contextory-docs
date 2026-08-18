# 未決事項

## 技術

- 最低対応macOSバージョンをいくつにするか。会社MacのOSを確認して確定する。
- SQLite accessに標準C API、軽量wrapper、ORMのどれを使うか。
- Source Manifestの残る必須項目とschema migration単位をどう設計するか。今回確定した`generation.operationId`、Transcript来歴、旧フィールド読込順、External Ticket Sourceのversion 4／record version 1境界は再検討対象にしない。
- SQLite Indexの列型、索引名、全文検索、再構築処理をどう設計するか。今回確定した`operationId`一意制約、重複時fail-closed、辞書・補正索引のBundle正本性、External Ticketのsnapshot／remote key／Job索引が再構築可能な投影であることは再検討対象にしない。
- 画像本文やURLの抽出を将来の任意機能として追加するか、追加する場合の方式と保存範囲は未決とする。Vision OCR、自動マスク、URL領域マスクをClaude送信の前提または必須機能にしない。
- SonnetからOpusを提案する自動ルーティング条件と、実行前確認をどの範囲で必要にするか。
- 画面録画から音声とキーフレームをどのように抽出するか。任意の画像本文／URL抽出とは独立して設計する。
- 即時処理Queueの追加機能をどう管理するか。`completion_sync_pending`／`completion_sync_failed`の復旧・上限・手動再同期と、既存`retry_waiting`の自動再試行禁止は確定済みとする。
- 常駐UIをmacOSメニューバーだけにするか、小型フローティングコントロールも用意するか。
- キャプチャ、録音、録画のグローバルショートカットをどう割り当てるか。
- 既存のContext／Analysis Bundleを統一SourceとGroup／Revisionモデルへ移行するschema migrationと後方互換をどう実装するか。
- Revision差分をMarkdownの行差分、構造化フィールド差分、または両方のどこまで表示するか。
- Slack／Teamsの対象Bundle ID一覧と、ユーザー設定での追加・変更UIをどう確定するか。
- 録音入力デバイスの安定識別子、開始前メーターの実装方式、マイク／システム音声別の無音・切断検出をどう設計するか。
- 無音警告のしきい値、観測時間（目安5〜10秒）、ノイズ環境別の調整方法をPoCでどう決めるか。無音と設定ミスを完全に区別しない前提を維持する。
- Analysis一覧に出す具体的要約の最大長をどう決めるか。ユーザー修正済み要約の優先順位は確定済みで、未決は最大長だけとする。
- Actionの期限候補の抽出規則、状態遷移、ActionからTaskを作成した後の同期範囲をどう決めるか。
- Whisperへ用語ヒントを渡す方式（`whisper-cli`の初期プロンプト等）、語数上限、実際の改善効果をPoCでどう確認するか。ヒントの効果は未検証であり、実装前に確定しない。

### 段階実装への影響

- 上記の未決事項は、確定済みの原本不変性、`operationId`収束、完了再同期上限、辞書Revision正本、決定的補正を変更しない。
- 要約の最大長は表示調整であり、保存・移行・一覧の実装を妨げない。確定までは保存本文を切り詰めず、UI側だけで折り返し・省略表示する。
- Whisper用語ヒントはロードマップの段階6に限定する。段階1〜5の一覧、完了収束、Transcript訂正、辞書保存、文字起こし後補正は先に実装できる。
- Topic／Task系の直近段階1〜2は、`task.json` schema version 3 reader、SQLite migration、Bundle検証をwriterより先に導入することで開始できる。複数GroupのWBS表示順、非Transcript span、PMカード規則には依存しない。
- 非Transcript長文のsnapshot／offset形式はTopic段階5の開始ゲートとする。未決のまま推測して保存せず、段階5の初期実装を固定Transcript snapshotを持つ音声・動画に限定する場合は、その制限をUIと受入試験へ明示する。
- 複数Group所属Taskの表示順規則はWBS段階9、PMカードの抽出・更新規則はデイリーブリーフィング段階8およびDecision Log／RAID段階11の開始前に確定する。Task紐付け、手動編集、コメント、Revision、Transcript、Topic基盤の先行段階を妨げない。
- その他のUI、Adapter、運用上の未決事項は、それぞれを使用するPhaseの開始前に解消し、先行Phaseのschemaや安全境界を再び未決へ戻さない。

## プロダクト

- 複数Groupに同時所属するTaskについて、MVPの単一`displayOrder`をGroupごとのWBS表示でどのように解釈するか。WBS専用正本を作らない方針は確定しているため、必要ならGroup link上の表示設定ではなく、Taskの順序規則またはUIの一時表示規則として定義する。
- Topic Sourceの手動範囲選択で、長文の非Transcript範囲をどのsnapshot／offset形式で表すか。音声・動画のTranscriptはUTF-8 byte範囲と録音session相対時刻で固定することは確定している。
- PM支援ビューの各カードについて、既存Source／Group／Taskからの抽出条件、更新タイミング、空状態をどこまで共通化するか。カードを別の管理正本にしないことは確定している。

- 案件とタスクの作成・関連付け候補をどの確信度から自動提示するか。
- キャプチャ直後に必要な確認操作は何か。
- 将来の任意機能として、人物名を保持しながら不要な個人情報をマスクする判定ルールを設けるか。実装しないことや検出失敗をClaude送信停止の理由にしない。
- 日次レビューの通知時刻、未確認件数、優先順位をどう表示するか。
- Library／Review Interfaceを同一アプリ内の別ウィンドウ、別アプリ、ローカルWeb UIのどれにするか。
- ゴミ箱からの復元時の表示と、参照中Sourceを削除候補画面でどう説明するか。
- Jira／ConfluenceのCloud・Data Center種別、Backlogの会社環境、必須Custom Field、Project／Space選択、添付制限をどう確定するか。Adapter実装時のAPI仕様はAtlassian／Nulabの公式資料だけで確認する。
- Jira Read Adapterの対象をCloud／Data Centerのどちらにするか、Backlog対象環境・認証方式・必要scope、instance IDの不変性、issue IDの一意scopeと不変性をどう確定するか。Read Adapterのpagination、rate limit、retry、timeout、pagination中のremote更新を検出するconsistency anchorの具体方式は、対象環境の公式資料で確認してから固定する。これは各Read Adapterの開始ゲートであり、`unconfirmed`手動取り込みを妨げない。
- 手動取り込みの`unconfirmed` External Ticket Sourceを、Adapterが取得した不変issue IDと照合してどの確認UIでconfirmed系列へ関連付けるか。既存Sourceのidentityを上書きせず、確認eventまたは新Sourceで関連を表す方式は段階2開始前に確定する。推測によるremote key付与・既存snapshotとの自動統合はしない。
- 添付のprovider別size上限、期限付きURL／cross-origin redirect、Content-Type不一致をどう扱うか。元credentialをredirect先へ転送しないことは確定しており、具体値は添付選択取得段階の開始ゲートとする。
- 外部公開で`outcome_unknown`となった場合のremote確認、既存remoteへの紐付け、明示的な再作成のUXをどう設計するか。
- Transcript訂正のUI形態（該当箇所の直接編集、全文編集、誤変換候補の提示）をどうするか。
- 辞書登録候補を提示するタイミング（訂正直後、レビュー時、まとめて確認）と、確認負荷をどう抑えるか。

## 運用

- 会社PCでのVault保存先とバックアップ方針。
- アプリ更新時にBundle IDと署名identityを維持し、画面収録・マイク権限が継続するか。
- 会社Macでのローカルビルドが必要になった場合のclone、build、update手順。
- ローカル監査記録の保持期間、容量上限、破損時の整合性検査をどう設計するか。

上記の会社Mac導入関連項目は機能実装の終盤まで保留せず、Phase 0の配布・権限スパイクで可能な限り解消する。

Phase 0で、GitHub Releaseによる受け渡し、SHA-256照合、ad-hoc署名アプリの起動手順、会社MacのOS／architecture、Claude Code実行ファイル検出を確認した。詳細は[Phase 0 配布・権限スパイク結果](../07-poc/phase-0-distribution-permissions-result.md)を参照する。

## 決定済み（2026-08-15）

未決から外した項目と、確定内容の参照先を記録する。いずれも仕様として確定済みであり、実装は未了である。

- 一覧のJST表示粒度と同一要約・同一分の識別: 既定は分単位、一致集合だけ秒、なお一致する場合だけ小数秒へ拡張する。詳細は[Source・Group・Task・Output要件](../02-requirements/source-group-task-output.md)。
- `completion_sync_pending`の再同期契機、上限回数、診断: 起動時復旧対象とし、各試行の開始前に回数を永続化する。5秒・30秒・5分の最大3回、以後は`completion_sync_failed`で自動停止し診断と手動再同期を提供する。部分保存・破損は`analysis_integrity_failed`へ分離する。詳細は同要件と[データモデル](../03-design/data-model.md)。
- `presentationSummary`とlegacy `presentationTitle`の移行: 新規書き込みは新フィールドのみ、読込は固定の優先順位、legacy値を削除せずbackfillしない。
- Transcript訂正・用語辞書・専用再文字起こしに関する過去の決定はRetiredであり、通常のSource／Revision／Queueモデルへ追加しない。撤去実装と検証が完了するまで、履歴仕様として[Transcript訂正・用語辞書要件](../02-requirements/transcript-correction-terminology.md)と[ADR-014](../06-adr/ADR-014-transcript-correction-terminology.md)を保持する。
- 同一操作の収束条件: `operationId`をidempotency keyとし、`requestFingerprint`は監査用に分離する。fingerprint一致だけで別operationを統合しない。
- schema切替順: version 3 Source／version 2 Revisionはversion 1／2／3 reader、SQLite nullable列・新規テーブル、Bundle検証、legacy nullを除く部分一意索引、検証後の新規writer有効化の順とする。`stagedInputRefs`を持つAnalysis Revision version 3もversion 1／2／3 reader、再構築可能なSQLite索引、Bundle内snapshotと参照hashの検証をwriterより先に提供する。External Ticket Source version 4はversion 1〜4 reader、SQLite migration、Bundle／snapshot検証、再索引、version 4 writerの別migrationとする。旧Bundleを一括backfillしない。
- Transcript訂正位置: 固定snapshot上のUTF-8 byte半開区間を使い、scalar境界、訂正前byte列、非重複を検証する。別モデルのTranscriptへ自動追従させない。
- role別Transcript統合: `mergeAlgorithmVersion: 1`はsession相対の開始時刻、終了時刻、role順、元segment連番で安定sortし、必須時刻が不正ならfail-closedとする。
- 辞書照合単位: `normalizationAlgorithmVersion: 1`はUnicode scalar列の完全一致とし、暗黙の大文字小文字・全角半角・かな・送り仮名・Unicode正規化を行わない。

## 決定済み（2026-08-16）

以下は仕様として確定済みであり、実装は未了である。

- ユーザーが明示選択した原画像は、会社契約Claude Codeへ未マスクで送信できる。Vision OCR、URL領域マスク、自動マスク、マスク失敗時の送信停止を必須とせず、パスワード、APIキー、Cookie、秘密鍵、認証QRコード等を含む画面は取得・送信しない運用境界とする。画像から抽出・保存・表示するURLはquery／fragmentを除去する。
- 初回解析、Revision再分析、AI対話、Group展開の全Claude実行は、選択済み入力だけを置く再生成可能な一時staging directoryをcwdとし、Source Bundleをcwdまたは`--add-dir`として直接公開しない。明示選択した原画像、テキスト、PDF、安全化済みURL、固定済みTranscript、代表フレームは配置できるが、音声・動画原本、`localOpenUrl`、未選択file、別Sourceの未選択原本、Vault全体は配置しない。
- 初回のRevision 1を含む各Revisionと、Revisionを作らないAI対話／Group展開の追加専用invocation auditへ`stagedInputRefs`を固定する。不変input snapshotまたは削除保護されたSource参照から当時のbyte集合を再現・検証可能にし、一時staging自体を正本にしない。詳細と正本境界は[ADR-012](../06-adr/ADR-012-minimal-claude-staging.md)に従う。
