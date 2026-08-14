# ADR-014: Transcript訂正を不変Sourceとし、用語辞書で決定的に補正する

- Status: Accepted
- Date: 2026-08-15
- Related: [ADR-005](ADR-005-separate-system-and-microphone-audio.md), [ADR-008](ADR-008-local-media-preprocessing.md), [ADR-009](ADR-009-analysis-source-revisions.md), [ADR-012](ADR-012-minimal-claude-staging.md)

## Context

実利用で、ローカルWhisperが固有名詞、製品名、社内用語、人名を誤変換することを確認した。誤ったTranscriptからSummaryとタスク分類を生成すると、誤りが下流へ伝播する。

対応の選択肢は3つある。

1. 生Transcriptをユーザーが直接編集して上書きする。
2. Whisperモデルを追加学習・fine-tuningする。
3. 生Transcriptを原本として保持し、訂正と用語辞書を別レイヤーで管理する。

1は原本性、来歴、再現性を失う。ADR-009の追加専用モデルとも矛盾する。2は学習データ整備、再現性、モデル配布、検証の負担が単一ユーザーの非公開ツールに見合わず、ADR-008で決めたモデル配置・SHA-256照合の運用も崩れる。

## Decision

- 生Transcriptは不変の原本として保持する。ユーザー訂正、辞書補正、再解析のいずれでも上書き・削除しない。ADR-005に従い`system`と`microphone`を別Transcriptとして保持し、参照は`rawTranscriptRefs`配列（録音Source ID、role、パス、SHA-256、言語、Whisperモデル）で持つ。
- role別Transcriptを統合する場合は、統合結果の不変snapshot、SHA-256、`mergeAlgorithmVersion`、入力順序をRevisionへ保存する。統合結果は生Transcriptを置き換えない。
- ユーザーの訂正は、対象録音Source、対象Transcript、音声モデルを固定参照する不変の訂正Sourceとして追加する。訂正Sourceは`kind: input`、`type: transcript_correction`とし、`parentSourceIds`へ対象Sourceを保存する。訂正をrole別と統合後のどちらへ適用したかも記録する。訂正Sourceは来歴用であり、通常Inputの保存後自動解析Queueへ単独投入せず、親Analysisの明示的な再生成Jobだけが参照する。
- 訂正を適用したTranscript snapshotは、対象Analysis SourceのRevisionへファイルとして保存し、パスとSHA-256を記録する。Revisionからは`rawTranscriptRefs`、統合snapshot、訂正Source ID、`dictionaryRevisionRefs`、音声モデル、生成モデルを逆引きできる。
- SummaryとActionsは訂正版Transcriptから再生成し、過去Revisionを保持する。
- Revisionは実際の変換を`transcriptTransformSteps`の順序付き配列として保存する。MVPの`pipelineVersion: 1`は、role別辞書補正、role別ユーザー訂正、role統合、統合後辞書補正、統合後ユーザー訂正の順とし、各stepの入力・出力snapshotとSHA-256、algorithm version、訂正Sourceまたは辞書Revision参照を固定する。隣接stepのhashが連続しない場合はfail-closedとする。
- `mergeAlgorithmVersion: 1`はsegmentを録音session相対の開始時刻、終了時刻、role順（`system`、`microphone`）、元segment連番の順で安定sortし、roleと時刻を残す。重複除去、話者推測、本文の自動統合を行わず、必須時刻が欠落・不正なら`preprocessing_failed`として統合とAnalysisを生成しない。
- MVPの訂正位置は、固定参照したsnapshot上のUTF-8 byte半開区間とする。UTF-8 scalar境界、訂正前byte列、範囲の非重複を検証し、offsetの大きい順に適用する。別モデルのTranscriptへ自動追従させず、ユーザー確認済みの新しい訂正Sourceを要求する。
- `operationId`を同一操作の再試行・復旧に使うidempotency keyとする。同じ`operationId`の再実行は既存Revisionへ収束させ、別`operationId`による意図的な再生成は新しいRevisionとして許可する。
- 監査用に`requestFingerprint`を別途保存する。fingerprintにはoperation種別、purpose、provider、model、`promptSchemaVersion`、role別生Transcript参照とhash、統合Transcript hash、訂正Source IDとhash、`dictionaryRevisionRefs`・使用した`entryId`・staging用辞書excerptのhash、訂正版snapshot hash、pipeline versionと各algorithm version、選択した追加Source IDとhashをcanonical順で含める。fingerprintが一致するだけで、異なる`operationId`のoperationを自動統合しない。
- 用語辞書は共通辞書とGroup／案件別辞書をLocal Vaultへ保持し、誤表記、正しい表記、スコープ、登録日時、根拠訂正Source ID、有効状態を保存する。辞書の登録、変更、無効化は必ずユーザー確認を挟み、自動確定しない。
- 辞書は追加専用のRevisionとして更新する。項目の修正・削除も新しいRevisionとして記録し、過去Revisionと補正履歴を保持して過去Analysisの再現性を維持する。
- 適用した辞書は`dictionaryRevisionRefs`配列（`scope`、`scopeId`、`revisionId`、`snapshotPath`、`snapshotSha256`）として保存する。MVPで同時適用できるのは共通辞書とユーザーが明示選択した1つのGroup辞書までとし、複数Group所属時も自動選択しない。適用した配列はAnalysis Revision、補正履歴、Whisperヒント履歴へ同じ内容で固定する。
- 辞書は次回以降の録音でWhisperへ渡す用語ヒントと、文字起こし後の決定的な表記補正に使う。補正前Transcriptは必ず保存し、補正後は別ファイルとする。自動補正履歴として適用項目、位置、`dictionaryRevisionRefs`、適用対象、補正algorithm version、適用日時、未適用項目と理由を保存する。
- `normalizationAlgorithmVersion: 1`はUnicode scalar列の完全一致だけを対象とし、暗黙の大文字小文字・全角半角・かな・送り仮名・Unicode正規化を行わない。決定的補正は左から右への単一passとし、置換結果を再置換しない。候補の順位は「UTF-8 byteでの開始位置が早い → 誤表記のUnicode scalar数が長い → 選択Group辞書を優先」とする。
- 選択したGroup辞書は共通辞書より優先し、同じ誤表記を意図的に上書きできる。競合として自動適用を止めるのは、同一scope内で同じ誤表記が複数の正しい表記を持つ場合だけとする。順位規則で結果が一意に決まらない場合と不正項目も自動適用せず、未適用として記録する。
- Whisperモデル自体の学習・fine-tuningは行わない。用語ヒントはモデルの重みを変更しない。
- Transcript訂正による再解析でstagingへ置けるのは、訂正版Transcript、適用済み辞書項目だけの派生excerpt、およびユーザーが選択した代表フレーム、画像、テキスト、安全化済みURLとする。`dictionaryExcerptVersion: 1`は元辞書Revision参照、使用した`entryId`、正しい表記、scopeを固定key順で持つUTF-8 JSONとし、entryをscope（common、group）と`entryId`でsortする。excerpt正本はRevision監査領域へ不変保存し、永続パスとSHA-256を記録してstagingにはcopyだけを置く。辞書Revision snapshot全体はユーザーが明示選択しない限り配置しない。この限定は訂正再解析の規則であり、初回メディア解析やその他の送信へ拡大しない。
- 音声・動画原本はClaudeへ送らない。初回メディア解析の前処理境界はADR-008、staging境界全体はADR-012を正本とする。

## Consequences

### Positive

- 原本、AIの解釈、ユーザー訂正の境界を保ったまま、誤変換を修正できる。
- 訂正版から再生成したSummaryとActionsを、どの入力から得たか逆引きできる。
- 同じ誤りを次回以降の録音で自動的に減らせる。
- モデルの追加学習と配布・検証の負担を負わずに、精度の実用的な改善を得られる。

### Negative

- Transcriptがrole別の「生」、統合、「補正後」、「訂正版snapshot」の系統に分かれ、UIとRevisionでの提示を明示する必要がある。
- 辞書Revisionと補正履歴の保存・再索引、`dictionaryRevisionRefs`のsnapshot保持が追加で必要になる。
- `operationId`による収束と`requestFingerprint`による監査を別々に実装・検証する必要がある。
- 辞書項目が増えるほど、競合・曖昧項目の検出と確認導線が重要になる。
- 用語ヒントの効果は文字起こし方式に依存し、PoCでの確認が必要になる。

## Relationship to existing ADRs

ADR-008を置き換えず補完する。前処理としてのローカル文字起こし、モデル配置、fail-closedの境界は維持し、その出力に対する訂正と辞書補正のレイヤーを追加する。ADR-005のrole別原本・role別文字起こし、ADR-009のRevision正本性、ADR-012のstaging境界も変更しない。
