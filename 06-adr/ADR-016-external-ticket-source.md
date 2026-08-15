# ADR-016: 外部チケットは不変External Ticket Sourceとして取り込む

- Status: Accepted
- Date: 2026-08-15
- Related: [ADR-009](ADR-009-analysis-source-revisions.md), [ADR-011](ADR-011-bundle-relationship-ownership.md), [ADR-012](ADR-012-minimal-claude-staging.md), [ADR-013](ADR-013-approved-external-publication.md), [ADR-015](ADR-015-topic-source-task-wbs.md)

## Context

Jira／BacklogのチケットにはTaskの根拠となる本文、状態、コメント、添付情報がある。しかし外部チケットをそのままContextory Taskとして扱うと、外部の更新、複数Taskへの展開、手動取得、AI候補とユーザー確定Taskが混在する。公開Adapterを取り込みに流用すると、読み取りと外部変更の安全境界も曖昧になる。

## Decision

- 外部チケットはSource Manifest version 4、`kind: input`、`type: external_ticket_snapshot`、`externalTicket.schemaVersion: 1`の不変Sourceとして保存する。version 1〜4 reader、SQLite migration、Bundle検証、再索引をwriterより先に導入し、旧Sourceを一括backfillしない。Taskへの直接変換や1対1対応を設けず、Task化・既存Task link・Group追加はユーザー確認後の別操作とする。
- canonical remote keyはprovider、providerが保証する不変instance ID（なければ共通version 1規則で正規化したinstance endpoint）、providerが不変と保証するissue ID、およびissue IDのscopeに必要な場合だけproject不変IDをRFC 8785 canonical JSON化したSHA-256とする。instance endpointは確認済みbase URLから作り、ticket URLのpathを推測で切り詰めない。不変instance IDを使う場合、変更可能なendpoint、project／issue key、表示URL、adapter version、remote日時を含めない。query、fragment、userinfo、資格情報を拒否する。provider固有IDを検証できない手動取り込みは`unconfirmed`のまま保存できるが、remote key検索・差分系列・自動統合に使わない。
- remote keyは更新系列を束ねる値であり、snapshot identityはremote key、取得scope、remote version、snapshot hashの組とする。non-nullの同じversionで異なるhashは新snapshotではなく競合としてfail-closedにし、versionが`null`ならhash変化を更新とする。同一snapshot identityと、同じ`importOperationId`・fingerprint・hashの再取り込みは冪等に既存Sourceへ収束する。同じoperation IDでpayloadが異なる場合もfail-closedとする。
- 更新時は既存Sourceを上書きせず、lock取得時点の一意な系列tipを`parentSourceIds`と`lineage.usedSourceIds`で参照する新snapshot Sourceを作る。並行作成による分岐、複数tip、循環、矛盾するremote key／Source ID、検証不能はfail-closedとする。
- snapshotには取得時のticket本文、コメント、添付metadata、project／issueの表示key、remote version／`updatedAt`、identity確認状態、取得scope／coverageだけをallowlist済みcanonical JSON payloadとして不変保存し、Sourceのprimary originalとpath／hashを一致させる。表示keyはremote identityへ含めないが、renameを履歴化するためpayload hashへ含める。取得時刻、方法、adapter version、endpoint alias、表示用ticket URL、request fingerprint、retry／pagination／rate-limit状態、Job／operation IDはManifest／Jobへ固定するが、再取得で変化するためpayload hashへ含めない。不透明なraw HTTP応答、header、cookie、認証情報、scope外fieldは保存しない。差分と最新ticketは系列から再生成するviewであり、別正本にしない。
- API取得はpageを一時stagingへ置き、requested coverageの完了、provider profileで定めたconsistency anchor、identity、canonical JSON、hashを検証してから`VaultMutationLock`内でBundleをatomic commitし、SQLiteへtransaction投影する。pagination未完了、または取得中にremote状態が変化したAPI結果はSource Bundleとして保存しない。SQLite失敗時はBundleを正本として再索引し、起動時は同じoperation IDのJobへ復旧する。
- Read Adapterは公開Adapterからinterface、Job、監査記録、Keychain credential namespaceを分離し、Read専用の最小scopeだけを使う。Read Adapterは外部チケットを変更、コメント、完了せず、Publication Adapterを呼び出さない。共有はcredentialとHTTP methodを持たない純粋な正規化処理だけに限定する。
- 添付本体はユーザー選択時だけ独立Sourceとして取得し、選択時の親External Ticket Source、remote attachment ID、保存本体hashを固定する。同じattachment IDの異なる内容を無検証で上書きしない。
- Claudeへ渡すのはユーザー選択済みsnapshot、添付由来Source、追加文脈だけとし、ADR-012のstagingと既存URL安全化規則に従う。AI結果はAnalysis Source／Revisionへ保存する。

## Consequences

### Positive

- APIなしの手動取り込みでも、外部チケットを根拠として安全にTaskへつなげられる。
- 外部の更新履歴とContextory Taskのユーザー確定内容を分離できる。
- 取り込みと公開のAdapter、資格情報、外部変更権限を分離できる。

### Negative

- remote identity正規化、snapshot系列、差分、部分取得、重複検出の実装と検証が必要になる。
- Jira／Backlog環境と認証方式を確定するまでRead Adapterを実装できない。
- confirmed remote identityと差分系列はproviderの不変ID規則を確定するまで実装できないが、`unconfirmed`手動Sourceの保存は先行できる。

## Relationship to existing ADRs

ADR-009の統一Source／Revision、ADR-011のTask link正本、ADR-012のClaude staging、ADR-013の公開Adapter、ADR-015のTask確認境界を変更しない。外部公開は外向きの承認済みOutput、外部取り込みは読み取り専用のInput Sourceとして分離する。
