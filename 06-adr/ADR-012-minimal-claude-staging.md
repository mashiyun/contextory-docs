# ADR-012: Claude実行には最小一時staging directoryを使用する

- Status: Accepted
- Date: 2026-08-12
- Related: [ADR-008](ADR-008-local-media-preprocessing.md), [ADR-009](ADR-009-analysis-source-revisions.md)

## Context

会社契約のClaude Codeは、ユーザーが明示取得・取り込みした業務情報の許可済み処理境界である。一方、Source Bundle全体をClaudeの作業ディレクトリにすると、選択していない原本、音声・動画、`localOpenUrl`までClaudeが読み取れる可能性がある。これは文脈限定、誤読、不要なトークン消費の観点で不適切である。

個人Claudeには業務情報を送信しない。パスワード入力画面、APIキー、Cookie、セッショントークンを表示する管理画面などは、ユーザーが明示的に取得・取り込みしない運用とする。万能な認証情報検出、マスク、fail-closedは実装しない。自動安全化はURLのquery／fragmentに限定する。

## Decision

- Claude実行ごとに、`jobId`と`createdAt`を持つSource Bundle外の一時staging directoryを作る。Source Bundle全体をClaudeの作業ディレクトリにしない。
- staging directoryには、選択済みの通常画像、テキスト、安全化済みURL、前処理済み文字起こし、代表フレームを配置する。
- URLを含む画像はURL領域をマスクした派生画像を配置し、`provided-text`はquery／fragmentを安全化した派生テキストを配置する。
- 音声・動画原本、`localOpenUrl`、未選択Sourceは配置しない。
- Claude Codeにはstaging directoryの選択済みファイルだけをRead限定で渡す。個人Claudeを送信先にしない。
- 処理完了、失敗、タイムアウト、中断のいずれでもstaging directoryを削除する。次回起動時には、実行中Jobへ属さない残存stagingも削除する。監査記録には送信時の入力ID、ハッシュ、モデル、prompt schema、結果状態を残せるが、`localOpenUrl`を残さない。

## Consequences

### Positive

- 会社契約Claudeへ送信する業務情報を、ユーザーが選択した文脈へ限定できる。
- Group展開・AI対話・再分析の送信文脈を明示選択したSourceだけに限定できる。
- Claudeが無関係なファイルを誤読することと、不要なトークン消費を抑えられる。

### Negative

- staging生成、整合性確認、例外時削除をProcessing Jobとして実装・テストする必要がある。
- staging削除失敗と、実行中Jobに属さない残存stagingを検出して安全に再試行する運用が必要になる。
