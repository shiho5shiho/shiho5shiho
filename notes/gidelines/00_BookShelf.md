# BookShelf 使用コードの観点まとめ

BookShelf の実装で使った Laravel / PHP のコードを、「どういう仕組みで、何のために使ったのか」という観点で整理した学習用ノート。

構成：

1. Eloquent でデータを絞る・集める
2. 入力を検証する（FormRequest）
3. 多対多を操作する
4. 出力を整形する（API Resource）
5. モデルの単体テスト

---

## 1. Eloquent でデータを絞る・集める

### when() の仕組み

`when()` は、「自分がコールバックを呼ぶときは、1番目にビルダ、2番目に値を渡す」という仕様。

```php
->when( 条件 , 実行する処理 )
//      第1引数    第2引数
```

- 第1引数：真偽を判定される「条件」
- 第2引数：条件が真だったときに実行される「関数」

```php
if ($value) {                   // ← 第1引数を条件として判定
    $callback($this, $value);   // ← 真なら第2引数の関数を実行し、値を渡す
}
```

第1引数は2つの役割を兼ねている。

1. 真偽の判定材料（実行するかどうかを決める）
2. 実行する場合に、関数へ渡される値

```php
$callback($this, $value);
//         │       └─ 材料（キーワードの文字列）
//         └─ 道具（クエリビルダ）
```

| 要素 | 役割 |
|---|---|
| `$this` / `$query` | 条件を書き足される対象（クエリビルダ）= 道具 |
| `$value` / `$keyword` | 書き足す条件に使う値（文字列）= 材料 |
| `->where(...)` | 実際の動作 |

**BookShelf で使った観点**：API の一覧取得で「キーワードが指定されたときだけ絞り込む」ために使用。`if` で囲むより、「値があればそれを使う」という頻出パターンを1つにまとめられる。

```php
Book::query()
    ->when($request->keyword, function ($query, $keyword) {
        $query->where('title', 'like', "%{$keyword}%")
              ->orWhere('author', 'like', "%{$keyword}%");
    })
    ->paginate(20);
```

---

### withCount() / withAvg() の仕組み

集計結果を **追加のカラムとして生やす** メソッド。カラム名は「渡した引数から機械的に組み立てられる」のがポイント。

```php
Book::withCount('reviews')
//               └─ リレーション名
// → 生えるカラム名： reviews _ count
//                   リレーション名  固定語

Book::withAvg('reviews', 'rating')
//            └─リレーション名  └─集計対象カラム
// → 生えるカラム名： reviews _ avg _ rating
//                   リレーション名 固定語 対象カラム
```

| 呼び出し | 生えるカラム名 |
|---|---|
| `withCount('reviews')` | `reviews_count` |
| `withAvg('reviews', 'rating')` | `reviews_avg_rating` |
| `withSum('reviews', 'rating')` | `reviews_sum_rating` |

つまりカラム名を覚える必要はなく、**「リレーション名 + 集計種別 + 対象カラム」を snake_case で繋ぐ**という規則で決まる。

**BookShelf で使った観点**：ランキング機能で「レビュー件数」と「平均評価」を1回のクエリで取得するため。N+1 を避けつつ、`orderByDesc('reviews_avg_rating')` のように **生えたカラムでそのまま並べ替え**できる。

```php
Book::withAvg('reviews', 'rating')
    ->withCount('reviews')
    ->has('reviews')                      // ← レビュー0件の書籍は除外
    ->orderByDesc('reviews_avg_rating')
    ->get();
```

> 補足：レビュー0件のとき `reviews_avg_rating` は `null`、`reviews_count` は `0` になる。この挙動が「レビューのない書籍はランキング対象外」という仕様の根拠になっている。

---

### with() と load() の仕組み

どちらも Eager Loading（関連データをまとめて先読みし、N+1 を防ぐ）だが、**「いつ読むか」のタイミングが違う**。

```php
// with() … クエリを作る段階で「一緒に取ってこい」と指示する
Book::with('reviews')->get();
//   ↑ まだ DB に行っていない。get() の時点で books と reviews を1度に取得

// load() … すでに取得済みのモデルに「後から足せ」と指示する
$book = Book::find($id);   // ← この時点では reviews は無い
$book->load('reviews');    // ← 後から reviews を読み込む
```

| | 対象 | タイミング |
|---|---|---|
| `with()` | これから取得するクエリ | 取得と**同時**に先読み |
| `load()` | 取得済みのモデル | 取得の**後**に追加読み |

判断基準：**まだ取得していない**なら `with()`、**もう手元にある**モデルに足したいなら `load()`。

**BookShelf で使った観点**：詳細画面（show）で、コントローラが `find` で受け取った1件の `$book` に対して関連（ジャンル・レビュー・レビュー投稿者・いいね）を後から足す必要があったため `load()` を使用。

```php
$book->load('genres', 'reviews.user', 'reviews.likedByUsers');
//                     └─ ドットで「レビューの、さらにその投稿者」まで先読みできる
```

---

## 2. 入力を検証する（FormRequest）

### FormRequest の責務

FormRequest は **「コントローラに入る前の関所」**。役割は入力の検証だけに限定される。

```
リクエスト → [ FormRequest（関所）] → コントローラ
                 │
                 ├─ rules()    … 通す条件
                 └─ messages() … 弾いたときの言葉
```

| メソッド | 役割 |
|---|---|
| `rules()` | どんな値なら通すか（検証ルール） |
| `messages()` | ルール違反時にどう伝えるか（エラーメッセージ） |
| `authorize()` | そもそもこのリクエストを許可するか |

**観点**：コントローラを「検証で汚さない」ため。コントローラに着いた時点で「値はもう正しい」と信頼できるので、本来の処理（保存・更新）に集中できる。

> つまずきポイント：`messages()` のキーは **実際のルール名と一致**していないと表示されない。BookShelf では `rating` のルールが `in:1,2,3,4,5` なのに `rating.between` というキーでメッセージを書いていて、メッセージが出ない不具合があった。正しくは `rating.in`。

```php
public function messages(): array
{
    return [
        'rating.in' => '評価は1〜5で選択してください。',
        //      └─ ルール名（in）と一致させる。ここが between だと出ない
    ];
}
```

---

### Rule::unique()->ignore() の仕組み

「重複禁止だが、**自分自身は例外**」を表現するためのルール。編集時に必要になる。

```php
Rule::unique('genres', 'name')->ignore($this->genre)
//           └─テーブル  └─カラム         └─ 除外する対象（編集中のレコード）
```

なぜ必要か：新規作成なら「同じ名前が既にあるか」を見るだけでいい。しかし **編集**では、名前を変えずに保存すると「自分と同じ名前が既にある（=自分）」と判定されて弾かれてしまう。`ignore()` で自分自身を検査対象から外す。

```
新規作成： unique だけでよい（自分はまだ存在しない）
編集    ： unique だと自分にぶつかる → ignore(自分) で除外が必要
```

**BookShelf で使った観点**：ジャンル名・書籍の ISBN など「一意であるべき値」の編集フォームで使用。作成用と更新用の FormRequest を分け、更新側だけ `ignore()` を付ける。

---

## 3. 多対多を操作する

### sync() と attach() の仕組み

中間テーブル（book_genre など）を更新するメソッド。**「差分を合わせる」か「足すだけ」か**が違う。

```php
$book->genres()->attach([1, 2]);   // ← 1,2 を追加（既存はそのまま）
$book->genres()->sync([1, 2]);     // ← 中間テーブルを [1,2] の状態に「合わせる」
```

`sync()` の動き：渡した配列を「あるべき最終状態」とみなし、

```
渡した配列に無いもの → 削除
渡した配列に有るもの → 追加
既にあって配列にも有る → そのまま
```

| メソッド | 動き |
|---|---|
| `attach()` | 足すだけ（重複しうる） |
| `detach()` | 外すだけ |
| `sync()` | 渡した配列の状態に一致させる（過不足を自動調整） |

**BookShelf で使った観点**：書籍編集でジャンルを付け替えるとき、「今チェックされているジャンルの状態にする」＝ `sync()` が最適。`attach`/`detach` を自前で組み合わせる必要がなく、フォームで送られた配列を1回渡すだけで済む。

```php
$book->genres()->sync($request->genre_ids);
//                    └─ フォームで選ばれたジャンルIDの配列 = あるべき最終状態
```

---

## 4. 出力を整形する（API Resource）

### FormRequest と Resource の責務分担

この2つは **入口と出口**で対になっている。

```
リクエスト → [ FormRequest（入力の検証）] → コントローラ → [ Resource（出力の整形）] → レスポンス
                     入口                                        出口
```

| | 担当 | 向き |
|---|---|---|
| FormRequest | 受け取る値が正しいか | 入力 |
| Resource | 返す値をどう見せるか | 出力 |

**観点**：「検証」と「整形」を混ぜないため。DB のカラムをそのまま JSON にすると、内部構造が外にダダ漏れになる。Resource を挟むことで「API として見せたい形」だけを出せる。

---

### Resource::collection() と new Resource() の仕組み

**1件か複数件か**で呼び分ける。

```php
new BookResource($book);              // ← 1件（単数）
BookResource::collection($books);     // ← 複数件（コレクション）
```

| 呼び方 | 対象 |
|---|---|
| `new BookResource($book)` | Model 1件 |
| `BookResource::collection($books)` | Collection / ページネーション結果 |

`collection()` は内部で「配列の各要素に Resource を1つずつ適用する」ので、複数件をまとめて同じ整形ルールに通せる。

**BookShelf で使った観点**：一覧 API は `collection()`、詳細 API は `new`。詳細では、より多くの情報を返すため専用の `BookDetailResource` を別に用意して使い分けた。

---

### whenLoaded() の仕組み

**「読み込まれていたら出す、なければ出さない」**を制御する。

```php
'reviews' => ReviewResource::collection($this->whenLoaded('reviews')),
//                                            └─ reviews が load 済みのときだけ含める
```

なぜ必要か：Resource の中で `$this->reviews` と直接書くと、**読み込んでいなくてもその場でクエリが飛ぶ（N+1 の温床）**。`whenLoaded()` を使うと「先読みしていないなら、そもそも出力に含めない」ので、意図しない追加クエリを防げる。

```
先読みしている → キーごと出力する
先読みしていない → キー自体を出さない（クエリも飛ばさない）
```

**観点**：一覧では関連を含めず軽く返し、詳細では `load()` してから返す、という **エンドポイントごとの出し分け**が、Resource 側を変えずに実現できる。

> 補足：集計カラム（`reviews_avg_rating` など）には `whenLoaded` は使わない。あれはリレーションではなく「クエリ時に生えた属性」なので、`whenLoaded` の判定対象（ロード済みリレーション）に当てはまらないため。

---

## 5. モデルの単体テスト

### Factory：make / create / for / count（DB を触るか）

テストデータの用意。**DB に保存するかどうか**が最大の分かれ目。

```php
Book::factory()->make();     // ← メモリ上に組み立てるだけ（DB に保存しない）
Book::factory()->create();   // ← DB に保存する（id や timestamps が付く）
```

| メソッド | DB 保存 | 使いどころ |
|---|---|---|
| `make()` | しない | 保存不要な検証（キャストの読み取りなど） |
| `create()` | する | id・リレーション・往復が必要な検証 |
| `for($parent)` | — | 親モデルに紐付けて生成 |
| `count(3)` | — | 3件まとめて生成 |

判断基準：**メモリ上でセットした値を見るだけ**なら `make()`。**DB を通った後の姿**（id・リレーション・スキーマとの整合）を見るなら `create()`。

```php
Book::factory()->count(3)->for($user)->create();
//              └─3件      └─$userに紐付け  └─DBに保存
```

**BookShelf で使った観点**：リレーションのテストは、外部キーで結びつく以上 id が必要なので `create()` 必須。関係ない他ユーザーの書籍もわざと数件混ぜて、「余計なものを拾っていない」ことまで検証した。

---

### RefreshDatabase + SQLite インメモリの仕組み

テストごとに **DB を毎回まっさらに戻す**仕組み。

```php
class BookTest extends TestCase
{
    use RefreshDatabase;   // ← 各テストの前にマイグレーションし、後で巻き戻す
```

なぜ必要か：前のテストが作ったデータが次のテストに残っていると、「3件のはずが4件あった」のような **テスト同士の汚染**が起きる。`RefreshDatabase` は各テストを独立させる。

さらに、テスト用 DB を **SQLite インメモリ**（メモリ上の使い捨て DB）にすると、

```
ファイルにも本番 MySQL にも書かない → 速い・汚さない・毎回まっさら
```

**観点**：`phpunit.xml` でテスト時だけ接続先を SQLite インメモリに差し替え、開発用データを汚さずに高速でテストを回せるようにした。

---

### assertEquals と assertSame の仕組み

どちらも「等しいか」を見るが、**型まで見るかどうか**が違う。

```php
$this->assertEquals(4.0, $value);   // ← 値が等しければ OK（型は問わない）
$this->assertSame(4.0, $value);     // ← 値も型も一致して初めて OK
```

| メソッド | 比較 | "4.0" と 4.0 は |
|---|---|---|
| `assertEquals` | 値のみ（`==`相当） | 通る |
| `assertSame` | 値 + 型（`===`相当） | 落ちる |

**BookShelf で使った観点**：集計値の平均評価は、**DB エンジンによって返る型が違う**（SQLite は float、MySQL は文字列で返すことがある）。ここで `assertSame` を使うと、環境が変わっただけでテストが落ちる。値が合っていればよい場面なので `assertEquals` を選んだ。

```php
// 平均評価：型が環境依存 → assertEquals（値だけ見る）
$this->assertEquals(4.0, $result->reviews_avg_rating);

// 件数：確実に int で返る → assertSame（型まで固定してよい）
$this->assertSame(3, $result->reviews_count);
```

判断基準：**型が環境や実装で揺れうる**なら `assertEquals`、**型まで保証したい・保証できる**なら `assertSame`。

---

## 巻末：観点の早見表

| コード | 一言で言うと | 判断の軸 |
|---|---|---|
| `when()` | 値があるときだけ条件を足す | 実行するか＋渡す値、を1つに |
| `withCount/withAvg` | 集計をカラムとして生やす | カラム名は「関係名+種別+列」で決まる |
| `with()` / `load()` | N+1 を防ぐ先読み | 未取得なら with、取得済みなら load |
| `Rule::unique()->ignore()` | 自分以外で重複禁止 | 編集では自分を除外する |
| `sync()` | 中間テーブルを状態に合わせる | 「足す」ではなく「一致させる」 |
| `FormRequest` / `Resource` | 入口の検証／出口の整形 | 検証と整形を混ぜない |
| `whenLoaded()` | 先読み済みだけ出す | 未ロードならクエリを飛ばさない |
| `make()` / `create()` | DB に保存するか | メモリで足りるか、往復が要るか |
| `RefreshDatabase` | テストごとに DB を戻す | テスト同士を独立させる |
| `assertEquals` / `assertSame` | 型まで見るか | 型が揺れるなら Equals |
