# ADR-006: Source・Context・Analysisを分離し派生結果を追加保存する

- Status: Accepted
- Date: 2026-08-11

## Context

同じBacklog画像をFigmaキャプチャと組み合わせて問い合わせとして整理した後、別のSlack情報と組み合わせて見積もりとして解析する場合がある。統合結果をSource Bundleの単一`summary.md`へ保存すると、再解析で過去結果を上書きし、Sourceと解釈の境界も曖昧になる。

## Decision

- Sourceは不変の一次データとして個別に保持する。
- Contextは1件以上のSource IDの組み合わせとして独立保存する。
- 同じSourceを複数Contextから参照できる。
- Analysisは解析目的ごとに固有IDを付け、Context配下または単体Analysis領域へ追加保存する。
- 再解析は新しいAnalysisを作り、過去のAnalysisを上書きしない。
- Analysisには根拠Source、Context、解析目的、Provider、モデル、生成日時、分類候補、レビュー状態を記録する。
- AI分類と解析結果はユーザー確認前に`proposed`とする。
- Context、Analysis、分類索引は永続ファイルからSQLiteへ再構築可能にする。

## Consequences

- 一次データを失わず、目的や組み合わせを変えた派生結果を継続的に追加できる。
- 過去のAI解釈と現在の解釈を比較できる。
- ContextとAnalysisの選択・確認・修正UIが別途必要になる。
- Local Vaultのファイル数は増えるが、原本の不要な複製は発生しない。
