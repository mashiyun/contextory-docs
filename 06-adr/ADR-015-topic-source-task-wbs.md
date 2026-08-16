# ADR-015: Topic SourceとTask／WBSを既存正本モデルへ追加する

- Status: Accepted
- Date: 2026-08-15
- Related: [ADR-005](ADR-005-separate-system-and-microphone-audio.md), [ADR-008](ADR-008-local-media-preprocessing.md), [ADR-009](ADR-009-analysis-source-revisions.md), [ADR-011](ADR-011-bundle-relationship-ownership.md), [ADR-012](ADR-012-minimal-claude-staging.md), [ADR-014](ADR-014-transcript-correction-terminology.md)

## Context

会議、動画、長文に含まれる話題、Decision、Actionを個別に扱うには、元の記録を分割・複製せず、固定した根拠範囲を持つ派生Sourceが必要である。同時に、Action候補は作業の根拠であってTaskの正本ではない。Task、WBS、返答待ち、blockerを別の重複データへ持つと、Source／Groupとの整合性と削除保護が崩れる。

## Decision

- Topic Sourceを`kind: topic`、`type: topic_excerpt`の派生Sourceとして導入する。Input、Analysis、Outputと同格のSourceであり、Group、Task、Revision、AI相談、派生Outputに再利用できる。
- Topic Sourceは`parentSourceIds`と`lineage`を持つ。各Evidence Spanは`spanId`、一意な`displayOrder`、snapshot所有者の親Source ID、原音所有者のmedia Source ID、nullableな親Revision ID、raw／revision snapshot種別、安全な相対path、snapshotと選択byte列のSHA-256、単一media role、録音session相対の整数millisecond半開区間、Transcript UTF-8 byte半開区間を固定する。byte端点はUnicode scalar境界とする。元メディアとTranscriptを複製せず、再生はmedia Sourceのrole・時刻へ解決する。任意の派生クリップはキャッシュ的な派生物であり、根拠の正本にしない。
- spanは親Revision／snapshotへ固定参照する。Transcript訂正、辞書更新、親Sourceの更新によって自動移動しない。別内容を使うには新spanまたは新Revisionを追加する。
- `parentSourceIds`は全Evidence Spanのsnapshot所有Source IDとmedia Source IDのsorted unique unionに一致させ、この子→親参照をSource来歴edgeの正本とする。`VaultMutationLock`内で追加後の全DAGを検証し、参照先欠損、自己参照、推移的循環またはunion不一致があれば正本も索引も変更しない。Topic／Task／Group／Revision／派生Source／公開監査等から参照されるSourceの削除は拒否し、走査またはhash検証不能時もfail-closedとする。
- 手動の範囲選択によるTopic作成を必須とする。AIの話題／Decision／Action分割、ActionからTask、WBS分解、コメントからblockerの生成はすべて提案に留め、ユーザーの採用まで正本を作成・変更しない。確定Topicの作成者は採用操作を行った`user`とし、`creationOrigin`と不変な`proposalId`でAI提案経由を区別する。手動作成の`operationId`は保存し、proposal ID、provider／model／prompt schemaは`null`として明示する。
- AI提案本文とprovider／model／prompt／根拠の正本は生成元Analysis Revisionの構造化record、採否は同じAnalysis Sourceの追加専用review eventとする。SQLiteは索引に限り、採用後のTopic／Taskはproposal、生成元Revision、review eventを参照する。これにより提案をTaskやTopicの確定値へ先書きしない。
- TaskはSourceと別のTask Bundleを個人作業管理の正本とする。Task–Source／Task–GroupはADR-011どおり`task.json`の`sourceLinks`／`groupLinks`だけを正本とし、Source Manifestの逆方向配列とSQLiteは正本にしない。
- Taskには手動編集可能な作業属性、`parentTaskId`、`displayOrder`、`dependencyTaskIds`、不変なコメント／blocker recordと追加専用event、追加専用変更eventを保存する。eventには一意なID、単調増加sequence、`operationId`を持たせる。Task現在値とeventは`VaultMutationLock`内で同じ`task.json`へ1回のatomic replaceで書き、SQLiteは再構築可能な投影とする。返答待ちは実作業状態と別に表現し、ユーザー確定内容をAIが上書きしない。
- MVPのWBSはGroupにリンクされたTaskから投影する。WBS専用データを作らず、番号は階層と表示順から生成する。親子と依存はそれぞれ追加後の全graphをロック内で検証し、欠損、重複、自己参照、推移的循環を拒否する。Group化はTask／WBSを自動作成しない。
- MVPのTask／Group削除はhard deleteではなくarchiveとする。Task／Group Bundle、既存link、コメント、blocker、履歴を保持し、archive状態をそれぞれのBundleへ保存する。BundleのTrash移動、参照元の自動unlink、Source移動、cascade deleteは行わない。
- PM支援機能はSource／Group／Taskから導出するビューまたは再生成可能なcacheとし、Decision Log、RAID等の重複正本を持たない。ユーザー確定操作はSource作成またはTask変更として正本へ記録する。
- 既存Taskはversion 1／2を読み、新規Task writerは`schemaVersion: 3`とする。version 3 reader、SQLiteのnullable列・新規表、Bundle検証をwriterより先に導入し、一括backfillを行わず未知fieldをround-tripで保持する。

## Consequences

### Positive

- 会議の一部を独立して扱いながら、録音原本、`system`／`microphone`の区別、Transcript訂正時点を失わない。
- 手動のみでもTask・WBSを管理でき、AIを使う場合も確認境界を監査できる。
- 関係、削除保護、来歴、再索引の正本を既存ADRと同じBundleに限定できる。

### Negative

- Evidence Spanのsnapshot検証、来歴DAG、Task親子／依存の循環検査が必要になる。
- 複数Groupに属するTaskのWBS表示、blockerの直接原因と根本原因の投影、外部同期はUI・索引・検証を追加で設計する必要がある。
- PM支援ビューは正本を持たないため、表示のフィルタ、集計規則、更新タイミングを個別に定義する必要がある。

## Relationship to existing ADRs

ADR-005のrole別原本、ADR-008／012のAI送信境界、ADR-009のSource・Revision・`operationId`、ADR-011の関係正本、ADR-014のTranscript snapshotと訂正来歴を変更しない。これらのモデルをTopic Sourceの根拠とTaskの参照へ拡張する。
