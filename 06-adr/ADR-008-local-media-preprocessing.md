# ADR-008: 音声・動画をローカル前処理してからAIへ渡す

## Status

Accepted

## Context

Claude CodeへM4AやMOVをそのまま渡しても、バイナリを読み取れない旨だけを記したAnalysisが生成され、Input整理として価値がない。音声・動画原本をローカル外へ出さず、AIが解釈できる前処理成果物を作る必要がある。全Claude送信の選択・staging・監査境界は[ADR-012](ADR-012-minimal-claude-staging.md)を正本とする。

## Decision

- 音声・動画Sourceは、初回Claude解析、Analysis Sourceの再分析、AI対話、Group展開を含むすべてのClaude送信の前に必ずローカル前処理する。
- システム音声とマイク音声をrole別にPCMへ変換し、`whisper-cli`とローカルモデルで個別に文字起こしする。
- 動画は音声文字起こしに加え、AVFoundationで最大12枚の代表フレームを生成する。
- Claude CodeにはM4A／MOV等の音声・動画原本を渡さず、固定済み文字起こし、代表フレーム等の前処理成果物だけを音声・動画Source由来の入力として使う。
- Claude実行時はADR-012の一時staging directoryへ、ユーザーが明示選択した入力だけを配置する。音声・動画Sourceからは前処理済み文字起こしと代表フレームだけを配置し、Source Bundleをcwdまたは`--add-dir`として直接公開しない。
- このADRは、ユーザーが明示選択した原画像、代表フレーム、固定済みTranscript、PDF、テキスト、安全化済みURLの送信を禁止しない。それらを含む全入力の許可範囲、資格情報非送信、`stagedInputRefs`、削除保護、staging回収はADR-012に従う。
- 選択範囲に未前処理または`preprocessing_failed`の音声・動画が1件でも含まれる場合は、対象を表示してfail-closedで送信を止める。別のSourceだけへ暗黙に範囲を縮小して送信しない。
- モデルはLocal VaultやGitへ置かず、`~/Library/Application Support/Contextory/Models/`へ配置する。設定画面から推奨モデルを取得でき、ファイル選択によるオフライン配置、フォルダ表示、削除にも対応する。
- モデルまたは`whisper-cli`がない、変換・文字起こしに失敗した場合は`preprocessing_failed`とし、Analysisを生成しない。
- whisper.cpp v1.9.2をarm64・`GGML_NATIVE=OFF`・静的ライブラリ構成で再現ビルドし、`whisper-cli`とMITライセンス本文をReleaseアプリへ同梱する。会社MacのHomebrewへ依存しない。
- 多言語`base`（147,951,465 bytes）と`small`（487,601,967 bytes）を併存・選択でき、取得後にモデルごとの公式SHA-256を照合する。モデルはアプリへ同梱しない。
- 文字起こしは`derived/media/<speech-model>/`へ分離し、同一Sourceを別モデルで再処理しても既存の前処理成果物とAnalysisを上書きしない。
- Analysis Manifestへ使用した音声モデルを記録し、同じSourceから派生した結果を比較可能にする。
- GPU初期化が利用環境に左右されないよう、MVPの文字起こしはCPU実行を既定とする。

## Consequences

- 読めないメディアについて無意味なAnalysisを作らなくなる。
- 原本を保ったまま前処理を再実行できる。
- モデル容量をアプリ更新と分離できる。
- 初回セットアップで選択モデルの取得または手動配置が必要になる。`small`は精度向上と引き換えに約488 MBの保存容量と処理時間を要する。
- 発話時刻の精密な統合は追加実装が必要である。
