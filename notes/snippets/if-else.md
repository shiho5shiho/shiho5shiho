## if / else の三項演算子

条件 ? 条件が真のときの値 : 条件が偽のときの値

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