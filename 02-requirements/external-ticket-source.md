# External Ticket Source要件

## Status

次期設計。Jira／BacklogからのExternal Ticket Source、Read Adapter、差分表示、添付選択取得、同期支援は未実装である。この仕様は外部チケットを変更・コメント・完了する機能を含まず、既存の外部公開機能とも別の境界である。API環境、認証方式、provider固有のID規則が未決でも、`unconfirmed` identityを持つ手動取り込みは先行実装できる。

## 正本と基本モデル

外部チケットは他のInput、Analysis、Output、Topic Sourceと同格のSourceとして、`kind: input`、`type: external_ticket_snapshot`で保存する。外部チケットを直接Taskへ変換しない。原則としてまずSourceを不変snapshotとして保存し、ユーザーが保存のみ、Group追加、新規Task作成、既存TaskへのSource Link、複数Task候補への分割のいずれかを選ぶ。

External Ticket Source writerはSource Manifest `schemaVersion: 4`と`externalTicket.schemaVersion: 1`を使用する。version 1〜3 readerを維持したversion 4 reader、SQLite migration、Bundle／snapshot検証、再索引を先に導入し、検証後だけwriterを有効化する。既存Sourceを一括backfillせず、未知fieldをround-tripで保持する。

- External Ticket SourceとContextory Taskは1対1に固定しない。1 Sourceから複数Task、複数Sourceから1 Task、Task化しない保存を許可する。
- Task–Source関係は既存`task.json` schema version 3の`sourceLinks`だけを正本とする。Taskにはcanonical `sourceId`をlinkし、legacy ID、remote ID、issue keyをSource IDの代わりに使わない。
- External Ticket Sourceは不変なのでTask linkの`revisionId`を必要としない。Source Manifestのlegacy `taskIds`を更新せず、SQLiteの`task_sources`は`task.json`からだけ再構築する。
- 将来Taskへremote IDと同期状態を保存しても、Taskの作業管理正本とExternal Ticket Sourceの取得snapshotを混同しない。外部状態の完了、再取得、AI解析はContextory Taskを自動完了・自動更新しない。
- Source、Group、Task、Analysis Revisionの既存の単一正本、`VaultMutationLock`、atomic replace、削除参照保護、DAG、AI確認境界を維持する。

## 取り込み経路

次の経路を同一のExternal Ticket Source schemaへ正規化する。

- 手動: URL、コピーしたタイトル・本文・コメント、取得範囲の申告。画面キャプチャ、画像、PDFは通常のInput Sourceまたは後続の選択添付Sourceとして原本を分離する。
- API: Jira／Backlog Read Adapterによる取得。

APIを利用できなくても、AIを使わず手動入力をSourceとして保存できる。AI解析を選ぶ場合も、その結果はAnalysis Source／Revisionとして保存し、Task候補、Task更新候補、Group候補は`proposed`に留める。ユーザー確認前にTask、Task link、Group linkを作成・変更しない。

手動入力でproviderが保証する不変issue IDを検証できない場合は、issue keyやURLが正しく見えても推測による一意化をしない。`remoteIdentity.state: unconfirmed`、`canonicalRemoteKey: null`として保存し、Adapterまたはユーザーが提示した検証可能な不変IDをユーザーが確認するまでremote key検索、再取得、差分の同一チケット判定に使わない。`unconfirmed` Source同士は内容hashが同じでも自動統合しない。同じ手動保存操作の再実行だけを`importOperationId`で冪等化する。

## Remote identityと重複判定

確定済みidentityは次のcanonical identity objectをUTF-8のRFC 8785 JSON Canonicalization Schemeで直列化し、そのbyte列のSHA-256小文字hexを`canonicalRemoteKey`とする。providerが不変instance IDを保証する場合はそれを使い、ない場合だけ正規化endpointをinstance identityとする。project／issue key、endpoint表示alias、表示URL、adapter version、remote `updatedAt`は変更可能なalias／取得属性であり、canonical identityへ含めない。

```json
{
  "canonicalRemoteKeyVersion": 1,
  "provider": "jira | backlog",
  "instanceIdentity": {"kind": "provider_instance_id | normalized_endpoint", "value": "..."},
  "projectStableId": null,
  "issueStableId": "20000"
}
```

- `endpointIdentity`は、ユーザーが確認したinstance base URLまたはprovider profileが返すinstance base URLを絶対URLとして共通normalizer version 1で正規化する。ticket URLから既知らしいpathを推測で除去しない。schemeを小文字化し、hostはUTS #46 non-transitional processingとSTD3規則でIDNA ASCII化して小文字化し、host末尾dotとdefault portを除去する。非default portは1〜65535の10進数として先頭zeroなしにする。pathの非ASCII文字をUTF-8 percent encodingへ変換し、percent encodingを検証してunreserved文字だけ復号、それ以外は大文字hexへ統一した後にdot segmentを除去し、rootまたはinstance base pathの不要な末尾slashを除去する。空pathと`/`は同一とするが、連続slashと異なるbase pathを統合しない。userinfo、query、fragment、資格情報、不正なUTF-8／percent encodingを拒否し、DNS解決結果をidentityに使わない。normalizer version 1は特定のUnicode／IDNA data versionを実装とtest vectorへ固定し、OS更新で暗黙に変更しない。IP literalはproviderが保証するinstance IDを使う場合だけendpoint aliasとして許可し、`normalized_endpoint` identityには使わない。Read Adapterとconfirmed identityはHTTPSだけを許可する。HTTP URLは手動Sourceの`unconfirmed`表示情報に限り、credentialを送らない。`instanceIdentity.kind: normalized_endpoint`の場合だけこの値をcanonical keyへ含める。
- canonical identity objectのkey集合は上記5件で固定し、不要な`projectStableId`も省略せず`null`にする。providerごとに、不変instance IDの有無、issue IDがinstance全体で一意かproject内だけで一意か、不変IDの文字列／case規則、必要な`projectStableId`を公式仕様に基づくidentity profileとして固定する。endpoint、issue key、project keyだけからこのprofileを推測しない。normalizerまたはidentity profileを互換性なく変更する場合は既存remote keyを新keyとして自動登録せず、旧keyとの明示的なalias migrationを別仕様で行う。
- Manifestにはcanonical remote key、表示用ticket URL、remote `updatedAt`、取得日時、取得方法（`manual`／`api`）、adapter versionを保存する。ticket URLは既存のURL安全化規則を適用し、資格情報を保存しない。
- `canonicalRemoteKey`単独は更新系列を束ねる値であり、複数snapshotに現れ得る。正規のsnapshot identityは`canonicalRemoteKey`、取得scope、remote version（取得できなければ`null`）、snapshot SHA-256の組である。同じ組に複数の異なる`sourceId`、または同じ`sourceId`に異なるidentity／payloadが見つかった場合はfail-closedとする。
- non-nullのremote versionが同じなのにhashが異なる場合はprovider応答または正規化の不整合としてfail-closedにし、新snapshotを作らない。同じremote versionとhashの再取り込みは既存Sourceへの冪等なno-opとする。versionが変わった場合、またはversionが`null`でhashが変わった場合だけ新Sourceを作る。remote `updatedAt`だけで同一性や順序を決めない。
- 新Sourceは、lock取得時点で同じremote key・scopeを持つ一意な系列tipを直前snapshotとして`parentSourceIds`へ保存する。`lineage.usedSourceIds`は`parentSourceIds`と同じ0件または1件にする。同じtipから異なるsnapshotを並行作成する分岐、循環、複数tip、直前snapshot不明は自動解決せずfail-closedとする。
- SQLiteのremote key索引は検索・最新view・重複検出のためだけに再構築し、正本にしない。「最新チケット」は系列の順序から投影するviewであり、別の重複正本を作らない。

## 不変snapshotと差分

External Ticket Sourceは、取得時のタイトル、説明、外部状態、担当者、期限、コメント、添付metadata、project／issueの表示key、remote version／`updatedAt`、identity確認状態、取得scope／coverageをallowlist済みcanonical JSON payload version 1へ正規化して不変保存する。HTTP header、cookie、認証応答、資格情報、取得scope外のfieldを含む不透明なraw API responseは保存しない。Source Manifestと取得Jobには、payloadのsnapshot path／SHA-256に加えて取得時刻、取得方法、adapter version、request fingerprintを固定するが、再取得ごとに変わるこれらの取得監査metadata、endpoint alias、表示用ticket URL、retry回数、pagination cursor、rate-limit状態、Job／operation IDはcanonical payloadとそのhashへ含めない。これにより同じremote version・scope・内容・表示keyの再取得は同じhashになる。manual取得では、ユーザーが提供した内容・範囲とidentity確認状態をpayloadへ固定する。

canonical snapshotはSource Bundleの`originals`に`role: primary`として登録し、`externalTicket.snapshotPath`／`snapshotSha256`をそのpath／hashと一致させる。`capturedAt`は`retrievedAt`と同じUTC時刻にする。snapshotの配列順、nullable field、Unicode、数値、remote日時のcanonical化規則を`snapshotSerializationVersion: 1`で固定し、scope／coverageもhash対象に含める。APIのコメントと添付metadataにはprovider profileで定めた不変remote item IDを必須とし、欠落または重複をfail-closedにする。これをID順でsortし、manual項目だけはユーザー確認済みordinalで順序を固定する。

snapshot version 1のtop-level key集合は`schemaVersion`、`remoteIdentity`、`remoteAliases`、`remoteVersion`、`remoteUpdatedAt`、`content`、`coverage`で固定する。`remoteAliases`のkey集合は`projectKey`と`issueKey`とし、remote keyの材料にはしないがrenameを不変snapshotの変更として残すためhash対象にする。`content`のkey集合は`title`、`description`、`status`、`assignee`、`dueDate`、`comments`、`attachments`とし、取得対象外または値なしのnullable fieldも省略せず`null`または空配列にする。`coverage.requested`／`completed`は`core_fields`、`comments`、`attachment_metadata`のUTF-8 byte昇順sorted unique配列とし、`core_fields`を必須にする。APIでは両配列の一致が確定条件であり、manualでは`mode: user_supplied`としてユーザーが申告・確認した範囲を両方へ保存する。API itemは`remoteItemId`を必須、manual itemは`remoteItemId: null`と一意な非負`ordinal`を必須にし、取得URLやlocal pathをpayloadへ入れない。

```json
{
  "schemaVersion": 1,
  "remoteIdentity": {"state": "unconfirmed", "canonicalRemoteKey": null},
  "remoteAliases": {"projectKey": "PROJ", "issueKey": "PROJ-123"},
  "remoteVersion": null,
  "remoteUpdatedAt": null,
  "content": {
    "title": "ログイン失敗を調査する",
    "description": "再現条件と期待結果",
    "status": {"remoteId": null, "name": "Open"},
    "assignee": null,
    "dueDate": null,
    "comments": [],
    "attachments": []
  },
  "coverage": {
    "mode": "user_supplied",
    "requested": ["core_fields"],
    "completed": ["core_fields"]
  }
}
```

`remoteIdentity`は`state`と`canonicalRemoteKey`、statusは`remoteId`と`name`、assigneeは`remoteId`と`displayName`の固定key集合を持つ。commentは`remoteItemId`、`ordinal`、`authorDisplayName`、`body`、`createdAt`、`updatedAt`、attachment metadataは`remoteItemId`、`ordinal`、`filename`、`contentType`、`sizeBytes`、`remoteVersion`、`remoteUpdatedAt`の固定key集合とし、nullable keyを省略しない。API itemは`ordinal: null`、manual itemは`remoteItemId: null`とする。IDはJSON numberへ変換せず文字列として保持する。Unicode文字列へ追加の正規化を行わず、日時はprovider profileで検証してUTC ISO 8601へ正規化し、日付だけの期限は`YYYY-MM-DD`のまま保持する。timezone不明、範囲外数値、非有限数、manual itemの重複ordinal、未定義keyはfail-closedとし、snapshot schemaを拡張する場合はpayload `schemaVersion`とManifestの`snapshotSerializationVersion`を同時に上げる。

- API取得は開始前に`requestedCoverage`を固定し、完了したfield／page集合を`completedCoverage`として検証する。コメントまたは添付metadataのpageが未完了など両者が一致しなければ、stagingと取得Jobだけを`partial`にしてSource Bundle、Manifest、remote key索引を確定しない。ユーザーが最初からコメントなし等の狭いscopeを選んだ場合は、その宣言scopeを完全取得できれば完全snapshotである。
- provider identity profileは、remote version、ETag、snapshot token、取得前後の再読込等、pagination全体が同じremote状態に属することを確認するconsistency strategyも固定する。取得中にanchorが変化した場合はJobを`inconsistent`としてSourceを確定しない。providerがpaginated範囲の一貫性を保証・検証できない場合はその範囲をAPIの`completed`へ含めず、単一応答で安全に固定できる狭いscopeだけを許可する。
- 部分API取得を自動で`user_supplied`へ格下げして保存しない。取得済み内容を手動Sourceとして残す場合は、別の`importOperationId`でユーザーに内容と範囲を提示し、明示確認を得た新規手動取り込みとして扱う。
- APIのpagination、rate limit、timeout、認可失敗、GET再試行、部分取得は取得Jobとして明示する。GET再試行は一時的な通信失敗とproviderのrate-limit指示に限定し、認可失敗や結果不明を成功扱いしない。秘密を含み得るpagination cursorは永続化せず、安全と確認できない場合は先頭から再取得する。
- 手動取り込みはAPI完全性を主張せず、`coverageMode: user_supplied`とユーザーが申告した範囲を保存する。手動の限定範囲をAPIの`partial`失敗と混同しない。
- 再取り込みでは直前snapshotとの差分を表示し、状態、期限、担当者、説明、コメント、添付の追加・変更を区別する。差分は新snapshotと親snapshotから再生成できる投影とし、別の正本にしない。
- 差分からTask更新候補を作成できるが、候補は根拠となる両snapshotと差分項目を参照する。ユーザー確定済みTask内容を外部再取得またはAIが上書きしない。

## Importの冪等性・atomicity・復旧

各手動保存／API取得は実行前に`importOperationId`、candidate `sourceId`、操作開始UTC時刻を確定して取得Jobへ永続化し、秘密を除いたprovider、endpoint identity、remote identity、requested coverage、snapshot serialization versionから`requestFingerprint`を作る。復旧時に同じoperation IDへ別Source IDや時刻を再採番しない。同じoperation ID・同じfingerprint・同じsnapshot hashは既存Job／Sourceへ収束し、同じoperation IDでfingerprintまたはhashが異なる場合はfail-closedとする。別operationが既存snapshotへ収束する場合、未使用のcandidate Source IDは再利用せず、Jobの`resolvedSourceId`だけを既存Sourceへ向ける。既存Sourceの`importOperationId`、親、Manifestは変更しない。`unconfirmed`手動取り込みはoperation ID単位でだけ収束させる。

API pageと添付候補は一時stagingへ保存し、coverage、canonical JSON、hash、identityを検証してから`VaultMutationLock`を取得する。lock内でBundleを再走査し、既存snapshot、系列tip、Source ID、operation IDを再検証する。完全な新Bundleをstagingで検証して同一filesystemへatomic renameした後にSQLiteをtransactionで投影する。SQLite失敗時もBundleを正本として残して再索引し、検証前の部分Bundleを`sourceId`の正本位置へ置かない。起動時は未完了staging／Jobを検証し、同じoperation IDへ復旧するか安全に破棄し、別Sourceを無断作成しない。

## コメント・添付

MVPはチケット全体を1つのsnapshotとして保存し、コメントごとのSource分割は後続とする。添付metadataはticket snapshotへ固定する。

- 添付本体を自動で全件取得しない。ユーザーが選択した添付だけを独立した`kind: input`、`type: external_ticket_attachment` Sourceとして保存する。
- 添付SourceはSource Manifest version 4と`externalTicketAttachment.schemaVersion: 1`を使い、Source作成を生じた最初の選択時の親External Ticket Sourceの`sourceId`を`parentSourceIds`へ保存してticketへ来歴を辿れるようにする。添付metadataのremote attachment IDと、保存した本体のSHA-256で識別し、同名だけで同一判定しない。後続snapshotから同一添付へ再度到達しても既存Sourceの親を追記・置換せず、取得Jobの`requestedFromSourceId`と`resolvedSourceId`でno-opの来歴を記録する。
- 添付は、選択時のticket snapshotに含まれるmetadataとremote attachment IDを照合してから取得する。同じticket remote key・attachment ID・本体hashの再取得は既存Sourceへ収束する。同じattachment IDで異なる本体を取得した場合は上書きせず、providerが保証するnon-null attachment versionの変更を検証できた場合だけ、`parentSourceIds`をticket snapshotとlock取得時点の一意な直前attachment tipのsorted unique unionにした新Sourceを作る。同じversionで異なるhash、version不明、分岐、複数tipは競合としてfail-closedにする。`lineage.usedSourceIds`はこのunionと一致させる。
- attachment取得失敗、ID不明、size上限超過、path／filename不正、Content-Type不整合、hash不一致はticket snapshotを変更せず、添付Sourceを確定しない。添付Sourceの取得は外部チケット更新ではない。redirect先へ元credentialを転送せず、providerが許可するHTTPS取得先だけを使う。

## Read Adapterと資格情報

Jira Cloud／Data Centerのどちらか、Backlogの対象環境と認証方式をAdapter実装前に確定する。Read Adapterは外部公開Adapterと別のinterface・Job・監査記録・credential namespaceを持ち、今回の取り込みでは外部チケットの変更、コメント、完了、添付の無断取得を行わない。共有できるのはcredentialやHTTP methodを持たない純粋なendpoint／payload正規化処理だけとし、Publication AdapterをRead経路から呼び出さない。

- 資格情報はRead専用の最小scopeでmacOS Keychainだけに保存し、provider、endpoint identity、account、用途`read`でPublication資格情報と別itemにする。アプリ内部だけで解決し、Vault、Manifest、UserDefaults、plist、URL、プロセス引数、環境変数、ログ、診断、クラッシュ情報へ保存・出力しない。OAuth token更新等の認証処理は許可するが、Adapter interfaceはticket取得と選択添付取得以外のremote mutationを公開しない。
- Adapterは取得範囲、pagination完了状態、rate limit、retry回数、timeout、adapter version、request fingerprintを記録する。fingerprintと監査記録は資格情報、URL query／fragment、未選択の本文を含めない。
- API仕様とrate-limit／pagination／retryの詳細は、対象環境確定後にAtlassianまたはNulabの公式資料で確認する。同期支援は必要性確認後の後続機能であり、Read Adapterの導入に含めない。

## Claude境界とUI

外部チケット原文はLocal Vaultへ保存する。Claudeへ送る場合は、ユーザーが明示選択したticket snapshot、添付由来Source、追加文脈だけをADR-012のstagingへ配置する。未選択の別チケット、添付、Groupを暗黙に追加せず、URLは既存安全化規則を適用する。AI結果はAnalysis Source／Revisionへ保存し、原snapshotを変更しない。

将来のタスク整理画面に「外部チケットを取り込む」を置く。常駐メニューバーのInput UIへ複雑な同期操作は追加しない。URL／手動入力とAPI取得を選べ、保存前にprovider、project、issue key、タイトル、取得範囲、重複候補、前回取得日時、差分を確認する。Task化または既存Task linkはユーザー確認後に実行する。

## 段階的実装と受入条件

実装順は、(1) Source Manifest version 4 reader、canonical snapshot、`unconfirmed`手動取り込みとoperation ID復旧、(2) provider identity profile、confirmed remote identity・重複／系列判定、(3) snapshot差分、(4) SourceからTask作成／既存Task Link、(5) Jira Read Adapter、(6) Backlog Read Adapter、(7) 添付選択取得、(8) 手動再取得、(9) 必要性確認後の同期支援とする。

- APIなし・AIなしでも外部チケットをSourceとして保存できる。段階1の受入境界は`unconfirmed` Sourceの保存までとし、Group追加は既存Group writerが利用可能になった時点、Task作成／既存Task linkは段階4で追加する。いずれも保存後の明示操作であり、Source確定transactionへ含めない。
- confirmed identityの同一snapshotはsnapshot identityへ、`unconfirmed`手動snapshotの再実行は同じoperation IDへ冪等に収束する。変更されたremote snapshotは親参照を持つ新Sourceになり、remote key、operation ID、Source IDの矛盾は上書きせずfail-closedとなる。
- endpoint normalizerはscheme／host case、IDNA、default port、空path／末尾slash、percent encoding後のdot segmentの同値test vectorを持ち、連続slashまたは異なるinstance base pathを統合せず、userinfo／query／fragment／不正encoding／HTTP confirmed endpointを拒否する。
- importの回帰試験は同一operation再実行、同一snapshot再取得、同じnon-null versionで異なるhash、pagination中のconsistency anchor変更、並行tip更新、commit各境界での中断、SQLite削除後の再構築を含む。
- 差分、Task更新候補、AI候補、外部状態完了はいずれもContextory Taskを自動変更しない。
- 部分API取得を完全snapshotとして確定せず、資格情報と未選択の外部データをVault／staging／ログへ出さない。
- Jira／Backlog環境、認証方式、API retryの具体値、添付redirect規則が未決でも、段階1はAPI／Keychain／confirmed identityを使わず実装できる。これらの未決事項は各Read Adapterまたは添付段階の開始ゲートとする。
