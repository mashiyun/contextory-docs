# ADR-011: Source／Group／Task関係の正本を単一Bundleへ限定する

- Status: Accepted
- Date: 2026-08-12
- Related: [ADR-007](ADR-007-task-output-lineage.md), [ADR-009](ADR-009-analysis-source-revisions.md)

## Context

TaskとSource／Group、GroupとSourceはいずれも多対多である。両側のBundleとSQLiteに同じ関係を正本として書くと、片側だけの更新、障害時の不整合、再索引時の優先順位不明が起きる。

## Decision

- Task–SourceおよびTask–Group関係の唯一の正本は各Task Bundleの`task.json`とする。
- `task.json`は`sourceLinks`と`groupLinks`を持ち、各linkに相手ID、role、追加日時を保存する。
- Group–Source関係の唯一の正本は各Group Bundleの`group.json`とする。
- `group.json`は`sourceLinks`を持ち、各linkにSource ID、role、追加日時を保存する。
- Source Manifestの`taskIds`と`groupIds`は関係の正本・通常Readerに使わない。関係は`task.json`と`group.json`だけから解決し、旧配列を持つManifestは次のcanonical書き込みで除去できる。
- SQLiteの`task_sources`、`task_groups`、`group_sources`は索引とし、Bundle走査で再構築する。関係の変更で逆方向BundleやSQLiteを正本として同期更新しない。

## Consequences

### Positive

- 各関係の更新先と障害時の復旧元が一意になる。
- SQLite削除・再構築時にもBundleの正本から関係を回復できる。
- 双方向ファイル更新を避け、原本と来歴の不整合を減らせる。

### Negative

- Source詳細で逆方向のTask／Groupを表示するにはSQLite索引またはBundle走査が必要になる。
- 旧配列を持つ既存Manifestを読んでも関係へ反映しない。`task.json`または`group.json`を検証できない場合は参照解決をfail-closedにする。
