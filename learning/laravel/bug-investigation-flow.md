# バグ調査のたどり方（Laravel）
模擬案件での学習内容を、AIの補助を使って整理した復習用メモです。

- 対象リポジトリ：[Certify-LMS](https://github.com/shiho5shiho/Certify-LMS)
- 題材：B-B-16「修了したユーザーがプラン機能にアクセスできてしまう」
- [学習記録の一覧に戻る](../README.md)

## 基本の考え方

**リクエストが通る順番にたどる。** ルートとコントローラーの間には「ミドルウェア」がある点に注意する。

```
URL にアクセス
  ↓
① ルート（routes/web.php）        … どの URL で、どのコントローラーを呼ぶか
  ↓
② ミドルウェア                      … ログイン・ロール・受講状態などのチェック
  ↓
③ コントローラー                    … 処理の入り口
  ↓
④ UseCase / Action / Policy        … 実際の処理や、細かい権限のチェック
  ↓
⑤ ビュー（Blade）                   … 画面の表示
```

毎回①から全部見る必要はない。バグの種類から「怪しい場所」の見当をつけて、そこを重点的に見る。

| バグの種類 | 重点的に見る場所 |
|---|---|
| アクセスできてはいけない人がアクセスできる | ① ルート → ② ミドルウェア → ④ Policy |
| 表示される値がおかしい | ③ コントローラー → ④ UseCase → ⑤ ビュー |
| 保存される値がおかしい | FormRequest（バリデーション）→ ④ Action |

---

## 実例：B-B-16 修了したユーザーがプラン機能にアクセスできてしまう

### ① ルートで、プラン機能の URL を探す

チケットにある「教材閲覧」から、`/learning` のルートを探す。

```bash
grep -n "learning" routes/web.php
```

```php
Route::middleware(['auth', 'role:student', 'active-learning'])
    ->prefix('learning')
```

分かったこと：

- `active-learning` という、受講状態を見ていそうなミドルウェアが付いている
- ほかのプラン機能（chat / 質問掲示板など）にも同じものが付いている

```bash
grep -n "active-learning" routes/web.php
```

→ ルートへの付け方は正しい。**ミドルウェアの中身が怪しい**と見当をつける。

### ② ミドルウェアの別名から、本体のクラスを探す

`active-learning` は別名なので、`Kernel.php` で本体を調べる。

```bash
grep -rn "active-learning" app/Http/Kernel.php
```

```php
'active-learning' => EnsureActiveLearning::class,
```

### ③ ミドルウェアの中身を読む

`app/Http/Middleware/EnsureActiveLearning.php`

- クラスコメント：「受講中(in_progress)以外のユーザーをプラン機能から弾く Middleware」
- 実際の処理：`if ($user === null)`（未ログインしか見ていない）

→ **コメントと処理が食い違っている**ので、ここが原因と特定。

### ④ 裏付けを取る

| ファイル | 確認したこと |
|---|---|
| `app/Enums/UserStatus.php` | 受講中は `UserStatus::InProgress` |
| `app/Models/User.php` | `status` が `UserStatus` Enum にキャストされている |
| `tests/Feature/Middleware/EnsureActiveLearningTest.php` | テストが「受講中以外は 403」を期待している |

今回はミドルウェアで原因が見つかったので、コントローラーより先は読まなかった。

### 修正内容

```php
use App\Enums\UserStatus;

// 修正前
if ($user === null) {

// 修正後
if ($user === null || $user->status !== UserStatus::InProgress) {
```

※ `$user->status` は Enum に変換されているので、文字列 `'in_progress'` と比べると常に「違う」と判定されてしまう。必ず `UserStatus::InProgress` と比べる。

---

## 原因を見つける手がかり

- **コメントと処理の食い違い**：コメントに書かれた「やりたいこと」を、処理がやっていないことがある
- **既存テストが期待している内容**：テストは「本来の正しい動き」を教えてくれる。修正前にテストを実行して、落ちるものを確認する
- **別名は本体までたどる**：ミドルウェアの別名は `app/Http/Kernel.php` で本体のクラスと結び付いている

## よく使うコマンド

```bash
# ルートでキーワードを探す（-H でファイル名も出すと、Ctrl + クリックで飛べる）
grep -Hn "キーワード" routes/web.php

# ルート一覧（URL・ミドルウェア・コントローラー）を表示する
sail artisan route:list --path=learning -v

# app 以下でキーワードを探す
grep -rn "キーワード" app

# 関係するテストだけ実行する
sail artisan test --filter=テストクラス名
```

## grep の使い方

`grep` は、**ファイルの中から、指定した文字を含む行を探すコマンド**。エディタの「検索（Ctrl + F）」をターミナルで行うイメージ。

```bash
grep [オプション] "探す文字" ファイル名またはフォルダ名
```

### よく使うオプション

| オプション | 由来 | 意味 |
|---|---|---|
| `-n` | number | 見つかった行の**行番号**も表示する |
| `-r` | recursive | **フォルダの中を全部**（サブフォルダも含めて）探す |
| `-H` | — | 見つかった行の**ファイル名**も表示する |
| `-i` | ignore case | 大文字・小文字を区別せずに探す |

オプションはまとめて書ける（`-rn` は `-r -n` と同じ意味）。

### 使い分け

| 書き方 | 使う場面 | 表示例 |
|---|---|---|
| `grep -Hn "文字" ファイル` | 探すファイルが決まっているとき | `routes/web.php:162:Route::middleware(...)` |
| `grep -rn "文字" フォルダ` | どのファイルにあるか分からないとき | `app/Http/Kernel.php:91:'active-learning' => ...` |
| `grep -rni "文字" フォルダ` | 大文字・小文字の違いも含めて探したいとき | `app/Enums/UserStatus.php:11:case Graduated = ...` |

### 注意：`-n` だけだとファイル名が出ない

ファイルを1つだけ指定して `grep -n` とすると、結果は行番号だけになる。

```
112:Route::middleware(...)    ← ファイル名がないので、Ctrl + クリックしても飛べない
```

→ ファイルを1つ指定するときは **`-Hn`**、フォルダを指定するときは **`-rn`** にしておく。

### B-B-16 で使った grep

| コマンド | 何を調べたか |
|---|---|
| `grep -Hn "learning" routes/web.php` | 教材閲覧のルートと、付いているミドルウェア |
| `grep -Hn "active-learning" routes/web.php` | 同じミドルウェアが付いている、ほかのプラン機能 |
| `grep -rn "active-learning" app bootstrap` | ミドルウェアの別名の登録場所（→ `app/Http/Kernel.php`） |

## 結果の行に飛ぶ方法（VS Code）

| 方法 | やり方 |
|---|---|
| ターミナルの結果から飛ぶ | `ファイル名:行番号` の部分を **Ctrl + クリック** |
| 開いているファイルの中で飛ぶ | エディタをクリック → **Ctrl + G** → 行番号を入力 |
| ファイル名と行番号で開く | **Ctrl + P** → `routes/web.php:162` と入力 |
| コマンドで開く | `code -g routes/web.php:162` |

## ミドルウェアの別名の登録場所

| Laravel のバージョン | 登録場所 |
|---|---|
| 10 以前（Certify LMS は 10） | `app/Http/Kernel.php` の `$middlewareAliases` |
| 11 以降 | `bootstrap/app.php` |

場所が分からないときは、`grep -rn "別名" app bootstrap` で探すか、`sail artisan route:list --path=URL -v` で本体のクラス名を直接表示する。

## PR の「調査の足取り」に書くこと

調べた順番が、そのまま「調査の足取り」になる。

1. どのファイルを見て
2. 何が分かり
3. そこから次にどこを見たか
4. 最終的に特定した根本原因
