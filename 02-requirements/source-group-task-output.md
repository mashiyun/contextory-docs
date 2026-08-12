# Source・Group・Task・Output要件

## Status

次回実装候補。2026-08-12時点では仕様整理のみで、未実装。

## Source中心モデル

Contextoryで保存する情報と成果物は、入力かAI生成かを問わず同じ大きさのSourceとして扱う。

- キャプチャ、録音、録画、PDF、手動入力は一次Sourceである。
- AI Analysis、議事録、要件定義、返信案などの生成結果もSourceである。
- Source単体または複数Sourceを材料として、新しい派生Sourceを生成できる。
- 派生Sourceをさらに別の生成へ再利用できる。
- 生成時に使用した親Source IDを固定して記録し、元Sourceを上書きしない。
- `analysis`や`output`は独立した大きさの異なるデータではなく、Sourceの種別・属性として表現する方向へ段階的に移行する。

## Group

Groupは、すぐに統合・出力しない関連Sourceを案件、テーマ、顧客、機能などの文脈で集める入れ物である。

- Groupへ追加しても新しいAnalysisやOutputを自動生成しない。
- 同じSourceを複数Groupから参照できる。
- Groupへ後からSourceを追加できる。
- AIは追加先Groupを提案できるが、ユーザーが確認・修正できる。
- Group全体から生成するときは、その時点で実際に使用した個別Source IDを派生Sourceへ記録する。

## Task

Taskは具体的な対応、期限、状態、待ち先、完了条件を管理する。

- Taskは複数Sourceを紐づけられる。
- Taskは複数Groupを紐づけられる。
- 同じSourceまたはGroupを複数Taskから参照できる。
- TaskへGroupを紐づけた場合、日常の整理画面ではGroupの最新状態を参照する。
- AI生成時はGroup参照だけを根拠として残さず、実際に使用した個別Source IDを固定して記録する。
- Task内のSourceには、根拠、文脈、入力、下書き、確定成果物などの役割を設定できる設計とする。

## 画面内URLの出典化

Backlog、Jira、Confluence、Slack、メール等のキャプチャでは、ユーザーが可能な範囲でアドレスバーやURLを画面内へ含める。AI処理は画面内で確認できたURLをSourceの出典候補として抽出する。

### 必須要件

- 画面内または同時入力テキストに存在するURLを抽出し、AI生成Sourceの本文とメタデータへ出典として表示する。
- URLごとに、抽出元Source IDと抽出方法（画像、提供テキスト、ユーザー入力）を保持する。
- 複数Sourceから生成した場合は、どのURLがどのSourceに由来するか追跡できる。
- Source詳細および生成結果からURLをクリックして既定ブラウザで開ける。
- URLを認識できない場合は推測・補完せず、「URL未検出」として扱う。
- OCR誤認の可能性があるURLはAI提案状態とし、ユーザー確認前に確定URLとして扱わない。
- URLのqueryやfragmentに認証token、署名、セッションID等の機密値が含まれる可能性がある場合は、安全化した表示URLとローカル原文を分離する。外部AIへ機密値を送信しない。

### 受入条件

- URLを含む画面キャプチャのAnalysis Sourceに、確認可能な出典URLが表示される。
- URLをクリックすると対象ページを開ける。
- 派生Sourceでも出典URLと根拠Sourceへ逆引きできる。
- 存在しないURLをAIが生成しない。

## Analysis Sourceに紐づくAI対話

Analysis Sourceの詳細画面に、当該Sourceの内容を前提としてClaudeへ質問・補足指示を送る入力欄を設ける。一般のClaude画面を別途開かず、相談内容と結果をContextoryの来歴として残す。

### 必須要件

- Analysis Source詳細から、ユーザーが自由文で質問または修正指示を入力できる。
- Claudeへ渡す文脈は、対象Analysis Sourceとユーザーが明示的に選択した追加Source／Groupに限定する。
- ユーザー発言、AI回答、日時、使用モデル、根拠Source IDを追加専用の対話履歴として保存する。
- AI回答を既存Source本文へ破壊的に上書きしない。
- 対話内容は対象Sourceの閲覧用`summary.md`へ時系列の追記セクションとして反映し、詳細画面を再表示しても確認できる。
- `summary.md`だけを正本にせず、構造化した対話履歴から再生成できるようにする。
- 重要な回答は、新しい派生Sourceとして確定・保存できる。
- AI回答はユーザー確認前には`proposed`とし、確認、修正、却下を区別する。

### 受入条件

- Analysis Source詳細から質問し、対象Sourceを踏まえた回答が得られる。
- アプリ再起動後も質問と回答が残る。
- `summary.md`に対話履歴が反映される。
- 対話から新しいSourceを作成した場合、元Analysis Sourceへ逆引きできる。
- 別Sourceの対話履歴と混在しない。

## 初期実装順

1. Groupの作成、名称変更、Source追加・除外、複数Group所属。
2. Source詳細への出典URL表示、確認、編集、リンク遷移。
3. TaskとSource／Groupの多対多関連。
4. Analysis Source詳細のAI対話と履歴保存、`summary.md`反映。
5. Source単体・複数Source・Groupから派生Sourceを生成。
6. Analysis／OutputをSource属性へ統一する既存データ移行。
