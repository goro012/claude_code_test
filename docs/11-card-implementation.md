# 11. カード実装例(Power Automate / Copilot Studio)

docs/10 の3種類のカードを、アクション単位・数式レベルでどう作るかの例です。テンプレートは templates/flow_f3_block.json、templates/flow_f2_blocks.json、templates/powerfx_result_card.txt。

## 1. 共通の組み立て方(F3・F2b)

**カード JSON は文字列連結ではなく「オブジェクト」として組み立てます。** 「作成(Compose)」の JSON エディタに動的コンテンツを差し込むと、値は文字列として扱われ、`"` や改行のエスケープが不要です。文字列を `concat` で作って後から `json()` すると、指摘文に `"` が入った瞬間に壊れます。

```
配列変数 blocks を初期化(Array)
Apply to each(指摘)  ※同時実行オフ(配列変数への追加順を守るため)
  ├ 作成「ID」
  ├ 作成「Bot 指摘」  ← json(items(...)?['crv_findingjson'])
  ├ 作成「ブロック」  ← 指摘1件分の Container(JSON エディタ + 動的コンテンツ)
  └ 配列変数に追加 blocks ← outputs('ブロック')
作成「ヘッダ」「末尾」
作成「カード」        ← body に ヘッダ・blocks・末尾 を並べる
Teams: アダプティブ カードを投稿して応答を待機
回答の取り出し → Dataverse 更新
```

「カード」の Compose(JSON エディタ):

```json
{
  "type": "AdaptiveCard",
  "$schema": "http://adaptivecards.io/schemas/adaptive-card.json",
  "version": "1.5",
  "body": @{union(createArray(outputs('ヘッダ')), variables('blocks'), createArray(outputs('末尾')))},
  "actions": [
    { "type": "Action.Submit", "title": "送信",
      "data": { "action": "f3_submit", "evalresult_id": "@{items('Apply_to_each_EvalResult')?['crv_evalresultid']}" } }
  ]
}
```

`union` は配列を連結します(同一オブジェクトは重複除去されますが、指摘ブロックは id が異なるので影響しません)。

**サイズ確認**: `length(string(outputs('カード')))` が 25,000 を超えたら分割します(`chunk(variables('blocks'), 6)` で 6 件ずつ、カードを複数投稿)。

## 2. F3 判定依頼カード(業務判定者向け)

**トリガー**: 手動(子フロー)。入力 `evalrun_id`、`scope`。

**EvalResult 1件分の手順**

1. **Dataverse: 行を一覧(EvalFinding)**
   ```
   フィルター: _crv_evalresultid_value eq @{items('Apply_to_each_EvalResult')?['crv_evalresultid']} and crv_reviewrequested eq false
   並べ替え:   crv_priority asc
   行数:       @{parameters('crv_MaxFindingsPerCard')}
   ```
   scope = disagreements のときはフィルターに `and (crv_kind eq <余分> or crv_kind eq <欠落> or crv_s2agree eq <誤抑制> or crv_s2agree eq <誤通過> or crv_s2expected eq <未確認>)` を足す(選択肢の数値はテナントの値)。
2. 0 件なら次の EvalResult へ(条件分岐)。
3. **配列変数 blocks を初期化**。
4. **Apply to each(EvalFinding、同時実行オフ)**
   1. **作成「ID」** `replace(items('Apply_to_each_EvalFinding')?['crv_evalfindingid'], '-', '')`
   2. **作成「Bot 指摘」** `json(items('Apply_to_each_EvalFinding')?['crv_findingjson'])`(欠落の行は `crv_findingjson` が空なので、対応する期待指摘から同じ形を組む: `json(concat('{"title":"', ..., '"}'))` ではなく、Compose の JSON エディタで `{"title": "@{...crv_description}", ...}` と書く)
   3. **作成「ブロック」** templates/flow_f3_block.json を貼り、`@{...}` を動的コンテンツに置き換える。
   4. **配列変数に追加** blocks ← `outputs('ブロック')`
5. **作成「ヘッダ」**
   ```json
   {
     "type": "Container", "style": "emphasis", "bleed": true,
     "items": [
       { "type": "ColumnSet", "columns": [
         { "type": "Column", "width": "stretch", "items": [ { "type": "TextBlock", "size": "Large", "weight": "Bolder", "text": "契約書レビュー 判定依頼" } ] },
         { "type": "Column", "width": "auto", "items": [ { "type": "TextBlock", "isSubtle": true, "horizontalAlignment": "Right", "text": "@{length(variables('blocks'))} 指摘" } ] } ] },
       { "type": "FactSet", "facts": [
         { "title": "ケース", "value": "@{items('Apply_to_each_EvalResult')?['crv_testcaseid']?['crv_name']} @{items('Apply_to_each_EvalResult')?['crv_testcaseid']?['crv_title']}" },
         { "title": "雛形", "value": "@{items('Apply_to_each_EvalResult')?['crv_templateid']?['crv_name']}" },
         { "title": "実行", "value": "@{triggerBody()?['text']}" },
         { "title": "回答期限", "value": "@{formatDateTime(addDays(utcNow(), 5), 'yyyy-MM-dd')}" } ] },
       { "type": "TextBlock", "size": "Small", "isSubtle": true, "wrap": true,
         "text": "指摘ごとに Q1(妥当か)と Q2(雛形にも該当するか)を選んでください。Bot の文はそのまま期待指摘になります。直したいときだけ「内容を修正する」を開いてください。" }
     ]
   }
   ```
   参照列(`crv_testcaseid` の名前など)は「行を一覧」の展開(`$expand`)で取るか、事前に TestCase を「ID で行を取得」します。
6. **作成「末尾」** … 「足りない指摘はありますか?」の Container(templates/adaptive_card_f3.json の末尾ブロックをそのまま)。
7. **作成「カード」** … 1 節の JSON。
8. **Dataverse: 行を更新(EvalFinding)** … 対象行の人判定依頼済 = はい(Apply to each で回す)。
9. **Teams: アダプティブ カードを投稿して応答を待機**
   - 投稿先: チャット(フロー ボット)、受信者: 判定者の UPN(TestCase の担当(業務)、無ければ環境変数 `crv_TeamsChannelOrUsers`)
   - メッセージ: `@{outputs('カード')}`(JSON エディタではなく、メッセージ欄に Compose の出力トークンを1つだけ置く)
   - 更新メッセージ: 「回答を受け付けました。ありがとうございます。」
   - 「…」→ 設定 → タイムアウト: `P5D`
10. **回答の取り出し** … 手順1の行でもう一度 **Apply to each(EvalFinding)**。出力は `body('カード投稿')?['data']` に入力 id をキーにして返ります。
    ```
    ID   : replace(items('Apply_to_each_2')?['crv_evalfindingid'], '-', '')
    q1   : body('カード投稿')?['data']?[concat('q1_',   outputs('ID_2'))]
    q1r  : body('カード投稿')?['data']?[concat('q1r_',  outputs('ID_2'))]
    q2   : body('カード投稿')?['data']?[concat('q2_',   outputs('ID_2'))]
    req  : body('カード投稿')?['data']?[concat('req_',  outputs('ID_2'))]
    sev  : body('カード投稿')?['data']?[concat('sev_',  outputs('ID_2'))]
    conf : body('カード投稿')?['data']?[concat('conf_', outputs('ID_2'))]
    title: body('カード投稿')?['data']?[concat('title_', outputs('ID_2'))]
    修正あり: or(not(equals(title, Bot 指摘 title)), not(equals(desc, Bot 指摘 detail)), not(equals(sugg, Bot 指摘 suggestion)))
    判定者: body('カード投稿')?['responder']?['email']
    ```
    **Dataverse: 行を更新(EvalFinding)** … 人判定 Q1 = q1 を選択肢値に変換(`if(equals(q1,'valid'), <妥当の値>, if(equals(q1,'invalid'), <不当の値>, <判定不能の値>))`)、Q2、不当の理由、自信、修正提案 JSON(`{"title": title, "description": desc, "suggestion": sugg, "required": req, "severity": sev, "template_applicable": q2}` を Compose で作って `string()`)、人コメント、判定者、判定日時 `utcNow()`。
11. **足りない指摘** … `body('カード投稿')?['data']?['miss_desc']` が空でなければ ExpectedFinding 候補を作成(docs/07 3.1)。
12. **タイムアウト分岐** … 手順10のアクションの「実行条件の構成」を「成功した」に、並列に置いた「作成: タイムアウト」の実行条件を「失敗した / タイムアウトした」にして、何もせず終了する。次回 F3 で再送するため、手順8の「人判定依頼済 = はい」は **回答後(手順10)に移す** のが簡単です。

## 3. F2b 本番フィードバックカード(利用者向け)

Copilot Studio から呼ばれるフローは 100 秒以内に応答する必要があり、10 分待つ F2b を子フローから直接呼べません。**Dataverse トリガーで非同期化** します。

**子フロー「レビュー実行」側**(応答を返す直前)

- **Dataverse: 行を追加(FeedbackRequest)** … 会話ID、利用者 UPN(トピックから渡した `System.User.Email` 等)、findings_json、prompt_version、ファイル名、状態 = 待ち。

**F2b**

1. **トリガー: Dataverse「行が追加、変更、または削除されたとき」** … テーブル = FeedbackRequest、変更の種類 = 追加。
2. **Delay** … `PT@{parameters('crv_FeedbackDelayMinutes')}M`
3. **Dataverse: ID で行を取得(FeedbackRequest)** … 状態が「回答済」(F2a で答えた)なら終了(条件分岐)。
4. **作成「findings」** `json(triggerOutputs()?['body/crv_findingsjson'])?['findings']`
5. **指摘の選択**
   - **Filter array「非該当」** From `outputs('findings')`、条件 `@equals(item()?['applies_to_template'], false)`
   - **Filter array「該当」** 条件 `@equals(item()?['applies_to_template'], true)`
   - **作成「表示」** `take(body('非該当'), int(parameters('crv_ProductionAskPerConversation')))`
   - **作成「非表示」** `take(body('該当'), 2)`
6. **Apply to each(表示、同時実行オフ)** … 作成「AID」= `concat('A', add(int(indexOf(string(outputs('表示')), string(item()))), 1))` は不安定なので、**Select で先に id を振る**: Select From `range(0, length(outputs('表示')))`、Map `{ "aid": concat('A', add(item(), 1)), "f": outputs('表示')[item()] }`。その配列を Apply to each し、templates/flow_f2_blocks.json の `shown_block` を Compose に貼って配列変数 shown_blocks に追加。
7. **Apply to each(非表示)** … 同様に `hidden_block` を hidden_blocks に追加(id は `H1, H2`)。
8. **作成「カード」** … templates/adaptive_card_f2.json の形。body = ヘッダ + 「表示した指摘」見出し + shown_blocks + 「表示しなかった指摘」見出し + hidden_blocks + 全体評価。actions = 送信(`f2_submit`)とスキップ(`f2_skip`)。`data` に `conversation_id`、`prompt_version`、`shown` = 表示指摘の配列(id と指摘 JSON)、`hidden` = 非表示指摘の配列を入れておくと、回答時に指摘 JSON を再取得せずに済みます。
9. **Dataverse: 行を更新(FeedbackRequest)** … 状態 = 送信済。
10. **Teams: アダプティブ カードを投稿して応答を待機** … 受信者 = 利用者 UPN、タイムアウト `P3D`。
11. **分岐** `body('カード投稿')?['data']?['action']`
    - **f2_submit**:
      - Apply to each(`body('カード投稿')?['data']?['shown']`): `r = data?[concat('r_', item()?['aid'])]` が空でなければ **Feedback 行を追加**(評価 = r、理由分類 = `data?[concat('rc_', aid)]`、指摘 JSON = `string(item()?['f'])`、非表示指摘か = いいえ、会話ID、プロンプト版、入力参照 = ファイル名、トリアージ状態 = 未処理)。
      - Apply to each(`hidden`): `h = data?[concat('h_', aid)]` が空でなければ Feedback 行(評価 = h、非表示指摘か = はい)。
      - `data?['overall']` が空でなければ会話単位の行(指摘 JSON = 空)。
      - FeedbackRequest 状態 = 回答済。
    - **f2_skip**: Feedback 行を1行(評価 = スキップ、指摘 JSON = 空、トリアージ状態 = 見送り)。FeedbackRequest 状態 = スキップ。
    - **タイムアウト**(実行条件「失敗した / タイムアウトした」): FeedbackRequest 状態 = 期限切れ。何も記録しない。

**F2a との二重送信防止**: F2a が回答を受けたら FeedbackRequest の状態を「回答済」にする。F2b は手順3でそれを見て終了する。

## 4. レビュー結果カード(Copilot Studio のトピック)

クラシックのトピックで、子フロー呼び出し(アクション)の後に「メッセージを送信」→「アダプティブカード」を追加し、「…」→「数式で編集」に templates/powerfx_result_card.txt を貼ります。

```
{
  type: "AdaptiveCard", version: "1.5",
  body: Table(
    { type: "Container", style: "emphasis", items: Table( ...ヘッダ... ) },
    { type: "Container",
      items: ForAll(
        Filter(Table(ParseJSON(Topic.findings_json).findings), !Boolean(Value.applies_to_template)),
        { type: "Container", separator: true, items: Table(
            { type: "TextBlock", weight: "Bolder", wrap: true,
              color: Switch(Text(Value.severity), "high", "Attention", "medium", "Warning", "Good"),
              text: "[" & Text(Value.severity) & "] " & Text(Value.clause_ref) & " " & Text(Value.category) },
            { type: "TextBlock", weight: "Bolder", wrap: true, text: Text(Value.title) },
            { type: "TextBlock", wrap: true, text: Text(Value.detail) },
            { type: "TextBlock", wrap: true, isSubtle: true, text: "提案: " & Text(Value.suggestion) }
        )}
      )
    },
    { type: "Container", separator: true, items: Table( ...非表示 n 件の注記... ) }
  )
}
```

- **ヘッダと一覧を別々の Container にし、一覧側の `items` に `ForAll` をそのまま入れる** のがポイントです。Power Fx ではテーブル同士を連結できないため、1つの `Table(...)` にヘッダと ForAll を混ぜられません。
- `Table(ParseJSON(...).findings)` で型なし配列をテーブルにし、各フィールドは `Text(Value.xxx)`、真偽値は `Boolean(Value.applies_to_template)` で取り出します。
- フローで `card_json` を作った場合は数式を `ParseJSON(Topic.card_json)` だけにできます。数式モードで型なしオブジェクトをそのままカードとして受け付けるかはテナントのバージョンで確認してください。動かない場合は上の Power Fx 版を使います。
- カード非対応チャネル向けに、同じトピックで `Topic.display_text`(Markdown 表)を送る分岐を残します(`System.Channel` で判定)。

## 5. F2a の送信を受けるトピック(Teams チャネルのみ)

1. 新しいトピック → トリガー「アクティビティを受信したとき」。
2. 条件(Power Fx): `!IsBlank(System.Activity.Value) && (Text(System.Activity.Value.action) = "f2_submit" || Text(System.Activity.Value.action) = "f2_skip")`
3. ノード「アクションを呼び出す」で F2a を呼び、入力に `System.Activity.Value` の各値(`Text(System.Activity.Value.r_A1)` など)と `System.Conversation.Id` を渡す。
4. F2a は Feedback 行を追加し、FeedbackRequest の状態を「回答済」に更新する。
5. 「メッセージを送信」で「ありがとうございました」と返してトピック終了。

このトリガーと `System.Activity.Value` の参照が使えるかは、テナントで新しいトピックを作って確認してください(docs/03 0節の前提チェック 9)。

## 6. 入力データと応答の例

各カードに「何を流し込むと」「何が返るか」の例です。ファイルは templates/sample_*.json にあります。

### 6.1 F3 判定依頼カード

| 段階 | 例 | 内容 |
|---|---|---|
| 入力 | templates/sample_f3_input.json | EvalResult 1件と EvalFinding 3行(一致・余分・欠落)。各行の `crv_findingjson` が Bot の指摘1件で、ここから templates/flow_f3_block.json のブロックが1つ生まれる。欠落の行は `crv_findingjson` が空なので、対応する期待指摘(`_expected_for_display`)を同じ形にして表示する |
| カード | templates/adaptive_card_f3.json | 上の入力から生成されるカードの形(指摘2件分のサンプル) |
| 応答 | templates/sample_f3_response.json | 「アダプティブ カードを投稿して応答を待機」の出力。`data` に `q1_<ID>` などの入力値と `Action.Submit` の `data` が混ざって返る。`responder.email` が判定者 |

応答の読み方(EvalFinding 1行分):

```
ID = replace(crv_evalfindingid, '-', '')          → 0a1b2c3d11114aaa8bbb000000000001
q1  = data?['q1_0a1b2c3d…0001']                    → "valid"       → 人判定 Q1 = 妥当
q2  = data?['q2_0a1b2c3d…0001']                    → "not_applicable" → 人判定 Q2 = 非該当
修正あり = title/desc/sugg のいずれかが Bot 指摘と異なる → ②は title が変わっているので「修正あり」
miss_desc が空でない                               → 「足りない指摘」→ ExpectedFinding 候補を作成
```

例の②(余分)は Q1 = 妥当、Q2 = 該当しない、自信 = 低、内容を修正して送信されています。後処理では ExpectedFinding 候補が「雛形該当(期待) = 非該当、必須/推奨 = 推奨、内容 = 修正後の文」で作られ、自信 = 低 なので校正会の議題になります(docs/07 3.1)。

### 6.2 F2b 本番フィードバックカード

| 段階 | 例 | 内容 |
|---|---|---|
| 入力 | templates/sample_f2_input.json | FeedbackRequest 1行。`crv_findingsjson` は子フロー出力そのもの(5件、非該当 3・該当 2)。`_selected_for_card` は F2b が選んだ表示 2 件(`ask_feedback = true` を優先)と非表示 2 件 |
| カード | templates/adaptive_card_f2.json | 生成されるカードの形 |
| 応答 | templates/sample_f2_response.json | `data` に `r_A1` / `rc_A2` / `h_H2` / `overall` と、送信時に `data` に埋めておいた `shown` / `hidden`(指摘 JSON 付き)が返る。`_resulting_feedback_rows` はそこから作られる Feedback 行 |

応答の読み方:

```
Apply to each(data?['shown'])
  r  = data?[concat('r_',  item()?['aid'])]   → A1: "helpful"  A2: "wrong"
  rc = data?[concat('rc_', item()?['aid'])]   → A2: "severity"
  r が空でなければ Feedback 行(指摘 JSON = item()?['f'], 非表示指摘か = いいえ)
Apply to each(data?['hidden'])
  h  = data?[concat('h_',  item()?['aid'])]   → H1: ""(未回答→行を作らない)  H2: "should_have_shown"
  h が空でなければ Feedback 行(非表示指摘か = はい)   ← H2 が誤抑制の本番ラベルになる
overall が空でなければ 会話単位の行(指摘 JSON = 空)
```

スキップの場合、`data` は `{"action": "f2_skip", "conversation_id": "…", "prompt_version": "…"}` だけです。

### 6.3 レビュー結果カード

| 段階 | 例 | 内容 |
|---|---|---|
| 入力 | templates/sample_result_input.json | 子フローがトピックに返す値。`Topic.findings_json` に templates/sample_f2_input.json の `crv_findingsjson` と同じ内容が入る |
| カード | templates/adaptive_card_result.json | Power Fx(templates/powerfx_result_card.txt)が生成するカードの形。非該当 3 件を表示し、末尾に「非表示 2 件」 |

Power Fx 側の読み方:

```
Table(ParseJSON(Topic.findings_json).findings)                       → 5 行
Filter(…, !Boolean(Value.applies_to_template))                        → 3 行(第12条・第5条・第15条)
Text(Value.title)、Text(Value.severity) …                             → 各 TextBlock へ
CountRows(Filter(…, Boolean(Value.applies_to_template)))              → 2(注記の件数)
```

## 7. 動作確認チェックリスト

| # | 確認 | 期待 |
|---|---|---|
| 1 | F3 を EvalFinding 3件のケースで実行 | 判定者の Teams に3指摘のカードが1通届く。ラジオボタンの初期値が Bot の判定になっている |
| 2 | 何も変えずに送信 | EvalFinding 3行の Q1 = 妥当、Q2 = Bot の判定、修正提案 JSON は空、判定者が入る |
| 3 | 「内容を修正する」を開いてタイトルを変えて送信 | 修正提案 JSON にタイトルが入り、「修正あり」になる |
| 4 | 5日放置(タイムアウトを `PT2M` に短縮して確認) | EvalFinding は更新されず、次回 F3 で再送される |
| 5 | Bot でレビューを実行 | FeedbackRequest 行が「待ち」で作成され、10 分後(確認時は 1 分)に利用者の Teams にカードが届く |
| 6 | カードで指摘1件に「違う」+理由、非表示指摘1件に「表示すべきだった」を選んで送信 | Feedback に3行(表示指摘、非表示指摘、会話単位)、FeedbackRequest が「回答済」 |
| 7 | スキップ | Feedback に評価=スキップの1行、FeedbackRequest が「スキップ」 |
| 8 | Bot でレビュー直後に「第12条について詳しく」と質問 | フォローアップ用トピックが答える。カードは会話を止めない |
| 9 | レビュー結果カード | Teams と Microsoft 365 Copilot の両方で表示される。非表示 n 件の注記がある |
| 10 | 指摘 10 件のケース | カードが 28 KB を超えず表示される。超える場合は分割される |
