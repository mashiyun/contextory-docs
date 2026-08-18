# Source・Group・Task・Output要件

## Status

正規Sourceモデルとlegacy Analysis移行の基盤、Analysis一覧の表示要約短縮、Task／Group archiveは実装済みである。この開発サイクルの最終全検証は別途実施する。v0.3.1の未参照Source一覧と複数選択破棄は、この文書のv0.3.2安全契約を満たさないため、修正と検証が完了するまで安全な削除導線として扱わない。v0.3.4では、Analysis選択直後をタイトルとActionsだけの段階表示にする受入仕様を確定し、実装と検証を本サイクルで行う。解析完了判定の修正、Revision／Group UI、Action管理、派生Outputと外部公開は未実装である。Transcript訂正と用語辞書は[Transcript訂正・用語辞書要件](transcript-correction-terminology.md)、次期のTopic Source、手動Task、WBS、PM支援は[Topic Source・Task・WBS・PM支援要件](topic-source-task-wbs.md)に分離する。

## Source中心モデル

Contextoryで保存する情報と成果物は、入力かAI生成かを問わず、同じ`Source`として扱う。Input、Analysis、Output、Topic Source、External Ticket SourceはUIや処理段階の呼称であり、永続モデルの別物ではない。

- キャプチャ、録音、録画、PDF、手動入力は一次Sourceである。
- AI Analysis、議事録、要件定義、返信案などの生成結果も派生Sourceである。
- 会議、動画、長文の一部を固定根拠で切り出すTopic Sourceも派生Sourceである。詳細は[Topic Source・Task・WBS・PM支援要件](topic-source-task-wbs.md)を参照する。
- Jira／Backlogチケットを不変snapshotとして保存するExternal Ticket SourceもInput Sourceである。詳細は[External Ticket Source要件](external-ticket-source.md)を参照する。
- Source単体または複数Sourceを材料として、新しい派生Sourceを生成できる。
- 派生Sourceをさらに別の生成へ再利用できる。
- 生成時に使用した親Source IDを固定して記録し、元Sourceを上書きしない。
- Sourceは`input`、`analysis`、`output`などの種別と、一次か派生かを表す属性を持つ。既存のAnalysis Bundleは移行期間の互換表現であり、新モデルでは対応するAnalysis Sourceとして扱う。
- 親Source ID、生成操作、生成日時、使用モデル、実際に使用したSource IDは派生Sourceの来歴として固定保存する。
- 正規IDは`sourceId`とする。既存の`analysisId`は読み取り互換用に`sourceId`との対応を保持し、新規書き込みの識別子として使わない。

## Group

Groupは、すぐに統合・出力しない関連Sourceを案件、テーマ、顧客、機能などの文脈で集める入れ物である。

- Groupへ追加しても新しいAnalysisやOutputを自動生成しない。
- 同じSourceを複数Groupから参照できる。
- Groupへ後からSourceを追加できる。
- AIは追加先Groupを提案できるが、ユーザーが確認・修正できる。
- Group全体から生成するときは、その時点で実際に使用した個別Source IDを派生Sourceへ記録する。
- Group–Source関係の正本はGroup Bundleの`group.json`であり、Group側の`sourceLinks`へSource ID、役割、追加日時を保存する。Source Manifestの`groupIds`は既存データの読み取り用cacheであり、新規更新しない。

## Analysis Source Revision

Analysis Sourceは詳細画面では同じSourceを継続して更新するように見せる。ただし、追加情報、AIの解釈、過去の要約を破壊的に上書きしない。

### 必須要件

- Analysis Sourceへ直接追加できる情報は、テキスト、画像、URLに限定する。
- 追加画像は、その時点で新規の不変Sourceとして保存し、RevisionからID参照する。元Analysis Sourceや既存の親Sourceへ画像を埋め込んで上書きしない。
- 音声、動画、PDFはRevisionへ直接追加しない。独自前処理を伴う新規Sourceとして取り込み、必要に応じて同じGroupへ追加してAnalysisを生成する。
- Revisionは連番、作成日時、作成理由、追加したSource ID、追加情報種別、ユーザー指示、使用Provider／モデル、確認状態、直前Revisionとの差分を持つ。
- 各Revisionはsummary本文の不変スナップショットを必ずファイルとして保存し、`summaryPath`とSHA-256を記録する。ハッシュだけ、または最新`summary.md`だけをRevisionの保存として扱わない。
- 初回AnalysisはRevision 1として、以後の再分析は各Revisionとして、実際にClaudeへ渡したfile集合を`stagedInputRefs`へ追加専用で固定する。一時staging directoryは正本にしない。
- AIは現在のsummary、今回の追加情報、今回のユーザー指示を中心に再分析し、最新表示用の`summary.md`を更新する。根拠として使用した個別Source IDも固定保存する。
- 過去のsummaryと追加情報はRevision単位で保持する。最新の`summary.md`は最新Revisionの不変スナップショットから再生成するmaterialized viewであり、構造化Revision履歴から再生成できる。
- 詳細画面では、Revision回数、追加情報、各Revisionのsummary、直前との差分、根拠Sourceを確認できる。
- AI対話によってsummaryが変わる場合も、対話履歴を正本として保持したうえで、変更後のsummaryを新しいRevisionとして保存する。

### 受入条件

- テキスト、画像、URLを追加しても同じAnalysis Sourceを開け、過去のsummaryと追加情報を失わない。
- 任意のRevisionについて、理由、追加情報、根拠Source、使用モデル、差分を確認できる。
- 任意のRevisionについて、保存済みのsummary本文、`summaryPath`、SHA-256を確認できる。
- 最新`summary.md`を削除または再生成する場合でも、最新Revisionの保存済みsummaryから同じ内容を復元できる。
- 任意のRevisionについて、`stagedInputRefs`と不変input snapshotまたは削除保護されたSource原本から、当時Claudeへ渡したbyte集合を再現・検証できる。

## Analysis一覧の表示項目

Analysis一覧は、内容が分かる具体的な要約とJST日時だけを表示する。実利用では、Analysis表記、分類名、処理状態、hash、Source IDが並ぶと、どの一覧項目が何の内容かを判別できなかった。技術情報は一覧から外し、詳細画面と診断画面で確認する。

- 一覧の表示項目は、内容が分かる具体的な要約と、JSTへ変換した日時の2つとする。
- Analysis表記、分類、処理状態、hash、Source ID、Revision番号、モデル名は通常の一覧へ表示しない。
- 詳細画面と診断画面では、Analysis Source ID、Revision、分類、処理状態、hash、使用モデル、生成日時を確認できる。
- 具体的な要約はAnalysis Sourceの表示メタデータとして保存し、本文、生成元Revision ID、根拠Source ID、生成Provider／モデル、生成日時、確認状態、ユーザー修正の有無を持つ。
- 保存済みの表示要約は変更・検証せず、一覧表示だけを短縮する。表示には実装言語の`Character`単位を一貫して使い、20 Characterを超える値は先頭19 CharacterとU+2026（`…`）を表示する。20 Character以下の値はそのまま表示し、locale、フォント、UTF-16 code unit、UTF-8 byte数で長さを判定しない。
- ユーザーはAI提案の要約を確認、修正、却下できる。ユーザー修正後の表示要約を、後続のAI再分析で黙って上書きしない。
- 分類は検索・絞り込みに使う属性であり、一覧の表示項目にはしない。
- 保存する時刻はUTCのISO 8601を正本とし、表示時にだけAsia/Tokyoへ変換する。既定の表示形式は分単位の`2026/08/14 10:30`とする。
- 表示要約が未生成の場合は推測せず、Source種別とJST日時による暫定表示を使い、`proposed`として確認対象にする。暫定表示はユーザー確定の要約ではない。
- 全Analysisを表示するサイドバー見出しは「Analysis」とする。「未確認Analysis」は、表示対象が確認待ち状態だけへ検証可能に絞られている場合にだけ使用できる。見出しと実際の絞り込みを一致させ、確認状態を持たない既存Analysisを未確認と推測しない。

#### 同一要約・同一時刻の区別

- 一覧の表示項目は要約とJST日時だけを維持し、hash、Source ID、短縮IDを表示しない。
- 通常は分単位で表示する。
- 表示要約とJST分が完全に一致する項目が複数ある場合だけ、その一致集合の日時を秒まで表示する。
- 秒まで表示しても一致する場合だけ、その一致集合の日時を小数秒まで表示する。
- 一致判定と桁の拡張判定はlocale非依存で決定的に行う。要約はUnicode正規化後のコードポイント列で比較し、日時はUTC正本の値で比較する。表示桁の拡張は一致集合の全行へ同時に適用する。
- 行の内部識別にはSource IDを使うが、画面へ表示しない。

#### 表示要約の保存と移行

- 新規書き込みは`presentationSummary`だけを使用する。legacyの`presentationTitle`へ新規に書き込まない。
- 読込時のeffective summaryの優先順位は次に固定する。
  1. ユーザー確認済みの`presentationSummary`
  2. ユーザー確認済みのlegacy `presentationTitle`
  3. `presentationSummary`
  4. legacy `presentationTitle`
  5. Source種別とJST日時によるfallback表示
- 両フィールドが存在する場合もlegacy値を削除しない。
- 読込時のbackfillと、既存Manifestの一括更新を行わない。
- 新フィールドへの投影は、次回の正規Revision書き込み時にappend-onlyで行う。
- schema version、旧version読込、新version書込、SQLite再索引のいずれでも同じeffective summary規則を使う。
- 確認状態とユーザー修正履歴はフィールド移行後も保持し、AIが上書きしない。
- legacy値を含む全てのeffective summaryは、読込時に保存値を変更せず同じ表示短縮規則を適用する。20 Character超過時だけ先頭19 Characterと`…`を表示し、20 Character以下はそのまま表示する。これは表示変換だけであり、legacy値を新規フィールドへbackfillしない。

### 受入条件

- 一覧から、各項目の内容とJST日時だけで内容を判別できる。
- 一覧にAnalysis表記、分類、処理状態、hash、Source IDが現れない。
- 詳細画面と診断画面から、Source ID、Revision、分類、処理状態、hash、使用モデルを確認できる。
- 保存済み時刻がUTCのISO 8601であり、表示だけがJSTである。
- 表示要約が未生成のAnalysisでも、Source種別とJST日時で一覧から識別できる。
- 表示要約とJST分が一致する項目が複数ある場合、その集合だけが秒まで、なお一致する場合だけ小数秒まで表示され、他の行の表示桁は変わらない。
- 同じデータからは、実行環境のlocaleに依存せず同じ表示桁と同じ並びが得られる。
- legacy `presentationTitle`だけを持つ既存Analysisでも、一覧が同じ優先順位でeffective summaryを表示し、読込によってManifestが更新されない。
- 新旧フィールドが併存する場合も、ユーザー確認済みの値が優先され、legacy値が削除されない。
- 任意の表示要約について、生成根拠、生成モデル、確認状態、ユーザー修正の有無を確認できる。
- 保存済みの要約は長さにかかわらず変更・保存拒否されず、20 Character超過時だけ一覧で先頭19 Characterと`…`が表示される。
- legacy `presentationTitle`を含む全ての値に同じ表示短縮規則を適用し、読込時にManifestを更新しない。

## Analysis詳細の段階読込

Analysis一覧で項目を選択する操作は、Inputの取得・取り込み、一覧の操作、ほかのウィンドウの応答を待たせない。選択直後に全ての詳細ファイルを読むのではなく、初期表示と明示的な詳細読込を分離する。これは表示順序の変更であり、Analysis、Revision、原本、追加情報の正本・来歴・削除保護を変更しない。

- 選択直後の初期表示は、一覧で解決済みのタイトルと、そのAnalysisに対する「あなたの対応」（Actions）だけとする。初期表示に必要な値は軽量な表示投影から取得し、Markdown本文、過去Revision、追加情報、根拠Source、画像・音声・動画、媒体metadata、role別／統合／訂正版Transcriptを展開・走査しない。
- 初期表示には「詳細を表示」操作を置く。選択だけで全詳細の読込を開始しない。
- ユーザーが「詳細を表示」を実行した時だけ、その時点で選択中のAnalysisのMarkdown、Revision、追加情報、根拠Source、媒体・Transcriptをバックグラウンドで遅延読込する。既存の詳細表示で確認できる情報は、この操作後も省略・破棄しない。
- 詳細読込中もタイトルとActions、一覧、Input操作を利用可能なままにする。読込中の状態、失敗、再試行は詳細領域内に表示し、一覧全体やInputをブロックしない。
- 選択変更、詳細表示の取り消し、ウィンドウ終了後に完了した古い読込結果は、現在の選択・表示要求を上書きしない。対象Analysisと選択世代を照合してから画面へ反映する。
- 初期表示または遅延読込の失敗は、永続データを変更せず、失敗した領域だけへ表示する。失敗を理由にAnalysisの再解析、Revision再生成、Source削除を自動実行しない。

### 受入条件

- 大きなMarkdown、複数Revision、追加情報、媒体またはTranscriptを持つAnalysisでも、一覧項目の選択直後にタイトルとActionsの初期表示へ遷移し、全詳細の走査完了を待たない。
- 初期表示中にMarkdown本文、過去Revision、追加情報、根拠Source、画像・音声・動画、媒体metadata、Transcriptの内容が表示・読込対象にならず、「詳細を表示」を実行した対象だけが遅延読込される。
- 「詳細を表示」後は、従来のMarkdown、Revision、追加情報、根拠Source、媒体・Transcriptの詳細を確認できる。
- 詳細読込中に別のAnalysisを選択しても、前のAnalysisの完了結果や失敗表示が新しい選択へ混入しない。
- 詳細読込の実行中・失敗時も、一覧の再選択とInput取得・取り込みを継続できる。
- この表示最適化でSource／Analysis／Revisionの永続schema、保存済み本文、来歴、削除規則を変更しない。

## 解析完了判定と状態表示

実利用で、Claude解析とAnalysis保存が成功しているのに「自動解析失敗」と表示される事象を確認した。原因は解析結果の保存後に行う親Manifestの完了更新であり、解析そのものの失敗ではない。解析の成否は、解析結果の保存が完了したかどうかで判定する。

### 必須要件

- canonical Analysis Source Manifest、最新Revision record、Revisionの不変Summary snapshot、保存済みSHA-256の検証に成功した時点で、その解析は成功として扱う。最新表示用`summary.md`はmaterialized viewであり成功境界へ含めず、欠損時はRevision snapshotから再生成する。
- 解析結果の保存前に発生した失敗と、保存後の状態同期に発生した失敗は、別の状態・別の表示として扱う。
  - 保存前の失敗は既存の`analysis_failed`／`retry_waiting`とし、再解析の対象とする。
  - 保存後の状態同期失敗は`completion_sync_pending`とし、Analysisは有効なものとして表示する。再解析の対象にせず、状態の再同期だけを行う。
- 完成済みのAnalysisを再生成しない。同じ`operationId`に対して重複するAnalysis Sourceを登録しない。
- 「自動解析失敗」は、Claude解析またはAnalysis保存が実際に失敗した場合だけ表示する。

#### `operationId`の確定と所有権

- `operationId`はProcessing Job作成時にUUIDとして確定し、Claude実行前に永続化する。実行後に採番しない。
- 新規のcanonical Analysis Sourceでは`operationId`を必須とし、Analysisの`generation.operationId`にも同じ値を保存する。
- JobとAnalysisの`operationId`はSQLiteへ索引化する。legacyのnull行を除く部分一意制約とし、新規writerはnullを拒否する。
- Bundle走査による再索引で同じ`operationId`の重複を検出した場合はfail-closedとする。重複を自動統合・自動削除せず、新規書き込みを止めて手動レビュー対象にする。
- Analysisの保存、Summaryの保存、親Source Manifestの完了更新は、実装上の`AnalysisStore`を唯一の所有者とする。
- `CaptureModel`はQueue、Processing Job、UI状態だけを担当し、親Manifestを更新しない。同じ親Manifestの完了更新を二重に行わない。
- 親Source Manifestの完了投影は`analysisCompletions`の追加専用recordとし、`operationId`、Analysis Source ID、Revision ID、Summary snapshotのSHA-256、UTC完了日時を持つ。同じ`operationId`・同じ参照・同じhashのrecord追加は冪等な成功とし、同じ`operationId`で参照またはhashが異なるrecordは`analysis_integrity_failed`として上書きしない。別`operationId`のrecordは既存recordを削除せず追加する。
- Processing Jobは完了投影先を`originatingSourceId`として1件だけ固定する。自動Input解析では取得元Input Source、Analysis Revision再生成では対象Analysis Sourceとする。複数の`usedSourceIds`やGroupメンバーのManifestへ完了状態を配布せず、来歴はcanonical Analysis側に固定する。
- 親Manifestの更新が競合した場合は最新Manifestを再読込する。同じ`operationId`・参照・hashの`analysisCompletions` recordが既にあればProcessing Jobを`completed`へ収束させ、recordがなければ追加を再試行する。競合そのものを解析失敗として扱わず、再試行でも更新できなければ`completion_sync_pending`とする。
- 復旧した`pending_analysis`／`analyzing` JobもClaude実行前に`operationId`でAnalysisを検索する。成功境界を満たすAnalysisがあれば再生成せず、親Manifestの完了状態が同じ`operationId`を示しているか、`AnalysisStore`による更新が成功した場合だけJobを`completed`へ収束させる。親Manifestを更新できなければ`completion_sync_pending`とする。同じ`operationId`のAnalysis Sourceが存在するのに成功境界を満たさない場合は`analysis_integrity_failed`として停止し、Claudeを再実行しない。

#### `completion_sync_pending`の復旧

- `completion_sync_pending`は起動時の復旧対象に含める。`pending_analysis`／`analyzing`と同様に、永続状態から古い順で復旧キューへ復元する。
- `completion_sync_pending`と`completion_sync_failed`はProcessing JobのSQLite運用状態を正本とする。更新に失敗した親Manifestへこの運用状態を書き込めたと仮定せず、Sourceの表示状態はJobとの対応から導出する。
- 復旧時はcanonical Analysis Source Manifest、最新Revision record、不変Summary snapshot、`operationId`の一致、保存済みSHA-256を検証する。完成済みと確認できた場合はAnalysisを再生成せず、親Source ManifestとProcessing Jobの状態だけを再同期する。最新表示用`summary.md`だけが欠けている場合はRevision snapshotから再生成する。
- canonical Analysis Sourceが存在しない場合だけ保存前失敗として既存の再解析経路へ戻す。Analysis Sourceが存在するのにRevision、Summary snapshot、`operationId`、SHA-256のいずれかが不正な場合は`analysis_integrity_failed`としてfail-closedに停止し、自動再解析・自動上書き・自動削除を行わず手動レビュー対象にする。完成済みAnalysisの親Manifest同期だけが失敗した`completion_sync_failed`と混同しない。
- Processing Jobは再同期用の永続項目として`syncAttemptCount`、`lastSyncAttemptAt`、`nextRetryAt`、`lastSyncError`を持つ。
- 自動再同期は最大3回とし、間隔は5秒、30秒、5分とする。`nextRetryAt`到達前に自動実行しない。
- 各自動再同期の開始前に、SQLite transactionで`syncAttemptCount`の加算、`lastSyncAttemptAt`、次の`nextRetryAt`を永続化してから親Manifest更新を試みる。更新中の終了・異常終了も1回として数え、再起動で試行回数を巻き戻さない。
- 3回失敗した場合は`completion_sync_failed`へ遷移し、自動処理を停止する。診断情報（最後のエラー、試行回数、対象Analysis Source ID、`operationId`）を表示し、手動再同期を提供する。
- 手動再同期もAnalysisを再生成せず、状態の再同期だけを行う。
- `completion_sync_pending`と`completion_sync_failed`のいずれも「自動解析失敗」として表示せず、再解析キューへ投入しない。
- `analysis_integrity_failed`は「解析保存物の整合性エラー」として表示し、破損・部分保存の対象ファイルと検証結果を診断へ示す。明示的な修復または新しい`operationId`での再生成をユーザーが選ぶまで自動処理しない。

### 受入条件

- Analysis SourceとSummaryが保存されている場合、一覧・詳細・常駐メニューのいずれにも「自動解析失敗」が表示されない。
- 親Manifestの更新が競合しても、Analysisが重複登録されず、Jobが`completed`へ収束する。
- `completion_sync_pending`のまま終了・異常終了しても、次回起動時に復旧対象として復元される。
- 復旧・手動再同期のいずれでも、完成済みAnalysisが再生成されず、Analysis Sourceが重複しない。
- 自動再同期が3回を超えて実行されず、3回失敗後は`completion_sync_failed`として診断と手動再同期の導線が表示される。
- 同じ`operationId`を持つJobまたはAnalysisが2件以上ある状態では、再索引がfail-closedで停止し、新規書き込みが行われない。
- 同じ`operationId`のAnalysis Sourceが存在してもRevisionまたはSummary snapshotの検証に失敗する場合、`analysis_integrity_failed`としてClaude再実行・上書き・削除が自動実行されない。
- 再同期中に終了しても開始済み試行が`syncAttemptCount`へ残り、再起動を繰り返しても自動試行が3回を超えない。
- 手動再実行の対象が、実際に解析または保存へ失敗したSourceに限定される。

## Actionsの強調とTask化

Analysis Source詳細の上部には「あなたの対応」を大きく表示し、Summary本文とは別にActionを表示する。段階読込では、タイトルとともにこの領域を初期表示し、Markdown本文その他の詳細は「詳細を表示」後に読む。ActionはAIの提案であり、確認前に完了・確定として扱わない。

- Summaryの構造化出力からActionだけを独立抽出し、自分の対応、他者への依頼、返答待ちを区別する。
- Actionは内容、種別、期限候補、状態、根拠Source ID、抽出元Revision ID、確認状態を持つ。
- 自分の対応、他者への依頼、返答待ちは表示上も区別する。
- Actionがない場合は「現在必要な対応はありません」と表示する。
- 将来、確認済みActionからTaskを作成できるよう、Actionから作成したTask IDを保存できる設計にする。Task作成は別操作であり、AIが自動作成しない。

### 受入条件

- Summaryの本文を読まなくても、現在の対応・依頼・返答待ちを区別して確認できる。
- Actionの根拠となるRevisionとSourceへ逆引きできる。
- ActionがないAnalysisでは空の対応リストではなく所定のメッセージを表示する。

## 原本閲覧と追加可能範囲

Analysis Sourceの詳細画面から、根拠となる原本へ戻れることを必須とする。

- 画像原本はアプリ内でプレビューできる。
- 音声・動画原本はアプリ内で再生できる。
- 原本、追加画像、追加テキストの保存場所はFinderで表示できる。
- Revisionに追加した画像とテキストも、追加情報ごとに閲覧できる。
- URLは出典として追加できるが、URL自体から自動取得・自動解析はしない。

## Source削除

Source削除は、複雑なロック操作ではなく、実際に利用中の参照確認と1回の明示確認で行う。完全削除はせず、対象BundleをmacOSのゴミ箱へ移動する。既存Manifestの`protection`は読み取り互換と削除transactionの内部復旧にだけ保持し、画面にロック解除・恒久ロックの操作を表示しない。

- 削除の順序は「参照確認 → 削除確認 → macOSのゴミ箱へ移動」とする。
- ロック解除は通常の閲覧、再分析、Task作成、削除のいずれにもユーザー操作として表示しない。
- 削除前にTask、Group、別Analysis／Revision、Addition画像Source、明示追加context Source、`stagedInputRefs`、Topicの親参照／Evidence Span／proposal、Taskのコメント・blocker・変更根拠、派生Source、外部公開記録からの参照整合性を`VaultMutationLock`内で確認する。参照中、参照走査不能、参照先欠損、path不正、hash不一致では削除を中止して判明した参照先を表示し、RevisionまたはAnalysisの削除でもcascade deleteしない。
- 親Sourceの`analysisCompletions`は追加専用の完了監査記録であり、利用中のAnalysisへの参照として削除を恒久的に妨げない。Analysis Sourceを削除してもこの監査記録は書き換えずに残し、削除済みAnalysisへの過去記録として扱う。Task、Group、別Analysis／Revision、Addition、来歴などの実利用参照は引き続き削除を妨げる。
- 条件を満たす削除は即時完全削除せず、Analysisと元データを含む対象BundleをmacOSのゴミ箱へ移動する。

### legacy互換不一致だけのAnalysis Source cleanup例外

通常のSource削除は、Bundle directory名とManifestのID・親IDを含む既存の厳格な一致検証を維持する。ただし、移行済みAnalysis Sourceを削除するためだけに、legacy Bundleのdirectory名および／またはlegacy AnalysisManifestの`sourceIds`の不一致を許容するcleanup例外を設ける。この例外は欠損・不正・未知の情報を補う仕組みではなく、次の条件をすべて満たす場合だけに限定する。

- 削除対象は`kind: analysis`、`schemaVersion: 3`の移行済みcanonical Analysis Sourceであり、解決済みlegacy Manifestの`schemaVersion`は厳密に`2`である。Input、Output、Topic、External Ticket、未移行Analysis、またはcanonical Analysis Sourceだけの削除には適用しない。
- canonical `sourceId`は削除要求、canonical Bundle directory名、canonical Manifestで完全一致し、canonical Bundle path、Manifest、必要なhashを通常削除と同じ規則で検証できる。canonical側のpathまたはManifest不一致をこの例外で許容しない。
- canonical側に保存したlegacy `analysisId`対応と、legacy Manifestの`analysisId`を検証し、対応するlegacy Analysis Bundleをちょうど1件だけ解決できる。0件または複数件、ID不正、対応不明はfail-closedで中止する。
- legacy Bundleの探索は承認済みlegacy root配下を再帰的に行ってよいが、解決結果はroot自身ではなく、そのrootに所有されるBundle境界の子directoryでなければならない。root直下の`analysis.json`、任意の親directory、非Bundle directoryを一致候補として採用せず、rootまたは複数Bundleをゴミ箱へ移動しない。
- 許容する不一致は、解決済みlegacy Bundleのdirectory名とlegacy `analysisId`、およびlegacy AnalysisManifestの`sourceIds`だけである。それ以外のidentity、schema、所有境界、参照、path、hashの検証失敗は通常どおり削除を中止する。
- `VaultMutationLock`内で参照整合性をfail-closedで走査する。走査不能、欠損、未知schema、参照中、参照先の不整合ではcleanupを実行せず、canonical／legacyのいずれの参照先もcascade deleteしない。
- ユーザー確認は1回とする。prepareで候補と検証結果を固定し、対象（canonical／legacy）、参照、identity、path、hashを再検証してからゴミ箱へ移動する。commit直前にも同じ検証を再実行し、いずれかが変化または失敗したら中止する。
- canonical Bundleと解決済みlegacy Bundleのゴミ箱移動は、永続化したprepare／commit記録を使う1つの回復可能な論理transactionとして扱う。片方だけの移動を成功とせず、移動失敗・中断・異常終了では起動時復旧で移動済みBundleを元の所有pathへ戻して`locked`へ収束させる。復元不能または状態不明ならfail-closedで隔離し、ユーザーの手動レビューまで再試行・削除を行わない。起動時復旧は再索引、サイドバー件数集計、Analysis一覧、未参照候補走査より先に有効な復旧recordを完了させ、不正・欠損・競合recordは削除・Source解釈せず隔離しなければならない。

### 受入条件

- ロック中のSourceは削除操作を実行できない。
- 削除にはロック解除確認と削除確認の両方が必要である。
- 参照中のSourceはゴミ箱へ移動せず、影響する参照を確認できる。
- backfill失敗、削除キャンセル、削除失敗、異常終了後に、Sourceが削除可能な解除状態として残らない。
- legacy cleanupは、`schemaVersion: 3`の移行済みcanonical Analysis Sourceと、`schemaVersion: 2`のちょうど1件の所有下legacy Bundleだけを対象にし、legacy側のdirectory名または`sourceIds`以外の不一致では実行されない。
- legacy root、非Bundle directory、0件または複数件のlegacy候補はゴミ箱へ移動せず、canonical／legacyの片方だけが移動した場合は起動時復旧で削除前のロック状態へ戻る。

## 未参照Source一覧と複数選択破棄

タスク整理画面は、削除候補を見つけるための「未参照Source」一覧を提供する。未参照は、現行Appが読めるSource、legacy Analysis／Context、Group、Taskの参照fieldに当該canonical `sourceId`がないことだけを指す。これは現在読める参照に基づく便宜的な一覧であり、Vault全体の完全な参照グラフを保証しない。削除候補にできるのは、Manifest、Bundle境界、種別を検証できるcanonical `kind: input` Sourceだけである。`kind: analysis`を含む全ての派生Source、legacy Analysis／Context、種別・schema・Bundle境界を検証できないSourceは、参照が0件に見えても候補にしない。読取エラーや未対応形式では、一覧を表示できない通常のエラーを示す。

- 一覧は現行Appが読める参照から見て未参照であり、かつ上記の候補資格を満たすSourceだけを候補にする。候補のSource種別、表示要約またはfallback、JST日時、容量を表示し、削除の可否は既存のSource削除規則に従う。候補判定は削除可否の代替ではない。
- ユーザーは候補を明示選択する。空の選択では開始せず、一覧の再読込やアプリ再起動で削除を予約・自動実行しない。
- 複数選択削除では、対象件数と「macOSのゴミ箱へ移動、直ちに完全削除しない」を示す明示確認を求める。確認後、選択したSourceを順に既存のSource単位のゴミ箱移動として処理する。batch用のcandidate snapshot永続化、全件atomic transaction、特別な全件再検証、batch rollbackは行わない。
- 各Sourceの既存削除に失敗した場合は通常のエラーを表示し、他の選択Sourceの処理結果とともに一覧を更新する。選択外のSourceを変更せず、cascade deleteを行わない。
- 削除復旧recordは専用transaction directoryに`job.json`とpayloadを組として保持する。`job.json`を持たないentryを削除JobまたはSourceとして解釈・移動・削除しない。復旧recordの欠損・不正・復元先競合は、そのrecordを安全に隔離して削除導線をfail-closedにし、診断へoperation IDと安全な理由だけを表示する。有効な復旧recordの処理と、復旧済みSource／Analysisの索引化・閲覧・件数表示は継続する。

### 受入条件

- 現行Appが読める参照から見て参照中のSourceは未参照一覧へ出ない。
- Analysis Source、他の派生Source、legacy Analysis／Context、種別またはBundle境界を検証できないSourceは、参照数にかかわらず未参照一覧へ出ない。
- 複数選択したSourceだけに明示確認後の順次ゴミ箱移動を実行し、各Sourceの成功・失敗を表示する。選択外のBundleを移動・更新しない。
- legacy Analysis cleanupの例外は未参照一覧・一括削除に適用されず、同cleanupだけの専用操作に限定される。
- 起動中の削除復旧が未完了または隔離された場合、未参照一覧と削除操作は有効化しない。一方で、隔離recordを削除JobまたはSourceとして解釈せず、有効な復旧recordと復旧済みAnalysisの一覧・件数表示を継続する。

## Task

Taskは具体的な対応、期限、状態、待ち先、完了条件を管理する。

- Taskは複数Sourceを紐づけられる。
- Taskは複数Groupを紐づけられる。
- 同じSourceまたはGroupを複数Taskから参照できる。
- TaskへGroupを紐づけた場合、日常の整理画面ではGroupの最新状態を参照する。
- AI生成時はGroup参照だけを根拠として残さず、実際に使用した個別Source IDを固定して記録する。
- Task内のSourceには、根拠、文脈、入力、下書き、確定成果物などの役割を設定できる設計とする。
- Task–Source／Task–Group関係の正本はTask Bundleの`task.json`である。`sourceLinks`と`groupLinks`へ相手ID、role、追加日時を保存する。
- Source Manifestの`taskIds`は既存データの読み取り用cacheであり、新規更新しない。双方向のBundle更新を行わず、SQLiteは索引として再構築する。
- MVPの「削除」はhard deleteではなくarchiveである。Task／Group Bundleと既存のlink・履歴を保持し、関連SourceやGroupを移動・unlink・削除しない。
- Taskの詳細な手動管理、コメント、返答待ち、blocker、WBSは[Topic Source・Task・WBS・PM支援要件](topic-source-task-wbs.md)に従う。

## 画面内URLの出典化

Backlog、Jira、Confluence、Slack、メール等のキャプチャでは、ユーザーが可能な範囲でアドレスバーやURLを画面内へ含められる。ユーザーが明示選択した原画像は、会社契約Claude Codeへ未マスクで送信できる。Vision OCR、URL領域マスク、マスク失敗時の送信停止は必須としない。原画像はローカルに不変のSourceとして残し、画像、`provided-text`、ユーザー入力からURLを抽出・保存・表示する場合はquery／fragmentを除去した安全化URLを使う。

### 必須要件

- 画像からのURL抽出は任意機能とし、送信の前提にしない。抽出する場合は抽出方法と元Source IDを記録し、認識できない値を推測・補完しない。
- `provided-text`とユーザー入力のURLは、ローカル解析で送信用に安全化する。
- URLごとに、抽出元Source ID、抽出方法（画像、提供テキスト、ユーザー入力）、`aiDisplayUrl`、ローカル遷移専用の`localOpenUrl`、確認状態を保持する。
- `aiDisplayUrl`はqueryとfragmentを除去したAI送信用・AI出力表示用URLとする。`localOpenUrl`はローカルVault内だけに保持し、既定ブラウザで開くためにだけ使う。`localOpenUrl`をClaude、AI出力、Markdown、ログへ渡さない。
- 複数Sourceから生成した場合は、どのURLがどのSourceに由来するか追跡できる。
- Source詳細から`localOpenUrl`をクリックして既定ブラウザで開ける。AI生成SourceとAI出力には`aiDisplayUrl`だけを表示する。
- URLを認識できない場合は推測・補完せず、「URL未検出」として扱う。
- 画像等からの抽出結果に誤認の可能性があるURLはAI提案状態とし、ユーザー確認前に確定URLとして扱わない。
- 抽出したURLの解析・query／fragment除去に失敗した場合は、そのURLを保存・表示・送信しない。画像自体は明示選択済みなら未マスクで送信でき、URL抽出やマスクを実行しないことを失敗にしない。

### 受入条件

- 明示選択した画面キャプチャは原画像をローカルに残したまま、会社契約Claude Codeへ未マスクで送信できる。
- AI出力には`aiDisplayUrl`だけが表示され、`localOpenUrl`のqueryやfragmentが現れない。
- アプリからURLを開く場合だけ`localOpenUrl`を使い、AI出力に表示するURLとの違いを詳細画面で明示する。
- 抽出URLの安全化に失敗した値は送信されず、画像の送信可否とは分離して確認できる。
- 派生Sourceでも出典URLと根拠Sourceへ逆引きできる。
- 存在しないURLをAIが生成しない。

## Analysis Sourceに紐づくAI対話

Analysis Sourceの詳細画面に、当該Sourceの内容を前提としてClaudeへ質問・補足指示を送る入力欄を設ける。一般のClaude画面を別途開かず、相談内容と結果をContextoryの来歴として残す。

### 必須要件

- Analysis Source詳細から、ユーザーが自由文で質問または修正指示を入力できる。
- Claudeへ渡す文脈は、対象Analysis Sourceとユーザーが明示的に選択した追加Source／Groupに限定する。
- ユーザー発言、AI回答、日時、使用モデル、根拠Source IDを追加専用の対話履歴として保存する。
- 各対話は`analysisSourceId`、基準Revision ID、送信時の`summaryPath`とSHA-256、Group展開後のSource ID一覧、使用したSource／派生物のIDとハッシュ、`stagedInputRefs`、モデル、prompt schema version、送信日時、結果状態を追加専用のローカル監査記録として固定保存する。
- 選択したGroupは送信時に個別Source IDへ展開し、そのsnapshotだけを対話の文脈とする。後日のGroup変更やRevision追加で過去対話の送信文脈を変えない。
- 対話、再分析、Group展開のいずれでも[ADR-008](../06-adr/ADR-008-local-media-preprocessing.md)を適用する。音声・動画の原本をClaudeへ渡さず、前処理済み文字起こし・代表フレームだけを使用する。未前処理または前処理失敗のメディアを含む場合はfail-closedで送信せず、対象を表示する。
- 初回解析、Revision再分析、AI対話、Group展開の全Claude実行で、Source Bundleをcwdまたは`--add-dir`として直接公開しない。選択済み送信対象だけを置く一時staging directoryをClaudeのcwdとし、明示選択した未マスク原画像、テキスト、PDF、安全化済みURL、固定済み文字起こし、代表フレームを配置できる。音声・動画原本、`localOpenUrl`、未選択ファイル、別Sourceの未選択原本、Vault全体は配置しない。
- staging metadataは`jobId`、`operationId`、`createdAt`を持ち、処理完了、失敗、タイムアウト、中断のいずれでも削除する。次回起動時には実行中Jobへ属さない残存stagingをすべて回収する。これは送信文脈の限定、Claudeの誤読と不要なトークン消費の防止を目的とする。
- 初回のRevision 1を含む各Analysis Revisionは、対象Revision ID、入力所有Source ID、`contentRole`、`inputType`、元の安全な相対path、staging論理path、staged bytesのSHA-256、MIME `contentType`、transformation種別／version、原ファイルSHA-256を`stagedInputRefs`へ固定する。非Source入力は保存先のAnalysis Sourceを所有者とし、別Revisionのsnapshotを使う場合は入力Revision IDも固定する。Revisionを作らないAI対話／Group展開は同じ配列を追加専用のinvocation auditへ固定する。
- 非Source入力または変更・削除され得る派生物は対象Revision Bundleへ不変input snapshotとして保存する。不変Source原本はSource ID、path、hashで参照し、Addition画像Source、明示追加context Source、`stagedInputRefs`参照先を削除保護する。走査不能、欠損、hash不一致はfail-closedとし、一時staging自体を正本にしない。
- AI回答を既存Source本文へ破壊的に上書きしない。
- 対話内容は構造化した履歴を正本とし、summaryに反映する場合は更新後のsummaryを新しいRevisionとして保存する。
- `summary.md`だけを正本にせず、構造化した対話履歴とRevision履歴から再生成できるようにする。
- 重要な回答は、新しい派生Sourceとして確定・保存できる。
- AI回答はユーザー確認前には`proposed`とし、確認、修正、却下を区別する。

### 受入条件

- Analysis Source詳細から質問し、対象Sourceを踏まえた回答が得られる。
- アプリ再起動後も質問と回答が残る。
- 対話により更新された`summary.md`と対応するRevisionを確認できる。
- 対話から新しいSourceを作成した場合、元Analysis Sourceへ逆引きできる。
- 別Sourceの対話履歴と混在しない。
- 過去の対話について、送信時のRevision summary、Group展開結果、各入力のハッシュ、モデル、prompt schema version、結果状態を再確認できる。
- 送信対象外のSource、音声・動画原本、`localOpenUrl`がstaging directoryに含まれず、処理後にstaging directoryが残らない。
- 正本Revisionが保存済みのJobを復旧した場合、Claudeを再実行せずmaterialized view、親Manifest、Processing Jobの同期だけで収束する。

## 初期実装順

1. Source統一・legacy移行: 既存Analysisへ正規`sourceId`とRevision 1の不変summary snapshotを割り当て、対応表、ハッシュ、Bundle走査による再構築を検証する。検証後にのみRevisionの新規書き込みを開始する。
2. Group: Group作成、名称変更、Source追加・除外、複数Group所属、Group–Source来歴を実装する。
3. Revision: Analysis Sourceへのテキスト・画像・URL追加、URL安全化、Revision履歴、summary再生成、原本／追加情報の閲覧を実装する。
4. AI対話: 選択済みSource／Groupだけの一時staging、監査履歴、Revisionを介した`summary.md`反映を実装する。
5. Task関連: TaskとSource／Groupの多対多関連、`task.json`のlink正本、逆引き索引を実装する。
6. 派生Source生成: Source単体・複数Source・Groupから派生Sourceを生成し、後続生成へ再利用する。
7. Review改善: Analysis一覧の具体的要約とJST日時表示、Actionの独立表示、参照確認付き削除を実装する。
8. 外部Output公開: 承認画面、Markdown添付、Adapter、公開監査と復旧を実装する。

## 外部Output公開基盤

外部公開は将来機能であり、まずSource単体・複数Source・GroupからMarkdownの派生Output Sourceを生成する。Markdownを共通成果物とし、Jira、Confluence、BacklogのAdapterが各サービスの要求形式へ変換する。詳細な安全境界は[ADR-013](../06-adr/ADR-013-approved-external-publication.md)に従う。

- 対象はJira Issue、Confluence Page、Backlog Issue／Wikiの将来対応とする。
- AI生成後は、ユーザーが本文、Project、Space、種別、添付を確認して明示承認した場合だけ公開する。AI生成、下書き保存、承認、送信は別状態とする。
- 承認時には`publicationId`、公開先、Project／Space、変換後payload、本文の不変snapshot、添付一覧と各SHA-256、承認日時を固定する。固定した項目のいずれかが変わった場合は承認を失効させ、再承認を必須とする。
- 公開本文に加え、元Markdownファイルを添付できるようにする。
- 送信前に`publicationId`、`attemptId`、`idempotencyKey`、`requestFingerprint`を永続化する。公開記録は対象Markdown派生Sourceに紐づけ、サービス、remote ID／Issue Key、URL、照合結果、outcome、送信日時、送信本文のsnapshot、添付、元Source ID、使用モデル、結果状態を保存する。
- API Tokenなどの資格情報はmacOS Keychainだけへ保存し、Keychain参照はアプリ内部だけで解決する。UserDefaults／plist、Local Vault、Markdown、ログ、Git、URL query、プロセス引数、環境変数、診断、クラッシュ情報、HTTPデバッグ出力へ保存・出力しない。
- 添付ごとにSHA-256、送信状態、remote attachment IDを保存する。作成成功後に添付だけ失敗した場合は、作成済みremote IDを保持し、失敗した添付だけを再実行できる。
- 作成結果が不明な場合は、新規作成を自動再試行しない。ユーザーがremote側を確認してから、既存remoteへの紐付けまたは明示的な再作成を選ぶ。

### 受入条件

- ユーザー承認なしに外部サービスへ作成、更新、添付を行わない。
- 承認済みの公開要求であっても、固定した宛先、payload、本文、添付のいずれかが変われば送信せず再承認を求める。
- 資格情報がUserDefaults／plist、Vault、Markdown、ログ、Git、URL query、プロセス引数、環境変数、診断、クラッシュ情報、HTTPデバッグ出力に現れず、アプリ内部のKeychain参照だけで解決される。
- 添付失敗時に新しいIssue／Pageを重複作成せず、既存remote IDに対して添付だけを再実行できる。
- 通信結果不明時に自動で新規作成を繰り返さない。
