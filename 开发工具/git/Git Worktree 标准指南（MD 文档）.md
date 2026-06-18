# Git Worktree 标准指南（MD 文档）

## 1. 说明 git-worktree

`git worktree` 用于在一个 Git 仓库中管理**多个工作目录**，每个目录对应不同分支/提交，适合大型项目需要同时维护多个分支、避免频繁 `checkout` 的场景。

### 优点

- **并行开发**：同一项目多个分支可在不同目录同时演进，互不干扰。
    
- **提交共享**：所有 worktree 共享同一个版本库（对象库、提交记录等），提交/拉取/推送行为一致。
    
- **省空间、速度快**：相比多次 `git clone`，worktree 通过硬链接共享对象库，更省磁盘且创建更快。
    

---

## 2. 新增工作目录：`git worktree add`

### 2.1 最简写法（基于当前分支，分支名=目录名）

```Bash
git worktree add <新路径>
```

### 2.2 基于当前分支，新建指定分支

```Bash
git worktree add <新路径> -b <新分支名>
```

### 2.3 基于指定分支，新建指定分支

```Bash
git worktree add <新路径> -b <新分支名> <指定分支名>
```

---

## 2-1. 为已有分支创建工作目录（切换出指定分支）

```Bash
git worktree add ../<path_name> <branch>
```

### 2-1-1. 查看所有工作目录

```Bash
git worktree list
```

示例输出：

```Plain
/Users/project_local/build  11669902 [release-1.3.x]
```

### 2-1-2. 创建工作目录示例

```Bash
git worktree add ../build_master master
```

示例输出：

```Plain
Preparing worktree (checking out 'master')
HEAD is now at 5d6f19ef xxxx to branch master
```

### 2-1-3. 再次查看

```Bash
git worktree list
```

示例输出：

```Plain
/Users/project_local/build         11669902 [release-1.3.x]
/Users/project_local/build_master  5d6f19ef [master]
```

### 2-1-4. 进入 worktree 查看分支

```Bash
cd ../build_master && git branch
```

示例输出：

```Plain
* master
  release-1.3.x
```

### 2-1-5. 查看分支详细信息

```Bash
git branch -vv
```

示例输出：

```Plain
* master        5d6f19ef [origin/master] 5d6f19ef xxxx to branch master
  release-1.3.x 11669902 (/Users/project_local/build) [origin/release-1.3.x: behind 18] xxxx to branch release-1.3.x
```

---

## 2-2. `-b` / `-B` 的使用

```Bash
git worktree add -b/-B <new_branch> ../<path_name> <branch>
```

- **`-b`**：分支已存在则**报错**，不覆盖。
    
- **`-B`**：分支已存在则**强制重置**该分支用于此 worktree。
    

### 2-2-1 使用 `-b`（同名分支会报错）

#### 2-2-1-1 查看分支

```Bash
git branch
```

示例输出：

```Plain
  ccc
  master
* release-1.3.x
```

#### 2-2-1-2 新建已存在分支名（报错）

```Bash
git worktree add -b ccc ../build_ccc release-1.3.x
```

示例输出：

```Plain
Preparing worktree (new branch 'ccc')
fatal: A branch named 'ccc' already exists.
```

#### 2-2-1-3 新建不存在分支名（成功）

```Bash
git worktree add -b ddd ../build_ddd release-1.3.x
```

示例输出：

```Plain
Preparing worktree (new branch 'ddd')
HEAD is now at 11669902 xxxxx to branch release-1.3.x
```

#### 2-2-1-4 进入查看分支

```Bash
cd ../build_ddd && git branch
```

示例输出：

```Plain
  ccc
* ddd
  master
  release-1.3.x
```

### 2-2-2 使用 `-B`（同名分支会被强制重置）

#### 2-2-2-1 查看分支

```Bash
git branch
```

示例输出：

```Plain
  ccc
  master
* release-1.3.x
```

#### 2-2-2-2 新建已存在分支名（强制重置）

```Bash
git worktree add -B ccc ../build_ccc master
```

示例输出：

```Plain
Preparing worktree (resetting branch 'ccc'; was at 5d6f19ef)
HEAD is now at 5d6f19ef xxxxx to branch master
```

#### 2-2-2-3 查看分支

```Bash
git branch
```

示例输出：

```Plain
  ccc
  master
* release-1.3.x
```

---

## 3. 删除 worktree

### 3-1 `git worktree remove`

移除一个工作区。**只有干净的工作区**（无未跟踪文件、无修改的跟踪文件）可被删除。

```Bash
git worktree remove [-f] <path_name>
```

- `-f`：强制执行（脏工作区或含子模块时可用）
    
- 注意：**主工作区不能被删除**
    

#### 3-1-1 查看 worktree

```Bash
git worktree list
```

示例输出：

```Plain
/Users/project_local/build          11669902 [release-1.3.x]
/Users/project_local/build_ccc_new  5d6f19ef [ccc]
/Users/project_local/build_ddd      11669902 [ddd]
/Users/project_local/build_master   5d6f19ef [master]
```

#### 3-1-2 删除某个 worktree

```Bash
git worktree remove build_ddd
```

#### 3-1-3 再次查看

```Bash
git worktree list
```

示例输出：

```Plain
/Users/project_local/build          11669902 [release-1.3.x]
/Users/project_local/build_ccc_new  5d6f19ef [ccc]
/Users/project_local/build_master   5d6f19ef [master]
```

---

### 3-2 `git worktree prune`

修剪 `$GIT_DIR/worktrees/` 中指向已不存在目录的 worktree 记录（适用于**手动** **`rm -rf`** **删除目录**后清理元数据）。

```Bash
git worktree prune [-n] [-v] [--expire <time>]
```

- `-n`：演练模式，只报告要删除的内容，不实际删除
    
- `-v`：详细输出移除情况
    
- `--expire <time>`：只淘汰超过指定时间未使用的 worktree
    

#### 3-2-1 查看文件夹

```Bash
cd ../ && ls -al
```

示例输出：

```Plain
drwxr-xr-x   46 root  staff      1472 Jul 13 10:44 build
drwxr-xr-x   50 root  staff      1600 Jul 13 11:33 build_ccc_new
drwxr-xr-x   50 root  staff      1600 Jul 13 10:47 build_master
```

#### 3-2-2 手动删除文件夹

```Bash
rm -rf build_ccc_new build_master
```

#### 3-2-3 查看 worktree（元数据仍在）

```Bash
cd build && git worktree list
```

示例输出：

```Plain
/Users/project_local/build          11669902 [release-1.3.x]
/Users/project_local/build_ccc_new  5d6f19ef [ccc]
/Users/project_local/build_master   5d6f19ef [master]
```

#### 3-2-4 使用 prune

##### 3-2-4-1 不删除，只查看

```Bash
git worktree prune -n
```

示例输出：

```Plain
Removing worktrees/build_master: gitdir file points to non-existent location
Removing worktrees/build_ccc: gitdir file points to non-existent location
```

##### 3-2-4-2 删除并输出

```Bash
git worktree prune -v
```

示例输出同上。

##### 3-2-4-3 直接删除（无输出）

```Bash
git worktree prune
```

#### 3-2-5 再次查看

```Bash
git worktree list
```

示例输出：

```Plain
/Users/project_local/build  11669902 [release-1.3.x]
```

---

## 4. 移动 worktree：`git worktree move`

```Bash
git worktree move <path_name> <new_path_name>
```

### 4-1 案例

#### 4-1-1 查看

```Bash
git worktree list
```

示例输出：

```Plain
/Users/project_local/build         11669902 [release-1.3.x]
/Users/project_local/build_ccc     5d6f19ef [ccc]
/Users/project_local/build_ddd     11669902 [ddd]
/Users/project_local/build_master  5d6f19ef [master]
```

#### 4-1-2 移动

```Bash
git worktree move build_ccc ../build_ccc_new
```

#### 4-1-3 再次查看

```Bash
git worktree list
```

示例输出：

```Plain
/Users/project_local/build          11669902 [release-1.3.x]
/Users/project_local/build_ccc_new  5d6f19ef [ccc]
/Users/project_local/build_ddd      11669902 [ddd]
/Users/project_local/build_master   5d6f19ef [master]
```

---

## 5. 查看 worktree：`git worktree list`

```Bash
git worktree list [--porcelain]
```

- `--porcelain`：机器可读格式，更详细。
    

### 5-1 案例

#### 5-1-1 普通显示

```Bash
git worktree list
```

示例输出：

```Plain
/Users/project_local/build         11669902 [release-1.3.x]
/Users/project_local/build_ccc     5d6f19ef [ccc]
/Users/project_local/build_ddd     11669902 [ddd]
/Users/project_local/build_master  5d6f19ef [master]
```

#### 5-1-2 `--porcelain` 显示

```Bash
git worktree list --porcelain
```

示例输出：

```Plain
worktree /Users/project_local/build
HEAD 116699020d67237a27cafe67ef92128ff4d5d5e8
branch refs/heads/release-1.3.x

worktree /Users/project_local/build_ccc
HEAD 5d6f19efe800908192b9b94ac6362bd81db9daac
branch refs/heads/ccc

worktree /Users/project_local/build_ddd
HEAD 116699020d67237a27cafe67ef92128ff4d5d5e8
branch refs/heads/ddd

worktree /Users/project_local/build_master
HEAD 5d6f19efe800908192b9b94ac6362bd81db9daac
branch refs/heads/master
```

---

## 6. 加锁 / 解锁 worktree

### 6-1 `git worktree lock`

锁定一个 worktree，防止被移动或删除。

```Bash
git worktree lock [--reason <string>] <worktree>
```

- `--reason`：附加锁定原因说明。
    

#### 6-1-1 无提示加锁

```Bash
git worktree lock <path_name>
```

##### 6-1-1-1 给 build_ccc 加锁

```Bash
git worktree lock build_ccc
```

##### 6-1-1-2 尝试移除（报错）

```Bash
git worktree remove build_ccc
```

示例输出：

```Plain
fatal: cannot remove a locked working tree;
use 'remove -f -f' to override or unlock first
```

#### 6-1-2 带原因加锁

```Bash
git worktree lock --reason="test lock" build_ccc
```

##### 6-1-2-2 尝试移除（报错含原因）

```Bash
git worktree remove build_ccc
```

示例输出：

```Plain
fatal: cannot remove a locked working tree, lock reason: test lock
use 'remove -f -f' to override or unlock first
```

---

### 6-2 `git worktree unlock`

解锁 worktree，允许修剪、移动或删除。

```Bash
git worktree unlock <path_name>
```

示例：

```Bash
git worktree unlock build_ccc
```

---

如果你希望我再补充：常见坑（如分支被多个 worktree 占用、子模块、路径跨盘符/跨文件系统硬链接限制）、以及推荐的目录组织规范，我也可以继续扩展。