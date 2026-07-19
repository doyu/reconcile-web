# learn/month-expand kata — HTMX の学び(2026-07-19)

issue #7(inline month expansion)より。#6 の続編。

1. **OOB swap は id 一致だけが頼り**。置換後の要素が id を持たないと次の OOB が迷子になる
   (▾ にも id=btn-{m} が必須 — panel レビューが実装前に検出)。ボタンは oob= 引数付きの
   共通コンストラクタで対称に。
2. **swap 先を常設 placeholder(同 id の Tr を outerHTML 置換)にすると冪等**。afterend +
   delete OOB 案の3リスク(連打増殖・OOB 迷子・detached swap)が構造ごと消えた。
   「レースを防ぐ」より「レースが存在しない形にする」。
3. **FastHTML は HX-Request ヘッダで fragment/full-page を切り替える**(core.py is_full_page)。
   TestClient で断片を検証するときは headers={'HX-Request': 'true'} 必須。
4. **テストは「どのレスポンスの仕様か」に置く**。expand の仕様(statement links)を collapse の
   テストに書いて偽 red を踏んだ。対称設計はテストも対称(expand=全部ある / collapse=空)。
5. リファクタで挙動が変わってもテストが黙っていたら、それはテストの穴(oob= 化で hx-swap-oob が
   消えたのに green のままだった)。
6. 実ブラウザ確認は Playwright で自動化できる(dev server は SessionMiddleware の
   https_only だけ外す3行パッチ。JupyUvi の代替)。
7. **パース禁止の不透明ブロックの見た目調整は CSS で** — status.md の H1 は DOM を触らず
   display:none だけで消した。契約・アーカイブ・収集スキルは無変更のまま表示だけ変える。

設計経緯: 初案 afterend+delete → cross-LLM panel + レビュー2巡で placeholder 冪等方式(v2)に。
経緯の詳細は issue #7 本文と編集履歴。
