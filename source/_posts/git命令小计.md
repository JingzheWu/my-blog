---
title: git命令小计
date: 2021-06-27 21:20:42
tags:
---
## git命令小计

- `git init`初始化一个本地git仓库repository
- `git status`查看状态
- `git add <file>`将工作区修改加到暂存区（stage）
- `git commit -m "<some message>"`将暂存区（stage）提交到仓库（repository）并写一些备忘信息
- `git reset --hard HEAD^`或者`git reset --hard HEAD~1`将工作区退回到仓库（repository）中上一个版本
- `git checkout -- <file>`将工作区中file文件的修改撤销，包括把删除的file复原（在添加到stage之前才有效，如果将工作区文件的修改add到了stage，那么再使用这个命令就会无效，这时应该先把stage的unstage撤销掉再使用上面的命令）
- `git reset HEAD <file>`将添加至stage的file文件unstage掉
- `git rm <file>`把file从repository中删除，之后再用commit就行
- `git remote add origin git@server-name:path/repo-name.git`关联一个远程库
- `git push -u origin master`第一次将本地的repository的master分支推送到远程，以后再push就不用加-u了
- `git clone  git@server-name:path/repo-name.git`从远程仓库clone一个仓库到本地，同时会关联到远程仓库
- `git clone -b <branchName>  git@server-name:path/repo-name.git`从远程clone指定分支到本地
- `git branch <name>`创建一个名为name的分支
- `git checkout <name>`切换到name分支上
- `git chekout -b <name>`上面两条命令合并之后的简写，即创建一个名为name的分支并切换到那个分支
- `git branch`查看本地所有分支，当前所在分支前面有一个*号
- `git branch -r`查看远程所有分支
- `git branch -a`查看本地和远程所有分支
- `git branch -d <name>`删除名为name的分支
- `git branch -D <name>`强行删除名为name的分支，用d删除的话，如果name分支没有被合并就无法删除，那么就要用D
- `git merge <name>`将指定name的分支合并到当前分支，其实就是把当前分支的内容更新为指定name分支的内容（前提是name分支的修改是上次commit之后最新的修改，或者说在上次commit之后name分支commit的次数比当前分支commit的次数多），如果name分支和当前分支的修改都是在上次commit之后最新的修改且内容不一样（或者说这两个分支自上次commit之后各自commit的次数是一样的），那么 就会产生冲突，需要解决冲突才能在当前分支上完成合并。解决冲突的方法就行修改当前分支内容，确定你想要当前分支的内容，然后再在当前分支上add和commit一次就可以了，这时，当前分支会比name分支多一次提交，也就是当前分支的修改比name分支的修改要新（如果此时你切换到name分支上，再merge当前分支，那么就会直接合并了，因为当前分支比name分支要新）
- `git merge <name>`时，如果name分支没有当前分支新，那么就会提示Already up to date，就是说当前分支比起name分支是新的。
- `git log`查看提交的日志信息
- `git log --pretty=oneline`查看只有一行的简略日志信息
- `git log --graph`查看日志信息的图示信息
- `git log --graph --pretty=oneline --abbrev-commit`
- `git merge --no-ff`表示在系统采取fast-forward时让系统采用普通合并方式，而不是采用快速合并，普通合并会多一次commit，这样在以后查看时能够看到合并历史，而fast-forward合并则看不到曾经做过的合并
- 分支管理策略：一般来说都是一个master分支，这个分支只用来发布新版本，不在上面干活，应该新建一个dev分支，平时都在dev上面干活，每个人一个分支，之后合并到dev上面就行，要发布时就把dev合并到master分支上即可。
- `git stash`把当前工作现场先存起来（不是add也不是commit），可以暂时去做其他事，之后（主要应用场景是当前手头工作没做完不能commit，但是又有紧急的bug需要切换到其他分支上去修复）
- `git stash list`查看所有存起来的工作现场
- `git stash apply <指定stash>`或者`git stash pop <指定stash>`，两者区别就是前者恢复工作现场后stash list当中并不删除指定stash，而后者就是直接将stash list中的pop出去删除了。
- `git remote`查看远程库信息，远程库默认名字是origin，`git remote -v`查看更详细的信息
- `git push origin <brancName>`将分支推送到远程的branch分支上
- 如果在git clone时不指定branch名称，那么clone下来的只会包含远程的master分支，在这之后如果要切换到远程的某一分支上，例如dev分支，那么就需要`git checkout -b dev origin/dev`，用这个命令创建本地的dev分支，然后就可以在这个dev上修改，并在之后随时push
- 如果多个人同时关联了远程库（多人协作时），在push的时候，发现有人已经向remote推送了他的提交，而你对同样的文件做了修改，那么在push的时候就会报错rejected推送失败，因为有冲突，所以这时候应该git pull拉取最新的提交并（自动地）在本地合并，产生冲突，然后本地解决冲突之后再add、commit，再push。
- `git pull`从远程拉取最新的修改到本地并合并，如果有冲突则需要在本地解决。git pull相当于git fetch加上merge
- 接着上面的说，如果本地解决冲突再push上去之后，其他的人是不会自动的获得更新消息的，就是说别人使用git status会提示up to date with origin/<branchName>，他们需要git fetch或者git pull之后才能获取到远程最新的变动
- `git fetch`从远程取回所有分支的更新（但不会自动合并）
- `git fetch origin <branchName>`从远程取回branchName分支的更新（但不会自动合并）。使用fetch之后就可以用git status查看到本地和远程之后的变动了。
- `git rebase` [变基](https://blog.csdn.net/wangnan9279/article/details/79287631)
- `git diff <filename>`不带参数就是比较工作区和stage中filename文件的不同
- `git diff --cached <filename>`比较stage和repository中filename文件的不同
- `git diff HEAD <filename>`比较工作区和repository中filename文件的不同
- 标签
- `git add .`只添加当前目录下的更改
- `git add -A`添加repository中所有目录下的修改
- 在本地新建一个分支再push到远端：`git checkout -b newbranch`，`git push origin newbranch`
- `git cherry-pick <commitid>`、`git cherry-pick <commitid1>..<commitid2>`这里的`<commitid>`一般来说不是当前分支上的，而是另一个分支上的`<commitid>`，用于将另一个分支上的提交commit（或者说复制到、摘抄到）到当前分支，详见：[git 场景——从一个分支cherry-pick一个或多个commit到另一个分支](https://blog.csdn.net/w958796636/article/details/78492017)
- 对于添加了.gitignore的文件，仍然会被trace到的解决方法：

    `git rm --cached dir_name/file_name`

    原因是添加了 .gitignore 忽略这些路径后， 由于这个路径是已经增加到过仓库管理中，所以尽管已经在 ignore 列表里，依然 会被 git trace 到每个文件的变化。这时只需删除已缓存的trace就可以了。