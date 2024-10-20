git

不要在根目录建库

用命令`git checkout -b *name*`来创建一个新的分支。

别人(远端)做出的改动，要用merge合并

git merge other  把other合并到当前分支 

gitche自己（本地）做出的改动，用rebase

git rebase main  把当前的分支合并到main

------

##### 先分支（）、checkout（取消暂存区工作区更改）、reset和stash（写错了）(HEAD、reflog)、rebase（合并提交）、merge(冲突)、revert撤销提交（保留提交记录则不能用reset）

------

## 一、撤销提交

一种常见的场景是，提交代码以后，你突然意识到这个提交有问题，应该撤销掉，这时执行下面的命令就可以了。

> ```bash
> $ git revert HEAD
> ```

上面命令的原理是，在当前提交后面，新增一次提交，抵消掉上一次提交导致的所有变化。它不会改变过去的历史，所以是首选方式，没有任何丢失代码的风险。

`git revert` 命令只能抵消上一个提交，如果想抵消多个提交，必须在命令行依次指定这些提交。比如，抵消前两个提交，要像下面这样写。

> ```bash
> $ git revert [倒数第一个提交] [倒数第二个提交]
> ```

`git revert`命令还有两个参数。

> - `--no-edit`：执行时不打开默认编辑器，直接使用 Git 自动生成的提交信息。
> - `--no-commit`：只抵消暂存区和工作区的文件变化，不产生新的提交。

## 二、丢弃提交

软撤销 --soft
本地代码不会变化，只是 git 转改会恢复为 commit 之前的状态

不删除工作空间改动代码，撤销 commit，不撤销 git add .
```bash
git reset --soft HEAD~1
```
1
表示撤销最后一次的 commit ，1 可以换成其他更早的数字

硬撤销
本地代码会直接变更为指定的提交版本，慎用

删除工作空间改动代码，撤销 commit，撤销 git add .

注意完成这个操作后，就恢复到了上一次的commit状态。
```
git reset --hard HEAD~1
```
1
如果仅仅是 commit 的消息内容填错了
输入

git commit --amend
1
进入 vim 模式，对 message 进行更改

还有一个 --mixed
```
git reset --mixed HEAD~1
```
1
意思是：不删除工作空间改动代码，撤销commit，并且撤销git add . 操作
这个为默认参数,git reset --mixed HEAD~1 和 git reset HEAD~1 效果是一样的。

------





## 三、替换上一次提交

提交以后，发现提交信息写错了，这时可以使用`git commit`命令的`--amend`参数，可以修改上一次的提交信息。

> ```bash
> $ git commit --amend -m "Fixes bug #42"
> ```

它的原理是会使用与当前提交相同的父节点进行一次新提交，旧的提交会被取消。

这时如果暂存区有发生变化的文件，会一起提交到仓库。所以，`--amend`不仅可以修改提交信息，还可以整个把上一次提交替换掉。

## 四、撤销工作区的文件修改

如果工作区的某个文件被改乱了，但还没有提交，可以用`git checkout`命令找回本次修改之前的文件。

> ```bash
> $ git checkout -- [filename]
> ```

它的原理是先找暂存区，如果该文件有暂存的版本，则恢复该版本，否则恢复上一次提交的版本。

注意，工作区的文件变化一旦被撤销，就无法找回了。

git restore 

## 五、从暂存区撤销文件

如果不小心把一个文件添加到暂存区，可以用下面的命令撤销。

> ```bash
>     $ git rm --cached [filename]
> ```

上面的命令不影响已经提交的内容。

git restore --staged 

git reset HEAD XXX/XXX/XXX.java 

## 方法四：使用 ‘git stash’ 命令

如果你在执行了 ‘git reset’ 操作之后，发现你之前的更改没有提交，你可以使用 ‘git stash’ 命令来暂存你的更改，然后恢复到之前的状态。下面是一些使用 ‘git stash’ 撤销 ‘git reset’ 的步骤：

1. 打开命令行工具，切换到你的项目目录下。
2. 运行以下命令来暂存你的更改：`git stash`
3. 现在，你的更改被暂存到 Git 的”储藏区”中，你可以切换到之前的状态进行工作。
4. 如果你想恢复你的更改，运行以下命令：`git stash apply`
5. 现在，你的更改被恢复，你可以继续你的工作。

使用 ‘git stash’ 命令可以暂存工作区和暂存区的更改，以便在 ‘git reset’ 操作之后恢复它们。



# git压缩

有两种方法可以实现 Git 压缩：

- git rebase -i 作为用于压缩提交的交互式工具

- git merge -squash 在合并时使用 -squash 选项

 ## 使用交互式 git rebase 工具压缩 Git 提交

#### 以下是使用交互式变基工具压缩最后 X 次提交的命令的语法。

```bash
git rebase -i HEAD~[X]`
```

#### 使用 git merge -squash 压缩 Git 提交

以下是将分支与当前分支（通常是 main）合并并压缩源分支的提交的命令语法。

```text
git merge --squash <source_branch_name_to_squash>
```

当我们使用 --squash 选项执行 merge 时，Git 不会像在正常合并中那样在目标分支中创建合并提交。相反，Git 接受源分支中的所有更改。feature1 并将其作为本地更改放入目标分支即 master 的工作副本中。

------

## git cherry-pick

`git cherry-pick`命令的作用，就是将指定的提交（commit）应用于其他分支。

> ```bash
> $ git cherry-pick <commitHash>
> ```

上面命令就会将指定的提交`commitHash`，应用于当前分支。这会在当前分支产生一个新的提交，当然它们的哈希值会不一样。

举例来说，代码仓库有`master`和`feature`两个分支。

> ```bash
>     a - b - c - d   Master
>          \
>            e - f - g Feature
> ```

现在将提交`f`应用到`master`分支。

> ```bash
> # 切换到 master 分支
> $ git checkout master
> 
> # Cherry pick 操作
> $ git cherry-pick f
> ```

上面的操作完成以后，代码库就变成了下面的样子。

> ```bash
>     a - b - c - d - f   Master
>          \
>            e - f - g Feature
> ```

从上面可以看到，`master`分支的末尾增加了一个提交`f`。

`git cherry-pick`命令的参数，不一定是提交的哈希值，分支名也是可以的，表示转移该分支的最新提交。

> ```bash
> $ git cherry-pick feature
> ```

上面代码表示将`feature`分支的最近一次提交，转移到当前分支。



## 取消rebase

```text
$ git reset --hard ORIG_HEAD
```

ORIG_HEAD 是由大幅移动你的 HEAD 的命令创建的，以在操作前记录 HEAD 的位置，以便你可以轻松地将分支的尖端更改回运行它们之前的状态。

当我们进行了一些「危险操作」时，比如 `git reset`、`git merge`、`git rebase` 等操作时，Git 会将当前 `HEAD` 指向的的 Commit-ID 原值保存至 `ORIG_HEAD` 文件内。需要注意的是，类似 `git commit` 等操作并不会更新 `ORIG_HEAD` 的内容。

这样的话，加入我们执行了一些「误操作」时，可以利用 `git reset --hard ORIG_HEAD` 回退至上一步。

## FETCH_HEAD

`git pull` 等价于 `git fetch` + `git merge FETCH_HEAD` 两个步骤的结合。

- `git fetch origin main`
   将远程仓库的 `main` 分支最新 Commit-ID 记录到 `.git/FETCH_HEAD` 中，此时 `FETCH_HEAD` 指向该 Commit-ID。
- `git merge FETCH_HEAD`
   将 `FETCH_HEAD` 对应的 Commit-ID 合并至本地 `main` 分支中。如果合并过程不存在冲突（即只是 Fast-Forward），那么可以顺利完成 `git pull` 最后一个步骤，否则的话，需要手动解决冲突。



git cherry_pick



## 稀疏检出

1. ```bash
   git remote add -f origin https://***/test.git 或 git@***/test.git
   git config core.sparsecheckout true
   
   echo "clone_file" >> .git/info/sparse-checkout
   
   // 例如仓库test下面的testone文件夹
   echo "test/testone" >> .git/info/sparse-checkout
   
   git pull origin main
   ```


