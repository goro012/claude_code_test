# 契約書レビューBot 評価ループ設計(Power Automate 自作フロー版)

Copilot Studio(クラシックオーケストレーション)で構築した契約書レビューBotについて、
「開発 → 実行 → 評価 → 改善」のループを速く回すための設計書とテンプレート一式です。
Power CAT Kit は使わず、Power Automate の自作フローと Dataverse で構成します。

## 狙い

- 業務部門なしで回る **内側ループ(自動回帰評価)** と、業務部門が判定だけを行う **外側ループ** に分離する。
- 業務部門には「教師データを書く」「全件見る」を頼まず、「Botの出力を直す」「割れたケースを判定する」だけを頼む。
- プロンプト修正から回帰結果までを数分にし、業務部門の関与は週30分の校正会と非同期判定に絞る。

## 読む順番

| ファイル | 内容 |
|---|---|
| [docs/01-loop-design.md](docs/01-loop-design.md) | ループ全体の設計、2つの呼び出しモード、出力の構造化、組み込み評価機能を使わない理由 |
| [docs/02-dataverse-schema.md](docs/02-dataverse-schema.md) | Dataverse テーブル5つの列定義 |
| [docs/03-flows.md](docs/03-flows.md) | 子フロー + 評価フロー4本のアクション単位の手順書 |
| [docs/04-judge-prompt.md](docs/04-judge-prompt.md) | LLM 審査プロンプト本文と入出力 JSON |
| [docs/05-rubric.md](docs/05-rubric.md) | 業務部門向け 採点ルーブリックと判定ガイド |
| [docs/06-operations.md](docs/06-operations.md) | 運用ルール、指標とリリースゲート、立ち上げ2週間計画 |

## テンプレート

| ファイル | 用途 |
|---|---|
| [templates/test_cases.csv](templates/test_cases.csv) | TestCase テーブル取り込み用(UTF-8 BOM 付き) |
| [templates/expected_findings.csv](templates/expected_findings.csv) | ExpectedFinding テーブル取り込み用(UTF-8 BOM 付き) |
| [templates/findings_schema.json](templates/findings_schema.json) | レビュー出力(指摘一覧 JSON)のスキーマ |
| [templates/judge_output_schema.json](templates/judge_output_schema.json) | 審査出力 JSON のスキーマ |

## 注意

- 製品 UI 上のアクション名・列名は更新されることがあります。本書で「デザイナーで確認」と注記した箇所はテナント上で実際の名称を確認してください。
- フロー定義 JSON やソリューション zip は含みません。手順書に従ってテナント上で作成し、ソリューションに含めて管理してください。
