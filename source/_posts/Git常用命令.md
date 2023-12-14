# Git 常用命令

## 基本概念

> 以下基本概念相关的描述引自菜鸟教程：
>
> [Git 工作区、暂存区和版本库 | 菜鸟教程](https://www.runoob.com/git/git-workspace-index-repo.html)

- 工作区：就是你在电脑里能看到的目录
- 暂存区：英文叫 stage 或 index。一般存放在 .git 目录下的 index 文件（.git/index）中，所以我们把暂存区有时也叫作索引（index）
- 版本库：工作区有一个隐藏目录 .git，这个不算工作区，而是 Git 的版本库

下面这个图展示了工作区、版本库中的暂存区和版本库之间的关系：

![工作区、暂存区和版本库之间的关系](https://static.youfindme.cn/blog/git_command/git_structure.jpg)

- 图中左侧为工作区，右侧为版本库。在版本库中标记为 "index" 的区域是暂存区（stage/index），标记为 "master" 的是 master 分支所代表的目录树。
- 图中我们可以看出此时 "HEAD" 实际是指向 master 分支的一个"游标"。所以图示的命令中出现 HEAD 的地方可以用 master 来替换。
- 图中的 objects 标识的区域为 Git 的对象库，实际位于 ".git/objects" 目录下，里面包含了创建的各种对象及内容。
- 当对工作区修改（或新增）的文件执行 `git add` 命令时，暂存区的目录树被更新，同时工作区修改（或新增）的文件内容被写入到对象库中的一个新的对象中，而该对象的ID被记录在暂存区的文件索引中。
- 当执行提交操作（`git commit`）时，暂存区的目录树写到版本库（对象库）中，master 分支会做相应的更新。即 master 指向的目录树就是提交时暂存区的目录树。
- 当执行 `git reset HEAD` 命令时，暂存区的目录树会被重写，被 master 分支指向的目录树所替换，但是工作区不受影响。
- 当执行 `git rm --cached <file>` 命令时，会直接从暂存区删除文件，工作区则不做出改变。
- 当执行 `git checkout .` 或者 `git checkout -- <file>` 命令时，会用暂存区全部或指定的文件替换工作区的文件。这个操作很危险，会清除工作区中未添加到暂存区中的改动。
- 当执行 `git checkout HEAD .` 或者 `git checkout HEAD <file>` 命令时，会用 HEAD 指向的 master 分支中的全部或者部分文件替换暂存区和以及工作区中的文件。这个命令也是极具危险性的，因为不但会清除工作区中未提交的改动，也会清除暂存区中未提交的改动。

**为方便下面的描述，简单区分一下工作区、暂存区和版本库：**

- **工作区：本地的项目中没有通过 `git add` 添加到暂存区的文件**
- **暂存区：本地的项目中通过 `git add` 添加到暂存区的文件**
- **版本库：本地的项目中通过 `git commit` 添加到提交历史中的文件**

## 基本命令

- `git init`: 初始化一个本地git仓库repository
- `git status`: 查看本地仓库的状态
- `add`
  - `git add <file>`: 将工作区下的某个文件的修改加到暂存区（stage）
  - `git add .`: 将工作区下的当前目录下的修改加到暂存区（stage）
  - `git add -A`: 将工作区下的所有目录下的修改加到暂存区（stage）
  - `git add -u`: 将工作区下的所有目录下的修改加到暂存区（stage），但是不包括新添加的文件
- `commit`
  - `git commit -m "<some message>"`: 将暂存区（stage）提交到版本库，并写一些描述信息
  - `git commit -m "<some message>" --no-verify`: 将暂存区（stage）提交到版本库，并写一些描述信息，但是不会执行 `pre-commit` 钩子（**慎用**，通常在pre-commit钩子中做一些代码检查，如果不执行pre-commit钩子，那么就会跳过代码检查，这样就可能会导致一些问题）
  - `git commit`: 将暂存区（stage）提交到版本库，会弹出一个编辑器，让你输入本次提交的说明，编辑器是git配置的默认编辑器，一般是vim
  - `git commit --no-verify`: 同上，只不过不会执行 `pre-commit` 钩子（**慎用**）
- `push`
  - `git push`: 将本地的最新改动记录，推送到远端的和本地分支同名的分支下（前提是远端有同名分支）
  - `git push -f`: 强制推送，即使远端有比本地更新的提交记录，也会强制推送（**慎用**，可能会导致远端的提交记录丢失，执行这个之前务必知道自己在做什么）

## `clone` 命令

- `git clone git@server-name:path/repo-name.git`: 从远程仓库clone一个仓库到本地，同时会关联到远程仓库
- `git clone -b <branchName> git@server-name:path/repo-name.git`: 从远程clone指定分支到本地

## `merge` & `rebase` 命令

- `merge`
  - `git merge <branchName>`: 将指定branchName的分支合并到当前分支
  - `git merge --no-ff`: 表示在系统采取 `fast-forward` 时让系统采用普通合并方式，而不是采用快速合并，普通合并会多一次commit，这样在以后查看时能够看到合并历史，而 `fast-forward` 合并则看不到曾经做过的合并
- `rebase`
  - `git rebase <branchName>`: 将当前分支的改动，rebase到branchName分支上，这样就可以保证当前分支的提交记录是线性的，而不是分叉的，这样在以后查看提交记录时会更清晰。但是如果有冲突，可能需要多次解决冲突，比较麻烦
  - `git rebase -i`: 交互式rebase，可以在rebase的过程中，对提交记录进行修改，比如说合并提交记录、删除提交记录等
  - [Git rebase详细解析_GhostStories的博客-CSDN博客_git rebase 参数](https://blog.csdn.net/wangnan9279/article/details/79287631)

## `fetch` & `pull` 命令

- `git fetch`: 从远程取回所有分支的更新（但不会自动合并）
- `git fetch origin <branchName>`: 从远程 `origin` （`origin` 换成其他名字，则从对应的远程仓库获取更新）取回branchName分支的更新（但不会自动合并）
- `git pull`: 从远程拉取最新的修改到本地并合并，如果有冲突则需要在本地解决。`git pull` 相当于 `git fetch` 加上 `git merge`

## `push` 命令

- `git push -u origin master`: 第一次将本地的repository的master分支推送到远程，以后再push就不用加 `-u` 了，[关于-u的用法](https://www.zhihu.com/question/20019419/answer/13687984)
- `git push origin <branchName>`: 将分支推送到远程仓库 `origin` 的branch分支上，如果远程仓库 `origin` 中没有叫做 `branchName` 的分支，则会创建一个

如果 `git push` 省略了远程仓库名则默认会将本地记录推送到名为 `origin` 的远程仓库上，且目标推送分支是 `origin` 远程仓库中和本地仓库分支同名的分支

## `stash` 命令

- `git stash`: 把当前工作现场先存起来（不是add也不是commit），可以暂时去做其他事，之后（主要应用场景是当前手头工作没做完不能commit，但是又有紧急的bug需要切换到其他分支上去修复）
- `git stash list`: 查看所有存起来的工作现场
- `git stash pop`: 将stash list中，最顶部的工作现场恢复，并将list顶部的给删除掉
- `git stash apply <指定stash>`: 将指定的工作现场恢复，但是在stash list当中并不删除指定stash
- `git stash clear`: 情况stash list（**慎用**）

## `checkout` 命令

- `git checkout -- <file>`: 将工作区中file文件的修改撤销，包括把删除的file复原（在添加到暂存区之前才有效，如果将工作区文件的修改add到了暂存区，那么再使用这个命令就会无效，这时应该先把暂存区的修改撤销，再使用这个命令）
- `git checkout <branchName>`: 切换到branchName分支上
- `git checkout -b <branchName>`: 是上面 `git branch <branchName>` 和 `git checkout <branchName>` 两条命令合并之后的简写，即创建一个名为name的分支并切换到那个分支
- `git checkout <commit_id>`: 切换到指定的commit（可以是当前分支也可以是其他分支）

## `reset` 命令

- `git reset --hard HEAD^` / `git reset --hard HEAD~1`: 将git的HEAD指针指向版本库中上一个版本，同时也会把整个项目的文件回退到上一个版本。这个会把最近的一个commit以及对应的改动完全丢弃，不再保留。如需要多回退几个版本，可以使用 `HEAD~n`，`n` 表示回退几个版本
- `git reset --hard <commit_id>`: 将git的HEAD指针指向版本库中某个commit指定的版本，同时也会把整个项目的文件回退到这个版本。这个会把commit_id之后的所有commit以及对应的改动完全丢弃，不再保留
- `git reset --soft HEAD^` / `git reset --soft HEAD~1`: 将将git的HEAD指针指向版本库中上一个版本，但是不会修项目的文件，会把上一个版本的改动，保留到git暂存区（不会修改工作区，只改暂存区）。如果当前暂存区有内容，会将上一个版本的改动和暂存区的内容合并到暂存区
- `git reset --soft <commit_id>`:  将将git的HEAD指针指向版本库中某个commit指定的版本，但是不会修项目的文件，会把commit_id之后的所有commit对应的改动，保留到git暂存区（不会修改工作区，只改暂存区）。如果当前暂存区有内容，会将commit_id之后的所有commit对应的改动和暂存区的内容合并到暂存区
- `git reset --mixed HEAD^` / `git reset --mixed HEAD~1` / `git reset HEAD^` / `git reset HEAD~1`: 将git的HEAD指针指向版本库中上一个版本，但是不会修项目的文件，会把上一个版本的改动，保留到git工作区，同时也会将暂存区的内容清空。如果当前暂存区有内容，会将上一个版本的改动和暂存区的内容合并到工作区
- `git reset --mixed <commit_id>` / `git reset <commit_id>`: 将git的HEAD指针指向版本库中某个commit指定的版本，但是不会修项目的文件，会把commit_id之后的所有commit对应的改动，保留到git工作区，同时也会将暂存区的内容清空。如果当前暂存区有内容，会将commit_id之后的所有commit对应的改动和暂存区的内容合并到工作区

其中，`--hard`、`--soft`、`--mixed`是三种不同的模式，`--mixed`是默认的模式，如果不指定模式，那么就是`--mixed`模式。

可以理解为 `git reset --hard` 通常用于丢弃最近的提交，而另外两个通常用于修改最近的提交。

## `revert` 命令

- `git revert <commit_id>`: 撤销提交记录中间的某次commit，同时会生成一条新的commit记录
- `git revert -m <commit_id>`: 撤销提交记录中间的某次commit，同时会生成一条新的commit记录（这次commit是别的分支合并过来的）
- `git revert --no-commit commit_id1..commit_id2`: 撤销提交记录中间的几个连续的commit（注意这是一个左开右闭区间，即不包括 `commit_id1`，但包括 `commit_id2`）

## `cherry-pick` 命令

- `git cherry-pick <commit_id>`: 将另一个分支上的指定提交记录，摘抄到当前分支
- `git cherry-pick <commit_id1>..<commit_id2>`: 将另一个分支上的从`commit_id1`到`commit_id2`之间的提交记录摘抄到到当前分支

详见：[git 场景——从一个分支cherry-pick一个或多个commit到另一个分支](https://blog.csdn.net/w958796636/article/details/78492017)

## `branch` 命令

- `git branch`: 查看本地所有分支，当前所在分支前面有一个 `*` 号
- `git branch -r`: 查看远程所有分支
- `git branch -a`: 查看本地和远程所有分支
- `git branch -d <name>`: 删除名为name的分支
- `git branch -D <name>`: 强行删除名为name的分支，用 `-d` 删除的话，如果name分支没有被合并就无法删除，那么就要用 `-D`
- `git branch <name>`: 创建一个名为name的分支

## `log` 命令

- `git log`: 查看提交的日志信息（按q退出）
- `git log --pretty=oneline`: 查看只有一行的简略日志信息
- `git log --graph`: 查看日志信息的图示信息
- `git log --graph --pretty=oneline --abbrev-commit`: 查看日志信息的图示信息，且只有一行的简略日志信息，且commit id只显示前几位

## `remote` 命令

- `git remote`: 查看远程仓库信息，远程库默认名字是 `origin`
- `git remote -v`: 查看更详细的远程仓库信息
- `git remote add origin git@server-name:path/repo-name.git`: 关联一个远程库，或者说添加一个远程库（一个本地仓库可以有多个远程库），在这里添加的远程仓库名为 `origin`
- `git remote remove origin`: 删除已添加的远程仓库地址，这里删除的是名为 `origin` 的远程仓库
- `git remote set-url origin https://xxxxx.xxx/repo-name.git`: 更改远程仓库 `origin` 的url，比如说github项目地址改了，或者运维配置的gitlab域名变更了，可以用这个修改
- `git remote rename origin old-origin`: 重命名远程仓库，将远程仓库 `origin` 重命名为 `old-origin`

## `diff` 命令

- `git diff <filename>`: 不带参数就是比较工作区和暂存区中filename文件的不同
- `git diff --cached <filename>`: 比较暂存区和版本库中filename文件的不同
- `git diff HEAD <filename>`: 比较工作区和版本库中filename文件的不同

## 其他

### 1. 首次关联本地仓库到远端空仓库

1. `git remote add origin git@server-name:path/repo-name.git`: 关联一个远程库
2. `git push -u origin master`: 第一次将本地的repository的master分支推送到远程，以后再push就不用加 `-u` 了

### 2. push时其他人已经push了新的提交，被reject

如果多个人同时关联了远程库（多人协作时），在push的时候，发现有人已经向remote推送了他的提交，而你对同样的文件做了修改，那么在push的时候就会报错rejected推送失败，因为有冲突，所以这时候应该git pull拉取最新的提交并（自动地）在本地合并，产生冲突，然后本地解决冲突之后再add、commit，再push

### 3. 在本地新建一个分支再push到远端

1. `git checkout -b newBranch`
2. `git push origin newBranch`

### 4. 对于已经添加到的.gitignore文件，仍然会被trace到的解决方法

`git rm --cached dir_name/file_name`

原因是添加了 `.gitignore` 忽略这些路径后， 由于这个路径是已经增加到过仓库管理中，所以尽管已经在 ignore 列表里，依然 会被 git trace 到每个文件的变化。这时只需删除已缓存的trace就可以了
