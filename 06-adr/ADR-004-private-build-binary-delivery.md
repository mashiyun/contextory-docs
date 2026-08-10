# ADR-004 私用Macでビルドして会社Macへバイナリ提供

- Status: Accepted
- Date: 2026-08-10

## Context

Contextoryは開発者本人だけが利用する非公開アプリである。通常は開発とビルドを私用Macに集約し、会社Macでは提供されたアプリバイナリだけを実行したい。一方、直接配布がGatekeeper、署名、互換性などで安定しない場合は、会社Macでローカルビルドする選択肢を残す。

macOSではApp Store外から取得したアプリにGatekeeperが適用される。画面収録とマイクを使用するため、更新時にBundle IDや署名identityが不安定になると、起動や権限の再確認が発生する可能性がある。

## Decision

- 通常の開発、Archive、Releaseビルドは私用Macで行う。
- 優先経路では会社Macへビルド済み`.app`バイナリだけを提供する。
- 優先経路では会社Macへソースコード、Gitリポジトリ、Xcode、Swift toolchainを置かない。
- 直接配布が安定しない場合は、会社MacへXcodeを導入し、リポジトリをcloneしてローカルビルドする。
- Bundle IDを初期段階で固定し、更新版でも維持する。
- Release成果物へ実業務データ、秘密情報、ログ、開発用設定を同梱しない。
- 配布前にコード署名状態、entitlements、同梱内容、SHA-256を確認する。
- 現在の本人限定配布はad-hoc署名を使用し、SHA-256照合後に会社Macでquarantine属性を削除して起動する。
- Developer ID署名・notarizationは、会社Macのポリシー変更、配布対象拡大、または手動許可運用が負担になった場合に再評価する。

## Preferred delivery

Apple Developer Programを利用できる場合は、Developer ID Applicationで署名し、Hardened Runtimeを有効にしてnotarizeした直接配布用アプリを優先する。App Store公開やApp Reviewは行わない。

Developer IDを利用できない場合は、XcodeのCopy Appによる未署名またはローカル署名バイナリを限定的に検証する。この場合、会社MacでGatekeeperの手動許可が必要になる可能性があるため、初回起動と更新を実機PoCで確認する。

## Fallback build

直接配布が次の理由で安定しない場合は、会社Macでローカルビルドする。

- Gatekeeperで継続的に起動できない。
- 署名identityを安定して維持できない。
- 画面収録・マイク権限が更新ごとに不安定になる。
- 私用Macの成果物と会社MacのmacOS／CPUに互換性がない。

フォールバックでは、会社MacへXcodeを導入し、`contextory-app`をcloneして同じBundle IDとBuild Settingsでビルドする。実業務データはリポジトリ外に保存し、Gitへ追加しない。

## Consequences

### Positive

- 会社Macに開発環境やソースコードを置かずに済む。
- 私用Macだけで依存関係、ビルド、テスト、成果物を管理できる。
- 会社Macでは通常のアプリとして起動するだけの運用にできる。
- 直接配布が難しい場合も会社Macでローカルビルドして継続できる。

### Negative

- 別Macへの直接配布としてGatekeeper、コード署名、場合によってnotarizationへの対応が必要になる。
- 私用Macと会社MacのmacOSバージョンとCPU architectureに互換性が必要になる。
- 署名identityやBundle IDが変わると、初回起動や画面収録・マイク権限を再確認する可能性がある。
- Developer IDを使用する場合はApple Developer Programが必要になる。
- フォールバック時は会社MacへのXcode、ソースコード、依存関係導入が必要になる。

## Delivery checks

- Release configurationでArchiveする。
- Bundle IDとversion/build番号を確認する。
- `codesign`で署名とentitlementsを検証する。
- 配布物に実データ、ログ、秘密情報がないことを確認する。
- 配布ファイルのSHA-256を記録する。
- 会社MacでGatekeeperの初回起動を確認する。
- 画面収録とマイク権限を付与し、キャプチャ・録音・録画を確認する。
- Claude Codeのパスと認証状態を確認する。
- 更新版へ置き換えた後もVaultと権限が維持されるか確認する。
- 直接配布が安定しない場合は、会社Macのローカルビルドで同じ確認を行う。

## Follow-up

- 更新版への置き換えで権限とアプリデータが維持されるか、次回Releaseで確認する。
- quarantine属性削除が会社Macのポリシーで禁止された場合、Developer ID署名・notarizationまたは会社Macでのローカルビルドを再評価する。

## Phase 0 result

2026-08-11に優先経路の実機PoCを完了した。私用Macで作成したarm64のReleaseアプリをGitHub Releaseから会社Macへ提供し、SHA-256照合後にquarantine属性を削除して起動できた。画面収録・マイク権限、Claude Code実行ファイル検出、Application Support保存、終了後の再起動と状態維持も確認した。詳細は[Phase 0 配布・権限スパイク結果](../07-poc/phase-0-distribution-permissions-result.md)を参照する。
