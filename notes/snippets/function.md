# function()とは？
## function() = 処理をまとめて名前をつけたもの
```
function greet($name) {
    return "こんにちは、{$name}さん";
}

echo greet('田中');  // 名前で呼び出す
```
「引数を受け取って → 処理して → return で返す」が基本形です。

## 無名関数（匿名関数 / クロージャ）
**名前をつけずにその場で作る関数** です。

「一度しか使わないから、わざわざ名前をつけるほどでもない」というときに使います。
```
$greet = function ($name) {
    return "こんにちは、{$name}さん";
};

echo $greet('田中');
```
変数に代入できる = 関数を「値」として扱える、というのが最大のポイントです。
>[!Tip]なぜ必要か：関数に処理を渡せる
Laravel でよく見るこの形が、まさにそれです。
```
$books = Book::where(function ($query) {
    $query->where('title', 'like', '%PHP%')
          ->orWhere('author', 'like', '%PHP%');
})->get();
```
`where()` に「どう絞り込むか」という処理そのものを渡している。名前をつける意味がないので無名関数にしています。

## PHP の落とし穴：`use`
無名関数は、外側の変数を**自動では見てくれません** 。
```
$keyword = 'Laravel';

// ✖ $keyword が undefined
$books = Book::where(function ($query) {
    $query->where('title', 'like', "%{$keyword}%");
})->get();

// ○ use で明示的に渡す
$books = Book::where(function ($query) use ($keyword) {
    $query->where('title', 'like', "%{$keyword}%");
})->get();
```
この「外の変数を閉じ込める(close over)」性質からクロージャと呼ばれます。

## アロー関数（PHP 7.4〜）
1行で済む場合の短縮形。こちらは use 不要で外の変数を自動で参照します。
```
$titles = $books->map(fn($book) => $book->title);
```
`fn` は式ひとつだけ、`return` も書かない、というルールです。

## まとめ
|   |名前|外の変数|
|---|---|---|
|function name() {}|あり|見えない|
|function () {}|なし|`use` が必要|
|fn() =>|なし|自動で見える（1式のみ）|

JavaScript も考え方はほぼ同じで、`function(){}` と `() => {}` の関係が PHP の無名関数とアロー関数に対応します。ただし JS は `use` なしで外の変数が見える点が違います。
