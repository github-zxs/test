# Git 常用命令与工作流程详解

下面是整理后的标准 Markdown 文档，结构清晰、层级统一、便于阅读和归档。

---

Git 常用命令与工作流程整理

---

# 1. Git Branch（分支）

## 1.1 分支合并

### 1.1.1 Fast-forward（快进合并）

- 不产生新的提交点
    
- 直接将当前分支指针移动到被合并分支的最新 commit
    

### 1.1.2 git merge --no-ff（非快进合并）

- 总是产生新的提交点
    
- 也叫 3-way merge
    

### 1.1.3 合并时添加备注

仅适用于 `--no-ff` 方式（会生成新 commit）。

命令：

```Plain
git merge 分支名 --no-ff -m "本次合并的注释信息"
```

如果合并后提示版本落后，需要先 pull：

```Plain
git pull origin 分支名
```

进入 vi 编辑界面后：

- 按 `i` 或 `a` 进入编辑模式
    
- 修改最上方的备注信息（无 # 号的那一行）
    
- 按 `ESC` 退出编辑
    
- 输入 `:wq!` 保存并退出
    

---

## 1.2 Git Rebase（变基）

### 1.2.1 Git Merge 示例

```Plain
git checkout origin
git merge mywork
```

### 1.2.2 Git Rebase 示例

```Plain
git checkout mywork
git rebase origin
```

### 1.2.3 Rebase 注意事项

- 绝对不要在与他人共享的分支上执行 rebase
    

### 1.2.4 Git Rebase 最佳实践

- 只在本地未推送的分支上使用
    
- 用于线性化提交历史
    

---

## 1.3 切换分支

分支允许你在不影响主分支的情况下进行独立开发。

常用命令：

- 创建分支
    

```Plain
git branch <new_branch_name>
```

- 查看所有分支
    

```Plain
git branch
```

- 切换分支
    

```Plain
git checkout <branch_name>
```

- 创建并切换分支
    

```Plain
git checkout -b <branch_name>
```

- 使用 switch 切换分支（Git 2.23+）
    

```Plain
git switch <branch_name>
```

- 创建并切换分支（switch）
    

```Plain
git switch -c <branch_name>
```

- 切换到远程分支
    

```Plain
git fetch
git checkout -t <remote_name>/<branch_name>
```

---

# 2. Commit 命令

## 2.1 查看提交日志

```Plain
git log --pretty=format:"%h - %an, %ar : %s"
```

常用 format 参数：

- %H：完整哈希
    
- %h：简写哈希
    
- %T：树完整哈希
    
- %t：树简写哈希
    
- %P：父提交完整哈希
    
- %p：父提交简写哈希
    
- %an：作者名
    
- %ae：作者邮箱
    
- %ad：作者日期
    
- %ar：作者日期（相对时间）
    
- %cn：提交者名
    
- %ce：提交者邮箱
    
- %cd：提交日期
    
- %cr：提交日期（相对时间）
    
- %s：提交说明
    

---

## 2.2 Git 版本回退

### 2.2.1 回退到上一个版本

```Plain
git reset --hard HEAD^
git reset --hard HEAD~1
git reset --hard <commit_id>
```

### 2.2.2 回退到某一个版本

```Plain
git reset --hard HEAD^^
git reset --hard HEAD~n
git reset --hard <commit_id>
```

---

## 2.3 修改已提交的 commit 注释

### 2.3.1 已 push 到远程仓库

#### 修改最近一次注释

```Plain
git pull
git commit --amend
```

修改后按提示 `git push`。

#### 修改前几次注释

```Plain
git pull
git rebase -i HEAD~4
```

在编辑界面将需要修改的 commit 前的 `pick` 改为 `edit`。

然后：

```Plain
git commit --amend
git rebase --continue
git log
git push / git push --force origin <branch>
```

注意：强制 push 可能覆盖他人提交，请确保安全。

---

### 2.3.2 未 push，仍在本地

```Plain
git pull
git commit --amend
```

---

### 2.3.3 返回到最新状态

```Plain
git reflog
```

查看文件每一行的修改记录：

```Plain
git blame test.txt
```

---

# 3. Git 远程仓库管理

## 3.1 git fetch

### 3.1.1 Fetch + Fast-forward

```Plain
git fetch
git merge origin/master
```

### 3.1.2 Fetch + 3-way merge

---

## 3.2 git pull

```Plain
git pull <remote> <branch>
```

---

## 3.3 git push

常用方式：

1. 推送本地分支到远程同名分支
    

```Plain
git push origin master
```

2. 删除远程分支
    

```Plain
git push origin :refs/for/master
```

3. 推送当前分支
    

```Plain
git push origin
```

4. 简单推送（需设置上游）
    

```Plain
git push
```

扩展用法：

- 设置默认上游
    

```Plain
git push -u origin master
```

- 推送所有分支
    

```Plain
git push --all origin
```

- 强制推送
    

```Plain
git push --force origin
```

- 推送标签
    

```Plain
git push origin --tags
```

---

## 3.4 分支追踪

设置本地分支跟踪远程分支：

```Plain
git branch --set-upstream-to=origin/currentbranch currentbranch
```

推送并设置追踪：

```Plain
git push -u origin currentbranch
```

取消追踪：

```Plain
git branch --unset-upstream bugfix
```

---

## 3.5 Clone 远程仓库

### 3.5.1 克隆分支

#### 克隆指定分支

```Plain
git clone -b <branch_name> https://xxx.git
```

#### 克隆标签

```Plain
git clone -b <tag_name> <url>
```

#### 克隆所有分支

```Plain
git clone https://xxx.git
cd xxx
git branch -a
git checkout -b template origin/template
git checkout -b dev origin/dev
```

---

## 3.6 Git Checkout 远程标签

查看远程标签：

```Plain
git ls-remote --tags <url>
```

方法一：基于标签创建分支

```Plain
git checkout -b new_branch <tag_name>
```

方法二：使用 FETCH_HEAD

```Plain
git fetch origin tag v1.0
git checkout FETCH_HEAD
git switch -c newBranch
```

---

## 3.7 删除远程分支

方法 1：

```Plain
git push origin --delete <branch_name>
```

方法 2：

```Plain
git push origin :<branch_name>
```

方法 3（推荐）：

```Plain
git push --delete origin <branch_name>
```

---

## 3.8 git remote

查看远程信息：

```Plain
git remote show origin
```

同步删除远程已删除的分支：

```Plain
git remote prune origin
```

---

# 4. 项目中 Git 管理方式

## 4.1 远程仓库管理

---

# 5. Git 分支冲突分析

冲突原因：

- 两个分支在同一个文件的同一行有不同修改
    
- 合并时 Git 无法自动合并
    

---

# 6. git stash

应用场景：

- 临时保存未完成的修改
    
- 切换分支修复 bug
    
- 误在错误分支开发
    

stash 不保存未跟踪文件（Untracked files）。

保存包括未跟踪文件：

```Plain
git stash -u
```

查看 stash：

```Plain
git stash list
```

恢复 stash：

```Plain
git stash apply
```

删除 stash：

```Plain
git stash drop
```

恢复并删除：

```Plain
git stash pop
```

---

# 7. git worktree

优点：

- 同时在多个分支上并行开发
    
- 节省磁盘空间
    
- 比 clone 更快
    

---

# 8. Git Pull Rebase

默认：

```Plain
git pull = git fetch + git merge
```

使用 rebase 方式：

```Plain
git pull --rebase
```

设置默认使用 rebase：

```Plain
git config pull.rebase true
```

---

如需我再帮你：

- 生成 PDF
    
- 生成 HTML
    
- 生成可打印版本
    
- 做成 Git 学习笔记网站
    

都可以告诉我！