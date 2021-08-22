## Git 基础

配置提交代码的提交人信息

`git config --global username "username"`

`git config --global user.email "email@xx.com"`

创建本地库

`git init`

本地修改和提交

`git add xxx` 提交到缓冲区

`git commit -m "xxx"` 提交到本地仓库

`git status` 查看 git 状态 `git status -sb -uno --show-stash`

`git stash` 本地不想提高的修改，可以放入暂存区

`git checkout -f` 撤销本地修改，本地修改全部丢弃

`git rm --cache xxx` 撤销添加到缓存区的修改

`git reset uuid` 回滚

`git tag` 给提交贴标签

`git log` 查看提交记录

## Git 分支操作

`git branch xxx` 添加新分支  `-a` 可以查看所有分支 `-v` 分支提交情况

`git checkout xxx` 切换分支 加上 `-b` 参数创建并切换到该分支

`git merge xxx` 合并 xxx 分支

分支策略

`master` 主分支

`release` 预发布分支，预发布后必须合并进 develop 和 master 分支

`develop` 日常开发分支

`feature` 功能分支从 develop 分支分出

`hotfix` 修复 bug 分支

## Git 远程仓库

`git clone` 克隆代码库

`git remote` 查看远程代码库 `-v` 详细信息 

`git remote update origin` 更新远程仓库信息

`git merge origin/develop` 合并远程仓库

`git fetch origin xxx` 拉取远程仓库信息

`git pull` = `git fetch` + `git merge`

`git push` 推送远程仓库 = `git push origin dev:dev`