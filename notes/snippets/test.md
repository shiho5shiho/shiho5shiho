# テストとは？
## 1. テストケースを「何を作るか」決める概念

これはAAAより手前の、設計段階の話です。

### 正常系・異常系
期待通りに動くケース（正常系）と、不正な入力やエラーを起こすケース（異常系）を両方用意する。異常系を忘れるとバグが本番で出ます。

### 同値分割
入力を「同じ結果になるグループ」に分け、各グループから代表値を1つだけ選ぶ。例えば「1〜100が有効」なら、有効グループから50、無効グループから-1を選べば十分。全部の数字を試す必要はない、という考え方です。

### 境界値分析
バグは境界に潜むので、境界とその前後を狙う。「1〜100が有効」なら0, 1, 100, 101をテストする。同値分割とセットで使います。

## 2. 1つのテストを「どう書くか」の概念 = AAA

作ると決めたケースを、この3ステップで書きます。

### Arrange（準備）= 前提条件
テストの土台を整える部分。モデルやテストデータを作る、DBに初期状態を用意する、Factoryでレコードを生成する、などがここ。「テスト対象を動かすために、何が揃っている必要があるか」を書きます。

```
// Arrange: Bookとそれに紐づくReviewを用意
$book = Book::factory()->create();
Review::factory()->count(3)->create(['book_id' => $book->id]);
```

### Act（実行）= 操作
テストしたい対象を 1つだけ 実行する。メソッド呼び出し、リレーションのアクセス、アクセサの取得など。ここが「検証したい振る舞い」そのものです。

```
// Act: reviewsリレーションを取得
$reviews = $book->reviews;
```

### Assert（検証）= 期待結果
実際の結果が期待通りかを比較する。設計段階で決めた「このケースならこうなるはず」をコードにする部分。

```
// Assert: 3件返ることを確認
$this->assertCount(3, $reviews);
```

## つながりを一言で

設計段階で **同値分割・境界値分析** を使って「どんなケースが必要か」を洗い出し、**正常系・異常系** で網羅する。そして洗い出した1ケースずつを **AAA** で実装する、という流れです。

補足すると、モデルの単体テストで主にArrangeに来るのはFactory/Seederによるデータ準備、Actはリレーションやキャスト・アクセサの呼び出し、AssertはassertInstanceOf（型の確認）やassertEquals（値の確認）あたりになります。


# 単体テスト
テーブル毎にテストを実装するのが望ましい。


# Featureテストを書くときの汎用的思考
テストを書く前に毎回こう自問する：
>このリクエストは、処理のどこまで到達する？

到達する所までに必要なものだけArrangeで用意すればいい。  
手前で止まるテストは準備が軽く、奥まで進むテストは準備が重い。

# PHPUnitがdataProviderをどう使うか
dataProviderは「テストメソッドに引数を外から流し込む仕組み」。こう動く：
```
public static function 認証必須GETページ(): array
{
    return [
        ['/books/create'],
        ['/genres'],
        ['/genres/1'],
        // ...
    ];
}

#[DataProvider('認証必須GETページ')]
public function test_未認証はログイン画面にリダイレクトされる(string $url): void
{
    $this->get($url)->assertRedirect('/login');
}
```
## 0. 全体像 — AAA とは「1テストの3ブロック構造」

1つのテストメソッドは、**空行で区切った3ブロック**で書く。

```php
public function test_〇〇できる(): void
{
    // Arrange（準備）：テストの土台を用意する
    $book = Book::factory()->create();

    // Act（実行）：検証したい振る舞いを "1つだけ" 叩く
    $reviews = $book->reviews;

    // Assert（検証）：結果が期待どおりか比べる
    $this->assertCount(3, $reviews);
}
```

| ブロック    | 役割                 | モデル単体テストで主に来るもの                     |
| ----------- | -------------------- | -------------------------------------------------- |
| **Arrange** | 前提条件を揃える     | Factory によるデータ生成（`make` / `create`）      |
| **Act**     | 対象の振る舞いを実行 | リレーションアクセス・集計クエリ・キャスト読み取り |
| **Assert**  | 期待結果と比較       | `assert*` 系メソッド                               |

> 読み方のコツ：**Act は必ず1つ**。Act が2つあるテストは「何を検証したいテストか」がぼやけているサイン。

---


## 1. Arrange（準備）のパターン

## `make`/`create`/`fresh` の使い分け
### make()
make() はメモリ上にモデルのインスタンスを組み立てるだけで、DBには保存しません。だから id や created_at のような「DB側が自動で埋める値」は入りません。使いどころは、DBを介さなくても検証できる純粋なロジックです。キャストやアクセサ(例えば日付を Carbon に変換する挙動、フルネームを組み立てるアクセサ)を確かめたいとき、わざわざ保存する必要がないので make() で十分、かつ速いです。
```
$book = Book::factory()->make(['published_at' => '2024-01-01']);
// $book->published_at が Carbon 型になっているか、を保存せず確認できる
```

### create()
create() はインスタンスを作ってそのままDBに保存します。保存した瞬間にDB側が id や created_at を埋めてくれるので、戻ってきたオブジェクトにはそれらが入っています。リレーションをたどるテストや、withCount・withAvg のような集計クエリのテストは、実際にレコードがDBに無いと成立しないので create() が必須です。
```
$book = Book::factory()->has(Review::factory()->count(3))->create();
// DBに本1件+レビュー3件が入るので、レビュー数の集計テストができる
```

### fresh()
fresh() は、すでに手元にあるモデルをDBから取り直すメソッドです。ここが少し分かりにくいところで、create() 直後の変数は「PHP側でこちらが指定した値」を保持したままです。それを fresh() すると、DBに実際に保存された値を、キャストを通した状態で取得し直せます。つまり「マイグレーションのカラム定義 → 保存 → キャスト」という往復が忠実に機能しているかを確かめたいときに使います。
```
$book = Book::factory()->create(['published_at' => '2024-01-01']);
$fresh = $book->fresh();
// DBに書き込まれ、読み戻したうえで published_at が期待通りか検証する
```
>道具に例えると、make() は「組み立てただけの試作品」、create() は「工場のラインに一度流して型番シールまで貼られた製品」、fresh() は「倉庫に入れたものを改めて取り出して検品する」感覚です。

>[!Tip] 判断の順番としては、まず「このテスト、DBに保存しないと成立しないか?」を問うのが早いです。
- 保存不要なら make()。
- 保存が要るなら create()。
- さらに「保存して読み戻した後の値」まで厳密に見たいなら create() + fresh()、
という三段階で選べば迷いにくいはずです。

### 1-1. `make()` と `create()` の使い分け ＝ 「DBが要るか」で決まる

```php
$book = Book::factory()->make();    // メモリ上に組み立てるだけ（DB保存しない）
$book = Book::factory()->create();  // DBにINSERTしてから返す
```

**判断基準：今テストしたい振る舞いは、DBを経由して初めて現れるか？**

| 対象の振る舞い                | 選ぶメソッド | 理由                           |
| ----------------------------- | ------------ | ------------------------------ |
| アクセサ / キャストの読み取り | `make()`     | モデルクラス内で完結。保存不要 |
| リレーション取得 / 集計クエリ | `create()`   | DBにレコードが無いと引けない   |

```php
// make() で十分な例：キャストはメモリ上のモデルでも効く
$book = Book::factory()->make(['published_date' => '2024-01-01']);
$date = $book->published_date;
$this->assertInstanceOf(Carbon::class, $date);
```

> 迷ったら「このメソッドは SQL を発行するか？」で考える。発行するなら `create()`。

### 1-2. 特定の値を仕込む ＝ 配列でオーバーライド

Factory のランダム値ではなく、**検証したい狙いの値**を渡すときは配列で上書きする。

```php
Review::factory()->for($book)->create(['rating' => 5]);
Review::factory()->for($book)->create(['rating' => 4]);
Review::factory()->for($book)->create(['rating' => 3]);
//                                     👆 平均が4.0になるよう意図的に固定
```

集計（平均・件数）や境界値を検証するテストでは、**期待値を自分で計算できる状態**にしておくのが要点。ランダム値のままだと Assert に書く数字が決まらない。

### 1-3. 関連付けヘルパー — `for()` / `count()`

```php
Book::factory()->count(3)->for($user)->create();
//              👆3件生成    👆$userに紐づける（user_idを自動セット）
```

- `count(n)` … n件まとめて生成
- `for($model)` … 親モデルに紐づける（`belongsTo` 側の外部キーを埋める）

### 1-4. 「わざと無関係データを混ぜる」テクニック

```php
// Arrange
$user = User::factory()->create();
Book::factory()->count(3)->for($user)->create();
Book::factory()->count(2)->create();  // 👈 別ユーザーの書籍。取得結果に含まれてはいけない
```

これがリレーションテストの**見せ場**。3件だけ用意すると「3件返る」しか言えないが、無関係な2件を混ぜておくと「**余計なものを拾っていない**（絞り込みが効いている）」まで証明できる。

---

## 2. Act（実行）のパターン

Act は「何を検証したいか」で3タイプに分かれる。ここが実質テストの主役。

### 2-1. リレーションアクセス

```php
$books = $user->books;   // hasMany
$user  = $book->user;    // belongsTo
$genres = $book->genres; // belongsToMany
```

プロパティとしてアクセスした瞬間にリレーションが引かれる。「リレーションが正しく定義されているか」を確かめる。

### 2-2. 集計クエリ（`withAvg` / `withCount`）

```php
$result = Book::withAvg('reviews', 'rating')->find($book->id);
//              👆reviewsのratingの平均を、reviews_avg_rating という属性に付けて返す

$result = Book::withCount('reviews')->find($book->id);
//              👆reviewsの件数を、reviews_count という属性に付けて返す
```

**命名規則が肝**：`with{集計}('{リレーション}', '{カラム}')` の結果は、`{リレーション}_{集計}_{カラム}` という属性名で生える。

| 書き方                         | 生える属性名         |
| ------------------------------ | -------------------- |
| `withAvg('reviews', 'rating')` | `reviews_avg_rating` |
| `withCount('reviews')`         | `reviews_count`      |
| `withSum('reviews', 'rating')` | `reviews_sum_rating` |

> なぜ `find()` と組み合わせるか：`withAvg` は**クエリビルダを返すだけ**なので、`find()`（or `get()` / `first()`）で実際に実行しないと結果が返らない。

### 2-3. 属性・キャストの読み取り

```php
$publishedDate = $book->published_date;
//               👆モデルの $casts で 'date' 指定されていれば Carbon が返る
```

DBには文字列で入っていても、モデルのキャスト設定で型変換された状態で読める。それを確認する。

---

## 3. Assert（検証）のパターン

### 3-1. 件数を数える — `assertCount`

```php
$this->assertCount(3, $books);   // $books が3件か
```

### 3-2. 全要素が条件を満たすか — `assertTrue` + `every`

```php
$this->assertTrue($books->every(fn ($book) => $book->user_id === $user->id));
//                       👆コレクションの全要素がコールバックを満たせば true
```

「絞り込みが効いている」の証明に使う。1件でも他人の本が混ざれば `every` が false になり落ちる。

### 3-3. `assertEquals` と `assertSame` の使い分け ★重要

| メソッド       | 比較                            | いつ使う                                 |
| -------------- | ------------------------------- | ---------------------------------------- |
| `assertEquals` | **値だけ**（型は緩い）`==` 相当 | 集計値など、DBエンジンで型が変わりうる値 |
| `assertSame`   | **値＋型**（厳密）`===` 相当    | 型まで保証したい値（件数、文字列など）   |

```php
// 集計値は assertEquals ─ 型がDB依存だから
$this->assertEquals(4.0, $result->reviews_avg_rating);
// ↑ SQLiteはfloat、MySQLは文字列 "4.0" で返す。assertSameだと環境依存で落ちる

// 件数は assertSame ─ intで返ると保証できるから
$this->assertSame(0, $result->reviews_count);
```

> 判断基準：**「型がDBエンジンや環境で変わりうるか？」** 変わりうるなら `assertEquals`、int/stringで確定なら `assertSame`。

### 3-4. `assertViewHas` の2つの使い方

assertViewHas('books') — 「booksという変数が渡されているか」だけ見る
assertViewHas('books', クロージャ) — 「渡されたbooksが条件を満たすか」まで見る

### 3-5. null チェック — `assertNull`

```php
$this->assertNull($result->reviews_avg_rating);  // レビュー0件 → 平均はnull
```

### 3-6. 型の確認 — `assertInstanceOf`

```php
$this->assertInstanceOf(Carbon::class, $publishedDate);  // Carbon型になっているか
```

キャストが効いているかの検証に使う。「日付として扱える型か」を型レベルで保証する。

---

## 4. パターン別まとめ（テストの種類 × AAA 対応表）

実装した4種類のテストを、AAAの各ブロックで何をしたかで一覧化。

### ① リレーションテスト

|             | 内容                                                       |
| ----------- | ---------------------------------------------------------- |
| **Arrange** | `create()` で親子データ生成 ＋ **無関係データを混ぜる**    |
| **Act**     | `$user->books` などリレーションにアクセス                  |
| **Assert**  | `assertCount` で件数 ＋ `assertTrue(every)` で絞り込み確認 |

```php
public function test_ユーザーが登録した書籍を取得できる(): void
{
    // Arrange
    $user = User::factory()->create();
    Book::factory()->count(3)->for($user)->create();
    Book::factory()->count(2)->create();  // 別ユーザーの書籍は含まれない

    // Act
    $books = $user->books;

    // Assert
    $this->assertCount(3, $books);
    $this->assertTrue($books->every(fn ($book) => $book->user_id === $user->id));
}
```

### ② 集計テスト（withAvg / withCount）

|             | 内容                                                      |
| ----------- | --------------------------------------------------------- |
| **Arrange** | `rating` を**固定値**で複数生成（期待平均を自分で決める） |
| **Act**     | `withAvg(...)->find()` で集計付きモデルを取得             |
| **Assert**  | `assertEquals`（型がDB依存のため）                        |

```php
public function test_書籍の平均評価を取得できる(): void
{
    // Arrange
    $book = Book::factory()->create();
    Review::factory()->for($book)->create(['rating' => 5]);
    Review::factory()->for($book)->create(['rating' => 4]);
    Review::factory()->for($book)->create(['rating' => 3]);

    // Act
    $result = Book::withAvg('reviews', 'rating')->find($book->id);

    // Assert
    $this->assertEquals(4.0, $result->reviews_avg_rating);
}
```

### ③ 境界値テスト（0件のとき）

|             | 内容                                             |
| ----------- | ------------------------------------------------ |
| **Arrange** | 子データを**あえて作らない**（0件という境界）    |
| **Act**     | 集計クエリを実行                                 |
| **Assert**  | 平均は `assertNull`、件数は `assertSame(0, ...)` |

```php
public function test_レビューがない書籍の平均評価はnullになる(): void
{
    // Arrange
    $book = Book::factory()->create();

    // Act
    $result = Book::withAvg('reviews', 'rating')->withCount('reviews')->find($book->id);

    // Assert
    $this->assertNull($result->reviews_avg_rating);
    $this->assertSame(0, $result->reviews_count);
}
```

> このテストは網羅のためだけでなく、**「レビュー0件の本はランキング対象外」という仕様の技術的根拠**になっている（平均がnull＝並び替えから除外できる）。

### ④ キャストテスト

|             | 内容                                         |
| ----------- | -------------------------------------------- |
| **Arrange** | キャスト対象カラムに**文字列**を渡して生成   |
| **Act**     | 属性を読み取る                               |
| **Assert**  | `assertInstanceOf` で型 ＋ `assertSame` で値 |

```php
public function test_出版日が日付型にキャストされる(): void
{
    // Arrange
    $book = Book::factory()->create(['published_date' => '2012-06-23']);

    // Act
    $publishedDate = $book->published_date;

    // Assert
    $this->assertInstanceOf(Carbon::class, $publishedDate);
    $this->assertSame('2012-06-23', $publishedDate->toDateString());
}
```

---

## 5. コードの読み方チートシート

テストを読むとき／書くときに毎回自問する順番。

1. **Arrange を見る** → 「何が前提として揃っている？」「無関係データは混ぜてある？」
2. **Act を見る** → 「叩いている振る舞いは1つか？」「リレーション / 集計 / キャストのどれ？」
3. **Assert を見る** → 「件数・値・型・nullのどれを見ている？」「`assertEquals` と `assertSame` の選択は正しい？」

### 選択の早見表

| 迷いポイント                      | 判断基準                                       |
| --------------------------------- | ---------------------------------------------- |
| `make()` か `create()` か         | SQLを発行する振る舞いなら `create()`           |
| `assertEquals` か `assertSame` か | 型がDB依存で変わりうるなら `assertEquals`      |
| 無関係データを混ぜるか            | 「絞り込み」を検証したいなら混ぜる             |
| `rating` をランダムか固定か       | 集計・境界を見るなら固定（期待値を確定させる） |

---

## 補足：テスト基盤の前提

- 全テストで `use RefreshDatabase;` → 各テストごとにDBを初期化し、テスト間の汚染を防ぐ
- テスト用DBは **SQLite インメモリ** → 高速・本番DBと分離
- ファイル構成：`tests/Unit/` に `BookTest` / `UserTest` / `GenreTest` / `ReviewTest` の4ファイル、計21ケース


