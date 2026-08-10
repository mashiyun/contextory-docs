# AGENTS.md

Codex向けの作業指示です。このリポジトリはContextory Docsです。プロダクト仕様の正本を管理しますが、このファイルへ長い仕様を複製しません。

## 作業範囲

- 対象: `/Users/mikey/workspace/vault/personal-vault/20-projects/contextory/docs`
- 全体入口: [README.md](README.md)
- MVP範囲: [02-requirements/mvp-scope.md](02-requirements/mvp-scope.md)
- 安全方針: [02-requirements/safety-principles.md](02-requirements/safety-principles.md)
- システム概要: [03-design/system-overview.md](03-design/system-overview.md)
- データモデル: [03-design/data-model.md](03-design/data-model.md)
- 開発フェーズ: [04-roadmap/development-phases.md](04-roadmap/development-phases.md)
- 未決事項: [05-decisions/open-questions.md](05-decisions/open-questions.md)
- ADR: [06-adr/](06-adr/)
- PoC: [07-poc/](07-poc/)

App実装、personal-vault本体、他リポジトリは、明示的な依頼がない限り変更しないでください。

## 文書方針

- 文書本文は原則日本語にしてください。
- 確定事項、仮定、未決事項、PoC結果を区別してください。
- 進行中・未実施の試験を成功・完了と記載しないでください。
- 推測で仕様や技術選定を確定しないでください。
- 仕様変更時はrequirements、design、ADR、PoC、roadmapの整合性を確認してください。
- READMEと変更文書のMarkdownリンクを確認してください。
- 秘密情報、APIキー、顧客情報、キャプチャ、録音、文字起こしを記録しないでください。

## Checkpoint

- 作業開始時に`.claude/work-progress.local.md`へ目的、対象、対象外、開始commit、Git状態を記録してください。
- 長い作業では完了項目、判断、検証結果、次の作業を更新してください。
- checkpointはローカル専用であり、秘密情報や実業務データを書かないでください。

## Git運用

- 作業開始時に`git status --short --branch`と差分を確認してください。
- CodexとClaude Codeが同時に同じリポジトリを編集しないでください。
- 他方やユーザーの未コミット変更を自分の変更として扱わないでください。
- 既存変更と重なる場合は作業を止めて報告してください。
- 読み取り専用の調査ではファイルを変更しないでください。
- 診断依頼では、修正の明示的依頼がない限り実装しないでください。
- commit/pushは明示的に依頼された場合のみ行ってください。
- force push、reset、rebase、履歴書き換えを行わないでください。

## 検証

```bash
git diff --check
```

READMEと変更した文書からの相対リンクが実在することを確認してください。

## 完了報告

変更ファイル、検証結果、Git状態、未実施事項を報告してください。
