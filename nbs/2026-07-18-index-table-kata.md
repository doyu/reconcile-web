# learn/index-table kata — FastHTML の学び(2026-07-18)

`index()` の Ul → Table 書き換え(#6)で踏んだ地雷と学び。diff は3行相当、学びは6個。

1. **FT の文字列はデフォルトで HTML エスケープ**。`show(response.text)` が効かないのはこのため
   (`to_xml` が str を escape する)。信頼済み HTML は `NotStr` で明示 — `month_view()` の
   `status_html` 包みと同じ仕組み。
2. **route の同一性は path + name + methods**。name は関数の qualified name 由来なので、
   `create_app` 内の `index` は `create_app_index`。notebook からのホットスワップは
   `@rt('/', name='create_app_index')` と明示しないと「置き換え」でなく「追記」になり、
   先勝ちで旧ハンドラが生き続ける。
3. **Jupyter のカーネル状態はソース編集に追随しない**。typo 修正・セル削除・改名の後に
   旧オブジェクトが生き残る事故を3回踏んだ。復旧はハーネスから流し直し、確定は
   Restart & Run All(= セルの並びだけが真実)。
4. **リストは `*` で展開して渡す**(`Tbody(*rows)`)。逆に FT オブジェクトに `*` を付けると
   子要素に分解されてタグが消える(`Table(*thead)` の罠)。
5. **CSS は継承より直指定が勝つ**。`.has-missing {color}` は tr 止まりで、Jupyter の td 規則に
   負ける → `.has-missing td {color}`。さらに未 Trust の notebook は `<style>` を
   サニタイザが剥がす(Trust Notebook で解決)。構造は notebook、見た目はブラウザで検証。
6. **TestClient は Cookie jar を持つステートフル**。前セルの logout がリークして負けた。
   テストセルは root/app/client を毎回作る自己完結型に(この notebook の既存流儀の理由)。

その他: `import *` は fast.ai 流として採用(nbdev が `__all__` を生成するので公開 API は不変)。
ローカル起動は自己署名証明書必須(`sess_https_only=True`)。box 更新は `reconcile-deploy` 一発。
