# データモデル

## Source統一モデル

一次入力、Analysis、Output、Topic Source、External Ticket Sourceはすべて同じ`Source`として扱い、`input`、`analysis`、`output`、`topic`は種別・属性で表現する。親Source、生成操作、実際に使用したSource IDを固定保存し、一次Sourceと派生Sourceのいずれも再利用可能にする。正規モデルは[ADR-009](../06-adr/ADR-009-analysis-source-revisions.md)、[ADR-015](../06-adr/ADR-015-topic-source-task-wbs.md)、[ADR-016](../06-adr/ADR-016-external-ticket-source.md)であり、現行実装の独立Analysis BundleとContextは読み取り互換の表現として維持する。

詳細は[Source・Group・Task・Output要件](../02-requirements/source-group-task-output.md)、[Topic Source・Task・WBS・PM支援要件](../02-requirements/topic-source-task-wbs.md)、[External Ticket Source要件](../02-requirements/external-ticket-source.md)、[ADR-009](../06-adr/ADR-009-analysis-source-revisions.md)を参照する。

## 時刻の保存と表示

- Manifest、Revision、`group.json`、`task.json`、SQLite、監査記録へ保存する時刻は、UTCのISO 8601（例: `2026-08-14T01:30:00Z`）を正本とする。
- 表示時にだけAsia/Tokyoへ変換する。一覧の表示形式は`2026/08/14 10:30`とする。
- 変換はUIの責務とし、保存済みの時刻をJSTで書き戻さない。
- 既存データにローカルオフセット付き時刻（`+09:00`）がある場合も同じ瞬時として読み込み、書き換えを目的とした一括変換は行わない。

### Project

顧客、製品、機能など、継続する業務文脈のまとまり。

### Task

Sourceとは別の、個人の作業管理正本。AIなしで作成・編集でき、担当、期限、状態、返答待ち、blockerを持てる。

Task Bundleの`task.json`には、Task ID、タイトル、説明、作業状態、優先度、担当者、予定／実績開始・終了日、進捗率、milestone、`parentTaskId`、`displayOrder`、`dependencyTaskIds`、確認状態、作成元、作成・更新日時を保存する。提案採用時はproposal ID、生成元Analysis Source／Revision ID、採用review event IDを持ち、手動作成時はこれらを`null`とする。`sourceLinks`と`groupLinks`をTask–Source／Task–Group関係の唯一の正本とし、各linkに相手ID、role、追加日時を保存する。同じSourceやGroupを複数Taskから参照でき、派生Task作成時も元Taskを上書きしない。新規writerは`schemaVersion: 3`とし、version 1／2／3 reader、SQLite migration、Bundle検証を先行させる。旧Bundleを一括変換せず、writerは未知のtop-level fieldとlink／event内の未知fieldを保持する。

```json
{
  "sourceLinks": [{"sourceId": "01K2ABC...", "role": "evidence", "addedAt": "2026-08-12T00:30:00Z"}],
  "groupLinks": [{"groupId": "group_...", "role": "context", "addedAt": "2026-08-12T00:30:00Z"}]
}
```

Taskの実作業状態と`coordinationState`（`none`／`waiting_response`／`blocked`）は別に保存する。コメントとblockerは不変な基底record、編集／削除／解消／再開は追加専用event、Task変更は追加専用eventとして同じ`task.json`へ保存する。各record／eventは一意なID、Task内で単調増加する`sequence`、`operationId`を持つ。現在値の変更とevent追加は`VaultMutationLock`内で1回のatomic replaceとして確定し、過去本文や解消済みblockerを消さない。ユーザー確定値とAI提案は別の確認状態で保存し、AIが確定値を上書きしない。将来の外部同期情報はremote ID、同期状態、最終同期日時、照合結果をTask側に保存する。

Taskのarchiveは`task.json`へ保存する状態とし、Task Bundle、`sourceLinks`、`groupLinks`、コメント、blocker、既存履歴を消さない。Source／Groupのlinkを自動変更しない。

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
  "capturedAt": "2026-08-10T00:30:00Z",
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

### 今回のschema切替

- Source Manifest schema version 3は、新規canonical Analysis Sourceの`generation.operationId`と`presentationSummary`、親Sourceの`analysisCompletions`を定義する。`presentationSummary`は任意、`generation.operationId`はversion 3で新規生成するAnalysisに必須とする。
- Source Manifest schema version 4はExternal Ticket Sourceの`externalTicket.schemaVersion: 1`と、External Ticket Attachmentの来歴recordを追加する。既存version 1〜3を読み、version 4 reader、SQLite migration、Bundle検証、再索引をwriterより先に提供する。既存Sourceを読込時または一括処理でversion 4へ変換しない。
- Analysis Revision schema version 2は、`operationId`、`rawTranscriptRefs`、`transcriptTransformSteps`、`dictionaryRevisionRefs`、訂正版snapshot、`requestFingerprint`を定義する。音声・動画の訂正再解析で新規作成するRevisionはversion 2を必須とする。
- Analysis Revision schema version 3は、初回解析とRevision再分析の`stagedInputRefs`と不変input snapshot参照を定義する。version 1／2／3 readerとSQLite再構築をwriterより先に提供し、切替後にClaudeが生成するRevisionはversion 3を必須とする。既存version 1／2 Revisionへ推測した`stagedInputRefs`をbackfillしない。Revisionを作らないAI対話／Group展開の追加専用invocation auditも同じrecord schemaを使う。
- 既存のManifest version 1／2とRevision version 1は読み取り互換で残す。欠落した`operationId`、Transcript来歴、`presentationSummary`を推測・合成・一括backfillせず、対応機能が必要な場合は手動レビュー対象にする。
- version 3 Source／version 2 Revisionの既存切替順は、(1) readerと旧フィールド解決規則、(2) SQLite migration、(3) Bundle検証、(4) 部分一意索引、(5) writer有効化の順を維持する。Revision version 3も同じreader先行順序で、`stagedInputRefs`用の再構築可能なSQLite表、Bundle内snapshotと参照hashの検証を追加してからwriterを有効化する。External Ticket Sourceのversion 4切替は別の後続migrationとし、いずれも旧Bundleを一括変換しない。
- 既存行のnullable `operation_id`はlegacy読み取り互換のためだけに許容する。新規writerはnullを拒否する。重複、schema不明、hash不一致、索引作成失敗のいずれかを検出した場合はwriter切替をfail-closedで停止し、旧データを変更しない。

親Source Manifestの`analysisCompletions`は追加専用recordの配列とし、各recordは`operationId`、`analysisSourceId`、`revisionId`、`summarySnapshotSha256`、`completedAt`を持つ。同じ`operationId`・同じ参照・同じhashの追加は冪等な成功とし、同じ`operationId`で参照またはhashが異なる場合は`analysis_integrity_failed`として上書きしない。別`operationId`のrecordは既存recordを削除せず追加する。この投影はAnalysisの成功境界ではなく、`AnalysisStore`だけがcanonical Analysis保存後に更新する。Processing Jobは投影先を`originatingSourceId`として1件だけ固定し、複数の`usedSourceIds`やGroupメンバーへ完了状態を配布しない。

### Group

関連するSourceを案件、テーマ、顧客、機能などの文脈で集める入れ物。Groupは`group.json`の`sourceLinks`で複数Sourceを参照し、同じSourceは複数Groupへ所属できる。Groupへの追加は統合AnalysisやOutputの生成を起動しない。Groupから派生Sourceを生成するときは、Group IDだけでなく、その実行で使用した個別Source IDを来歴へ固定する。Group archiveは`group.json`へ保存する状態とし、Group Bundleと`sourceLinks`を保持する。参照元Bundleのunlink、Source移動、cascade deleteは行わない。

```json
{
  "sourceLinks": [{"sourceId": "01K2ABC...", "role": "member", "addedAt": "2026-08-12T00:30:00Z"}]
}
```

既存の`Context`は移行期間の互換名とし、新規設計ではGroupを正規の関連単位とする。いずれもSource原本を移動、複製、上書きしない。

### Topic SourceとEvidence Span

`kind: topic`、`type: topic_excerpt`を持つ派生Source。会議、動画、長文の一部を指すが、親Sourceを分割・変更・複製しない。`parentSourceIds`、`lineage.operation: topic_excerpt_created`、作成`operationId`、`createdByType: user`、`creationOrigin: manual | accepted_ai_proposal`、nullableな`proposalId`、確認状態を保存する。手動作成ではproposal ID、provider／model、`promptSchemaVersion`を`null`として明示する。AI生成来歴は不変なproposal側に保存し、確定Sourceの作成者と混同しない。

`evidenceSpans`は複数可能な追加専用配列であり、各recordに`spanId`、Topic Revision内で一意な非負整数`displayOrder`、snapshot所有者の`parentSourceId`、原音所有者の`mediaSourceId`、nullableな親Revision ID、raw／revision snapshot種別、安全な相対snapshot path、snapshotと選択byte列のSHA-256、単一`mediaRole`、録音session相対の整数millisecond半開区間、Transcript UTF-8 byte半開区間を保存する。byte端点はUnicode scalar境界とする。原本では両Source IDを同じ録音Source、Revision snapshotでは`mediaSourceId`とroleを固定済み`rawTranscriptRefs`から選ぶ。Topic Source Bundleに親メディアやTranscriptを複製せず、再生はmedia Sourceのroleと時刻へ解決する。親のTranscript訂正・辞書更新・Revision追加では既存spanを移動せず、新しい内容を使うには新spanまたはTopic Revisionを追加する。`parentSourceIds`は全spanの両Source IDのsorted unique unionとし、そのedgeをSource来歴DAGの正本としてTask親子／依存graphから分離する。

AIの話題／Decision／Action候補はTopic Sourceではなく、生成元Analysis Revision内の構造化proposal recordとして保存する。採用・修正・却下・保留は同じAnalysis Sourceの追加専用review eventへ保存し、ユーザー採用後だけ`topic_excerpt`を確定する。Topic Sourceはproposal ID、生成元Analysis Source／Revision ID、採用review event IDを参照し、通常のSourceと同様にGroup、Task、Revision、AI相談、派生Outputの参照先になれる。

### External Ticket Source

Source Manifest `schemaVersion: 4`、`kind: input`、`type: external_ticket_snapshot`を持つ不変Source。Jira／Backlogのチケットを手動またはRead Adapterから同じschemaへ正規化し、Taskそのものにしない。`externalTicket` recordはschema version、import operation ID、identity確認状態、canonical remote key、provider、正規化済みendpoint identity、project／issueの不変IDと表示alias、表示用ticket URL、remote `updatedAt`／version、取得日時、取得方法、adapter version、snapshot serialization version、snapshot path／SHA-256、request fingerprint、取得scope／coverageを持つ。endpoint、表示用ticket URL、request metadata、fingerprintへquery／fragment、userinfo、資格情報を保存しない。ticket本文中のURLは不変なSource内容としてLocal Vaultに保持できるが、AI送信前に既存URL安全化を適用する。

```json
{
  "schemaVersion": 4,
  "sourceId": "01K2TICKET...",
  "kind": "input",
  "type": "external_ticket_snapshot",
  "capturedAt": "2026-08-15T00:31:00Z",
  "originals": [{"role": "primary", "path": "snapshots/external-ticket.json", "sha256": "...", "contentType": "application/json", "bytes": 1234}],
  "parentSourceIds": ["previous-snapshot-source-id"],
  "lineage": {"operation": "external_ticket_import", "usedSourceIds": ["previous-snapshot-source-id"]},
  "externalTicket": {
    "schemaVersion": 1,
    "importOperationId": "uuid",
    "remoteIdentity": {
      "state": "confirmed",
      "canonicalRemoteKey": "sha256-of-canonical-identity",
      "canonicalRemoteKeyVersion": 1,
      "provider": "jira",
      "instanceIdentity": {"kind": "provider_instance_id", "value": "instance-100"},
      "endpointIdentity": "https://jira.example.invalid/base",
      "endpointNormalizationVersion": 1,
      "projectStableId": null,
      "issueStableId": "20000",
      "projectKey": "PROJ",
      "issueKey": "PROJ-123"
    },
    "snapshotPath": "snapshots/external-ticket.json",
    "snapshotSha256": "...",
    "snapshotSerializationVersion": 1,
    "remoteUpdatedAt": "2026-08-15T00:30:00Z",
    "remoteVersion": "...",
    "retrievedAt": "2026-08-15T00:31:00Z",
    "acquisitionMethod": "api",
    "adapterVersion": "...",
    "requestFingerprint": "...",
    "coverage": {"mode": "api_complete", "requested": ["attachment_metadata", "comments", "core_fields"], "completed": ["attachment_metadata", "comments", "core_fields"]}
  },
  "protection": {"state": "locked"}
}
```

canonical remote keyはprovider、providerが保証する不変instance ID（なければ共通version 1規則で正規化したendpoint）、providerが不変と保証するissue ID、scope上必要な場合だけproject不変IDをUTF-8のRFC 8785 canonical JSON化したSHA-256とする。endpointと変更可能なproject／issue keyは表示aliasであり、不変instance IDを使うkeyへ含めない。confirmed identityで同じremote key・scope・remote version（nullable）・snapshot SHA-256を持つSourceは1件だけ許可し、再取り込みはそのSourceへ収束する。non-nullのremote versionが同じなのに異なるhash、同じoperation IDで異なるfingerprint／hash、Source ID重複、identity不整合はfail-closedとする。version変更、またはversionが`null`でのhash変更だけが新Sourceを作り、lock取得時点の一意な系列tipを親に固定する。remote key単独は更新系列を表すため一意制約にしないが、系列の分岐、複数tip、循環はfail-closedとする。manualで不変IDを検証できない場合は`state: unconfirmed`、`canonicalRemoteKey: null`とし、推測した統合・差分・再取得を行わない。

snapshot本文にはタイトル、説明、外部状態、担当者、期限、コメント、添付metadata、project／issueの表示key、remote version／`updatedAt`、identity確認状態、取得scope／coverageをallowlist済みcanonical JSONとして不変保存し、Sourceのprimary originalとpath／hashを一致させる。表示keyはremote identityへ含めないがsnapshot hashへ含める。取得時刻、取得方法、adapter version、endpoint alias、表示用ticket URL、request fingerprint、retry／pagination／rate-limit状態、Job／operation IDはManifest／Jobの取得監査metadataに固定し、payloadとそのhashへ含めない。不透明なraw HTTP応答や取得scope外fieldは保存しない。APIで要求範囲のpaginationが未完了ならstaging／Jobを`partial`としてSourceを確定しない。完全取得後に`VaultMutationLock`内で系列とoperation IDを再検証し、Bundleをatomic commitしてからSQLiteへtransaction投影する。差分と「最新」はこの系列から再生成するviewであり、Taskへの更新候補は両snapshotを根拠にしたproposalとして扱う。選択取得した添付本体は`type: external_ticket_attachment`の独立Sourceとし、選択時の親ticket Source ID、remote attachment ID、保存本体hashを固定する。

snapshot serialization version 1はUTF-8のRFC 8785 JSON Canonicalization Schemeを使う。APIのコメントと添付metadataはprovider profileで定めた不変remote IDを必須としてID順に固定し、欠落または重複はfail-closedにする。manual項目はユーザーが保存時に確認したordinalを保持する。取得scope、coverage、nullable fieldもcanonical JSONへ含め、表示順の偶然や辞書順でhashが変化しないようにする。

### 汎用Source Revision

既存のAnalysis Revision schemaを読み取り互換のまま維持し、revision対応のTopic Sourceへ追加専用Revision recordを導入する。recordはRevision ID、連番、作成日時、理由、作成者種別、`operationId`、provider／model、`promptSchemaVersion`、確認状態、使用Source ID、直前との差分、構造化snapshot pathとSHA-256を持つ。Topic Revisionのsnapshotには表示タイトル、説明、Evidence Span集合、ユーザー補足、候補の採否を含め、範囲修正・統合・再分割で過去spanを置換しない。手動操作のAI関連値は`null`とする。

汎用Revisionのreader、Bundle検証、SQLite索引、writerの順に導入する。不明schema、snapshot欠損、hash不一致、`operationId`衝突では対象writerをfail-closedで停止し、旧Bundleへの一括backfill、自動修復、自動削除を行わない。

### Analysis

`kind: analysis`を持つ派生Source。Source単体、複数Source、またはGroupから生成し、解析目的、根拠Source ID、Group ID、Provider、モデル、生成日時、分類候補、レビュー状態を持つ。同じAnalysis Sourceへ情報を追加して再分析する場合は、新しいAnalysis Sourceを作らず、追加専用のRevisionを作る。目的または使用Sourceの組み合わせを変える独立分析は新しいAnalysis Sourceとして作成する。

AIとの対話は構造化した追加専用履歴を正本とする。summaryを更新する対話は対応Revisionへ参照を残す。重要な回答は新しい派生Sourceとして確定できる。

新規のcanonical Analysis Sourceは`generation.operationId`を必須項目として持つ。値は対応するProcessing Jobで確定した`operationId`と同一とする。

Analysis一覧表示用に、Analysis Sourceは`presentationSummary`を持てる。これは分類とは別の、内容が分かる短い具体的な要約であり、本文、生成元Revision ID、根拠Source ID、生成Provider／モデル、生成日時、確認状態、ユーザー修正の有無を保存する。ユーザー修正した要約は、明示的な再生成操作なしにAIが上書きしない。保存値は長さにかかわらず保持し、一覧表示だけは実装言語の`Character`単位で20 Characterを超える値を先頭19 Character＋U+2026へ短縮する。

新規書き込みは`presentationSummary`だけを使う。legacyの`presentationTitle`は読み取り互換フィールドとして残し、新規に書き込まない。読込時のeffective summaryは、ユーザー確認済み`presentationSummary`、ユーザー確認済みlegacy `presentationTitle`、`presentationSummary`、legacy `presentationTitle`、Source種別とJST日時によるfallbackの順に解決する。両フィールドが存在してもlegacy値を削除せず、読込時のbackfillや既存Manifestの一括更新も行わない。新フィールドへの投影は次回の正規Revision書き込み時にappend-onlyで行う。schema version、旧version読込、新version書込、SQLite再索引のいずれでも同じ解決規則を使う。確認状態とユーザー修正履歴はフィールド移行後も保持する。legacyを含む全effective summaryは表示時だけ同じ20 Character短縮規則を適用する。

一覧に表示するのは、この具体的な要約とJSTへ変換した日時だけとする。Analysis表記、分類、処理状態、hash、Source ID、Revision番号、モデル名は一覧へ表示せず、詳細画面と診断画面で確認する。日時は既定で分単位とし、表示要約とJST分が一致する項目が複数ある場合だけ、その一致集合を秒まで、なお一致する場合だけ小数秒まで拡張する。比較と拡張判定はlocale非依存で決定的に行い、要約はUnicode正規化後のコードポイント列、日時はUTC正本の値で比較する。行の内部識別にはSource IDを使うが表示しない。要約が未生成の場合は、Source種別とJST日時による暫定表示を`proposed`として扱う。この暫定表示はユーザー確定の要約ではない。

### Analysis Revision

Analysis Sourceに属する不変の解析履歴。初回AnalysisもRevision 1として表し、`revisionNumber`、作成日時、理由、追加情報（テキスト、画像、URL）、追加画像Source ID、追加情報種別、ユーザー指示、使用Provider／モデル、確認状態、使用Source ID、直前Revisionとの差分、summary本文の不変スナップショット、`summaryPath`、`summarySha256`、`stagedInputRefs`を持つ。summary本文または送信入力監査を持たない新規Revision、ハッシュだけのRevisionは作らない。

最新の`summary.md`は最新Revisionの保存済みsnapshotから生成するmaterialized viewであり、正本ではない。すべてのRevisionを構造化して保存し、過去summary、差分、根拠Sourceを詳細画面で逆引きできるようにする。

音声・動画Sourceから生成したRevisionは、使用したTranscriptの来歴も保存する。

```json
{
  "operationId": "9f1c1c62-...",
  "rawTranscriptRefs": [
    {"sourceId": "01K2ABC...", "role": "system", "path": "derived/media/small/transcript-system.md", "sha256": "...", "language": "ja", "speechModel": "small"},
    {"sourceId": "01K2ABC...", "role": "microphone", "path": "derived/media/small/transcript-microphone.md", "sha256": "...", "language": "ja", "speechModel": "small"}
  ],
  "combinedTranscriptSnapshotPath": "transcript-combined.md",
  "combinedTranscriptSha256": "...",
  "mergeAlgorithmVersion": 1,
  "mergeInputOrder": ["system", "microphone"],
  "correctionSourceRefs": [{"sourceId": "01K2COR...", "sha256": "...", "appliedTo": "combined"}],
  "dictionaryRevisionRefs": [
    {"scope": "common", "scopeId": null, "revisionId": "01K2DIC...", "snapshotPath": "dictionaries/common/revisions/01K2DIC....json", "snapshotSha256": "..."},
    {"scope": "group", "scopeId": "group_...", "revisionId": "01K2DIG...", "snapshotPath": "dictionaries/groups/group_.../revisions/01K2DIG....json", "snapshotSha256": "..."}
  ],
  "correctedTranscriptSnapshotPath": "transcript-corrected.md",
  "correctedTranscriptSha256": "...",
  "stagedInputRefs": [
    {
      "revisionId": "revision-2",
      "inputOwnerSourceId": "01K2ABC...",
      "inputRevisionId": null,
      "contentRole": "selected_original",
      "inputType": "image",
      "contentType": "image/png",
      "sourceRelativePath": "source.png",
      "stagingLogicalPath": "inputs/addition-1.png",
      "stagedSha256": "...",
      "transformation": {"kind": "none", "version": 1},
      "originalSha256": "...",
      "snapshotPath": null
    }
  ],
  "requestFingerprint": "..."
}
```

`stagedInputRefs`は、当該RevisionでClaudeへ実際に渡した全fileの固定順配列である。各recordは対象`revisionId`、`inputOwnerSourceId`、nullableな入力`inputRevisionId`、`contentRole`、`inputType`、元の安全な相対`sourceRelativePath`、staging内の一意な論理path、staged bytesのSHA-256、MIME `contentType`、transformation種別／version、原ファイルSHA-256、nullableな不変`snapshotPath`を持つ。非Source入力は保存先のAnalysis Source IDを`inputOwnerSourceId`とし、元bytesを対象Revision Bundle内へsnapshotして`originalSha256`を固定し、`sourceRelativePath`にはその不変snapshotの相対pathを使う。fieldの欠落、論理path重複、Bundle外path、symlink、hash不一致はfail-closedとする。

非Source入力、または将来変更・削除され得るTranscript、代表フレーム、辞書excerpt等の派生fileは、対象Revision Bundleの`inputs/`へ不変input snapshotとして保存する。不変Source原本はSource ID、安全な相対path、Manifestの原ファイルSHA-256を固定して直接参照できるが、そのSourceを削除保護対象にする。stagingはこれらの参照またはsnapshotから再生成するcopyであり正本ではない。`stagedInputRefs`の順序、論理path、hashから当時Claudeへ渡したbyte集合を再現・検証できなければ、再送、再分析、削除を行わない。

`rawTranscriptRefs`は配列とし、各要素へ録音Source IDを含めて`system`と`microphone`の生Transcriptをrole別に不変で保持する。role別Transcriptを統合した場合は、統合結果の不変snapshotのパスとSHA-256、`mergeAlgorithmVersion`、統合時の入力順序をRevisionへ保存する。訂正はrole別Transcriptと統合後Transcriptのどちらへ適用したかを`appliedTo`として記録する。訂正版Transcriptから再生成したRevisionでも、role別の生Transcriptと過去Revisionは保持する。

`dictionaryRevisionRefs`は配列とし、各要素に`scope`、`scopeId`、`revisionId`、`snapshotPath`、`snapshotSha256`を保存する。MVPで同時適用できるのは共通辞書と、ユーザーが明示選択した1つのGroup辞書までとする。適用した配列をAnalysis Revision、補正履歴、Whisperヒント履歴へ同じ内容で固定する。

Revisionは実際に行った変換を`transcriptTransformSteps`の順序付き配列として保存する。MVPの`pipelineVersion: 1`は、role別辞書補正、role別ユーザー訂正、role統合、統合後辞書補正、統合後ユーザー訂正の順とし、不要なstepは省略できる。各stepは種別、algorithm version、入力Source ID・パス・SHA-256、出力snapshotのパスとSHA-256、使用した訂正Sourceまたは`dictionaryRevisionRefs`を持つ。前stepの出力SHA-256と次stepの入力SHA-256が一致しない場合はfail-closedとし、RevisionとAnalysisを生成しない。ユーザー訂正を同じ対象に対する辞書補正より後へ固定し、辞書変更でユーザー訂正を黙って上書きしない。

`operationId`は同一操作の再試行・復旧に使うidempotency keyである。同じ`operationId`の再実行は新規Revisionを作らず既存Revisionへ収束させる。別`operationId`による意図的な再生成は新しいRevisionとして許可する。

`requestFingerprint`は監査用に別途保存する入力の指紋であり、Revisionの同一性判定には使わない。次の項目をcanonical順で含める。

1. operation種別
2. purpose
3. provider
4. model
5. `promptSchemaVersion`
6. role別生Transcriptの参照とhash
7. 統合Transcriptのhash
8. 訂正Source IDとhash
9. `dictionaryRevisionRefs`、使用した`entryId`、staging用辞書excerptのhash
10. 訂正版snapshotのhash
11. pipeline version、統合algorithm version、辞書補正algorithm version、訂正algorithm version
12. 選択した追加Source IDとhash
13. `stagedInputRefs`の固定順、論理path、staged／original hash、transformation種別／version

`requestFingerprint`が一致するだけで、異なる`operationId`のoperationを自動統合しない。一致は監査上の同一入力を示すに留め、収束の判断は`operationId`で行う。

RevisionはSummary本文から抽出した`actionItems`を持てる。各Actionは内容、種別（`self_action`、`delegated_request`、`waiting_response`）、期限候補、状態、根拠Source ID、抽出元Revision ID、確認状態、作成済みTask IDを持つ。Actionがないことも構造化して表せるため、UIは「現在必要な対応はありません」と表示できる。ActionからのTask作成は明示操作であり、Action自体はTaskの正本ではない。

### Source Addition

Analysis Revisionへ関連付ける追加情報。テキスト、画像、URLだけを受け付ける。画像は必ず独立した不変Sourceとして保存し、Addition record、Revision、`stagedInputRefs`からSource IDとhashを参照して削除保護する。音声、動画、PDFはSource Additionの種別に含めず、新規一次Sourceとして取り込み、必要なら同じGroupへ関連付ける。明示追加context SourceもRevisionへIDを固定して削除保護する。

### Raw Transcript

要件は[Transcript訂正・用語辞書要件](../02-requirements/transcript-correction-terminology.md)、判断は[ADR-014](../06-adr/ADR-014-transcript-correction-terminology.md)に従う。

`whisper-cli`が生成した不変の文字起こし原本。[ADR-005](../06-adr/ADR-005-separate-system-and-microphone-audio.md)に従い、`system`と`microphone`を別の生Transcriptとして不変で保持する。音声モデルとroleごとに`derived/media/<speech-model>/`へ保存し、録音Source ID、role、パス、SHA-256、言語、Whisperモデル、生成日時を持つ。参照側は単数の`rawTranscript`ではなく`rawTranscriptRefs`配列で保持する。

role別Transcriptを時刻順へ統合した結果は、`combinedTranscriptSnapshotPath`、SHA-256、`mergeAlgorithmVersion`、入力順序とともに不変snapshotとして保存する。統合結果は生Transcriptを置き換えない。

用語ヒントとして渡した辞書は、Whisperヒント履歴へ`dictionaryRevisionRefs`と同じ形式（`scope`、`scopeId`、`revisionId`、`snapshotPath`、`snapshotSha256`）で記録する。ユーザー訂正、辞書補正、再解析のいずれでも生Transcriptを上書き・削除しない。

### Transcript Correction Source

ユーザーによるTranscript訂正を保持する不変Source。`kind: input`、`type: transcript_correction`とし、`parentSourceIds`へ対象の録音Sourceを保存する。

`type: transcript_correction`は来歴用のユーザー操作Sourceとして通常Inputの保存後自動解析Queueから除外する。訂正Source自体を新しいAnalysisの入力として自動送信せず、親Analysis Sourceを訂正版から再生成する明示的なProcessing Jobだけが参照する。

```json
{
  "kind": "input",
  "type": "transcript_correction",
  "parentSourceIds": ["01K2ABC..."],
  "target": {
    "appliedTo": "role",
    "transcriptRole": "microphone",
    "speechModel": "small",
    "transcriptPath": "derived/media/small/transcript-microphone.md",
    "transcriptSha256": "..."
  },
  "corrections": [
    {"before": "誤表記", "after": "正しい表記", "location": {"startUtf8ByteOffset": 120, "endUtf8ByteOffset": 132}, "reason": "固有名詞の誤変換"}
  ],
  "createdAt": "2026-08-14T01:30:00Z"
}
```

`target.appliedTo`は`role`または`combined`とし、role別Transcriptと統合後Transcriptのどちらへ訂正を適用したかを記録する。`role`の場合は対象roleとそのSHA-256、`combined`の場合は統合snapshotのパスとSHA-256を固定する。

MVPの訂正位置は、対象snapshotの先頭から数えたUTF-8 byteの半開区間`[startUtf8ByteOffset, endUtf8ByteOffset)`で表す。両端がUTF-8 scalar境界であること、範囲のbyte列が`before`のUTF-8 byte列と一致すること、同じ訂正Source内の範囲が重ならないことを検証する。適用はoffsetの大きい順に行い、検証不一致時はfail-closedで訂正SourceとRevisionを確定しない。別モデルによる再文字起こしへ位置を自動追従させず、新しいTranscript snapshotに対するユーザー確認と新しい訂正Sourceを要求する。

1回の訂正操作で複数箇所を訂正した場合も1つの訂正Sourceへ構造化して保存する。訂正Sourceは既存の訂正Sourceを上書きせず、追加で作成する。`user-notes.md`のユーザー補足とは別の記録として扱う。

### Terminology Dictionary

共通辞書とGroup／案件別辞書。Local Vault内に保持し、外部サービスへ同期しない。各項目は誤表記、正しい表記、スコープ（`common`またはGroup ID）、登録日時、根拠となる訂正Source ID、有効状態を持つ。

```json
{
  "dictionaryRevisionId": "01K2DIC...",
  "scope": "group",
  "scopeId": "group_...",
  "entries": [
    {
      "entryId": "01K2ENT...",
      "wrongForm": "誤表記",
      "correctForm": "正しい表記",
      "status": "active",
      "createdAt": "2026-08-14T01:30:00Z",
      "evidenceCorrectionSourceIds": ["01K2COR..."]
    }
  ]
}
```

辞書は追加専用のRevisionとして更新する。項目の追加、修正、削除はいずれも新しい辞書Revisionとして記録し、過去Revisionと、過去Revisionを参照する補正履歴を保持して過去Analysisの再現性を維持する。辞書項目の登録・変更・無効化は必ずユーザー確認を経て確定し、AIやアプリが自動登録しない。AIは登録候補までを提示する。

MVPで同時に適用できるのは、共通辞書と、ユーザーが明示選択した1つのGroup辞書までとする。Sourceが複数Groupへ属する場合もGroup辞書を自動選択せず、ユーザーが1つ選ぶ。適用した辞書は`dictionaryRevisionRefs`配列として、Analysis Revision、補正履歴、Whisperヒント履歴へ同じ内容で固定する。

決定的補正の走査規則は次のとおりとする。

- `normalizationAlgorithmVersion: 1`はUnicode scalar列の完全一致だけを対象とし、大文字小文字、全角半角、かな、送り仮名、Unicode正規化による暗黙の同一視を行わない。必要な表記揺れは別項目としてユーザー確認付きで登録する。
- 走査は左から右への単一passとし、置換結果へ再置換しない。
- 候補の順位は「UTF-8 byteでの開始位置が早い → 誤表記のUnicode scalar数が長い → 選択Group辞書を優先」で決める。
- 選択したGroup辞書は共通辞書より優先し、同じ誤表記に対する共通辞書の正しい表記を意図的に上書きできる。これは競合ではない。
- 同一scope内で同じ誤表記が複数の正しい表記を持つ場合だけ競合として扱い、自動適用しない。
- 上記の順位でも結果が一意に決まらない場合は適用せず、未適用として補正履歴へ記録する。
- 空文字、または他項目の正しい表記を誤表記として持つ不正な項目は自動補正へ使わない。
- 自動適用しなかった項目は、ユーザー確認対象として保持し、補正履歴から確認できる。

### Transcript Normalization Record

辞書による自動補正の履歴。補正前Transcriptのパスとハッシュ、補正後Transcriptのパスとハッシュ、適用した`dictionaryRevisionRefs`配列、適用した`entryId`と適用位置の一覧、適用対象（role別か統合後か）、`normalizationAlgorithmVersion`、適用日時、未適用として確認対象にした項目と理由（同一scope内の競合、順位不確定、不正項目）を持つ。補正前Transcriptを上書きせず、補正結果は別ファイルとして保存する。

### Provenance URL

Sourceまたは派生Sourceが参照する出典URL。`aiDisplayUrl`、ローカルVault限定の`localOpenUrl`、抽出元Source ID、抽出方法（画像、提供テキスト、ユーザー入力）、抽出日時、確認状態を持つ。`aiDisplayUrl`はqueryとfragmentを除去し、AI送信・AI出力表示にだけ使う。`localOpenUrl`は既定ブラウザで開くためだけに使い、Claude、AI出力、Markdown、ログへ渡さない。

画像からのURL抽出は任意であり、原画像送信の前提にしない。抽出する場合は推測・補完せず、query／fragment除去に成功した値だけを`aiDisplayUrl`として保存・表示・送信する。ユーザーが明示選択した原画像は未マスクで会社契約Claude Codeへ送信でき、Vision OCR、URL領域マスク、マスク失敗時の送信停止を必須としない。

### Analysis Conversation

Analysis Sourceに属する追加専用の対話履歴。発言、回答、日時、使用Provider／モデル、対象`analysisSourceId`、基準`revisionId`、送信時の`summaryPath`とSHA-256、ユーザーが明示選択した追加Source ID／Group ID、Group展開後のSource ID snapshot、実際に使用した個別Source／派生物のIDとハッシュ、`stagedInputRefs`、prompt schema version、送信日時、結果状態を保存する。対話の文脈に未選択のSourceやGroupを暗黙に追加しない。Revisionを作らないGroup展開も同じ配列を追加専用のAI invocation auditへ保存する。

Groupの後日変更やRevision追加で過去の送信文脈を変えないため、対話ごとに送信時のsnapshotを必ず使う。対話、再分析、Group展開で選択したメディアはADR-008に従う前処理済み文字起こし・代表フレームだけを使い、未前処理または前処理失敗ならfail-closedで送信しない。

### Recording Reminder State

録音忘れ防止のローカル設定と一時状態。初期状態は無効とし、対象アプリごとの有効状態、前面継続閾値（既定20秒）、cooldown（既定60分）、`nextEligibleAt`、`suppressionReason`を持つ。`suppressionReason`は`today_suppressed`、`snoozed`、`cooldown`のいずれか、または未設定とする。

パネル表示時は`nextEligibleAt`を表示時刻から60分後、`suppressionReason`を`cooldown`へ設定する。`15分後に通知`は`nextEligibleAt`を操作時刻から15分後、`suppressionReason`を`snoozed`へ上書きする。「今回は通知しない」は操作時刻から60分後の`cooldown`、「今日は通知しない」は翌日のローカル日付開始時刻までの`today_suppressed`へ設定する。`nextEligibleAt`到達時は`suppressionReason`を自動解除し、`today_suppressed`も翌日のローカル日付開始時刻に自動解除する。判定優先順位は、録音中、`today_suppressed`、`snoozed`、`cooldown`、表示可能とする。対象アプリと閾値は設定で変更できる。前面アプリの判定に必要な最小限の情報だけを保存し、会話内容、ウィンドウタイトル、画面内容、外部送信履歴は保存しない。

### AI Staging

初回解析、Revision再分析、AI対話、Group展開の全Claude実行で作る再生成可能な一時ディレクトリ。metadataに`jobId`、`operationId`、`createdAt`を持ち、Claudeのcwdに固定する。Local Vault、Source Bundle、別Source Bundleをcwdまたは`--add-dir`として直接公開しない。選択済みの未マスク原画像、テキスト、PDF、安全化済みURL、固定済み文字起こし、代表フレームを配置できる。Transcript訂正による再解析では、訂正版Transcriptと、適用済み辞書項目だけを抽出した派生excerptも配置できる。`dictionaryExcerptVersion: 1`は元辞書Revision参照、使用した`entryId`、正しい表記、scopeを固定key順で持つUTF-8 JSONとし、entryをscope（common、group）と`entryId`でsortする。excerpt正本をRevision監査領域へ不変保存して永続パスとSHA-256を記録し、stagingにはcopyだけを置く。辞書Revision snapshot全体はユーザーが明示選択しない限り配置しない。境界の正本は[ADR-012](../06-adr/ADR-012-minimal-claude-staging.md)とする。音声・動画原本、`localOpenUrl`、未選択file、別Sourceの未選択原本、Source Bundle全体、Vault全体は配置しない。staging自体は正本にせず、全fileをRevisionの`stagedInputRefs`、またはRevisionを作らないAI対話／Group展開の追加専用invocation auditへ固定する。処理完了、失敗、タイムアウト、中断後に削除し、次回起動時は実行中Jobへ属さない残存stagingをすべて回収する。正本Revisionが保存済みならClaudeを再実行せず、状態同期だけを行う。

### Recording Input Preference and Monitoring

ローカル設定には、選択済みマイクの安定識別子、表示名、選択日時を保存する。各録音Sourceには、実際に使用した入力デバイス、マイク／システム音声ごとのレベル状態、無音・切断・入力停止の警告イベント、録音区間の開始・停止時刻を保存できる。突然の切断・権限喪失では、取得済みのマイク原本を確定し、マイク欠落の開始・終了時刻、原因、使用デバイスをRecording Source Manifestへ記録する。システム音声は可能な限り継続保存し、未接続デバイスを別デバイスへ無断で置換しない。設定は次回選択用であり、Sourceの実績メタデータを上書きしない。

### External Publication

Markdown派生Output Sourceの外部公開記録。公開先サービス、対象種別、Project／Space、remote ID／Issue Key、URL、送信日時、送信本文の不変snapshot、添付、元Source ID、使用モデル、承認状態、送信結果を持つ。

承認Recordは、`publicationId`、公開先、Project／Space、変換後payload、本文snapshot、添付一覧と各SHA-256、承認日時を不変に固定する。固定対象のいずれかが変われば、その承認は無効で再承認を要求する。送信前には`publicationId`、`attemptId`、`idempotencyKey`、`requestFingerprint`を永続化し、送信後はremote ID／Issue Key、URL、照合結果、outcomeを追記する。添付ごとにSHA-256、送信状態、remote attachment IDを保存し、失敗した添付だけを再送できる。作成結果不明は`outcome_unknown`として保存し、新規作成の自動再試行を禁止する。資格情報そのものはこの記録にもSource Bundleにも保存しない。

### Summary

Analysis SourceのRevisionから生成する閲覧用Markdown。生成モデル、生成日時、根拠Source、レビュー状態を持つ。最新`summary.md`は最新Revision snapshotからのmaterialized viewであり、Revision履歴の代替正本ではない。

### Grouping Proposal

Sourceを既存または新規のProject／Taskへ関連付けるAI候補。生成元Analysis Revisionの構造化recordとして候補先、理由、確信度を持ち、確認状態の変更は同じAnalysis Sourceの追加専用review eventで表す。

ユーザーが候補を修正した場合、AI候補、修正後の関連、修正日時を分離して保持する。再分析でユーザー確認済みの関連を自動上書きしない。

### Task Classification

Analysisへ主分類、複数タグ、0〜1の確信度、分類理由、確認状態を保存する。初期候補には問い合わせ、質問、MTG整理、要件定義書作成、議事録作成、見積もり、調査、不具合、仕様確認、スケジュール調整、顧客返信、社内依頼、資料作成、その他を含める。AI結果は`proposed`であり、ユーザーが`confirmed`、`corrected`、`rejected`へ変更できる設計とする。

### Knowledge Item

Sourceから抽出したFact、Decision、Action、Question、Risk、Waiting。

### Review

AIの提案に対するユーザーのConfirmed、Corrected、Deferred、Rejected状態。

### Processing Job

文字起こし、AI要約、Markdown生成、グルーピングなどの処理単位。実行待ち、実行中、完了、失敗、再試行回数、最後のエラー、完了投影先を1件に固定する`originatingSourceId`を持つ。運用状態はSQLiteで管理する。

音声・動画では`pending_preprocessing`、`preprocessing_failed`、`analyzing`、`needs_review`を区別する。`derived/media/`へrole別文字起こし、動画代表フレーム、前処理索引を保存する。前処理成果物が存在しないメディアSourceからAnalysisを作成しない。

モデル比較時は`derived/media/base/`、`derived/media/small/`のように音声モデル単位で分離する。Analysis Sourceは`usedSourceIds`に加えて`speechModel`を持ち、同じSourceから生成したbase版とsmall版を別Analysis Sourceとして保持する。

自動実行と手動再実行を同じProcessing Jobモデルで扱い、実行契機を記録する。

解析Jobの成否は、canonical Analysis Source Manifest、最新Revision record、Revisionの不変Summary snapshot、保存済みSHA-256の検証が完了したかどうかで判定する。最新表示用`summary.md`はmaterialized viewであり成功境界へ含めず、欠損時はRevision snapshotから再生成する。

`operationId`はProcessing Job作成時にUUIDとして確定し、Claude実行前に永続化する。新規canonical Analysis Sourceでは必須とし、Analysisの`generation.operationId`へ同じ値を保存する。JobとAnalysisの`operationId`はSQLiteへ索引化し、それぞれ一意制約を持たせる。Bundle走査による再索引で重複を検出した場合はfail-closedとし、自動統合・自動削除を行わず新規書き込みを停止して手動レビュー対象とする。

Analysisの保存、Summaryの保存、親Source Manifestの完了更新は、実装上の`AnalysisStore`を唯一の所有者とする。`CaptureModel`はQueue、Processing Job、UI状態だけを担当し、親Manifestを更新しない。復旧したJobはClaude実行前に`operationId`でAnalysisを検索する。成功境界を満たすAnalysisがあれば再生成せず、親Manifestが同じ`operationId`の完了状態を示すか、`AnalysisStore`による更新が成功した場合だけJobを`completed`へ収束させる。親Manifestを更新できなければ`completion_sync_pending`とする。同じ`operationId`のAnalysis Sourceが存在するのに成功境界を満たさない場合は`analysis_integrity_failed`として停止し、Claudeを再実行しない。

保存前の失敗と保存後の状態同期失敗は別状態として扱う。保存前の失敗は`analysis_failed`または`retry_waiting`とし、再解析の対象とする。保存後の状態同期失敗は`completion_sync_pending`とし、Analysisを有効なものとして扱い、再解析ではなく状態の再同期だけを行う。

`completion_sync_pending`と`completion_sync_failed`はProcessing JobのSQLite運用状態を正本とし、Sourceの表示状態はJobとの対応から導出する。起動時の復旧対象に含め、canonical Analysis Source Manifest、最新Revision record、不変Summary snapshot、`operationId`、保存済みSHA-256を検証する。完成済みならAnalysisを再生成せず親ManifestとJobだけを再同期し、最新表示用`summary.md`だけの欠損はRevision snapshotから復元する。Analysis Sourceが存在しない場合だけ保存前失敗として再解析経路へ戻す。Analysis Sourceが存在するのに他の検証が失敗した場合は`analysis_integrity_failed`としてfail-closedに停止し、自動再解析・自動上書き・自動削除を行わない。これは有効なAnalysisの状態同期だけが失敗した`completion_sync_failed`とは別状態である。

再同期のためにProcessing Jobは`syncAttemptCount`、`lastSyncAttemptAt`、`nextRetryAt`、`lastSyncError`を永続項目として持つ。各試行の開始前にSQLite transactionで試行回数と時刻を永続化し、更新中の終了・異常終了も1回として数える。自動再同期は最大3回、間隔は5秒、30秒、5分とする。3回失敗した場合は`completion_sync_failed`へ遷移して自動処理を停止し、診断表示と手動再同期を提供する。手動再同期もAnalysisを再生成しない。`completion_sync_pending`と`completion_sync_failed`を「自動解析失敗」として表示しない。

## 基本関係

- Projectは複数Taskを持つ。
- Groupは複数Sourceを参照し、同じSourceは複数Groupへ所属できる。
- Taskは複数Sourceおよび複数Groupを参照できる。
- Sourceは複数Project・Taskに関連できる。
- Sourceは複数Groupから参照でき、Groupは派生Sourceの生成候補を集められる。
- Topic Sourceは親Source／親Revisionの複数Evidence Spanを参照し、親Sourceを上書き・複製しない。
- External Ticket Sourceは更新系列の親snapshotを参照し、同じSourceから0件以上のTaskへlinkできる。外部ticket IDはTask IDまたはSource IDにならない。
- Analysis Sourceは複数の不変Revisionと追加専用の対話履歴を持つ。
- Taskは複数Source・Analysis・親Taskを来歴として参照できる。
- Taskは親子・依存をTask IDで参照する。親子関係と依存関係はそれぞれ有向非巡回であり、WBSはGroupにリンクされたTaskの投影である。
- 派生Output Sourceは入力Task、Analysis Source、親Sourceを来歴として参照し、根拠へ逆引きできる。既存の派生Output Taskは読み取り互換として扱う。
- Actionは抽出元Revisionと根拠Sourceを参照し、確認済みの場合だけ明示操作でTaskを作成できる。
- External PublicationはMarkdown派生Output Sourceを参照し、公開先remote IDと結果を保存する。remote IDは削除前の参照整合性確認対象である。
- External Ticket Sourceは外部公開記録と別のInputであり、Read Adapterは外部ticketを変更しない。
- Transcript Correction Sourceは対象の録音Sourceと生Transcriptを参照し、生Transcriptを上書きしない。
- Analysis Revisionは使用した生Transcript、訂正Source、辞書Revisionを参照し、訂正版Transcript snapshotから逆引きできる。
- Terminology Dictionaryの項目は根拠となる訂正Sourceを参照し、辞書Revisionは補正履歴から参照される。
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
analysis_staged_inputs
revision_additions
provenance_urls
analysis_conversations
recording_reminder_state
recording_input_preferences
recording_input_events
transcript_corrections
terminology_revisions
terminology_entries
transcript_normalizations
source_protection_events
external_publications
publication_approvals
publication_attempts
publication_attachments
external_ticket_snapshots
external_ticket_remote_keys
external_ticket_import_jobs
external_ticket_attachment_refs
tasks
task_sources
task_groups
task_comments
task_comment_events
task_blockers
task_blocker_events
task_change_events
task_proposals
topic_evidence_spans
topic_proposals
group_sources
legacy_analysis_source_map
ai_invocation_audits
```

Library／Review Interfaceを実装する段階で次を追加候補とする。

```text
review_items
knowledge_items
```

SQLiteへ画像、音声、動画をBLOBとして保存しない。SQLiteを唯一の永続正本にせず、破損・削除時にSource Bundleを走査して再構築できる索引として設計する。`task_sources`、`task_groups`、`group_sources`はそれぞれ`task.json`の`sourceLinks`／`groupLinks`、`group.json`の`sourceLinks`から再構築する索引であり、関係の正本ではない。`analysis_staged_inputs`はRevisionと追加専用AI invocation auditの`stagedInputRefs`から再構築する削除保護・監査用索引であり、送信入力の正本ではない。`transcript_corrections`、`terminology_revisions`、`terminology_entries`、`transcript_normalizations`も、訂正Source Bundle、辞書Revisionファイル、補正履歴ファイルから再構築する索引であり、正本ではない。

`processing_jobs.operation_id`と`sources.operation_id`（新規`kind: analysis`／`topic`）には、legacyのnull行を除く部分一意制約を置く。同一操作・同一payloadの再試行は既存Sourceへ収束し、同じIDで異なるpayloadはfail-closedとする。新規writerはrequiredなoperation IDのnullを拒否する。Bundle走査による再索引で重複を検出した場合はfail-closedとし、重複の自動統合・自動削除を行わない。

`analysis_revisions.operation_id`にもlegacyのnull行を除く部分一意制約を置き、version 2／3 writerはnullを拒否する。再索引で同じ`operationId`を持つRevisionが複数見つかった場合はfail-closedとし、自動統合・自動削除を行わない。

`external_ticket_remote_keys`はconfirmed identityのremote key、取得scope、remote version、snapshot SHA-256、Source ID、親Source IDをBundleから再構築する索引とする。同じsnapshot identityに異なるSource ID、同じremote key・scope・non-null versionに異なるhash、同じSource IDに異なるsnapshot hashまたはremote identity、系列分岐／複数tipを検出した場合はfail-closedとする。remote keyのみの重複は更新系列として許容する。`external_ticket_import_jobs.import_operation_id`は一意とし、同じoperation ID・fingerprint・hashは収束、異なるpayloadはfail-closedとする。Jobはretry、pagination、rate limit、timeout、部分取得を表す運用状態であり、snapshot正本ではない。

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
- Claude実行前と削除前は`VaultMutationLock`内で、Addition画像Source、明示追加context Source、Revisionおよび追加専用AI invocation auditの`stagedInputRefs`が参照するSource／Revision／snapshotを走査し、所有Source ID、相対path、symlink不使用、staged／original SHA-256を検証する。走査不能、欠損、Bundle外path、hash不一致ではfail-closedとし、Analysis／Revision削除時も参照先をcascade deleteしない。
- 移行済みAnalysis Sourceのlegacy cleanupだけは、canonical Manifestが`schemaVersion: 3`かつ`kind: analysis`で、要求・canonical Bundle directory・Manifestの`sourceId`、canonical path／Manifest／hash、legacy `analysisId`対応が検証済みであり、`schemaVersion: 2`のlegacy AnalysisManifestを持つ所有下の子Bundleを承認済みlegacy rootの再帰走査からちょうど1件だけ解決できる場合に限る。legacy root自身、root直下の単独`analysis.json`、親directory、非Bundle directoryを候補にせず、canonical側の不一致、またはlegacy Bundle directory名／legacy AnalysisManifestの`sourceIds`以外の不一致はfail-closedとする。通常削除の厳格な一致検証は緩めない。
- このcleanupは`VaultMutationLock`内でprepare、一時ロック解除、commit直前にcanonical／legacyのidentity、schema、所有path、Manifest、hash、全参照を再検証し、2回のユーザー確認（一時解除、ゴミ箱移動）を要求する。canonical／legacy BundleのTrash移動は永続prepare／commit記録を持つ回復可能な論理transactionとし、片方だけの移動を成功としない。中断・失敗時は起動時に移動済みBundleを元の所有pathへ復元して`locked`へ戻し、復元不能・状態不明はfail-closedで隔離して自動削除・cascade deleteを行わない。
- Project／Task／Group関連を変更しても原本ファイルを移動・複製しない。
- 関係の更新時は、Task–Source／Task–Groupなら対象`task.json`だけ、Group–Sourceなら対象`group.json`だけを更新する。Source Manifestへ逆方向のID配列を書き戻さない。
- Topic Source確定時は`VaultMutationLock`内で親Source／親Revision／snapshotと選択byte列のhash、時刻／byte／scalar境界、追加後の全Source来歴DAGを検証する。参照先欠損、`spanId`／`displayOrder`／`parentSourceIds`の重複、union不一致、自己参照、推移的循環を拒否する。Topic／Task／Group／Revision／派生Source／公開監査から参照されるSource、または参照走査不能なSourceは削除不可とする。
- External Ticket Source確定時は`VaultMutationLock`内でschema version、import operation ID／fingerprint、canonical remote key、取得scope／coverage、snapshot primary originalとhash、remote version、一意な親snapshot系列、Source IDを再検証する。同一snapshot identityの重複、non-nullの同じversionに対する異なるhash、系列分岐・複数tip・循環、identity不整合、部分API取得、資格情報を含むpath／fingerprintは拒否する。外部ticket snapshotまたは選択添付を参照するSourceは削除不可とする。
- Taskの親子・依存を追加・変更する場合は`VaultMutationLock`内で追加後の各全graphを独立に検証し、参照先欠損、重複、自己参照、推移的循環を拒否する。コメント・blockerは基底recordを変更せず追加eventで編集、削除、解消、再開を表し、Task現在値とeventを同じatomic replaceで保存する。
- MVPのTask／Group削除はarchiveとし、Bundle、link、履歴を有効なまま保持する。archiveは状態の保存だけであり、参照元の自動unlink、Source移動、cascade deleteを行わない。
- 既存`analyses/`を走査してlegacy `analysisId`と正規`sourceId`の対応を作り、旧Bundleは読み取り可能なまま残す。対応、Revision 1の不変snapshot、Bundle走査によるSQLite再構築を確認するまで新規書き込みを正規モデルへ切り替えない。

列型・索引名を含む詳細なSQLite DDLは実装前に確定する。Source Manifest version 4とExternal Ticket record version 1の正本境界、保存方式の基本判断は本節、[ADR-016](../06-adr/ADR-016-external-ticket-source.md)、[ADR-001](../06-adr/ADR-001-local-vault-storage.md)に従う。
