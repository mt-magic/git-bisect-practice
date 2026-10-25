# git-bisect-practice
「git bisect」のハンズオン

# ハンズオン

git bisectは、バグが混入したコミットを二分探索（バイナリサーチ）で高速に特定するための非常に強力なGitのコマンドです。「Rebase and merge」などのクリーンなリニア履歴（一本道の履歴）を採用すると、このgit bisectの効果がより発揮されやすくなります。

実際に手を動かして体感できるハンズオン形式の練習手順を作成しました。

## 1. 練習用のリポジトリを作る

まずは手元（ローカル）にテスト用のGitリポジトリを作成し、意図的にバグが混入するまでのコミット履歴を作ります。

### 手順1: リポジトリの初期化と最初のコミット（正常な状態）

```Bash
mkdir git-bisect-practice
cd git-bisect-practice
git init

# 簡易的な計算アプリ（ファイル）を作成
echo 'def add(a, b):' > app.py
echo '    return a + b' >> app.py

git add app.py
git commit -m "feat: 初期実装 (add関数)"
```

### 手順2: いくつかの機能追加コミット（正常）

```Bash
# 適当な機能追加（例としてコメントや別の関数を追加）
echo 'def sub(a, b):' >> app.py
echo '    return a - b' >> app.py

git add app.py
git commit -m "feat: sub関数を追加"
```

### 手順3: バグが混入するコミット（「Rebase and merge」などでマージされた想定）

ここで、PRで誤ってバグ（足し算なのに引き算になってしまう）が混入したと仮定します。

```Bash
# バグを混入させる（return a - b に書き換えるなど）
cat << 'EOF' > app.py
def add(a, b):
    return a - b  # バグ：足し算なのに引き算している

def sub(a, b):
    return a - b
EOF

git add app.py
git commit -m "fix: 計算処理を最適化 (ここにバグが混入)"
```

### 手順4: さらにその後のコミットをいくつか重ねる

バグが混入したあとに、さらに開発が進んだ状況を再現します。

```Bash
echo '# version 1.2' >> app.py
git add app.py
git commit -m "docs: バージョン情報を追加"

echo '# ready for production' >> app.py
git add app.py
git commit -m "chore: リリース準備"
```

これで、合計5つのコミット履歴ができました。現在の最新（HEAD）ではバグが含まれている状態です。

## 2. git bisect でバグ混入コミットを特定する

ここからが本番です。「最新のコードでは動かない（バグがある）」けれど、「初期のコミットの頃は動いていた」という状況から、どのコミットでバグが混入したのかを自動で探します。

### ステップ1: bisectを開始する

```Bash
git bisect start
```

## ステップ2: 現在の状態（最新）が「バグっている（bad）」ことを教える

```Bash
git bisect bad
```

### ステップ3: 過去の「正常だった（good）」コミットを教える

最初のコミットのハッシュ（または HEAD~4 など）を指定します。

```Bash
# 履歴を確認して最初のコミットハッシュを調べるか、相対指定する
git bisect good HEAD~4
```

※コマンドを実行すると、Gitが自動的に「ちょうど真ん中のコミット」にチェックアウトしてくれます。

### ステップ4: チェックアウトされた状態で動作確認する

Gitが案内してくれたコミットの状態に移動しているので、テスト（今回の場合はPythonスクリプトの動作確認など）を行います。

```Bash
# テスト用の簡易スクリプトを実行して確認してみる
python3 -c "import app; print(app.add(2, 3))"
```

期待値: 5
結果: 2 - 3 なので -1 になっていれば「バグっている（bad）」、5 なら「正常（good）」です。

### ステップ5: 判定をGitに伝える

確認結果に応じて、以下のいずれかのコマンドを入力します。

- バグっている場合: git bisect bad
- 正常な場合: git bisect good

これを数回繰り返すと、Gitが自動的に「最初にバグを持ち込んだ犯人のコミット（The first bad commit）」を特定して画面に表示してくれます！

### ステップ6: bisectを終了する

調査が終わったら、通常のブランチに戻ります。

```Bash
git bisect reset
```

## 3. なぜ「Rebase and merge」が有利なのか？

このハンズオンを体験すると、なぜ「Rebase and merge（またはキレイなリニア履歴）」が git bisect と相性が良いのかが見えてきます。

履歴が一本道（Linear）であることのメリット:git bisect はコミット履歴を二分探索します。マージコミットが入り乱れる複雑なグラフ（Create a merge commit多用の環境）だと、分岐やコンフリクトの影響で中間地点のビルドやテストが通らない（あるいは調査ルートが複雑になる）ケースがあります。

「Rebase and merge」なら:
すべてのPRがきれいな一本のタイムラインとして直線状に並ぶため、git bisect が指し示す「真ん中のコミット」が確実にビルド・テスト可能な状態になりやすく、バグ調査の精度とスピードが圧倒的に上がるというメリットを体感できます。
