# Transcript訂正・用語辞書要件

## Status

未実装の将来仕様。正規Sourceモデル（[ADR-009](../06-adr/ADR-009-analysis-source-revisions.md)）とローカル前処理（[ADR-008](../06-adr/ADR-008-local-media-preprocessing.md)）の上に追加する。判断は[ADR-014](../06-adr/ADR-014-transcript-correction-terminology.md)に従う。

## 目的

ローカルWhisperの文字起こしは、固有名詞、製品名、社内用語、人名を誤変換する。誤ったTranscriptのままAnalysisを生成すると、Summaryとタスク分類まで誤りが伝播する。ユーザーが誤りを訂正でき、同じ誤りを次回以降の録音で自動的に減らせるようにする。

これはWhisperモデル自体の学習・fine-tuningではない。モデルの重みは変更せず、文字起こし前の用語ヒントと、文字起こし後の決定的な表記補正、およびユーザーによる訂正記録だけで扱う。

## Transcript原本と訂正

### 必須要件

- `whisper-cli`が生成した生Transcriptは不変の原本として保持する。訂正、辞書補正、再生成のいずれでも上書き・削除しない。
- 生Transcriptは[ADR-005](../06-adr/ADR-005-separate-system-and-microphone-audio.md)に従い、`system`と`microphone`を別のTranscriptとして不変で保持する。参照は`rawTranscriptRefs`配列とし、各要素へ録音Source ID、role、パス、SHA-256、言語、Whisperモデルを保存する。
- role別Transcriptを時刻順へ統合する場合は、統合結果の不変snapshotのパスとSHA-256、`mergeAlgorithmVersion`、統合時の入力順序をRevisionへ保存する。統合結果は生Transcriptを置き換えない。
- ユーザーの訂正は生Transcriptへの上書きではなく、独立した不変の訂正Sourceとして追加する。訂正Sourceは対象の録音Source、対象Transcript、使用した音声モデルを固定参照する。
- `type: transcript_correction`は来歴を保持するユーザー操作Sourceであり、通常Inputの保存後自動解析Queueへ単独投入しない。訂正Sourceの確定と、親Analysis Sourceを訂正版から再生成するProcessing Jobは別操作として記録する。
- 訂正Sourceには、訂正前文字列、訂正後文字列、対象位置、訂正理由、作成日時を保存する。MVPの対象位置は、固定参照したsnapshotの先頭から数えたUTF-8 byteの半開区間`[startUtf8ByteOffset, endUtf8ByteOffset)`とする。両端のUTF-8 scalar境界、対象byte列と訂正前文字列の一致、範囲の非重複を検証し、不一致ならfail-closedで確定しない。1回の訂正操作で複数箇所を訂正した場合も、1つの訂正Sourceへ構造化し、offsetの大きい順に適用する。
- 訂正をrole別Transcriptと統合後Transcriptのどちらへ適用したかを記録する。適用対象のパスとSHA-256も固定する。
- 訂正を適用したTranscript snapshotは、対象Analysis SourceのRevisionへファイルとして保存し、パスとSHA-256を記録する。snapshotを持たないRevisionを有効なRevisionとして扱わない。
- Revisionからは、`rawTranscriptRefs`、統合Transcript snapshot、適用した訂正Source ID、適用した`dictionaryRevisionRefs`、音声モデル、Provider／モデルを逆引きできる。
- 訂正版Transcriptからは、SummaryとActionsを再生成する。過去Revisionのsummaryとその根拠は保持する。
- ユーザー補足（`user-notes.md`）とTranscript訂正は別の記録として扱う。補足は画面外の前提やAI解釈への訂正であり、Transcript訂正は文字起こし文字列の誤りに対する訂正である。
- 訂正操作で、原音、生Transcript、過去Revisionを削除しない。
- 別モデルによる再文字起こしへ訂正位置を自動追従させない。新しいTranscript snapshotに対してユーザーが内容を確認し、新しい訂正Sourceを作成する。

### Transcript変換順序

- Revisionは実際に行った変換を`transcriptTransformSteps`の順序付き配列として保存する。各stepは種別、algorithm version、入力Source ID・パス・SHA-256、出力snapshotのパスとSHA-256、使用した訂正Sourceまたは`dictionaryRevisionRefs`を持つ。
- MVPの`pipelineVersion: 1`は、role別辞書補正、role別ユーザー訂正、role統合、統合後辞書補正、統合後ユーザー訂正の順とする。不要なstepは省略できるが、順序を入れ替えない。
- 前stepの出力SHA-256と次stepの入力SHA-256が一致しない場合はfail-closedとし、RevisionとAnalysisを生成しない。
- ユーザー訂正は同じ対象に対する辞書補正より後へ固定し、辞書変更でユーザー訂正を黙って上書きしない。
- `mergeAlgorithmVersion: 1`は、各segmentの録音session開始からの`startMilliseconds`、`endMilliseconds`、role、元segment連番を入力とし、開始時刻、終了時刻、role順（`system`、`microphone`）、元segment連番の順で安定sortする。出力にはroleと時刻を残し、重複除去、話者推測、本文の自動統合を行わない。必須時刻の欠落・不正・逆転を検出した場合は`preprocessing_failed`としてfail-closedに停止し、統合TranscriptとAnalysisを生成しない。

### 冪等性と再生成

- `operationId`を、同一操作の再試行・復旧に使うidempotency keyとする。値はProcessing Job作成時にUUIDとして確定し、実行前に永続化する。
- 同じ`operationId`の再実行は新規Revisionを作らず、既存Revisionへ収束させる。
- 別の`operationId`による意図的な再生成は、新しいRevisionとして許可する。
- 監査用に`requestFingerprint`を別途保存する。fingerprintには次をcanonical順で含める。
  1. operation種別
  2. purpose
  3. provider
  4. model
  5. `promptSchemaVersion`
  6. role別生Transcriptの参照とhash
  7. 統合Transcriptのhash
  8. 訂正Source IDとhash
  9. `dictionaryRevisionRefs`、使用した`entryId`、staging用辞書excerptのhash
  10. 訂正版snapshotのhash
  11. pipeline version、統合algorithm version、辞書補正algorithm version、訂正algorithm version
  12. 選択した追加Source IDとhash
- `requestFingerprint`が一致するだけで、異なる`operationId`のoperationを自動統合しない。fingerprintは監査と差分確認に使い、収束の判断には使わない。

### 受入条件

- 訂正後も生Transcriptの原文を確認でき、role別の内容とSHA-256が変わらない。
- 詳細画面から、role別原文、統合Transcript、現在の訂正版、訂正履歴、過去Summaryを確認できる。
- 任意のRevisionについて、`rawTranscriptRefs`、統合snapshotと`mergeAlgorithmVersion`、訂正Source、`dictionaryRevisionRefs`、音声モデル、生成モデルを確認できる。
- 訂正がrole別と統合後のどちらへ適用されたかを確認できる。
- `transcriptTransformSteps`の入出力SHA-256が連続し、辞書補正後にユーザー訂正が適用されたことを確認できる。不連続ならRevisionとAnalysisが生成されない。
- 訂正位置がUTF-8 scalar境界でない、訂正前byte列と一致しない、または範囲が重なる場合は訂正SourceとRevisionが確定されない。
- `type: transcript_correction` Sourceを保存しても通常Inputの自動解析Queueへ投入されず、親Analysisの明示的な再生成Jobだけが参照する。
- 同じ`operationId`で再実行してもRevisionが増えず、別`operationId`の再生成では新しいRevisionが残る。
- 任意のRevisionについて`requestFingerprint`を確認でき、fingerprint一致だけで別operationが統合されていない。

## 用語辞書

### 必須要件

- 共通辞書と、Group／案件別辞書をLocal Vault内に保持する。外部サービスへ辞書を同期しない。
- 各辞書項目は、誤表記、正しい表記、スコープ（共通またはGroup ID）、登録日時、根拠となる訂正Source ID、有効状態を保存する。
- 辞書への登録、変更、無効化は必ずユーザー確認を挟む。AIやアプリが訂正内容から自動確定しない。登録候補の提示までをAIの役割とする。
- 辞書は次の2箇所で利用する。
  - 次回以降の録音の文字起こし時に、Whisperへ渡す用語ヒント。
  - 文字起こし後の、決定的な規則による表記補正。
- 用語ヒントを渡した場合も、Whisperモデルの重みは変更しない。ヒントとして渡した辞書は、Whisperヒント履歴へ`dictionaryRevisionRefs`と同じ形式で記録する。
- 適用した辞書は単一の`dictionaryRevision`ではなく`dictionaryRevisionRefs`配列として保存する。各要素は`scope`、`scopeId`、`revisionId`、`snapshotPath`、`snapshotSha256`を持つ。
- MVPで同時に適用できるのは、共通辞書と、ユーザーが明示選択した1つのGroup辞書までとする。対象Sourceが複数Groupへ属する場合もGroup辞書を自動選択せず、ユーザーが1つ選ぶ。
- 適用した`dictionaryRevisionRefs`は、Analysis Revision、補正履歴、Whisperヒント履歴へ同じ内容で固定する。
- 表記補正の前後で、補正前Transcriptを必ず保存する。補正結果は別ファイルとして保存し、補正前ファイルを上書きしない。
- 自動補正履歴として、適用した辞書項目、適用位置、適用時の`dictionaryRevisionRefs`、適用対象（role別か統合後か）、`normalizationAlgorithmVersion`、適用日時、未適用項目と理由を保存する。
- 辞書項目を修正・削除しても、過去Analysisの再現性を維持する。辞書は追加専用のRevisionとして更新し、過去Revisionと、過去Revisionを参照する補正履歴を保持する。

### 決定的補正の規則

- `normalizationAlgorithmVersion: 1`はUnicode scalar列の完全一致だけを対象とし、大文字小文字、全角半角、かな、送り仮名、Unicode正規化による暗黙の同一視を行わない。必要な表記揺れは別項目としてユーザー確認付きで登録する。
- 走査は左から右への単一passとし、置換結果を再び置換対象にしない（連鎖置換の禁止）。
- 候補の順位は「UTF-8 byteでの開始位置が早い → 誤表記のUnicode scalar数が長い → 選択したGroup辞書を優先」で決める。
- 選択したGroup辞書は共通辞書より優先し、同じ誤表記に対する共通辞書の正しい表記を意図的に上書きできる。これはscopeをまたぐ正常な優先であり、競合として扱わない。
- 同一scope内で同じ誤表記が複数の正しい表記を持つ場合だけ、競合として自動適用しない。
- 上記の順位でも結果が一意に決まらない場合は適用せず、未適用として補正履歴へ記録する。
- 次の項目は自動補正へ使わず、ユーザー確認対象として提示する。
  - 同一scope内で同じ誤表記に異なる正しい表記が定義されている競合項目。
  - 順位規則で結果が一意に決まらない項目。
  - 空文字、または他項目の正しい表記を誤表記として持つ不正な項目。
- 補正結果はAI提案ではなく、規則による決定的な変換として扱う。ただしユーザーは補正結果をさらに訂正できる。

### 受入条件

- ユーザー確認なしに辞書項目が増えない。
- 同じ`dictionaryRevisionRefs`と同じ生Transcriptからは、常に同じ補正結果が得られる。
- 補正前Transcriptと補正後Transcriptの両方を確認でき、どの項目がどの位置へ適用されたか確認できる。
- 辞書項目を削除した後も、その項目を使った過去Analysisの入力と結果を、保存済みsnapshotから再現できる。
- 選択Group辞書が共通辞書の同じ誤表記を上書きした場合、競合として止まらず適用され、履歴から適用元scopeを確認できる。
- 同一scope内の競合、順位不確定、不正項目は自動適用されず、未適用として理由とともに確認できる。
- 複数Groupへ属するSourceでは、Group辞書が自動選択されず、ユーザーが選んだ1つだけが適用される。
- `normalizationAlgorithmVersion: 1`では完全一致以外の暗黙の表記正規化が行われず、同じ入力と`dictionaryRevisionRefs`から同じ補正結果が得られる。

## セキュリティと保存

- 生Transcript、訂正Source、訂正版Transcript、辞書、補正履歴はすべてLocal Vault内で管理する。
- staging境界全体の正本は[ADR-012](../06-adr/ADR-012-minimal-claude-staging.md)、初回メディア解析の前処理境界の正本は[ADR-008](../06-adr/ADR-008-local-media-preprocessing.md)とする。本節はそれらを狭めるものではない。
- 「訂正版Transcriptと辞書だけを送る」規則は、Transcript訂正による再解析に限定した規則である。初回メディア解析やその他の送信へ拡大適用しない。
- Transcript訂正の再解析でstagingへ置けるのは、訂正版Transcript、解析に必要な適用済み辞書項目だけを抽出した派生excerpt、およびユーザーが選択した代表フレーム、画像、テキスト、安全化済みURLとする。`dictionaryExcerptVersion: 1`はUTF-8 JSONとし、元`dictionaryRevisionRefs`、使用した`entryId`、正しい表記、scopeを固定key順で保存し、entryはscope（common、group）と`entryId`の順にsortする。excerpt正本はRevision監査領域へ不変保存し、永続パスとSHA-256を記録したうえでstagingへcopyする。辞書Revision snapshot全体は、ユーザーがその送信を明示選択しない限りstagingへ置かない。
- 音声・動画原本はClaudeへ渡さない。訂正・辞書機能でもこの境界を変更しない。
- `localOpenUrl`、未選択Source、Source Bundle全体はstagingへ置かない。
- 訂正・辞書操作で、原音、生Transcript、過去Revisionを削除しない。Source削除は既定の保護ロックと参照整合性確認を通す手順に従う。

### 受入条件

- stagingに音声・動画原本、Source Bundle全体、未選択Source、`localOpenUrl`、無条件の辞書Revision snapshotが含まれない。
- stagingへcopyした辞書excerptと、Revision監査領域の不変excerpt正本のSHA-256が一致し、処理後にstagingだけが削除される。

## 実装順

[開発フェーズ](../04-roadmap/development-phases.md)の「実環境フィードバック反映の実装順」に従い、Transcript訂正とRevision再生成、共通／Group辞書、決定的補正、PoC後のWhisperヒントの順で実装する。
