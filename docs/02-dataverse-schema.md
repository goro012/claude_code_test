# 02. Dataverse テーブル定義

スキーマ名の接頭辞は `crv_`(Contract Review)としています。環境の発行者接頭辞に合わせて読み替えてください。
すべてソリューションに含め、環境間で移送できるようにします。

## テーブル一覧と関係

```
TestCase 1 ─── * ExpectedFinding
TestCase 1 ─── * EvalResult * ─── 1 EvalRun
TestCase 1 ─── * Feedback(採用時のみ紐付け)
```

## 1. TestCase(テストケース) `crv_testcase`

ゴールデンセットの入力1件。契約書1本に対応します。

| 表示名 | スキーマ名 | 型 | 必須 | 説明 |
|---|---|---|---|---|
| ケースID | `crv_name`(主列) | テキスト(100) | ○ | 例 `TC-0001`。人が読める一意キー |
| タイトル | `crv_title` | テキスト(200) | ○ | 例「業務委託契約(受託側)・損害賠償上限なし」 |
| 契約種別 | `crv_contracttype` | 選択肢 | ○ | 業務委託 / 秘密保持 / 売買 / 賃貸借 / ライセンス / その他 |
| 当社の立場 | `crv_position` | 選択肢 | ○ | 発注側 / 受注側 / 対等 |
| 契約書テキスト | `crv_contracttext` | 複数行テキスト(memo, 最大長) | ○ | 直接モードの入力。Word から抽出済みの本文 |
| Wordファイル URL | `crv_fileurl` | URL | | E2E モードの入力。SharePoint ドキュメントライブラリ上のパス |
| 出所 | `crv_source` | 選択肢 | ○ | 実契約書(匿名化済) / 合成 / 本番フィードバック由来 |
| 状態 | `crv_status` | 選択肢 | ○ | 草案 / 承認済 / 退役 |
| タグ | `crv_tags` | テキスト(200) | | `smoke`, `critical`, `regression-only` などカンマ区切り。F1 の絞り込みに使う |
| 重要ケース | `crv_iscritical` | はい/いいえ | ○ | はい なら必須指摘の欠落1件でリリースゲート不合格 |
| 担当(業務) | `crv_owner_business` | ユーザー参照 | | ExpectedFinding を仕上げる業務判定者 |
| 備考 | `crv_notes` | 複数行テキスト | | 匿名化の方法、意図している論点など |

## 2. ExpectedFinding(期待指摘) `crv_expectedfinding`

TestCase 1件に対して「Bot が指摘すべき事項」を1行ずつ持ちます。

| 表示名 | スキーマ名 | 型 | 必須 | 説明 |
|---|---|---|---|---|
| 期待指摘ID | `crv_name`(主列) | テキスト(100) | ○ | 例 `TC-0001-01` |
| テストケース | `crv_testcaseid` | 参照(TestCase) | ○ | |
| 条項参照 | `crv_clauseref` | テキスト(50) | ○ | 例「第12条」「第5条2項」「該当条項なし」 |
| カテゴリ | `crv_category` | 選択肢 | ○ | docs/05 のカテゴリ語彙(損害賠償 / 契約解除 / 知的財産 / 秘密保持 / 準拠法・裁判管轄 / 支払条件 / 契約期間・更新 / 反社条項 / 再委託 / その他) |
| 重要度 | `crv_severity` | 選択肢 | ○ | high / medium / low |
| 必須/推奨 | `crv_required` | 選択肢 | ○ | 必須(must) / 推奨(should)。再現率は必須のみで計算 |
| 期待する指摘内容 | `crv_description` | 複数行テキスト | ○ | Bot が伝えるべき内容。文言一致は求めない |
| 根拠 | `crv_rationale` | 複数行テキスト | | なぜ問題か。審査 LLM への補助情報にもなる |
| 由来 | `crv_origin` | 選択肢 | | 業務部門作成 / Bot 出力を修正 / 本番フィードバック |

## 3. EvalRun(評価実行) `crv_evalrun`

F1 の1回の実行に対応します。

| 表示名 | スキーマ名 | 型 | 必須 | 説明 |
|---|---|---|---|---|
| 実行ID | `crv_name`(主列) | テキスト(100) | ○ | 例 `RUN-20260906-1432` |
| 開始日時 | `crv_startedon` | 日時 | ○ | |
| 終了日時 | `crv_finishedon` | 日時 | | |
| モード | `crv_mode` | 選択肢 | ○ | 直接 / E2E |
| プロンプト版ラベル | `crv_promptversion` | テキスト(50) | ○ | 例 `v1.4.0`。docs/06 の命名規則 |
| モデル | `crv_model` | テキスト(100) | | 子フローの出力を記録 |
| タグ絞り込み | `crv_tagfilter` | テキスト(200) | | 実行時に指定した絞り込み |
| 状態 | `crv_status` | 選択肢 | ○ | 実行中 / 完了 / 失敗 |
| ケース数 | `crv_casecount` | 整数 | | |
| 必須指摘 総数 | `crv_musttotal` | 整数 | | |
| 必須指摘 一致数 | `crv_musthit` | 整数 | | |
| 必須指摘 再現率 | `crv_mustrecall` | 小数 | | musthit / musttotal |
| 推奨指摘 再現率 | `crv_shouldrecall` | 小数 | | |
| 余分指摘 総数 | `crv_extratotal` | 整数 | | 期待に無い指摘の合計 |
| 重要度一致率 | `crv_severityagreement` | 小数 | | 一致した指摘のうち重要度も一致した割合 |
| JSON 整形失敗数 | `crv_parsefailures` | 整数 | | |
| 平均応答時間(秒) | `crv_avglatency` | 小数 | | |
| 前回比 必須再現率 | `crv_mustrecalldelta` | 小数 | | 同モードの直前の完了実行との差 |
| ゲート判定 | `crv_gate` | 選択肢 | | 合格 / 不合格 / 未判定 |
| 実行者 | `crv_runby` | ユーザー参照 | | |
| メモ | `crv_notes` | 複数行テキスト | | 変更内容の要約 |

## 4. EvalResult(評価結果) `crv_evalresult`

EvalRun × TestCase の1件。

| 表示名 | スキーマ名 | 型 | 必須 | 説明 |
|---|---|---|---|---|
| 結果ID | `crv_name`(主列) | テキスト(100) | ○ | `RUN-…_TC-0001` |
| 評価実行 | `crv_evalrunid` | 参照(EvalRun) | ○ | |
| テストケース | `crv_testcaseid` | 参照(TestCase) | ○ | |
| 生応答 | `crv_rawresponse` | 複数行テキスト(memo) | | 子フロー or Bot の返答そのまま |
| 指摘 JSON | `crv_findingsjson` | 複数行テキスト(memo) | | パース後の指摘一覧 |
| JSON 整形失敗 | `crv_parsefailed` | はい/いいえ | ○ | |
| 審査結果 JSON | `crv_judgejson` | 複数行テキスト(memo) | | templates/judge_output_schema.json 準拠 |
| 必須一致数 | `crv_musthit` | 整数 | | |
| 必須総数 | `crv_musttotal` | 整数 | | |
| 推奨一致数 | `crv_shouldhit` | 整数 | | |
| 推奨総数 | `crv_shouldtotal` | 整数 | | |
| 余分指摘数 | `crv_extracount` | 整数 | | |
| 重要度不一致数 | `crv_severitymismatch` | 整数 | | |
| 審査理由 | `crv_judgereasoning` | 複数行テキスト | | 審査 LLM の短い説明 |
| 応答時間(秒) | `crv_latency` | 小数 | | |
| 人判定 | `crv_humanverdict` | 選択肢 | ○ | 未 / 審査に同意 / 審査に不同意 / 判定不能 |
| 人コメント | `crv_humancomment` | 複数行テキスト | | 不同意の理由。校正会の議題になる |
| 判定者 | `crv_reviewer` | ユーザー参照 | | |
| 判定日時 | `crv_reviewedon` | 日時 | | |
| 人判定依頼済 | `crv_reviewrequested` | はい/いいえ | ○ | F3 の二重送信防止 |

## 5. Feedback(本番フィードバック) `crv_feedback`

Bot 利用者が押した「役に立った / 違う」を記録します。

| 表示名 | スキーマ名 | 型 | 必須 | 説明 |
|---|---|---|---|---|
| フィードバックID | `crv_name`(主列) | テキスト(100) | ○ | 会話ID + 連番 |
| 会話ID | `crv_conversationid` | テキスト(100) | ○ | Copilot Studio の `System.Conversation.Id` |
| 日時 | `crv_receivedon` | 日時 | ○ | |
| 利用者 | `crv_user` | テキスト(200) | | 匿名運用ならメール以外の識別子 |
| 評価 | `crv_rating` | 選択肢 | ○ | 役に立った / 違う |
| 理由 | `crv_reason` | 複数行テキスト | | 「違う」のとき利用者に入力を促す |
| Bot 応答 | `crv_botresponse` | 複数行テキスト(memo) | | |
| 入力参照 | `crv_inputref` | テキスト(500) | | 契約書ファイル名など。本文は保存しない(機密) |
| トリアージ状態 | `crv_triage` | 選択肢 | ○ | 未処理 / テストケース化 / 見送り / 重複 |
| 紐付けテストケース | `crv_testcaseid` | 参照(TestCase) | | テストケース化した場合 |
| トリアージ担当 | `crv_triagedby` | ユーザー参照 | | |

## 6. ビュー(最低限)

| テーブル | ビュー名 | 条件 |
|---|---|---|
| TestCase | 承認済み | 状態 = 承認済 |
| TestCase | 仕上げ待ち | 状態 = 草案 かつ 担当(業務) = 自分 |
| EvalRun | 最近の実行 | 状態 = 完了、開始日時 降順 |
| EvalResult | 人判定待ち | 人判定 = 未 かつ 人判定依頼済 = はい |
| EvalResult | 不同意一覧(校正会用) | 人判定 = 審査に不同意 |
| Feedback | 未処理 | トリアージ状態 = 未処理 |

## 7. アプリ

上記テーブルからモデル駆動アプリを自動生成すると、業務判定者向けの「仕上げ待ち」「人判定待ち」画面がほぼ無償で用意できます。
最初は Teams のアダプティブカード(docs/03 の F3・F4)だけで運用し、件数が増えたらアプリを追加してください。
