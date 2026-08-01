# load() とは
すでに取得済みのモデル・コレクションに、あとからリレーションを読み込むメソッドです。
```
$book = Book::find(1);   // この時点ではリレーション未取得

$book->load('reviews');  // ここで reviews をまとめて取得
```
### load() 系メソッド一覧

| メソッド           | 読み方                   | 第1引数                                              | 第2引数                                     | 結果の取り出し方                   |
| ------------------ | ------------------------ | ---------------------------------------------------- | ------------------------------------------- | ---------------------------------- |
| `load()`           | ロード                   | リレーション名（文字列 / 配列 / クロージャ付き配列） | ―                                           | `$book->reviews`                   |
| `loadMissing()`    | ロード・ミッシング       | 同上                                                 | ―                                           | `$book->reviews`                   |
| `loadCount()`      | ロード・カウント         | 同上                                                 | ―                                           | `$book->reviews_count`             |
| `loadSum()`        | ロード・サム             | リレーション名                                       | 集計するカラム名                            | `$book->reviews_sum_score`         |
| `loadAvg()`        | ロード・アベレージ       | リレーション名                                       | 集計するカラム名                            | `$book->reviews_avg_score`         |
| `loadMin()`        | ロード・ミニマム         | リレーション名                                       | 集計するカラム名                            | `$book->reviews_min_score`         |
| `loadMax()`        | ロード・マックス         | リレーション名                                       | 集計するカラム名                            | `$book->reviews_max_score`         |
| `loadExists()`     | ロード・イグジスツ       | リレーション名                                       | ―                                           | `$book->reviews_exists`（bool）    |
| `loadAggregate()`  | ロード・アグリゲート     | リレーション名                                       | カラム名（第3引数に集計関数）               | `$book->reviews_max_score`         |
| `loadMorph()`      | ロード・モーフ           | ポリモーフィックのリレーション名                     | `[モデルクラス => [リレーション名]]` の配列 | `$comment->commentable->author`    |
| `loadMorphCount()` | ロード・モーフ・カウント | 同上                                                 | 同上                                        | `$comment->commentable->xxx_count` |

>[!Tip]
with() 側にはすべて同名の兄弟がいます。  
load        ↔ with  
loadCount   ↔ withCount  
loadAvg     ↔ withAvg  
loadExists  ↔ withExists


## load() が活きる場面
条件によってリレーションが必要かどうか変わるとき。
```
$book = Book::findOrFail($id);

if ($request->has('show_reviews')) {
    $book->load('reviews.user');  // 必要なときだけ読み込む
}
```

Route Model Binding（`public function show(Book $book)`）のように、**すでにモデルが渡されてくる**ケースでも `load()` が必須になります。`with()` を書く余地がないためです。

>[!Tip]補足:
 件数だけ欲しいときは `loadCount('reviews')`、すでに読み込み済みなら再クエリを避ける `loadMissing('reviews')` もあります。

 ## with() との違い
|          | タイミング           | 使う場所                                       |
| -------- | -------------------- | ---------------------------------------------- |
| `with()` | 取得と同時に読み込む | クエリビルダ（`Book::with('reviews')->get()`） |
| `load()` | 取得した後で読み込む | モデル・コレクションのインスタンス             |

どちらも「Eager Loading（N+1対策）」であることは同じで、実行タイミングが違うだけです。