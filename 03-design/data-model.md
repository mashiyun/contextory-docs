# データモデル

## 暫定モデル

### Project

顧客、製品、機能など、継続する業務文脈のまとまり。

### Task

Project内で完了条件を持つ対応単位。担当、期限、状態、待ち先を持てる。

### Source

判断の根拠となる原本または記録。Capture、Audio Recording、Screen Recording、Transcript、Emailなどを含む。取得、保存、文字起こし、AI処理、レビューの状態を持つ。

Sourceは固有IDを持つSource Bundleとして保存する。Source Bundleには次を含められる。

- `manifest.json`: 機械可読な永続メタデータ。
- `source.<ext>`: 変更しない原本。
- `transcript.md`: 音声または動画の文字起こし。
- `summary.md`: AIによる要約と抽出結果。
- `preview.jpg`: 一覧表示用の派生画像。
- `frames/`: 画面録画から抽出したキーフレーム。

Source IDにはULIDまたは同等の衝突しにくい識別子を使用する。ファイルパスには顧客名、人物名、件名を含めない。

### Source Manifest

最低限、次の項目を持つ。

```json
{
  "schemaVersion": 1,
  "id": "01K2ABC...",
  "type": "capture",
  "capturedAt": "2026-08-10T09:30:00+09:00",
  "sourceApplication": {
    "name": "Example App",
    "bundleIdentifier": "com.example.app"
  },
  "original": {
    "path": "source.png",
    "sha256": "..."
  },
  "transcript": null,
  "summary": "summary.md",
  "processingStatus": "needs_review",
  "projectIds": ["project_customer_a"],
  "taskIds": ["task_schedule_reply"],
  "reviewStatus": "proposed"
}
```

実際のスキーマは実装前に確定する。この例へ秘密情報や不要な個人情報を追加しない。

Phase 1では`sourceApplication`へ取得時の前面アプリ名とBundle IDだけを保存する。メール件名、文書名、ウィンドウタイトルは自動メタデータへ含めない。既存Manifestに`sourceApplication`がない場合も読み込める後方互換を維持する。

### Summary

Sourceまたは複数SourceからAIが生成する要約。生成モデル、生成日時、根拠Source、レビュー状態を持つ。

### Grouping Proposal

Sourceを既存または新規のProject／Taskへ関連付けるAI候補。候補先、理由、確信度、レビュー状態を持つ。

ユーザーが候補を修正した場合、AI候補、修正後の関連、修正日時を分離して保持する。再分析でユーザー確認済みの関連を自動上書きしない。

### Knowledge Item

Sourceから抽出したFact、Decision、Action、Question、Risk、Waiting。

### Review

AIの提案に対するユーザーのConfirmed、Corrected、Deferred、Rejected状態。

### Processing Job

文字起こし、AI要約、Markdown生成、グルーピングなどの処理単位。実行待ち、実行中、完了、失敗、再試行回数、最後のエラーを持つ。運用状態はSQLiteで管理する。

自動実行と手動再実行を同じProcessing Jobモデルで扱い、実行契機を記録する。

## 基本関係

- Projectは複数Taskを持つ。
- Sourceは複数Project・Taskに関連できる。
- Knowledge Itemは根拠となるSourceを参照する。
- SummaryとGrouping ProposalはAI生成物であり、確定情報とは分離する。
- AI生成内容とユーザー確認済み内容を別状態で保持する。
- ユーザーの修正履歴を残し、次回の関連判定へ利用できるようにする。

## SQLite Index

初期実装では次のテーブルを最小候補とする。

```text
sources
processing_jobs
groups
source_groups
```

Library／Review Interfaceを実装する段階で次を追加候補とする。

```text
review_items
knowledge_items
```

SQLiteへ画像、音声、動画をBLOBとして保存しない。SQLiteを唯一の永続正本にせず、破損・削除時にSource Bundleを走査して再構築できる索引として設計する。

## 整合性

- Source取得時は原本を保存し、ハッシュを計算してから`manifest.json`を確定する。
- `manifest.json`とMarkdownの更新は一時ファイルへ書き、同一ファイルシステム上で置き換える。
- SQLite更新に失敗してもSource Bundleを失わない。
- 再索引時はSource IDと`schemaVersion`を使用する。
- Project／Task関連を変更しても原本ファイルを移動・複製しない。

詳細なSQLiteスキーマとManifest schemaは未確定である。保存方式の基本判断は[ADR-001](../06-adr/ADR-001-local-vault-storage.md)で確定する。
