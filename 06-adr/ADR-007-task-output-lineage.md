# ADR-007: TaskとOutputの根拠来歴をID参照で保持する

- Status: Accepted
- Date: 2026-08-11

## Context

顧客からの見積依頼を根拠にJira依頼Taskを作り、後からJiraの見積結果を別Taskとして取り込み、両Taskから顧客向け回答を生成する。このとき最終Outputから、依頼、結果、画像、PDF、テキスト、各AI解析まで遡れる必要がある。

## Decision

- Taskは固有IDを持つTask Bundleとして保存する。
- Taskは`sourceIds`、`analysisIds`、`parentTaskIds`を保持する。
- 派生Output Taskは入力Taskを`parentTaskIds`、直接利用した解析・原本を各ID配列として保持する。
- Task作成や再生成でSource、Analysis、親Taskを上書き・移動しない。
- Task来歴は`task.json`を正本とし、SQLite `tasks` Indexを再構築可能にする。
- ユーザー確認前のTaskとOutputは`proposed`として扱う。
- 来歴グラフ描画は必須とせず、将来ノードと辺として表示できるID参照を先に保存する。

## Consequences

- 最終Outputの文言から根拠となるTask、Analysis、Sourceへ逆引きできる。
- 同じ見積結果をBacklog返信、メール、Excelなど複数Outputへ再利用できる。
- Taskの統合・分岐が増えても過去履歴を保持できる。
- SourceやTask削除時の参照整合性確認と、来歴表示UIが別途必要になる。
