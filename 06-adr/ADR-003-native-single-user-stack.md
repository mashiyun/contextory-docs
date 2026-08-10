# ADR-003 macOSネイティブ・単一ユーザー構成

- Status: Accepted
- Date: 2026-08-10

## Context

Contextoryは開発者本人だけが、自分のMacと会社Macで利用する非公開ツールである。一般公開、App Store配布、複数ユーザー、クラウドサービスは想定しない。

中心機能はmacOSメニューバー常駐、グローバルショートカット、範囲キャプチャ、システム音声・マイク録音、画面録画であり、macOS固有APIへの依存が大きい。Web技術によるクロスプラットフォーム性より、macOS公式フレームワークとの直接統合、常駐時の軽量性、権限処理の明確さを優先する。

## Decision

- 言語はSwiftを使用する。
- UIはSwiftUIを使用する。
- 常駐UIは`MenuBarExtra`を基本とする。
- 画面・音声取得にはScreenCaptureKitと必要なmacOS公式フレームワークを使用する。
- ローカル保存はADR-001のSource BundleとSQLite Indexを使用する。
- AI処理はローカルにインストールされた会社契約のClaude Codeを非対話実行する。
- アプリは私用Macでのビルドを優先し、会社Macではビルド済みバイナリを実行する。直接配布が安定しない場合は、会社Macでのローカルビルドを許容する。
- 一般公開、App Store対応、Windows対応、複数ユーザー、クラウドBackendを初期設計へ含めない。

## Consequences

### Positive

- ScreenCaptureKitとmacOS権限へ直接アクセスできる。
- メニューバー常駐とグローバルショートカットをネイティブに実装できる。
- Tauri／Electronとネイティブ処理を橋渡しする層が不要になる。
- 公開配布、アカウント、サーバー、課金、クロスプラットフォーム対応を省略できる。
- 自分の利用環境に合わせてOS・デバイス・UXを限定できる。
- 通常経路では会社MacにXcode、Swift、ソースコードを置かずに済む。

### Negative

- Swift／SwiftUIとmacOS APIの実装知識が必要になる。
- WindowsやWebへ直接展開できない。
- macOSバージョン差と画面収録・マイク権限の影響を受ける。
- 将来第三者へ配布する場合は、署名、notarization、インストール、サポート、安全設計を再評価する必要がある。

## Development environment observed

2026-08-10時点の開発Macで次を確認した。

- macOS 26.2、Apple Silicon arm64。
- Xcode 26.6。
- Swift 6.3.3。
- Claude Code 2.1.226、`/opt/homebrew/bin/claude`。

会社MacのmacOS、CPU architecture、Claude Codeは別途確認し、最低対応macOSバージョンとビルド対象architectureを確定する。通常経路では会社MacへのXcodeとSwift導入を不要とし、直接配布が安定しない場合だけ導入する。

## Follow-up

- 会社MacのmacOS、CPU architecture、Claude Codeを確認する。
- 最低対応macOSバージョンを決定する。
- SwiftUIメニューバーアプリと範囲キャプチャの縦切りPoCを実装する。
- Claude Code非対話実行と構造化出力をPoCする。
- SQLite access方式を決定する。
