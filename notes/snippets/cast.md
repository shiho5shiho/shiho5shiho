## キャスト(cast)とは？
「キャスト（cast）」は型変換のことです。実は今の説明で2種類のキャストが出てきているので、区別して説明します。

### 型キャスト（PHPの言語機能）
(float) $value のように、値の型を強制的に変換する書き方です。
```
(float) "4.6666"   // 文字列 → 浮動小数点数 4.6666
(int) "10"         // 文字列 → 整数 10
(string) 42        // 整数 → 文字列 "42"
```

### モデルの $casts（Eloquentの機能）
**「このカラムを取り出すとき、自動的にこの型に変換して」** というモデルへの設定
```
// app/Models/Book.php
protected $casts = [
    'published_date' => 'date',
];
```
DBのデータは、実はすべて文字列としてPHPに届きます。<br>
`published_date` も本来は "2012-06-23" という文字列です。<br>
しかし上の設定をしておくと、Eloquentが取り出す瞬間に Carbon という日付オブジェクトに変換してくれます。
```
// $casts なし
$book->published_date;                    // "2012-06-23"（文字列）
$book->published_date->format('Y-m-d');   // エラー：文字列にformat()はない

// $casts あり
$book->published_date;                    // Carbonオブジェクト
$book->published_date->format('Y/m/d');   // "2012/06/23"
$book->published_date->addDays(7);        // 日付計算もできる
```
つまり $casts は **「毎回自分で変換しなくて済むよう、モデル側で一括設定しておく仕組み」** です。<br>
日付以外にもよく使われます。
```
protected $casts = [
    'published_date' => 'date',        // Carbonへ
    'is_published'   => 'boolean',     // 0/1 → true/false
    'price'          => 'integer',     // "1500" → 1500
];
```
2つの関係
両方とも型変換ですが、タイミングと責任範囲が違います。
|   |(float) $value　　　　　　　　|$casts　　　　　　　　　|
|---|----------------------------|----------------       |
|何 |PHPの構文                    |Eloquentの設定         |
|いつ|その場で1回                 |モデルから取り出すたび自動|
|どこに書く| 使う場所（Resource等）|モデルクラス|