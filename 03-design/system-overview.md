# システム概要

## 暫定構成

```text
Capture / Audio Recording / Screen Recording
                    ↓
             Local Source Store
                    ↓
      Transcription / metadata extraction
                    ↓
     Pre-authorized Claude Code analysis
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
- AI Adapter: 事前許可されたSourceをClaude Codeへ渡す境界と処理結果。
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
│               ├── transcript.md
│               ├── summary.md
│               ├── preview.jpg
│               └── frames/
├── exports/
└── index/
    └── contextory.sqlite3
```

- Source BundleはSource ID単位の永続的な記録である。
- 原本、文字起こし、要約、プレビュー、動画キーフレームを同じBundleへまとめる。
- Source BundleをProject／Taskフォルダへ物理移動しない。複数Project／Taskとの関連はメタデータで表現する。
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

## 自動処理とフォールバック

- 事前許可されたSourceは、保存完了後に文字起こし・Claude Code分析・Markdown生成・グルーピング候補生成を自動開始する。
- 各工程をProcessing Jobとして分離し、途中失敗から再開できるようにする。
- 認証切れ、タイムアウト、不正な出力、アプリ終了などを失敗状態として残す。
- 自動処理に失敗してもSource Bundleを保持し、ユーザーが手動で再実行できるようにする。
- 完全自動化のために安全性や復旧性を犠牲にしない。
- Claude CodeのモデルはSonnetを既定とし、ユーザーがOpusへ切り替えられるようにする。
- 選択モデルはローカル設定として保持し、Claude Code起動時の`--model`へ明示的に渡す。
- AI生成物には使用Provider、モデル、実行日時、根拠Sourceを記録する。
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
- 一覧管理・詳細レビューのUI形態は、常駐型取得エージェントのMVPとは分けて決定する。

## 未確定事項

- デスクトップ技術スタック。
- ローカルVaultの正本形式。
- 検索インデックスの方式。
- 録音・文字起こし方式。
- Claude Codeの安全な起動・入出力方式。
- macOSメニューバーと任意の小型フローティングコントロールの使い分け。
- グローバルショートカットの初期値と競合解決。
- 自動グルーピングの類似度、時間、人物、アプリ情報の重み付け。
- 日次レビューの通知方法と締め時間。
