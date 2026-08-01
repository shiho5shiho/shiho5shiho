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


# テストコードリファレンス（単体 ＋ Feature / AAA別）

BookShelf のテストコードを、**Arrange / Act / Assert のどこで・何を・なぜ使うか**の観点でまとめたもの。
後から「このパターンどう書くんだっけ」を引くための逆引き。

- **前半（1〜5章）… 単体テスト**（Issue #15 / `tests/Unit/`）
- **後半（6〜9章）… Feature テスト**（Issue #16 CRUD・#17 認証・#18 公開API / `tests/Feature/`）

---

## 00. 単体テストと Feature テストの違い — 「どの層を通すか」

同じ AAA でも、**テストが通る層**が違う。ここを最初に押さえると、Act に何を書くかが決まる。

|                       | 単体テスト（Unit）           | Feature テスト                                          |
| --------------------- | ---------------------------- | ------------------------------------------------------- |
| 通す層                | モデルクラスの中だけ         | ルート → コントローラ → レスポンス（HTTP丸ごと）        |
| **Act** で叩くもの    | リレーション・集計・キャスト | `$this->get()` / `post()` などの**HTTPリクエスト**      |
| **Assert** で見るもの | 値・型・件数                 | **レスポンス**（ステータス・画面・JSON）と **DBの状態** |
| ログイン              | 不要                         | 必要なら `actingAs($user)`                              |

> 一言で：単体は「部品が正しいか」、Feature は「画面から操作したとき正しく動くか」を見る。

---

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

### 3-4. null チェック — `assertNull`

```php
$this->assertNull($result->reviews_avg_rating);  // レビュー0件 → 平均はnull
```

### 3-5. 型の確認 — `assertInstanceOf`

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

---
---

# ▼ Feature テスト編（Issue #16〜#18）

`tests/Feature/`。HTTPリクエストを投げて、**レスポンス**と**DBの状態**を検証する。
Issue の切り方は #16 CRUD系 / #17 認証 / #18 公開API の3本。

---

## 6. Feature テストの Arrange / Act / Assert

単体テストとの違いは Act と Assert に集中している。

### 6-1. Arrange — 単体とほぼ同じ ＋ ログインユーザーの用意

```php
// Arrange
$user = User::factory()->create();          // 操作する人
$book = Book::factory()->for($user)->create();  // 操作対象
```

Feature 特有なのは「**誰がリクエストするか**」を用意する点。保護されたルートを叩くテストでは、ログインユーザーを Arrange で作る。

### 6-2. Act — HTTPリクエストを1本投げる

単体では `$user->books` のようにモデルを直接叩いたが、Feature では**HTTPメソッド**で叩く。

```php
$response = $this->get('/books');                    // 一覧を見る
$response = $this->post('/books', ['title' => '…']); // 登録する
$response = $this->put("/books/{$book->id}", [...]);  // 更新する
$response = $this->delete("/books/{$book->id}");      // 削除する
```

ログインが要るルートは `actingAs()` を前に噛ませる。

```php
$response = $this->actingAs($user)->post('/books', [...]);
//                👆このリクエストを$userとして送る
```

> 戻り値の `$response` に対して Assert を繋げていくのが Feature の型。

### 6-3. Assert — 「レスポンス」と「DBの状態」の2系統

Feature の Assert は2方向から確認する。**片方だけだと不十分**。

```php
$response->assertRedirect('/books');            // ①レスポンス：どこへ飛んだか
$this->assertDatabaseHas('books', ['title' => '吾輩は猫である']);  // ②DB：実際に保存されたか
```

- **レスポンス系**（`$response->assert*`）… 画面の挙動が正しいか
- **DB系**（`$this->assertDatabase*`）… データが正しく変わったか

「リダイレクトはしたが保存されていない」バグは①だけ見ると見逃す。だから両方見る。

---

## 7. レスポンス系 Assert（`$response->assert*`）

`$response` の戻り値に繋げるメソッド群。

| メソッド                      | 見るもの                                     |
| ----------------------------- | -------------------------------------------- |
| `assertOk()`                  | ステータス200か（`assertStatus(200)`の別名） |
| `assertStatus(201)`           | 任意のステータスコード（登録成功=201など）   |
| `assertRedirect('/books')`    | リダイレクト先                               |
| `assertForbidden()`           | 403（認可で弾かれたか＝Policyの検証）        |
| `assertSee('吾輩は猫である')` | 画面に文字列が表示されるか                   |
| `assertDontSee('…')`          | 画面に表示されて**いない**か                 |
| `assertJson([...])`           | JSONに指定の内容が含まれるか（API用）        |
| `assertJsonCount(3, 'data')`  | JSON配列の件数（API用）                      |
| `assertJsonStructure([...])`  | JSONのキー構造（API用）                      |

---

## 8. DB系 Assert（`$this->assertDatabase*`）

`RefreshDatabase` と組み合わせ、DBの行を直接見る。

| メソッド                                | 見るもの                                       |
| --------------------------------------- | ---------------------------------------------- |
| `assertDatabaseHas('books', [...])`     | その条件の行が**存在する**（登録・更新の確認） |
| `assertDatabaseMissing('books', [...])` | その条件の行が**存在しない**（削除の確認）     |
| `assertDatabaseCount('books', 3)`       | テーブルの行数                                 |

> 読み方：`Has` は「入ったか」、`Missing` は「消えたか」。登録テストは `Has`、削除テストは `Missing` とセットで覚える。

---

## 9. パターン別まとめ（Issue #16〜#18 × AAA）

### ① CRUD系テスト（Issue #16）

操作ごとに「レスポンス＋DB」の組み合わせが決まっている。

| 操作     | Act                         | レスポンス               | DB                                |
| -------- | --------------------------- | ------------------------ | --------------------------------- |
| 一覧表示 | `get('/books')`             | `assertOk` + `assertSee` | —                                 |
| 登録     | `post('/books', [...])`     | `assertRedirect`         | `assertDatabaseHas`               |
| 更新     | `put("/books/{id}", [...])` | `assertRedirect`         | `assertDatabaseHas`（新しい値で） |
| 削除     | `delete("/books/{id}")`     | `assertRedirect`         | `assertDatabaseMissing`           |

**登録テストの例：**

```php
public function test_書籍を登録できる(): void
{
    // Arrange
    $user = User::factory()->create();
    $genre = Genre::factory()->create();

    // Act
    $response = $this->actingAs($user)->post('/books', [
        'title'     => '吾輩は猫である',
        'isbn'      => '9781234567890',
        'genres'    => [$genre->id],
    ]);

    // Assert
    $response->assertRedirect('/books');                       // 画面：一覧へ戻る
    $this->assertDatabaseHas('books', ['title' => '吾輩は猫である']);  // DB：保存された
}
```

**削除テストの例：**

```php
public function test_書籍を削除できる(): void
{
    // Arrange
    $user = User::factory()->create();
    $book = Book::factory()->for($user)->create();

    // Act
    $response = $this->actingAs($user)->delete("/books/{$book->id}");

    // Assert
    $response->assertRedirect('/books');
    $this->assertDatabaseMissing('books', ['id' => $book->id]);  // DB：消えた
}
```

### ② 認証テスト（Issue #17）

「ログインの有無・持ち主かどうか」でアクセスが変わることを検証。**異常系**が主役。

| 検証したいこと               | Arrange           | Act                           | Assert                     |
| ---------------------------- | ----------------- | ----------------------------- | -------------------------- |
| 未ログインは弾かれる         | ユーザー作らない  | `get('/books')`               | `assertRedirect('/login')` |
| ログイン済みは入れる         | `actingAs($user)` | `get('/books')`               | `assertOk`                 |
| 他人の物は操作不可（Policy） | 別ユーザーで      | `delete("/books/{他人のid}")` | `assertForbidden`（403）   |

**未ログイン拒否の例：**

```php
public function test_未ログインユーザーは書籍一覧にアクセスできない(): void
{
    // Arrange（ログインユーザーを用意しない ＝ これ自体が前提）

    // Act
    $response = $this->get('/books');

    // Assert
    $response->assertRedirect('/login');  // ログイン画面へ飛ばされる
}
```

**Policy（他人の書籍は削除できない）の例：**

```php
public function test_他人の書籍は削除できない(): void
{
    // Arrange
    $owner = User::factory()->create();
    $other = User::factory()->create();
    $book  = Book::factory()->for($owner)->create();  // ownerの本

    // Act
    $response = $this->actingAs($other)->delete("/books/{$book->id}");  // otherが削除を試みる

    // Assert
    $response->assertForbidden();                            // 403で弾かれる
    $this->assertDatabaseHas('books', ['id' => $book->id]);  // 消えずに残っている
}
```

> ポイント：Policy テストは「弾かれた（403）」だけでなく「**データが残っている**」まで見ると完全。

### ③ 公開APIテスト（Issue #18）

JSONレスポンスの中身を検証。`getJson()` を使うと `Accept: application/json` が自動で付く。

| 検証したいこと | Act                             | Assert                               |
| -------------- | ------------------------------- | ------------------------------------ |
| 一覧が返る     | `getJson('/api/v1/books')`      | `assertOk` + `assertJsonCount`       |
| 中身が正しい   | `getJson("/api/v1/books/{id}")` | `assertJson` / `assertJsonStructure` |

```php
public function test_書籍一覧APIがJSONを返す(): void
{
    // Arrange
    Book::factory()->count(3)->create();

    // Act
    $response = $this->getJson('/api/v1/books');
    //                👆 getJson は Accept: application/json を自動付与

    // Assert
    $response->assertOk();
    $response->assertJsonCount(3, 'data');  // data配列に3件
}
```

```php
public function test_書籍詳細APIが正しい構造を返す(): void
{
    // Arrange
    $book = Book::factory()->create(['title' => '坊っちゃん']);

    // Act
    $response = $this->getJson("/api/v1/books/{$book->id}");

    // Assert
    $response->assertOk();
    $response->assertJson(['data' => ['title' => '坊っちゃん']]);        // 値の確認
    $response->assertJsonStructure(['data' => ['id', 'title', 'isbn']]); // キー構造の確認
}
```

> `assertJson`（値が合っているか）と `assertJsonStructure`（キーが揃っているか）は役割が違う。両方あると「形も中身も正しい」と言える。

---

## 10. Feature テスト 読み方チートシート

1. **Arrange** → 「誰がリクエストする？」（`actingAs` が要るか）「操作対象は誰の物？」
2. **Act** → 「どのHTTPメソッド？」（get/post/put/delete のどれか＝正常/異常のどちらを見るか）
3. **Assert** → 「**レスポンス**と**DB**の両方を見ているか？」片方だけは要注意

### 単体 vs Feature 早見表（総まとめ）

|                 | Act で叩くもの                            | Assert の代表                                            |
| --------------- | ----------------------------------------- | -------------------------------------------------------- |
| 単体            | `$model->relation` / `withAvg` / キャスト | `assertCount` / `assertEquals` / `assertInstanceOf`      |
| Feature（CRUD） | `post` / `put` / `delete`                 | `assertRedirect` + `assertDatabaseHas`/`Missing`         |
| Feature（認証） | `actingAs`→保護ルート                     | `assertRedirect('/login')` / `assertForbidden`           |
| Feature（API）  | `getJson`                                 | `assertJson` / `assertJsonStructure` / `assertJsonCount` |
