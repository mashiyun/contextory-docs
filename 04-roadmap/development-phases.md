# 開発フェーズ

## 正規モデル実装順

未実装の正規モデルは、次の順で実装する。後続のPhaseは前段の永続化・検証条件を満たすまで開始しない。

1. Source統一・legacy移行
2. Group
3. Task–Source／Group紐付け
4. Task手動追加・編集・コメント
5. 汎用Revision・追加情報
6. Topic Source・話題分割
7. Topic SourceからTask作成
8. 返答待ち・blocker
9. デイリーブリーフィング
10. WBS親子・依存関係
11. AI相談
12. Decision Log／RAID
13. 派生Output・外部公開

## 実環境フィードバック反映の実装順

実利用で確認した不具合は、次の順で実装する。前段の永続化・検証条件を満たすまで後続を開始しない。

1. v0.3.4: Analysis詳細の段階読込。選択直後はタイトルとActionsだけを軽量な表示投影から表示し、Markdown、Revision、追加情報、根拠Source、媒体・Transcriptを自動読込しない。「詳細を表示」を押した対象だけをバックグラウンドで読込み、対象IDと選択世代を照合して古い結果を破棄する。読込中・失敗時もInputと一覧操作を止めず、永続schema・保存本文・来歴を変更しない。
2. Analysis一覧のJST日時表示と簡素化。一覧を具体的な約10〜20 Characterの要約とJST日時だけにし、技術情報を詳細・診断画面へ移す。保存時刻はUTCのISO 8601のままとする。同一要約・同一分の場合だけ秒、なお一致する場合だけ小数秒へ拡張する。`presentationSummary`とlegacy `presentationTitle`のeffective summary解決規則を読込・書込・再索引で統一し、保存値を変更せず、20 Character超過時だけ表示を19 Character＋`…`へ短縮する。version 1／2／3 readerをwriterより先に提供する。
3. 解析成功後の状態競合修正。`operationId`をJob作成時にUUIDで確定して実行前に永続化し、Analysis保存・Summary保存・親Manifest更新を`AnalysisStore`へ集約する。SQLite migration、Bundle検証、legacy nullを除く部分一意索引の順で導入し、検証後だけ新規writerを有効化する。保存前失敗、保存後の状態同期失敗、保存物の整合性失敗を分離する。`completion_sync_pending`は起動時復旧対象として試行開始前に回数を永続化し、5秒・30秒・5分の最大3回で再同期、3回失敗後は`completion_sync_failed`とする。部分保存・破損は`analysis_integrity_failed`で自動処理を止める。
4. canonicalモデル統合。旧Analysisは一回限りのimporterでcanonical SourceとRevision 1へ変換し、検証後にFinderのゴミ箱へ移動する。ゴミ箱移動に失敗した場合は元の所有pathへ残してAnalysis writerを停止する。旧Context、旧Task Bundle、旧Analysis ID索引、旧Bundleを対にした削除は通常経路から撤去する。削除はcanonical Source Bundleだけを対象にし、参照確認、1回確認、回復可能なTrash transactionを維持する。

Transcript訂正、用語辞書、用語ヒント、専用の高精度再文字起こしはRetiredである。Appの撤去と検証が完了するまで、[Transcript訂正・用語辞書要件](../02-requirements/transcript-correction-terminology.md)と[ADR-014](../06-adr/ADR-014-transcript-correction-terminology.md)を履歴として保持する。

## 次のLibrary安全管理実装順

v0.3.1までに以下の導線を実装したが、未参照Source候補と削除復旧にはv0.3.2の緊急安全修正が必要であり、修正・独立レビュー・回帰検証が完了するまで削除導線の完了を主張しない。実利用Vaultは検証対象にしない。

1. Analysis一覧の要約を、保存値を変更せず表示時だけ約10〜20 Characterへ短縮した。20 Character超過時は19 Character＋`…`とし、legacy値にも同じ規則を適用する。
2. v0.3.1で現行Appが読める参照だけを使う未参照Source一覧を追加した。v0.3.2では候補を検証済みcanonical `kind: input` Sourceへ限定し、Analysisを含む派生・legacy・種別不明Sourceを候補から恒久的に除外する。
3. TaskとGroupの削除導線をarchiveへ統一した。Bundle、link、履歴を保持し、状態だけを保存する。
4. v0.3.1で未参照Sourceの複数選択Trash移動を実装した。v0.3.2では削除復旧を再索引・サイドバー集計・一覧走査より先に有効recordだけ完了させ、不正・非Job・競合recordは削除・Source解釈せず安全に隔離する。隔離中は未参照一覧と削除操作をfail-closedにするが、有効なAnalysis一覧・件数表示は継続する。
5. v0.3.2で、全Analysis表示のサイドバー見出しを「Analysis」へ修正する。「未確認Analysis」は確認待ちだけへ検証可能に絞った投影以外では使用しない。

## 次期Topic Source・Task・PM実装順

Group実装後は、[Topic Source・Task・WBS・PM支援要件](../02-requirements/topic-source-task-wbs.md)と[ADR-015](../06-adr/ADR-015-topic-source-task-wbs.md)に従い、次の順で実装する。各段階は前段の正本・整合性検証を満たすまで開始しない。

1. Task–Source／Group紐付け
2. Task手動追加・編集・コメント
3. 汎用Revision・追加情報
4. Topic Source・話題分割
5. Topic SourceからTask作成
6. 返答待ち・blocker
7. デイリーブリーフィング
8. WBS親子・依存関係
9. AI相談
10. Decision Log／RAID
11. 派生Output・外部公開

段階1〜2はcanonical `task.json` reader、SQLiteの必要な表、Bundle検証、writerの順で導入する。旧Task Bundleは一括backfillせず、推測変換・通常読込を行わない。段階4のTopic writerは、Evidence Spanのsnapshot／hash／時刻／byte境界とSource DAG検証を実装してから有効化する。非Transcript長文のoffset形式が未決の間は固定Transcript snapshotを持つ音声・動画だけを対象とする。複数GroupのWBS表示順は段階8、PMカードの抽出規則は段階7／10の開始ゲートであり、段階1〜6を妨げない。

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
- 実装中: Source／Group／Analysisのcanonical統合。旧Contextを通常Readerから外し、解析目的ごとの派生結果をcanonical Sourceとして追加保存する。
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
- 実装中: 旧`analyses/`を一回限りのimporterとしてcanonical `sourceId`とRevision 1へ変換し、検証後に通常保存域から退避する。旧`contexts/`と旧Task Bundleは通常Readerで扱わない。
- 実装済み: 新規Analysis staging Jobの復旧要求保存、同一`operationId`の重複防止、未確定の複数Source Analysisを通常Input Queueへ分解しない復旧。
- 未実施: 実利用Vaultでの変換実行。既存Bundleを推測で一括変換・削除しない。Revisionの新規書き込みUIは別途実装する。
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
- Group／Task／派生Source／外部公開記録を含む「参照確認→削除確認→macOSのゴミ箱へ移動」の順で削除する。
- Analysis一覧を、内容が分かる具体的な要約とJST日時だけの表示へ簡素化する。Analysis表記、分類、状態、hash、Source IDは詳細・診断画面へ移す。要約の根拠・確認状態・ユーザー修正を保存し、未生成時はSource種別とJST日時で暫定表示する。同一要約・同一分の集合だけ秒、なお一致する場合だけ小数秒まで拡張する。
- `presentationSummary`を新規書き込み先とし、legacy `presentationTitle`を読み取り互換で残す。effective summaryの優先順位を読込・書込・再索引で統一し、読込時のbackfillと一括更新を行わない。
- 解析成功後の状態競合修正。Revisionの不変Summary snapshot検証を成功判定とし、`AnalysisStore`を親Manifest更新の唯一の所有者にする。`operationId`の事前確定・一意制約・fail-closed再索引、保存前失敗・`completion_sync_pending`・`analysis_integrity_failed`の分離、起動時復旧、試行前の回数永続化、最大3回の自動再同期、`completion_sync_failed`と手動再同期を実装する。
- role別生Transcriptの原本保持とローカル前処理は維持する。訂正Source、用語辞書、決定的補正、用語ヒント、専用の高精度再文字起こしはRetiredであり、App撤去と検証の完了までは履歴仕様を参照する。
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
