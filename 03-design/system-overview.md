# システム概要

## 暫定構成

```text
Capture / Audio Recording / Screen Recording
                    ↓
          Local Source Store / Group
                    ↓
 Local OCR・URL安全化 / media preprocessing
                    ↓
  Claude Code analysis / Analysis Revision
                    ↓
 Source-derived summary / provenance URL
                    ↓
 Project / Task candidate grouping
                    ↓
       Review data pending confirmation
                    ↓
        Confirmed Project / Task context
```

## 境界

- Resident Capture Agent: macOSメニューバー、グローバルショートカット、キャプチャ、音声録音、画面録画、開始・停止、取得・処理状態。
- Local Vault: 原本、Markdown、派生物の保存。
- Processing Pipeline: 文字起こし、要約、Markdown生成、関連判定。
- Media Preprocessor: PCM変換、ローカルWhisper文字起こし、AVFoundationによる動画代表フレーム抽出。成果物がない場合はAI解析へ進めない。
- URL Sanitizer: 画像のローカルVision OCR、URL領域をマスクした派生画像、`provided-text`のローカルURL解析を行う。URLを含む原画像はマスク済み派生画像に置き換え、`localOpenUrl`をClaudeへ渡さない。
- AI Adapter: 会社契約のClaude Codeを、ユーザーがSource単位で確認した業務情報の許可済み処理境界とする。個人Claudeへは送信しない。
- AI Staging: Source Bundleから選択済みの通常画像・テキスト、安全化済みURL、文字起こし、代表フレームだけを一時ディレクトリへ複製し、Claude処理後に削除する。URLを含む画像はマスク済み派生画像を使い、音声・動画原本と`localOpenUrl`は配置しない。
- Source Lineage: Input、Analysis、Outputを同じSourceとして保存し、親Sourceと実際に使用した個別Source IDを固定する来歴管理。
- Analysis Revision: 同じAnalysis Sourceへのテキスト・画像・URL追加、再分析、summary差分、過去summaryを追加専用で管理する境界。
- AI Invocation Audit: 再分析・対話・Group展開の送信時点のRevision、summary、Source／派生物ハッシュ、Group展開結果、モデル、prompt schema、結果状態をローカルで固定記録する。
- Recording Reminder: Slack／Teamsの前面化をローカルで検知し、録音開始をユーザーへ確認する。会議・マイク・画面内容の精密検知や自動録音は行わない。
- Grouping: Project／Task候補、関連根拠、確信度の生成。
- Library／Review Interface: 一覧、検索、詳細確認、グルーピング修正、日次レビュー。別途設計する。
- Docs: プロダクト仕様と意思決定の正本。

## 技術スタック

- Language: Swift
- UI: SwiftUI
- Resident UI: `MenuBarExtra`
- Capture / Recording: ScreenCaptureKitとmacOS公式フレームワーク
- Local Storage: Source BundleとSQLite Index
- Claude integration: ローカルのClaude Codeを非対話実行
- Runtime: macOSローカルプロセスのみ
- Speech Runtime: Releaseアプリ同梱のarm64 `whisper-cli`。モデルはApplication Supportへ別配置する。
- Preferred Build: 私用Mac
- Delivery: ビルド済み`.app`を会社Macへ直接提供
- Preferred Company Mac: アプリ実行とローカルVault保存のみ
- Fallback Company Mac: Xcodeを導入し、ソースをcloneして同一構成でローカルビルド

詳細と判断理由は[ADR-003](../06-adr/ADR-003-native-single-user-stack.md)を参照する。
ビルドと会社Macへの提供は[ADR-004](../06-adr/ADR-004-private-build-binary-delivery.md)を参照する。

## Local Vault

Local VaultはSource BundleとSQLite Indexから構成する。

```text
Contextory Vault/
├── sources/
│   └── YYYY/
│       └── MM/
│           └── <source-id>/
│               ├── manifest.json
│               ├── source.<ext>
│               ├── provided-text.md
│               ├── user-notes.md
│               ├── transcript.md
│               ├── preview.jpg
│               ├── frames/
│               └── derived/
│                   ├── sanitized/  # URLマスク済み送信専用派生物
│                   └── analysis/
│                       ├── revisions/
│                       │   └── <revision-id>/
│                       │       ├── revision.json
│                       │       └── summary.md  # 不変snapshot
│                       ├── conversations/
│                       ├── audits/
│                       └── summary.md          # 最新Revisionからのmaterialized view
├── contexts/             # 既存Context Bundleの読み取り互換専用。新規書き込みしない。
├── analyses/             # 既存Analysis Bundleの読み取り互換専用。新規書き込みしない。
├── groups/
│   └── <group-id>/
│       └── group.json
├── tasks/
│   └── <task-id>/
│       └── task.json
├── exports/
└── index/
    └── contextory.sqlite3
```

- Source Bundleは正規`sourceId`単位の永続的な記録である。新規のInput、Analysis、Outputは正規Sourceモデルへ書き込む。
- 原本、文字起こし、プレビュー、動画キーフレームをSource Bundleへまとめる。
- 外部サービスから取り込んだ原文とユーザー補足は、AI生成物および原本と分離する。
- Source BundleをProject／Taskフォルダへ物理移動しない。複数Project／Taskとの関連はメタデータで表現する。
- GroupはSource IDだけを参照し、同じSourceを複数Groupへ再利用できる。Groupへの追加は生成を起動しない。
- Analysisは`kind: analysis`の派生Sourceである。既存Contextと`analyses/`は移行期間の読み取り互換表現とし、新規書き込み先ではない。
- 同じAnalysis Sourceへ追加して再分析する場合、Revision Bundleを追加する。各Revisionは追加情報、理由、使用モデル、確認状態、差分、実際に使用したSource IDを持つ。
- `summary.md`は最新Revisionの閲覧用投影であり、Revisionと対話履歴から再生成できる。
- 各Revisionは`summaryPath`とSHA-256を伴うsummary本文の不変snapshotを持つ。snapshotがないRevisionは有効なRevisionとして扱わない。
- Task–Source／Task–Groupは各Task Bundleの`task.json`、Group–Sourceは各Group Bundleの`group.json`を唯一の正本とする。Source Manifestの逆方向ID配列とSQLiteはlegacy cacheまたは再構築可能な索引である。
- `manifest.json`は機械可読な永続メタデータを持つ。
- MarkdownはユーザーとClaude Codeが確認・再利用できる成果物とする。
- SQLiteは検索、関連、処理状態、再試行、レビューキュー用のローカル索引とする。
- SQLiteの永続情報は可能な限りSource Bundleから再構築可能にする。
- 実行待ち、処理中、ロック、再試行回数などの一時的な運用状態はSQLiteのみで保持できる。
- Claude実行用のstaging directoryはLocal VaultのBundle外にジョブ単位で作る一時領域であり、`jobId`と`createdAt`を持つ。永続化・バックアップ・Git管理しない。処理完了、失敗、タイムアウト、中断後に削除し、次回起動時には実行中Jobへ属さない残存stagingも削除する。

詳細と判断理由は[ADR-001](../06-adr/ADR-001-local-vault-storage.md)を参照する。

## 常駐型取得エージェント

- 初期対象OSはmacOSのみとする。Windows固有実装をMVPへ含めない。
- 通常時はmacOSメニューバーに収まり、常設ウィンドウを表示しない。
- キャプチャ、音声録音、画面録画を1操作またはグローバルショートカットで開始する。
- 録音・録画中は取得中であること、経過時間、停止操作を明示する。
- 保存中、AI処理中、完了、失敗を小さな状態表示またはOS通知で示す。
- 誤操作に備え、直近の取得を破棄できるようにする。
- メニューバーに確定済みSource数とLocal Vault使用量を表示する。
- 直近の取得の破棄は2段階確認後にSource BundleをmacOSのゴミ箱へ移動し、即時完全削除しない。
- タスク整理画面ではAnalysis単体、未参照Source単体、Analysisと未参照Source一式をゴミ箱へ移動できる。Taskまたは別Analysisから参照中の場合は削除を禁止する。
- 音声モデルは`Application Support/Contextory/Models`へ置き、Local Vault、Git、アプリ更新から分離する。
- 保存完了通知の権限は自動要求せず、ユーザーの明示操作で有効化する。
- 常駐メニューにはInput操作と処理状態だけを表示する。
- 録音確認は初期状態で無効とし、ユーザーが有効化した対象アプリだけを検知する。状態はアプリ別の`nextEligibleAt`と`suppressionReason`へ統一する。
- 判定は、録音中、当日抑制、15分snooze、60分cooldown、表示可能の順に評価する。`nextEligibleAt`到達時は`suppressionReason`を自動解除し、`today_suppressed`も翌日のローカル日付開始時刻に解除する。Slack／Teamsが20秒継続して前面で、表示可能な場合だけパネルを出す。
- 操作は録音開始、15分後に通知、今回は通知しない（60分抑制）、今日は通知しないとする。対象アプリ、20秒閾値、60分cooldownは設定で変更できる。
- 録音確認の検知・設定・cooldownはローカルに限定し、前面アプリの検知だけを初期対象とする。
- 補足、関連付け、手動再解析、結果詳細確認を常駐メニューへ置かない。
- 権限、Claudeモデル、診断、Local Vault表示は「設定・診断」サブメニューへ集約する。

## AI処理とフォールバック

- ユーザーが明示的に取得・取り込みしたSourceは保存後にClaude Code分析を自動開始する。
- Input取得状態とAI解析状態を分離し、Claude解析中も新しいInputを保存してQueueへ追加できるようにする。
- Claude Code処理は同時実行せず1件ずつ直列処理し、待機件数を常駐メニューへ表示する。
- `pending_analysis`と`analyzing`のSourceは起動時にSQLite／Manifestから古い順でQueueへ復元する。
- Claude Codeプロセスの上限時間は5分とし、超過時は終了要求後に必要なら強制終了する。
- 使用上限、認証切れ、タイムアウトは`retry_waiting`、不正出力と一般実行失敗は`analysis_failed`として記録する。
- 失敗Sourceは現在のQueueから隔離して後続を処理し、`retry_waiting`は自動再試行しない。
- Source Bundle全体をClaude実行の作業ディレクトリにしない。選択済みの画像、テキスト、安全化済みURL、文字起こし、代表フレームだけをジョブ単位の一時staging directoryへ配置し、Claude Codeのツールを`Read`だけに限定して、セッションを保存せず構造化JSONを受け取る。
- staging directoryに音声・動画原本と`localOpenUrl`を置かない。URLを含む画像はマスク済み派生画像、提供テキストはquery／fragmentを除去した安全化版を置く。処理完了、失敗、タイムアウト、中断後に削除し、次回起動時は実行中Jobに属さない残存stagingを削除する。
- 会社契約のClaude Codeへは明示取得・取り込みした業務情報を送信できるが、個人Claudeへは送信しない。パスワード入力画面、APIキー、Cookie、セッショントークンを表示する管理画面はユーザーが明示取得しない運用とし、万能な認証情報検出・マスク・fail-closedは実装しない。
- 画像は送信前にローカルVision OCRでURLを検出し、URL領域をマスクした派生画像だけを渡す。`provided-text`はローカルでURL解析・安全化し、queryとfragmentを除去した`aiDisplayUrl`だけを渡す。前処理に失敗した場合は`needs_review`として自動送信しない。
- 初回`summary.md`は固有Analysis SourceのRevisionとして原子的に保存し、ユーザー確認前は`proposed`として扱う。最新表示用`summary.md`はRevision履歴から再生成できる。
- Groupから生成する際は、Group IDだけでなく実際に使用した個別Source IDを固定し、原本を移動せずClaude Codeへ渡す。
- Analysis Sourceへの再分析では、現在のsummary、今回のテキスト・画像・URL追加、ユーザー指示を中心に渡す。音声、動画、PDFの追加は新規Source取り込みと前処理を経由する。
- Analysis SourceのAI対話では、対象Sourceとユーザーが明示選択した追加Source／Groupだけを文脈にする。Groupは送信時に個別Source IDへ展開してsnapshot保存し、対話とsummary更新は追加専用で保存する。
- AI対話、再分析、Group展開を含むすべてのClaude送信でADR-008を適用し、音声・動画原本ではなく前処理済み文字起こし・代表フレームだけを渡す。未前処理または前処理失敗のメディアが選択範囲にある場合はfail-closedで送信を止め、対象を表示する。
- 各工程をProcessing Jobとして分離し、途中失敗から再開できるようにする。
- 認証切れ、タイムアウト、不正な出力、アプリ終了などを失敗状態として残す。
- 処理に失敗してもSource Bundleを保持し、ユーザーが手動で再実行できるようにする。
- 完全自動化のために安全性や復旧性を犠牲にしない。
- Claude CodeのモデルはSonnetを既定とし、ユーザーがOpusへ切り替えられるようにする。
- 選択モデルはローカル設定として保持し、Claude Code起動時の`--model`へ明示的に渡す。
- AI生成物には使用Provider、モデル、実行日時、根拠Sourceを記録する。
- AI生成物には解析目的とタスク分類候補（主分類、タグ、確信度、理由）も記録する。
- 内容に応じた自動モデル選択は、手動選択とルーティング理由を確認できる設計を整えてから追加する。
- Microsoft 365 CopilotはローカルCLIと同等に扱えず会社テナント側の許可が必要なため、初期AI Adapterには含めない。

## グルーピング

- 自動グルーピングはProject／Taskの候補、根拠、確信度として保存する。
- AI候補を確認済み所属として扱わない。
- ユーザーは候補の承認、変更、新規グループ作成、関連解除を行える。
- ユーザーが修正した確定結果を、次回以降の候補生成に利用できるよう履歴を保持する。

## Library／Review Interface

- 未確認のAI結果をReview Queueに集約する。
- 新規Project／Task候補、既存グループへの追加候補、低確信度、矛盾を優先表示する。
- ユーザーは確認、修正、保留、却下を行う。
- 確定前の内容を確認済み知識や外部出力へ利用しない。
- Sourceへの補足、複数Sourceの関連付け、解析目的を変えた再解析、解析結果・分類の詳細確認を担う。
- Input操作は提供せず、常駐型Inputエージェントと責務を明確に分離する。
- 最初の縦切りではAnalysis一覧・Markdown詳細・根拠Source表示・Task作成・Task来歴・再試行待ちの手動再実行を提供する。
- Task詳細から根拠Analysis、Source、親Taskへ逆引きできる。
- GitHub風の来歴グラフは後続UIとし、先にグラフ描画可能なID参照を欠損なく保存する。
- 一覧管理・詳細レビューのUI形態は、常駐型取得エージェントのMVPとは分けて決定する。

## 未確定事項

- デスクトップ技術スタック。
- ローカルVaultの正本形式。
- 検索インデックスの方式。
- 録音・文字起こし方式。
- macOSメニューバーと任意の小型フローティングコントロールの使い分け。
- グローバルショートカットの初期値と競合解決。
- 自動グルーピングの類似度、時間、人物、アプリ情報の重み付け。
- 日次レビューの通知方法と締め時間。
