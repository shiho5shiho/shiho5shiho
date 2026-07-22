# Git Cheatsheet

GitHubへコードを管理するための基本コマンド。

> [!NOTE]
> 基本は VS Code を使用。
> ターミナルで操作が必要になった場合の早見表。

---

# 現在の状態を確認

## 変更されたファイルを確認

```bash
git status
```

---

# 変更をコミットする

## 変更をすべてステージング

```bash
git add .
```

## 特定のファイルだけ追加

```bash
git add ファイル名
```

例

```bash
git add app/Models/Task.php
```

## コミット

```bash
git commit -m "タスクCRUDを実装"
```

---

# GitHubへ反映

## mainブランチへPush

```bash
git push origin main
```

---

# 最新を取得

## GitHubから取得して反映

```bash
git pull origin main
```

---

# ブランチ確認

## 現在のブランチ

```bash
git branch
```

## ブランチ一覧

```bash
git branch -a
```

---

# コミット履歴

## 履歴を見る

```bash
git log
```

## 1行表示

```bash
git log --oneline
```

---

# 変更内容確認

## 変更箇所を見る

```bash
git diff
```

## ステージング済みとの差分

```bash
git diff --cached
```

---

# ファイルを元に戻す

## 変更を破棄

```bash
git restore ファイル名
```

例

```bash
git restore app/Models/Task.php
```

## ステージング解除

```bash
git restore --staged ファイル名
```

---

# よく使うGitHub

## リモート確認

```bash
git remote -v
```

---

# 困った時

## ブランチの削除
```
# 1. 別のブランチへ移動（自分自身は削除できないため）
git switch main

# 2. 削除
git branch -d feature/xxxxx
```

### `-d` と `-D` の違い

|オプション|挙動|
|---|---|
|`-d`|マージ済みのブランチだけ削除。未マージだとエラーで止まる（安全）|
|`-D`|未マージでも強制削除|


## 直前のコミットを取り消す（変更は残す）

```bash
git reset --soft HEAD~1
```

## コミットだけ取り消す（ステージングも解除）

```bash
git reset HEAD~1
```

> [!WARNING]
> `git reset --hard` は変更内容も削除されるため、基本的には使用しない。

---

# よく使う実装ループ

## 作業開始

```bash
git pull origin main
```

## 作業完了

```bash
git status
git add .
git commit -m "実装内容"
git push origin main
```

---

# GitHubでPull Requestを作る流れ

```text
Issue作成
    ↓
実装
    ↓
git add .
    ↓
git commit
    ↓
git push
    ↓
Pull Request作成
    ↓
レビュー
    ↓
Merge
```

---

# よく使うコマンド

```bash
git status             # 状態確認

git add .              # 変更を追加
git commit -m "msg"    # コミット
git push origin main   # GitHubへ反映

git switch main        # メインへ切り替え
git pull origin main   # 最新取得
git log --oneline      # 履歴確認
git diff               # 差分確認
git restore ファイル名 # 変更を戻す
```