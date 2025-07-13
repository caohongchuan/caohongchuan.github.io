---
title: git常用命令
category: git
---

# Git 常用命令

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

## Switch

| 场景                | 命令                                   | 说明       |
| ------------------- | -------------------------------------- | ---------- |
| 切换已有本地分支    | `git switch dev`                       | 快速切换   |
| 创建并切换新分支    | `git switch -c dev`                    | 不关联远程 |
| 基于远程创建并跟踪  | `git switch --track -c dev origin/dev` | 推荐用法   |
| 基于 tag 创建新分支 | `git switch -c dev v1.0`               | 基于 tag   |
| 切换到 tag          | `git switch v1.0`                      | 游离 HEAD  |
| 切换到 commit       | `git switch a1b2c3d`                   | 游离 HEAD  |
| 强制切换丢弃修改    | `git switch -f dev`                    | 小心使用   |
| 游离 HEAD（只查看） | `git switch --detach origin/dev`       | 只看不提交 |

Git在切换分支时，如果工作区或暂存区存在数据并没有提交，Git会尝试将这些数据（未追踪的数据，修改但没加入暂存区的数据，加入暂存区的数据）转移到新分支的工作区和暂存区，如果合并过程中出现冲突，那么Git会阻止分支切换。

解决冲突的方案：

* 在切换分支之前进行Commit操作，将所有工作区，暂存区的数据都提交，放入到版本库中
* 在切换分支之前使用Stash将工作区，暂存区的数据都保留镜像，等到切换回该分支后再执行pop将工作区，暂存区恢复。
* 直接放弃在工作区，暂存区的修改。

注意：一般来说，在切换分支前需要将工作区、暂存区清空，防止将一些不必要的修改带入到其他分支。

工作区中包含了**未被追踪的数据**和**加入暂存区后又被修改的数据**

暂存区中包含了**上一次add时数据的状态**

## Restore

| 命令                                                       | 作用                                                         |
| ---------------------------------------------------------- | ------------------------------------------------------------ |
| `git restore <file>`                                       | 恢复指定文件到最新提交版本，覆盖工作区的改动（**未提交的更改会丢失**）。 |
| `git restore .`                                            | 恢复所有文件，丢弃全部工作区未提交的更改。                   |
| `git restore --staged <file>`                              | 取消暂存区的指定文件，保留工作区改动。                       |
| `git restore --staged .`                                   | 取消所有已暂存的文件，保留工作区改动。                       |
| `git restore --staged --worktree <file>`                   | 同时恢复暂存区和工作区，彻底还原文件到最近一次提交版本。     |
| `git restore --source=<branch> --staged --worktree <file>` | 从指定提交恢复文件，同时覆盖工作区和暂存区。                 |

| 需求                   | 命令                            |
| ---------------------- | ------------------------------- |
| 恢复工作区到仓库版本   | `git restore <file>`            |
| 恢复工作区到暂存区版本 | `git restore --source=: <file>` |
| 恢复暂存区到仓库版本   | `git restore --staged <file>`   |

## Stash

| 功能                        | 命令                                    |
| --------------------------- | --------------------------------------- |
| 保存修改                    | `git stash`                             |
| 保存修改（含未跟踪）        | `git stash -u`                          |
| 保存所有                    | `git stash -a`                          |
| 查看列表                    | `git stash list`                        |
| 查看修改                    | `git stash show -p stash@{0}`           |
| 应用最新 stash （但不删除） | `git stash apply`                       |
| 应用并删除                  | `git stash pop`                         |
| 删除指定 stash              | `git stash drop stash@{0}`              |
| 清空 stash                  | `git stash clear`                       |
| 交互式 stash                | `git stash push -p`                     |
| 新建分支恢复 stash          | `git stash branch new-branch stash@{0}` |

## Merge / Rebase

![image-20250629210400266](https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20250629210400266.png)

## Reset / Revert

| 操作                         | 修改历史 | 生成新提交 | 是否安全 | 用途                                                  |
| ---------------------------- | -------- | ---------- | -------- | ----------------------------------------------------- |
| `git reset --soft <commit>`  | ✅ 是     | ❌ 否       | ❌ 否     | 回退到目标 commit，将当前commit保留到暂存区           |
| `git reset --mixed <commit>` | ✅ 是     | ❌ 否       | ❌ 否     | 回退到目标 commit，将当前commit保留到工作区           |
| `git reset --hard <commit>`  | ✅ 是     | ❌ 否       | ❌ 否     | 回退到目标 commit，丢弃所有修改                       |
| `git revert`                 | ❌ 否     | ✅ 是       | ✅ 是     | 撤销当前 commit，生成一个新的节点，协作开发，不改历史 |

| 命令                      | HEAD 指针移动 | 暂存区变化             | 工作区变化                                       | 场景总结                                  |
| ------------------------- | ------------- | ---------------------- | ------------------------------------------------ | ----------------------------------------- |
| `git reset --soft HEAD^`  | ✅ 回退        | 保持回退前commit的内容 | ❌ 保留（工作区内容不变，保持回退前commit的内容） | 回退 commit，修改回到暂存区，适合继续提交 |
| `git reset --mixed HEAD^` | ✅ 回退        | 更改至目标commit的内容 | ❌ 保留（工作区内容不变，保持回退前commit的内容） | 回退 commit，修改回到工作区，适合继续编辑 |
| `git reset --hard HEAD^`  | ✅ 回退        | 更改至目标commit的内容 | ✅ 恢复为目标commit的状态                         | 回退 commit，直接丢弃修改，彻底干净       |

## Push

添加远程分支

| 功能         | 命令                                  |
| ------------ | ------------------------------------- |
| 添加远程     | `git remote add origin <url>`         |
| 查看远程     | `git remote -v`                       |
| 修改远程地址 | `git remote set-url origin <new-url>` |
| 删除远程     | `git remote remove origin`            |

push本地分支到远程分支并关联在一起

| 场景                   | 是否需要手动指定                     |
| ---------------------- | ------------------------------------ |
| 第一次推送主分支       | 需要执行 `git push -u origin main`   |
| 后续推送主分支         | 直接 `git push` 即可                 |
| 新建其他分支第一次推送 | 需要执行 `git push -u origin 分支名` |
| 新建其他分支后续推送   | 直接 `git push` 即可                 |

## Fetch / Pull

`git fetch` + `git merge` = `git pull`
