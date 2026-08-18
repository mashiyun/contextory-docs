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
- [ADR-008 音声・動画をローカル前処理してからAIへ渡す](ADR-008-local-media-preprocessing.md)
- [ADR-009 Source統一モデルとAnalysis Revisionを正規モデルにする](ADR-009-analysis-source-revisions.md)
- [ADR-010 前面アプリ検知によるローカル録音確認を採用](ADR-010-local-foreground-recording-reminder.md)
- [ADR-011 Source／Group／Task関係の正本を単一Bundleへ限定する](ADR-011-bundle-relationship-ownership.md)
- [ADR-012 Claude実行には最小一時staging directoryを使用する](ADR-012-minimal-claude-staging.md)
- [ADR-013 外部Output公開は承認済みMarkdownとAdapterを介して行う](ADR-013-approved-external-publication.md)
- [ADR-015 Topic SourceとTask／WBSを既存正本モデルへ追加する](ADR-015-topic-source-task-wbs.md)
- [ADR-016 外部チケットは不変External Ticket Sourceとして取り込む](ADR-016-external-ticket-source.md)

## Amended

- [ADR-007 Taskの根拠来歴をID参照で保持](ADR-007-task-output-lineage.md)（ADR-009／011によりAmended）

## Superseded

- [ADR-006 Source・Context・Analysisを分離し派生結果を追加保存（legacy）](ADR-006-append-only-context-analysis.md)（ADR-009によりSuperseded）

## Retired

- [ADR-014 Transcript訂正を不変Sourceとし、用語辞書で決定的に補正する](ADR-014-transcript-correction-terminology.md)（canonical model consolidation。App撤去・検証は進行中）
