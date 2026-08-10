# Phase 0 配布・権限スパイク結果

- Status: 完了
- 実施日: 2026-08-11
- 対象: 開発者本人の会社Mac
- App Release: `v0.0.2-phase0`
- App version/build: `0.0.2 (2)`
- Bundle ID: `com.mashiyun.contextory`
- App commit: `8476c76`

## 結果

| 確認項目 | 結果 |
| --- | --- |
| 私用MacでのReleaseビルド | 成功 |
| ad-hoc署名、entitlements、同梱内容の検証 | 成功 |
| GitHub Releaseによる会社MacへのZIP提供 | 成功 |
| SHA-256照合 | 成功 |
| 会社Macでの起動 | 成功。quarantine属性の削除が必要だった |
| メニューバー常駐 | 成功 |
| 画面収録権限 | 許可済み |
| マイク権限 | 許可済み |
| Claude Code実行ファイル検出 | `/opt/homebrew/bin/claude`を検出 |
| Application Supportへのテスト保存 | 成功 |
| 診断情報のコピー | 成功 |
| 終了後の再起動 | 成功 |
| 再起動後の権限・保存状態維持 | 成功 |
| 会社Macでのローカルビルド | 優先経路が成功したため未実施 |

会社MacはmacOS 26.5.2（build 25F84）、arm64だった。Claude Codeは実行ファイルの検出までをPhase 0の対象とし、実際の非対話処理と認証状態はPhase 2で確認する。

更新版への置き換えによる権限維持はまだ未確認であり、次回Releaseで継続確認する。

## 配布手順

1. 私用MacでReleaseビルド、署名検証、同梱内容確認を行う。
2. `Contextory-macos.zip`と`Contextory-macos.zip.sha256`をprivate GitHub Releaseへ添付する。
3. 会社MacでGitHub Releaseから両ファイルをダウンロードする。
4. ZIPを展開する前にSHA-256を照合する。
5. 一致した場合だけアプリのquarantine属性を削除する。
6. アプリを起動し、必要な権限と診断情報を確認する。

Phase 0で確認したSHA-256は次のとおり。

```text
6785006234b407944fed4d792a97549cd340aab4d331ac4fb84d4dddc8e623b2  Contextory-macos.zip
```

会社Macでの確認例:

```bash
cd ~/Downloads
shasum -a 256 Contextory-macos.zip
xattr -dr com.apple.quarantine Contextory.app
codesign --verify --deep --strict --verbose=2 Contextory.app
open Contextory.app
```

`xattr`は、配布元とSHA-256が確認できた本人ビルドにだけ実行する。Mac全体のGatekeeperは無効化しない。将来のReleaseでは、そのReleaseに添付された新しいSHA-256を使用する。

## ローカル診断

会社Macから共有する診断情報には、アプリ版、OS、architecture、権限状態、Claude Code検出、Application Support保存結果、生成日時だけを含める。画面、音声、業務内容は含めない。

操作結果ログは次へ保存する。

```text
~/Library/Logs/Contextory/contextory.log
```

Application Supportのテストファイルは次へ保存する。

```text
~/Library/Application Support/Contextory/phase0-probe.json
```
