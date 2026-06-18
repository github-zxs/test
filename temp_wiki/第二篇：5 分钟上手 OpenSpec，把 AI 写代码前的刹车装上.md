AI追梦少年 *2026年6月4日 20:30*

## 第二篇：5 分钟上手 OpenSpec，把 AI 写代码前的刹车装上

上一篇讲完 OpenSpec 之后，有个朋友问我：这东西听起来挺对，但会不会又是那种“理念很好，用起来很烦”的工具？

我懂这个担心。

开发者对“流程工具”天然警惕。因为很多工具一上来就要你建模板、填字段、改团队习惯，最后功能还没开始写，仪式感先把人劝退了。

OpenSpec 的好处是，它的第一步很轻。

你不需要换 IDE，不需要放弃 Claude Code、Cursor 或 Codex，也不需要把项目重构成某种新架构。它做的事很克制：在你的仓库里加一个 `openspec/` 目录，让 AI 在动手前先读规则、写规格、确认边界。

这一篇我们不讲大道理。

直接跑一遍：安装 OpenSpec，初始化项目，创建第一个 Change，然后让 AI 按规格实现。

你会看到它到底多重，也会看到它为什么能管住 AI 那只“顺手多改一点”的手。

---

## 开始前，先准备一个小需求

别一上来就拿“重构整个权限系统”练手。

第一次用 OpenSpec，建议选一个小而完整的功能。最好满足三个条件：

- • 涉及 2-4 个文件
- • 有明确输入输出
- • 有清楚的“不做什么”

比如：

```
给用户资料页增加头像上传功能。
支持 JPG/PNG，限制 2MB。
上传成功后立即刷新头像。
不做图片裁剪，不改现有资料页布局。
```

这类需求很适合 OpenSpec。

它不至于小到“改一行文案”，也不至于大到第一次就迷路。更重要的是，它天然有边界：文件类型、大小限制、成功行为、非目标。

OpenSpec 最喜欢这种边界清楚的任务。

---

## 第 0 步：确认环境

先确认 Node 版本。

官方当前要求是 Node.js `20.19.0+` 。你可以先跑：

```
node -v
```

如果版本太低，用你常用的方式升级就行。 `nvm` 用户可以这样：

```
nvm install 20
nvm use 20
```

再确认项目是一个 Git 仓库：

```
git status
```

如果还没初始化 Git，先执行：

```
git init
```

这一步不是形式主义。

OpenSpec 的核心价值是让规格和代码一起演进。没有 Git，你当然也能用，但就少了 review、diff、回滚这些安全感。尤其是第一次让 AI 跑 Apply 阶段时，手里有干净的 Git 状态，心会稳很多。

我自己的习惯是：第一次接入 OpenSpec 前，先保证工作区干净。

```
git status --short
```

如果输出一堆未提交文件，先处理掉。别把 OpenSpec 初始化和已有改动混在一起，不然后面看 diff 会很难受。

---

## 第 1 步：安装 OpenSpec

官方推荐先安装 CLI：

```
npm install -g @fission-ai/openspec@latest
```

如果你用的是 pnpm、yarn、bun，也可以用对应的全局安装方式。这里不用纠结包管理器，核心是拿到 `openspec` 这个命令。

安装完成后，确认一下版本：

```
openspec --version
```

然后进入项目目录，执行初始化：

```
cd your-project
openspec init
```

如果你不想全局安装，官方也提供 Nix 的直接运行方式：

```
nix run github:Fission-AI/OpenSpec -- init
```

执行后，它会问你几个问题。不同版本的提示可能略有差异，但大意是这些：

- • 项目名称是什么？
- • 你用哪个 AI 工具？
- • 要不要生成 AI 助手规则？
- • 使用核心工作流还是扩展工作流？
- • 是否是存量项目？

第一次不用纠结，按默认选项走就行。

如果你已经在用 Claude Code，就选 Claude Code。用 Cursor，就选 Cursor。用 Codex，也没关系，重点是让它生成一套 AI 可读的项目规则。

OpenSpec 不是让你换工具，它是给现有工具加一层规格约束。

---

## init 之后发生了什么

初始化完成后，项目里会多一个 `openspec/` 目录。

典型结构长这样：

```
openspec/
├── specs/
│   └── project.spec.md
├── changes/
│   └── .gitkeep
├── AGENTS.md
├── project.md
└── config.yaml
```

这几个文件先不用全部读完，但你要知道各自干什么。

`AGENTS.md` 是给 AI 看的规矩。它会告诉 AI：这个项目使用 OpenSpec，不能随便跳过规格直接实现，做变更要先走 Propose。

`project.md` 是项目背景。技术栈、目录结构、约束条件、团队习惯，都可以放在这里。它解决的是“AI 每次新会话都不认识这个项目”的问题。

`specs/` 是当前系统能力的真相来源。不是未来计划，不是灵感清单，而是“这个系统现在正式具备什么能力”。

`changes/` 是活跃变更区。每次你要做一个新功能或大改动，就会在这里生成一个独立 Change。

`config.yaml` 是工作流配置。比如是否强制先 Propose、归档路径怎么组织、是否启用 Brownfield 规则。

如果你只记一件事，就记这个：

> `specs/` 写现在， `changes/` 写接下来。

这个分工一旦清楚，OpenSpec 的大部分设计就顺了。

![OpenSpec 初始化后生成的目录结构](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

OpenSpec 初始化后生成的目录结构

---

## 第 2 步：先让 AI 读项目，不要急着提需求

很多人装完工具后第一反应是立刻让 AI 写功能。

别急。

先让 AI 建立上下文。

如果你的工具支持 OpenSpec 的斜杠命令，可以先执行：

```
/opsx:explore
```

它会读取 `openspec/AGENTS.md` 、 `openspec/project.md` 、 `openspec/specs/` 和当前活跃变更，给出项目规格状态摘要。

这一步的价值很大。

AI 最容易出问题的地方，是它以为自己懂项目。 `explore` 的作用，就是让它先把“我现在知道什么、不知道什么”摊开。

如果你用的环境暂时没有斜杠命令，也可以用自然语言说：

```
请阅读 openspec/AGENTS.md、openspec/project.md 和 openspec/specs/，总结当前项目规格状态。先不要修改代码。
```

这句话里的“先不要修改代码”很重要。

第一次接入 OpenSpec 时，先建立只读上下文，再进入变更。这个节奏能避免 AI 上来就翻箱倒柜。

---

## 第 3 步：创建第一个 Change

现在开始做第一个小需求。

还是用头像上传这个例子：

```
/opsx:propose 给用户资料页增加头像上传功能。支持 JPG/PNG，限制 2MB。上传成功后立即刷新头像。不做图片裁剪，不改现有资料页布局。
```

如果不用斜杠命令，也可以这样说：

```
请按 OpenSpec 工作流，为“给用户资料页增加头像上传功能”创建一个 Proposed Change。
要求：支持 JPG/PNG，限制 2MB；上传成功后立即刷新头像；不做图片裁剪；不改现有资料页布局。先只生成 proposal、design 和 tasks，不要实现代码。
```

注意最后一句： **不要实现代码** 。

这就是 OpenSpec 的核心手感。你不是在请 AI “马上干活”，你是在请它“先把活定义清楚”。

一个正常的 Change 目录大概会长这样：

```
openspec/changes/CHNG-001-add-avatar-upload/
├── proposal.md
├── design.md
├── tasks.md
└── delta.spec.md
```

不同配置下文件名可能略有差异，但核心是三件套：proposal、design、tasks。

`proposal.md` 讲 Why 和 What。

它应该写清楚问题、目标、非目标、影响范围、成功标准。

`design.md` 讲 How。

比如上传接口怎么调用、前端状态怎么处理、错误提示放在哪里、是否复用现有组件。

`tasks.md` 讲执行顺序。

AI 后面 Apply 的时候，不是自由发挥，而是照这个清单一步步做。

![从一句需求生成 OpenSpec Change 包](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从一句需求生成 OpenSpec Change 包

---

## 先检查 proposal，别急着 Apply

这是很多人会跳过的一步。

AI 生成 proposal 后，不要立刻进入 Apply。先打开看一眼。

重点看四个地方。

第一，目标是不是你真正想要的。

比如“上传成功后立即刷新头像”有没有写进去。没有就补。

第二，非目标够不够硬。

如果你不想做图片裁剪，就必须写进非目标。不要指望 AI 自己克制。

第三，影响范围有没有离谱。

一个头像上传功能，如果 proposal 里出现“重构用户中心状态管理”“替换 UI 组件库”，你就要警觉了。

第四，成功标准能不能验证。

“体验良好”不算成功标准。“上传 2.5MB 图片时显示错误提示，且不发送请求”才算。

我建议你把第一版 proposal 当成 AI 写的草稿，而不是最终协议。

OpenSpec 的 Propose 阶段，本质上是一次低成本对齐。这里多改几分钟，比后面清代码便宜得多。

---

## 一个合格的 tasks.md 长什么样

很多 AI 工具会写任务清单，但大部分任务清单太虚。

比如：

```
- [ ] 实现头像上传
- [ ] 增加错误处理
- [ ] 测试功能
```

这不够。

OpenSpec 里的 tasks.md 最好写成这种颗粒度：

```
- [ ] Task 1: 添加头像上传 API 调用函数
  - 路径：src/api/profile.ts
  - 要求：只接受 JPG/PNG，上传前校验文件大小不超过 2MB

- [ ] Task 2: 在 ProfileAvatar 组件中增加文件选择入口
  - 路径：src/components/ProfileAvatar.tsx
  - 要求：复用现有按钮样式，不调整资料页布局

- [ ] Task 3: 上传成功后刷新用户资料
  - 路径：src/pages/ProfilePage.tsx
  - 要求：成功后重新拉取 profile 数据，显示新头像

- [ ] Task 4: 增加错误提示和测试
  - 要求：覆盖文件类型错误、文件过大、接口失败三种场景
```

这里有两个细节。

第一，每个任务最好有路径。

路径能约束 AI 的手。它知道主要该碰哪些文件，就不容易跑去改无关目录。

第二，每个任务最好有可验证条件。

“增加错误处理”太虚。“文件超过 2MB 时显示错误提示，且不发送请求”就清楚很多。

这也是 OpenSpec 比普通 Prompt 更稳的地方：它把“我脑子里的验收标准”写成 AI 能执行的清单。

---

## 第 4 步：Apply，让 AI 按清单实现

检查 proposal、design、tasks 都没问题后，再进入 Apply。

```
/opsx:apply CHNG-001-add-avatar-upload
```

或者自然语言：

```
请按 openspec/changes/CHNG-001-add-avatar-upload/tasks.md 顺序实现。每完成一个任务，更新 tasks.md 勾选状态。不要实现 tasks.md 之外的功能。
```

这句话里最关键的是最后一句。

不要实现 tasks.md 之外的功能。

AI 的“主动性”很有用，但在 Apply 阶段，它应该被关进轨道。Propose 阶段可以讨论，Apply 阶段应该执行。

实现过程中，你要盯三件事。

第一，它有没有按任务顺序做。

如果一上来就大改架构，停下来。

第二，它有没有即时更新 tasks.md。

任务打勾不是仪式感，它是进度记录。长任务中断后，能靠它恢复。

第三，它有没有主动扩大范围。

比如头像上传做到一半，它顺手加了头像裁剪、压缩、拖拽上传。这些功能可能都不错，但不在本次 Change 里，就应该先回来。

OpenSpec 不反对改需求。

但改需求应该回到 Propose，而不是在 Apply 里偷偷长出来。

---

## 第 5 步：验证，不要相信“我完成了”

AI 说完成了，不代表真的完成。

我现在有个习惯：只要 AI 说“已完成”，我默认再问一句：

```
请根据 proposal.md 的成功标准逐条验证，列出通过/未通过，并说明证据。不要修改代码。
```

如果项目有测试，跑测试。

```
npm test
```

如果有类型检查，跑类型检查。

```
npm run typecheck
```

如果有 lint，跑 lint。

```
npm run lint
```

OpenSpec 不是替代测试。它是让测试更有目标。

以前你让 AI “测试一下”，它可能随便跑个命令就说完成。现在你有 proposal 里的成功标准，有 tasks.md 里的完成条件，它就必须逐条对照。

这时候你会发现一个很现实的好处：规格写得越清楚，验收越不扯皮。

---

## 第 6 步：Archive，把这次变更变成项目记忆

功能实现并验证通过后，最后一步是 Archive。

```
/opsx:archive CHNG-001-add-avatar-upload
```

Archive 会做两件事。

第一，把这个 Change 从活跃变更区移到归档区。

第二，把最终能力写回 `specs/` 。

这一步非常关键。

如果你只完成代码，不更新规格，下次 AI 进来还是不知道“头像上传已经是系统能力的一部分”。它可能重复实现，或者在别的需求里误判现有行为。

Archive 的意义，就是把一次会话里的成果沉淀成长期上下文。

这和普通 AI 编程最大的区别就在这里。

普通对话结束了，知识散在聊天记录里。OpenSpec 归档后，知识留在仓库里。

![Propose 到 Archive 的完整闭环](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Propose 到 Archive 的完整闭环

---

## 第一次使用最容易踩的几个坑

第一个坑：需求太大。

第一次不要做“重构支付系统”。OpenSpec 能管复杂变更，但你刚开始需要先建立手感。选一个小功能，完整走完 Propose、Apply、Archive，比半路卡在大需求里更有价值。

第二个坑：proposal 写得像口号。

“提升用户体验”“优化代码结构”“增强稳定性”都太虚。AI 看了会继续猜。尽量写成能验证的句子。

第三个坑：非目标没写。

非目标不是可选项。AI 最容易越界的地方，往往就是你没写“不做”的地方。

第四个坑：Apply 前不 review。

AI 生成的 proposal 和 tasks 是草稿。你不看，它就按草稿执行。很多事故不是 Apply 阶段发生的，而是 Propose 阶段埋下的。

第五个坑：做完不 Archive。

不归档，OpenSpec 就只是一套临时任务清单。归档之后，它才变成项目记忆。

---

## 用 Codex App 怎么配合 OpenSpec

如果你是在 Codex App 里写代码，思路也一样。

你可以直接这样开一个新任务：

```
请阅读 openspec/AGENTS.md、openspec/project.md 和 openspec/specs/，先总结当前规格状态，不要修改代码。
```

然后创建 Change：

```
请按 OpenSpec 工作流，为“给用户资料页增加头像上传功能”创建 Proposed Change。
要求：支持 JPG/PNG，限制 2MB；上传成功后立即刷新头像；不做图片裁剪；不改现有资料页布局。
只生成 proposal、design、tasks，不要实现代码。
```

确认后再说：

```
请按这个 Change 的 tasks.md 顺序实现。每完成一个任务就更新勾选状态。不要实现 tasks.md 之外的功能。
```

你会发现，OpenSpec 和 Codex App 的协作点很自然。

Codex 擅长读仓库、改文件、跑命令。OpenSpec 负责告诉它边界和顺序。一个负责执行，一个负责约束。

这比“请你谨慎一点”可靠。

因为“谨慎”是态度， `tasks.md` 是轨道。

---

## 这一篇你真正要带走什么

安装 OpenSpec 不难。

真正需要改变的，是你和 AI 开始任务的方式。

以前是：

```
帮我实现这个功能。
```

现在换成：

```
先帮我把这个变更写成规格。
```

这句话看起来只多了一步，但协作关系变了。

AI 不再是听到需求就冲出去写代码的实习生，而是先和你一起把任务边界画清楚，再按清单执行。

OpenSpec 的价值不在于目录多漂亮，也不在于命令多高级。

它的价值在于让你多一个中间层：

> 模糊想法 → 可执行规格 → 代码实现 → 项目记忆

这个中间层，就是 AI 编程从“靠感觉”走向“可控交付”的关键。

下一篇，我们把视野拉开一点：OpenSpec、Kiro、BMAD、Spec-Kit 到底怎么选？为什么 2026 年大家突然都在谈 SDD？

先别急着让 AI 写代码。

让它先把规格写明白。

**微信扫一扫赞赏作者**

OpenSpec 规范精解 · 目录

继续滑动看下一个

FutureCraft AI

向上滑动看下一个