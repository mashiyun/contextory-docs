# ADR-006: Source・Context・Analysisを分離し派生結果を追加保存する（legacy）

- Status: Superseded by ADR-009
- Date: 2026-08-11
- Superseded by: [ADR-009](ADR-009-analysis-source-revisions.md)

## Context

同じBacklog画像をFigmaキャプチャと組み合わせて問い合わせとして整理した後、別のSlack情報と組み合わせて見積もりとして解析する場合がある。統合結果をSource Bundleの単一`summary.md`へ保存すると、再解析で過去結果を上書きし、Sourceと解釈の境界も曖昧になる。

## Decision

このDecisionは歴史的な判断の記録として残す。旧Context／旧Analysisを通常Readerや来歴解釈へ残す根拠にはしない。新規書き込み、正規ID、Revision、Group、Outputの正規モデルはADR-009に従う。

- Sourceは不変の一次データとして個別に保持する。
- Contextは1件以上のSource IDの組み合わせとして独立保存する。新規設計では、関連Sourceを集める入れ物をGroupと呼び、Contextは移行期間の互換表現とする。
- 同じSourceを複数Context／Groupから参照できる。
- Analysisは解析目的ごとに固有IDを付け、Context／Group配下または単体Analysis領域へ追加保存する。
- 解析目的または使用Source集合が独立する再解析は新しいAnalysisを作り、過去のAnalysisを上書きしない。同じAnalysis Sourceへの追加情報を伴う更新は、[ADR-009](ADR-009-analysis-source-revisions.md)に従い新Revisionとして保存する。
- Analysisには根拠Source、Context、解析目的、Provider、モデル、生成日時、分類候補、レビュー状態を記録する。
- AI分類と解析結果はユーザー確認前に`proposed`とする。
- Context、Analysis、分類索引は永続ファイルからSQLiteへ再構築可能にする。

## Consequences

- 一次データを失わず、目的や組み合わせを変えた派生結果を継続的に追加できる。
- 過去のAI解釈と現在の解釈を比較できる。
- ContextとAnalysisの選択・確認・修正UIが別途必要になる。
- Local Vaultのファイル数は増えるが、原本の不要な複製は発生しない。
- Analysis Sourceの同一性とRevision履歴の境界はADR-009で具体化する。
- 旧`contexts/`は非対応形式であり、通常読込・新規書き込みを行わない。旧`analyses/`はcanonical Analysis Sourceへの一回限りの変換入力に限定する。
