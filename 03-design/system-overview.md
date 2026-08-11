# システム概要

## 暫定構成

```text
Capture / Audio Recording / Screen Recording
                    ↓
             Local Source Store
                    ↓
      Transcription / metadata extraction
                    ↓
       Confirmed Claude Code analysis
                    ↓
      Summary and Markdown generation
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
- AI Adapter: ユーザーがSource単位で確認したデータをClaude Codeへ渡す境界と処理結果。
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
│               └── frames/
├── contexts/
│   └── <context-id>/
│       ├── context.json
│       └── analyses/
│           └── <analysis-id>/
│               ├── analysis.json
│               └── summary.md
├── analyses/             # Source単体の派生Analysis
├── exports/
└── index/
    └── contextory.sqlite3
```

- Source BundleはSource ID単位の永続的な記録である。
- 原本、文字起こし、プレビュー、動画キーフレームをSource Bundleへまとめる。
- 外部サービスから取り込んだ原文とユーザー補足は、AI生成物および原本と分離する。
- Source BundleをProject／Taskフォルダへ物理移動しない。複数Project／Taskとの関連はメタデータで表現する。
- ContextはSource IDだけを参照し、同じSourceを複数の組み合わせへ再利用できる。
- Analysisは固有IDを持つ追加専用の派生物とし、再解析で過去結果を上書きしない。
- `manifest.json`は機械可読な永続メタデータを持つ。
- MarkdownはユーザーとClaude Codeが確認・再利用できる成果物とする。
- SQLiteは検索、関連、処理状態、再試行、レビューキュー用のローカル索引とする。
- SQLiteの永続情報は可能な限りSource Bundleから再構築可能にする。
- 実行待ち、処理中、ロック、再試行回数などの一時的な運用状態はSQLiteのみで保持できる。

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
- 保存完了通知の権限は自動要求せず、ユーザーの明示操作で有効化する。
- 常駐メニューにはInput操作と処理状態だけを表示する。
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
- Source Bundleを作業ディレクトリにし、Claude Codeのツールを`Read`だけに限定して、セッションを保存せず構造化JSONを受け取る。
- `summary.md`は固有Analysis Bundleへ原子的に保存し、ユーザー確認前は`proposed`として扱う。
- Context Groupへ手動で関連付けたSourceは、原本を移動せずまとめてClaude Codeへ渡し、統合した解釈を生成する。
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
