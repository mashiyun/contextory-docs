# Source・Group・Task・Output要件

## Status

次回実装候補。2026-08-12時点では仕様整理のみで、未実装。

## Source中心モデル

Contextoryで保存する情報と成果物は、入力かAI生成かを問わず、同じ`Source`として扱う。Input、Analysis、OutputはUIや処理段階の呼称であり、永続モデルの別物ではない。

- キャプチャ、録音、録画、PDF、手動入力は一次Sourceである。
- AI Analysis、議事録、要件定義、返信案などの生成結果も派生Sourceである。
- Source単体または複数Sourceを材料として、新しい派生Sourceを生成できる。
- 派生Sourceをさらに別の生成へ再利用できる。
- 生成時に使用した親Source IDを固定して記録し、元Sourceを上書きしない。
- Sourceは`input`、`analysis`、`output`などの種別と、一次か派生かを表す属性を持つ。既存のAnalysis Bundleは移行期間の互換表現であり、新モデルでは対応するAnalysis Sourceとして扱う。
- 親Source ID、生成操作、生成日時、使用モデル、実際に使用したSource IDは派生Sourceの来歴として固定保存する。
- 正規IDは`sourceId`とする。既存の`analysisId`は読み取り互換用に`sourceId`との対応を保持し、新規書き込みの識別子として使わない。

## Group

Groupは、すぐに統合・出力しない関連Sourceを案件、テーマ、顧客、機能などの文脈で集める入れ物である。

- Groupへ追加しても新しいAnalysisやOutputを自動生成しない。
- 同じSourceを複数Groupから参照できる。
- Groupへ後からSourceを追加できる。
- AIは追加先Groupを提案できるが、ユーザーが確認・修正できる。
- Group全体から生成するときは、その時点で実際に使用した個別Source IDを派生Sourceへ記録する。
- Group–Source関係の正本はGroup Bundleの`group.json`であり、Group側の`sourceLinks`へSource ID、役割、追加日時を保存する。Source Manifestの`groupIds`は既存データの読み取り用cacheであり、新規更新しない。

## Analysis Source Revision

Analysis Sourceは詳細画面では同じSourceを継続して更新するように見せる。ただし、追加情報、AIの解釈、過去の要約を破壊的に上書きしない。

### 必須要件

- Analysis Sourceへ直接追加できる情報は、テキスト、画像、URLに限定する。
- 追加画像は、その時点で新規の不変Sourceとして保存し、RevisionからID参照する。元Analysis Sourceや既存の親Sourceへ画像を埋め込んで上書きしない。
- 音声、動画、PDFはRevisionへ直接追加しない。独自前処理を伴う新規Sourceとして取り込み、必要に応じて同じGroupへ追加してAnalysisを生成する。
- Revisionは連番、作成日時、作成理由、追加したSource ID、追加情報種別、ユーザー指示、使用Provider／モデル、確認状態、直前Revisionとの差分を持つ。
- 各Revisionはsummary本文の不変スナップショットを必ずファイルとして保存し、`summaryPath`とSHA-256を記録する。ハッシュだけ、または最新`summary.md`だけをRevisionの保存として扱わない。
- AIは現在のsummary、今回の追加情報、今回のユーザー指示を中心に再分析し、最新表示用の`summary.md`を更新する。根拠として使用した個別Source IDも固定保存する。
- 過去のsummaryと追加情報はRevision単位で保持する。最新の`summary.md`は最新Revisionの不変スナップショットから再生成するmaterialized viewであり、構造化Revision履歴から再生成できる。
- 詳細画面では、Revision回数、追加情報、各Revisionのsummary、直前との差分、根拠Sourceを確認できる。
- AI対話によってsummaryが変わる場合も、対話履歴を正本として保持したうえで、変更後のsummaryを新しいRevisionとして保存する。

### 受入条件

- テキスト、画像、URLを追加しても同じAnalysis Sourceを開け、過去のsummaryと追加情報を失わない。
- 任意のRevisionについて、理由、追加情報、根拠Source、使用モデル、差分を確認できる。
- 任意のRevisionについて、保存済みのsummary本文、`summaryPath`、SHA-256を確認できる。
- 最新`summary.md`を削除または再生成する場合でも、最新Revisionの保存済みsummaryから同じ内容を復元できる。

## 原本閲覧と追加可能範囲

Analysis Sourceの詳細画面から、根拠となる原本へ戻れることを必須とする。

- 画像原本はアプリ内でプレビューできる。
- 音声・動画原本はアプリ内で再生できる。
- 原本、追加画像、追加テキストの保存場所はFinderで表示できる。
- Revisionに追加した画像とテキストも、追加情報ごとに閲覧できる。
- URLは出典として追加できるが、URL自体から自動取得・自動解析はしない。

## Task

Taskは具体的な対応、期限、状態、待ち先、完了条件を管理する。

- Taskは複数Sourceを紐づけられる。
- Taskは複数Groupを紐づけられる。
- 同じSourceまたはGroupを複数Taskから参照できる。
- TaskへGroupを紐づけた場合、日常の整理画面ではGroupの最新状態を参照する。
- AI生成時はGroup参照だけを根拠として残さず、実際に使用した個別Source IDを固定して記録する。
- Task内のSourceには、根拠、文脈、入力、下書き、確定成果物などの役割を設定できる設計とする。
- Task–Source／Task–Group関係の正本はTask Bundleの`task.json`である。`sourceLinks`と`groupLinks`へ相手ID、role、追加日時を保存する。
- Source Manifestの`taskIds`は既存データの読み取り用cacheであり、新規更新しない。双方向のBundle更新を行わず、SQLiteは索引として再構築する。

## 画面内URLの出典化

Backlog、Jira、Confluence、Slack、メール等のキャプチャでは、ユーザーが可能な範囲でアドレスバーやURLを画面内へ含める。AI送信前にローカルVision OCRで画像内のURL文字列を検出し、URL領域をマスクした派生画像と安全化済みURLだけをClaudeへ渡す。原画像はローカルに不変のSourceとして残す。`provided-text`もローカルでURLを解析・安全化してから送る。

### 必須要件

- 画像のURL検出にはローカルVision OCRを使い、検出したURL領域をマスクした送信用派生画像を作る。派生画像の親Source IDとハッシュを記録する。
- `provided-text`とユーザー入力のURLは、ローカル解析で送信用に安全化する。
- URLごとに、抽出元Source ID、抽出方法（Vision OCR、提供テキスト、ユーザー入力）、`aiDisplayUrl`、ローカル遷移専用の`localOpenUrl`、確認状態を保持する。
- `aiDisplayUrl`はqueryとfragmentを除去したAI送信用・AI出力表示用URLとする。`localOpenUrl`はローカルVault内だけに保持し、既定ブラウザで開くためにだけ使う。`localOpenUrl`をClaude、AI出力、Markdown、ログへ渡さない。
- 複数Sourceから生成した場合は、どのURLがどのSourceに由来するか追跡できる。
- Source詳細から`localOpenUrl`をクリックして既定ブラウザで開ける。AI生成SourceとAI出力には`aiDisplayUrl`だけを表示する。
- URLを認識できない場合は推測・補完せず、「URL未検出」として扱う。
- OCR誤認の可能性があるURLはAI提案状態とし、ユーザー確認前に確定URLとして扱わない。
- ローカルOCR、URL解析、URL領域マスクのいずれかに失敗した場合は自動送信しない。対象Sourceを`needs_review`として失敗理由とともに表示する。

### 受入条件

- URLを含む画面キャプチャでは、原画像をローカルに残し、ClaudeへURL領域をマスクした派生画像だけを送る。
- AI出力には`aiDisplayUrl`だけが表示され、`localOpenUrl`のqueryやfragmentが現れない。
- アプリからURLを開く場合だけ`localOpenUrl`を使い、AI出力に表示するURLとの違いを詳細画面で明示する。
- ローカル前処理に失敗したSourceは自動送信されず、`needs_review`で確認できる。
- 派生Sourceでも出典URLと根拠Sourceへ逆引きできる。
- 存在しないURLをAIが生成しない。

## Analysis Sourceに紐づくAI対話

Analysis Sourceの詳細画面に、当該Sourceの内容を前提としてClaudeへ質問・補足指示を送る入力欄を設ける。一般のClaude画面を別途開かず、相談内容と結果をContextoryの来歴として残す。

### 必須要件

- Analysis Source詳細から、ユーザーが自由文で質問または修正指示を入力できる。
- Claudeへ渡す文脈は、対象Analysis Sourceとユーザーが明示的に選択した追加Source／Groupに限定する。
- ユーザー発言、AI回答、日時、使用モデル、根拠Source IDを追加専用の対話履歴として保存する。
- 各対話は`analysisSourceId`、基準Revision ID、送信時の`summaryPath`とSHA-256、Group展開後のSource ID一覧、使用したSource／派生物のIDとハッシュ、モデル、prompt schema version、送信日時、結果状態をローカル監査記録として固定保存する。
- 選択したGroupは送信時に個別Source IDへ展開し、そのsnapshotだけを対話の文脈とする。後日のGroup変更やRevision追加で過去対話の送信文脈を変えない。
- 対話、再分析、Group展開のいずれでも[ADR-008](../06-adr/ADR-008-local-media-preprocessing.md)を適用する。音声・動画の原本をClaudeへ渡さず、前処理済み文字起こし・代表フレームだけを使用する。未前処理または前処理失敗のメディアを含む場合はfail-closedで送信せず、対象を表示する。
- Claude実行時はSource Bundle全体を作業ディレクトリにしない。選択済み送信対象だけを置く一時staging directoryを作り、通常画像、テキスト、安全化済みURL、文字起こし、代表フレームを配置する。URLを含む画像にはURL領域をマスクした派生画像を使う。音声・動画原本と`localOpenUrl`は配置しない。
- staging directoryは`jobId`と`createdAt`を持ち、処理完了、失敗、タイムアウト、中断のいずれでも削除する。次回起動時には実行中Jobへ属さない残存stagingも削除する。これは送信文脈の限定、Claudeの誤読と不要なトークン消費の防止を目的とする。
- AI回答を既存Source本文へ破壊的に上書きしない。
- 対話内容は構造化した履歴を正本とし、summaryに反映する場合は更新後のsummaryを新しいRevisionとして保存する。
- `summary.md`だけを正本にせず、構造化した対話履歴とRevision履歴から再生成できるようにする。
- 重要な回答は、新しい派生Sourceとして確定・保存できる。
- AI回答はユーザー確認前には`proposed`とし、確認、修正、却下を区別する。

### 受入条件

- Analysis Source詳細から質問し、対象Sourceを踏まえた回答が得られる。
- アプリ再起動後も質問と回答が残る。
- 対話により更新された`summary.md`と対応するRevisionを確認できる。
- 対話から新しいSourceを作成した場合、元Analysis Sourceへ逆引きできる。
- 別Sourceの対話履歴と混在しない。
- 過去の対話について、送信時のRevision summary、Group展開結果、各入力のハッシュ、モデル、prompt schema version、結果状態を再確認できる。
- 送信対象外のSource、音声・動画原本、`localOpenUrl`がstaging directoryに含まれず、処理後にstaging directoryが残らない。

## 初期実装順

1. Source統一・legacy移行: 既存Analysisへ正規`sourceId`とRevision 1の不変summary snapshotを割り当て、対応表、ハッシュ、Bundle走査による再構築を検証する。検証後にのみRevisionの新規書き込みを開始する。
2. Group: Group作成、名称変更、Source追加・除外、複数Group所属、Group–Source来歴を実装する。
3. Revision: Analysis Sourceへのテキスト・画像・URL追加、URL安全化、Revision履歴、summary再生成、原本／追加情報の閲覧を実装する。
4. AI対話: 選択済みSource／Groupだけの一時staging、監査履歴、Revisionを介した`summary.md`反映を実装する。
5. Task関連: TaskとSource／Groupの多対多関連、`task.json`のlink正本、逆引き索引を実装する。
6. 派生Source生成: Source単体・複数Source・Groupから派生Sourceを生成し、後続生成へ再利用する。
