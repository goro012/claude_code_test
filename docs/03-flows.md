# 03. Power Automate フロー手順書

子フロー1本と評価フロー4本を作成します。すべて同じソリューションに入れ、接続参照と環境変数を使います。

| フロー | トリガー | 役割 |
|---|---|---|
| 子フロー: レビュー実行 | 手動(子フロー) | 契約書テキストを受け取り、段階1(レビュー)と段階2(雛形照合)を実行して指摘 JSON を返す |
| F1 評価実行 | 手動(インスタント) | ゴールデンセット全件を実行し、段階1を審査、段階2をフラグ比較して EvalRun / EvalResult / EvalFinding に記録 |
| F2 フィードバック記録 | Copilot Studio から呼び出し | Bot の「役に立った / 違う」を Feedback に記録 |
| F3 人判定依頼 | F1 の末尾から呼び出し(または手動) | 指摘単位で Q1(妥当か)/ Q2(雛形にも該当するか)を Teams カードで業務判定者に聞き、回答を ExpectedFinding に反映 |
| F4 フィードバック・トリアージ | 週次 Recurrence | 未処理 Feedback をカードで提示し、採用分を TestCase 草案にする |

## 0. 前提チェック(最初に1回)

| # | 確認項目 | 方法 |
|---|---|---|
| 1 | Power Automate の「Microsoft Copilot Studio」コネクタに **Execute Agent** / **Execute Agent and wait** が表示される | 新しいフローでアクション検索 |
| 2 | Execute Agent and wait の詳細パラメータに添付ファイル(attachments)がある。ない場合は E2E モードはテキスト送信のみにする | デザイナーで確認 |
| 3 | 評価対象の Bot が公開済みで、認証設定がフローからの呼び出しを許可している | Copilot Studio の設定 > セキュリティ |
| 4 | 既存のカスタムコネクタ(Foundry)がフローから直接呼べる。段階1・段階2とも JSON 出力を返すようプロンプトを変更できる | 既存フローを確認 |
| 5 | 段階2のプロンプトを「該当する指摘を削除する」から「全件に判定フラグ・雛形条項・引用・理由を付けて返す」に変更できる | 既存フローを確認 |
| 6 | Dataverse に docs/02 のテーブルを作成済み。Template に契約種別 × 立場ごとの有効な雛形が登録済み | ソリューション |
| 7 | 業務判定者が Teams でアダプティブカードを受け取れる | テスト送信 |
| 8 | 環境変数を作成: `crv_JudgeModelDeployment`(審査用モデル)、`crv_FinalRecallThreshold`(例 0.05)、`crv_TeamsChannelOrUsers`、`crv_TestCaseLibraryUrl`、`crv_MaxFindingsPerCard`(例 10) | ソリューション > 環境変数 |

## 1. 子フロー: レビュー実行

既存のレビューフローから「LLM 呼び出し部分(段階1・段階2)」を切り出します。Bot のトピックと F1 の両方がこれを呼びます。

**トリガー**: 手動でフローをトリガー(子フローとして実行を許可)

| 入力 | 型 | 説明 |
|---|---|---|
| contract_text | テキスト | 契約書本文 |
| prompt_version | テキスト | プロンプト版ラベル(記録用) |
| contract_type | テキスト | 契約種別 |
| position | テキスト | 当社の立場 |
| template_id | テキスト(任意) | 雛形の GUID。空なら契約種別と立場で Template を検索 |

**アクション**

1. **雛形の取得**
   - 条件: template_id が空でない → **Dataverse: ID で行を取得(Template)**。
   - 空 → **Dataverse: 行を一覧(Template)** `crv_contracttype eq <種別> and crv_position eq <立場> and crv_status eq 有効`、上位 1。0件ならエラーで終了(段階2ができないため)。
   - 作成「雛形テキスト」「雛形ID」「雛形版」。
2. **段階1: レビュー**
   - 作成「段階1 システムプロンプト」… レビュー用プロンプト。末尾に「出力は次の JSON スキーマに厳密に従うこと」として `findings` の段階1項目(clause_ref, category, severity, title, detail, suggestion)を書く。カテゴリ・重要度の語彙は docs/05 と同一。
   - **カスタムコネクタ(Foundry)呼び出し** … temperature 0〜0.2。
   - スコープ「段階1 JSON パース」… 応答から `{…}` を切り出し(`substring` + `indexOf('{')` / `lastIndexOf('}')`)、**JSON の解析**。失敗時は `parse_failed = true`、findings = 空配列にして手順4へ。
   - **Select「段階1指摘に ID 付与」** … `A1, A2, …`(`concat('A', add(indexOf(...),1))` は使えないため、Apply to each + 変数カウンタ、または `range(0, length(findings))` を Select の元にして `findings[item()]` を参照する方法で付与)。
3. **段階2: 雛形照合**
   - 作成「段階2 システムプロンプト」… 内容の要点:
     - 入力は「雛形契約書テキスト」と「段階1指摘一覧(id 付き)」。
     - 各指摘について、雛形契約書にも **同じ状態(同じ論点で同じ条件)** が存在するかを判定する。存在すれば `applies_to_template = true`。
     - 雛形に同趣旨の条項があっても、緩和条項や上限など指摘の趣旨を解消する内容が付いていれば `false`。
     - 全件に `id`, `applies_to_template`, `template_clause_ref`(無ければ「該当条項なし」), `template_excerpt`(雛形の該当箇所を原文のまま 200 字以内で引用。無ければ空), `template_reason`(1文)を付けて返す。**削除しない。**
     - 出力は JSON のみ。
   - **カスタムコネクタ(Foundry)呼び出し** … temperature 0。
   - スコープ「段階2 JSON パース」… `{…}` 切り出し + JSON の解析。失敗時は全指摘を `applies_to_template = false`、`template_reason = "段階2失敗"` として扱い、`parse_failed_s2 = true` を記録(最終出力に全件出す方が、黙って消えるより安全)。
4. **結合** … Select で段階1指摘に段階2の項目を id で結合(段階2の配列を `filter` で id 一致させて先頭を取る)。結果を templates/findings_schema.json に合わせた `findings` 配列にする。
5. **作成「表示用テキスト」** … `findings` を **Filter array** `applies_to_template = false` → Select で「【重要度】条項 カテゴリ: タイトル\n 内容\n 提案」→ `join`。0件なら「雛形契約書と同水準の内容であり、追加の指摘はありません」。
6. **PowerApps または Flow に応答する**
   - findings_json(テキスト): 結合後の JSON 文字列
   - display_text(テキスト)
   - parse_failed(はい/いいえ): 段階1失敗
   - parse_failed_s2(はい/いいえ): 段階2失敗
   - model(テキスト)
   - template_id、template_version(テキスト)

**Bot 側の変更**: トピックのアクションをこの子フローに差し替え、`display_text` を表示する。`findings_json` は表示しないが F2 の記録用に変数に保持する。

## 2. F1 評価実行

**トリガー**: 手動でフローをトリガー

| 入力 | 型 | 説明 |
|---|---|---|
| mode | テキスト(選択肢: direct / e2e) | 既定 direct |
| prompt_version | テキスト | 例 `v1.4.0` |
| tag_filter | テキスト(任意) | 例 `smoke`。空なら承認済み全件 |
| notes | テキスト(任意) | 変更内容の要約 |
| review_scope | テキスト(選択肢: disagreements / all / none) | F3 に渡す人判定の範囲。既定 disagreements |

**アクション**

1. **作成「実行ID」** … `concat('RUN-', formatDateTime(utcNow(), 'yyyyMMdd-HHmm'))`
2. **Dataverse: 行を追加(EvalRun)** … 実行ID、開始日時、モード、プロンプト版ラベル、タグ絞り込み、状態=実行中、実行者、メモ。
3. **Dataverse: 行を一覧(TestCase)** … `crv_status eq 承認済`。tag_filter が空でなければ `and contains(crv_tags, '<tag>')` を追加。
4. **Apply to each(TestCase)** … 同時実行制御をオン。direct: 5、e2e: 1〜2(モードにより Apply to each を2つに分ける)。
   1. 作成「開始時刻」= `utcNow()`
   2. **条件: mode = direct**
      - はい: **子フローの実行「レビュー実行」** … contract_text、prompt_version、contract_type、position、template_id = TestCase の雛形参照。
      - いいえ(e2e): **Microsoft Copilot Studio: Execute Agent and wait** … Agent = 対象 Bot、メッセージ = 「[EVAL] この契約書をレビューしてください。契約種別: <種別> 当社の立場: <立場>」、添付 = Wordファイル URL から **SharePoint: ファイル コンテンツの取得(パス指定)** で取得したコンテンツ。応答から `{…}` を切り出して JSON の解析(E2E 評価用トピックが `findings_json` を返す前提。docs/06 の 8 節)。
   3. 作成「応答時間」= `div(sub(ticks(utcNow()), ticks(outputs('開始時刻'))), 10000000)`
   4. **Dataverse: 行を一覧(ExpectedFinding)** … `_crv_testcaseid_value eq <TestCase GUID>`
   5. **Select「期待指摘 JSON」** … id, clause_ref, category, severity, required(must/should), description, rationale, template_applicable(該当→`true`、非該当→`false`、未確認→`null`)。
   6. **作成「審査入力」** … docs/04 の入力 JSON(`expected_findings`, `actual_findings`, `contract_type`, `position`)。actual_findings には段階2項目も含める(審査 LLM は照合に使わないが、記録の整合のため)。
   7. **カスタムコネクタ(Foundry)呼び出し「審査」** … システムプロンプト = docs/04、モデル = `crv_JudgeModelDeployment`、temperature 0。
   8. **JSON の解析「審査結果」** … templates/judge_output_schema.json。
   9. **EvalResult 行を追加** … 結果ID、評価実行、テストケース、雛形、生応答、指摘 JSON、JSON 整形失敗(段階1 or 段階2)、審査結果 JSON、審査理由、応答時間、人判定依頼済=いいえ。件数列はこの時点では 0 にしておき、手順 10 の後で更新する。
   10. **Apply to each(審査結果 `matches`)** … 同時実行オフ。
       - 対応する期待指摘(`expected_id` で ExpectedFinding を検索)を取得。
       - **matched = true** の場合:
         - Bot 指摘 = `actual_findings` から `matched_actual_id` で取り出す。
         - 段階2 Bot 判定 = Bot 指摘の `applies_to_template`。段階2 期待 = 期待指摘の雛形該当。
         - 段階2 一致 = 期待が未確認 → 評価不能 / Bot=該当 かつ 期待=非該当 → **誤抑制** / Bot=非該当 かつ 期待=該当 → **誤通過** / それ以外 → 一致。
         - **EvalFinding 行を追加**(種別 = 一致、Bot 指摘ID、Bot 指摘 JSON、対応する期待指摘、段階2 各列、重要度一致 = `severity_match`、審査理由 = `reason`)。
       - **matched = false** の場合: **EvalFinding 行を追加**(種別 = 欠落、対応する期待指摘、段階2 期待、審査理由)。
   11. **Apply to each(審査結果 `extras`)** … **EvalFinding 行を追加**(種別 = 余分、Bot 指摘ID、Bot 指摘 JSON、段階2 Bot 判定、段階2 一致 = 評価不能)。
   12. **ケース単位の件数を計算して EvalResult を更新** … 手順 10・11 で作った EvalFinding を **Dataverse: 行を一覧** `_crv_evalresultid_value eq <EvalResult GUID>` で取り直し、Filter array + `length()` で次を求める。
       - 段階1 必須総数 = 期待指摘のうち must の数、段階1 必須一致数 = 種別=一致 かつ 期待が must
       - 余分指摘数 = 種別=余分
       - 段階2 評価対象数 = 種別=一致 かつ 段階2一致 ≠ 評価不能、段階2 一致数 = 段階2一致 = 一致
       - 誤抑制数、誤通過数
       - 最終出力 必須総数 = 期待指摘のうち must かつ 雛形該当=非該当、最終出力 必須一致数 = 種別=一致 かつ 期待が must かつ 期待=非該当 かつ Bot=非該当
       - 重要度不一致数
5. **実行全体の集計** … **Dataverse: 行を一覧(EvalResult)** `_crv_evalrunid_value eq <EvalRun GUID>` を取得し、**Apply to each(同時実行オフ)** で変数に加算(件数が数百件までならこれで十分。超える場合は FetchXML の `aggregate` を使う)。
6. **率の計算** … 各率は `if(equals(分母,0), 0, div(float(分子), float(分母)))`。最終出力 必須再現率、段階1 必須再現率、段階2 一致率、誤抑制率(分母 = 期待が非該当の一致指摘数)、誤通過率(分母 = 期待が該当の一致指摘数)、重要度一致率、平均応答時間。
7. **Dataverse: 行を一覧(EvalRun)「前回」** … `crv_status eq 完了 and crv_mode eq <同モード> and crv_evalrunid ne <今回>`、`crv_startedon desc`、上位 1。
8. **作成「前回比」** = 今回の最終出力 必須再現率 − 前回の値(前回が無ければ 0)。
9. **ゲート判定** … 不合格条件のいずれか:
   - 前回比 < −`crv_FinalRecallThreshold`
   - 重要ケース(`crv_iscritical = はい`)の EvalResult に「最終出力 必須一致数 < 総数」または「誤抑制数 > 0」がある
   - JSON 整形失敗数 > 0
10. **Dataverse: 行を更新(EvalRun)** … 終了日時、状態=完了、全指標、前回比、ゲート判定、ケース数、モデル。
11. **Teams: メッセージを投稿** … 実行ID、モード、プロンプト版、**最終出力 必須再現率(前回比)**、段階1 必須再現率、誤抑制数 / 誤通過数、余分指摘数、ゲート判定と不合格理由、EvalRun へのリンク。
12. **条件: review_scope ≠ none** → **子フローの実行「F3 人判定依頼」** … 入力 = EvalRun GUID、scope = review_scope。
13. **失敗時**(フロー全体を Try スコープで囲む): 状態=失敗で EvalRun を更新し、Teams に通知。

## 3. F2 フィードバック記録

**トリガー**: Copilot Studio から呼び出す(「エージェントがフローを呼び出したとき」)

| 入力 | 説明 |
|---|---|
| conversation_id | `System.Conversation.Id` |
| rating | `helpful` / `wrong` |
| reason | 利用者の入力(任意) |
| bot_response | 直前の表示テキスト(最終出力) |
| input_ref | ファイル名など。契約書本文は渡さない |

**アクション**

1. **Dataverse: 行を追加(Feedback)** … フィードバックID = `concat(conversation_id, '-', formatDateTime(utcNow(),'HHmmss'))`、日時、評価、理由、Bot 応答、入力参照、トリアージ状態=未処理。
2. 応答を返す(受付完了)。

**Bot 側**: 最終出力の直後に「この結果は役に立ちましたか?」(役に立った / 違う)を出し、「違う」の場合は理由を1問だけ聞いてから F2 を呼ぶ。理由は「指摘が違う」「必要な指摘が無い」「雛形と同じ内容なのに指摘された」「その他」の選択肢+自由記入にすると F4 のトリアージが速くなります。

## 4. F3 人判定依頼(指摘単位の Q1 / Q2)

**トリガー**: 手動(子フロー)。入力 `evalrun_id`(GUID)、`scope`(`disagreements` / `all`)

**対象の決め方**

| scope | 対象の EvalFinding | 用途 |
|---|---|---|
| disagreements(定常) | 種別 = 余分、種別 = 欠落、段階2一致 = 誤抑制 / 誤通過、段階2 期待 = 未確認 | 割れたところだけ聞く |
| all(立ち上げ期・新規ケース) | その EvalRun の全 EvalFinding | 回答をそのまま ExpectedFinding にする |

**アクション**

1. **Dataverse: 行を一覧(EvalResult)** … `_crv_evalrunid_value eq <GUID> and crv_reviewrequested eq false`。重要ケース優先で並べる(TestCase を展開して `crv_iscritical desc`)。
2. **Apply to each(EvalResult、同時実行オフ)**
   1. **Dataverse: 行を一覧(EvalFinding)** … `_crv_evalresultid_value eq <EvalResult GUID> and crv_reviewrequested eq false` に scope の条件を加える。0 件なら次へ。
   2. 上限 `crv_MaxFindingsPerCard` 件(既定 10)ずつに分割(`chunk()` 関数)。超過分は次回に回す。
   3. **作成「カード JSON」** … Adaptive Card 1.5。構成:
      - ヘッダ: ケースID、タイトル、契約種別、当社の立場、雛形ID(版)、実行ID。
      - 指摘ごとに Container:
        - 種別バッジ(一致 / 余分 / 欠落)、カテゴリ、重要度、対象契約書の条項参照。
        - **Bot の指摘**: タイトル、内容、提案(欠落の場合は代わりに **期待していた指摘** を表示)。
        - **雛形の該当箇所**: `template_clause_ref` と `template_excerpt`(無ければ「該当条項なし」)。
        - **Bot の段階2判定**: 「雛形にも該当する(表示しない)」/「雛形には該当しない(表示する)」+ `template_reason`。
        - `Input.ChoiceSet` id = `q1_<評価指摘ID>`: **Q1 この指摘は妥当ですか?** 妥当 / 不当 / 判定不能(欠落の場合は「この期待指摘は今も必要ですか?」必要 / 不要 / 判定不能 と読み替え)。
        - `Input.ChoiceSet` id = `q2_<評価指摘ID>`: **Q2 雛形契約書にも同じ内容が当てはまりますか?** 該当する / 該当しない / 判定不能。
        - `Input.Text` id = `c_<評価指摘ID>`: コメント(任意、1行)。
      - 末尾: `Action.Submit` 「送信」。
   4. **Dataverse: 行を更新(EvalFinding)** … 対象行の人判定依頼済 = はい。
   5. **Teams: アダプティブ カードを投稿して応答を待機**(業務判定者。チャネル投稿にすると複数名のうち先に答えた人の回答になる。2名の判定を取りたい場合はユーザーごとに投稿し、EvalFinding ではなく別テーブルに記録する。初期は1名回答で十分)。タイムアウト 5 日。
   6. **Apply to each(対象 EvalFinding)** … 回答 JSON から `q1_<ID>` `q2_<ID>` `c_<ID>` を取り出し、**Dataverse: 行を更新(EvalFinding)** … 人判定 Q1、人判定 Q2、人コメント、判定者、判定日時。タイムアウト時は更新しない。
   7. **Dataverse: 行を更新(EvalResult)** … 人判定依頼済 = はい。
3. **後処理: 人判定を期待指摘に反映**(同じフロー内、または別の子フロー「人判定反映」)
   - **Dataverse: 行を一覧(EvalFinding)** … この EvalRun で `Q1 ≠ 未 and 期待指摘へ反映済 = いいえ`。
   - 種別 = **余分** かつ Q1 = 妥当 → **ExpectedFinding 行を追加**(状態は TestCase 側で管理するため、由来 = 人判定から自動作成、条項参照・カテゴリ・重要度・内容・提案は Bot 指摘 JSON から転記、必須/推奨 = 推奨(校正会で必須に昇格)、雛形該当(期待) = Q2 が該当→該当 / 該当しない→非該当 / 判定不能→未確認、雛形条項参照 = `template_clause_ref`)。TestCase の状態を「雛形要再確認」にしない(未確認のときだけする)。
   - 種別 = **一致** かつ Q2 ≠ 未 かつ Q2 ≠ 判定不能 かつ Q2 ≠ 段階2 期待 → 期待指摘の雛形該当を **直接は変えず**、人コメントに「Q2 不一致」を追記し校正会議題にする(期待値の変更は校正会の決定で行う)。
   - 種別 = **欠落** かつ Q1(必要か) = 不要 → 期待指摘を退役候補として校正会議題にする。
   - 種別 = **余分** かつ Q1 = 不当 → 件数を EvalRun のメモに「段階1 不当指摘 n 件(カテゴリ別)」として追記。
   - 処理した行の 期待指摘へ反映済 = はい。
4. Teams に「送信 n 件、残り n 件、自動作成した期待指摘 n 件、校正会議題 n 件」を投稿。

## 5. F4 フィードバック・トリアージ

**トリガー**: Recurrence(週1回、校正会の前日)

**アクション**

1. **Dataverse: 行を一覧(Feedback)** … `crv_triage eq 未処理`、上位 20、古い順。
2. **Apply to each(同時実行オフ)**
   1. **Teams: アダプティブ カードを投稿して応答を待機**(業務オーナーまたは判定者)
      - 本文: 日時、評価、理由、Bot 応答(先頭 1,000 文字)、入力参照。
      - 入力: `decision`(テストケース化 / 見送り / 重複)、`expected_note`(「本来はこう指摘すべき」1〜3行)、`template_note`(「雛形にも当てはまるか」該当 / 非該当 / 不明)。
   2. **条件: decision = テストケース化**
      - **TestCase 行を追加** … ケースID = `TC-FB-<フィードバックID>`(仕上げ時に振り直す)、タイトル = 理由の先頭 60 文字、契約種別・立場 = 不明なら「その他 / 対等」で仮置き、雛形 = 種別・立場から有効な Template を検索(無ければ空のまま状態 = 雛形要再確認)、出所 = 本番フィードバック由来、状態 = 草案、担当(業務) = 応答者、契約書テキスト = 空(利用者に匿名化済みテキストの再提供を依頼)、備考 = 「Feedback <ID> 由来」。
      - **ExpectedFinding 行を追加** … 期待指摘ID = `<ケースID>-01`、条項参照 = 「未確認」、カテゴリ = その他、重要度 = medium、必須/推奨 = 必須、期待する指摘内容 = `expected_note`、雛形該当(期待) = `template_note`(不明→未確認)、由来 = 本番フィードバック。
      - **Feedback 行を更新** … トリアージ状態、紐付けテストケース、担当。
   3. **それ以外**: Feedback のトリアージ状態と担当を更新。
3. Teams に「今週のトリアージ結果: 採用 n / 見送り n / 重複 n、仕上げ待ち TestCase 一覧」を投稿。

## 6. TestCase 登録の補助(任意のフロー F0)

SharePoint ドキュメントライブラリ(環境変数 `crv_TestCaseLibraryUrl`)に Word を置くと、既存 Bot と同じ Word → テキスト変換を実行して TestCase の契約書テキスト列に保存するフロー。
「ファイルが作成されたとき」トリガー → 変換 → TestCase 作成(状態=草案、雛形 = ファイル名の契約種別・立場から検索)→ 担当に Teams 通知。
変換の妥当性チェックとして `第\d+条` の出現数を数え、Word の条数と大きく違う場合は備考に警告を書く。

同じフローを Template の登録にも流用できます(ライブラリのフォルダで分ける)。

## 7. 実装時の注意

- **Apply to each の同時実行と変数**: 並列中の「変数の値を増やす」は競合します。集計は EvalFinding / EvalResult を取り直して行ってください(F1 手順 4-12、5)。
- **JSON 切り出し**: LLM 応答にコードフェンスや前置きが混ざる前提で `{` から `}` を切り出す。失敗した行は `parse_failed` として記録し、指標で追う。
- **段階2の失敗は安全側に**: 段階2の JSON が壊れたときは全件「非該当」として表示する(黙って消すより見える方が安全)。EvalRun の JSON 整形失敗にも数える。
- **カードの入力 ID**: Adaptive Card の入力 id に使えるのは英数字と `_` `-`。評価指摘ID に日本語や記号が入る場合は GUID を使う。
- **カードの上限**: Teams のカードは 28 KB 程度が上限。`template_excerpt` は 200 字、`detail` は 300 字で切って載せる。
- **スロットリング**: E2E モードは Bot 側のレート制限に当たりやすい。並列 1 で開始し、問題なければ 2 にする。
- **機密**: Feedback には契約書本文を保存しない。TestCase・Template のテキストは匿名化済みか合成、または社内雛形のみ登録する。
- **接続参照**: Foundry のカスタムコネクタ、Dataverse、Teams、SharePoint、Copilot Studio を接続参照にし、環境移送時に差し替える。
- **タイムアウト**: 「アダプティブ カードを投稿して応答を待機」は 30 日が上限。F3 は 5 日、F4 は 3 日で設定し、未応答は次回に再送する(人判定依頼済フラグは応答時に立てる設計にすると再送が容易)。
