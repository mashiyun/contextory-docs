# ADR-012: Claude実行には最小一時staging directoryを使用する

- Status: Accepted
- Date: 2026-08-12
- Related: [ADR-008](ADR-008-local-media-preprocessing.md), [ADR-009](ADR-009-analysis-source-revisions.md)

## Context

会社契約のClaude Codeは、ユーザーが明示取得・取り込みした業務情報の許可済み処理境界である。一方、Source Bundle全体をClaudeの作業ディレクトリにすると、選択していない原本、音声・動画、`localOpenUrl`までClaudeが読み取れる可能性がある。これは文脈限定、誤読、不要なトークン消費の観点で不適切である。

個人Claudeには業務情報を送信しない。パスワード、APIキー、Cookie、秘密鍵、セッショントークン、認証QRコード等を含む画面は、ユーザーが取得・送信対象として選択しない運用とする。万能な認証情報検出、自動マスク、検出失敗を理由とする一律のfail-closedは実装しない。自動安全化は、画像・提供テキスト・ユーザー入力から抽出して保存・表示・送信するURLのquery／fragment除去に限定する。

## Decision

- 初回解析、Analysis Revision再分析、AI対話、Group展開を含む全Claude実行で、`jobId`、`operationId`、`createdAt`を持つSource Bundle外の再生成可能な一時staging directoryを作る。Local Vault、Source Bundle、別Source BundleをClaude Codeのcwdまたは`--add-dir`として直接公開せず、Claudeのcwdはstaging directoryに固定する。
- staging directoryには、その実行でユーザーが明示選択した原画像、テキスト、PDF、安全化済みURLだけを配置できる。原画像は会社契約Claude Codeへ未マスクで送信でき、Vision OCR、URL領域マスク、マスク失敗時の送信停止を必須としない。
- 選択した音声・動画Sourceはローカル前処理を必須とし、固定済みTranscript、代表フレーム等の前処理成果物だけを配置する。音声・動画原本は配置しない。
- `localOpenUrl`、未選択ファイル、選択していない別Sourceの原本、Source Bundle全体、Local Vault全体を配置しない。`provided-text`や画像等から抽出して保存・表示・送信するURLはquery／fragmentを除去した派生入力だけを配置する。
- Claude Codeにはstaging directoryの選択済みファイルだけをRead限定で渡す。個人Claudeを送信先にせず、画像内容をログ、診断、クラッシュ情報へ出さない。
- 各Analysis Revisionは、初回解析のRevision 1を含め、実際に配置した全fileを`stagedInputRefs`として不変保存する。各recordは対象Revision ID、入力所有Source ID、`contentRole`、`inputType`、元の安全な相対path、staging内の論理path、staged bytesのSHA-256、MIME `contentType`、transformation種別／version、原ファイルSHA-256を固定する。非Source入力は保存先のAnalysis Sourceを所有者とし、別Revisionのsnapshotを入力にした場合はその入力Revision IDも固定する。Revisionを作らないAI対話／Group展開は、同じ配列を対象Analysis Sourceと基準Revisionへ結び付けた追加専用のinvocation auditへ保存する。
- 非Source入力、または将来変更・削除され得る派生ファイルは、対象Analysis Revision Bundle内へ不変input snapshotとして保存し、そのpathとhashを`stagedInputRefs`から参照する。不変Source原本を直接参照する場合もSource ID、path、原ファイルhashを固定し、参照Sourceを削除保護対象にする。一時staging自体は正本にせず、`stagedInputRefs`と不変snapshotから当時のbyte集合を再現・検証できるようにする。
- Addition画像Source、明示追加context Source、`stagedInputRefs`が参照するSource／Revision／派生物は削除保護対象とする。参照走査不能、参照先欠損、path不正、hash不一致ではClaude実行と削除をfail-closedにし、RevisionまたはAnalysis削除時もcascade deleteしない。
- 処理完了、失敗、タイムアウト、中断のいずれでもstaging directoryを削除する。次回起動時には実行中Jobへ属さない残存stagingをすべて回収する。正本のAnalysis Revisionと`stagedInputRefs`が保存済みならClaudeを再実行せず、materialized view、親Manifest、Processing Jobの状態同期だけを行う。

## Consequences

### Positive

- 会社契約Claudeへ送信する業務情報を、ユーザーが選択した文脈へ限定できる。
- Group展開・AI対話・再分析の送信文脈を明示選択したSourceだけに限定できる。
- Claudeが無関係なファイルを誤読することと、不要なトークン消費を抑えられる。
- 破棄可能なstagingを正本にせず、各Revisionの`stagedInputRefs`から当時送信したbyte集合を監査・再現できる。

### Negative

- staging生成、整合性確認、例外時削除をProcessing Jobとして実装・テストする必要がある。
- staging削除失敗と、実行中Jobに属さない残存stagingを検出して安全に再試行する運用が必要になる。
- 不変input snapshot、参照hash検証、削除前の逆参照走査を、Revision writerとSource削除処理の双方へ実装する必要がある。
