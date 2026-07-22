# where() = 「条件で絞り込む」メソッド
SQLの WHERE 句を、PHPのメソッドとして書けるようにしたものです。

```
Book::where('genre_id', 3)->get();
// SELECT * FROM books WHERE genre_id = 3
```
 **基本形**
```
where('カラム名', '値')             // = で比較（第2引数が値のとき）
where('カラム名', '演算子', '値')   // 演算子を指定するとき
```
```
where('price', 1000)             // price = 1000
where('price', '>=', 1000)       // price >= 1000
where('title', 'like', '%星%')   // 部分一致
```

**つなげるとAND条件**
```
Book::where('genre_id', 3)
    ->where('price', '>=', 1000)
    ->get();
// WHERE genre_id = 3 AND price >= 1000
```
OR にしたいときは `orWhere()`。

>[!Important]
**大事なポイント：`get()` するまでSQLは実行されない**  
`where()` が返すのは「まだ実行していないクエリ（Builder）」です。だから条件を後から足していけます。
```
$query = Book::query();

if ($request->keyword) {
    $query->where('title', 'like', "%{$request->keyword}%");
}
if ($request->genre_id) {
    $query->where('genre_id', $request->genre_id);
}

$books = $query->get(); // ここで初めてSQL発行
```
検索フォームの実装はこの形が定番です。

### 派生メソッド
#### 【肯定系】
| メソッド                             | 第1引数        | 第2引数               | 生成されるSQL                                | 読み方                                  |
| ------------------------------------ | -------------- | --------------------- | -------------------------------------------- | --------------------------------------- |
| `where('title', 'コード')`           | カラム名       | 比較する値            | `title = 'コード'`                           | 「titleが"コード"と等しいもの」         |
| `where('rating', '>=', 4)`           | カラム名       | 演算子（第3引数に値） | `rating >= 4`                                | 「ratingが4以上のもの」                 |
| `whereIn('id', [1,2,3])`             | カラム名       | 値の配列              | `id IN (1,2,3)`                              | 「idが1か2か3のいずれかのもの」         |
| `whereNull('deleted_at')`            | カラム名       | なし                  | `deleted_at IS NULL`                         | 「deleted_atが空のもの」                |
| `whereBetween('price', [500, 2000])` | カラム名       | [下限, 上限] の配列   | `price BETWEEN 500 AND 2000`                 | 「priceが500以上2000以下のもの」        |
| `whereHas('reviews', fn($q) => ...)` | リレーション名 | クロージャ（条件）    | `EXISTS (SELECT ... FROM reviews WHERE ...)` | 「条件を満たすreviewsを持っている書籍」 |

>[!Tip] 
`whereHas` だけ第1引数がカラム名ではない点が最大の違いです。他はすべて「自分のテーブルのカラム」を指定しますが、whereHas は「リレーション名」を指定して別のテーブルを見に行きます。だから第2引数もクロージャで、「その別テーブルに対する条件」を書く形になります。


#### 【否定形】
| メソッド                                | 第1引数        | 第2引数              | 生成されるSQL                          | 読み方                             |
| --------------------------------------- | -------------- | -------------------- | -------------------------------------- | ---------------------------------- |
| `whereNotIn('id', [1,2,3])`             | カラム名       | 値の配列             | `id NOT IN (1,2,3)`                    | 「idが1,2,3のいずれでもないもの」  |
| `whereNotNull('deleted_at')`            | カラム名       | なし                 | `deleted_at IS NOT NULL`               | 「deleted_atに値が入っているもの」 |
| `whereNotBetween('price', [500, 2000])` | カラム名       | [下限, 上限] の配列  | `price NOT BETWEEN 500 AND 2000`       | 「priceが範囲外のもの」            |
| `whereDoesntHave('reviews')`            | リレーション名 | クロージャ（省略可） | `NOT EXISTS (SELECT ... FROM reviews)` | 「reviewsを1件も持っていないもの」 |

>[!Tip]
最後の `whereDoesntHave` だけ `whereNotHas` ではない点に注意です。「持っていない」を自然な英語にすると doesn't have になるためで、Laravelの命名は「英文として読めるか」を優先しています。


>[!warning] Collection の where() とは別物
`get()` した後のコレクションにも `where()` がありますが、こちらはDBではなくPHPのメモリ上でフィルタします。  
全件取得してから絞るので非効率です。絞り込みは必ず get() の前に書くのが原則です。