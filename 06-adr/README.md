# ADR

長期的な影響を持つ技術・運用判断を記録します。

## 候補

- デスクトップ技術スタック。
- 原本・派生物・確認済み知識の分離。
- Claude Code連携境界。
- OCRと文字起こし方式。

ADRにはContext、Decision、Consequences、Statusを記載します。未決の内容はDecisionとして確定しません。

## Accepted

- [ADR-001 Local VaultのファイルとSQLite構成](ADR-001-local-vault-storage.md)
- [ADR-002 初期対象OSをmacOSに限定](ADR-002-macos-first.md)
- [ADR-003 macOSネイティブ・単一ユーザー構成](ADR-003-native-single-user-stack.md)
- [ADR-004 私用Macでビルドして会社Macへバイナリ提供](ADR-004-private-build-binary-delivery.md)
- [ADR-005 システム音声とマイク音声を別原本として保存](ADR-005-separate-system-and-microphone-audio.md)
- [ADR-006 Source・Context・Analysisを分離し派生結果を追加保存](ADR-006-append-only-context-analysis.md)
