# 録音忘れ防止要件

## Status

次回実装候補。2026-08-12時点では仕様整理のみで、未実装。

## 目的

SlackまたはTeamsを開いて会話を始める前に、録音が必要かをユーザー自身が判断できるようにする。会議内容やマイク利用を監視して録音を自動化する機能ではない。

録音開始後の入力デバイス選択、レベル表示、無音・切断候補の警告は、[録音入力選択・無音警告要件](recording-input-monitoring.md)で別に定める。前面アプリ検知とマイク無音検知を同一の会議検知機能として扱わない。

## 検知と表示

- 初期状態は無効とし、ユーザーが設定から明示的に有効化した場合だけ検知する。
- macOSが提供する前面アプリの変化を利用し、SlackまたはTeamsが20秒間継続して前面であることを検知する。
- Contextoryが録音中でなく、`nextEligibleAt`到達後であり、`suppressionReason`が当日抑制でないときだけ、録音確認パネルを表示する。
- 初期対象は前面アプリのBundle IDまたはアプリ名による判定である。Huddle／会議開始、通話状態、他アプリのマイク利用、画面UI内容の精密検知は行わない。
- アプリ別cooldownの既定値は60分とする。継続時間、cooldown、対象アプリは設定で変更できる。
- 状態はアプリ別の`nextEligibleAt`と`suppressionReason`へ統一する。`suppressionReason`は`today_suppressed`、`snoozed`、`cooldown`、または未設定とする。
- `nextEligibleAt`到達時は`suppressionReason`を自動解除する。`today_suppressed`も、翌日のローカル日付開始時刻に`nextEligibleAt`へ到達して自動解除する。
- 検知、cooldown、対象アプリ設定、今回の選択はローカルだけで処理・保存する。会話内容、ウィンドウタイトル、画面内容、アプリ利用履歴を外部送信しない。

## ユーザー操作

確認パネルには次の操作を設ける。

- パネル表示時: `nextEligibleAt`を表示時刻から60分後、`suppressionReason`を`cooldown`へ設定する。
- `録音開始`: 通常の録音開始操作へ進む。録音開始には常にこの明示操作が必要である。
- `15分後に通知`: `nextEligibleAt`を操作時刻から15分後、`suppressionReason`を`snoozed`へ上書きする。前のcooldownよりこの値を優先する。
- `今回は通知しない`: `nextEligibleAt`を操作時刻から60分後、`suppressionReason`を`cooldown`へ設定する。
- `今日は通知しない`: `nextEligibleAt`を翌日のローカル日付開始時刻、`suppressionReason`を`today_suppressed`へ設定する。

## 制約と受入条件

- 前面アプリになっただけで録音、画面録画、AI送信を自動開始しない。
- 録音中は同じ通知を表示しない。
- `15分後に通知`、`今回は通知しない`、`今日は通知しない`を選んだ後、それぞれの抑制期間は繰り返し表示しない。
- 有効化済みでも、Slack／Teamsが前面になってから20秒未満では候補表示しない。
- 表示可否の判定優先順位は、録音中、`today_suppressed`、`snoozed`、`cooldown`、表示可能とする。
- 対象外アプリや通知無効設定では検知・通知しない。
- ネットワーク接続、クラウドサービス、外部APIなしで動作する。
