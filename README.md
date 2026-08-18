# Contextory ドキュメント

Contextoryは、PMが見た画面と聞いた会話をローカルに収集し、AIの整理とユーザー確認を通して、案件・タスク単位の信頼できる業務文脈へ育てるデスクトップアプリです。

自分だけが使う非公開ツールとして開発し、一般公開、App Store配布、複数ユーザー利用は想定しません。

通常は私用Macでビルドし、会社Macにはビルド済みアプリバイナリを直接提供して実行します。直接配布がGatekeeper、署名、互換性などで安定しない場合は、会社MacへXcodeを導入してローカルビルドするフォールバックを使用します。

## 現在の方針

- 画面キャプチャ、音声録音、画面録画によるInput収集を簡単にする。
- 初期対象OSはmacOSのみに限定する。
- macOSメニューバーに常駐し、画面を邪魔せず、すぐ開始・停止できるようにする。
- 取得したSourceを即時にAIへ渡し、要約とMarkdownを生成する。
- ユーザーが明示的に取得・取り込みしたSourceは保存後に自動処理し、失敗を状態表示する。
- 過去のSourceとの関連を調べ、案件・タスク候補へ自動グルーピングする。
- 自動グルーピングが誤っている場合はユーザーが修正できるようにする。
- 案件が文脈を持ち、タスクが対応を持ち、Sourceが根拠になる。
- 原本、AIによる解釈、ユーザー確認済み情報を分離する。
- Input、Analysis、Outputを正規`sourceId`を持つSourceとして再利用し、Analysis更新は不変Revisionとして追加する。
- Claude送信前にURLとメディアをローカル前処理し、送信時点の文脈とハッシュをローカル監査記録へ固定する。
- Claude実行には選択済み送信対象だけを置く一時staging directoryを使い、会社契約Claudeを業務情報の許可済み処理境界とする。URLのquery／fragmentは安全化し、機密入力画面は取得しない運用とする。
- 画像・PDF・テキストを手動取り込みし、AIのタスク分類候補を確認可能な状態で保存する。
- 原本はローカルに保存し、Claude Codeへ即時送信する対象と範囲はユーザーが事前に許可する。
- ユーザーが最低1日1回、要約とグルーピングを確認・修正して確定する。
- 撮り溜めたコンテンツの一覧管理・詳細レビューUIは、常駐型取得エージェントとは別に設計する。
- 常駐メニューはInput専用、将来のタスク整理画面は確認・補足・関連付け・再解析専用とし、相互の操作を混在させない。
- 将来のExcel、PowerPoint、Jira、Backlog、Slack出力は確認済み文脈から生成する。
- Analysis一覧は、内容が分かる具体的な要約とJST日時だけを表示する。要約は保存値を変えず、20 Character超過時だけ表示を19 Characterと`…`へ短縮する。Analysis表記、分類、状態、hash、Source IDは詳細・診断画面で確認する。保存時刻はUTCのISO 8601を正本とし、表示時だけAsia/Tokyoへ変換する（表示例`2026/08/14 10:30`）。
- Analysis一覧で項目を選択した直後は、タイトルと「あなたの対応」（Actions）だけを軽量に表示する。Markdown、Revision、追加情報、根拠Source、媒体・文字起こしなどの全詳細は「詳細を表示」を押した対象だけへ非同期で遅延読込し、Inputと一覧操作を待たせない（v0.3.4で実装・検証中）。
- 解析の成否はcanonical Analysis SourceとRevisionの不変Summary snapshotの保存・hash検証で判定する。`operationId`はJob作成時に確定して一意制約付きで索引化し、Analysis保存・Summary保存・親Manifest更新の所有者を1つに集約する。保存後の状態同期失敗は`completion_sync_pending`として起動時復旧の対象とし、試行回数を実行前に永続化して最大3回後は`completion_sync_failed`で手動対応へ切り替える。部分保存・破損は`analysis_integrity_failed`として自動再解析しない（将来実装）。
- Whisperの生Transcriptはrole別に不変の原本として保持し、ユーザー訂正は不変の訂正Sourceとして追加する。訂正版Transcriptから要約とActionsを再生成し、過去Revisionを保持する。同一操作の収束は`operationId`、監査は`requestFingerprint`で分けて扱う（将来実装）。
- 共通辞書とGroup／案件別辞書をローカルに保持し、次回録音の用語ヒントと文字起こし後の決定的な表記補正へ使う。同時適用は共通辞書と明示選択した1つのGroup辞書までとし、適用内容を`dictionaryRevisionRefs`として固定する。辞書登録はユーザー確認を必須とし、Whisperモデル自体の学習・fine-tuningは行わない（将来実装）。
- Source削除は、実利用中の参照確認と1回の明示確認を通してmacOSのゴミ箱へ移動する。未参照削除候補は検証済みcanonical `kind: input` Sourceに限り、Analysisを含む派生・legacy・種別不明Sourceを含めない。削除復旧は再索引・一覧走査より先に有効recordを完了させ、不正・非Job・競合recordは削除・Source解釈せず隔離する。隔離中は削除を停止するが、有効なAnalysisの閲覧・索引化は継続する。
- 外部Output公開は、承認時に固定したMarkdown派生SourceをJira、Confluence、Backlog Adapterへ変換する将来設計とする。送信識別子・結果・添付状態を保存し、結果不明の自動再作成は行わない。資格情報はアプリ内部でmacOS Keychainからのみ解決し、設定、Vault、URL、プロセス、ログ、診断、Gitへ出さない。
- 外部チケット取り込みは公開と別の将来設計とし、Jira／Backlogチケットを不変External Ticket Sourceとして保存してから、ユーザー確認後にTaskまたはGroupへ関連付ける。Read Adapterは外部ticketを変更せず、APIなしの手動取り込みでも利用できる。

## 文書一覧

- [プロダクトビジョン](01-vision/product-vision.md)
- [MVPスコープ](02-requirements/mvp-scope.md)
- [Source・Group・Task・Output要件](02-requirements/source-group-task-output.md)
- [Topic Source・Task・WBS・PM支援要件](02-requirements/topic-source-task-wbs.md)
- [External Ticket Source要件](02-requirements/external-ticket-source.md)
- [録音忘れ防止要件](02-requirements/recording-reminder.md)
- [録音入力選択・無音警告要件](02-requirements/recording-input-monitoring.md)
- [Transcript訂正・用語辞書要件](02-requirements/transcript-correction-terminology.md)
- [安全・プライバシー原則](02-requirements/safety-principles.md)
- [システム概要](03-design/system-overview.md)
- [データモデル](03-design/data-model.md)
- [開発フェーズ](04-roadmap/development-phases.md)
- [未決事項](05-decisions/open-questions.md)
- [ADR一覧](06-adr/README.md)
- [ADR-001 Local VaultのファイルとSQLite構成](06-adr/ADR-001-local-vault-storage.md)
- [ADR-002 初期対象OSをmacOSに限定](06-adr/ADR-002-macos-first.md)
- [ADR-003 macOSネイティブ・単一ユーザー構成](06-adr/ADR-003-native-single-user-stack.md)
- [ADR-004 私用Macでビルドして会社Macへバイナリ提供](06-adr/ADR-004-private-build-binary-delivery.md)
- [ADR-005 システム音声とマイク音声を別原本として保存](06-adr/ADR-005-separate-system-and-microphone-audio.md)
- [ADR-006 Source・Context・Analysisを分離し派生結果を追加保存（legacy）](06-adr/ADR-006-append-only-context-analysis.md)
- [ADR-007 Taskの根拠来歴をID参照で保持](06-adr/ADR-007-task-output-lineage.md)
- [ADR-008 音声・動画をローカル前処理してからAIへ渡す](06-adr/ADR-008-local-media-preprocessing.md)
- [ADR-009 Source統一モデルとAnalysis Revisionを正規モデルにする](06-adr/ADR-009-analysis-source-revisions.md)
- [ADR-010 前面アプリ検知によるローカル録音確認を採用](06-adr/ADR-010-local-foreground-recording-reminder.md)
- [ADR-011 Source／Group／Task関係の正本を単一Bundleへ限定する](06-adr/ADR-011-bundle-relationship-ownership.md)
- [ADR-012 Claude実行には最小一時staging directoryを使用する](06-adr/ADR-012-minimal-claude-staging.md)
- [ADR-013 外部Output公開は承認済みMarkdownとAdapterを介して行う](06-adr/ADR-013-approved-external-publication.md)
- [ADR-014 Transcript訂正を不変Sourceとし、用語辞書で決定的に補正する](06-adr/ADR-014-transcript-correction-terminology.md)
- [ADR-015 Topic SourceとTask／WBSを既存正本モデルへ追加する](06-adr/ADR-015-topic-source-task-wbs.md)
- [ADR-016 外部チケットは不変External Ticket Sourceとして取り込む](06-adr/ADR-016-external-ticket-source.md)
- [PoC一覧](07-poc/README.md)
- [Phase 0 配布・権限スパイク結果](07-poc/phase-0-distribution-permissions-result.md)

## リポジトリ境界

- Docs: `/Users/mikey/workspace/vault/personal-vault/20-projects/contextory/docs`
- App: `/Users/mikey/workspace/apps/contextory-app`
- 実業務データ: 会社PCのローカルアプリデータ領域。Git管理しない。

## 現在の状態

Phase 0の配布・権限スパイクとPhase 1のInput Captureは完了しました。Phase 2の自動解析Queueと、Inputと分離したタスク整理画面の最初の縦切り（Analysis確認、根拠Source、Task作成、Task来歴、失敗解析の手動再実行）を実装しました。正規Sourceモデルとlegacy Analysis移行の基盤（canonical `sourceId`、Revision 1 snapshot、Bundle再索引、fail-closed gate、staging復旧）、約10〜20 CharacterのAnalysis一覧要約、Task／Group archiveは実装済みです。v0.3.1の未参照Source一覧と複数選択破棄は安全契約違反が判明しており、v0.3.2で候補の限定、削除復旧の起動順、一覧見出しを修正・検証するまで使用しません。この開発サイクルの最終全検証は別途実施します。Group整理、Revision追加UI、URL安全化、原本閲覧、監査可能なAnalysis Source対話、Actions、録音入力選択／無音警告、外部Output公開は未実装の将来仕様です。既存`analyses/`／`contexts/`は読み取り互換として残します。

実利用のフィードバックから、Analysis一覧のJST・簡素化は実装済みです。解析成功後の状態競合修正、Transcript訂正とRevision再生成、共通／Group辞書、決定的補正、PoC後のWhisperヒントは未実装であり、実装順は[開発フェーズ](04-roadmap/development-phases.md)に記載します。
