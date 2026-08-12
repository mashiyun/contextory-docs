# ADR-007: Taskの根拠来歴をID参照で保持する

- Status: Amended by ADR-009 and ADR-011
- Date: 2026-08-11
- Amended by: [ADR-009](ADR-009-analysis-source-revisions.md), [ADR-011](ADR-011-bundle-relationship-ownership.md)

## Context

顧客からの見積依頼を根拠にJira依頼Taskを作り、後からJiraの見積結果を別Taskとして取り込み、両Taskから顧客向け回答を生成する。このとき最終Outputから、依頼、結果、画像、PDF、テキスト、各AI解析まで遡れる必要がある。

## Decision

- Taskは固有IDを持つTask Bundleとして保存する。
- Taskは`parentTaskIds`を保持する。Task–Source／Task–Groupの正規関係はADR-011に従い、`task.json`の`sourceLinks`／`groupLinks`へ保存する。
- 既存の`sourceIds`、`analysisIds`は読み取り互換のために保持できる。legacy `analysisId`はADR-009に従って正規`sourceId`へ対応付ける。
- OutputはTaskの特別な正規形ではなく、ADR-009に従う`kind: output`の派生Sourceとする。既存の派生Output Taskは読み取り可能なTask来歴として残す。
- Task作成や再生成でSource、Analysis、親Taskを上書き・移動しない。
- Task来歴は`task.json`を正本とし、SQLite `tasks`、`task_sources`、`task_groups` IndexをBundle走査で再構築可能にする。
- ユーザー確認前のTaskとOutputは`proposed`として扱う。
- 来歴グラフ描画は必須とせず、将来ノードと辺として表示できるID参照を先に保存する。

## Consequences

- Taskから根拠となるTask、Source、legacy Analysisへ逆引きできる。正規Output Sourceからも親SourceとTaskへ逆引きできる。
- 同じ見積結果をBacklog返信、メール、Excelなど複数Outputへ再利用できる。
- Taskの統合・分岐が増えても過去履歴を保持できる。
- SourceやTask削除時の参照整合性確認と、来歴表示UIが別途必要になる。
