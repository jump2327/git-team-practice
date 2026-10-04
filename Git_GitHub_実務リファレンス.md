# Git / GitHub 実務リファレンス

> 業務中に「この場合どうするんだっけ？」となったときに見返すための実践用メモ。\
> PowerShell + GitHub
> を前提に、日常操作・PR・レビュー・競合・復旧を状況別に整理しています。

------------------------------------------------------------------------

## 1. まず覚えておく全体像

Gitでは、変更はおおむね次の順番で進みます。

``` text
Working Tree（作業中のファイル）
        │
        │ git add
        ▼
Staging Area / Index（次にコミットする変更）
        │
        │ git commit
        ▼
Local Repository（自分のPCの履歴）
        │
        │ git push
        ▼
Remote Repository（GitHub）
```

### 状態確認の基本

迷ったら、まずこれ。

``` powershell
git status
```

履歴を見たい場合：

``` powershell
git log --oneline --graph --decorate --all -10
```

変更内容を確認：

``` powershell
git diff
```

ステージ済みの変更を確認：

``` powershell
git diff --staged
```

------------------------------------------------------------------------

# 2. 普段の業務で使う基本フロー

## 作業開始時

まず `main` を最新にします。

``` powershell
git switch main
git pull
```

次に作業ブランチを作ります。

``` powershell
git switch -c feature/xxx
```

例：

``` powershell
git switch -c feature/add-login
```

### 基本イメージ

``` text
main
  │
  A──B
      \
       C  ← feature/add-login
```

原則として、**main上で直接開発しない**ようにします。

------------------------------------------------------------------------

## ファイルを変更したら

状態確認：

``` powershell
git status
```

変更確認：

``` powershell
git diff
```

ステージ：

``` powershell
git add ファイル名
```

例：

``` powershell
git add .\hello.cpp
```

すべてステージする場合：

``` powershell
git add .
```

コミット：

``` powershell
git commit -m "Add login validation"
```

GitHubへpush：

``` powershell
git push -u origin feature/add-login
```

最初のpush後は通常、

``` powershell
git push
```

だけでOK。

------------------------------------------------------------------------

# 3. branch の操作

## ブランチ一覧

``` powershell
git branch
```

リモートも含める：

``` powershell
git branch -a
```

追跡関係も確認：

``` powershell
git branch -vv
```

## ブランチを切り替える

``` powershell
git switch main
```

## 新規作成して切り替える

``` powershell
git switch -c feature/xxx
```

## ローカルブランチ削除

マージ済み：

``` powershell
git branch -d feature/xxx
```

強制削除：

``` powershell
git branch -D feature/xxx
```

`-D` は未マージのコミットも消せるので注意。

------------------------------------------------------------------------

# 4. origin / main / origin/main の違い

``` text
main
```

自分のPCにあるローカルブランチ。

``` text
origin
```

リモートリポジトリにつけられた名前。通常はGitHub。

``` text
origin/main
```

「最後に取得したGitHub側mainの状態」をローカルで表す参照。

確認：

``` powershell
git remote -v
```

------------------------------------------------------------------------

# 5. fetch と pull の違い

## git fetch

``` powershell
git fetch
```

GitHubの最新情報を取得しますが、**自分の作業ブランチには自動で混ぜません**。

安全に状況確認したいときに便利です。

``` text
GitHub
  │
  │ fetch
  ▼
origin/main

main はそのまま
```

## git pull

``` powershell
git pull
```

基本的には、

``` text
fetch + 取り込み
```

です。

状況が分からないときに闇雲に `pull` するより、

``` powershell
git fetch
git status
git log --oneline --graph --decorate --all -10
```

で確認する方が安全です。

------------------------------------------------------------------------

# 6. Pull Request（PR）の基本

典型的な流れ：

``` text
main
 ↓
featureブランチ作成
 ↓
修正
 ↓
add
 ↓
commit
 ↓
push
 ↓
GitHubでPR作成
 ↓
レビュー
 ↓
必要なら修正
 ↓
push
 ↓
承認
 ↓
merge
```

PR作成前に、GitHub上で

-   Base: `main`
-   Compare: 自分のfeatureブランチ
-   変更ファイル
-   コミット数

を確認します。

意図していないコミットが大量に表示された場合は、**そのままPRを作らず原因を確認**します。

------------------------------------------------------------------------

# 7. レビュー指摘を受けたとき

同じfeatureブランチで修正します。

``` powershell
git switch feature/xxx
```

修正後：

``` powershell
git diff
git add .
git commit -m "Address review feedback"
git push
```

既存PRに自動的に新しいコミットが追加されます。

新しいPRを作る必要はありません。

------------------------------------------------------------------------

# 8. コンフリクト（競合）

例えば同じ行を別々に変更すると、

``` text
<<<<<<< HEAD
自分側の変更
=======
取り込もうとしている変更
>>>>>>> other-commit
```

のようになります。

必要な内容に手作業で修正し、

``` powershell
git add ファイル名
```

その後、実行していた処理に応じて続行します。

mergeの場合：

``` powershell
git commit
```

rebaseの場合：

``` powershell
git rebase --continue
```

cherry-pickの場合：

``` powershell
git cherry-pick --continue
```

中止したい場合：

``` powershell
git merge --abort
git rebase --abort
git cherry-pick --abort
```

------------------------------------------------------------------------

# 9. restore / reset / revert の使い分け

ここは非常に重要です。

## restore：ファイルの変更を戻す

まだコミットしていない変更を捨てる：

``` powershell
git restore ファイル名
```

例：

``` powershell
git restore .\hello.cpp
```

ステージだけ取り消す：

``` powershell
git restore --staged ファイル名
```

------------------------------------------------------------------------

## reset：ブランチの位置を戻す

直前のコミットを取り消して変更を残す：

``` powershell
git reset --soft HEAD~1
```

コミットとステージを取り消し、ファイル変更は残す：

``` powershell
git reset HEAD~1
```

変更も含めて完全に戻す：

``` powershell
git reset --hard HEAD~1
```

### 注意

`--hard` は追跡中ファイルの変更を消します。

実行前に、

``` powershell
git status
git log --oneline -5
```

で確認する癖をつけます。

------------------------------------------------------------------------

## revert：公開済みコミットを打ち消す

``` powershell
git revert コミットID
```

例：

``` powershell
git revert abc1234
```

元のコミットを削除するのではなく、

``` text
A──B──C──D
         ↑
      Cを打ち消すコミット
```

のように履歴を残したまま元に戻します。

### 目安

``` text
まだ自分だけのローカル履歴
→ reset が候補

すでにpushして他人と共有している
→ revert が安全
```

------------------------------------------------------------------------

# 10. main に間違えてコミットした場合

今回の練習でも発生した重要パターンです。

例えば、

``` text
main
  │
  └── 656c93f  ← 間違えて作ったコミット
```

本来featureブランチに入れたかった場合。

## 必要なコミットをfeatureへ移す

``` powershell
git switch feature/xxx
git cherry-pick 656c93f
```

競合したら解消して、

``` powershell
git add .
git cherry-pick --continue
```

featureをpushしてPRへ反映します。

## PRがmainへマージされた後

まず、

``` powershell
git switch main
git fetch
git status
```

状況を確認。

ローカルmainの誤コミットが不要で、GitHubのmainを正とすると判断できた場合：

``` powershell
git reset --hard origin/main
```

これで、

``` text
HEAD -> main
          │
          ▼
      origin/main
```

と一致させられます。

**状況確認なしで `reset --hard` しないこと。**

------------------------------------------------------------------------

# 11. cherry-pick

別ブランチの「特定コミットだけ」を現在のブランチへ持ってきます。

``` powershell
git cherry-pick コミットID
```

例：

``` powershell
git cherry-pick 656c93f
```

イメージ：

``` text
branch-A: A──B──C
               ↑
               Cだけ欲しい

branch-B: A──D
             │
             └─ cherry-pick C
```

結果としてbranch-Bには、Cと同じ変更を持つ**新しいコミット**が作られます。

------------------------------------------------------------------------

# 12. stash

作業途中だけれど、一時的に別ブランチへ移りたい場合。

``` powershell
git stash
```

一覧：

``` powershell
git stash list
```

戻す：

``` powershell
git stash pop
```

特定のstashを適用：

``` powershell
git stash apply 'stash@{0}'
```

PowerShellでは `stash@{0}` をクォートしておくと安全です。

stashは「一時退避場所」であり、正式な履歴として残すものではありません。

------------------------------------------------------------------------

# 13. rebase

featureブランチの土台を最新mainへ付け替えるイメージです。

変更前：

``` text
A──B──C  main
    \
     D──E  feature
```

rebase後：

``` text
A──B──C  main
       \
        D'──E'  feature
```

例：

``` powershell
git switch feature/xxx
git fetch
git rebase origin/main
```

rebaseはコミットを書き換えるため、共有済みブランチでは扱いに注意します。

------------------------------------------------------------------------

# 14. .gitignore

Gitに管理させたくないファイルを指定します。

例：

``` gitignore
*.exe
secret.txt
build/
```

重要：

**`.gitignore` はファイルを削除する機能ではありません。**

また、すでにGit管理下にあるファイルを書いても、それだけでは追跡解除されません。

追跡だけ外す場合：

``` powershell
git rm --cached secret.txt
```

その後：

``` powershell
git commit -m "Ignore secret file"
```

------------------------------------------------------------------------

# 15. ブランチを切り替えても untracked ファイルが残る理由

Gitが切り替えるのは基本的に**Gitが追跡しているファイル**です。

例えば、

``` text
secret.txt
```

がuntrackedなら、

``` powershell
git switch main
```

しても物理ファイルはそのまま残ることがあります。

確認：

``` powershell
git status
```

不要なファイルをPowerShellで削除する場合：

``` powershell
Remove-Item .\secret.txt
```

------------------------------------------------------------------------

# 16. リモートブランチの整理

GitHub側でブランチを削除した後も、

``` text
origin/feature/xxx
```

がローカルに残って見えることがあります。

整理：

``` powershell
git fetch --prune
```

または：

``` powershell
git remote prune origin
```

ローカルブランチも不要なら：

``` powershell
git branch -d feature/xxx
```

------------------------------------------------------------------------

# 17. 「ahead」「behind」「diverged」の意味

## ahead

``` text
Your branch is ahead of 'origin/main' by 1 commit.
```

ローカルにだけコミットがあります。

``` text
origin/main
     ↓
A──B──C
      ↑
     main
```

## behind

GitHub側の方が進んでいます。

## diverged

例えば：

``` text
Your branch and 'origin/main' have diverged,
and have 1 and 3 different commits each
```

これは、

``` text
          X  ← local main
         /
A──B────
         \
          C──D──E ← origin/main
```

のように双方に固有コミットがある状態です。

**この表示が出たら、反射的に `git pull` しない。**

まず：

``` powershell
git status
git log --oneline --graph --decorate --all -10
```

で履歴を確認します。

------------------------------------------------------------------------

# 18. reflog：消えたコミットを探す

resetなどでコミットが見えなくなっても、すぐなら `reflog`
から見つけられる場合があります。

``` powershell
git reflog
```

例：

``` text
abc1234 HEAD@{0}: reset: moving to HEAD~1
def5678 HEAD@{1}: commit: Important change
```

必要ならコミットIDを使って、

``` powershell
git switch -c recovery def5678
```

のように救出できます。

------------------------------------------------------------------------

# 19. diff / show / blame

## 未コミット変更を見る

``` powershell
git diff
```

## 特定コミットを見る

``` powershell
git show コミットID
```

## 誰がどの行を変更したか確認

``` powershell
git blame ファイル名
```

業務では障害調査や変更理由の追跡にも便利です。

------------------------------------------------------------------------

# 20. tag

リリース地点などに名前を付けられます。

``` powershell
git tag v1.0.0
```

一覧：

``` powershell
git tag
```

GitHubへ送る：

``` powershell
git push origin v1.0.0
```

------------------------------------------------------------------------

# 21. 業務中の逆引き

  --------------------------------------------------------------------------------------
  状況                                まず使うコマンド
  ----------------------------------- --------------------------------------------------
  今どうなっている？                  `git status`

  履歴を見たい                        `git log --oneline --graph --decorate --all -10`

  変更内容を見たい                    `git diff`

  mainを最新にしたい                  `git switch main` → `git pull`

  作業ブランチを作りたい              `git switch -c feature/xxx`

  GitHubへ送りたい                    `git push -u origin ブランチ名`

  ファイル変更を捨てたい              `git restore ファイル名`

  addを取り消したい                   `git restore --staged ファイル名`

  作業を一時退避したい                `git stash`

  退避を戻したい                      `git stash pop`

  特定コミットだけ欲しい              `git cherry-pick <commit>`

  公開済み変更を打ち消したい          `git revert <commit>`

  GitHubの情報だけ更新したい          `git fetch`

  消えたコミットを探したい            `git reflog`

  リモート削除済みbranch表示を整理    `git fetch --prune`

  競合処理を中止したい                `git merge --abort` / `git rebase --abort` /
                                      `git cherry-pick --abort`
  --------------------------------------------------------------------------------------

------------------------------------------------------------------------

# 22. トラブル時にまず実行する3つ

何が起きたか分からなくなったら、むやみに操作せず：

``` powershell
git status
git branch -vv
git log --oneline --graph --decorate --all -10
```

この3つで、

-   現在のブランチ
-   未コミット変更
-   upstream
-   mainとorigin/mainの位置
-   ブランチの分岐
-   最近のコミット

をかなり把握できます。

------------------------------------------------------------------------

# 23. 特に慎重に扱うコマンド

以下は便利ですが、意味を確認してから実行します。

``` powershell
git reset --hard
git branch -D
git push --force
git clean
```

特に共有ブランチに対する強制pushは、他メンバーの履歴へ影響する可能性があります。

分からないときは、まず：

``` powershell
git status
git log --oneline --graph --decorate --all -10
```

------------------------------------------------------------------------

# 24. 実務でのおすすめ習慣

1.  作業前に現在のブランチを確認する。
2.  `main` では直接修正せずfeatureブランチを作る。
3.  `git add` 前に `git diff` を見る。
4.  `git commit` 前に `git status` を見る。
5.  `git push` 前にコミット内容を確認する。
6.  PRでは「Files changed」を必ず確認する。
7.  意図しないコミットがPRに入っていたらマージしない。
8.  トラブル時は操作を増やす前に状態確認する。
9.  共有済み履歴では `reset` より `revert` を優先して考える。
10. `reset --hard` やforce pushは「何が消えるか」を理解してから使う。

------------------------------------------------------------------------

# 25. 最低限これだけ覚える

日常業務では、まず以下が自然に使えれば十分です。

``` powershell
git status
git switch
git switch -c
git diff
git add
git commit
git fetch
git pull
git push
git log
```

そしてトラブル対応として、

``` powershell
git restore
git stash
git revert
git reset
git cherry-pick
git reflog
```

を「必要なときに調べながら使える」状態を目指せばOKです。

------------------------------------------------------------------------

## 最後に

Gitで重要なのは、コマンドの丸暗記よりも、

``` text
今どのブランチにいるか
↓
変更はどの段階にあるか
↓
ローカルとGitHubのどちらが進んでいるか
↓
どの履歴を残したいか
```

を判断できることです。

迷ったら、まず `git status`。

``` powershell
git status
```

それでも分からなければ履歴を可視化。

``` powershell
git log --oneline --graph --decorate --all -10
```

この2つを確認してから次の操作を決めるのが、安全なGit運用の基本です。
