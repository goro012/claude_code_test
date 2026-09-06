# 契約書レビューBot 評価ループ設計(Power Automate 自作フロー版)

Copilot Studio(クラシックオーケストレーション)で構築した契約書レビューBotについて、
「開発 → 実行 → 評価 → 改善」のループを速く回すための設計書とテンプレート一式です。
Power CAT Kit は使わず、Power Automate の自作フローと Dataverse で構成します。

Bot は **段階1(対象契約書のレビュー) → 段階2(雛形契約書との照合で該当する指摘を抑制)** の2段階で動くため、
評価も段階1・段階2・最終出力を別々に測り、業務部門には指摘ごとに **Q1 指摘は妥当か** と **Q2 雛形にも該当するか** を答えてもらいます。

## 狙い

- 業務部門なしで回る **内側ループ(自動回帰評価)** と、業務部門が判定だけを行う **外側ループ** に分離する。
- 業務部門には「教師データを書く」「全件見る」を頼まず、「Botの出力を直す」「割れたケースを判定する」だけを頼む。
- プロンプト修正から回帰結果までを数分にし、業務部門の関与は週30分の校正会と非同期判定に絞る。

## 読む順番

| ファイル | 内容 |
|---|---|
| [docs/01-loop-design.md](docs/01-loop-design.md) | ループ全体の設計、2つの呼び出しモード、出力の構造化、組み込み評価機能を使わない理由 |
| [docs/02-dataverse-schema.md](docs/02-dataverse-schema.md) | Dataverse テーブル7つ(Template、TestCase、ExpectedFinding、EvalRun、EvalResult、EvalFinding、Feedback)の列定義 |
| [docs/03-flows.md](docs/03-flows.md) | 子フロー + 評価フロー4本のアクション単位の手順書 |
| [docs/04-judge-prompt.md](docs/04-judge-prompt.md) | LLM 審査プロンプト本文と入出力 JSON |
| [docs/05-rubric.md](docs/05-rubric.md) | 業務部門向け 採点ルーブリックと Q1/Q2 の判定ガイド |
| [docs/06-operations.md](docs/06-operations.md) | 運用ルール、指標とリリースゲート、立ち上げ2週間計画 |
| [docs/07-annotation.md](docs/07-annotation.md) | 教師データを評価・フィードバックの副産物として育てるアノテーション設計(添削型カード、本番の指摘単位フィードバック、優先度と予算、LLM 下書き・過去資産からの候補生成、品質の仕組み) |
| [docs/08-utilization.md](docs/08-utilization.md) | アノテーション結果の活用(回帰の分母、プロンプト改善分類、段階2の受け入れ済み条件一覧、例示、審査の校正、業務側資産への還元、週次集約フロー F6) |

## テンプレート

| ファイル | 用途 |
|---|---|
| [templates/templates.csv](templates/templates.csv) | Template(雛形契約書)テーブル取り込み用(UTF-8 BOM 付き) |
| [templates/test_cases.csv](templates/test_cases.csv) | TestCase テーブル取り込み用(UTF-8 BOM 付き) |
| [templates/expected_findings.csv](templates/expected_findings.csv) | ExpectedFinding テーブル取り込み用(UTF-8 BOM 付き、雛形該当の列あり) |
| [templates/findings_schema.json](templates/findings_schema.json) | レビュー出力(段階1指摘 全件 + 段階2判定)のスキーマ |
| [templates/judge_output_schema.json](templates/judge_output_schema.json) | 審査出力 JSON のスキーマ |

## 注意

- 製品 UI 上のアクション名・列名は更新されることがあります。本書で「デザイナーで確認」と注記した箇所はテナント上で実際の名称を確認してください。
- フロー定義 JSON やソリューション zip は含みません。手順書に従ってテナント上で作成し、ソリューションに含めて管理してください。
