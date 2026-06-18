# Git Stash：用法与功能详解

你想了解 `git stash` 这个 Git 命令的用法和作用，对吧？简单来说，`git stash` 是 Git 中用来**临时保存工作区未提交修改**的核心命令，能帮你在不提交代码的情况下切换分支、拉取代码或处理其他任务。

### 一、`git stash` 核心概念

你可以把 `git stash` 理解成一个“临时剪贴板”：当你正在开发一个功能，代码还没写完，但需要紧急切换到其他分支（比如修复 bug），直接切换会因为工作区有未提交修改而失败，这时候 `git stash` 就能把当前工作区的修改“暂存”起来，让工作区回到干净状态；等你处理完其他任务后，再把暂存的修改恢复回来。

### 二、常用用法（附示例）

#### 1. 基础用法：暂存当前修改

```Bash
# 暂存所有未提交的修改（已跟踪文件）
git stash

# 推荐用法：给暂存加备注，方便后续识别（多人协作/多暂存时必备）
git stash save "修复订单模块bug：暂存未完成的支付逻辑"
```

- 执行后，你的工作区会回到上一次提交（`HEAD`）的干净状态，修改被保存到 Git 的 stash 栈中。
    
- 注意：`git stash` 默认只暂存**已跟踪文件**（已经被 `git add` 过的文件），新增的未跟踪文件需要加参数：
    
    ```Bash
    # 暂存所有修改（包括未跟踪文件、已跟踪文件）
    git stash -u  # 等价于 git stash --include-untracked
    # 暂存所有修改（包括忽略文件，如node_modules）
    git stash -a  # 等价于 git stash --all
    ```
    

#### 2. 查看暂存列表

```Bash
git stash list
```

输出示例：

```Plain
stash@{0}: On feature/pay: 修复订单模块bug：暂存未完成的支付逻辑
stash@{1}: On master: 临时修改配置文件
```

- `stash@{n}` 是暂存的唯一标识，`n` 从 0 开始（最新的暂存在最前面）。
    

#### 3. 恢复暂存的修改

有两种核心方式，根据需求选择： 在window下 使用 git stash pop "stash@{0}" 

```Bash
# 方式1：恢复最新的暂存（stash@{0}），且保留 stash 列表中的记录
git stash apply

# 方式2：恢复指定的暂存（比如stash@{1}）
git stash apply stash@{1}  

# 方式3：恢复最新的暂存，并从 stash 列表中删除该记录（推荐，避免堆积）
git stash pop

# 方式4：恢复指定暂存并删除
git stash pop stash@{1}
```

#### 4. 删除暂存

```Bash
# 删除最新的暂存
git stash drop

# 删除指定暂存
git stash drop stash@{1}

# 清空所有暂存（谨慎使用）
git stash clear
```

#### 5. 查看暂存的具体修改内容

```Bash
# 查看最新暂存的修改差异
git stash show

# 查看详细的修改内容（推荐）
git stash show -p

# 查看指定暂存的详细修改
git stash show -p stash@{1}
```

### 三、典型使用场景

假设你正在 `feature/user` 分支开发，突然需要修复 `master` 分支的紧急 bug：

```Bash
# 1. 暂存当前未完成的修改（加备注）
git stash save "user模块：暂存未完成的登录逻辑"

# 2. 切换到master分支
git checkout master

# 3. 修复bug，提交代码
# ...（修改代码）
git add .
git commit -m "fix: 修复master分支的支付bug"

# 4. 切回feature/user分支，恢复暂存的修改
git checkout feature/user
git stash pop

# 5. 继续开发user模块
```

### 总结 

1. `git stash` 核心作用是**临时保存未提交的修改**，让工作区回到干净状态，方便切换分支/处理紧急任务。
    
2. 常用组合：`git stash save "备注"`（暂存）→ `git stash list`（查看）→ `git stash pop`（恢复并删除）。
    
3. 关键参数：`-u` 暂存未跟踪文件，`-a` 暂存所有文件（包括忽略文件），`apply` 保留暂存记录，`pop` 恢复并删除记录。