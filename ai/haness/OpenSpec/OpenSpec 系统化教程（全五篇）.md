> 本篇由 @幽人 编写，OpenSpec 系统化教程全五篇合集
> 包含：入门篇 · 核心理念与规范化框架 · 命令与工作流详解 · 自定义与配置 · 迁移与多语言支持

---

# 一、OpenSpec 入门篇

幽人 *2026年5月8日 21:51*

## 1.1 什么是 OpenSpec

在软件开发过程中，需求变更、理解深化是常态。传统的工作流程往往采用瀑布模式——先规划、再实现、后验收——这种方式无法适应真实开发中的变化。OpenSpec 采用了一种更加灵活的方法： **流体化的工作流程** ，允许你在开发过程中随时迭代和调整需求文档。

OpenSpec 的核心价值在于：

- • **需求前置** ：在编写代码之前，明确系统应该做什么
- • **增量规范** ：采用 Delta Specs（增量规范）来描述变更，而不是重写整个规范
- • **AI 集成** ：通过技能系统（Skills）与各种 AI 编程工具深度集成
- • **版本追踪** ：完整的变更历史记录，支持审计和回溯

简单来说，OpenSpec 就是一个让你的 AI 编程助手"懂需求"的机制。它不仅仅是一套文档规范，更是一种人与 AI 高效协作的工作方式。

## 1.2 核心设计理念

OpenSpec 建立在四个核心理念之上：

### 1.2.1 流体化而非僵化

传统规范系统会将你锁定在阶段门（phase gates）中：先规划、再实现，最后才能结束。OpenSpec 更加灵活——你可以按照任何有意义的顺序创建工件。规范、代码、设计都可以在开发过程中随时调整。

### 1.2.2 迭代式而非瀑布式

需求会变化，理解会深化。开始时看似正确的方法，在看到代码后可能并不成立。OpenSpec 拥抱这一现实，允许你随时回溯和更新文档。

### 1.2.3 简单化而非复杂化

一些规范框架需要繁重的设置、僵化的格式或重量级的流程。OpenSpec 保持简洁——几秒初始化，立即开始工作，只有在需要时才进行自定义。

### 1.2.4 存量优先

大多数软件工作不是从零开始构建，而是修改现有系统。OpenSpec 的增量式方法使得描述现有行为的变更变得容易，而不仅仅是描述新系统。

## 1.3 安装与初始化

### 1.3.1 环境要求

在安装 OpenSpec 之前，请确保你的开发环境满足以下要求：

- • **Node.js 20.19.0 或更高版本** ：检查方法是在终端运行 `node --version`

### 1.3.2 安装方法

OpenSpec 支持多种包管理器进行安装：

**npm 安装：**

```coffeescript
npm install -g @fission-ai/openspec@latest
```

**pnpm 安装：**

```
pnpm add -g @fission-ai/openspec@latest
```

**yarn 安装：**

```
yarn global add @fission-ai/openspec@latest
```

**Bun 安装：**

```
bun add -g @fission-ai/openspec@latest
```

> 注意：Bun 可以全局安装 OpenSpec，但 OpenSpec 实际上运行在 Node.js 上。你仍然需要 Node.js 20.19.0 或更高版本在 PATH 中可用。

**Nix 运行：**

如果你使用 Nix，也可以直接运行而不需要安装：

```applescript
nix run github:Fission-AI/OpenSpec -- init
```

或者安装到你的环境配置中：

```
nix profile install github:Fission-AI/OpenSpec
```

### 1.3.3 验证安装

安装完成后，运行以下命令验证是否成功：

```ada
openspec --version
```

如果输出版本号（如 `0.x.x` ），说明安装成功。

### 1.3.4 项目初始化

在你的项目目录中运行初始化命令：

```bash
cd your-project
openspec init
```

初始化命令会进行以下操作：

- • 创建 `openspec/` 目录结构
- • 生成 `openspec/specs/` 目录（当前规范的真相来源）
- • 生成 `openspec/changes/` 目录（待处理的变更）
- • 生成项目配置文件 `openspec/config.yaml` （可选）
- • 为你选择的 AI 工具生成技能文件

### 1.3.5 目录结构

初始化后，你的项目将拥有以下结构：

```nix
openspec/
├── specs/              # 规范目录（系统行为的真相来源）
│   └── <domain>/
│       └── spec.md    # 各领域的规范文档
├── changes/            # 变更目录（待处理的修改，一个文件夹对应一个变更）
│   └── <change-name>/
│       ├── proposal.md
│       ├── design.md
│       ├── tasks.md
│       └── specs/      # 增量规范（描述变更内容）
│           └── <domain>/
│               └── spec.md
└── config.yaml         # 项目配置文件（可选）
```

## 1.4 工作模式

OpenSpec 支持两种工作模式，通过配置文件选择：

### 1.4.1 核心模式（Core Profile）

新安装默认使用 `core` 模式，提供以下核心命令：

- • `/opsx:propose` ——快速创建一个变更并生成所有规划工件
- • `/opsx:explore` ——探索思考，调查问题，澄清需求
- • `/opsx:apply` ——执行任务清单中的工作
- • `/opsx:sync` ——将增量规范同步到主规范
- • `/opsx:archive` ——归档完成的变更

典型工作流程：

```bash
/opsx:propose ──► /opsx:apply ──► /opsx:sync ──► /opsx:archive
```

### 1.4.2 扩展模式（Expanded Workflow）

如果你需要更精细的控制，可以启用扩展工作流命令：

```
openspec config profile
openspec update
```

扩展模式额外提供以下命令：

- • `/opsx:new` ——创建一个新的变更脚手架
- • `/opsx:continue` ——根据依赖关系创建下一个工件
- • `/opsx:ff` ——快速推进，一次性创建所有规划工件
- • `/opsx:verify` ——验证实现是否符合规范
- • `/opsx:bulk-archive` ——批量归档多个变更
- • `/opsx:onboard` ——引导完成整个工作流程的教程

## 1.5 快速开始示例

让我们通过添加"深色模式"功能来演示完整的 OpenSpec 工作流程。

### 1.5.1 步骤一：创建变更

```
你：/opsx:propose add-dark-mode
```

AI 响应：

```
AI：已创建 openspec/changes/add-dark-mode/
     ✓ proposal.md — 变更原因和内容
     ✓ specs/ — 需求和场景
     ✓ design.md — 技术方案
     ✓ tasks.md — 实现检查清单
     准备开始实现！
```

### 1.5.2 步骤二：实现

```bash
你：/opsx:apply
```

AI 响应：

```erlang
AI：开始执行任务...
     ✓ 1.1 创建 ThemeContext（浅色/深色状态）
     ✓ 1.2 添加 CSS 自定义属性到 globals.css
     ✓ 1.3 实现 localStorage 持久化
     ✓ 2.1 创建 ThemeToggle 组件
     ...
     所有任务完成！
```

### 1.5.3 步骤三：归档

```bash
你：/opsx:archive
```

AI 响应：

```
AI：正在归档 add-dark-mode...
     ✓ 已将规范合并到 openspec/specs/ui/spec.md
     ✓ 已移动到 openspec/changes/archive/2025-01-24-add-dark-mode/
     完成！准备下一个功能。
```

## 1.6 AI 工具支持

OpenSpec 支持多种 AI 编程助手，通过技能系统集成。初始化时会询问你要配置哪些工具。

支持的工具包括：

- • Claude Code
- • Cursor
- • Windsurf
- • GitHub Copilot
- • Trae
- • Kimi CLI
- • 以及更多……

每种工具都有对应的技能目录，OpenSpec 会根据你的选择自动生成配置文件。

## 1.7 常用命令参考

### 1.7.1 CLI 命令

```
# 初始化项目
openspec init

# 查看当前配置
openspec config list

# 列出所有变更
openspec list

# 查看变更详情
openspec show <change-name>

# 验证变更
openspec validate <change-name>

# 归档变更
openspec archive <change-name>

# 更新技能文件
openspec update
```

### 1.7.2 Slash 命令（在 AI 工具中使用）

```bash
/opsx:propose <功能名称>    # 创建新变更
/opsx:explore               # 探索思考
/opsx:apply                 # 执行实现
/opsx:archive              # 归档完成
```

## 1.8 总结

本篇介绍了 OpenSpec 的基本概念、安装方法和快速开始流程。OpenSpec 的核心价值在于帮助人和 AI 在编写代码之前就需求达成一致，通过增量式的规范文档来追踪变更，并保持完整的审计历史。

关键要点：

1. **安装简单** ：一行命令即可全局安装
2. **初始化快速** ：几秒钟完成项目配置
3. **工作流灵活** ：支持核心模式和扩展模式
4. **AI 集成** ：广泛的 AI 工具支持
5. **版本追踪** ：完整的变更历史

---

# 二、OpenSpec 核心理念与规范化框架

幽人 *2026年5月9日 22:17*

> 本篇深入介绍 OpenSpec 的核心概念、规范体系和工作原理，帮助你全面理解这个规范化框架的设计思想。

## 2.1 核心设计哲学

OpenSpec 的设计哲学建立在四个支柱之上：

### 2.1.1 流体化而非僵化

传统规范系统将你锁定在阶段门中：先规划、然后实现、最后才算完成。这种模式无法适应真实开发中的变化。

OpenSpec 更加灵活——你可以按照任何有意义的顺序创建工件。规范、代码、设计都可以在开发过程中随时调整。

### 2.1.2 迭代式而非瀑布式

需求会变化，理解会深化。开始时看似正确的方法，在看到代码后可能并不成立。

OpenSpec 拥抱这一现实——你在实现过程中发现的任何问题，都可以回去更新之前的文档。这就是"流体化工作流"的真正含义：你可以随时编辑任何工件，然后继续。

### 2.1.3 简单化而非复杂化

某些规范框架需要繁重的设置、僵化的格式或重量级的流程。OpenSpec 保持简洁：

- • 初始化只需要几秒
- • 立即可以开始工作
- • 只有在需要时才进行自定义

### 2.1.4 存量优先（Brownfield-First）

大多数软件工作不是从零开始构建，而是修改现有系统。OpenSpec 的增量式方法使得描述对现有行为的变更变得容易，而不是仅仅描述新系统。

这就是 Delta Specs（增量规范）的核心价值——它描述的是"改变了什么"，而不是"整个系统是什么"。

## 2.2 两大核心概念：Specs 与 Changes

OpenSpec 将工作组织为两个核心区域：

```
┌────────────────────────────────────────────────────────────────────┐
│                        openspec/                                   │
│                                                                    │
│   ┌─────────────────────┐      ┌───────────────────────────────┐   │
│   │       specs/        │      │         changes/              │   │
│   │                     │      │                               │   │
│   │  真相来源         │◄─────│  建议修改                   │   │
│   │  描述系统        │ 合并  │  每个变更 = 一个文件夹      │   │
│   │  当前的行为      │      │  包含工件 + 增量规范        │   │
│   │                     │      │                               │   │
│   └─────────────────────┘      └───────────────────────────────┘   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```
- • **Specs（规范）** 是真相来源——描述系统当前如何工作
- • **Changes（变更）** 是待处理的修改——它们保存在单独的文件夹中，直到你准备好合并它们

这种分离是关键。你可以并行处理多个变更而不会冲突。你可以在合并变更之前审查变更。当你归档一个变更时，它的增量干净地合并到真相来源中。

## 2.3 规范（Specs）详解

### 2.3.1 规范的结构

规范按照领域（domain）组织——对你的系统有意义的逻辑分组：

```
openspec/specs/
├── auth/
│   └── spec.md           # 认证行为
├── payments/
│   └── spec.md           # 支付处理
├── notifications/
│   └── spec.md           # 通知系统
└── ui/
    └── spec.md           # UI 行为和主题
```

常见的组织模式包括：

- • **按功能区域** ：auth、payments、search
- • **按组件** ：api、frontend、workers
- • **按有界上下文** ：ordering、fulfillment、inventory

### 2.3.2 规范格式

一个规范包含需求（Requirements），每个需求都有场景（Scenarios）：

```
# Auth 规范

## 目的
应用程序的认证和会话管理。

## 需求

### 需求：用户认证
系统应在成功登录后发放 JWT 令牌。

#### 场景：有效凭证
- 给定：用户具有有效凭证
- 当：用户提交登录表单
- 然后：返回 JWT 令牌
- 并且：用户被重定向到仪表板

#### 场景：无效凭证
- 给定：无效凭证
- 当：用户提交登录表单
- 然后：显示错误消息
- 并且：不发放令牌

### 需求：会话过期
系统必须在 30 分钟不活动后使会话过期。

#### 场景：空闲超时
- 给定：已认证的会话
- 当：30 分钟没有活动
- 然后：会话无效
- 并且：用户必须重新认证
```

### 2.3.3 关键元素

| 元素 | 用途 |
| --- | --- |
| `## 目的` | 该规范领域的高级描述 |
| `### 需求：` | 系统必须具备的特定行为 |
| `#### 场景：` | 该需求的具体示例 |
| SHALL/MUST/SHOULD | RFC 2119 关键字表示需求强度 |

### 2.3.4 RFC 2119 关键字

- • **MUST/SHALL** ——绝对需求
- • **SHOULD** ——建议的，但存在例外
- • **MAY** ——可选的

### 2.3.5 规范是什么（不是什么）

规范是一份 **行为契约** ，而不是实现计划。

**规范中应该包含的内容：**

- • 用户或下游系统依赖的可观察行为
- • 输入、输出和错误条件
- • 外部约束（安全、隐私、可靠性、兼容性）
- • 可以测试或明确验证的场景

**规范中应该避免的内容：**

- • 内部类/函数名
- • 库或框架选择
- • 逐步实现细节（这些属于 design.md 或 tasks.md）

**快速测试：** 
如果实现可以在不改变外部可观察行为的情况下改变，那么它可能不属于规范。

### 2.3.6 渐进式严谨

OpenSpec 旨在避免官僚化。使用仍然可以验证变更的最轻级别。

**轻量级规范（默认）：**

- • 简短的行为优先需求
- • 明确的目标和非目标
- • 几个具体的验收检查

**完整规范（用于更高风险）：**

- • 跨团队或跨仓库变更
- • API/契约变更、迁移、安全/隐私问题
- • 容易导致昂贵返工的不确定性变更

大多数变更应该保持在轻量级模式。

## 2.4 变更（Changes）详解

### 2.4.1 什么是变更

变更是对待处理修改的封装，包含理解和实现它所需的一切。

### 2.4.2 变更结构

```
openspec/changes/add-dark-mode/
├── proposal.md           # 原因和内容
├── design.md             # 如何做（技术方案）
├── tasks.md             # 实现检查清单
├── .openspec.yaml       # 变更元数据（可选）
└── specs/             # 增量规范
    └── ui/
        └── spec.md     # ui/spec.md 中正在改变的内容
```

每个变更都是自包含的，包含：

- • **工件** ——捕获意图、设计和任务的文档
- • **增量规范** ——要添加、修改或删除的规范
- • **元数据** ——该特定变更的可选配置

### 2.4.3 为什么采用文件夹形式

将变更打包为文件夹有几个好处：

1. **一切都在一起** 。提案、设计、任务和规范放在一个地方。无需在不同位置搜寻。
2. **并行工作** 。多个变更可以同时存在而不会冲突。在 add-dark-mode 工作的同时，fix-auth-bug 也可以进行。
3. **干净的历史** 。归档时，变更及其完整上下文被移动到 changes/archive/。你可以回顾并理解不仅仅是改变了什么，还有为什么改变。
4. **便于审查** 。变更文件夹易于审查——打开它，阅读提案，检查设计，看看增量规范。

## 2.5 工件（Artifacts）详解

### 2.5.1 工件类型

工件是变更中指导工作的文档。

```
proposal ──────► specs ──────► design ──────► tasks ──────► implement
    │               │             │              │
   why            what           how          steps
 + scope        changes       approach      to take
```

工件相互构建。每个工件为下一个提供上下文。

### 2.5.2 Proposal（提案）

提案捕获 **意图** 、 **范围** 和 **方法** 。

```
# 提案：添加深色模式

## 意图
用户请求深色模式选项，以减少夜间使用时的眼睛疲劳，
并匹配系统偏好设置。

## 范围
范围内：
- 设置中的主题切换
- 系统偏好检测
- localStorage 中的持久化偏好

范围外：
- 自定义颜色主题（未来工作）
- 页面级别的主题覆盖

## 方法
使用 CSS 自定义属性进行主题设置，使用 React Context
进行状态管理。首次加载时检测系统偏好，
允许手动覆盖。
```

**何时更新提案：**

- • 范围改变（缩小或扩大）
- • 意图明确（更好地理解问题）
- • 方法基本转变

### 2.5.3 Specs（规范）

增量规范描述 **相对于当前规范改变了什么** 。详见 2.6 节。

### 2.5.4 Design（设计）

设计捕获 **技术方法** 和 **架构决策** 。

```
# 设计：添加深色模式

## 技术方法
主题状态通过 React Context 管理以避免属性传递。
CSS 自定义属性支持运行时切换，无需类切换。

## 架构决策

### 决策：Context over Redux
使用 React Context 管理主题状态，因为：
- 简单的二元状态（浅色/深色）
- 没有复杂的状态转换
- 避免添加 Redux 依赖

### 决策：CSS 自定义属性
使用 CSS 变量而不是 CSS-in-JS，因为：
- 与现有样式表配合工作
- 无运行时开销
- 浏览器原生解决方案

## 数据流
ThemeProvider (context)
       │
       ▼
ThemeToggle ◄──► localStorage
       │
       ▼
CSS 变量（应用于 :root）

## 文件变更
- src/contexts/ThemeContext.tsx (新建)
- src/components/ThemeToggle.tsx (新建)
- src/styles/globals.css (修改)
```

**何时更新设计：**

- • 实现表明方法行不通
- • 发现更好的解决方案
- • 依赖或约束改变

### 2.5.5 Tasks（任务）

任务是 **实现检查清单** ——带有复选框的具体步骤。

```
# 任务

## 1. 主题基础设施
- [ ] 1.1 创建具有浅色/深色状态的 ThemeContext
- [ ] 1.2 为颜色添加 CSS 自定义属性
- [ ] 1.3 实现 localStorage 持久化
- [ ] 1.4 添加系统偏好检测

## 2. UI 组件
- [ ] 2.1 创建 ThemeToggle 组件
- [ ] 2.2 将切换添加到设置页面
- [ ] 2.3 更新 Header 以包含快速切换

## 3. 样式
- [ ] 3.1 定义深色主题调色板
- [ ] 3.2 更新组件以使用 CSS 变量
- [ ] 3.3 测试对比度的可访问性
```

**任务最佳实践：**

- • 相关任务分组在标题下
- • 使用分层编号（1.1、1.2 等）
- • 保持任务足够小以在一个会话中完成
- • 完成后勾选任务

## 2.6 增量规范（Delta Specs）详解

### 2.6.1 为什么需要增量规范

增量规范是让 OpenSpec 适合存量开发的关键概念。它们描述的是 **相对于当前规范的改变** ，而不是重述整个规范。

### 2.6.2 格式

```
# Auth 增量规范

## 新增需求

### 需求：双因素认证
系统必须支持基于 TOTP 的双因素认证。

#### 场景：2FA 注册
- 给定：未启用 2FA 的用户
- 当：用户在设置中启用 2FA
- 然后：显示二维码用于身份验证器应用设置
- 并且：用户必须先验证代码才能激活

#### 场景：2FA 登录
- 给定：已启用 2FA 的用户
- 当：用户提交有效凭证
- 然后：显示 OTP 挑战
- 并且：仅在有效 OTP 后才能完成登录

## 修改需求

### 需求：会话过期
系统必须在 15 分钟不活动后会话过期。
（之前：30 分钟）

#### 场景：空闲超时
- 给定：已认证的会话
- 当：15 分钟没有活动
- 然后：会话无效

## 移除需求

### 需求：记住我
（已弃用，改用 2FA。用户应每个会话重新认证。）
```

### 2.6.3 增量部分

| 部分 | 含义 | 归档时发生什么 |
| --- | --- | --- |
| `## 新增需求` | 新行为 | 追加到主规范 |
| `## 修改需求` | 改变的行为 | 替换现有需求 |
| `## 移除需求` | 弃用的行为 | 从主规范中删除 |

### 2.6.4 为什么使用增量而非完整规范

**清晰度。** 增量显示 Exact 正在改变什么。阅读完整规范，你不得不在脑海中与当前版本对比。

**避免冲突。** 多个变更可以接触同一个规范文件而不会冲突，只要它们修改不同的需求。

**审查效率。** 审查者看到的是改变，而不是未改变的上下文。关注重要的内容。

**存量适配。** 大多数工作修改现有行为。增量使修改成为一等公民，而不是事后才想到的。

### 2.6.5 归档流程

归档完成一个变更，将其增量规范合并到主规范中，并为历史保留该变更。

```
归档前：

openspec/
├── specs/
│   └── auth/
│       └── spec.md ◄────────────────┐
└── changes/                         │
    └── add-2fa/                     │
        ├── proposal.md              │
        ├── design.md                │ 合并
        ├── tasks.md                 │
        └── specs/                   │
            └── auth/                │
                └── spec.md ─────────┘

归档后：

openspec/
├── specs/
│   └── auth/
│       └── spec.md        # 现在包含 2FA 需求
└── changes/
    └── archive/
        └── 2025-01-24-add-2fa/    # 为历史保留
            ├── proposal.md
            ├── design.md
            ├── tasks.md
            └── specs/
                └── auth/
                    └── spec.md
```

**归档过程：**

1. **合并增量** 。每个增量规范部分（新增/修改/移除）被应用到相应的主规范。
2. **移动到归档** 。变更文件夹移动到 changes/archive/，并带有日期前缀以按时间顺序排列。
3. **保留上下文** 。所有工件在归档中保持完整。你总是可以回顾并理解为什么进行了更改。

## 2.7 模式（Schemas）详解

### 2.7.1 什么是模式

模式定义了工作流中的工件类型及其依赖关系。

### 2.7.2 内置模式

**spec-driven（默认）**

标准的需求驱动开发工作流：

```
proposal → specs → design → tasks → implement
```

最适合：大多数功能工作，你想在实现之前就需求达成一致。

### 2.7.3 自定义模式

你可以为团队的工作流创建自定义模式：

```
# openspec/schemas/research-first/schema.yaml
name: research-first
artifacts:
  - id: research
    generates: research.md
    requires: []           # 先做研究

  - id: proposal
    generates: proposal.md
    requires: [research]   # 提案由研究结果决定

  - id: tasks
    generates: tasks.md
    requires: [proposal]   # 跳过 specs/design，直接到任务
```

## 2.8 工作空间（Workspaces）

### 2.8.1 什么是工作空间

工作空间是跨链接仓库或文件夹进行规划的家庭。当一个工作跨越多个仓库时使用。

### 2.8.2 工作空间结构

```
workspace-folder/
├── changes/                       # 工作空间级规划
└── .openspec-workspace/
    ├── workspace.yaml             # 共享的工作空间身份和链接名称
    └── local.yaml                  # 本机的本地路径
```

### 2.8.3 链接名称

链接名称是工作空间规划如何指代仓库和文件夹的稳定名称。共享工作空间状态保持如 api、web 或 checkout 这样的名称；每个机器将其映射到自己的本地路径。

```
# .openspec-workspace/workspace.yaml
version: 1
name: platform
links:
  api: {}
  web: {}
```
```
# .openspec-workspace/local.yaml
version: 1
paths:
  api: /repos/api
  web: /repos/web
```

## 2.9 关键概念图解

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              OPENSPEC FLOW                                  │
│                                                                              │
│   ┌────────────────┐                                                         │
│   │  1. 开始       │  /opsx:propose (核心) 或 /opsx:new (扩展)               │
│   │     变更       │                                                         │
│   └───────┬────────┘                                                         │
│           │                                                                  │
│           ▼                                                                  │
│   ┌────────────────┐                                                         │
│   │  2. 创建       │  /opsx:ff 或 /opsx:continue (扩展工作流)                │
│   │     工件       │  创建 proposal → specs → design → tasks                 │
│   │                │  （基于模式依赖关系）                                    │
│   └───────┬────────┘                                                         │
│           │                                                                  │
│           ▼                                                                  │
│   ┌────────────────┐                                                         │
│   │  3. 实现       │  /opsx:apply                                            │
│   │     任务       │  执行任务，勾选完成                                     │
│   │                │◄──── 更新工件，因为你有了新的发现                        │
│   └───────┬────────┘                                                         │
│           │                                                                  │
│           ▼                                                                  │
│   ┌────────────────┐                                                         │
│   │  4. 验证       │  /opsx:verify (可选)                                    │
│   │     工作       │  检查实现是否符合规范                                    │
│   └───────┬────────┘                                                         │
│           │                                                                  │
│           ▼                                                                  │
│   ┌────────────────┐     ┌──────────────────────────────────────────────┐    │
│   │  5. 归档       │────►│  增量规范合并到主规范                        │    │
│   │     变更       │     │  变更文件夹移动到 archive/                   │    │
│   └────────────────┘     │  规范现在是更新后的真相来源                  │    │
│                          └──────────────────────────────────────────────┘    │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

**良性循环：**

1. 规范描述当前行为
2. 变更提出修改（作为增量）
3. 实现使变更成为现实
4. 归档将增量合并到规范中
5. 规范现在描述新的行为
6. 下一个变更建立在更新后的规范之上

## 2.10 术语表

| 术语 | 定义 |
| --- | --- |
| **工件** | 变更中的文档（proposal、design、tasks 或增量规范） |
| **归档** | 完成变更并将其增量合并到主规范的过程 |
| **变更** | 系统中待处理的修改，打包为包含工件的文件夹 |
| **增量规范** | 描述相对于当前规范改变什么的规范（新增/修改/移除） |
| **领域** | 规范的逻辑分组（如 auth、payments） |
| **需求** | 系统必须具备的特定行为 |
| **场景** | 需求的具体示例，通常采用 Given/When/Then 格式 |
| **模式** | 工件类型及其依赖关系的定义 |
| **规范** | 描述系统行为的规范，包含需求和场景 |
| **真相来源** | openspec/specs/ 目录，包含当前约定的行为 |

## 2.11 总结

本篇深入介绍了 OpenSpec 的核心概念和规范化框架。

关键要点：

1. **两大核心目录** ：specs（真相来源）和 changes（待处理变更）
2. **四种工件类型** ：proposal、specs、design、tasks
3. **增量规范** ：新增/修改/移除三种类型，支持存量开发
4. **归档机制** ：增量合并 + 历史保留
5. **模式系统** ：可自定义的工作流定义

---

# 三、OpenSpec 命令与工作流详解

幽人 *2026年5月10日 20:41*

> 本篇详细介绍 OpenSpec 的所有命令和工作流模式，帮助你在实际开发中选择最合适的工具。

## 3.1 命令体系概述

OpenSpec 提供两套命令体系：

1. **Slash 命令** ：在 AI 编程助手的聊天界面中使用
2. **CLI 命令** ：在终端中使用

## 3.2 Slash 命令详解

### 3.2.1 核心模式命令（默认）

| 命令 | 用途 |
| --- | --- |
| `/opsx:propose` | 快速创建一个变更并生成所有规划工件 |
| `/opsx:explore` | 探索思考，调查问题，不创建任何工件 |
| `/opsx:apply` | 执行任务清单 |
| `/opsx:sync` | 将增量规范同步到主规范（可选） |
| `/opsx:archive` | 归档完成的变更 |

**语法：**

```
/opsx:propose [变更名称或描述]
/opsx:explore [主题]
/opsx:apply [变更名称]
/opsx:sync [变更名称]
/opsx:archive [变更名称]
```

**示例：**

```
你：/opsx:propose add-dark-mode

AI：已创建 openspec/changes/add-dark-mode/
     ✓ proposal.md
     ✓ specs/ui/spec.md
     ✓ design.md
     ✓ tasks.md
     准备实现！运行 /opsx:apply
```

### 3.2.2 扩展模式命令

启用扩展模式：

```
openspec config profile
openspec update
```

| 命令 | 用途 |
| --- | --- |
| `/opsx:new` | 开始一个新变更脚手架 |
| `/opsx:continue` | 根据依赖关系创建下一个工件 |
| `/opsx:ff` | 快速推进，一次性创建所有规划工件 |
| `/opsx:verify` | 验证实现是否符合规范 |
| `/opsx:bulk-archive` | 批量归档多个变更 |
| `/opsx:onboard` | 引导式教程 |

**`/opsx:new` 示例：**

```
你：/opsx:new add-dark-mode

AI：已创建 openspec/changes/add-dark-mode/
     Schema: spec-driven

     准备创建：proposal
     使用 /opsx:continue 创建它，或 /opsx:ff 创建所有工件。
```

**`/opsx:continue` 示例：**

```
你：/opsx:continue

AI：变更：add-dark-mode

     工件状态：
     ✓ proposal    (已完成)
     ◆ specs       (就绪)
     ◆ design      (就绪)
     ○ tasks       (阻塞 - 需要：specs)

     正在创建 specs...

     [读取 proposal.md，基于需求创建 specs]

     ✓ 已创建 openspec/changes/add-dark-mode/specs/ui/spec.md

     现在可用：tasks
     运行 /opsx:continue 创建下一个工件。
```

**`/opsx:ff` 示例：**

```
你：/opsx:ff add-dark-mode

AI：快速推进 add-dark-mode...

     ✓ 正在创建 proposal.md
     ✓ 正在创建 specs/ui/spec.md
     ✓ 正在创建 design.md
     ✓ 正在创建 tasks.md

     所有规划工件完成！
     准备实现。运行 /opsx:apply 开始。
```

**`/opsx:verify` 示例：**

```
你：/opsx:verify

AI：正在验证 add-dark-mode...

     完整性
     ✓ tasks.md 中的所有 8 个任务已完成
     ✓ spec 中的所有需求都有对应代码
     ⚠ 场景"系统偏好检测"没有测试覆盖

     正确性
     ✓ 实现符合规范意图
     ✓ 处理了场景中的边缘情况
     ✓ 错误状态符合规范定义

     一致性
     ✓ 设计决策反映在代码结构中
     ✓ 命名约定与 design.md 一致
     ⚠ 设计提到"CSS 变量"但实现使用 Tailwind 类

     摘要
     ─────────────────────────────
     关键问题：0
     警告：2
     可以归档：是（带警告）

     建议：
     1. 为系统偏好检测添加测试
     2. 更新 design.md 反映 Tailwind 使用情况，或重构使用 CSS 变量
```

**`/opsx:sync` 示例：**

```
你：/opsx:sync

AI：正在同步 add-dark-mode 增量规范...

     正在读取：openspec/changes/add-dark-mode/specs/ui/spec.md
     目标： openspec/specs/ui/spec.md

     要应用的更改：
     ✓ 新增：主题选择需求（2 个场景）
     ✓ 新增：可访问性需求（1 个场景）

     正在合并...
     ✓ openspec/specs/ui/spec.md 已更新

     变更保持活动状态。运行 /opsx:archive 完成。
```

**`/opsx:archive` 示例：**

```
你：/opsx:archive

AI：正在归档 add-dark-mode...

     工件状态：
     ✓ proposal.md 存在
     ✓ specs/ 存在
     ✓ design.md 存在
     ✓ tasks.md 存在（8/8 任务完成）

     增量规范：尚未同步
     → 现在同步？（推荐）

你：是的

AI：✓ 已将规范同步到 openspec/specs/ui/spec.md
     ✓ 已移动到 openspec/changes/archive/2025-01-24-add-dark-mode/

     变更归档成功。
```

**`/opsx:bulk-archive` 示例：**

```
你：/opsx:bulk-archive

AI：找到 3 个已完成的变更：
     - add-dark-mode (8/8 任务完成)
     - fix-login-bug (3/3 任务完成)
     - update-footer (2/2 任务完成)

     正在检查规范冲突...
     ⚠ add-dark-mode 和 update-footer 都修改 specs/ui/

     检查代码库以解决冲突...
     两个变更都已实现。将按时间顺序合并：
     1. add-dark-mode（1 月 20 日创建）
     2. update-footer（1 月 22 日创建）

     归档所有 3 个变更？

你：是的

AI：✓ 已归档 add-dark-mode
     ✓ 已归档 fix-login-bug
     ✓ 已归档 update-footer
     规范合并：add-dark-mode → update-footer
```

**`/opsx:onboard` 示例：**

```
你：/opsx:onboard

AI：欢迎使用 OpenSpec！

     我将使用你的实际代码库带你完成完整的工作流。
     我们会找一个小而安全的改进点，为它创建正式的变更，
     实现它，然后归档它。

     让我扫描你的代码库以寻找机会...

     [分析代码库]

     我发现几个可以处理的事项：
     1. 为联系表单添加输入验证
     2. 改进 auth 流中的错误消息
     3. 为异步按钮添加加载状态

     你对哪个感兴趣？（或提出其他建议）
```

## 3.3 工作流模式

### 3.3.1 快速功能模式

当你知道要构建什么，只需要执行时：

```
/opsx:new ──► /opsx:ff ──► /opsx:apply ──► /opsx:verify ──► /opsx:archive
```

**最佳用途：** 小到中等功能、错误修复、简单的变更。

### 3.3.2 探索模式

当需求不明确或需要先调查时：

```
/opsx:explore ──► /opsx:new ──► /opsx:continue ──► ... ──► /opsx:apply
```

**最佳用途：** 性能优化、调试、架构决策、需求不明确的情况。

### 3.3.3 并行变更

同时处理多个变更：

```
变更 A: /opsx:new ──► /opsx:ff ──► /opsx:apply (进行中)
                                         │
                                    上下文切换
                                         │
变更 B: /opsx:new ──► /opsx:ff ──────► /opsx:apply
```

**最佳用途：** 并行工作流、紧急中断、团队协作。

当你有多个已完成的变更时，使用 `/opsx:bulk-archive` ：

```
你：/opsx:bulk-archive

AI：找到 3 个已完成的变更...
     [检测并解决规范冲突]
     按时间顺序归档所有
```

### 3.3.4 完成模式

推荐的完成流程：

```
/opsx:apply ──► /opsx:verify ──► /opsx:archive
                    │                 │
              验证实现            提示同步（如需要）
```

## 3.4 何时使用什么

### 3.4.1 /opsx:ff vs /opsx:continue

| 场景 | 使用 |
| --- | --- |
| 需求明确，准备构建 | `/opsx:ff` |
| 探索中，想审查每一步 | `/opsx:continue` |
| 想在规范之前迭代提案 | `/opsx:continue` |
| 时间压力，需要快速行动 | `/opsx:ff` |
| 复杂变更，想要控制 | `/opsx:continue` |

**经验法则：** 如果你能完整描述范围，使用 `/opsx:ff` 。如果正在摸索中，使用 `/opsx:continue` 。

### 3.4.2 更新 vs 新建变更

**更新现有变更当：**

- • 同一意图，更精细的执行
- • 范围缩小（先发布 MVP，其他后续）
- • 学习驱动的修正（代码库不是你想的那样）

**新建变更当：**

- • 意图基本改变
- • 范围膨胀到完全不同的工作
- • 原始变更可以独立标记为"完成"
- • 补丁会使事情变得更混乱而不是更清晰

### 3.4.3 何时运行 /opsx:sync

运行 sync 当：

| 场景 | 使用 sync？ |
| --- | --- |
| 长期变更，希望归档前规范在主分支中 | 是 |
| 多个并行变更需要更新的基础规范 | 是 |
| 想单独预览/审查合并 | 是 |
| 快速变更，直接归档 | 否（archive 处理） |

## 3.5 CLI 命令参考

### 3.5.1 常用命令

```
# 初始化项目
openspec init

# 列出所有变更
openspec list

# 列出所有规范
openspec list --specs

# 查看变更详情
openspec show <change-name>

# 查看规范详情
openspec show <规范名> --type spec

# 查看工件状态
openspec status --change <change-name>

# 验证变更
openspec validate <change-name>

# 验证所有
openspec validate --all

# 归档变更
openspec archive <change-name>

# 归档所有（跳过提示）
openspec archive <change-name> --yes

# 获取某工件指令
openspec instructions proposal --change <change-name>

# 列出可用模式
openspec schemas

# 查看配置
openspec config list
```

### 3.5.2 模式命令

```
# 创建新模式（交互式）
openspec schema init <name>

# 创建新模式（非交互式）
openspec schema init rapid --description "快速迭代工作流" --artifacts "proposal,tasks" --default

# 复制现有模式
openspec schema fork spec-driven my-workflow

# 验证模式
openspec schema validate my-workflow

# 查看模式来源
openspec schema which my-workflow
```

### 3.5.3 配置命令

```
# 查看配置路径
openspec config path

# 列出所有设置
openspec config list

# 获取特定值
openspec config get telemetry.enabled

# 设置值
openspec config set telemetry.enabled false

# 配置文件
openspec config edit

# 配置工作流
openspec config profile
```

### 3.5.4 工作空间命令

```
# 设置工作空间（交互式）
openspec workspace setup

# 设置工作空间（非交互式）
openspec workspace setup --no-interactive --name platform --link /repos/api --link web=/repos/web

# 列出工作空间
openspec workspace list
openspec workspace ls

# 链接仓库
openspec workspace link /repos/api

# 重新链接
openspec workspace relink api-service /new/path/to/api

# 检查工作空间
openspec workspace doctor
openspec workspace doctor --workspace platform

# 打开工作空间
openspec workspace open
openspec workspace open platform --agent github-copilot
```

### 3.5.5 工具命令

```
# 更新配置文件
openspec update

# 安装特定工具
openspec init --tools claude,cursor

# 安装所有工具
openspec init --tools all

# 提交反馈
openspec feedback "添加自定义工件类型支持" --body "我想定义自己的工件类型..."
```

## 3.6 工作流最佳实践

### 3.6.1 保持变更聚焦

每个变更应该是逻辑上的一个工作单元。如果你在做"添加功能 X 并重构 Y"，考虑分成两个单独的变更。

**为什么重要：**

- • 更容易审查和理解
- • 更干净的历史记录
- • 可以独立发布
- • 更简单的回滚（如需要）

### 3.6.2 使用 /opsx:explore 处理不明确的需求

在承诺变更之前，探索问题空间：

```
你：/opsx:explore

AI：你想探索什么？

你：我们应该如何处理 API 的限流？
```

探索可以在创建工件之前澄清思路。

### 3.6.3 归档前验证

使用 `/opsx:verify` 检查实现是否与工件匹配：

```
你：/opsx:verify

AI：验证 add-dark-mode...

     ✓ tasks.md 中的所有任务已完成
     ✓ spec 中的需求有对应代码

     可以归档！
```

在关闭变更之前捕获不匹配。

### 3.6.4 命名清晰

好的命名使 `openspec list` 有用：

```
好的：                      避免：
add-dark-mode              feature-1
fix-login-redirect         update
optimize-product-query     changes
implement-2fa              wip
```

## 3.7 命令速查表

### 3.7.1 核心模式

| 命令 | 用途 | 何时使用 |
| --- | --- | --- |
| `/opsx:propose` | 创建变更 + 规划工件 | 快速默认路径 |
| `/opsx:explore` | 思考想法 | 需求不明确、调查 |
| `/opsx:apply` | 实现任务 | 准备写代码 |
| `/opsx:sync` | 合并增量规范 | 扩展模式，可选 |
| `/opsx:archive` | 完成变更 | 所有工作完成 |

### 3.7.2 扩展模式

| 命令 | 用途 | 何时使用 |
| --- | --- | --- |
| `/opsx:new` | 开始变更脚手架 | 扩展模式，显式工件控制 |
| `/opsx:continue` | 创建下一个工件 | 扩展模式，逐步创建 |
| `/opsx:ff` | 创建所有规划工件 | 扩展模式，范围明确 |
| `/opsx:verify` | 验证实现 | 扩展模式，归档前 |
| `/opsx:sync` | 合并增量规范 | 可选，手动 |
| `/opsx:archive` | 归档变更 | 所有工作完成 |
| `/opsx:bulk-archive` | 批量归档 | 扩展模式，并行工作 |
| `/opsx:onboard` | 引导教程 | 学习工作流 |

## 3.8 总结

本篇详细介绍了 OpenSpec 的所有命令和工作流模式。

关键要点：

1. **两种模式** ：核心模式（默认）和扩展模式
2. **`/opsx:ff`** 用于范围明确的快速创建
3. **`/opsx:continue`** 用于逐步创建和精细控制
4. **`/opsx:verify`** 在归档前验证实现
5. **`/opsx:bulk-archive`** 用于并行工作流
6. **CLI 命令** 补充 slash 命令，提供终端管理能力

---

# 四、OpenSpec 自定义与配置

幽人 *2026年5月11日 21:19*

> 本篇介绍 OpenSpec 的自定义选项，包括项目配置、模式自定义和工作流定制。

## 4.1 自定义层级

OpenSpec 提供三个自定义层级：

| 层级 | 作用 | 最适合 |
| --- | --- | --- |
| **项目配置** | 设置默认值，注入上下文/规则 | 大多数团队 |
| **自定义模式** | 定义自己的工件和工作流 | 有独特流程的团队 |
| **全局覆盖** | 跨项目共享模式 | 高级用户 |

## 4.2 项目配置

### 4.2.1 配置位置

项目配置文件位于 `openspec/config.yaml` ：

```
your-project/
├── openspec/
│   ├── config.yaml        # 项目配置
│   ├── schemas/          # 自定义模式
│   └── changes/         # 变更
└── src/
```

### 4.2.2 创建配置

运行初始化时自动创建，或手动创建：

```
# openspec/config.yaml
schema: spec-driven

context: |
  技术栈：TypeScript、React、Node.js、PostgreSQL
  API 风格：RESTful，文档在 docs/api.md
  测试：Jest + React Testing Library
  我们重视所有公共 API 的向后兼容性

rules:
  proposal:
    - 包含回滚计划
    - 确定受影响的团队
  specs:
    - 使用 Given/When/Then 格式
    - 借鉴现有模式，避免创新
```

### 4.2.3 配置字段

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `schema` | string | 新变更的默认模式 |
| `context` | string | 注入所有工件的上下文 |
| `rules` | object | 每个工件的规则，按工件 ID 键入 |

### 4.2.4 如何工作

**默认模式：**

```
# 无配置
openspec new change my-feature --schema spec-driven

# 有配置 - 自动使用默认
openspec new change my-feature
```

**上下文和规则注入：**

生成任何工件时，你的上下文和规则被注入到 AI 提示中：

```
<context>
技术栈：TypeScript、React、Node.js、PostgreSQL
...
</context>

<rules>
- 包含回滚计划
- 确定受影响的团队
</rules>

<template>
[模式的内置模板]
</template>
```
- • **上下文** 出现在所有工件中
- • **规则** 只出现在匹配的工件中

### 4.2.5 模式解析顺序

当 OpenSpec 需要一个模式时，按此顺序检查：

1. CLI 标志： `--schema <name>`
2. 变更元数据（`.openspec.yaml` 在变更目录中）
3. 项目配置（ `openspec/config.yaml` ）
4. 默认（ `spec-driven` ）

## 4.3 自定义模式

### 4.3.1 何时使用

当项目配置不够时，创建完全自定义的工作流。例如：

- • 添加专属工件类型（如 review、research）
- • 自定义工件依赖关系
- • 完全控制的工件模板

### 4.3.2 创建方式

**复制现有模式（推荐）：**

```
openspec schema fork spec-driven my-workflow
```

这会将 `spec-driven` 模式复制到 `openspec/schemas/my-workflow/` ，你可以自由编辑。

**从头创建：**

```
# 交互式
openspec schema init research-first

# 非交互式
openspec schema init rapid \
  --description "快速迭代工作流" \
  --artifacts "proposal,tasks" \
  --default
```

### 4.3.3 模式结构

```
openspec/schemas/my-workflow/
├── schema.yaml           # 工作流定义
└── 模板/
    ├── proposal.md      # proposal 工件的模板
    ├── spec.md          # specs 工件的模板
    ├── design.md        # design 工件的模板
    └── tasks.md        # tasks 工件的模板
```

### 4.3.4 schema.yaml 示例

```
# openspec/schemas/my-workflow/schema.yaml
name: my-workflow
version: 1
description: 我的团队的自定义工作流

artifacts:
  - id: proposal
    generates: proposal.md
    description: 初始提案文档
    template: proposal.md
    instruction: |
      创建一个解释为什么需要此变更的提案。
      关注问题，而不是解决方案。
    requires: []

  - id: design
    generates: design.md
    description: 技术设计
    template: design.md
    instruction: |
      创建一个解释如何实现的设计文档。
    requires:
      - proposal    # proposal 存在前无法创建 design

  - id: tasks
    generates: tasks.md
    description: 实现检查清单
    template: tasks.md
    requires:
      - design

apply:
  requires: [tasks]
  tracks: tasks.md
```

### 4.3.5 关键字段

| 字段 | 用途 |
| --- | --- |
| `id` | 唯一标识符，用于命令和规则 |
| `generates` | 输出文件名（支持 glob 如 `specs/**/*.md` ） |
| `template` | `模板/` 目录中的模板文件 |
| `instruction` | 创建此工件的 AI 指令 |
| `requires` | 哪些工件必须先存在 |

### 4.3.6 模板文件

模板是指导 AI 创建工件的 Markdown 文件。可以包含：

- • AI 应填充的部分标题
- • 包含 AI 指导的 HTML 注释
- • 显示预期结构的示例格式
```
<!-- 模板/proposal.md -->
## 为什么

<!-- 解释此变更的动机。此变更解决什么问题？ -->

## 什么会改变

<!-- 描述将发生什么。具体说明新能力或修改。 -->

## 影响

<!-- 受影响的代码、API、依赖、系统 -->
```

### 4.3.7 验证模式

使用前验证模式：

```
openspec schema validate my-workflow
```

验证检查：

- • `schema.yaml` 语法正确
- • 所有引用的模板存在
- • 无循环依赖
- • 工件 ID 有效

### 4.3.8 使用自定义模式

创建后，通过以下方式使用：

```
# 命令中指定
openspec new change feature --schema my-workflow

# 或设置为 config.yaml 中的默认值
schema: my-workflow
```

### 4.3.9 调试模式解析

不确定使用哪个模式？检查：

```
# 查看特定模式的来源
openspec schema which my-workflow

# 列出所有可用模式
openspec schema which --all
```

输出显示来自项目、用户目录还是包：

```
模式：my-workflow
来源：project
路径：/path/to/project/openspec/schemas/my-workflow
```

### 4.3.10 模式解析优先级

1. 项目： `openspec/schemas/<name>/`
2. 包：内置模式

## 4.4 社区模式

OpenSpec 还支持社区维护的模式，通过独立仓库分发。

| 模式 | 维护者 | 仓库 | 描述 |
| --- | --- | --- | --- |
| `superpowers-bridge` | @JiangWay | JiangWay/openspec-schemas | 将 OpenSpec 的工件治理与 obra/superpowers 执行技能集成 |

### 4.4.1 使用社区模式

要使用社区模式，将模式包复制到项目的 `openspec/schemas/<schema-name>/` 目录。每个仓库的 README 有安装说明。

## 4.5 OPSX 自定义

### 4.5.1 项目配置字段详解

项目配置文件（ `openspec/config.yaml` ）允许你设置默认值并将项目特定的上下文注入所有工件。

**创建配置：**

配置在 `openspec init` 期间创建，或手动创建：

```
# openspec/config.yaml
schema: spec-driven

context: |
  Tech stack: TypeScript, React, Node.js
  API conventions: RESTful, JSON responses
  Testing: Vitest for unit tests, Playwright for e2e
  Style: ESLint with Prettier, strict TypeScript

rules:
  proposal:
    - Include rollback plan
    - Identify affected teams
  specs:
    - Use Given/When/Then format for scenarios
  design:
    - Include sequence diagrams for complex flows
```

**配置字段：**

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `schema` | string | 新变更的默认模式 |
| `context` | string | 注入所有工件的项目上下文 |
| `rules` | object | 每个工件的规则 |

### 4.5.2 规则注入

- • 上下文被前置到每个工件的指令
- • 用 `<context>...</context>` 标签包装
- • 帮助 AI 理解项目的约定
- • 规则仅注入到匹配的工件
- • 用 `<rules>...</rules>` 标签包装
- • 在上下文之后、模板之前出现

### 4.5.3 配置验证

- • 规则中未知的工件 ID 生成警告
- • 模式名称根据可用模式验证
- • 上下文有 50KB 大小限制
- • 无效 YAML 带行号报告

### 4.5.4 故障排除

**"规则中未知的工件 ID: X"**

- • 检查工件 ID 与你的模式匹配
- • 运行 `openspec schemas --json` 查看每个模式的工件 ID

**配置未应用：**

- • 确保文件在 `openspec/config.yaml` （不是 `.yml` ）
- • 用验证器检查 YAML 语法
- • 配置更改立即生效（无需重启）

**上下文太大：**

- • 上下文限制为 50KB
- • 总结或链接到外部文档

## 4.6 工作流自定义

### 4.6.1 核心 vs 扩展模式

**核心模式（默认）：**

默认全局配置文件使用 `core` ，包含： `propose` 、 `explore` 、 `apply` 、 `sync` 、 `archive`

```
/opsx:propose ──► /opsx:apply ──► /opsx:sync ──► /opsx:archive
```

**扩展工作流：**

启用扩展工作流命令：

```
openspec config profile
openspec update
```

扩展模式提供： `new` 、 `continue` 、 `ff` 、 `verify` 、 `bulk-archive` 、 `onboard`

### 4.6.2 快速切换

```
# 切换到核心模式
openspec config profile core
openspec update

# 切换扩展模式
openspec config profile
# 选择自定义工作流
openspec update
```

## 4.7 总结

本篇介绍了 OpenSpec 的自定义和配置选项。

关键要点：

1. **config.yaml** ：最简单的方式设置默认值和注入上下文
2. **自定义模式** ：完全控制工件和依赖关系
3. **模式解析优先级** ：CLI 标志 > 变更元数据 > 配置 > 默认
4. **模板可编辑** ：自定义模式可以编辑模板
5. **社区模式** ：使用预构建的工作流

---

# 五、OpenSpec 迁移与多语言支持

幽人 *2026年5月12日 22:31*

> 本篇介绍从旧版 OpenSpec 迁移到 OPSX 的方法，以及如何配置多语言支持。

## 5.1 迁移到 OPSX

### 5.1.1 发生了什么变化？

OPSX 用流体化、基于行动的方法取代旧的阶段锁定工作流。以下是关键变化：

| 方面 | 旧版 | OPSX |
| --- | --- | --- |
| **命令** | `/openspec:propose` 、 `/openspec:apply` 、 `/openspec:archive` | 默认： `/opsx:propose` 、 `/opsx:apply` 、 `/opsx:sync` 、 `/opsx:archive` （扩展模式命令可选） |
| **工作流** | 一次性创建所有工件 | 增量或一次性创建——由你选择 |
| **回退** | 尴尬的阶段门 | 自然——随时更新任何工件 |
| **自定义** | 固定结构 | 模式驱动，完全可定制 |
| **配置** | `CLAUDE.md` 标记 + `project.md` | 干净的 `config.yaml` 配置 |

### 5.1.2 旧工作安全

迁移过程设计为保留你的现有工作：

- • **活跃变更（ `openspec/changes/` ）** ：完全保留，可以使用 OPSX 命令继续
- • **归档变更** ：保持不变，历史记录保持完整
- • **主规范（ `openspec/specs/` ）** ：保持不变，这是你的真相来源
- • **你在 `CLAUDE.md` 、 `AGENTS.md` 等中的内容** ：保持不变，只有 OpenSpec 标记被移除

### 5.1.3 什么被移除

仅移除被替换的 OpenSpec 管理文件：

- • 旧的斜杠命令目录/文件（被新的技能系统取代）
- • `openspec/AGENTS.md` （已过时的工作流触发器）
- • `CLAUDE.md` 、 `AGENTS.md` 等中的 OpenSpec 标记

**注意：** `openspec/project.md` 不会自动删除，因为它可能包含你编写的内容。你需要：

1. 审查其内容
2. 将有用的上下文移到 `config.yaml` 的 `context` 部分
3. 准备好后删除该文件

### 5.1.4 运行迁移

**使用 `openspec init` ：**

```
openspec init
```

检测旧文件并引导你完成清理：

```
升级到新的 OpenSpec

OpenSpec 现在使用代理技能，这是编码的
代理的新兴标准。这简化了你的设置
同时保持一切如前工作。

要删除的文件
无用户内容保留：
  • .claude/commands/openspec/
  • openspec/AGENTS.md

要更新的文件
OpenSpec 标记将被移除，内容保留：
  • CLAUDE.md
  • AGENTS.md

需要你的关注
  • openspec/project.md
    我们不会删除此文件。它可能包含有用的项目上下文。

    新的 openspec/config.yaml 有一个"上下文："部分用于规划
    上下文。这被包含在每个 OpenSpec 请求中，并且
    比旧的 project.md 方法更可靠。

    审查 project.md，将有用的内容移到 config.yaml 的 context
    部分，然后准备好后删除该文件。
```

**说"是"后发生什么：**

1. 删除旧的斜杠命令目录
2. 从 `CLAUDE.md` 、 `AGENTS.md` 等中移除 OpenSpec 标记（你的内容保留）
3. 删除 `openspec/AGENTS.md`
4. 在 `.claude/skills/` 中安装新技能
5. 创建带有默认模式的 `openspec/config.yaml`

**使用 `openspec update` ：**

如果你只想迁移并刷新现有工具到最新版本：

```
openspec update
```

### 5.1.5 非交互式 / CI 环境

对于脚本化迁移：

```
openspec init --force --tools claude
```

`--force` 标志跳过提示并自动接受清理。

### 5.1.6 迁移 project.md 到 config.yaml

旧 `openspec/project.md` 是用于项目上下文的自由格式 Markdown 文件。新的 `openspec/config.yaml` 是结构化的，并且——关键地—— **被注入到每个规划请求** 中，所以你的约定总是在 AI 创建工件时存在。

**之前（project.md）：**

```
# 项目上下文

这是一个使用 React 和 Node.js 的 TypeScript monorepo。
我们使用 Jest 进行测试，并遵循严格的 ESLint 规则。
我们的 API 是 RESTful 的，文档在 docs/api.md 中。

## 约定

- 所有公共 API 必须保持向后兼容性
- 新功能应包含测试
- 规范使用 Given/When/Then 格式
```

**之后（config.yaml）：**

```
schema: spec-driven

context: |
  技术栈：TypeScript、React、Node.js
  测试：Jest + React Testing Library
  API：RESTful，文档在 docs/api.md
  我们为所有公共 API 维护向后兼容性

rules:
  proposal:
    - Include rollback plan for risky changes
  specs:
    - Use Given/When/Then format for scenarios
    - Reference existing patterns before inventing new ones
  design:
    - Include sequence diagrams for complex flows
```

**关键差异：**

| project.md | config.yaml |
| --- | --- |
| 自由格式 Markdown | 结构化 YAML |
| 一段文本 | 分开 context 和 per-artifact rules |
| 不清楚何时使用 | 上下文出现在所有工件；规则仅出现在匹配的工件 |
| 无模式选择 | 显式 `schema:` 字段设置默认工作流 |

**迁移步骤：**

1. **创建 config.yaml** （如果 init 还没有创建）：
   ```
   schema: spec-driven
   ```
2. **添加你的 context** （简洁——这会在每个请求中出现）：
   ```
   context: |
     你的项目背景在这里。
     关注 AI 真正需要知道的。
   ```
3. **添加 per-artifact rules**
   ```
   rules:
     proposal:
       - 你的 proposal 特定指南
     specs:
       - 你的 spec 编写规则
   ```
4. **删除 project.md** 当你移动完有用的内容。

**不要想太多。** 从最基本的开始迭代。如果你注意到 AI 遗漏了什么重要的，添加它。如果 context 感觉太冗长，缩减它。这是一份活文档。

### 5.1.7 继续现有变更

你进行的变更与 OPSX 命令无缝协作。

**有旧工作流的活跃变更？**

```
/opsx:apply add-my-feature
```

OPSX 读取现有工件并从你停下的地方继续。

**想向现有变更添加更多工件？**

```
/opsx:continue add-my-feature
```

显示基于已存在的内容可以创建什么。

**需要查看状态？**

```
openspec status --change add-my-feature
```

## 5.2 迁移后的新命令

### 5.2.1 命令可用性

**默认（core 配置文件）：**

| 命令 | 用途 |
| --- | --- |
| `/opsx:propose` | 快速创建一个变更并生成规划工件 |
| `/opsx:explore` | 思考想法，无结构要求 |
| `/opsx:apply` | 执行 tasks.md 中的任务 |
| `/opsx:archive` | 完成并归档变更 |

**扩展工作流（自定义选择）：**

| 命令 | 用途 |
| --- | --- |
| `/opsx:new` | 开始新的变更脚手架 |
| `/opsx:continue` | 一次创建一个工件 |
| `/opsx:ff` | 快速推进——一次性创建规划工件 |
| `/opsx:verify` | 验证实现是否符合规范 |
| `/opsx:sync` | 将增量规范合并到主规范 |
| `/opsx:bulk-archive` | 批量归档多个变更 |
| `/opsx:onboard` | 引导完成整个工作流的教程 |

启用扩展命令：

```
openspec config profile
openspec update
```

### 5.2.2 命令映射

| 旧版 | OPSX 等效 |
| --- | --- |
| `/openspec:propose` | `/opsx:propose` （默认）或 `/opsx:new` + `/opsx:ff` （扩展） |
| `/openspec:apply` | `/opsx:apply` |
| `/openspec:archive` | `/opsx:archive` |

### 5.2.3 新能力

这些是扩展工作流命令集的一部分。

**精细化工件创建：**

```
/opsx:continue
```

一次基于依赖关系创建一个工件。使用这个当你想审查每一步。

**探索模式：**

```
/opsx:explore
```

在承诺变更之前，像伙伴一样思考想法。

## 5.3 多语言支持

### 5.3.1 快速设置

在 `openspec/config.yaml` 中添加语言说明：

```
schema: spec-driven

context: |
  语言：简体中文
  所有产出物必须用简体中文撰写。

  # 你的其他项目上下文...
  技术栈：TypeScript、React、Node.js
```

所有生成的工件现在将用简体中文。

### 5.3.2 语言示例

**葡萄牙语（巴西）：**

```
context: |
  语言：葡萄牙语（pt-BR）
  所有产出物必须用巴西葡萄牙语撰写。
```

**西班牙语：**

```
context: |
  语言：西班牙语
  所有产出物必须用西班牙语撰写。
```

**简体中文：**

```
context: |
  语言：简体中文
  所有产出物必须用简体中文撰写。
```

**日语：**

```
context: |
  语言：日语
  所有产出物必须用日语撰写。
```

**法语：**

```
context: |
  语言：法语
  所有产出物必须以法语撰写。
```

**德语：**

```
context: |
  语言：德语
  所有产出物必须用德语撰写。
```

### 5.3.3 处理技术术语

决定如何处理技术术语：

```
context: |
  语言：日语
  用日语撰写，但：
  - 将"API"、"REST"、"GraphQL"等技术术语保留为英语
  - 代码示例和文件路径保留为英语
```

### 5.3.4 与其他上下文结合

语言设置可以与其他项目上下文结合：

```
schema: spec-driven

context: |
  语言：简体中文
  所有产出物必须用简体中文撰写。

  技术栈：TypeScript、React 18、Node.js 20
  数据库：PostgreSQL + Prisma ORM
```

### 5.3.5 验证语言配置

验证语言配置是否生效：

```
# 检查指令——应显示你的语言上下文
openspec instructions proposal --change my-change

# 输出将包含你的语言上下文
```

## 5.4 总结

本篇介绍了 OpenSpec 的迁移和多语言支持。

关键要点：

1. **迁移安全** ：现有工作完全保留
2. **project.md 迁移** ：需要手动移动到 config.yaml
3. **新命令** ：核心模式和扩展模式
4. **多语言** ：通过 context 配置简单设置

---

**微信扫一扫赞赏作者**

openspec-系统化教程 · 目录

作者提示: 个人观点，仅供参考
