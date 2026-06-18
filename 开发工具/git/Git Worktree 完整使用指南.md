# Git Worktree 完整使用指南

你希望把之前关于 `git worktree` 的所有讲解内容整理成一份完整的 Markdown 文件，方便你查阅和使用，对吧？我已经按照清晰的结构把所有知识点整合好了，你可以直接复制下面的内容保存为 `.md` 文件（比如 `git-worktree-complete-guide.md`），内容涵盖概念、用法、切换、合并、清理等全流程，结构清晰且易于阅读。

---

# Git Worktree 完全使用指南

`git worktree` 是 Git 2.5+ 版本引入的实用工具，允许一个本地仓库对应多个工作目录，每个目录可独立检出不同分支，避免频繁切换分支、stash 代码的麻烦。

## 一、核心概念与解决的问题

日常开发中，若在分支 A 开发时需切换到分支 B 修复 Bug，传统方式需 `git stash` 暂存未完成代码，切换分支修复后再切回 `git stash pop`，过程繁琐且易出错。

`git worktree` 解决的核心问题：**一个 Git 仓库（****`.git`** **目录）对应多个工作目录**，每个工作目录独立检出不同分支，彼此互不干扰。

## 二、前置条件

确保 Git 版本 ≥ 2.5.0，可通过以下命令检查：

```Bash
git --version
```

## 三、基本用法

### 1. 创建新的工作目录（核心命令）

```Bash
# 基本格式：git worktree add <新工作目录路径> <分支名>
# 示例1：在当前仓库同级目录创建 dev 目录，检出 dev 分支
git worktree add ../my-project-dev dev

# 示例2：创建目录并检出远程分支（本地无该分支时）
git worktree add ../my-project-feat feat/new-function

# 示例3：创建临时工作目录（检出提交/标签，分支为只读）
git worktree add ../my-project-temp 1234567  # 1234567 是提交哈希
```

- 新目录的 `.git` 是指向主仓库 `.git` 的文本文件，共享同一个仓库；
    
- 新目录可独立执行 `git pull`/`git commit` 等操作，修改同步到主仓库。
    

### 2. 查看所有工作目录

```Bash
git worktree list
```

输出示例：

```Plain
/Users/you/projects/my-project  1234567 [main]
/Users/you/projects/my-project-dev  89abcde [dev]
```

### 3. 删除工作目录

#### 方式1：手动删除 + 清理记录（通用）

```Bash
# 第一步：删除物理目录
rm -rf ../my-project-dev

# 第二步：清理 Git 的 worktree 记录（关键）
git worktree prune
```

#### 方式2：一键删除（Git 2.17+ 推荐）

```Bash
# 安全删除（会检查是否有未提交修改，避免误删）
git worktree remove ../my-project-dev
```

### 4. 移动/重命名工作目录

```Bash
# 1. 移动物理目录
mv ../my-project-dev ../my-project-dev-new

# 2. 更新 Git 记录
git worktree move ../my-project-dev-new ../my-project-dev-new
```

## 四、工作树的切换

`git worktree` 无需像 `git checkout` 那样“切换分支”，**切换工作树 = 切换目录**，核心操作如下：

### 1. 终端中切换

直接用 `cd` 命令进入对应工作树目录：

```Bash
# 从主仓库切换到 dev 分支工作树
cd /Users/you/projects/my-project-dev

# 查看当前分支（确认切换成功）
git branch  # 输出：* dev

# 切换到 Bug 修复分支工作树
cd ../my-project-fix
```

### 2. 编辑器/IDE 中切换

- **VS Code**：在不同窗口打开各工作树目录，每个窗口对应一个分支；
    
- **IntelliJ IDEA/Android Studio**：为每个工作树单独创建项目窗口，或通过 `File -> Open -> New Window` 打开。
    

### 3. 注意事项

禁止在一个工作树中执行 `git checkout` 切换到其他分支（会破坏多工作树结构），如需操作其他分支，直接进入对应工作树目录即可。

### 4. 快速切换技巧（Linux/Mac）

编辑 `~/.bashrc` 或 `~/.zshrc` 设置别名：

```Bash
# 定义别名，快速进入对应工作树
alias goto-dev='cd /Users/you/projects/my-project-dev'
alias goto-fix='cd /Users/you/projects/my-project-fix'
alias goto-main='cd /Users/you/projects/my-project'
```

生效后，终端输入 `goto-dev` 即可直接进入 dev 分支工作树。

## 五、工作树之间的代码合并

多个工作树共享同一个 Git 仓库，合并逻辑与普通 Git 分支合并一致，核心是**在目标分支对应的工作树目录下执行合并命令**。

### 实操示例

假设：

- 目标分支：`main`（工作树路径：`~/projects/my-project`）；
    
- 源分支：`dev`（工作树路径：`~/projects/my-project-dev`）。
    

需求：将 `dev` 分支代码合并到 `main` 分支。

#### 步骤1：确保源分支代码已提交

```Bash
# 进入 dev 分支工作树
cd ~/projects/my-project-dev

# 查看代码状态（无未提交修改）
git status

# 如有修改，提交代码
git add .
git commit -m "完成dev分支功能开发"

# 可选：推送到远程
git push origin dev
```

#### 步骤2：进入目标分支工作树执行合并

```Bash
# 进入 main 分支工作树
cd ~/projects/my-project

# 拉取远程最新代码（避免冲突）
git pull origin main

# 合并 dev 分支到 main 分支
git merge dev
```

#### 步骤3：处理合并冲突（如有）

1. 打开冲突文件，修改 `<<<<<<< HEAD`（main 分支）、`=======`（dev 分支）、`>>>>>>> dev` 标记的冲突区域；
    
2. 提交解决后的代码：
    
    ```Bash
    git add .
    git commit -m "合并dev分支，解决xxx冲突"
    ```
    

### 进阶：用 rebase 替代 merge（整洁提交历史）

```Bash
# 进入 dev 分支工作树，将 dev 变基到 main 最新代码
cd ~/projects/my-project-dev
git rebase main
```

## 六、合并后工作树的清理

### 核心结论

`git merge` 仅合并代码，**不会自动删除工作树**，工作树会持久保留，需手动清理。

### 清理步骤

```Bash
# 方式1：一键删除（推荐）
git worktree remove ~/projects/my-project-dev

# 方式2：手动删除 + 清理记录
rm -rf ~/projects/my-project-dev  # 删除物理目录
git worktree prune                # 清理 Git 记录
```

### 验证清理结果

```Bash
git worktree list  # 输出中无已删除的工作树路径即清理成功
```

## 七、使用场景

1. **多分支并行开发**：主目录在 `main`，其他目录分别在 `feat1`/`feat2` 分支，各自开发互不干扰；
    
2. **紧急 Bug 修复**：无需 stash 未完成代码，创建临时工作树修复，完成后删除；
    
3. **分支代码对比**：多个工作树目录并排，直观对比文件差异。
    

## 八、核心总结

1. `git worktree` 核心价值：一个仓库对应多个工作目录，避免频繁切换分支、stash 代码；
    
2. 核心命令：`add`（创建）、`list`（查看）、`remove/prune`（删除/清理）；
    
3. 工作树切换：通过 `cd` 命令切换目录，禁止在工作树中 `git checkout` 其他分支；
    
4. 代码合并：在目标分支工作树执行 `git merge 源分支`，逻辑与普通合并一致；
    
5. 工作树清理：合并后不会自动删除，需手动执行 `remove` 或 `rm + prune`。
    

---

### 使用说明

1. 复制上述全部内容，保存为 `git-worktree-complete-guide.md`；
    
2. 可根据自己的实际路径替换示例中的目录（如 `/Users/you/projects/my-project`）；
    
3. 所有命令均可直接复制执行（确保路径正确），适合日常查阅和新手学习。