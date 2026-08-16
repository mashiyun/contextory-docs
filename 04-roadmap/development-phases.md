# 開発フェーズ

## 正規モデル実装順

未実装の正規モデルは、次の順で実装する。後続のPhaseは前段の永続化・検証条件を満たすまで開始しない。

1. Source統一・legacy移行
2. Group
3. Task–Source／Group紐付け
4. Task手動追加・編集・コメント
5. 汎用Revision・追加情報
6. Transcript訂正・用語辞書
7. Topic Source・話題分割
8. Topic SourceからTask作成
9. 返答待ち・blocker
10. デイリーブリーフィング
11. WBS親子・依存関係
12. AI相談
13. Decision Log／RAID
14. 派生Output・外部公開

## 実環境フィードバック反映の実装順

実利用で確認した不具合と、音声文字起こしの訂正・用語辞書は、次の順で実装する。前段の永続化・検証条件を満たすまで後続を開始しない。

1. Analysis一覧のJST日時表示と簡素化。一覧を具体的な約10〜20 Characterの要約とJST日時だけにし、技術情報を詳細・診断画面へ移す。保存時刻はUTCのISO 8601のままとする。同一要約・同一分の場合だけ秒、なお一致する場合だけ小数秒へ拡張する。`presentationSummary`とlegacy `presentationTitle`のeffective summary解決規則を読込・書込・再索引で統一し、保存値を変更せず、20 Character超過時だけ表示を19 Character＋`…`へ短縮する。version 1／2／3 readerをwriterより先に提供する。
2. 解析成功後の状態競合修正。`operationId`をJob作成時にUUIDで確定して実行前に永続化し、Analysis保存・Summary保存・親Manifest更新を`AnalysisStore`へ集約する。SQLite migration、Bundle検証、legacy nullを除く部分一意索引の順で導入し、検証後だけ新規writerを有効化する。保存前失敗、保存後の状態同期失敗、保存物の整合性失敗を分離する。`completion_sync_pending`は起動時復旧対象として試行開始前に回数を永続化し、5秒・30秒・5分の最大3回で再同期、3回失敗後は`completion_sync_failed`とする。部分保存・破損は`analysis_integrity_failed`で自動処理を止める。
3. Transcript訂正とRevision再生成。role別生Transcriptを`rawTranscriptRefs`として不変保持し、確定済みの`mergeAlgorithmVersion: 1`で統合する。訂正位置は固定snapshot上のUTF-8 byte半開区間とし、`transcriptTransformSteps`へ変換順序と入出力hashを保存する。訂正を不変Sourceとして追加し、訂正版snapshotを持つRevisionからSummaryとActionsを再生成する。`operationId`で収束させ、`requestFingerprint`を監査用に保存する。
4. 共通／Group辞書。ユーザー確認を必須とする辞書登録、追加専用の辞書Revision、`dictionaryRevisionRefs`によるsnapshot固定、削除後も過去Analysisを再現できる保存を実装する。同時適用は共通辞書＋明示選択した1つのGroup辞書までとする。
5. 決定的な文字起こし後補正。`normalizationAlgorithmVersion: 1`の完全一致規則を実装し、補正前Transcript、適用位置、`dictionaryRevisionRefs`、未適用項目と理由を保存する。
6. Whisper用語ヒント。PoCで方式、語数上限、改善効果を確認してから有効化する。未検証のヒントを自動適用せず、段階1〜5の実装をこのPoC待ちにしない。

詳細は[Transcript訂正・用語辞書要件](../02-requirements/transcript-correction-terminology.md)、[Source・Group・Task・Output要件](../02-requirements/source-group-task-output.md)、[ADR-014](../06-adr/ADR-014-transcript-correction-terminology.md)を参照する。

## 次のLibrary安全管理実装順

以下は実装済みである。この開発サイクルの最終全検証は別途実施する。実利用Vaultは対象にしない。

1. Analysis一覧の要約を、保存値を変更せず表示時だけ約10〜20 Characterへ短縮した。20 Character超過時は19 Character＋`…`とし、legacy値にも同じ規則を適用する。
2. 現行Appが読める参照だけを使う未参照Source一覧を追加した。読取エラーや未対応形式では通常のエラーを表示し、完全な参照グラフは保証しない。
3. TaskとGroupの削除導線をarchiveへ統一した。Bundle、link、履歴を保持し、状態だけを保存する。
4. 未参照Sourceの複数選択Trash移動を実装した。対象を明示確認した後、既存のSource単位ゴミ箱移動を順に実行し、各Sourceの結果を表示する。batchのsnapshot、atomicity、全件再検証、rollbackは追加しない。

## 次期Topic Source・Task・PM実装順

Group実装後は、[Topic Source・Task・WBS・PM支援要件](../02-requirements/topic-source-task-wbs.md)と[ADR-015](../06-adr/ADR-015-topic-source-task-wbs.md)に従い、次の順で実装する。各段階は前段の正本・整合性検証を満たすまで開始しない。

1. Task–Source／Group紐付け
2. Task手動追加・編集・コメント
3. 汎用Revision・追加情報
4. Transcript訂正・用語辞書
5. Topic Source・話題分割
6. Topic SourceからTask作成
7. 返答待ち・blocker
8. デイリーブリーフィング
9. WBS親子・依存関係
10. AI相談
11. Decision Log／RAID
12. 派生Output・外部公開

段階1〜2は`task.json` version 1／2／3 reader、SQLiteのnullable列・新規表、Bundle検証、version 3 writerの順で導入し、旧Taskを一括backfillしない。段階5のTopic writerは、Evidence Spanのsnapshot／hash／時刻／byte境界とSource DAG検証を実装してから有効化する。非Transcript長文のoffset形式が未決の間は固定Transcript snapshotを持つ音声・動画だけを対象とする。複数GroupのWBS表示順は段階9、PMカードの抽出規則は段階8／11の開始ゲートであり、段階1〜7を妨げない。

簡易タイムライン、Markdown／CSV／Excel出力、要件変更・影響分析、ステータスレポート、顧客フィードバック整理、優先順位付け、リリース準備確認、Jira／Backlog同期はこの順の後続とする。工数、原価、リソース配分、複数ユーザー共同編集、権限管理は対象外とする。

## 次期External Ticket Source実装順

[External Ticket Source要件](../02-requirements/external-ticket-source.md)と[ADR-016](../06-adr/ADR-016-external-ticket-source.md)に従い、外部公開とは別の読み取り専用取り込みを次の順で実装する。

1. Source Manifest version 4 reader、canonical snapshot、`unconfirmed`手動取り込み、operation ID復旧
2. provider identity profile、confirmed remote identity・重複／系列判定
3. snapshot差分
4. SourceからTask作成／既存Task Link
5. Jira Read Adapter
6. Backlog Read Adapter
7. 添付選択取得
8. 手動再取得
9. 必要性確認後の同期支援

段階1はversion 1〜4 reader、SQLite migration、Bundle／snapshot検証、再索引、version 4 writerの順で導入し、既存Sourceを一括backfillしない。providerの不変IDを検証できない手動Sourceは`unconfirmed`として保存し、内容hashによる自動統合を行わない。Group／Task関連はSource確定と別transactionとし、Task schema version 3の`sourceLinks`だけを更新する。

Jira Cloud／Data Center、Backlog対象環境、認証方式、provider identity profile、pagination／rate limit／retryの具体値を確定するまで段階5／6を開始しない。添付size上限とredirect規則は段階7の開始ゲートとする。これらは段階1の手動保存を妨げない。reader・SQLite migration・Bundle検証を先行させるまでRead Adapter writerを有効化せず、取り込み段階に外部ticketの変更、コメント、完了、外部公開Adapterの呼出しを含めない。

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
- システム音声とマイク音声を別原本として保存し、roleと同期情報をManifestへ記録。
- 画面録画。
- 開始、停止、経過時間、取得状態の表示。
- 原本とメタデータのローカル保存。
- 取得元アプリ名・Bundle IDのManifest記録。
- Source Bundle、Manifest schema、SQLite Indexの最小実装。
- 保存、AI処理、完了、失敗の状態表示。
- 総コンテンツ数とLocal Vault使用量の表示。
- Source Bundleから再構築可能なSQLite `sources` Index。
- 直近取得の2段階確認とゴミ箱への移動。
- 再起動後の直近Source復元。
- 明示的に有効化する保存完了通知。

## Phase 2: Immediate Processing

- 実装済み: 明示的に取得・取り込みしたSourceの保存後自動送信。
- 実装済み: Inputと解析状態の分離、直列解析Queue、待機件数表示、起動時Queue復元。
- 実装済み: 5分タイムアウト、使用上限・認証・タイムアウト・不正出力・実行失敗の分類、再試行待ち隔離、後続Queue継続。
- 実装済み: Claude Codeの非対話実行、Read限定、セッション非保存、構造化JSON出力。
- 実装済み: Sonnet／Opusの手動選択とCLIへのモデル指定。
- 実装済み: `summary.md`の原子的保存、`needs_review`状態、処理ジョブと失敗状態の記録。
- 実装済み: ファイルと一次ソース原文の手動取り込み。
- 基盤実装済み・UIはPhase 4へ移動: ユーザー補足の追記と補足込み再分析。
- 基盤実装済み・UIはPhase 4へ移動: 複数Sourceの手動Context関連付けとグループ全体の統合分析。
- 実装済み: 画像・PDF・テキストの形式制限、Content Type・byte数・SHA-256記録、暗号化・破損PDF拒否。
- 実装済み: 貼り付けテキスト単体の一次Source登録。
- 実装済み（互換モデル）: Source／Context／Analysisの分離と、解析目的ごとの追加専用派生結果。
- 実装済み: AIによるタスク主分類、タグ、確信度、理由の`proposed`保存。
- 実装済み: `whisper-cli`のarm64静的ビルド、MITライセンス同梱、会社MacでのHomebrew非依存化。
- 実装済み: 多言語baseモデルのApplication Support配置、SHA-256検証付き取得、手動配置、削除導線。
- 実装済み: base／smallの併存・選択、モデル別前処理成果物、同一Sourceの追加専用再処理、Analysisへの音声モデル記録。
- 実装済み: 音声・動画をClaudeへ渡す前のローカル文字起こし。
- 実装済み: 動画代表フレーム抽出と、前処理不能時にAnalysisを生成しない制御。
- 未確認: Contextoryで取得した実音声・実動画によるアプリ全体E2E。
- システム音声・マイク音声の個別文字起こしとタイムスタンプ統合。
- 実機でのClaude Code認証・非対話分析確認。
- 使用モデルとルーティング理由の処理結果への記録。
- 単純な処理をSonnet、複雑な複数Source処理をOpus候補とする自動ルーティングの検討。
- 処理失敗時の状態保持と手動再実行。

## Phase 3: Canonical Source and Grouping Data

- 実装済み: 新規Analysisを`kind: analysis`の正規`sourceId`を持つ派生Sourceとして保存する基盤。Output生成はPhase 5の未実装範囲とする。
- 実装済み: `analyses/`／`contexts/`の読み取り互換、legacy `analysisId`から`sourceId`への対応、Revision 1 snapshot検証、Bundle走査によるSQLite再索引、新規Analysis書き込みのfail-closed gate。
- 実装済み: 新規Analysis staging Jobの復旧要求保存、同一`operationId`の重複防止、未確定の複数Source Analysisを通常Input Queueへ分解しない復旧。
- 未実装: 既存Analysisへの正規`sourceId`とRevision 1割り当てを実利用Vaultで実行する運用と、Revisionの新規書き込みUI。
- Groupの作成、名称変更、削除、Source追加・除外、複数Group所属。
- Groupは関連情報を集めるだけとし、追加時にAnalysisやOutputを自動生成しない。
- Groupから派生Sourceを生成する前に、実際に使用する個別Sourceを明示・固定する。
- Group–Sourceの正本を`group.json`の`sourceLinks`に置き、`group_sources`を再構築可能なSQLite索引として実装する。
- 案件・タスク候補への自動グルーピング。
- グルーピング候補、根拠、確信度の保存。
- 後続の手動修正を可能にする関連モデル。
- Fact、Decision、Action、Question、Risk、Waiting候補の生成。
- Library／Review Interfaceが利用できる確認待ちデータの保存。

## Phase 4: Library and Daily Review

- 実装済み: Inputと分離したタスク整理ウィンドウ。
- 実装済み: Analysis一覧、Markdown詳細、根拠SourceのFinder表示。
- 実装済み: AnalysisからのTask作成と、Source・Analysis・親Taskの来歴保存・逆引き。
- 実装済み: `retry_waiting`／`analysis_failed`の一覧と解析Queueへの手動再投入。
- 実装済み: Task Bundleから再構築可能なSQLite `tasks` Index。
- 撮り溜めたコンテンツの一覧、検索、詳細表示。
- 最低1日1回のReview Queue確認。
- グルーピングとAI理解の修正・確定。
- 修正履歴と確認済み文脈の蓄積。
- Inputを持たないタスク整理画面として、解析結果表示、補足、関連付け、目的別再解析を提供する。
- Analysis Sourceへのテキスト・画像・URL追加、Revision履歴、過去summary、差分、根拠Source、最新`summary.md`のmaterialized view再生成を提供する。各Revisionのsummary snapshot、パス、SHA-256を必須とする。
- Analysis Source詳細から原本画像のプレビュー、原本音声・動画の再生、原本と追加情報のFinder表示を提供する。
- ユーザーが明示選択した原画像は、会社契約Claude Codeへ未マスクで送信できるようにする。Vision OCR、URL領域マスク、自動マスク、マスク失敗時の送信停止を必須工程にしない。画像、提供テキスト、ユーザー入力からURLを抽出・保存・表示する場合はquery／fragmentを除去し、AI表示URLとローカル遷移URLを分離する。
- Analysis Source詳細にClaudeへの質問・補足入力を設け、選択済みの追加Source／Groupだけを文脈にして対話履歴とRevisionを保存する。送信時のRevision、summary、Group展開結果、入力ハッシュ、モデル、prompt schema、結果状態を監査記録へ固定する。
- 初回解析、Revision再分析、AI対話、Group展開の全Claude実行で、`jobId`、`operationId`、`createdAt`を持つ再生成可能な一時staging directoryをClaudeのcwdにする。Source Bundleをcwdまたは`--add-dir`として直接公開せず、明示選択した未マスク原画像、テキスト、PDF、安全化済みURL、固定済みTranscript、代表フレームだけを配置する。音声・動画原本、`localOpenUrl`、未選択file、別Sourceの未選択原本、Vault全体は配置せず、完了、失敗、タイムアウト、中断後と、起動時に実行中Jobへ属さない残存stagingを回収する。
- 初回のRevision 1を含む各Revisionと、Revisionを作らないAI対話／Group展開の追加専用invocation auditへ、実際に送信したbyte集合を`stagedInputRefs`として固定する。非Sourceまたは変更・削除され得る派生入力はRevision Bundleへ不変snapshotし、不変Source原本はSource ID、相対path、hashを固定して削除保護する。一時stagingを正本にせず、正本Revision保存済みの復旧ではClaudeを再実行せず状態同期だけを行う。
- AI対話、再分析、Group展開でADR-008を適用し、未前処理／前処理失敗メディアを含む場合はfail-closedで送信を止める。
- TaskとSource／Groupを多対多で関連付ける。Task–Source／Task–Groupの正本を`task.json`の`sourceLinks`／`groupLinks`に置き、`task_sources`／`task_groups`を再構築可能なSQLite索引とする。
- 複数Taskを選択し、出力目的・形式を指定した派生Output Sourceを生成する。
- 来歴をGitHub風のラインで表示するグラフUIを検討する。
- 音声・動画の削除候補提示と、対象確認を伴う明示承認。期間による自動削除は行わない。
- Analysis／Sourceの参照確認付きゴミ箱移動。
- 新規・既存Sourceの既定保護ロック。Group／Task／派生Source／外部公開記録を含む「参照確認→一時ロック解除の確認→削除確認→macOSのゴミ箱へ移動」の順で削除する。
- Analysis一覧を、内容が分かる具体的な要約とJST日時だけの表示へ簡素化する。Analysis表記、分類、状態、hash、Source IDは詳細・診断画面へ移す。要約の根拠・確認状態・ユーザー修正を保存し、未生成時はSource種別とJST日時で暫定表示する。同一要約・同一分の集合だけ秒、なお一致する場合だけ小数秒まで拡張する。
- `presentationSummary`を新規書き込み先とし、legacy `presentationTitle`を読み取り互換で残す。effective summaryの優先順位を読込・書込・再索引で統一し、読込時のbackfillと一括更新を行わない。
- 解析成功後の状態競合修正。Revisionの不変Summary snapshot検証を成功判定とし、`AnalysisStore`を親Manifest更新の唯一の所有者にする。`operationId`の事前確定・一意制約・fail-closed再索引、保存前失敗・`completion_sync_pending`・`analysis_integrity_failed`の分離、起動時復旧、試行前の回数永続化、最大3回の自動再同期、`completion_sync_failed`と手動再同期を実装する。
- role別生Transcriptを不変原本として保持し、統合snapshotと`mergeAlgorithmVersion`を保存する。ユーザー訂正を不変Sourceとして追加し、訂正版snapshotを持つRevisionからSummaryとActionsを再生成する。role別原文・統合・訂正版・訂正履歴・過去Summaryを詳細画面で確認できるようにする。
- 共通辞書とGroup／案件別辞書をユーザー確認付きで登録し、追加専用のRevisionで更新する。適用内容を`dictionaryRevisionRefs`として固定し、同時適用は共通辞書＋明示選択した1つのGroup辞書までとする。辞書の修正・削除後も過去Analysisを再現できるようにする。
- 文字起こし後の決定的な表記補正を`normalizationAlgorithmVersion: 1`の完全一致規則で適用する。補正前Transcriptと、適用項目・位置・`dictionaryRevisionRefs`・未適用理由を含む補正履歴を保存する。
- 次回録音でのWhisper用語ヒントはPoCで方式・語数上限・効果を確認した後に有効化する。段階1〜5をこのPoC待ちにせず、Whisperモデル自体の学習・fine-tuningは行わない。
- Analysis詳細上部の「あなたの対応」、自分の対応／他者への依頼／返答待ちの独立表示、Action根拠・期限候補・状態の保存、確認済みActionからのTask作成。
- 初期無効のSlack／Teams録音確認を提供する。前面20秒継続、アプリ別既定60分cooldown、15分snooze、60分抑制、当日抑制、対象アプリ・閾値・cooldown設定を実装する。自動録音と会議・マイク・UI内容の精密検知は行わない。
- 未実装の次期項目として、選択中マイク名、開始前／録音中の入力レベル表示、マイク／システム音声を分けた無音・切断候補の警告を提供する。切断・権限喪失時は取得済みマイク原本を確定し、システム音声を可能な限り継続保存する。無断デバイス切替と自動停止は行わない。

## Phase 5: Output Support

- Source単体、複数Source、Groupから新しい派生Sourceを生成する。
- 派生Sourceを後続の生成に再利用し、親Sourceと実際に使用した個別Source IDを逆引きできるようにする。
- Markdownを共通成果物として、Jira、Confluence、Backlog向けのAdapterで各サービス形式へ変換する。
- Jira Issue、Confluence Page、Backlog Issue／Wikiの公開前に、本文、Project／Space、種別、添付をユーザーが確認・承認する。
- 元Markdownの添付、公開先、remote ID／Issue Key、URL、送信本文、添付、元Source、使用モデル、結果の監査保存。
- 作成成功後の添付失敗は既存remote IDに限定して再実行し、結果不明の新規作成は自動再試行しない。
- API Token等をmacOS Keychainへ保存し、アプリ内部だけで参照を解決する。UserDefaults／plist、Vault、Markdown、ログ、Git、URL query、プロセス引数、環境変数、診断、クラッシュ情報、HTTPデバッグ出力へ保存・出力しない。
- Excel、PowerPointなどの成果物生成。
- 外部サービスへの反映はユーザー承認後に行う。Jira／ConfluenceのCloud・Data Center種別とBacklogの会社環境は未決のため、Adapter実装前に公式資料で確認する。
