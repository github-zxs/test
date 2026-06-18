# Git 合并与变基完整实操指南（含关键缺失步骤）

# Git 合并Commit与变基（rebase）完整实操指南

本文基于你提供的实操流程，补充**关键缺失步骤**、**核心注意事项**和**操作逻辑解析**，让Git合并Commit、变基的实操更完整、易理解，所有命令与流程均与你的测试环境一致。

## 一、合并Commit（交互式变基 git rebase -i）

### 1-4 使用squash合并（补充缺失的1-4-2挑选commit步骤）

#### 1-4-2 挑选commit（vim编辑界面修改）

执行 `git rebase -i f73f8c0` 后进入vim编辑界面，原始内容为：

```Plain
pick 66f3e76 add d.txt
pick 00bfccd add e.txt
pick 3ce2c47 add f.txt
pick 12fd5e8 add g.txt
# 下方为注释说明，无需修改
```

**修改规则**：仅保留第一个commit为`pick`，后续需要合并的commit全部改为`squash`（或缩写`s`），修改后：

```Plain
pick 66f3e76 add d.txt
s 00bfccd add e.txt
s 3ce2c47 add f.txt
s 12fd5e8 add g.txt
```

**保存退出**：按 `Esc` 键，输入 `:wq` 回车，进入合并信息编辑界面。

#### 1-4-3 编辑合并信息（vim界面）

进入合并信息编辑界面后，原始内容会列出所有被合并commit的信息：

```Plain
# This is a combination of 4 commits.
# This is the 1st commit message:
add d.txt
# This is the commit message #2:
add e.txt
# This is the commit message #3:
add f.txt
# This is the commit message #4:
add g.txt
# 下方注释可删除，保留最终提交信息即可
```

**修改为最终合并信息**：删除多余注释和原始信息，只保留一行核心描述：

```Plain
add d e f g
```

**保存退出**：按 `Esc` 键，输入 `:wq` 回车，完成squash合并。

### 1-5 使用fixup合并（补充1-5-2完整挑选步骤+关键说明）

#### 1-5-2 挑选commit（vim编辑界面修改）

执行 `git rebase -i f73f8c0` 后，原始内容与1-4-2一致，**修改规则**：第一个commit保留`pick`，后续全部改为`fixup`（或缩写`f`）：

```Plain
pick 66f3e76 add d.txt
f 00bfccd add e.txt
f 3ce2c47 add f.txt
f 12fd5e8 add g.txt
```

**保存退出**：`Esc + :wq` 回车。

#### 关键说明

- fixup**不会进入提交信息编辑界面**，直接使用**第一个pick的commit信息**作为合并后的信息（因此修改原add d.txt为add d,e,f,g不会生效，因为fixup会忽略后续所有commit的信息）；
    
- 适合“快速合并、无需保留次要commit注释”的场景，比squash少一步信息编辑。
    

## 二、变基（git rebase）核心补充与注意事项

### 2-1 变基核心逻辑（强化理解）

变基的本质是**“重放提交”**：将当前分支的所有提交，按顺序“移植”到目标分支的最新提交之上，让当前分支的提交历史**基于目标分支最新状态重新构建**。

- 无冲突时：直接完成移植，分支历史变为“线性”，可实现**快进合并（fast-forward）**，无多余合并commit；
    
- 有冲突时：Git会暂停变基，需手动解决冲突后继续，或选择跳过/撤销。
    

### 2-2 无冲突变基（补充2-2-4完整日志解析）

执行变基后日志：

```Plain
* 15d529e (HEAD -> dev) add dev_2.txt
* 5306bba add dev_1.txt
* fbba993 (master) add b.txt
* a612327 add a.txt
```

**解析**：dev分支的2个提交（add dev_1.txt、add dev_2.txt）被成功移植到master最新提交（add b.txt）之上，master和dev分支历史完全线性，此时执行 `git checkout master && git merge dev` 会直接快进，无新commit生成。

### 2-3 有冲突变基（补充冲突文件修改+继续变基的细节）

#### 2-3-4 修改冲突文件的正确姿势

冲突文件a.txt原始内容：

```Plain
a
<<<<<<< HEAD
master
=======
m dev
>>>>>>> 9357d18 (dev update a.txt)
```

- `<<<<<<< HEAD` 到 `=======` 之间：**目标分支（master）** 的内容；
    
- `=======` 到 `>>>>>>> 9357d18` 之间：**当前分支（dev）** 的内容；
    
- **修改规则**：删除冲突标记（<<<<<<<、=======、>>>>>>>），保留需要的内容（可合并双方内容）。
    

修改后正确内容（合并master和dev的修改）：

```Plain
a
master
m dev
```

#### 2-3-5 继续变基的关键：标记冲突已解决

必须先执行 `git add a.txt` 将解决后的冲突文件加入暂存区，告诉Git“冲突已处理”，再执行 `git rebase --continue` 继续变基流程，否则会提示“no changes added to commit”。

### 2-4 跳过冲突（git rebase --skip）核心风险提醒

- 执行 `git rebase --skip` 会**直接丢弃当前冲突提交的所有修改**（包括非冲突文件的修改），仅保留目标分支（master）的内容；
    
- 适用场景：确认当前提交的修改无意义，或可后续重新提交，**非紧急情况禁止使用**，避免代码丢失。
    

### 2-5 撤销变基（补充更通用的撤销方法）

#### 方法1：基于reflog重置（你提供的方法，精准撤销）

- `git reflog` 查看Git操作日志，找到**rebase (start)** 之前的commit ID（如9357d18）；
    
- 执行 `git reset --hard <commit ID>`，直接恢复到变基前的分支状态，**注意：--hard会丢弃工作区未提交修改，执行前确认无未保存代码**。
    

#### 方法2：变基过程中直接撤销（更便捷）

如果变基还未完成（处于冲突暂停状态，未执行--continue/--skip），直接执行 `git rebase --abort`，一键恢复到变基前的分支状态，无需查找commit ID，**推荐优先使用**。

## 三、核心高频操作速查表

### 1. 合并Commit（交互式变基）

|   |   |   |
|---|---|---|
|场景|命令|关键步骤|
|合并当前分支后续提交|git rebase -i <起始commit>|第一个pick，后续s/f，保存退出|
|区间合并Commit|git rebase -i <起始> <结束>|合并后生成游离分支，需创建新分支保留|
|保留所有注释合并|分支内修改为squash（s）|需编辑最终合并提交信息|
|忽略次要注释合并|分支内修改为fixup（f）|无需编辑信息，直接使用第一个pick的注释|

### 2. 变基操作

|   |   |   |
|---|---|---|
|场景|命令|冲突处理|
|分支变基到目标分支|git checkout 待变基分支 && git rebase 目标分支|无冲突自动完成，有冲突暂停|
|解决冲突后继续变基|git add <冲突文件> && git rebase --continue|先标记解决，再继续|
|跳过当前冲突提交|git rebase --skip|丢弃当前提交修改，谨慎使用|
|一键撤销变基（未完成）|git rebase --abort|恢复到变基前状态，推荐优先使用|
|恢复已完成的变基|git reflog && git reset --hard <历史ID>|基于操作日志回滚，精准恢复|

## 四、重要禁忌（必看）

1. **禁止对已推送到远程公共分支的提交执行rebase（合并Commit/变基）**：rebase会改写提交历史，远程公共分支的提交被多人依赖，改写后会导致团队成员历史不一致，引发严重冲突；
    
2. **rebase过程中禁止执行其他Git操作**：如commit、merge、checkout，需先完成rebase（--continue/--skip/--abort）再执行其他操作；
    
3. **执行git reset --hard前务必确认**：该命令会永久丢弃工作区和暂存区未提交的修改，无恢复可能；
    
4. **区间合并后生成的游离分支需及时处理**：要么创建新分支保留（git switch -c 新分支），要么合并到原分支，否则切换分支后游离分支的修改可能丢失。
    

以上就是Git合并Commit和变基的完整实操补充，核心记住**“交互式rebase做历史整理，变基做分支同步，公共分支不改写历史”** 即可灵活运用。