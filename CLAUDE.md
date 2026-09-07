# このリポジトリの作業ルール

Copilot Studio 契約書レビューBot の評価ループ設計書(docs/01〜11)とテンプレート(templates/)を管理しています。

## 設計変更時のルール

- **設計を変更したら、docs/09-diagrams.md の対応する図を同じコミットで必ず更新する。** フロー(子フロー、F0〜F6)、データモデル、ループ構造、ライフサイクル、活用の流れ、リリース手順のいずれかに触れたら該当する図を直す。新しいフローやテーブルを追加したら図も追加し、docs/09 冒頭の対応表に行を足す。
- 図は Mermaid で書く(GitHub でそのまま描画されるため)。ノードのラベルは必ずダブルクォートで囲む。
- 語彙(列名、指標名、Q1/Q2、誤抑制/誤通過、fewshot など)は docs 間で揃える。変更したら grep で全文書を確認する。
- ルーブリック(docs/05)とプロンプトの境界事例は同じ文で揃える。

## 文書の対応

| 変更対象 | 直す文書 |
|---|---|
| ループ構造・2モード・構造化出力 | docs/01, docs/09 図1a・1b・2 |
| Dataverse テーブル・ビュー | docs/02, docs/09 図3 |
| フロー手順 | docs/03, docs/09 図4〜10 |
| 審査プロンプト | docs/04 |
| 業務部門向けルーブリック | docs/05 |
| 指標・ゲート・運用・リリース | docs/06, docs/09 図13 |
| アノテーション収集 | docs/07, docs/09 図2・7・11 |
| アノテーション活用 | docs/08, docs/09 図9・12 |
| CSV / JSON テンプレート | templates/、docs/02 の列定義 |
| Teams カードの UI・入力 ID・チャネル対応 | docs/10, templates/adaptive_card_*.json, docs/images/(JSON 変更時はモックアップを再生成) |
| カードの実装手順(アクション・数式) | docs/11, templates/flow_*.json, templates/powerfx_*.txt |

## 検証

- `python3 -m json.tool templates/*.json`
- CSV は UTF-8 BOM 付きで全行の列数が一致すること
- Mermaid は mermaid-cli(`mmdc`)で描画できること(手順は docs/09 末尾)
- Adaptive Card JSON は `python3 -m json.tool` でパースでき、docs/10 末尾の手順でモックアップを再生成すること
