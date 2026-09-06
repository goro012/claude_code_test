# 02. Dataverse テーブル定義

スキーマ名の接頭辞は `crv_`(Contract Review)としています。環境の発行者接頭辞に合わせて読み替えてください。
すべてソリューションに含め、環境間で移送できるようにします。

## テーブル一覧と関係

```
Template 1 ─── * TestCase 1 ─── * ExpectedFinding
                     │
                     └── * EvalResult * ─── 1 EvalRun
                              │
                              └── * EvalFinding * ─── 0..1 ExpectedFinding
TestCase 1 ─── * Feedback(採用時のみ紐付け)
```

## 1. Template(雛形契約書) `crv_template`

法務部チェック済みの雛形契約書。段階2(雛形照合)の基準です。

| 表示名 | スキーマ名 | 型 | 必須 | 説明 |
|---|---|---|---|---|
| 雛形ID | `crv_name`(主列) | テキスト(100) | ○ | 例 `TPL-業務委託-受注側-v3` |
| 契約種別 | `crv_contracttype` | 選択肢 | ○ | TestCase と同じ選択肢 |
| 当社の立場 | `crv_position` | 選択肢 | ○ | 発注側 / 受注側 / 対等 |
| 版 | `crv_version` | テキスト(20) | ○ | 例 `v3`。法務の版番号に合わせる |
| 法務承認日 | `crv_approvedon` | 日付 | ○ | |
| 雛形テキスト | `crv_templatetext` | 複数行テキスト(memo, 最大長) | ○ | 段階2のプロンプトに渡す本文 |
| Wordファイル URL | `crv_fileurl` | URL | | 原本 |
| 状態 | `crv_status` | 選択肢 | ○ | 有効 / 退役 |
| 備考 | `crv_notes` | 複数行テキスト | | 前版からの主な変更点 |

契約種別 × 立場 で「有効」は常に1件にします(Bot 経由の検索を一意にするため)。

## 2. TestCase(テストケース) `crv_testcase`

ゴールデンセットの入力1件。契約書1本に対応します。

| 表示名 | スキーマ名 | 型 | 必須 | 説明 |
|---|---|---|---|---|
| ケースID | `crv_name`(主列) | テキスト(100) | ○ | 例 `TC-0001`。人が読める一意キー |
| タイトル | `crv_title` | テキスト(200) | ○ | 例「業務委託契約(受託側)・損害賠償上限なし」 |
| 契約種別 | `crv_contracttype` | 選択肢 | ○ | 業務委託 / 秘密保持 / 売買 / 賃貸借 / ライセンス / その他 |
| 当社の立場 | `crv_position` | 選択肢 | ○ | 発注側 / 受注側 / 対等 |
| 雛形 | `crv_templateid` | 参照(Template) | ○ | 段階2の基準。期待指摘の「雛形該当」はこの版に対する判断 |
| 契約書テキスト | `crv_contracttext` | 複数行テキスト(memo, 最大長) | ○ | 直接モードの入力。Word から抽出済みの本文 |
| Wordファイル URL | `crv_fileurl` | URL | | E2E モードの入力。SharePoint ドキュメントライブラリ上のパス |
| 出所 | `crv_source` | 選択肢 | ○ | 実契約書(匿名化済) / 合成 / 本番フィードバック由来 |
| 状態 | `crv_status` | 選択肢 | ○ | 草案 / 承認済 / 退役 / 雛形要再確認 |
| タグ | `crv_tags` | テキスト(200) | | `smoke`, `critical`, `regression-only`, `fewshot` などカンマ区切り。F1 の絞り込みに使う。`fewshot` = 期待指摘をプロンプトの例示に使ったケース。ゲート指標から除外(docs/08 0節) |
| 重要ケース | `crv_iscritical` | はい/いいえ | ○ | はい なら必須指摘の欠落または誤抑制1件でリリースゲート不合格 |
| 担当(業務) | `crv_owner_business` | ユーザー参照 | | ExpectedFinding を仕上げる業務判定者 |
| 備考 | `crv_notes` | 複数行テキスト | | 匿名化の方法、意図している論点など |

## 3. ExpectedFinding(期待指摘) `crv_expectedfinding`

TestCase 1件に対して「段階1で指摘すべき事項」を1行ずつ持ち、各行に「雛形にも該当するか(= 段階2で抑制されるべきか)」の期待値を持ちます。

| 表示名 | スキーマ名 | 型 | 必須 | 説明 |
|---|---|---|---|---|
| 期待指摘ID | `crv_name`(主列) | テキスト(100) | ○ | 例 `TC-0001-01` |
| テストケース | `crv_testcaseid` | 参照(TestCase) | ○ | |
| 条項参照 | `crv_clauseref` | テキスト(50) | ○ | 例「第12条」「第5条2項」「該当条項なし」 |
| カテゴリ | `crv_category` | 選択肢 | ○ | docs/05 のカテゴリ語彙 |
| 重要度 | `crv_severity` | 選択肢 | ○ | high / medium / low |
| 必須/推奨 | `crv_required` | 選択肢 | ○ | 必須(must) / 推奨(should)。再現率は必須のみで計算 |
| 期待する指摘内容 | `crv_description` | 複数行テキスト | ○ | 段階1で伝えるべき内容。文言一致は求めない |
| 根拠 | `crv_rationale` | 複数行テキスト | | なぜ問題か |
| 雛形該当(期待) | `crv_templateapplicable` | 選択肢 | ○ | 該当 / 非該当 / 未確認。**該当 = 段階2で抑制されるのが正しい**、非該当 = 最終出力に出るべき |
| 雛形条項参照(期待) | `crv_templateclauseref` | テキスト(50) | | 雛形側の該当条項。該当/非該当いずれでも記録(人判定カードに表示) |
| 由来 | `crv_origin` | 選択肢 | | 業務部門作成 / Bot 出力を修正 / 人判定(Q1/Q2)から自動作成 / 判定者追加 / LLM 下書き / 過去資産 / 本番フィードバック |
| 状態 | `crv_state` | 選択肢 | ○ | 候補 / 承認済 / 退役。**再現率の分母は承認済のみ**(docs/07) |
| 自信 | `crv_confidence` | 選択肢 | | 高 / 低。低は校正会の議題 |
| 作成者(判定者) | `crv_createdby_annotator` | ユーザー参照 | | 誰の判定から生まれたか |
| 確認者 | `crv_confirmedby` | ユーザー参照 | | 2人目の同意者。候補 → 承認済の条件 |
| 生成元 Bot 版 | `crv_sourcepromptversion` | テキスト(50) | | どのプロンプト版の出力から作られたか |
| 生成元 評価指摘 | `crv_sourceevalfindingid` | 参照(EvalFinding) | | 追跡用 |

「未確認」の行は再現率の計算から除外し、TestCase の状態を「雛形要再確認」にします。
「候補」の行も分母に入れず、「候補込み再現率」として参考表示します。

## 4. EvalRun(評価実行) `crv_evalrun`

F1 の1回の実行に対応します。

| 表示名 | スキーマ名 | 型 | 必須 | 説明 |
|---|---|---|---|---|
| 実行ID | `crv_name`(主列) | テキスト(100) | ○ | 例 `RUN-20260906-1432` |
| 開始日時 | `crv_startedon` | 日時 | ○ | |
| 終了日時 | `crv_finishedon` | 日時 | | |
| モード | `crv_mode` | 選択肢 | ○ | 直接 / E2E |
| プロンプト版ラベル | `crv_promptversion` | テキスト(50) | ○ | 例 `v1.4.0`。docs/06 の命名規則 |
| モデル | `crv_model` | テキスト(100) | | 子フローの出力を記録 |
| タグ絞り込み | `crv_tagfilter` | テキスト(200) | | |
| 状態 | `crv_status` | 選択肢 | ○ | 実行中 / 完了 / 失敗 |
| ケース数 | `crv_casecount` | 整数 | | |
| 段階1 必須総数 | `crv_s1musttotal` | 整数 | | |
| 段階1 必須一致数 | `crv_s1musthit` | 整数 | | |
| 段階1 必須再現率 | `crv_s1mustrecall` | 小数 | | 診断用 |
| 段階1 推奨再現率 | `crv_s1shouldrecall` | 小数 | | |
| 余分指摘 総数 | `crv_extratotal` | 整数 | | 期待に無い段階1指摘の合計 |
| 段階2 評価対象数 | `crv_s2total` | 整数 | | 段階1で一致し、期待の雛形該当が「未確認」でない指摘数 |
| 段階2 一致数 | `crv_s2agree` | 整数 | | |
| 段階2 一致率 | `crv_s2agreement` | 小数 | | s2agree / s2total |
| 誤抑制数 | `crv_falsesuppress` | 整数 | | Bot = 該当、期待 = 非該当(利用者から見えなくなった指摘) |
| 誤抑制率 | `crv_falsesuppressrate` | 小数 | | falsesuppress / 期待が非該当の一致指摘数 |
| 誤通過数 | `crv_falsepass` | 整数 | | Bot = 非該当、期待 = 該当 |
| 誤通過率 | `crv_falsepassrate` | 小数 | | falsepass / 期待が該当の一致指摘数 |
| 最終出力 必須総数 | `crv_finalmusttotal` | 整数 | | 期待が「非該当 かつ 必須」の指摘数 |
| 最終出力 必須一致数 | `crv_finalmusthit` | 整数 | | そのうち Bot が一致 かつ 非該当 と判定した数 |
| 最終出力 必須再現率 | `crv_finalmustrecall` | 小数 | ○(主指標) | finalmusthit / finalmusttotal |
| 最終出力 必須再現率(fewshot 除外) | `crv_finalmustrecall_eval` | 小数 | ○(ゲート用) | `fewshot` タグの無いケースだけで計算。ゲートと前回比はこちらを使う |
| 候補込み 最終出力再現率 | `crv_finalmustrecall_withcand` | 小数 | | 状態=候補の期待指摘も分母に含めた参考値(docs/07) |
| 重要度一致率 | `crv_severityagreement` | 小数 | | |
| JSON 整形失敗数 | `crv_parsefailures` | 整数 | | |
| 平均応答時間(秒) | `crv_avglatency` | 小数 | | |
| 前回比 最終出力再現率 | `crv_finalmustrecalldelta` | 小数 | | 同モードの直前の完了実行との差 |
| ゲート判定 | `crv_gate` | 選択肢 | | 合格 / 不合格 / 未判定 |
| 実行者 | `crv_runby` | ユーザー参照 | | |
| メモ | `crv_notes` | 複数行テキスト | | 変更内容の要約、Q1=不当 の余分指摘の傾向など |

## 5. EvalResult(評価結果) `crv_evalresult`

EvalRun × TestCase の1件。人判定はここではなく EvalFinding(指摘単位)で持ちます。

| 表示名 | スキーマ名 | 型 | 必須 | 説明 |
|---|---|---|---|---|
| 結果ID | `crv_name`(主列) | テキスト(100) | ○ | `RUN-…_TC-0001` |
| 評価実行 | `crv_evalrunid` | 参照(EvalRun) | ○ | |
| テストケース | `crv_testcaseid` | 参照(TestCase) | ○ | |
| 雛形 | `crv_templateid` | 参照(Template) | ○ | 実行時に使われた雛形(版ずれの検出用) |
| 生応答 | `crv_rawresponse` | 複数行テキスト(memo) | | 子フロー or Bot の返答そのまま |
| 指摘 JSON | `crv_findingsjson` | 複数行テキスト(memo) | | 段階1全件 + 段階2判定 |
| JSON 整形失敗 | `crv_parsefailed` | はい/いいえ | ○ | |
| 審査結果 JSON | `crv_judgejson` | 複数行テキスト(memo) | | templates/judge_output_schema.json 準拠 |
| 段階1 必須一致数 / 総数 | `crv_s1musthit` / `crv_s1musttotal` | 整数 | | |
| 段階1 推奨一致数 / 総数 | `crv_s1shouldhit` / `crv_s1shouldtotal` | 整数 | | |
| 余分指摘数 | `crv_extracount` | 整数 | | |
| 段階2 一致数 / 評価対象数 | `crv_s2agree` / `crv_s2total` | 整数 | | |
| 誤抑制数 | `crv_falsesuppress` | 整数 | | |
| 誤通過数 | `crv_falsepass` | 整数 | | |
| 最終出力 必須一致数 / 総数 | `crv_finalmusthit` / `crv_finalmusttotal` | 整数 | | |
| 重要度不一致数 | `crv_severitymismatch` | 整数 | | |
| 審査理由 | `crv_judgereasoning` | 複数行テキスト | | 審査 LLM の短い説明 |
| 応答時間(秒) | `crv_latency` | 小数 | | |
| 人判定依頼済 | `crv_reviewrequested` | はい/いいえ | ○ | F3 の二重送信防止 |

## 6. EvalFinding(評価指摘) `crv_evalfinding`

EvalResult 1件に対し、**Bot の段階1指摘1件につき1行**。さらに欠落した期待指摘も1行(種別 = 欠落)持ちます。
業務判定者の Q1/Q2 はこの行に記録します。

| 表示名 | スキーマ名 | 型 | 必須 | 説明 |
|---|---|---|---|---|
| 評価指摘ID | `crv_name`(主列) | テキスト(100) | ○ | `<結果ID>_A1`、欠落は `<結果ID>_M_<期待指摘ID>` |
| 評価結果 | `crv_evalresultid` | 参照(EvalResult) | ○ | |
| 種別 | `crv_kind` | 選択肢 | ○ | 一致 / 余分 / 欠落 / LLM 下書き / 過去資産 |
| 優先度 | `crv_priority` | 整数 | | F1 が付与。F3 は高い順に予算まで送る(docs/07 4節) |
| 不当の理由 | `crv_q1reason` | 選択肢 | | 読み違え / 当社の立場では不要 / 重複 / その他(Q1 = 不当のとき) |
| 修正提案 JSON | `crv_proposedfix` | 複数行テキスト | | 判定者が編集した title / description / suggestion / 必須推奨 / 重要度 / 雛形該当。校正会承認後に期待指摘へ反映 |
| 自信 | `crv_confidence` | 選択肢 | | 高 / 低 |
| Bot 指摘ID | `crv_actualid` | テキスト(20) | | `A1`, `A2`… 欠落では空 |
| Bot 指摘 JSON | `crv_findingjson` | 複数行テキスト | | 指摘1件分(条項、カテゴリ、重要度、内容、提案、段階2項目) |
| 対応する期待指摘 | `crv_expectedfindingid` | 参照(ExpectedFinding) | | 一致・欠落で設定 |
| 段階2 Bot 判定 | `crv_s2bot` | 選択肢 | | 該当 / 非該当 |
| 段階2 期待 | `crv_s2expected` | 選択肢 | | 該当 / 非該当 / 未確認(一致・欠落のみ) |
| 段階2 一致 | `crv_s2agree` | 選択肢 | | 一致 / 誤抑制 / 誤通過 / 評価不能(余分・未確認) |
| 重要度一致 | `crv_severitymatch` | はい/いいえ | | 一致のみ |
| 審査理由 | `crv_judgereason` | テキスト(500) | | 審査 LLM の1文 |
| **人判定 Q1(指摘は妥当か)** | `crv_humanq1` | 選択肢 | ○ | 未 / 妥当 / 不当 / 判定不能 |
| **人判定 Q2(雛形にも該当するか)** | `crv_humanq2` | 選択肢 | ○ | 未 / 該当 / 非該当 / 判定不能 |
| 人コメント | `crv_humancomment` | 複数行テキスト | | 不当・不一致の理由。校正会の議題 |
| 判定者 | `crv_reviewer` | ユーザー参照 | | |
| 判定日時 | `crv_reviewedon` | 日時 | | |
| 人判定依頼済 | `crv_reviewrequested` | はい/いいえ | ○ | |
| 期待指摘へ反映済 | `crv_appliedtoexpected` | はい/いいえ | ○ | F3 後処理で ExpectedFinding を作成・更新したら はい |

## 7. Feedback(本番フィードバック) `crv_feedback`

Bot 利用者の評価を **指摘単位** で記録します(会話単位の行も同じテーブルに持ち、指摘 JSON が空の行が会話単位)。

| 表示名 | スキーマ名 | 型 | 必須 | 説明 |
|---|---|---|---|---|
| フィードバックID | `crv_name`(主列) | テキスト(100) | ○ | 会話ID + 連番 |
| 会話ID | `crv_conversationid` | テキスト(100) | ○ | Copilot Studio の `System.Conversation.Id` |
| 日時 | `crv_receivedon` | 日時 | ○ | |
| 利用者 | `crv_user` | テキスト(200) | | 匿名運用ならメール以外の識別子 |
| 評価 | `crv_rating` | 選択肢 | ○ | 役に立った / 違う / 表示すべきだった / 非表示で正しい |
| 理由分類 | `crv_reasoncode` | 選択肢 | | 読み違え / 当社には不要 / 雛形と同じ内容 / 重要度が違う / その他 |
| 理由 | `crv_reason` | 複数行テキスト | | 任意の自由記述 |
| 指摘 JSON | `crv_findingjson` | 複数行テキスト | | 評価対象の指摘1件。会話単位の行では空 |
| 非表示指摘か | `crv_washidden` | はい/いいえ | | 段階2で抑制された指摘への回答なら はい(誤抑制の検出) |
| プロンプト版 | `crv_promptversion` | テキスト(50) | | 回答時の Bot 版 |
| Bot 応答 | `crv_botresponse` | 複数行テキスト(memo) | | 最終出力(表示テキスト) |
| 入力参照 | `crv_inputref` | テキスト(500) | | 契約書ファイル名など。本文は保存しない(機密) |
| トリアージ状態 | `crv_triage` | 選択肢 | ○ | 未処理 / テストケース化 / 見送り / 重複 |
| 紐付けテストケース | `crv_testcaseid` | 参照(TestCase) | | テストケース化した場合 |
| トリアージ担当 | `crv_triagedby` | ユーザー参照 | | |

## 8. ビュー(最低限)

| テーブル | ビュー名 | 条件 |
|---|---|---|
| Template | 有効な雛形 | 状態 = 有効 |
| TestCase | 承認済み | 状態 = 承認済 |
| TestCase | 仕上げ待ち | 状態 = 草案 または 雛形要再確認、かつ 担当(業務) = 自分 |
| ExpectedFinding | 雛形該当 未確認 | 雛形該当(期待) = 未確認 |
| EvalRun | 最近の実行 | 状態 = 完了、開始日時 降順 |
| EvalFinding | 人判定待ち | 人判定 Q1 = 未 かつ 人判定依頼済 = はい |
| EvalFinding | 誤抑制一覧 | 段階2 一致 = 誤抑制 |
| EvalFinding | 校正会議題 | Q1 = 不当 または Q1 = 判定不能 または Q2 = 判定不能 または (種別 = 一致 かつ Q2 ≠ 段階2 期待) |
| Feedback | 未処理 | トリアージ状態 = 未処理 |
| ExpectedFinding | 受け入れ済み条件一覧(雛形別) | 状態 = 承認済 かつ 雛形該当 = 該当。列: 雛形(TestCase 経由)、カテゴリ、雛形条項参照、期待する指摘内容、期待指摘ID。子フローが段階2の前に取得(docs/08 3節) |
| ExpectedFinding | 指摘すべき差分一覧(雛形別) | 状態 = 承認済 かつ 雛形該当 = 非該当。列は同上 |
| ExpectedFinding | 雛形弱点(high × 該当) | 状態 = 承認済 かつ 雛形該当 = 該当 かつ 重要度 = high。四半期の法務向けレポート(docs/08 6節) |

## 9. アプリ

上記テーブルからモデル駆動アプリを自動生成すると、業務判定者向けの「仕上げ待ち」「人判定待ち」「誤抑制一覧」画面がほぼ無償で用意できます。
最初は Teams のアダプティブカード(docs/03 の F3・F4)だけで運用し、件数が増えたらアプリを追加してください。
