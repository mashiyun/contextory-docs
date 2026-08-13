# データモデル

## Source統一モデル

一次入力、Analysis、Outputはすべて同じ`Source`として扱い、`input`、`analysis`、`output`は種別・属性で表現する。親Source、生成操作、実際に使用したSource IDを固定保存し、一次Sourceと派生Sourceのいずれも再利用可能にする。正規モデルは[ADR-009](../06-adr/ADR-009-analysis-source-revisions.md)であり、現行実装の独立Analysis BundleとContextは読み取り互換の表現として維持する。

詳細は[Source・Group・Task・Output要件](../02-requirements/source-group-task-output.md)および[ADR-009](../06-adr/ADR-009-analysis-source-revisions.md)を参照する。

### Project

顧客、製品、機能など、継続する業務文脈のまとまり。

### Task

Project内で完了条件を持つ対応単位。担当、期限、状態、待ち先を持てる。

Task Bundleの`task.json`には最低限、Task ID、タイトル、種別、状態、親Task ID、作成・更新日時、レビュー状態を保存する。`sourceLinks`と`groupLinks`をTask–Source／Task–Group関係の唯一の正本とし、各linkに相手ID、role、追加日時を保存する。同じSourceやGroupを複数Taskから参照でき、派生Task作成時も元Taskを上書きしない。

```json
{
  "sourceLinks": [{"sourceId": "01K2ABC...", "role": "evidence", "addedAt": "2026-08-12T09:30:00+09:00"}],
  "groupLinks": [{"groupId": "group_...", "role": "context", "addedAt": "2026-08-12T09:30:00+09:00"}]
}
```

### Source

判断の根拠となる不変の原本または記録。Capture、Audio Recording、Screen Recording、Transcript、Email、Analysis、Outputなどを含む。取得、保存、文字起こし、AI処理、レビューの状態を持つ。派生Sourceは親Sourceを上書きせず、`parentSourceIds`と生成来歴を持つ。

Sourceは固有IDを持つSource Bundleとして保存する。Source Bundleには次を含められる。

- `manifest.json`: 機械可読な永続メタデータ。
- `source.<ext>`: 変更しない原本。
- `provided-text.md`: BacklogやSlack等から一緒に取り込んだ変更しない原文。
- `user-notes.md`: 一次Sourceに対するAI解釈の訂正や画面外の前提としてユーザーが追加した補足。Analysis Sourceへの追加はRevisionに保存する。
- `transcript.md`: 音声または動画の文字起こし。
- `preview.jpg`: 一覧表示用の派生画像。
- `frames/`: 画面録画から抽出したキーフレーム。

派生Sourceは原本ファイルに加えて、生成本文と来歴を保持できる。Analysis Sourceの現在表示用summaryはRevisionの投影であり、Revision自体を上書きしない。

Source IDにはULIDまたは同等の衝突しにくい識別子を使用する。ファイルパスには顧客名、人物名、件名を含めない。

新規書き込みは`sourceId`を正規IDとして、`sources/`配下のSource Bundleへ保存する。既存`analyses/`配下のAnalysis Bundleは読み取り専用で残し、対応する`sourceId`とlegacy `analysisId`の対応を保存する。移行で既存の原本・Analysis Bundleを書き換えない。

### Source Manifest

最低限、次の項目を持つ。

```json
{
  "schemaVersion": 2,
  "sourceId": "01K2ABC...",
  "legacyAnalysisIds": [],
  "kind": "input",
  "type": "capture",
  "capturedAt": "2026-08-10T09:30:00+09:00",
  "sourceApplication": {
    "name": "Example App",
    "bundleIdentifier": "com.example.app"
  },
  "originals": [
    {
      "role": "primary",
      "path": "source.png",
      "sha256": "...",
      "contentType": "image/png",
      "bytes": 123456
    }
  ],
  "transcript": null,
  "parentSourceIds": [],
  "lineage": {
    "operation": "captured",
    "usedSourceIds": []
  },
  "providedText": "provided-text.md",
  "sanitizedProvidedText": "derived/sanitized/provided-text.md",
  "userNotes": "user-notes.md",
  "processingStatus": "needs_review",
  "reviewStatus": "proposed",
  "protection": {"state": "locked"},
  "favorite": false
}
```

実際のスキーマは実装前に確定する。この例へ秘密情報や不要な個人情報を追加しない。

画面または提供テキストから抽出した出典URLは、安全化URL、抽出元Source ID、抽出方法、確認状態を持つ構造化属性として保存する。AIは存在しないURLを推測せず、認証token等を含む可能性がある値を外部AI、ログ、Markdownへ渡さない。

既存Source Manifestの`id`、`summary`、`taskIds`、`groupIds`は後方互換のため読み込める。新規書き込みでは`sourceId`を使い、`taskIds`と`groupIds`を更新しない。新規のAnalysis SourceはRevision Bundleの最新投影として`summary.md`を持ち、構造化Revision履歴から再生成する。既存`analyses/`は新規結果の保存先とせず、対応するAnalysis Sourceの読み取り互換表現とする。

`protection`はSource Manifestの正本フィールドである。`state`が欠落・不正・未知値なら必ず`locked`として読む。SQLiteの保護状態はBundle走査から再構築できる索引であり、削除可否の正本ではない。`favorite`は既定`false`の独立属性であり、保護ロックや削除許可を示さない。

既存Bundleへのbackfillは、一時ファイルを検証してから同一filesystem上でManifestを原子的に置換する、冪等な操作とする。backfillが失敗・中断・未検証なら、そのBundleは削除不可とする。通常の恒久的な手動ロック解除と、削除操作だけに用いる一時解除は別の状態として記録する。一時解除は参照確認後に削除確認のためだけに有効にし、参照検出、キャンセル、失敗、異常終了、またはゴミ箱移動の成功時には自動的に`locked`へ戻す。

Phase 1では`sourceApplication`へ取得時の前面アプリ名とBundle IDだけを保存する。メール件名、文書名、ウィンドウタイトルは自動メタデータへ含めない。既存Manifestに`sourceApplication`がない場合も読み込める後方互換を維持する。

AIの観察、外部サービスから貼り付けた原文、ユーザーの補足を混在させない。再分析時は`user-notes.md`をユーザーによる優先情報として扱い、矛盾があれば黙って上書きせずQuestionまたはRiskへ残す。

GroupはSource Bundleを物理移動せずIDで関連付ける。Group–Source関係の唯一の正本はGroup Bundleの`group.json`にある`sourceLinks`であり、各linkはSource ID、role、追加日時を持つ。Source Manifest側の`groupIds`は新規更新しないlegacy cacheとする。手動で確定した関連とAIが提案した関連は別状態として扱う。

音声Sourceは`originals`に`system`と`microphone`のroleを持つ複数原本を保存できる。既存のschema version 1にある単一`original`は`primary`として読み替え、後方互換を維持する。詳細は[ADR-005](../06-adr/ADR-005-separate-system-and-microphone-audio.md)を参照する。

### Group

関連するSourceを案件、テーマ、顧客、機能などの文脈で集める入れ物。Groupは`group.json`の`sourceLinks`で複数Sourceを参照し、同じSourceは複数Groupへ所属できる。Groupへの追加は統合AnalysisやOutputの生成を起動しない。Groupから派生Sourceを生成するときは、Group IDだけでなく、その実行で使用した個別Source IDを来歴へ固定する。

```json
{
  "sourceLinks": [{"sourceId": "01K2ABC...", "role": "member", "addedAt": "2026-08-12T09:30:00+09:00"}]
}
```

既存の`Context`は移行期間の互換名とし、新規設計ではGroupを正規の関連単位とする。いずれもSource原本を移動、複製、上書きしない。

### Analysis

`kind: analysis`を持つ派生Source。Source単体、複数Source、またはGroupから生成し、解析目的、根拠Source ID、Group ID、Provider、モデル、生成日時、分類候補、レビュー状態を持つ。同じAnalysis Sourceへ情報を追加して再分析する場合は、新しいAnalysis Sourceを作らず、追加専用のRevisionを作る。目的または使用Sourceの組み合わせを変える独立分析は新しいAnalysis Sourceとして作成する。

AIとの対話は構造化した追加専用履歴を正本とする。summaryを更新する対話は対応Revisionへ参照を残す。重要な回答は新しい派生Sourceとして確定できる。

Analysis一覧表示用に、Analysis Sourceは`presentationTitle`を持てる。これは分類とは別の短い具体的タイトルであり、本文、生成元Revision ID、根拠Source ID、生成Provider／モデル、生成日時、確認状態、ユーザー修正の有無を保存する。ユーザー修正したタイトルは、明示的な再生成操作なしにAIが上書きしない。同一タイトルでは日時と短縮Source IDを補助表示し、未生成時は「種別・日時・短縮Source ID」を暫定表示する。この暫定表示はユーザー確定タイトルではない。

### Analysis Revision

Analysis Sourceに属する不変の再分析履歴。`revisionNumber`、作成日時、理由、追加情報（テキスト、画像、URL）、追加画像Source ID、追加情報種別、ユーザー指示、使用Provider／モデル、確認状態、使用Source ID、直前Revisionとの差分、summary本文の不変スナップショット、`summaryPath`、`summarySha256`を持つ。summary本文を持たないRevision、またはハッシュだけのRevisionは作らない。

最新の`summary.md`は最新Revisionの保存済みsnapshotから生成するmaterialized viewであり、正本ではない。すべてのRevisionを構造化して保存し、過去summary、差分、根拠Sourceを詳細画面で逆引きできるようにする。

RevisionはSummary本文から抽出した`actionItems`を持てる。各Actionは内容、種別（`self_action`、`delegated_request`、`waiting_response`）、期限候補、状態、根拠Source ID、抽出元Revision ID、確認状態、作成済みTask IDを持つ。Actionがないことも構造化して表せるため、UIは「現在必要な対応はありません」と表示できる。ActionからのTask作成は明示操作であり、Action自体はTaskの正本ではない。

### Source Addition

Analysis Revisionへ関連付ける追加情報。テキスト、画像、URLだけを受け付ける。画像は必ず独立した不変Sourceとして保存する。音声、動画、PDFはSource Additionの種別に含めず、新規一次Sourceとして取り込み、必要なら同じGroupへ関連付ける。

### Provenance URL

Sourceまたは派生Sourceが参照する出典URL。`aiDisplayUrl`、ローカルVault限定の`localOpenUrl`、抽出元Source ID、抽出方法（Vision OCR、提供テキスト、ユーザー入力）、抽出日時、確認状態を持つ。`aiDisplayUrl`はqueryとfragmentを除去し、AI送信・AI出力表示にだけ使う。`localOpenUrl`は既定ブラウザで開くためだけに使い、Claude、AI出力、Markdown、ログへ渡さない。

画像ではローカルVision OCRの検出結果に基づくURLマスク済み派生画像のSource ID、パス、SHA-256も保存する。OCR、URL解析、マスクの失敗時は自動送信を止め、`needs_review`とする。

### Analysis Conversation

Analysis Sourceに属する追加専用の対話履歴。発言、回答、日時、使用Provider／モデル、対象`analysisSourceId`、基準`revisionId`、送信時の`summaryPath`とSHA-256、ユーザーが明示選択した追加Source ID／Group ID、Group展開後のSource ID snapshot、実際に使用した個別Source／派生物のIDとハッシュ、prompt schema version、送信日時、結果状態を保存する。対話の文脈に未選択のSourceやGroupを暗黙に追加しない。

Groupの後日変更やRevision追加で過去の送信文脈を変えないため、対話ごとに送信時のsnapshotを必ず使う。対話、再分析、Group展開で選択したメディアはADR-008に従う前処理済み文字起こし・代表フレームだけを使い、未前処理または前処理失敗ならfail-closedで送信しない。

### Recording Reminder State

録音忘れ防止のローカル設定と一時状態。初期状態は無効とし、対象アプリごとの有効状態、前面継続閾値（既定20秒）、cooldown（既定60分）、`nextEligibleAt`、`suppressionReason`を持つ。`suppressionReason`は`today_suppressed`、`snoozed`、`cooldown`のいずれか、または未設定とする。

パネル表示時は`nextEligibleAt`を表示時刻から60分後、`suppressionReason`を`cooldown`へ設定する。`15分後に通知`は`nextEligibleAt`を操作時刻から15分後、`suppressionReason`を`snoozed`へ上書きする。「今回は通知しない」は操作時刻から60分後の`cooldown`、「今日は通知しない」は翌日のローカル日付開始時刻までの`today_suppressed`へ設定する。`nextEligibleAt`到達時は`suppressionReason`を自動解除し、`today_suppressed`も翌日のローカル日付開始時刻に自動解除する。判定優先順位は、録音中、`today_suppressed`、`snoozed`、`cooldown`、表示可能とする。対象アプリと閾値は設定で変更できる。前面アプリの判定に必要な最小限の情報だけを保存し、会話内容、ウィンドウタイトル、画面内容、外部送信履歴は保存しない。

### AI Staging

Claude実行ごとに作る一時ディレクトリ。`jobId`と`createdAt`を持ち、選択済みの通常画像、テキスト、安全化済みURL、文字起こし、代表フレームを配置する。URLを含む画像はマスク済み派生画像を使う。Source Bundle全体、音声・動画原本、`localOpenUrl`は配置しない。stagingのファイル一覧と入力ハッシュは監査記録へ残せるが、staging自体は永続化せず、処理完了、失敗、タイムアウト、中断後に削除する。次回起動時は、実行中Jobへ属さない残存stagingを削除する。

### Recording Input Preference and Monitoring

ローカル設定には、選択済みマイクの安定識別子、表示名、選択日時を保存する。各録音Sourceには、実際に使用した入力デバイス、マイク／システム音声ごとのレベル状態、無音・切断・入力停止の警告イベント、録音区間の開始・停止時刻を保存できる。突然の切断・権限喪失では、取得済みのマイク原本を確定し、マイク欠落の開始・終了時刻、原因、使用デバイスをRecording Source Manifestへ記録する。システム音声は可能な限り継続保存し、未接続デバイスを別デバイスへ無断で置換しない。設定は次回選択用であり、Sourceの実績メタデータを上書きしない。

### External Publication

Markdown派生Output Sourceの外部公開記録。公開先サービス、対象種別、Project／Space、remote ID／Issue Key、URL、送信日時、送信本文の不変snapshot、添付、元Source ID、使用モデル、承認状態、送信結果を持つ。

承認Recordは、`publicationId`、公開先、Project／Space、変換後payload、本文snapshot、添付一覧と各SHA-256、承認日時を不変に固定する。固定対象のいずれかが変われば、その承認は無効で再承認を要求する。送信前には`publicationId`、`attemptId`、`idempotencyKey`、`requestFingerprint`を永続化し、送信後はremote ID／Issue Key、URL、照合結果、outcomeを追記する。添付ごとにSHA-256、送信状態、remote attachment IDを保存し、失敗した添付だけを再送できる。作成結果不明は`outcome_unknown`として保存し、新規作成の自動再試行を禁止する。資格情報そのものはこの記録にもSource Bundleにも保存しない。

### Summary

Analysis SourceのRevisionから生成する閲覧用Markdown。生成モデル、生成日時、根拠Source、レビュー状態を持つ。最新`summary.md`は最新Revision snapshotからのmaterialized viewであり、Revision履歴の代替正本ではない。

### Grouping Proposal

Sourceを既存または新規のProject／Taskへ関連付けるAI候補。候補先、理由、確信度、レビュー状態を持つ。

ユーザーが候補を修正した場合、AI候補、修正後の関連、修正日時を分離して保持する。再分析でユーザー確認済みの関連を自動上書きしない。

### Task Classification

Analysisへ主分類、複数タグ、0〜1の確信度、分類理由、確認状態を保存する。初期候補には問い合わせ、質問、MTG整理、要件定義書作成、議事録作成、見積もり、調査、不具合、仕様確認、スケジュール調整、顧客返信、社内依頼、資料作成、その他を含める。AI結果は`proposed`であり、ユーザーが`confirmed`、`corrected`、`rejected`へ変更できる設計とする。

### Knowledge Item

Sourceから抽出したFact、Decision、Action、Question、Risk、Waiting。

### Review

AIの提案に対するユーザーのConfirmed、Corrected、Deferred、Rejected状態。

### Processing Job

文字起こし、AI要約、Markdown生成、グルーピングなどの処理単位。実行待ち、実行中、完了、失敗、再試行回数、最後のエラーを持つ。運用状態はSQLiteで管理する。

音声・動画では`pending_preprocessing`、`preprocessing_failed`、`analyzing`、`needs_review`を区別する。`derived/media/`へrole別文字起こし、動画代表フレーム、前処理索引を保存する。前処理成果物が存在しないメディアSourceからAnalysisを作成しない。

モデル比較時は`derived/media/base/`、`derived/media/small/`のように音声モデル単位で分離する。Analysis Sourceは`usedSourceIds`に加えて`speechModel`を持ち、同じSourceから生成したbase版とsmall版を別Analysis Sourceとして保持する。

自動実行と手動再実行を同じProcessing Jobモデルで扱い、実行契機を記録する。

## 基本関係

- Projectは複数Taskを持つ。
- Groupは複数Sourceを参照し、同じSourceは複数Groupへ所属できる。
- Taskは複数Sourceおよび複数Groupを参照できる。
- Sourceは複数Project・Taskに関連できる。
- Sourceは複数Groupから参照でき、Groupは派生Sourceの生成候補を集められる。
- Analysis Sourceは複数の不変Revisionと追加専用の対話履歴を持つ。
- Taskは複数Source・Analysis・親Taskを来歴として参照できる。
- 派生Output Sourceは入力Task、Analysis Source、親Sourceを来歴として参照し、根拠へ逆引きできる。既存の派生Output Taskは読み取り互換として扱う。
- Actionは抽出元Revisionと根拠Sourceを参照し、確認済みの場合だけ明示操作でTaskを作成できる。
- External PublicationはMarkdown派生Output Sourceを参照し、公開先remote IDと結果を保存する。remote IDは削除前の参照整合性確認対象である。
- Knowledge Itemは根拠となるSourceを参照する。
- SummaryとGrouping ProposalはAI生成物であり、確定情報とは分離する。
- AI生成内容とユーザー確認済み内容を別状態で保持する。
- ユーザーの修正履歴を残し、次回の関連判定へ利用できるようにする。

## SQLite Index

初期実装では次のテーブルを最小候補とする。

```text
sources
processing_jobs
groups
analyses                 # legacy Analysis Bundleの索引
analysis_revisions
revision_additions
provenance_urls
analysis_conversations
recording_reminder_state
recording_input_preferences
recording_input_events
source_protection_events
external_publications
publication_approvals
publication_attempts
publication_attachments
tasks
task_sources
task_groups
group_sources
legacy_analysis_source_map
ai_invocation_audits
```

Library／Review Interfaceを実装する段階で次を追加候補とする。

```text
review_items
knowledge_items
```

SQLiteへ画像、音声、動画をBLOBとして保存しない。SQLiteを唯一の永続正本にせず、破損・削除時にSource Bundleを走査して再構築できる索引として設計する。`task_sources`、`task_groups`、`group_sources`はそれぞれ`task.json`の`sourceLinks`／`groupLinks`、`group.json`の`sourceLinks`から再構築する索引であり、関係の正本ではない。

## 手動取り込み形式

- 画像: PNG、JPEG、HEIC、HEIF、WebP、GIF、TIFF。
- PDF: 非暗号化でPDFKitが読み込めるPDF。暗号化・破損PDFは取り込まずエラーを表示する。
- テキスト: TXT、Markdown、CSV、TSV、JSON、XML、YAML。
- 画面へ貼り付けたテキストはMarkdownの一次原本として保存できる。
- Manifestには原本ごとのContent Type、byte数、SHA-256を記録する。
- Officeファイルの直接取り込みは初期対象外とし、PDFまたは画面キャプチャを使用する。

## 整合性

- Source取得時は原本を保存し、ハッシュを計算してから`manifest.json`を確定する。
- `manifest.json`とMarkdownの更新は一時ファイルへ書き、同一ファイルシステム上で置き換える。
- SQLite更新に失敗してもSource Bundleを失わない。
- 再索引時はSource IDと`schemaVersion`を使用する。
- Project／Task／Group関連を変更しても原本ファイルを移動・複製しない。
- 関係の更新時は、Task–Source／Task–Groupなら対象`task.json`だけ、Group–Sourceなら対象`group.json`だけを更新する。Source Manifestへ逆方向のID配列を書き戻さない。
- 既存`analyses/`を走査してlegacy `analysisId`と正規`sourceId`の対応を作り、旧Bundleは読み取り可能なまま残す。対応、Revision 1の不変snapshot、Bundle走査によるSQLite再構築を確認するまで新規書き込みを正規モデルへ切り替えない。

詳細なSQLiteスキーマとManifest schemaは未確定である。保存方式の基本判断は[ADR-001](../06-adr/ADR-001-local-vault-storage.md)で確定する。
