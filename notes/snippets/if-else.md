## if / else の三項演算子

### 条件 ? 条件が真のときの値 : 条件が偽のときの値

```
'average_rating' => $this->reviews_avg_rating !== null   // ← 条件
    ? round((float) $this->reviews_avg_rating, 2)        // ← 真のとき（? の後ろ）
    : null,                                              // ← 偽のとき（: の後ろ）
```
同じ処理を`if`で書くと以下。
```
if ($this->reviews_avg_rating !== null) {
    $averageRating = round((float) $this->reviews_avg_rating, 2);
} else {
    $averageRating = null;
}
```
※配列の中に if 文は書けないので、配列の値として条件分岐したいときは三項演算子を使う、という事情もあります。

### 読み方　
`x > 0 ? a : b`　「x は 0 より大きいの? → だったら a、そうでなければ b」

つまり ? は if ではなく、疑問符そのもの。「これって成り立つ?」と問いかけて、その答えを後ろに並べる。: は「または／さもなくば」を表す区切り記号で、case: や辞書の見出しのような「ここから中身」的な用法の仲間です。

>[!TIP] 「これって成り立つ」？　成り立つならこっち　：　成り立たないならこっち　