# 09. フロー図集

設計書 docs/01〜08 のフローとデータ構造を Mermaid で図にしたものです。GitHub 上でそのまま描画されます。
**設計を変更したときは、対応する図をこのファイルで必ず更新します**(ルールは CLAUDE.md)。

| # | 図 | 対応する設計書 |
|---|---|---|
| 1a/1b | 全体構成(本番経路 / 評価・アノテーション経路) | docs/01 |
| 2 | 内側ループと外側ループ | docs/01, 07 |
| 3 | データモデル | docs/02 |
| 4 | 子フロー: レビュー実行 | docs/03 1節, 08 3節 |
| 5 | F1 評価実行 | docs/03 2節 |
| 6 | F2 フィードバック記録(本番: F2b Teams 投稿が既定) | docs/03 3節, 07 3.2節, 10 |
| 7 | F3 人判定依頼 | docs/03 4節, 07 3.1節・4節 |
| 8 | F4 フィードバック・トリアージ | docs/03 5節 |
| 9 | F6 週次集約 | docs/03 7節, 08 8節 |
| 10 | F0 テストケース登録(任意) | docs/03 6節 |
| 11 | 期待指摘のライフサイクル | docs/07 2節 |
| 12 | アノテーション結果の活用 | docs/08 9節 |
| 13 | リリース手順 | docs/06 6節 |

## 1. 全体構成

全体を「本番経路」と「評価・アノテーション経路」の2図に分けています。両者は Dataverse のゴールデンセットで繋がります。

### 1a. 本番経路(利用者がレビューを受ける)

```mermaid
flowchart LR
    U["Bot 利用者"] -->|"Word 添付"| T1["Copilot Studio\nレビュー トピック"]
    T1 --> CF["子フロー\nレビュー実行"]
    CF --> L1["Foundry\n段階1 レビュー"]
    CF --> L2["Foundry\n段階2 雛形照合"]
    GOLD[("Dataverse\nTemplate / ExpectedFinding\n受け入れ済み条件一覧")] -.-> CF
    CF -->|"card_json / display_text(非該当のみ)\nfindings_json(全件)"| T1
    T1 -->|"レビュー結果カード\n(非対応チャネルは Markdown 表)"| U
    T1 -->|"完了後(非同期)"| F2B["F2b フィードバック カード送信\n(10分遅延・回答任意・スキップ可)"]
    F2B -->|"指摘単位ボタン + 非表示指摘の確認"| TEAMSU["利用者の Teams チャット"]
    TEAMSU --> U
    U -->|"役に立った / 違う(理由)\n表示すべきだった / 非表示で正しい"| TEAMSU
    TEAMSU --> F2B --> FB[("Dataverse\nFeedback")]
    T1 -.->|"Teams チャネルのみ(補助)"| T3["F2a 会話内カード"] -.-> FB
```

### 1b. 評価・アノテーション経路(開発者と業務部門)

```mermaid
flowchart TB
    DEV["開発者"] -->|"プロンプト修正後に実行"| F1["F1 評価実行"]
    F1 -->|"直接モード"| CF["子フロー レビュー実行\n(E2E モードは EVAL トピック経由)"]
    F1 --> LJ["Foundry 審査(段階1の照合)"]
    GOLD[("ゴールデンセット\nTemplate / TestCase / ExpectedFinding")] -->|"期待指摘"| F1
    F1 -->|"EvalRun / EvalResult / EvalFinding"| EVAL[("評価結果")]
    F1 -->|"主指標・ゲート"| TEAMS["Teams"]
    F1 --> F3["F3 人判定依頼"]
    EVAL --> F3
    F3 -->|"Q1/Q2 添削カード"| TEAMS
    TEAMS <-->|"回答"| R["業務判定者(法務)"]
    F3 -->|"候補作成・確認者記録"| GOLD
    FB[("Feedback")] --> F4["F4 FBトリアージ(週次)"]
    F4 -->|"カード"| TEAMS
    F4 -->|"TestCase 草案"| GOLD
    EVAL --> F6["F6 週次集約"]
    GOLD --> F6
    F6 -->|"直す場所 / 一覧差分 / 例示候補 / 審査不一致"| TEAMS
    TEAMS --> DEV
    TEAMS -->|"法務向けレポート(四半期)"| O["業務オーナー"]
    SP["SharePoint(Word 原本)"] --> F0["F0 登録"] --> GOLD
```

## 2. 内側ループと外側ループ

```mermaid
flowchart TB
    subgraph INNER["内側ループ(開発者のみ・毎変更・数分)"]
        P["プロンプト修正"] --> RUN["F1 評価実行(直接モード)"]
        RUN --> RES["回帰結果: 最終出力再現率 / 誤抑制 / 余分"]
        RES -->|"ゲート不合格"| P
        RES -->|"合格"| REL["リリース手順へ"]
    end

    subgraph OUTER["外側ループ(業務部門・週次・非同期)"]
        SEL["優先度順に予算まで抽出\n(誤抑制 > 境界事例 > カバレッジ空白 > 未確認 > 審査疑い > 2人目同意 > 無作為)"]
        CARD["F3 Q1/Q2 カード(添削型)"]
        PROD["本番の指摘単位フィードバック\n(非表示指摘の確認を含む)"]
        CAL["校正会(週30分・割れた事例のみ)"]
        GOLD["ExpectedFinding 候補 → 承認済"]
        SEL --> CARD --> GOLD
        PROD --> F4T["F4 トリアージ"] --> GOLD
        CARD --> CAL --> GOLD
    end

    RES -->|"余分・欠落・誤抑制・誤通過"| SEL
    GOLD -->|"分母が育つ / 受け入れ済み条件一覧 / 例示"| RUN
    F6W["F6 週次集約: 直す場所の特定"] --> P
    GOLD --> F6W
```

## 3. データモデル

```mermaid
erDiagram
    Template ||--o{ TestCase : "雛形"
    Template ||--o{ EvalResult : "実行時の雛形"
    TestCase ||--o{ ExpectedFinding : "期待指摘"
    TestCase ||--o{ EvalResult : "評価対象"
    TestCase |o--o{ Feedback : "採用時に紐付け"
    EvalRun ||--o{ EvalResult : "実行"
    EvalResult ||--o{ EvalFinding : "指摘単位"
    ExpectedFinding |o--o{ EvalFinding : "対応する期待指摘"
    EvalFinding |o--o{ ExpectedFinding : "生成元(候補作成)"

    Template {
        string crv_name "雛形ID"
        choice contracttype "契約種別"
        choice position "当社の立場"
        string version "版"
        date approvedon "法務承認日"
        memo templatetext "雛形テキスト"
        choice status "有効/退役"
    }
    TestCase {
        string crv_name "ケースID"
        choice contracttype "契約種別"
        choice position "当社の立場"
        memo contracttext "契約書テキスト"
        url fileurl "Word URL"
        choice status "草案/承認済/退役/雛形要再確認"
        string tags "smoke,critical,fewshot..."
        bool iscritical "重要ケース"
    }
    ExpectedFinding {
        string crv_name "期待指摘ID"
        string clauseref "条項参照"
        choice category "カテゴリ"
        choice severity "high/medium/low"
        choice required "必須/推奨"
        memo description "期待する指摘内容"
        choice templateapplicable "該当/非該当/未確認"
        string templateclauseref "雛形条項参照"
        choice state "候補/承認済/退役"
        choice confidence "高/低"
        string sourcepromptversion "生成元 Bot 版"
    }
    EvalRun {
        string crv_name "実行ID"
        choice mode "直接/E2E"
        string promptversion "プロンプト版"
        decimal finalmustrecall_eval "最終出力再現率(fewshot除外)"
        decimal s1mustrecall "段階1再現率"
        decimal s2agreement "段階2一致率"
        int falsesuppress "誤抑制数"
        int falsepass "誤通過数"
        choice gate "合格/不合格"
    }
    EvalResult {
        string crv_name "結果ID"
        memo findingsjson "段階1全件+段階2判定"
        memo judgejson "審査結果"
        bool parsefailed "JSON失敗"
        decimal latency "応答時間"
    }
    EvalFinding {
        string crv_name "評価指摘ID"
        choice kind "一致/余分/欠落/LLM下書き/過去資産"
        memo findingjson "Bot指摘"
        choice s2bot "段階2 Bot判定"
        choice s2expected "段階2 期待"
        choice s2agree "一致/誤抑制/誤通過/評価不能"
        int priority "優先度"
        choice humanq1 "未/妥当/不当/判定不能"
        choice humanq2 "未/該当/非該当/判定不能"
        choice q1reason "不当の理由"
        memo proposedfix "修正提案"
        choice confidence "高/低"
    }
    Feedback {
        string crv_name "フィードバックID"
        string conversationid "会話ID"
        choice rating "役に立った/違う/表示すべきだった/非表示で正しい"
        choice reasoncode "理由分類"
        memo findingjson "指摘(指摘単位のとき)"
        bool washidden "非表示指摘か"
        choice triage "未処理/テストケース化/見送り/重複"
    }
```

## 4. 子フロー: レビュー実行

```mermaid
flowchart TD
    IN["入力: contract_text, prompt_version,\ncontract_type, position, template_id(任意)"] --> T{"template_id あり?"}
    T -->|"はい"| TG["Template を ID で取得"]
    T -->|"いいえ"| TS["契約種別 × 立場 × 有効 で Template 検索(上位1)"]
    TS --> TE{"0件?"}
    TE -->|"はい"| ERR["エラー終了(段階2不可)"]
    TE -->|"いいえ"| S1
    TG --> S1

    subgraph S1["段階1: レビュー"]
        P1["段階1 システムプロンプト\n(観点・語彙・JSON 指示・例示)"] --> C1["Foundry 呼び出し(temp 0〜0.2)"]
        C1 --> J1["{...} 切り出し → JSON 解析"]
        J1 -->|"失敗"| PF1["parse_failed=true, findings=[]"]
        J1 -->|"成功"| ID1["指摘に A1, A2... の ID を付与"]
    end

    ID1 --> ACC["ExpectedFinding ビュー\n受け入れ済み条件一覧 / 指摘すべき差分一覧\n(雛形ID・段階1のカテゴリで絞る)"]
    ACC --> S2

    subgraph S2["段階2: 雛形照合"]
        P2["段階2 システムプロンプト\n(雛形本文 + 一覧 + 段階1指摘)\n削除せず全件にフラグ・雛形条項・引用・理由"] --> C2["Foundry 呼び出し(temp 0)"]
        C2 --> J2["{...} 切り出し → JSON 解析"]
        J2 -->|"失敗"| PF2["全件 applies_to_template=false\n(黙って消さない), parse_failed_s2=true"]
        J2 -->|"成功"| MG["段階1 と 段階2 を id で結合"]
    end

    PF1 --> OUT
    PF2 --> OUT
    MG --> DISP["Filter: applies_to_template=false\n→ display_text(Markdown 表) と\ncard_json(レビュー結果カード) を同じ配列から生成"]
    DISP --> OUT["出力: findings_json(全件+判定), display_text, card_json,\nparse_failed, parse_failed_s2, model,\ntemplate_id, template_version"]
```

## 5. F1 評価実行

```mermaid
flowchart TD
    IN["入力: mode(direct/e2e), prompt_version,\ntag_filter, notes, review_scope"] --> RUN["EvalRun 作成(実行中)"]
    RUN --> LIST["TestCase 一覧(承認済 + タグ絞り込み)"]
    LIST --> LOOP

    subgraph LOOP["Apply to each: TestCase(direct 並列5 / e2e 並列1〜2)"]
        M{"mode?"}
        M -->|"direct"| CF["子フロー: レビュー実行\n(template_id = TestCase の雛形)"]
        M -->|"e2e"| EA["Copilot Studio: Execute Agent and wait\n(EVAL トピック, Word 添付)"]
        CF --> LAT["応答時間を計測"]
        EA --> LAT
        LAT --> EXP["ExpectedFinding 一覧(承認済 + 候補)\n→ 期待指摘 JSON"]
        EXP --> JUDGE["審査 LLM(段階1の意味照合のみ)\n→ matches / extras"]
        JUDGE --> RES["EvalResult 作成"]
        RES --> MT

        subgraph MT["Apply to each: matches"]
            MQ{"matched?"}
            MQ -->|"true"| CMP["段階2 比較:\nBot applies_to_template vs 期待 雛形該当\n→ 一致 / 誤抑制 / 誤通過 / 評価不能"]
            CMP --> EF1["EvalFinding(種別=一致)"]
            MQ -->|"false"| EF2["EvalFinding(種別=欠落)"]
        end

        RES --> XT
        subgraph XT["Apply to each: extras"]
            EF3["EvalFinding(種別=余分, 段階2=評価不能)"]
        end

        EF1 --> PRI["優先度を付与\n(誤抑制=1, 版跨ぎ境界=2, カバレッジ空白=3,\n未確認=4, 審査疑い=5, 候補同意=6, 無作為=7)"]
        EF2 --> PRI
        EF3 --> PRI
        PRI --> CNT["EvalFinding を取り直して\nケース単位の件数を計算 → EvalResult 更新"]
    end

    LOOP --> AGG["EvalResult を取り直して実行全体を集計\n(fewshot 除外 / 全件 / 候補込み)"]
    AGG --> PREV["同モードの前回 EvalRun を取得 → 前回比"]
    PREV --> GATE{"ゲート判定\n前回比 < -閾値?\n重要ケースで欠落 or 誤抑制?\nJSON 失敗?"}
    GATE -->|"いずれか該当"| NG["不合格"]
    GATE -->|"該当なし"| OK["合格"]
    NG --> UPD["EvalRun 更新(完了・指標・ゲート)"]
    OK --> UPD
    UPD --> TEAMS["Teams 投稿(主指標・前回比・誤抑制/誤通過・余分・ゲート理由)"]
    TEAMS --> SC{"review_scope ≠ none?"}
    SC -->|"はい"| F3["子フロー F3 人判定依頼(scope)"]
    SC -->|"いいえ"| END["終了"]
    F3 --> END
```

## 6. F2 フィードバック記録(本番)

```mermaid
sequenceDiagram
    actor U as Bot 利用者(Teams / M365 Copilot / Web)
    participant B as Copilot Studio Bot
    participant CF as 子フロー レビュー実行
    participant F2 as F2b カード送信
    participant T as 利用者の Teams チャット
    participant DV as Dataverse Feedback

    U->>B: Word 添付でレビュー依頼
    B->>CF: contract_text, position, ...
    CF-->>B: findings_json(全件+判定), display_text
    B-->>U: display_text(非該当の指摘のみ)
    B-)F2: 完了後に非同期で呼ぶ<br/>findings_json, UPN, ファイル名, 会話ID
    U->>B: レビュー結果について質問(会話は止まらない)
    B-->>U: フォローアップ用トピックが Global.findings_json を使って回答
    F2->>F2: Delay(crv_FeedbackDelayMinutes, 既定10分)
    F2->>F2: 表示指摘 最大2件 + 非表示指摘 最大2件を選ぶ
    F2->>T: アダプティブ カードを投稿して応答を待機(3日)
    T-->>U: 「先ほどのレビュー結果について(30秒)」回答は任意
    alt 送信
        U->>T: 表示指摘: 役に立った/違う+理由, 非表示指摘: 表示すべきだった/非表示で正しい, 全体
        T-->>F2: data(action=f2_submit)
        F2->>DV: 指摘ごとに Feedback 行を追加(契約書本文は保存しない)
    else スキップ
        U->>T: スキップ
        T-->>F2: data(action=f2_skip)
        F2->>DV: 評価=スキップ の会話単位行を1行(回答率の計測)
    else 3日で未回答
        F2->>F2: 何も記録せず終了
    end
    Note over B,DV: F2a(会話内カード)は Teams チャネルのみの補助。「質問」ノードで待たず、<br/>送信は System.Activity.Value を見る別トピックで受ける。回答があれば F2b は送らない
```

## 7. F3 人判定依頼(指摘単位の Q1 / Q2)

```mermaid
flowchart TD
    IN["入力: evalrun_id, scope(disagreements / all)"] --> SEL{"scope?"}
    SEL -->|"disagreements"| D["対象: 余分・欠落・誤抑制・誤通過・\n段階2期待=未確認・LLM下書き・過去資産"]
    SEL -->|"all"| A["対象: 全 EvalFinding"]
    D --> PRI["優先度順に並べ、週の予算まで抽出\n+ 10% は二重判定用に2人へ + 10% 無作為"]
    A --> PRI
    PRI --> CH["ケースごと・最大10指摘ずつに分割"]
    CH --> CARD["アダプティブカード生成(添削型)\n指摘ごとに: 種別 / Bot指摘 / 雛形引用 / Bot段階2判定\nQ1 妥当? (不当の理由) / Q2 雛形該当?\n必須推奨・重要度(初期値=Bot) / 自信\n▸ 内容を修正(折りたたみ, Bot文で埋める)\n末尾: 足りない指摘は?"]
    CARD --> FLAG["EvalFinding 人判定依頼済=はい"]
    FLAG --> POST["Teams: カードを投稿して応答を待機(5日)"]
    POST -->|"タイムアウト"| SKIP["更新せず次回再送"]
    POST -->|"回答"| UPD["EvalFinding 更新\n(Q1, Q2, 不当の理由, 修正提案, 自信, コメント, 判定者)"]
    UPD --> PP

    subgraph PP["後処理: 人判定 → 期待指摘"]
        K{"種別 × 回答"}
        K -->|"余分 & Q1=妥当"| NEW["ExpectedFinding 候補を作成\n(雛形該当=Q2, 必須推奨・重要度=回答値)"]
        K -->|"余分 & Q1=不当"| REASON["不当の理由を記録\n(段階1改善の分類)"]
        K -->|"一致 & 修正あり or Q2≠期待"| PROP["修正提案として保持\n→ 校正会で承認後に反映"]
        K -->|"一致 & 修正なし"| CONF["確認者を記録\n候補なら承認済に昇格"]
        K -->|"欠落 & 不要"| RET["期待指摘を退役候補(校正会)"]
        K -->|"足りない指摘あり"| ADD["ExpectedFinding 候補を作成(由来=判定者追加)"]
    end

    PP --> SUM["Teams 要約: 送信 n / 残 n / 候補作成 n / 校正会議題 n"]
```

## 8. F4 フィードバック・トリアージ

```mermaid
flowchart TD
    REC["Recurrence: 週1回(校正会の前日)"] --> LIST["Feedback 一覧(未処理, 上位20, 古い順)"]
    LIST --> LOOP
    subgraph LOOP["Apply to each(同時実行オフ)"]
        CARD["カード: 日時 / 評価 / 理由分類 / Bot応答 / 入力参照\n入力: テストケース化・見送り・重複 / 本来の指摘(1〜3行) / 雛形にも当てはまるか"]
        CARD --> D{"decision?"}
        D -->|"テストケース化"| TC["TestCase 作成(草案, 雛形を種別×立場で検索,\n契約書テキスト=空 → 匿名化テキストの再提供依頼)"]
        TC --> EF["ExpectedFinding 作成(必須, 雛形該当=回答, 由来=本番FB)"]
        EF --> UP1["Feedback 更新(状態, 紐付けTestCase)"]
        D -->|"見送り / 重複"| UP2["Feedback 更新(状態)"]
    end
    LOOP --> SUM["Teams: 採用 n / 見送り n / 重複 n, 仕上げ待ち TestCase 一覧"]
```

## 9. F6 週次集約(アノテーション活用)

```mermaid
flowchart TD
    REC["Recurrence: 週1回(校正会の前日, F4 の後)"] --> G1
    REC --> G2
    REC --> G3
    REC --> G4

    subgraph G1["1. 理由分類別の件数"]
        A1["EvalFinding(直近7日) を集計:\nQ1不当の理由 × カテゴリ /\nQ2不一致の方向 × カテゴリ /\n重要度修正 / 足りない指摘 /\nFeedback 表示すべきだった"] --> A2["先週との差分 → 最多分類を『今週直す候補』に"]
    end
    subgraph G2["2. 受け入れ済み条件一覧の差分"]
        B1["ビュー: 承認済 & 雛形該当=該当(雛形別)"] --> B2["前回スナップショットと比較 → 追加/削除行"] --> B3["スナップショット更新"]
    end
    subgraph G3["3. 例示候補"]
        C1["校正会で決着した自信=低の行 /\n二重判定で不一致→一致の行"] --> C2["期待指摘 + 対象条文抜粋を添えて列挙\n(採用したら TestCase に fewshot タグ)"]
    end
    subgraph G4["4. 審査不一致事例"]
        D1["欠落 & コメントに『該当』 / 一致 & Q1=不当"] --> D2["審査・人判定一致率"]
    end

    A2 --> POST["Teams 投稿: 4点 + 承認済の週次増分 /\n候補の滞留 / カバレッジ表(契約種別 × カテゴリ)"]
    B3 --> POST
    C2 --> POST
    D2 --> POST
    POST --> Q{"四半期の初回?"}
    Q -->|"はい"| LEGAL["法務向けレポート: 雛形弱点(high × 該当) / 足りない指摘の累計"]
    Q -->|"いいえ"| END["終了"]
```

## 10. F0 テストケース登録(任意)

```mermaid
flowchart LR
    SP["SharePoint ライブラリに Word を配置\n(フォルダで TestCase / Template を分ける)"] --> TR["ファイル作成トリガー"]
    TR --> CV["既存 Bot と同じ Word → テキスト変換"]
    CV --> CHK["『第n条』の出現数で抽出漏れを簡易チェック\n(不一致なら備考に警告)"]
    CHK --> K{"フォルダ?"}
    K -->|"TestCase"| TC["TestCase 作成(草案, 雛形をファイル名の種別×立場で検索)"]
    K -->|"Template"| TP["Template 作成(有効, 旧版を退役)"]
    TC --> N["担当に Teams 通知"]
    TP --> N
```

## 11. 期待指摘のライフサイクル

```mermaid
stateDiagram-v2
    [*] --> Candidate : Q1=妥当(余分) / 足りない指摘 /\n本番FB / LLM下書き / 過去資産
    Candidate --> Approved : 2人目の同意 or 校正会の決定
    Candidate --> Retired : 校正会で不採用 / 重複
    Approved --> Approved : 修正提案は校正会承認後に反映
    Approved --> Retired : 雛形改版で無効 / 欠落&不要 / 重複整理
    Approved --> Recheck : 雛形改版(雛形該当=未確認)
    Recheck --> Approved : 雛形該当を再確認
    Retired --> [*]

    state "候補(candidate)\n参考値のみ" as Candidate
    state "承認済(approved)\n再現率の分母" as Approved
    state "雛形要再確認\n集計から除外" as Recheck
    state "退役(retired)\n削除しない" as Retired
```

## 12. アノテーション結果の活用

```mermaid
flowchart LR
    AN["アノテーション\nQ1/Q2・修正・足りない指摘・本番FB"] --> APPR["承認済 ExpectedFinding"]
    AN --> AGGR["理由分類・修正の集計(F6)"]
    AN --> JCMP["Q1 と審査の突き合わせ(F6)"]
    AN --> BIZ["high×該当 / 足りない指摘 / 本番FB"]

    APPR --> U1["1. 回帰の分母\n(fewshot 除外でゲート)"]
    APPR -->|"雛形該当=該当"| U3["3. 受け入れ済み条件一覧\n→ 段階2プロンプトへ自動供給"]
    APPR -->|"雛形該当=非該当"| U3b["3'. 指摘すべき差分一覧\n→ 候補生成・段階2へ"]
    APPR -->|"境界事例"| U4["4. 例示(few-shot)\n→ 段階1/段階2(マイナー版)\n採用ケースに fewshot タグ"]
    AGGR --> U2["2. 直す場所の特定\n最多分類から1つずつ"]
    JCMP --> U5["5. 審査プロンプトの校正\n(一致率 90% 未満のとき)"]
    BIZ --> U6["6. 雛形弱点レポート / チェックリスト /\n教材 / 利用者向け説明(四半期)"]

    U2 --> PRM["段階1・段階2 プロンプト / 子フロー後処理"]
    U3 --> PRM
    U4 --> PRM
    PRM --> F1["F1 回帰(fewshot 除外後の主指標で確認)"]
    U1 --> F1
```

## 13. リリース手順

```mermaid
flowchart TD
    S1["1. プロンプト修正(段階1 / 段階2, ラベル更新)\n直す分類は1つ"] --> S2["2. F1 direct, 全件, review_scope=disagreements"]
    S2 --> G{"ゲート合格?"}
    G -->|"不合格"| S1
    G -->|"合格"| S3["3. F1 e2e, tag=smoke, review_scope=none"]
    S3 --> G2{"Bot 経路で正常?"}
    G2 -->|"いいえ"| FIX["Bot / 子フロー呼び出しを修正"] --> S3
    G2 -->|"はい"| MJ{"メジャー版の変更?"}
    MJ -->|"はい"| OWN["業務オーナーの事前確認"] --> S4
    MJ -->|"いいえ"| S4["4. Bot を公開"]
    S4 --> S5["5. F1 e2e, 全件, review_scope=none\n→ EvalRun メモに『公開済み』"]
    S5 --> S6["6. 手順2 の F3 人判定を待つ(2営業日)\n議題があれば校正会へ"]
```

## 図の検証方法

mermaid-cli で全図を描画して構文を確認します(Chromium が必要)。

```bash
mkdir -p /tmp/mmd && npx -y -p @mermaid-js/mermaid-cli mmdc -i docs/09-diagrams.md -o /tmp/mmd/diagrams.md
```

Markdown を入力にすると、各コードブロックが `diagrams-1.svg`, `diagrams-2.svg` … として出力されます。エラーが出た図の番号を直してください。
