# Source・Group・Task・Output要件

## Status

正規Sourceモデルとlegacy Analysis移行の基盤は実装済み。以下の一覧簡素化、解析完了判定の修正、保護ロック、Revision／Group UI、Action管理、派生Outputと外部公開は未実装である。Transcript訂正と用語辞書は[Transcript訂正・用語辞書要件](transcript-correction-terminology.md)、次期のTopic Source、手動Task、WBS、PM支援は[Topic Source・Task・WBS・PM支援要件](topic-source-task-wbs.md)に分離する。

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
- ユーザーはAI提案の要約を確認、修正、却下できる。ユーザー修正後の表示要約を、後続のAI再分析で黙って上書きしない。
- 分類は検索・絞り込みに使う属性であり、一覧の表示項目にはしない。
- 保存する時刻はUTCのISO 8601を正本とし、表示時にだけAsia/Tokyoへ変換する。既定の表示形式は分単位の`2026/08/14 10:30`とする。
- 表示要約が未生成の場合は推測せず、Source種別とJST日時による暫定表示を使い、`proposed`として確認対象にする。暫定表示はユーザー確定の要約ではない。

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

Analysis Source詳細の上部には「あなたの対応」を大きく表示し、Summary本文とは別にActionを表示する。ActionはAIの提案であり、確認前に完了・確定として扱わない。

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

## Source削除防止ロック

新規・既存を問わず、すべてのSourceは既定で保護ロックする。お気に入りは将来の絞り込み用であり、保護ロックとは独立した状態である。

- 保護ロック中は、当該Sourceの「Analysisと元データの削除」を実行できない。
- `protection`の欠落、不正値、未知値は必ず`locked`として読み、削除を許可しない。Source Manifestを保護状態の唯一の正本とし、SQLiteはBundle走査で再構築できる索引に限定する。
- 既存Sourceは保護状態を`locked`へatomicかつ冪等にbackfillする。backfillが未完了・失敗・検証不能なSourceは削除を許可しない。
- 削除の順序は「参照確認 → 削除操作中だけの一時ロック解除確認 → 削除確認 → macOSのゴミ箱へ移動」とする。
- 恒久的な手動ロック解除は、削除操作中の一時解除とは別操作・別状態とする。一時解除は削除成功、参照検出、キャンセル、失敗、異常終了のいずれでも自動的に`locked`へ戻す。
- ロック解除・削除は通常の閲覧、再分析、Task作成から視覚的に分離した危険操作領域に置く。
- 削除前にTask、Group、別Analysis／Revision、Addition画像Source、明示追加context Source、`stagedInputRefs`、Topicの親参照／Evidence Span／proposal、Taskのコメント・blocker・変更根拠、派生Source、外部公開記録からの参照整合性を`VaultMutationLock`内で確認する。参照中、参照走査不能、参照先欠損、path不正、hash不一致では削除を中止して判明した参照先を表示し、RevisionまたはAnalysisの削除でもcascade deleteしない。
- 条件を満たす削除は即時完全削除せず、Analysisと元データを含む対象BundleをmacOSのゴミ箱へ移動する。
- 既存Sourceへの導入時も既定を保護ロックとし、Manifestへのbackfill完了まではロック解除済みと推測しない。

### 受入条件

- ロック中のSourceは削除操作を実行できない。
- 削除にはロック解除確認と削除確認の両方が必要である。
- 参照中のSourceはゴミ箱へ移動せず、影響する参照を確認できる。
- backfill失敗、削除キャンセル、削除失敗、異常終了後に、Sourceが削除可能な解除状態として残らない。

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
7. Review改善: Analysis一覧の具体的要約とJST日時表示、Actionの独立表示、保護ロック付き削除を実装する。
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
