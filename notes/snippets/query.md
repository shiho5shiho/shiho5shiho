# クエリとは？
クエリ（query）= データベースへの問い合わせ文のことです。

日本語だと「照会」で、要するに「こういうデータをくれ」とDBに出す注文書です。
```
SELECT * FROM books WHERE title LIKE '%コード%' ORDER BY id DESC LIMIT 20;
```
👆「booksテーブルから、タイトルに"コード"を含むものを、id降順で20件ください」という注文書

```
// 手書きだとこうなる（やりたくない）
$sql = "SELECT * FROM books";
if ($keyword) {
    $sql .= " WHERE title LIKE '%{$keyword}%'";
}
if ($genreId) {
    $sql .= $keyword ? " AND " : " WHERE ";   // ← この判断が面倒
    $sql .= "genre_id = {$genreId}";
}
```
これを肩代わりしてくれるのが クエリビルダ（query builder = クエリを組み立てる人）

# クエリビルダ本体とは？
「ここまでに積んだ条件を全部覚えている、組み立て途中のオブジェクト」 

中身をイメージすると、こんなメモを持っている状態
```
Book::query()              // メモ：{ テーブル: books }
  ->with('genres')         // メモ：{ テーブル: books, 事前読込: [genres] }
  ->withCount('reviews')   // メモ：{ ..., 集計: [reviews_count] }
  ->where('title', ...)    // メモ：{ ..., 条件: [title LIKE ...] }
  ->orderByDesc('id')      // メモ：{ ..., 並び順: [id DESC] }
  ```
各メソッドは **メモに書き足しているだけ** で、この時点ではまだDBに何も送っていません。
そして最後に get() や paginate() を呼んだ瞬間、溜めたメモからSQL文を組み立てて、DBに送信して結果を受け取ります。

# PHPの関数呼び出しルール
「渡した順番に、定義した引数へ順番に入る」
```
function 何か($a, $b) {
    // $a には1番目に渡された値
    // $b には2番目に渡された値
}

何か("りんご", "みかん");   // $a="りんご", $b="みかん"
何か(100, 200);            // $a=100, $b=200
```
中身が何であれ、順番だけで対応が決まります。名前は関係ありません。