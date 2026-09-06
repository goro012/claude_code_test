# 04. LLM 審査プロンプト(LLM as judge)

F1 の手順 5-7 で Foundry に渡すプロンプトです。被評価側のレビューモデルとは別のデプロイ(環境変数 `crv_JudgeModelDeployment`)を使い、temperature は 0 にします。

## 審査の責務は段階1の意味照合だけ

審査 LLM に任せるのは「段階1の各期待指摘が、Bot の段階1指摘のどれに対応するか」の照合だけです。
段階2(雛形にも該当するか)の一致は、照合が決まれば Bot の `applies_to_template` と期待指摘の「雛形該当(期待)」のフラグ比較で決定的に計算できるため、LLM に判断させません(F1 手順 4-10)。
`actual_findings` に段階2の項目を含めるのは記録の整合のためで、審査 LLM には「照合に使わない」と明示します。

## 設計方針

- **意味で照合する。** 文言の一致は求めない。条項参照が多少ずれていても(「第12条」と「第12条第1項」)、同じ論点なら一致とみなす。
- **1つの期待指摘に対して、実際の指摘は最大1つ対応させる。** 1つの実際の指摘が複数の期待指摘を包含していれば、複数に一致してよい。
- **重要度は別に判定する。** 一致していても重要度が違う場合は `severity_match = false` にし、一致自体は維持する。
- **余分な指摘は「期待に無い」だけを意味する。** 正しいか誤りかは審査 LLM には判断させず、人判定(F3)に回す。
- **出力は JSON のみ。** 前置き・コードフェンス禁止。

## システムプロンプト

```text
あなたは契約書レビューの品質評価者です。
与えられた「期待指摘(expected_findings)」と「実際の指摘(actual_findings)」を比較し、
期待指摘のそれぞれについて、実際の指摘の中に同じ論点を扱うものがあるかを判定してください。

判定ルール:
1. 照合は意味で行います。文言の違い、言い回しの違いは無視してください。
2. 条項参照は補助情報です。条項が多少ずれていても同じ論点であれば一致(matched=true)としてください。
   条項が同じでも論点が異なる場合は不一致(matched=false)です。
3. 期待指摘1件に対して、実際の指摘は最も近い1件だけを対応させ、その id を matched_actual_id に入れてください。
   対応が無ければ null にしてください。
4. 一致した場合、重要度(severity)が同じなら severity_match=true、違えば false としてください。
   不一致の場合、severity_match は null にしてください。
5. どの期待指摘にも対応しなかった実際の指摘を extras に列挙してください。
   extras の正誤は判断しないでください。列挙だけを行います。
6. 各判定に、根拠を日本語で1文だけ添えてください(reason)。
   実際の指摘に含まれる applies_to_template, template_clause_ref, template_excerpt, template_reason は
   照合の判断に使わないでください(雛形に該当するかどうかは別工程で評価します)。
7. overall_comment には、欠落した必須指摘があれば何が欠けたかを、なければ「必須指摘はすべて含まれています」と、日本語で2文以内で書いてください。
8. 出力は次の JSON スキーマに厳密に従い、JSON 以外の文字(前置き、コードフェンス、注釈)を一切含めないでください。

出力スキーマ:
{
  "matches": [
    {
      "expected_id": "string",
      "expected_required": "must" | "should",
      "matched": true | false,
      "matched_actual_id": "string" | null,
      "severity_match": true | false | null,
      "reason": "string"
    }
  ],
  "extras": [
    { "actual_id": "string", "title": "string" }
  ],
  "overall_comment": "string"
}
```

## ユーザーメッセージ(F1 が組み立てる審査入力)

```json
{
  "contract_type": "業務委託",
  "position": "受注側",
  "expected_findings": [
    {
      "id": "TC-0001-01",
      "clause_ref": "第12条",
      "category": "損害賠償",
      "severity": "high",
      "required": "must",
      "description": "受注側である当社の損害賠償責任に上限が無い点を指摘し、対価総額を上限とする修正を提案すること。",
      "rationale": "上限が無いと当社の負担が無制限になる。",
      "template_applicable": false
    },
    {
      "id": "TC-0001-02",
      "clause_ref": "第15条",
      "category": "準拠法・裁判管轄",
      "severity": "low",
      "required": "should",
      "description": "専属的合意管轄が相手方本店所在地の裁判所になっている点を指摘すること。",
      "rationale": "",
      "template_applicable": true
    }
  ],
  "actual_findings": [
    {
      "id": "A1",
      "clause_ref": "第12条第1項",
      "category": "損害賠償",
      "severity": "medium",
      "title": "損害賠償の範囲・上限が定められていない",
      "detail": "乙が負う損害賠償の範囲および上限額の定めがなく、乙の負担が過大になるおそれがある。",
      "suggestion": "賠償額の上限を委託料の総額とする条項を追加する。",
      "applies_to_template": false,
      "template_clause_ref": "第14条",
      "template_excerpt": "乙の損害賠償責任は、本契約に基づき乙が受領した委託料の総額を上限とする。",
      "template_reason": "雛形は上限を定めており、対象契約書とは異なるため非該当。"
    },
    {
      "id": "A2",
      "clause_ref": "第9条",
      "category": "再委託",
      "severity": "medium",
      "title": "再委託に甲の書面承諾が必要",
      "detail": "…",
      "suggestion": "…",
      "applies_to_template": true,
      "template_clause_ref": "第10条",
      "template_excerpt": "乙は、甲の書面による事前の承諾なく、本業務を第三者に再委託してはならない。",
      "template_reason": "雛形も同じ条件を定めているため該当。"
    }
  ]
}
```

`actual_findings` の `id` は F1 側で `A1, A2, …` と連番を振ります(子フローの出力には id が無いため、Select で付与)。

## 期待される出力例

```json
{
  "matches": [
    {
      "expected_id": "TC-0001-01",
      "expected_required": "must",
      "matched": true,
      "matched_actual_id": "A1",
      "severity_match": false,
      "reason": "A1 は第12条の損害賠償上限の欠如を指摘し、対価総額を上限とする修正を提案しており同じ論点。ただし重要度は high に対し medium。"
    },
    {
      "expected_id": "TC-0001-02",
      "expected_required": "should",
      "matched": false,
      "matched_actual_id": null,
      "severity_match": null,
      "reason": "裁判管轄に関する指摘は実際の指摘に含まれていない。"
    }
  ],
  "extras": [
    { "actual_id": "A2", "title": "再委託に甲の書面承諾が必要" }
  ],
  "overall_comment": "必須指摘はすべて含まれています。推奨指摘の裁判管轄が欠落しています。"
}
```

## F1 での集計の読み替え

審査出力と、Bot 指摘の `applies_to_template`(以下 Bot 判定)、期待指摘の `template_applicable`(以下 期待)を組み合わせて計算します。

| 指標 | 計算 |
|---|---|
| 段階1 必須一致数 | `matches` のうち `expected_required = must` かつ `matched = true` の件数 |
| 段階1 必須総数 | `matches` のうち `expected_required = must` の件数 |
| 段階1 推奨一致数 / 総数 | 同様に `should` |
| 余分指摘数 | `extras` の件数 |
| 重要度不一致数 | `matched = true` かつ `severity_match = false` の件数 |
| 段階2 評価対象数 | `matched = true` かつ 期待 ≠ null の件数 |
| 段階2 一致数 | そのうち Bot 判定 = 期待 の件数 |
| 誤抑制数 | `matched = true` かつ Bot 判定 = true かつ 期待 = false(利用者から見えなくなった指摘) |
| 誤通過数 | `matched = true` かつ Bot 判定 = false かつ 期待 = true |
| 最終出力 必須総数 | `expected_required = must` かつ 期待 = false の件数 |
| 最終出力 必須一致数 | そのうち `matched = true` かつ Bot 判定 = false の件数 |

`extras`(期待に無い指摘)は段階2を評価できないため、人判定(F3)で Q1(妥当か)と Q2(雛形にも該当するか)を聞き、妥当なら期待指摘に取り込みます。

## 審査の信頼性を検証する

- F3 で業務判定者が「審査に不同意」とした割合を毎週見る。10% を超えたら審査プロンプトかルーブリックの語彙を見直す。
- 最初の2週間は、審査結果のうち無作為 10 件も F3 に混ぜて送り、審査が「怪しくない」ケースでも正しいことを確認する。
- 審査プロンプトを変更したときは、過去の EvalRun 1回分を同じ EvalResult に対して再審査し、人判定との一致率が下がっていないことを確認してから切り替える。
