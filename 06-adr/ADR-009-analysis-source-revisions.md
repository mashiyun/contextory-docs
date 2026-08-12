# ADR-009: Source統一モデルとAnalysis Revisionを正規モデルにする

- Status: Accepted
- Date: 2026-08-12
- Supersedes: [ADR-006](ADR-006-append-only-context-analysis.md)
- Amends: [ADR-007](ADR-007-task-output-lineage.md)のOutput正規モデル

## Context

ユーザーは既存のAnalysis Sourceへ、後から得たテキスト、画像、URLを追加して同じ分析を深めたい。一方で、summaryだけを置き換えると、何を追加し、どの指示・モデルで解釈が変化したかを失い、AIの解釈とユーザー確認の監査もできない。

ADR-006はAnalysisを追加専用で保存する方針を定めたが、Context、Analysis、Outputを別の永続単位として扱っており、同一のAnalysis SourceをUI上で更新する場合の履歴粒度、`summary.md`の正本性、既存Analysisとの移行境界を定めていない。ADR-007もOutput TaskをOutputの正規形としているため、Input、Analysis、Outputを同じSourceとして再利用するモデルと一致しない。

## Decision

- Input、Analysis、Outputを同じSourceモデルで扱い、正規IDを`sourceId`とする。Analysisは`kind: analysis`、Outputは`kind: output`の派生Sourceとして表現する。
- 親Source ID、生成操作、実際に使用した個別Source ID、生成日時、モデルを派生Sourceに固定保存する。Groupからの生成でもGroup IDだけでなく展開済みの個別Source IDを保存する。
- 新規のInput、Analysis、Outputは正規Source Bundleとして`sources/`配下へ書き込む。既存`analyses/`と`contexts/`は読み取り互換専用とし、新規書き込み先にしない。
- legacy `analysisId`から正規`sourceId`への対応を永続保存する。移行時は旧Bundleを上書きせず、既存summaryをRevision 1の不変snapshotとして正規Source側へ保存する。
- 新規書き込みへの切替は、全legacy Analysisの対応作成、Revision 1の`summaryPath`とSHA-256検証、Bundle走査でのSQLite再構築を確認した後に行う。確認不能なlegacy Analysisは読み取り互換に留め、手動レビュー対象とする。
- 同じAnalysis Sourceへのテキスト、画像、URL追加と再分析は、新しいAnalysis Sourceではなく不変のAnalysis Revisionを追加する。
- Revisionは連番、日時、理由、追加情報、追加画像Source ID、ユーザー指示、使用Provider／モデル、確認状態、実際に使用したSource ID、直前との差分、summary本文の不変snapshot、`summaryPath`、SHA-256を構造化して保存する。ハッシュだけのRevisionは認めない。
- Revisionへ追加する画像は独立した不変Sourceとして保存する。音声、動画、PDFはRevisionへ直接追加せず、新規Sourceとして取り込んで前処理・Analysisを行う。
- 最新の`summary.md`は最新Revisionの保存済みsnapshotから生成するmaterialized viewとし、Revision履歴から再生成可能にする。過去summaryを削除または上書きしない。
- Analysis Source上のAI対話は追加専用の構造化履歴として保存する。各対話に`analysisSourceId`、`revisionId`、送信時summaryのパスとSHA-256、Group展開済みSource ID snapshot、使用したSource／派生物IDとハッシュ、モデル、prompt schema version、送信日時、結果状態を保存する。対話によってsummaryが変わる場合は、対応する新Revisionを追加する。
- AI対話、再分析、Group展開を含むすべての送信にADR-008を適用する。音声・動画原本を送信せず、前処理済み文字起こし・代表フレームだけを使う。未前処理または前処理失敗メディアを含む場合はfail-closedで送信を止める。
- すべてのClaude送信はADR-012の最小一時staging directoryを通す。Source Bundle全体と`localOpenUrl`をClaudeの作業ディレクトリに置かない。選択済みの通常画像・テキストは配置可能とし、URLはquery／fragmentを安全化する。
- 解析目的または使用Sourceの組み合わせが独立した分析を表す場合は、新しい派生Analysis Sourceを作る。Groupから生成する場合も実際に使用した個別Source IDを固定する。

## Consequences

### Positive

- 同じAnalysis Sourceを継続利用しながら、追加情報とAI解釈の変化を失わない。
- 過去summary、差分、根拠、使用モデルを詳細画面で確認できる。
- `summary.md`の破損・再生成時にも、構造化履歴を正本として復元できる。
- 正規Sourceモデルへ段階移行しても、既存Analysis Bundleを失わずに読み取れる。
- 画像追加の原本性と、音声・動画・PDFの前処理境界を保てる。

### Negative

- Revision、追加情報、対話、差分を保存・再索引するスキーマとUIが必要になる。
- 最新summaryの投影とRevision正本の整合性検査が必要になる。
- 「同一分析の更新」と「新しい派生分析」の選択基準をUIで明示する必要がある。
- legacy `analysisId`と`sourceId`の対応不全、Revision snapshot不全は新規書き込みの切替を阻害するため、検出・手動レビューが必要になる。

## Supersession and amendment

この判断によりADR-006をSupersededとする。ADR-006の既存Bundleは読み取り互換のために残すが、新規書き込みは本ADRに従う。ADR-007のTask来歴と`task.json`正本の方針は維持する一方、Output Taskを正規Outputとする部分は`kind: output`の派生Sourceへ置き換える。Task–Source／Task–Group／Group–Source関係の単一正本はADR-011に従う。
