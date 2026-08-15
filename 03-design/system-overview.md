# システム概要

## 暫定構成

```text
Capture / Audio Recording / Screen Recording
                    ↓
          Local Source Store / Group
                    ↓
       URL安全化 / media preprocessing
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
                    ↓
 Markdown derived Output Source → user approval → External Output Adapter
```

## 境界

- Resident Capture Agent: macOSメニューバー、グローバルショートカット、キャプチャ、音声録音、画面録画、開始・停止、取得・処理状態。
- Local Vault: 原本、Markdown、派生物の保存。
- Processing Pipeline: 文字起こし、要約、Markdown生成、関連判定。
- Media Preprocessor: PCM変換、ローカルWhisper文字起こし、AVFoundationによる動画代表フレーム抽出。成果物がない場合はAI解析へ進めない。生Transcriptは不変の原本として保持する。
- Transcript Correction: 生Transcriptを上書きせず、ユーザー訂正を不変の訂正Sourceとして追加し、訂正版Transcript snapshotをAnalysis Revisionへ保存する境界。
- Terminology Dictionary: 共通辞書とGroup／案件別辞書をローカル保持し、次回録音の用語ヒントと、文字起こし後の決定的な表記補正へ使う境界。辞書登録はユーザー確認を必須とし、Whisperモデル自体の学習・fine-tuningは行わない。MVPで同時適用できるのは共通辞書とユーザーが明示選択した1つのGroup辞書までとし、適用内容を`dictionaryRevisionRefs`として固定する。
- URL Sanitizer: 画像、`provided-text`、ユーザー入力から抽出して保存・表示・送信するURLのquery／fragmentを除去する。画像からのURL抽出は任意で送信の前提にせず、`localOpenUrl`をClaudeへ渡さない。
- AI Adapter: 会社契約のClaude Codeを、ユーザーがSource単位で確認した業務情報の許可済み処理境界とする。個人Claudeへは送信しない。
- AI Staging: 初回解析、Revision再分析、AI対話、Group展開の全実行で、選択済みの未マスク原画像・テキスト・PDF・安全化済みURL・固定済み文字起こし・代表フレームだけをSource Bundle外の一時ディレクトリへ複製し、Claudeのcwdにする。Source Bundleをcwdまたは`--add-dir`として公開せず、音声・動画原本と`localOpenUrl`は配置しない。
- Source Lineage: Input、Analysis、Output、Topic Sourceを同じSourceとして保存し、親Sourceと実際に使用した個別Source IDを固定する来歴管理。Topic SourceのEvidence Spanは不変snapshot、snapshotと選択byte列のhash、role、整数millisecond／UTF-8 byte半開区間を固定し、全親参照を含むSource来歴DAGを`VaultMutationLock`内で検証する。
- External Ticket Import: Jira／BacklogのRead Adapterまたは手動入力をSource Manifest version 4、`kind: input`、`type: external_ticket_snapshot`へ正規化する境界。検証済みの不変remote ID、endpoint identity、operation ID、完全取得scope、snapshot系列、重複、差分を検証する。API pageは一時stagingへ置き、完全性確認後だけ`VaultMutationLock`内でSourceを確定する。外部ticketを変更せずTaskを自動作成・更新しない。
- Analysis Revision: 同じAnalysis Sourceへのテキスト・画像・URL追加、再分析、summary差分、過去summaryを追加専用で管理する境界。
- AI Invocation Audit: 初回解析を含む各Revisionの`stagedInputRefs`へ送信時点の入力所有Source、Revision、`contentRole`、`inputType`、元path、staging論理path、staged bytesと原ファイルのhash、MIME `contentType`、transformation／versionを固定し、Group展開結果、モデル、prompt schema、結果状態と結び付ける。Revisionを作らないAI対話／Group展開にも同じ配列を追加専用のauditとして残し、一時stagingは正本にしない。
- Recording Reminder: Slack／Teamsの前面化をローカルで検知し、録音開始をユーザーへ確認する。会議・マイク・画面内容の精密検知や自動録音は行わない。
- Recording Input Monitor: 選択中マイク名と開始前／録音中の入力レベルを表示し、マイク音声とシステム音声を別々に監視する。無音・切断候補は警告のみで、自動停止・無断のデバイス切替を行わない。
- Source Protection: Source Manifestを正本として既定で保護ロックし、Addition画像Source、明示追加context Source、`stagedInputRefs`を含む参照整合性確認、一時ロック解除確認、削除確認を通す危険操作境界。走査不能、欠損、hash不一致ではfail-closedとし、cascade deleteを行わない。SQLiteは再構築可能な索引に限定する。
- Task Management: Task Bundleを手動作業管理の正本とし、Source／Group多対多、不変コメント・blockerと追加専用event、返答待ち、Task親子・依存を管理する境界。現在値とeventは同じ`task.json`へatomicに保存し、WBSはGroupにリンクされたTaskの投影として専用正本を作らない。
- PM Support Views: Source／Group／Taskからデイリーブリーフィング、Decision Log、RAID等を導出する表示境界。カードは根拠への参照を持つ再生成可能なcacheに限り、ユーザー確定値はSourceまたはTaskへ保存して重複した管理正本を持たない。
- External Output Adapter: 承認時に固定した内容だけをJira、Confluence、Backlogの各Adapterへ変換・公開する境界。送信識別子と結果を保存して重複作成を防ぎ、資格情報はアプリ内部でmacOS Keychainからだけ解決し、設定、Vault、URL query、プロセス、環境、診断、クラッシュ情報、HTTPデバッグ出力へ出さない。
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
│                   ├── sanitized/  # URL安全化済み送信専用派生物
│                   ├── media/
│                   │   └── <speech-model>/
│                   │       ├── transcript-<role>.md            # role別の不変な生Transcript
│                   │       ├── transcript-combined.md          # role別を統合した不変snapshot
│                   │       ├── transcript-<role>-normalized.md # 辞書補正後
│                   │       └── normalization.json              # 補正履歴
│                   └── analysis/
│                       ├── revisions/
│                       │   └── <revision-id>/
│                       │       ├── revision.json
│                       │       ├── inputs/                  # 非Source・可変派生物の不変input snapshot
│                       │       ├── transcript-corrected.md  # 訂正版の不変snapshot
│                       │       └── summary.md               # 不変snapshot
│                       ├── conversations/
│                       ├── audits/
│                       └── summary.md          # 最新Revisionからのmaterialized view
├── contexts/             # 既存Context Bundleの読み取り互換専用。新規書き込みしない。
├── analyses/             # 既存Analysis Bundleの読み取り互換専用。新規書き込みしない。
├── dictionaries/
│   ├── common/
│   │   ├── terminology.json      # 最新辞書Revisionの投影
│   │   └── revisions/
│   │       └── <dictionary-revision-id>.json
│   └── groups/
│       └── <group-id>/
│           ├── terminology.json
│           └── revisions/
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

- Source Bundleは正規`sourceId`単位の永続的な記録である。新規のInput、Analysis、Output、Topic Source、External Ticket Sourceは正規Sourceモデルへ書き込む。
- 原本、文字起こし、プレビュー、動画キーフレームをSource Bundleへまとめる。
- 外部サービスから取り込んだ原文とユーザー補足は、AI生成物および原本と分離する。
- Source BundleをProject／Taskフォルダへ物理移動しない。複数Project／Taskとの関連はメタデータで表現する。
- GroupはSource IDだけを参照し、同じSourceを複数Groupへ再利用できる。Groupへの追加は生成を起動しない。
- Analysisは`kind: analysis`の派生Sourceである。既存Contextと`analyses/`は移行期間の読み取り互換表現とし、新規書き込み先ではない。
- Topic Sourceは`kind: topic`、`type: topic_excerpt`の派生Sourceである。snapshot所有Source、原音所有Source、nullableな親Revision、`system`／`microphone`等の単一role別Evidence Span、不変snapshotと選択byte列のhash、時刻／byte半開区間を保存し、原本／Transcriptを複製せず原音Sourceの該当時刻を再生する。親更新でspanを自動追従させず、Topic／Task／Group／Revision／派生Source／公開監査から参照される親Sourceは削除しない。
- External Ticket Sourceは`kind: input`、`type: external_ticket_snapshot`の不変Inputである。provider、providerが保証する不変instance ID（なければ正規化endpoint）、不変issue ID、必要時だけproject不変IDからremote keyを確定し、endpointと変更可能なissue／project keyは表示aliasとする。remote version、取得scope、snapshot hash、operation IDで更新系列を検証し、変更時だけ一意な系列tipを参照する新Sourceを作る。不変IDを検証できない手動Sourceは`unconfirmed`のまま独立保存する。Read Adapterは公開Adapterとinterface、Job、資格情報を分け、外部ticketを変更しない。
- 同じAnalysis Sourceへ追加して再分析する場合、Revision Bundleを追加する。初回のRevision 1を含む各Revisionは追加情報、理由、使用モデル、確認状態、差分、実際に使用したSource IDと`stagedInputRefs`を持つ。非Sourceまたは変更・削除され得る派生入力はRevision Bundleの`inputs/`へ不変snapshotする。
- `summary.md`は最新Revisionの閲覧用投影であり、Revisionと対話履歴から再生成できる。
- Analysisの一覧表示用の具体的要約、Action、Source保護ロック、外部公開記録は、SourceとRevisionの来歴を参照する構造化メタデータとして保存する。
- 各Revisionは`summaryPath`とSHA-256を伴うsummary本文の不変snapshotを持つ。snapshotがないRevisionは有効なRevisionとして扱わない。
- Task–Source／Task–Groupは各Task Bundleの`task.json`、Group–Sourceは各Group Bundleの`group.json`を唯一の正本とする。Source Manifestの逆方向ID配列とSQLiteはlegacy cacheまたは再構築可能な索引である。
- Taskは手動作成・編集可能で、タイトル、説明、状態、優先度、担当、予定／実績日、進捗、milestone、確認状態、作成元、`parentTaskId`、表示順、依存を持つ。コメントとblockerは追加専用とし、返答待ちは実作業状態と別に表現する。ユーザー確定値をAIが上書きしない。
- `manifest.json`は機械可読な永続メタデータを持つ。
- Sourceの保護状態はManifestを正本とし、欠落・不正・未知値を`locked`として扱う。SQLiteの保護状態はBundle走査から再構築する。
- 生Transcriptは`system`と`microphone`のrole別に、不変の原本として`derived/media/<speech-model>/`へ保持する。統合snapshot、辞書補正結果、訂正版snapshotは別ファイルとして保存し、生Transcriptを上書きしない。
- 用語辞書は`dictionaries/`へ共通・Group別に保持し、追加専用のRevisionとして更新する。過去Revisionと補正履歴を保持して過去Analysisの再現性を維持する。
- 保存する時刻はUTCのISO 8601を正本とし、表示時にだけAsia/Tokyoへ変換する。
- MarkdownはユーザーとClaude Codeが確認・再利用できる成果物とする。
- SQLiteは検索、関連、処理状態、再試行、レビューキュー用のローカル索引とする。
- SQLiteの永続情報は可能な限りSource Bundleから再構築可能にする。
- 実行待ち、処理中、ロック、再試行回数などの一時的な運用状態はSQLiteのみで保持できる。
- Claude実行用のstaging directoryはLocal VaultのBundle外にジョブ単位で作る一時領域であり、`jobId`、`operationId`、`createdAt`を持つ。永続化・バックアップ・Git管理せず、Claudeのcwdに固定する。処理完了、失敗、タイムアウト、中断後に削除し、次回起動時には実行中Jobへ属さない残存stagingをすべて回収する。

詳細と判断理由は[ADR-001](../06-adr/ADR-001-local-vault-storage.md)を参照する。

## 常駐型取得エージェント

- 初期対象OSはmacOSのみとする。Windows固有実装をMVPへ含めない。
- 通常時はmacOSメニューバーに収まり、常設ウィンドウを表示しない。
- キャプチャ、音声録音、画面録画を1操作またはグローバルショートカットで開始する。
- 録音・録画中は取得中であること、経過時間、停止操作を明示する。
- 保存中、AI処理中、完了、失敗を小さな状態表示またはOS通知で示す。
- 誤操作に備え、直近の取得を破棄できるようにする。
- メニューバーに確定済みSource数とLocal Vault使用量を表示する。
- Sourceは既定で保護ロックする。削除は参照整合性確認、一時ロック解除確認、削除確認を通過してからSource BundleをmacOSのゴミ箱へ移動し、即時完全削除しない。参照検出、キャンセル、失敗、異常終了、ゴミ箱移動成功時には一時解除を自動再ロックする。恒久的な手動ロック解除とは別に扱う。
- タスク整理画面ではAnalysis単体、未参照Source単体、Analysisと未参照Source一式を削除候補として表示できる。Task、Group、別Analysis／Revision、Addition画像Source、明示追加context Source、`stagedInputRefs`、派生Source、外部公開記録から参照中の場合は削除を禁止する。参照走査不能、欠損、hash不一致でもfail-closedとし、cascade deleteしない。
- 音声モデルは`Application Support/Contextory/Models`へ置き、Local Vault、Git、アプリ更新から分離する。
- 保存完了通知の権限は自動要求せず、ユーザーの明示操作で有効化する。
- 常駐メニューにはInput操作と処理状態だけを表示する。
- 録音確認は初期状態で無効とし、ユーザーが有効化した対象アプリだけを検知する。状態はアプリ別の`nextEligibleAt`と`suppressionReason`へ統一する。
- 判定は、録音中、当日抑制、15分snooze、60分cooldown、表示可能の順に評価する。`nextEligibleAt`到達時は`suppressionReason`を自動解除し、`today_suppressed`も翌日のローカル日付開始時刻に解除する。Slack／Teamsが20秒継続して前面で、表示可能な場合だけパネルを出す。
- 操作は録音開始、15分後に通知、今回は通知しない（60分抑制）、今日は通知しないとする。対象アプリ、20秒閾値、60分cooldownは設定で変更できる。
- 録音確認の検知・設定・cooldownはローカルに限定し、前面アプリの検知だけを初期対象とする。
- 録音開始前は選択中マイク名と短時間の入力レベルを表示する。前回選択のデバイスが使えない場合は、別デバイスへ黙って切り替えず、録音開始を保留して選択を求める。
- 録音中はマイクとシステム音声を別々に監視する。およそ5〜10秒の無音、USBマイク切断、入力停止は警告候補とし、「マイクを選択」「システム設定を開く」「録音停止」「継続」を表示する。自動停止は行わない。突然の切断・権限喪失では取得済みマイク原本を確定し、欠落開始・終了時刻、原因、使用デバイスをManifestへ記録する。システム音声は可能な限り継続保存し、別マイクへ無断で切り替えない。
- 録音中のマイク変更は、現在の区間を安全に保存してから、新しい選択で明示的に再開する。
- 補足、関連付け、手動再解析、結果詳細確認を常駐メニューへ置かない。
- 権限、Claudeモデル、診断、Local Vault表示は「設定・診断」サブメニューへ集約する。

## AI処理とフォールバック

- ユーザーが明示的に取得・取り込みした通常Input Sourceは保存後にClaude Code分析を自動開始する。来歴用の`type: transcript_correction` Sourceはこの自動解析Queueから除外し、親Analysisの明示的な再生成Jobだけが参照する。
- Input取得状態とAI解析状態を分離し、Claude解析中も新しいInputを保存してQueueへ追加できるようにする。
- Claude Code処理は同時実行せず1件ずつ直列処理し、待機件数を常駐メニューへ表示する。
- `pending_analysis`／`analyzing`のSourceと、`completion_sync_pending`のProcessing Jobは起動時に永続状態から古い順で復元する。`completion_sync_pending`は解析Queueではなく状態再同期の対象としてSQLiteから復元する。
- Claude Codeプロセスの上限時間は5分とし、超過時は終了要求後に必要なら強制終了する。
- 使用上限、認証切れ、タイムアウトは`retry_waiting`、不正出力と一般実行失敗は`analysis_failed`として記録する。
- 失敗Sourceは現在のQueueから隔離して後続を処理し、`retry_waiting`は自動再試行しない。
- 初回解析、Revision再分析、AI対話、Group展開の全Claude実行で、Local Vault、Source Bundle、別Source Bundleをcwdまたは`--add-dir`として直接公開しない。ユーザーが明示選択した入力だけを再生成可能な一時stagingへ配置し、Claudeのcwdをstaging directoryに固定する。Claude Codeのツールは`Read`だけに限定し、セッションを保存せず構造化JSONを受け取る。
- staging directoryには選択済みの未マスク原画像、テキスト、PDF、安全化済みURL、固定済みTranscript、代表フレームを配置できる。音声・動画原本、`localOpenUrl`、未選択ファイル、別Sourceの未選択原本、Vault全体は置かない。処理完了、失敗、タイムアウト、中断後に削除し、次回起動時は実行中Jobに属さない残存stagingをすべて回収する。
- 会社契約のClaude Codeへは明示取得・取り込みした業務情報を送信できるが、個人Claudeへは送信しない。パスワード、APIキー、Cookie、秘密鍵、セッショントークン、認証QRコード等を含む画面はユーザーが取得・送信対象として選択しない運用とし、万能な認証情報検出、自動マスク、検出失敗を理由とする一律のfail-closedは実装しない。画像内容をログ、診断、クラッシュ情報へ出さない。
- ユーザーが明示選択した原画像は未マスクで送信でき、Vision OCR、URL領域マスク、マスク失敗時の送信停止を必須としない。画像、`provided-text`、ユーザー入力から抽出して保存・表示・送信するURLはquery／fragmentを除去した`aiDisplayUrl`を使い、抽出・安全化失敗したURL値だけを送信対象から除外する。
- 初回のRevision 1を含む各Analysis Revisionは、実際に配置したfileの対象Revision ID、入力所有Source ID、`contentRole`、`inputType`、元の安全な相対path、staging論理path、staged bytesのSHA-256、MIME `contentType`、transformation種別／version、原ファイルSHA-256を`stagedInputRefs`へ固定する。非Source入力は保存先のAnalysis Sourceを所有者とし、別Revisionのsnapshotを使う場合は入力Revision IDも固定する。Revisionを作らないAI対話／Group展開は同じ配列を追加専用のinvocation auditへ固定する。
- 非Source入力と変更・削除され得る派生入力は対象Revision Bundleへ不変snapshotし、不変Source原本はSource ID、path、hashで参照して削除保護する。一時staging自体は正本にせず、`stagedInputRefs`から当時のbyte集合を再現・検証する。走査不能、欠損、path不正、hash不一致ではClaude実行と削除をfail-closedにする。
- 初回`summary.md`は固有Analysis SourceのRevisionとして原子的に保存し、ユーザー確認前は`proposed`として扱う。最新表示用`summary.md`はRevision履歴から再生成できる。
- Groupから生成する際は、Group IDだけでなく実際に使用した個別Source IDを固定し、原本を移動せずClaude Codeへ渡す。
- Analysis Sourceへの再分析では、現在のsummary、今回のテキスト・画像・URL追加、ユーザー指示を中心に渡す。音声、動画、PDFの追加は新規Source取り込みと前処理を経由する。
- Analysis SourceのAI対話では、対象Sourceとユーザーが明示選択した追加Source／Groupだけを文脈にする。Groupは送信時に個別Source IDへ展開してsnapshot保存し、対話とsummary更新は追加専用で保存する。
- AI対話、再分析、Group展開を含むすべてのClaude送信でADR-008を適用し、音声・動画原本ではなく前処理済み文字起こし・代表フレームだけを渡す。未前処理または前処理失敗のメディアが選択範囲にある場合はfail-closedで送信を止め、対象を表示する。
- 解析の成否は、canonical Analysis Source Manifest、最新Revision record、Revisionの不変Summary snapshot、保存済みSHA-256の検証完了で判定する。最新表示用`summary.md`はmaterialized viewとしてRevision snapshotから復元する。
- `operationId`はProcessing Job作成時にUUIDとして確定し、Claude実行前に永続化する。新規canonical Analysis Sourceでは必須とし、Analysisの`generation.operationId`へ同じ値を保存する。JobとAnalysisの`operationId`はlegacyのnull行を除く部分一意制約付きでSQLiteへ索引化し、新規writerはnullを拒否する。再索引時の重複はfail-closedとする。
- Analysis保存、Summary保存、親Manifestの完了更新は`AnalysisStore`が唯一の所有者とする。Jobは投影先を`originatingSourceId`として1件だけ固定し、親Manifestには`operationId`、Analysis Source ID、Revision ID、Summary snapshot hash、完了日時を`analysisCompletions`の追加専用recordとして保存する。複数の根拠SourceやGroupメンバーへ完了状態を配布しない。`CaptureModel`はQueue、Job、UI状態だけを担当し、親Manifestを重複更新しない。
- 復旧した`pending_analysis`／`analyzing` JobもClaude実行前に`operationId`でAnalysisを確認する。成功境界を満たすAnalysisがあれば再生成せず、親Manifestが同じ`operationId`の完了状態を示すか、`AnalysisStore`による更新が成功した場合だけJobを`completed`へ収束させる。更新できなければ`completion_sync_pending`とする。同じ`operationId`のAnalysis Sourceが存在するのに成功境界を満たさない場合は`analysis_integrity_failed`として停止し、Claudeを再実行しない。
- 解析結果の保存前に発生した失敗は`analysis_failed`／`retry_waiting`とし、保存後の状態同期失敗は`completion_sync_pending`として別表示にする。「自動解析失敗」はClaude解析またはAnalysis保存が実際に失敗した場合だけ表示する。
- `completion_sync_pending`／`completion_sync_failed`はProcessing JobのSQLite運用状態を正本とし、Source表示はJobとの対応から導出する。再同期ではcanonical Analysis Source Manifest、最新Revision record、不変Summary snapshot、`operationId`、SHA-256を検証し、完成済みならAnalysisを再生成せず親ManifestとJobだけを同期する。Analysis Sourceが存在するのに他の検証が失敗した場合は`analysis_integrity_failed`としてfail-closedで停止し、自動再解析・上書き・削除を行わない。Jobは`syncAttemptCount`、`lastSyncAttemptAt`、`nextRetryAt`、`lastSyncError`を持ち、各試行の開始前にSQLite transactionで試行回数と時刻を永続化する。自動再同期は5秒、30秒、5分の最大3回とし、3回失敗後は`completion_sync_failed`として自動処理を止め、診断表示と手動再同期を提供する。
- Transcript訂正による再解析では、訂正版Transcript、適用済み辞書項目だけの派生excerpt、ユーザーが選択した代表フレーム・画像・テキスト・安全化済みURLをstagingへ置く。excerpt正本はRevision監査領域へ不変保存してpathとSHA-256を記録し、stagingにはcopyだけを置く。辞書Revision snapshot全体はユーザーが明示選択しない限り置かず、音声・動画原本は渡さない。この限定は訂正再解析の規則であり、初回メディア解析はADR-008、staging境界全体はADR-012を正本とする。
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
- Analysis一覧は、内容が分かる具体的な要約とJST日時だけを表示する。Analysis表記、分類、状態、hash、Source IDは一覧へ出さず、詳細画面と診断画面で確認する。要約未生成時はSource種別とJST日時で暫定表示する。要約の根拠・確認状態を保存し、ユーザー確定の要約をAIが上書きしない。
- 一覧の日時は既定で分単位とし、表示要約とJST分が一致する集合だけを秒、なお一致する場合だけ小数秒まで拡張する。比較と拡張判定はlocale非依存で決定的に行い、行の内部識別に使うSource IDは表示しない。
- 一覧の表示要約は、ユーザー確認済み`presentationSummary`、ユーザー確認済みlegacy `presentationTitle`、`presentationSummary`、legacy `presentationTitle`、種別と日時のfallbackの順に解決する。読込時にlegacy値の削除やManifestのbackfillを行わない。
- Analysis詳細では、role別の生Transcript、統合Transcript、辞書補正後Transcript、現在の訂正版、訂正履歴、過去Summaryを確認できる。訂正版からSummaryとActionsを再生成し、過去Revisionを保持する。
- Analysis詳細の上部に「あなたの対応」を置き、自分の対応、他者への依頼、返答待ちをSummary本文と独立して表示する。Actionがない場合は所定の空状態を表示し、確認済みActionからTaskを作成できる設計にする。
- Markdown派生Output Sourceは外部公開前に、本文、Project／Space、種別、添付をユーザーが確認・承認する。承認時にはpublication ID、公開先、Project／Space、変換後payload、本文snapshot、添付一覧と各SHA-256、承認日時を固定し、変更時は再承認する。送信前にattempt ID、idempotency key、request fingerprintを保存し、remote ID、照合結果、outcomeを保存する。添付はhash、送信状態、remote attachment ID単位で失敗分だけを再実行し、結果不明の新規作成は自動再試行しない。
- Task詳細から根拠Analysis、Source、親Taskへ逆引きできる。
- タスク整理画面には将来「外部チケットを取り込む」を置き、URL／手動入力とAPI取得を選ぶ。保存前にidentity、取得範囲、重複候補、前回取得日時、差分を確認し、保存のみ・Group追加・Task作成・既存Task linkをユーザーが選ぶ。常駐メニューバーへ複雑な同期操作を追加しない。
- Source詳細には将来「話題として切り出す」を置き、Transcriptまたはタイムラインの手動範囲選択でTopic Sourceを作成する。AI候補は採用、却下、範囲／タイトル修正、統合、再分割のレビュー対象であり、自動確定しない。
- WBS表はGroupにリンクされたTaskの階層、予定日、状態、進捗、担当者、blockerを表示する。親子・依存の循環を禁止し、WBS番号は表示順から生成してTask IDの代わりにしない。簡易タイムラインとMarkdown／CSV／Excel出力は後続とする。
- デイリーブリーフィング、返答待ち・期限超過、Decision Log、RAID、要件変更・影響分析、ステータスレポート、顧客フィードバック整理、優先順位付け、リリース準備確認は、既存Source／Group／Taskから導出する後続ビューとする。
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
