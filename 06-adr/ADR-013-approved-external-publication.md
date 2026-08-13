# ADR-013: 外部Output公開は承認済みMarkdownとAdapterを介して行う

- Status: Accepted
- Date: 2026-08-13
- Related: [ADR-001](ADR-001-local-vault-storage.md), [ADR-009](ADR-009-analysis-source-revisions.md), [ADR-011](ADR-011-bundle-relationship-ownership.md)

## Context

ContextoryはSource単体、複数Source、Groupから派生Outputを生成する。Jira、Confluence、Backlogへ出す成果物は、サービスごとに入力形式、Project／Space、Issue種別、認証、添付APIが異なる。一方、AI生成結果をそのまま公開すると、誤った本文・宛先・添付を外部へ送るおそれがある。

作成要求の通信結果が不明な場合に新規作成を自動再試行すると、同じIssueやPageを重複作成する。作成成功後に添付だけが失敗した場合も、新規作成ではなく既存remoteへの再試行が必要である。

## Decision

- Source単体、複数Source、GroupからまずMarkdownの派生`kind: output` Sourceを生成する。Markdownを共通成果物とし、Jira、Confluence、BacklogのAdapterがサービス形式へ変換する。
- AI生成、下書き、ユーザー承認、送信を別状態として扱う。Adapterは、ユーザーが本文、Project／Space、種別、添付を確認・明示承認した公開要求だけを実行する。承認時に`publicationId`、公開先、Project／Space、変換後payload、本文snapshot、添付一覧と各SHA-256、承認日時を不変に固定し、そのいずれかが変われば再承認を必須とする。
- 公開記録の正本は対象Markdown派生Output Sourceに紐づく構造化記録とする。サービス、対象種別、remote ID／Issue Key、URL、送信日時、送信本文snapshot、添付、元Source ID、使用モデル、結果状態を保存する。送信前には`publicationId`、`attemptId`、`idempotencyKey`、`requestFingerprint`を永続化し、送信後にはremote ID、照合結果、outcomeを保存する。添付ごとにSHA-256、送信状態、remote attachment IDを保存する。SQLiteはこれらの索引であり、公開記録の正本ではない。
- API Token、OAuth refresh token、Cookieなどの資格情報はmacOS Keychainだけへ保存し、Keychain参照はアプリ内部だけで解決する。UserDefaults／plist、Local Vault、Markdown、ログ、Git、URL query、プロセス引数、環境変数、診断、クラッシュ情報、HTTPデバッグ出力には保存・出力しない。
- 作成成功後の添付失敗は、保存済みremote IDを使って添付だけを再実行する。新規Issue／Pageを作り直さない。
- 作成結果が不明な場合は`outcome_unknown`として記録し、新規作成を自動再試行しない。ユーザーがremote側を確認した後、既存remoteへの紐付けか、明示的な再作成を選ぶ。
- Jira Cloud／Data Center、Confluence Cloud／Data Center、Backlogの具体的なAdapterは、会社環境の種別と必須Custom Fieldを確定してから実装する。API仕様の確認にはAtlassianおよびNulabの公式資料だけを使う。

## Consequences

### Positive

- 公開前の本文・宛先・添付の確認を必須にできる。
- Markdownを共通成果物とするため、Source来歴を保ったままサービス固有の変換を分離できる。
- 添付失敗と作成結果不明を区別し、重複作成を防げる。
- 資格情報をVaultとGitから分離できる。

### Negative

- サービスごとのAdapter、認証、必須フィールド、添付、エラー分類を個別に実装・検証する必要がある。
- 公開要求、承認、remote ID、添付状態を保存・復旧するスキーマとUIが必要になる。
- 結果不明では自動復旧よりユーザー確認を優先するため、公開が保留になる場合がある。

## Follow-up

- 会社環境のJira／Confluence Cloud・Data Center種別、Backlogプロジェクト、必須Custom Fieldを確認する。
- 各Adapterの認証方式、Keychain access group、添付上限、remote側の重複確認手段を公式資料で検証する。
- 公開承認画面と、`attachment_pending`／`outcome_unknown`の復旧UXをPoCで確認する。
