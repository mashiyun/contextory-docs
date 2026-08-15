# Topic Source・Task・WBS・PM支援要件

## Status

次期設計。ここで定めるTopic Source、手動Task管理、WBS、ブロッカー、PM支援ビューは未実装である。既存のSource／Group正本、Revision、保護ロック、来歴を拡張する仕様であり、既存の実利用Vaultを移行・変更する作業は含まない。

## 正本と共通原則

- `Source`はInput、Analysis、Output、Topic Sourceを同格に扱う。一次原本と派生Sourceは、いずれもGroup、Task、WBS、Revision、AI相談、派生Outputの根拠または入力にできる。
- `Group`は文脈のまとまり、Group自身のmetadata、Group–Source membershipの正本、`Task`は個人の作業管理とTask–Source／Task–Group関係の正本、`Source`は入力・分析・成果物・根拠本文の正本である。GroupとTaskはSource本文を複製せず、相互参照はIDで行う。
- Task–Source／Task–Groupの正本は各Task Bundleの`task.json`、Group–Sourceの正本は各Group Bundleの`group.json`とする。SQLiteは再構築可能な索引である。
- すべてのAI提案は`proposed`であり、ユーザーの確認、修正、却下、保留と区別する。ユーザー確定済みの値をAIが黙って上書きしない。
- Topic／Task／WBS／blockerのAI提案本文と生成来歴の正本は、それを生成したAnalysis Revisionの構造化recordとし、採用・修正・却下・保留は同じAnalysis Sourceの追加専用review eventへ保存する。SQLiteのproposal表は再構築可能な索引に限る。採用で作るTopic SourceまたはTaskはユーザー操作として新しい`operationId`を持ち、proposal ID、生成元Analysis Source ID／Revision ID、採用review event IDを参照する。
- 永続化は`VaultMutationLock`の下で行い、Bundle正本の検証に失敗した場合はfail-closedとする。ID参照、hash、Revision、`operationId`、削除保護の既存規則を維持する。Bundle更新は一時ファイルへの完全書込み、検証、atomic replaceで行い、SQLite更新失敗時はBundleを正本として再構築する。

## 汎用Revision・追加情報

既存のAnalysis Revisionを壊さず、Topic Sourceを含むrevision対応Sourceへ同じ追加専用の原則を広げる。旧Analysis Revisionのreader／writer、summary snapshot、`operationId`の既存規則は維持し、一括backfillは行わない。

- 新しい汎用Revision recordは、Revision ID、連番、作成日時、理由、作成者種別、`operationId`、provider／model、prompt schema version、確認状態、使用Source ID、直前Revisionとの差分、構造化snapshotのpathとSHA-256を持つ。手動操作のAI関連値は`null`として明示する。
- Topic Sourceでは、表示タイトル、説明、Evidence Span集合、ユーザー補足、AI候補の採否をRevision snapshotへ固定する。範囲修正、統合、再分割は新Revisionで表し、過去spanを上書き・削除しない。
- 追加情報は既存のSource Addition規則に従う。テキスト、画像、URLは独立した来歴とhashを持って参照できる。音声、動画、PDFはRevisionへ直接追加せず、新規Sourceとして取り込み、必要ならGroupまたはTaskから関連付ける。
- generic Revisionのreader、Bundle検証、SQLite索引、writer有効化はこの順で導入する。不明schema、snapshot欠損、hash不一致、`operationId`衝突は対象writerをfail-closedで止め、既存Bundleを自動修復・削除しない。

## Topic Source

### モデルと来歴

Topic Sourceは、会議録音、動画、長文などの一部を指す派生Sourceである。`kind: topic`、`type: topic_excerpt`とし、親Sourceを変更、分割、移動、削除せずに保持する。Topic Source自体も通常のSourceとして、Group追加、Taskとの関連、Revision、AI相談、派生Outputの入力に使える。

- `parentSourceIds`には根拠となる親Source IDを保存し、`lineage.operation`を`topic_excerpt_created`とする。値は全Evidence Spanの`parentSourceId`と`mediaSourceId`のsorted unique unionに一致させ、その子→親参照をSource来歴edgeの正本とする。Source IDは複数spanで再利用できるが、`spanId`と`displayOrder`の重複、またはunionとの不一致を拒否する。`VaultMutationLock`内で追加後の全グラフを検査し、参照先欠損、自己参照、推移的循環のいずれかがあればBundleも索引も書き込まない。
- 確定Topic Sourceの作成記録には`operationId`、`createdByType: user`、`creationOrigin: manual | accepted_ai_proposal`、作成日時、確認状態を保存する。手動作成では`proposalId`、provider／model、`promptSchemaVersion`を`null`とする。AI提案のprovider／model／prompt schemaはSourceの作成者情報へ転記せず、不変なproposalを`proposalId`で参照する。
- AI候補と確定Topic Sourceは別物である。候補は根拠span、タイトル、種別（話題／Decision／Action）、確信度、理由、provider／model／prompt schemaを`proposed`として保存し、ユーザーが採用して初めてTopic Sourceを作成する。候補の採用後も候補の監査記録を失わない。同じ`operationId`と同じpayloadの再実行はno-op、同じIDで異なるpayloadはfail-closedとする。
- Topic Sourceの説明、表示タイトル、使用範囲を更新する場合は汎用Revision／変更履歴に追加し、作成時のEvidence Span snapshotを置換しない。過去Revisionから当時の根拠へ逆引きできるようにする。

### Evidence Span

1つのTopic Sourceは、離れた複数範囲をEvidence Spanとして参照できる。音声・動画本体やTranscript原本をTopic Source Bundleへ複製しない。

各spanは最低限、安定した`spanId`、Topic Revision内で一意な非負整数`displayOrder`、snapshot所有者の`parentSourceId`、原音所有者の`mediaSourceId`、nullableな親Revision ID、`snapshotKind: raw_transcript | revision_transcript`、親Source Bundleからの安全な相対`transcriptSnapshotPath`、`transcriptSnapshotSha256`、選択byte列の`selectedUtf8Sha256`、`mediaRole`（`system`／`microphone`／`primary`）、開始・終了時刻、Transcript範囲を持つ。原本Transcriptでは`parentSourceId`と`mediaSourceId`を同じ録音Source、`parentRevisionId: null`とする。訂正版では固定したAnalysis Revisionをsnapshot所有者として、Revisionの`rawTranscriptRefs`から該当する録音Sourceを`mediaSourceId`へ保存する。架空の「Revision相当ID」は作らず、pathは`latest`等の可変参照ではなく不変snapshotだけを指す。

Transcript範囲は対象snapshot上のUTF-8 byte半開区間`[startUtf8ByteOffset, endUtf8ByteOffset)`、時刻は録音session開始を0とする整数millisecondの半開区間`[startMilliseconds, endMilliseconds)`で保存する。開始は終了より小さく、byte境界はUnicode scalar境界と一致し、時刻は該当roleのmedia duration内でなければならない。1spanは1つの`mediaRole`だけを指し、role境界をまたぐ選択は複数spanへ分割する。複数spanの表示・結合順は一意な`displayOrder`で決定し、配列の偶然の読込順に依存しない。

- span確定時に、親Source・親Revision・snapshot path・snapshot全体と選択byte列のSHA-256・時刻／byte範囲・scalar境界を検証する。原本ならroleが`mediaSourceId`に存在すること、Revisionなら`mediaSourceId`とroleが固定済み`rawTranscriptRefs`へ含まれることも検証する。範囲超過、hash不一致、来歴不一致では確定しない。
- 再生は`mediaSourceId`が指す原音Sourceの該当role・時刻を開く。必要時だけ再生用の派生クリップを生成できるが、Evidence Spanと原音Sourceを正本として扱い、クリップを根拠の置換物にしない。
- Transcript訂正、辞書更新、親Sourceへの新Revisionがあっても、既存spanを新しい内容へ自動追従させない。過去のsnapshot hashと範囲から切り出し根拠を再現する。新しいTranscriptを使う場合は、ユーザー確認のうえ新spanまたは新Revisionを追加する。
- 子Topic Sourceの親参照またはEvidence Span、Topic／Task proposal、Task／Group link、Taskのコメント・blocker・変更根拠、Analysis／Revision、派生Source、公開監査のいずれかから参照されるSourceは削除不可とする。参照走査不能、DAG不正、hash不一致も削除を拒否する（fail-closed）。

### 話題分割UI

将来のSource詳細に「話題として切り出す」を設ける。Transcriptまたはタイムラインから範囲を手動選択し、タイトルを入力してTopic Sourceを作成できる。

- AIはユーザーの明示操作により話題、Decision、Actionの分割候補を提示できるが、候補からTopic Sourceを自動確定・自動生成しない。過剰な数のTopic Sourceを一会議から自動生成しないため、候補はレビュー対象としてまとめ、ユーザーが必要なものだけ採用する。
- ユーザーは候補を採用、却下、タイトル修正、範囲修正、統合、再分割できる。統合・再分割も既存Topic Sourceまたは候補を破壊的に置換せず、変更来歴を残す。
- 作成後に、Groupへ追加、Taskを新規作成、既存Taskへ紐付ける操作を提示する。いずれも任意であり、Topic作成だけでGroupやTaskを自動生成しない。

## Task基盤

TaskはSourceとは別の作業管理正本であり、AIがなくても手動で作成・編集できる。TaskとSource、TaskとGroupはいずれも多対多とする。

新規writerは`task.json`の`schemaVersion: 3`を使用する。version 1／2／3 reader、SQLiteのnullable列・新規テーブル、Bundle検証、新規writerの順で有効化し、旧Bundleを一括変換しない。readerは未知のtop-level fieldと、link／event内の未知fieldを保持し、既存Taskを更新するwriterもそれらを脱落させない。

`task.json`は最低限、Task ID、タイトル、説明、状態、優先度、担当者、予定開始・終了日、実績開始・終了日、進捗率、milestone、`sourceLinks`、`groupLinks`、確認状態、作成元、`parentTaskId`、`displayOrder`、`dependencyTaskIds`、コメント・blocker・変更event ledgerを持つ。作成元は`manual`、`topic_action_proposal`、`analysis_action_proposal`などを区別する。提案採用時はproposal ID、生成元Analysis Source ID／Revision ID、採用review event IDを保存し、手動作成時はこれらを`null`とする。

- 手動入力とAI提案を別フィールド・別確認状態で保存する。ユーザー確定値への更新はユーザー操作または明示的な採用操作だけが行える。
- Topic SourceまたはAnalysis RevisionのActionからTask候補を作成できる。候補は根拠Source、抽出元Revision、期限候補、提案内容を持つが、ユーザーが確認して「Taskとして登録」を実行するまでTask正本を作らない。
- Taskの編集は現在値に加え、変更者種別、変更日時、変更前後、根拠を追跡できる追加専用の変更記録を残す。現在値の更新とevent追加は同じ`task.json`の1回のatomic replaceで確定し、これはコメントの正本を置き換えない。
- Contextory内Taskを個人管理上の正本とする。将来Jira／Backlogへ公開または同期する場合は、remote ID、同期状態、最終同期日時、照合結果をTaskに保持し、外部側を正本にしない。

### コメント・返答待ち・ブロッカー

コメントはTask Bundle内の不変な基底recordとして、コメントID、日時、作成者種別、本文、根拠Source IDを保存する。編集・削除は既存本文を上書き・消去せず、対象コメントID、操作、日時、作成者、変更後本文または削除理由を持つ追加eventで表す。

Taskの実作業状態と協働状態は区別する。実作業が`in_progress`でも、相手または対象への返答待ちは`coordinationState: waiting_response`として明示できる。返答待ちを自分の作業中、完了、ブロッカー解除と同一視しない。

ブロッカーはTask Bundle内の不変な基底recordとして、blocker ID、理由、待っている相手または対象、登録日時、解消予定日、根拠Source、初期状態を持つ。解消、再開、理由訂正は既存recordを更新・削除せず、対象blocker ID、event種別、日時、作成者、変更内容を持つ追加eventで表し、現在状態と解消日時は基底recordとevent列から投影する。

コメント、blocker、Task変更の各record／eventは一意なID、Task内で単調増加する`sequence`、`operationId`を持つ。同じ`operationId`と同じpayloadの再実行はno-op、異なるpayloadはfail-closedとする。event追加とTask現在値の更新は`VaultMutationLock`内で1つの`task.json`へatomicに書き、SQLiteのコメント・blocker・event表はそこから再構築する。AIはコメント、Action、期限、依存関係から返答待ち・ブロッカー候補を提案できるが、コメント投稿、状態更新、blocker確定を自動実行しない。

## WBSと管理スケジュール

MVPではWBS専用Bundleや重複したWBS正本を作らない。Groupに紐づくTaskの`parentTaskId`、`displayOrder`、`dependencyTaskIds`からWBS表を投影する。WBS番号はGroup内の階層と表示順から生成する表示値であり、Task IDの代替・永続IDにしない。

- 手動でTask追加、子Task追加、インデント、アウトデント、並び替え、依存追加・解除をできる。親子関係を変えてもTask IDと来歴は保持する。
- 親子関係は参照先欠損、重複、自己参照、推移的循環を禁止する。依存関係も同じ検証を独立に行う。`VaultMutationLock`内で追加後の全Task graphを検証してから1つのTask Bundleをatomic更新し、検証失敗時はTaskも索引も変更しない。親子TaskをGroupのWBSとして表示するには、親子とも当該Groupへリンクされていなければならない。
- 前工程未完了、確認待ち、期限超過は直接ブロッカーとして区別して表示する。依存関係を辿って最初の未解消原因を根本ブロッカーとして別に示し、表示上のブロッカーを新しい正本として保存しない。
- Group化だけでTaskやWBSを自動生成しない。AIによるWBS分解は候補に限定し、ユーザー確認後にTask・親子・依存を登録する。
- WBS表には階層、予定日、状態、進捗、担当者、ブロッカーを表示する。簡易タイムラインは後続実装とする。工数、原価、リソース配分、複数ユーザー共同編集、権限管理は対象外とする。
- Taskの構造はMarkdown、CSV、Excelへ出力可能なID・日付・階層・順序・依存を保持する。出力機能自体は後続とする。

### archive・削除参照

- MVPのTask削除はhard deleteではなくarchiveとし、コメント、blocker、変更eventを保持する。親子／依存Task、proposal、派生Output、公開監査から参照中のTaskはhard deleteしない。
- Groupのarchiveは、`groupLinks`を持つTask、Group辞書、Groupを参照するAnalysis／Topic／Output／公開監査が存在する間は拒否する。Source削除とGroup archiveは`VaultMutationLock`内で直前に全正本を走査し、欠損、不明schema、破損、走査不能があればfail-closedとする。Task archiveは参照と履歴を有効なまま保持する非破壊操作とする。
- UI上のunlinkは参照元正本だけを更新する別操作であり、参照先Bundleの削除と同一操作にしない。複数Bundleをまたぐ自動cascade deleteは行わない。

## PM支援ビュー

次の機能は、既存のSource、Group、Task、Revision、コメント、blockerから生成するビューまたは構造化カードとしてロードマップへ置く。独自の重複正本を作らず、各カードから根拠Source／Task／Groupへ逆引きできるようにする。保存する場合も再生成可能なcacheに限り、Decision／RAID等をユーザーが確定する操作はSource作成またはTask変更として正本へ記録する。

- デイリーブリーフィング
- 返答待ち・期限超過
- Decision Log
- RAID
- 要件変更・影響分析
- ステータスレポート
- 顧客フィードバック整理
- 優先順位付け
- リリース準備確認

## 受入条件

- 固定Transcript snapshotを対象に、手動で、AIを使わずに複数の離れたspanを持つTopic SourceとTaskを作成・編集できる。非Transcript長文はoffset形式の確定後に同じ不変参照原則で追加する。
- Topic Sourceから親Source、親Revision、role別の固定snapshot、hash、範囲へ逆引きでき、親の更新後も根拠範囲が変わらない。
- AIのTopic／Action／WBS／blocker候補は、ユーザー確認までTopic Source、Task、依存、blockerの正本を作成・変更しない。
- Task–Source／Task–Group、Group–Source、Task親子／依存、Topic Sourceの来歴について、単一正本と再構築可能な索引の境界が一意である。
- 循環関係、削除対象への参照、コメント・blocker履歴の破壊的変更を拒否し、検証不能時はfail-closedとなる。
- WBS番号はTask IDを置換せず、Group化だけではTask・WBSを生成しない。PM支援ビューは重複正本を保持しない。
