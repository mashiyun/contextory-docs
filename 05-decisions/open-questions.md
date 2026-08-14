# 未決事項

## 技術

- 最低対応macOSバージョンをいくつにするか。会社MacのOSを確認して確定する。
- SQLite accessに標準C API、軽量wrapper、ORMのどれを使うか。
- Source Manifestの必須項目とschema migrationをどう設計するか。今回確定した`generation.operationId`、Transcript来歴、旧フィールド読込順は再検討対象にせず、残る項目とmigration実装単位だけを決める。
- SQLite Indexの初期テーブル、全文検索、再構築処理をどう設計するか。今回確定した`operationId`一意制約、重複時fail-closed、辞書・補正索引のBundle正本性は再検討対象にしない。
- URL検出用のローカルVision OCRは確定している。画面本文の一般OCRを実装するか、実装する場合の方式と保存範囲は未決とする。
- SonnetからOpusを提案する自動ルーティング条件と、実行前確認をどの範囲で必要にするか。
- 画面録画から音声とキーフレームをどのように抽出するか。一般本文OCRは未決のため、URL検出用Vision OCRと混同しない。
- 即時処理Queueの追加機能をどう管理するか。`completion_sync_pending`／`completion_sync_failed`の復旧・上限・手動再同期と、既存`retry_waiting`の自動再試行禁止は確定済みとする。
- 常駐UIをmacOSメニューバーだけにするか、小型フローティングコントロールも用意するか。
- キャプチャ、録音、録画のグローバルショートカットをどう割り当てるか。
- 既存のContext／Analysis Bundleを統一SourceとGroup／Revisionモデルへ移行するschema migrationと後方互換をどう実装するか。
- Revision差分をMarkdownの行差分、構造化フィールド差分、または両方のどこまで表示するか。
- 確定したURL検出用Vision OCRについて、URL領域検出精度、マスク範囲、tokenらしさの具体的な判定規則をどう検証・調整するか。
- Slack／Teamsの対象Bundle ID一覧と、ユーザー設定での追加・変更UIをどう確定するか。
- 録音入力デバイスの安定識別子、開始前メーターの実装方式、マイク／システム音声別の無音・切断検出をどう設計するか。
- 無音警告のしきい値、観測時間（目安5〜10秒）、ノイズ環境別の調整方法をPoCでどう決めるか。無音と設定ミスを完全に区別しない前提を維持する。
- Analysis一覧に出す具体的要約の最大長をどう決めるか。ユーザー修正済み要約の優先順位は確定済みで、未決は最大長だけとする。
- Actionの期限候補の抽出規則、状態遷移、ActionからTaskを作成した後の同期範囲をどう決めるか。
- Whisperへ用語ヒントを渡す方式（`whisper-cli`の初期プロンプト等）、語数上限、実際の改善効果をPoCでどう確認するか。ヒントの効果は未検証であり、実装前に確定しない。

### 段階実装への影響

- 上記の未決事項は、確定済みの原本不変性、`operationId`収束、完了再同期上限、辞書Revision正本、決定的補正を変更しない。
- 要約の最大長は表示調整であり、保存・移行・一覧の実装を妨げない。確定までは保存本文を切り詰めず、UI側だけで折り返し・省略表示する。
- Whisper用語ヒントはロードマップの段階6に限定する。段階1〜5の一覧、完了収束、Transcript訂正、辞書保存、文字起こし後補正は先に実装できる。
- その他のUI、Adapter、運用上の未決事項は、それぞれを使用するPhaseの開始前に解消し、先行Phaseのschemaや安全境界を再び未決へ戻さない。

## プロダクト

- 案件とタスクの作成・関連付け候補をどの確信度から自動提示するか。
- キャプチャ直後に必要な確認操作は何か。
- 人物名を保持しながら不要な個人情報をマスクする判定ルール。
- 日次レビューの通知時刻、未確認件数、優先順位をどう表示するか。
- Library／Review Interfaceを同一アプリ内の別ウィンドウ、別アプリ、ローカルWeb UIのどれにするか。
- 保護ロックの解除履歴、ゴミ箱からの復元時のロック状態、参照中Sourceを削除候補画面でどう説明するか。
- Jira／ConfluenceのCloud・Data Center種別、Backlogの会社環境、必須Custom Field、Project／Space選択、添付制限をどう確定するか。Adapter実装時のAPI仕様はAtlassian／Nulabの公式資料だけで確認する。
- 外部公開で`outcome_unknown`となった場合のremote確認、既存remoteへの紐付け、明示的な再作成のUXをどう設計するか。
- Transcript訂正のUI形態（該当箇所の直接編集、全文編集、誤変換候補の提示）をどうするか。
- 辞書登録候補を提示するタイミング（訂正直後、レビュー時、まとめて確認）と、確認負荷をどう抑えるか。

## 運用

- 会社PCでのVault保存先とバックアップ方針。
- アプリ更新時にBundle IDと署名identityを維持し、画面収録・マイク権限が継続するか。
- 会社Macでのローカルビルドが必要になった場合のclone、build、update手順。
- ローカル監査記録の保持期間、容量上限、破損時の整合性検査をどう設計するか。

上記の会社Mac導入関連項目は機能実装の終盤まで保留せず、Phase 0の配布・権限スパイクで可能な限り解消する。

Phase 0で、GitHub Releaseによる受け渡し、SHA-256照合、ad-hoc署名アプリの起動手順、会社MacのOS／architecture、Claude Code実行ファイル検出を確認した。詳細は[Phase 0 配布・権限スパイク結果](../07-poc/phase-0-distribution-permissions-result.md)を参照する。

## 決定済み（2026-08-15）

未決から外した項目と、確定内容の参照先を記録する。いずれも仕様として確定済みであり、実装は未了である。

- 一覧のJST表示粒度と同一要約・同一分の識別: 既定は分単位、一致集合だけ秒、なお一致する場合だけ小数秒へ拡張する。詳細は[Source・Group・Task・Output要件](../02-requirements/source-group-task-output.md)。
- `completion_sync_pending`の再同期契機、上限回数、診断: 起動時復旧対象とし、各試行の開始前に回数を永続化する。5秒・30秒・5分の最大3回、以後は`completion_sync_failed`で自動停止し診断と手動再同期を提供する。部分保存・破損は`analysis_integrity_failed`へ分離する。詳細は同要件と[データモデル](../03-design/data-model.md)。
- `presentationSummary`とlegacy `presentationTitle`の移行: 新規書き込みは新フィールドのみ、読込は固定の優先順位、legacy値を削除せずbackfillしない。
- Group辞書の適用範囲: MVPは共通辞書＋ユーザーが明示選択した1つのGroup辞書までとし、複数Group所属時も自動選択しない。詳細は[Transcript訂正・用語辞書要件](../02-requirements/transcript-correction-terminology.md)と[ADR-014](../06-adr/ADR-014-transcript-correction-terminology.md)。
- 辞書Revisionの粒度と過去Analysisの再現: scope単位のRevision snapshotを`dictionaryRevisionRefs`（`scope`、`scopeId`、`revisionId`、`snapshotPath`、`snapshotSha256`）として固定参照する。
- 同一操作の収束条件: `operationId`をidempotency keyとし、`requestFingerprint`は監査用に分離する。fingerprint一致だけで別operationを統合しない。
- schema切替順: version 1／2／3 reader、SQLite nullable列・新規テーブル、Bundle検証、legacy nullを除く部分一意索引、検証後の新規writer有効化の順とする。旧Bundleを一括backfillしない。
- Transcript訂正位置: 固定snapshot上のUTF-8 byte半開区間を使い、scalar境界、訂正前byte列、非重複を検証する。別モデルのTranscriptへ自動追従させない。
- role別Transcript統合: `mergeAlgorithmVersion: 1`はsession相対の開始時刻、終了時刻、role順、元segment連番で安定sortし、必須時刻が不正ならfail-closedとする。
- 辞書照合単位: `normalizationAlgorithmVersion: 1`はUnicode scalar列の完全一致とし、暗黙の大文字小文字・全角半角・かな・送り仮名・Unicode正規化を行わない。

正規モデルでのClaude実行は、選択済み送信対象だけを置く一時staging directoryを`Read`限定で渡し、セッション非保存、構造化JSON出力とする。通常画像・テキストは選択済みならstagingへ置けるが、Source Bundle全体、音声・動画原本、`localOpenUrl`は置かない。URLはquery／fragmentを安全化する。このstaging境界は未実装であり、既存の実行経路を置き換える。ユーザーが明示的に取得・取り込みしたSourceは、その操作を処理許可として保存後に自動分析し、失敗時の自動再送は行わない。
