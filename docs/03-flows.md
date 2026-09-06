# 03. Power Automate フロー手順書

子フロー1本と評価フロー4本を作成します。すべて同じソリューションに入れ、接続参照と環境変数を使います。

| フロー | トリガー | 役割 |
|---|---|---|
| 子フロー: レビュー実行 | 手動(子フロー) | 契約書テキストを受け取り、Foundry でレビューして指摘 JSON を返す |
| F1 評価実行 | 手動(インスタント) | ゴールデンセット全件を実行し、審査して EvalRun / EvalResult に記録 |
| F2 フィードバック記録 | Copilot Studio から呼び出し | Bot の「役に立った / 違う」を Feedback に記録 |
| F3 人判定依頼 | F1 の末尾から呼び出し(または手動) | 審査が怪しい結果を Teams カードで業務判定者に送る |
| F4 フィードバック・トリアージ | 週次 Recurrence | 未処理 Feedback をカードで提示し、採用分を TestCase 草案にする |

## 0. 前提チェック(最初に1回)

| # | 確認項目 | 方法 |
|---|---|---|
| 1 | Power Automate の「Microsoft Copilot Studio」コネクタに **Execute Agent** / **Execute Agent and wait** が表示される | 新しいフローでアクション検索 |
| 2 | Execute Agent and wait の詳細パラメータに添付ファイル(attachments)がある。ない場合は E2E モードはテキスト送信のみにする | デザイナーで確認 |
| 3 | 評価対象の Bot が公開済みで、認証設定がフローからの呼び出しを許可している | Copilot Studio の設定 > セキュリティ |
| 4 | 既存のカスタムコネクタ(Foundry)がフローから直接呼べる。JSON 出力を返すようプロンプトを変更できる | 既存フローを確認 |
| 5 | Dataverse に docs/02 のテーブルを作成済み | ソリューション |
| 6 | 業務判定者が Teams でアダプティブカードを受け取れる(ゲスト等の制約なし) | テスト送信 |
| 7 | 環境変数を作成: `crv_JudgeModelDeployment`(審査用モデル)、`crv_MustRecallThreshold`(例 0.05)、`crv_TeamsChannelOrUsers`、`crv_TestCaseLibraryUrl` | ソリューション > 環境変数 |

## 1. 子フロー: レビュー実行

既存のレビューフローから「LLM 呼び出し部分」を切り出します。Bot のトピックと F1 の両方がこれを呼びます。

**トリガー**: 手動でフローをトリガー(子フローとして実行を許可)

| 入力 | 型 | 説明 |
|---|---|---|
| contract_text | テキスト | 契約書本文 |
| prompt_version | テキスト | プロンプト版ラベル(記録用。プロンプト本体の切り替えに使ってもよい) |
| contract_type | テキスト | 契約種別(任意) |
| position | テキスト | 当社の立場(任意) |

**アクション**

1. **作成(Compose)「システムプロンプト」** … レビュー用プロンプト。末尾に「出力は次の JSON スキーマに厳密に従うこと」として templates/findings_schema.json の要点を書く。カテゴリ・重要度の語彙は docs/05 と同一にする。
2. **カスタムコネクタ(Foundry)呼び出し** … 既存のアクション。temperature は低め(0〜0.2)。
3. **作成「応答テキスト」** … コネクタ応答から本文を取り出す。
4. **スコープ「JSON パース」**
   - 作成「JSON 文字列」: 応答にコードフェンスが含まれる場合を考慮し、`substring` と `indexOf('{')` / `lastIndexOf('}')` で `{…}` 部分だけを切り出す。
   - JSON の解析(Parse JSON): スキーマは templates/findings_schema.json。
5. **スコープ「JSON パース」の失敗時に実行**(実行条件: 失敗)
   - 作成「パース失敗フラグ」= true。findings は空配列にする。
6. **作成「表示用テキスト」** … `findings` を Select で「【重要度】条項 カテゴリ: タイトル\n 内容\n 提案」に整形して `join`。Bot はこれを表示する。
7. **PowerApps または Flow に応答する**
   - findings_json(テキスト): `string(body('JSONの解析'))`
   - display_text(テキスト)
   - parse_failed(はい/いいえ)
   - model(テキスト): コネクタ応答のモデル名。取れなければ環境変数から

**Bot 側の変更**: トピックのアクションをこの子フローに差し替え、`display_text` を表示する。`findings_json` は表示しないが、F2 の記録用に変数に保持しておく。

## 2. F1 評価実行

**トリガー**: 手動でフローをトリガー

| 入力 | 型 | 説明 |
|---|---|---|
| mode | テキスト(選択肢: direct / e2e) | 既定 direct |
| prompt_version | テキスト | 例 `v1.4.0` |
| tag_filter | テキスト(任意) | 例 `smoke`。空なら承認済み全件 |
| notes | テキスト(任意) | 変更内容の要約 |

**アクション**

1. **作成「実行ID」** … `concat('RUN-', formatDateTime(utcNow(), 'yyyyMMdd-HHmm'))`
2. **Dataverse: 行を追加(EvalRun)** … 実行ID、開始日時、モード、プロンプト版ラベル、タグ絞り込み、状態=実行中、実行者、メモ。
3. **Dataverse: 行を一覧(TestCase)** … フィルター `crv_status eq <承認済の値>`。tag_filter が空でなければ `and contains(crv_tags, '<tag>')` を追加(式で条件文字列を組み立てる)。
4. **変数を初期化**: `mustTotal`, `mustHit`, `shouldTotal`, `shouldHit`, `extraTotal`, `sevMismatch`, `parseFailures`(整数 0)、`latencySum`(浮動小数 0)、`sevMatched`(整数 0)。
5. **Apply to each(TestCase)** … 同時実行制御をオン。direct: 5、e2e: 1〜2(設定で固定。モードにより2つの Apply to each を分けるのが簡単)。
   1. 作成「開始時刻」= `utcNow()`
   2. **条件: mode = direct**
      - はい: **子フローの実行「レビュー実行」** … contract_text = 契約書テキスト、prompt_version、contract_type、position。
      - いいえ(e2e): **Microsoft Copilot Studio: Execute Agent and wait** … Agent = 対象 Bot、メッセージ = 「この契約書をレビューしてください。当社の立場: <position>」、添付 = Wordファイル URL から **SharePoint: ファイル コンテンツの取得(パス指定)** で取得したコンテンツ(添付パラメータの形式はデザイナーで確認)。応答テキストを受け取り、`{…}` 切り出し + JSON の解析で findings を得る(Bot の表示テキストから JSON が取れない場合は、Bot 側で JSON も返すよう「E2E 評価用トピック」を用意する。docs/06 参照)。
   3. 作成「応答時間」= `div(sub(ticks(utcNow()), ticks(outputs('開始時刻'))), 10000000)`
   4. **Dataverse: 行を一覧(ExpectedFinding)** … `_crv_testcaseid_value eq <TestCase の GUID>`
   5. **Select「期待指摘 JSON」** … id, clause_ref, category, severity, required(must/should), description, rationale に整形。
   6. **作成「審査入力」** … docs/04 の入力 JSON を組み立てる(`expected_findings`, `actual_findings`, `contract_type`, `position`)。
   7. **カスタムコネクタ(Foundry)呼び出し「審査」** … システムプロンプト = docs/04 の審査プロンプト、ユーザーメッセージ = 審査入力 JSON。モデルは環境変数 `crv_JudgeModelDeployment`。temperature 0。
   8. **JSON の解析「審査結果」** … スキーマ templates/judge_output_schema.json。`{…}` 切り出しは子フローと同じ。
   9. **集計用の作成**
      - `mustTotalCase` = `length(filter(expected, required == 'must'))` は式で書けないため、Select で `required` だけの配列にしてから `length(filter(...))` 相当を **Filter array** で作る(must の配列、should の配列)。
      - `mustHitCase` = 審査結果 `matches` のうち `expected_required = must` かつ `matched = true` の件数(Filter array → length)。
      - `shouldHitCase` 同様。
      - `extraCase` = `length(body('審査結果')?['extras'])`
      - `sevMismatchCase` = matches のうち `matched = true` かつ `severity_match = false` の件数。
   10. **Dataverse: 行を追加(EvalResult)** … 結果ID = `<実行ID>_<ケースID>`、評価実行、テストケース、生応答、指摘 JSON、JSON 整形失敗、審査結果 JSON、各件数、審査理由(`overall_comment`)、応答時間、人判定=未、人判定依頼済=いいえ。
   11. **変数の値を増やす** … 各集計変数に加算(同時実行中の変数加算は競合し得るため、**集計は Apply to each の外で EvalResult を再集計** する方が安全。次の手順6を採用する場合は手順4・11は省略可)。
6. **集計(推奨方式)** … **Dataverse: 行を一覧(EvalResult)** `_crv_evalrunid_value eq <EvalRun GUID>` を取得し、Select + `sum` 相当は式に無いので、**Apply to each(同時実行オフ)** で変数に加算するか、Dataverse の **集計 FetchXML**(`aggregate="true"` の `sum`)を「行を一覧」の FetchXML クエリで実行する。件数が数百件までなら同時実行オフの Apply to each で十分。
7. **作成「必須再現率」** = `if(equals(variables('mustTotal'),0), 0, div(float(variables('mustHit')), float(variables('mustTotal'))))`。推奨再現率、重要度一致率、平均応答時間も同様。
8. **Dataverse: 行を一覧(EvalRun)「前回」** … `crv_status eq 完了 and crv_mode eq <同モード> and crv_evalrunid ne <今回>`、並べ替え `crv_startedon desc`、上位 1。
9. **作成「前回比」** = 今回の必須再現率 − 前回の `crv_mustrecall`(前回が無ければ 0)。
10. **ゲート判定**
    - 不合格条件: 前回比 < −`crv_MustRecallThreshold`、または 重要ケース(`crv_iscritical = はい`)の EvalResult に `musthit < musttotal` がある、または JSON 整形失敗数 > 0。
    - それ以外は合格。
11. **Dataverse: 行を更新(EvalRun)** … 終了日時、状態=完了、各指標、前回比、ゲート判定、ケース数、モデル。
12. **Teams: チャットまたはチャネルでメッセージを投稿** … 実行ID、モード、プロンプト版、必須再現率(前回比)、余分指摘数、ゲート判定、不合格理由、EvalRun へのリンク。
13. **子フローの実行「F3 人判定依頼」** … 入力 = EvalRun GUID。
14. **失敗時**(フロー全体を Try スコープで囲む): 状態=失敗で EvalRun を更新し、Teams に通知。

## 3. F2 フィードバック記録

**トリガー**: Copilot Studio から呼び出す(「エージェントがフローを呼び出したとき」)

| 入力 | 説明 |
|---|---|
| conversation_id | `System.Conversation.Id` |
| rating | `helpful` / `wrong` |
| reason | 利用者の入力(任意) |
| bot_response | 直前の表示テキスト |
| input_ref | ファイル名など。契約書本文は渡さない |

**アクション**

1. **Dataverse: 行を追加(Feedback)** … フィードバックID = `concat(conversation_id, '-', formatDateTime(utcNow(),'HHmmss'))`、日時、評価、理由、Bot 応答、入力参照、トリアージ状態=未処理。
2. 応答を返す(受付完了)。

**Bot 側**: レビュー結果表示の直後に「この結果は役に立ちましたか?」の質問(選択肢: 役に立った / 違う)を出し、「違う」の場合は理由を1問だけ聞いてから F2 を呼ぶ。理由入力はスキップ可にする。

## 4. F3 人判定依頼

**トリガー**: 手動(子フロー)。入力 `evalrun_id`(GUID)

**アクション**

1. **Dataverse: 行を一覧(EvalResult)** … `_crv_evalrunid_value eq <GUID> and crv_reviewrequested eq false and (crv_extracount gt 0 or crv_musthit lt crv_musttotal or crv_parsefailed eq true)`。件数上限は 20(それ以上なら重要ケース優先で切り、Teams 要約に「残り n 件」と書く)。
2. **Apply to each(同時実行オフ)**
   1. **Dataverse: 行を更新** … 人判定依頼済 = はい。
   2. **Teams: アダプティブ カードを投稿して応答を待機**(対象: 業務判定者。2名以上に送る場合はチャネル投稿にする)
      - 本文: ケースID、タイトル、期待指摘(必須のみ、箇条書き)、Bot の指摘(タイトルのみ、箇条書き)、審査の判定(欠落 n 件・余分 n 件)と審査理由。
      - 入力: `verdict`(選択: 審査に同意 / 審査に不同意 / 判定不能)、`comment`(複数行、任意)。
      - タイムアウト: 5 日(SLA 2 営業日 + 余裕)。
   3. **Dataverse: 行を更新(EvalResult)** … 人判定、人コメント、判定者(応答者)、判定日時。タイムアウト時は更新せず終了。
3. 送信件数と残件数を Teams の要約に追記(F1 の投稿への返信でもよい)。

## 5. F4 フィードバック・トリアージ

**トリガー**: Recurrence(週1回、校正会の前日)

**アクション**

1. **Dataverse: 行を一覧(Feedback)** … `crv_triage eq 未処理`、上位 20、古い順。
2. **Apply to each(同時実行オフ)**
   1. **Teams: アダプティブ カードを投稿して応答を待機**(業務オーナーまたは判定者)
      - 本文: 日時、評価、理由、Bot 応答(先頭 1,000 文字)、入力参照。
      - 入力: `decision`(選択: テストケース化 / 見送り / 重複)、`expected_note`(「本来はこう指摘すべき」を1〜3行)。
   2. **条件: decision = テストケース化**
      - **Dataverse: 行を追加(TestCase)** … ケースID = 次番(`concat('TC-', formatNumber(add(<既存最大番号>,1), '0000'))` は取得が面倒なため、暫定 `TC-FB-<フィードバックID>` とし、仕上げ時に振り直す)、タイトル = 理由の先頭 60 文字、出所 = 本番フィードバック由来、状態 = 草案、担当(業務) = 応答者、契約書テキスト = 空(機密のため利用者に再提供を依頼)、備考 = 「Feedback <ID> 由来。契約書テキストの登録が必要」。
      - **Dataverse: 行を追加(ExpectedFinding)** … 期待指摘ID = `<ケースID>-01`、条項参照 = 「未確認」、カテゴリ = その他、重要度 = medium、必須/推奨 = 必須、期待する指摘内容 = `expected_note`、由来 = 本番フィードバック。
      - **Dataverse: 行を更新(Feedback)** … トリアージ状態、紐付けテストケース、担当。
   3. **それ以外**: Feedback のトリアージ状態と担当を更新。
3. Teams に「今週のトリアージ結果: 採用 n / 見送り n / 重複 n、仕上げ待ち TestCase 一覧」を投稿。

## 6. TestCase 登録の補助(任意のフロー F0)

SharePoint ドキュメントライブラリ(環境変数 `crv_TestCaseLibraryUrl`)に Word を置くと、既存 Bot と同じ Word → テキスト変換を実行して TestCase の契約書テキスト列に保存するフロー。
「ファイルが作成されたとき」トリガー → 変換 → TestCase 作成(状態=草案)→ 担当に Teams 通知。
変換の妥当性チェックとして、`第\d+条` の出現を正規表現相当(`split` と `length`)で数え、Word の条数と大きく違う場合は備考に警告を書く。

## 7. 実装時の注意

- **Apply to each の同時実行と変数**: 並列中の「変数の値を増やす」は競合します。集計は Apply to each の外で EvalResult を再集計してください(F1 手順6)。
- **JSON 切り出し**: LLM 応答にコードフェンスや前置きが混ざる前提で `{` から `}` を切り出す。それでも失敗した行は `parse_failed = true` として記録し、指標で追う。
- **スロットリング**: E2E モードは Bot 側のレート制限に当たりやすい。並列 1 で開始し、問題なければ 2 にする。
- **機密**: Feedback には契約書本文を保存しない。TestCase の契約書テキストは匿名化済みか合成のみ登録する。
- **接続参照**: Foundry のカスタムコネクタ、Dataverse、Teams、SharePoint、Copilot Studio を接続参照にし、環境移送時に差し替える。
- **タイムアウト**: 「アダプティブ カードを投稿して応答を待機」は 30 日が上限。F3 は 5 日、F4 は 3 日で設定し、未応答は次回に再送する。
