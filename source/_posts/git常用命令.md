---
title: git常用命令
category: git
---

# Git 命令

> Git分为工作区，版本库（暂存区和提交历史）
>
> 工作区即项目目录，版本库为其中的`.git`隐藏文件夹，其中index即为暂存区。

```bash
git add <filename>
git commit -m "message"
git commit -a -m "message"
```

```bash
git switch <branchname>
git swtich -c <branchname>
git checkout <branchname>
git checkout -b <branchname>
```

```bash
git branch
git branch -d <branchname>
git branch -D <branchname> #force delete
```

```bash
git stash
git stash pop
git stash supply <stashid>
```

```bash
git cherry-pick <commithash>
git cherry-pick <startcomit> <endcommit>  #(startcommit, endcommit]
git cherry-pick --continue                #deal with conflict
git cherry-pick --abort                   #abort cherry pick
```

```bash
git log --graph --pretty=oneline --abbrev-commit
```



Git在切换分支时，如果工作区或暂存区存在数据并没有提交，Git会尝试将这些数据（未追踪的数据，修改但没加入暂存区的数据，加入暂存区的数据）转移到新分支的工作区和暂存区，如果合并过程中出现冲突，那么Git会阻止分支切换。

解决冲突的方案：

* 在切换分支之前进行Commit操作，将所有工作区，暂存区的数据都提交，放入到版本库中
* 在切换分支之前使用Stash将工作区，暂存区的数据都保留镜像，等到切换回该分支后再执行pop将工作区，暂存区恢复。
* 直接放弃在工作区，暂存区的修改。

注意：一般来说，在切换分支前需要将工作区、暂存区清空，防止将一些不必要的修改带入到其他分支。

工作区中包含了**未被追踪的数据**和**加入暂存区后又被修改的数据**

暂存区中包含了**上一次add时数据的状态**