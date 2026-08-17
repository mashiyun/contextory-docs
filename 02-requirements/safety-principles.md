# 安全・プライバシー原則

## 確定方針

- 単一ユーザーの非公開ツールとして扱い、第三者向けデータ分離やクラウド保存を追加しない。
- 原本と業務データは会社PCのローカル領域へ保存する。
- Local Vaultはアプリのソースコードリポジトリ外へ保存する。
- 業務データをGitHubへ保存しない。
- iCloud、OneDriveなどへの暗黙の同期を前提にしない。
- キャプチャ、音声録音、画面録画はユーザーの明示操作で開始する。
- Slack／Teamsの前面化による録音確認はローカル通知に限り、録音は必ずユーザーの明示操作で開始する。
- 原本を上書きせず、AI送信用の変換データとAI結果を派生物として扱う。
- ユーザーが明示選択した原画像は、会社契約Claude Codeへ未マスクで送信できる。Vision OCR、URL領域マスク、マスク失敗時の送信停止を必須とせず、原画像はローカルに保持する。
- 提供テキストのURLはローカルで解析・安全化し、queryとfragmentを除去したAI送信用URLだけを送る。ローカル遷移専用URLはClaude、AI出力、Markdown、ログへ渡さない。
- URLとして抽出した値の安全化、または音声・動画のメディア前処理に失敗したSourceは自動送信せず、`needs_review`とする。画像のOCR・マスクを実行しないことは失敗にしない。
- ユーザーが明示操作でキャプチャ、録音、録画、手動取り込みしたSourceは、その操作をAI処理の許可として保存後にClaude Codeへ自動送信する。
- 送信対象外アプリと一時停止を設定できるようにする。
- 録音忘れ防止の前面アプリ検知、対象アプリ設定、cooldown、ユーザー選択はローカルだけで処理し、会話内容、画面内容、ウィンドウタイトル、アプリ利用履歴を外部送信しない。
- 送信失敗時も原本と処理状態を保持し、無断で再送を繰り返さない。
- 会社契約のClaude Codeには、ユーザーが明示取得・取り込みした業務情報を送信できる。個人契約のClaudeや個人アカウントへは送信しない。
- パスワード、APIキー、Cookie、秘密鍵、セッショントークン、認証QRコード等を含む画面は、ユーザーが取得・送信対象として選択しない運用とする。万能な認証情報検出、自動マスク、検出失敗を理由とする一律のfail-closedは実装しない。
- URLはqueryとfragmentを除去した`aiDisplayUrl`だけをClaudeへ渡し、`localOpenUrl`は渡さない。署名付きURLの秘密部分もこのURL安全化の対象とする。
- 初回解析、Revision再分析、AI対話、Group展開の全Claude実行で、選択済み送信対象だけを置く一時staging directoryを作成する。Source Bundleをcwdまたは`--add-dir`として直接公開せず、Claudeのcwdをstaging directoryに固定し、処理完了・失敗・タイムアウト・中断後に削除する。
- 各Analysis Revisionは初回のRevision 1を含め、実際に送信したfileを`stagedInputRefs`へ固定する。Revisionを作らないAI対話／Group展開は同じ配列を追加専用のinvocation auditへ固定する。非Source入力と変更・削除され得る派生物はRevision Bundle内へ不変snapshotし、不変Source原本はSource ID、安全な相対path、原ファイルSHA-256を固定して削除保護する。一時stagingは正本にしない。
- staging metadataには`jobId`、`operationId`、`createdAt`を持たせ、起動時に実行中Jobへ属さない残存stagingをすべて回収する。正本Revision保存済みの場合はClaudeを再実行せず、完了状態だけを同期する。
- 生Transcript、ユーザー訂正、訂正版Transcript、用語辞書、補正履歴はLocal Vault内で管理し、外部サービスへ同期しない。
- Transcript訂正による再解析でstagingへ置けるのは、訂正版Transcript、適用済み辞書項目だけの派生excerpt、およびユーザーが選択した代表フレーム、画像、テキスト、安全化済みURLとする。excerpt正本はRevision監査領域へ不変保存して元辞書Revision参照、使用した項目、永続パス、SHA-256を記録し、stagingにはcopyだけを置く。辞書Revision snapshot全体はユーザーが明示選択しない限り送らない。この限定は訂正再解析の規則であり、初回メディア解析はADR-008、staging境界全体はADR-012を正本とする。
- 音声・動画原本は訂正・辞書機能でもClaudeへ渡さない。既存のADR-008の前処理境界を変更しない。
- 訂正・辞書操作で、原音、生Transcript、過去Revisionを削除しない。
- 用語辞書への登録は必ずユーザー確認を経て確定する。訂正内容からの自動登録は行わない。
- Sourceは新規・既存を問わず既定で保護ロックし、ロック情報の欠落・不正・未知値も`locked`として扱う。正本はSource Manifestであり、SQLiteは再構築可能な索引に限定する。お気に入りは保護ロックとは別の将来の絞り込み属性とする。
- 未参照Source一覧は、現行Appが読める参照だけを候補根拠とする。候補はManifest、Bundle境界、種別を検証できるcanonical `kind: input` Sourceに限定し、Analysisを含む派生Source、legacy Analysis／Context、種別不明Sourceを候補にしない。読取エラーや未対応形式では通常のエラーを表示し、完全な参照グラフを保証しない。
- 未参照Sourceの複数選択削除は、対象を示す明示確認後に既存のSource単位ゴミ箱移動を順に実行する。batchのsnapshot、atomicity、全件再検証、rollbackを追加せず、各Sourceの通常の成功・失敗を表示する。候補資格が検証できない場合と削除復旧が未完了または隔離中の場合は、一覧と削除をfail-closedにする。
- Task／Groupの「削除」はarchiveであり、Bundle、根拠link、履歴を物理削除・Trash移動しない。
- 外部公開はユーザーの明示承認後だけに行う。公開先の資格情報はmacOS Keychainに限定し、アプリ内部でだけ参照を解決する。UserDefaults／plist、Local Vault、Markdown、ログ、Git、URL query、プロセス引数、環境変数、診断、クラッシュ情報、HTTPデバッグ出力へ保存・出力しない。
- 外部チケットのRead Adapterはinterface、Job、監査、Keychain credentialを外部公開Adapterから分離し、Read専用の最小scopeだけを使う。外部ticketの変更、コメント、完了を行わず、pagination未完了などの部分API取得をSourceとして保存しない。不変remote IDを検証できない手動Sourceは`unconfirmed`として独立保存し、推測で統合しない。添付redirect先へ元credentialを転送しない。
- 外部サービスの作成結果が不明な場合は、新規作成を自動再試行しない。作成済みremote IDが確認できる添付失敗は、同じremoteへの添付だけを再実行できる。

バックグラウンドで無断取得したデータやユーザー操作を伴わないSourceは送信しない。処理失敗を隠さず、無断で再送せず、タスク整理画面から手動で再実行できることを要件とする。

使用上限、認証切れ、タイムアウトとなったSourceは`retry_waiting`へ隔離し、自動再試行しない。原本を保持し、後続Queueを止めず、将来のタスク整理画面からユーザーが再開する。

非公開ツールであっても、macOSの画面収録・マイク権限、録音中表示、ローカルデータ保護、Claude Code送信範囲は省略しない。

会社Macへ提供するバイナリは私用Macで作成し、ビルド後に署名検証、内容確認、ハッシュ記録を行う。配布物へSource Bundle、SQLite、ログ、秘密情報、開発用設定を同梱しない。

## Local Vault

- 画像、音声、動画などの原本は通常ファイルとして保存し、SQLite BLOBへ格納しない。
- Sourceごとに固有IDを持つフォルダを作り、原本、`manifest.json`、文字起こし、要約、派生物をまとめる。
- フォルダ名とファイル名に顧客名、人物名、メール件名などの機密情報を含めない。
- SQLiteは検索、関連付け、処理キュー、レビュー対象抽出に使用する。
- 永続的な業務知識をSQLiteだけに閉じ込めず、Source Bundleから索引を再構築できるようにする。
- 原本のハッシュを保持し、取得後の意図しない変更を検出できるようにする。
- 総コンテンツ数とSource Bundleの合計使用量をLocal Vaultから再計算できるようにする。一時ファイルは確定済みコンテンツの集計へ含めない。

## 原本の保持と容量管理

- キャプチャ画像は容量が比較的小さいため、原則として保持する。
- 音声・動画は再確認や操作手順の根拠として必要になるため、期間による自動削除を行わない。
- `summary.md`や文字起こしが生成済みでも原本を自動削除しない。削除候補を提示し、対象をユーザーが確認して明示承認した場合だけ処理する。
- 原本を削除した後もSource Bundle、Manifest、原本SHA-256、要約、文字起こし、削除日時と理由を保持する。
- ただし、誤取得や不要データについてユーザーが「Analysisと元データを削除」を明示した場合は、Task、Group、別Analysis／Revision、Addition、追加context、`stagedInputRefs`から参照されていないことを確認したうえでSource BundleごとmacOSのゴミ箱へ移動できる。この操作は容量整理ではなくSource全体の破棄として扱う。
- 削除transactionの起動時復旧は、再索引、サイドバー集計、一覧表示、未参照候補走査より先に有効recordを完了させる。復旧recordの不正、欠損、復元先競合は安全に隔離して削除導線を停止するが、有効なSource／Analysisの閲覧や件数表示は継続する。`job.json`を持たないentryを削除JobまたはSourceとして解釈・移動・削除しない。
- 保護ロック中はこの削除を実行できない。削除の順序は「参照確認→削除操作中だけの一時ロック解除確認→対象・参照影響を表示する削除確認→macOSゴミ箱移動」に統一する。Group、Task、別Analysis／Revision、Addition画像Source、明示追加context Source、`stagedInputRefs`、派生Source、外部公開記録も参照整合性確認の対象とし、走査不能、欠損、hash不一致ではfail-closedにする。cascade deleteは行わない。
- 恒久的な手動ロック解除と、削除操作中の一時解除を分離する。一時解除は参照検出、キャンセル、失敗、異常終了、ゴミ箱移動成功のいずれでも自動再ロックする。既存Sourceの保護backfillが失敗・中断・未検証なら削除不可とする。
- AI処理失敗中、未確認、処理中、返答や判断の根拠として必要な原本は削除候補にしない。
- 削除候補の条件は実際の容量増加と利用状況を確認して設計するが、動画を含む原本の明示承認は常に省略しない。

## 情報最小化

- 業務上必要な送信者名、所属、役割、発言は保持できるようにする。
- 電話番号、個人メール、住所などは将来の任意マスク候補にできるが、万能な自動マスクを送信の前提にしない。資格情報を含む画面は取得・送信しない運用境界とする。
- 自動判定結果はユーザーが確認・修正できるようにする。

## 禁止事項

- 実業務データをfixtureやテストスナップショットへ転用しない。
- 原本を無断で削除・上書きしない。生Transcriptも原本として上書き・削除しない。
- ユーザー確認なしに用語辞書へ項目を登録・変更しない。競合・曖昧・不正な辞書項目を自動適用しない。
- 保護ロックを解除せず、または参照整合性確認を通さずにSource、Analysis、原本を削除しない。
- AIの推測をユーザー確認済み事実として保存しない。
- AIによるグルーピング候補をユーザー確認済みの所属として扱わない。
- ユーザーによるグルーピング修正を禁止・上書きしない。
- 前面アプリ検知だけを根拠に録音、画面録画、AI送信を自動開始しない。
- 音声・動画原本はAI対話、再分析、Group展開を含むいかなるClaude送信にも渡さない。ADR-008の前処理成果物がない場合はfail-closedで送信しない。
- staging directoryには、選択済みの未マスク原画像、テキスト、PDF、安全化済みURL、固定済み文字起こし、代表フレームを配置できる。音声・動画原本、`localOpenUrl`、未選択ファイル、別Sourceの未選択原本、Source BundleまたはVault全体は配置しない。
- 外部Outputを、ユーザーが本文、Project／Space、種別、添付を確認・承認する前に公開しない。承認時にはpublication ID、公開先、Project／Space、変換後payload、本文snapshot、添付一覧と各SHA-256、承認日時を固定し、1つでも変われば再承認を要求する。
- 送信前にpublication ID、attempt ID、idempotency key、request fingerprintを永続化し、remote ID、照合結果、outcomeを保存する。結果不明では新規作成を自動再試行しない。添付ごとにhash、送信状態、remote attachment IDを保存し、失敗分だけを再実行する。
- API Token、OAuth refresh token、Cookieなどの資格情報をUserDefaults／plist、Vault、Markdown、監査ログ、Git、URL query、プロセス引数、環境変数、診断、クラッシュ情報、HTTPデバッグ出力へ保存・出力しない。Keychain参照はアプリ内部だけで解決する。
- ログ、診断、クラッシュ情報へ画像内容、文字起こし、秘密情報を出さない。
