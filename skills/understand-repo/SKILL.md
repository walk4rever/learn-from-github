---
name: understand-repo
description: >-
  帮助普通技术人快速理解一个陌生的 GitHub 项目。通过 5 个并行分析 Agent，从技术栈、
  架构设计、入口流程、依赖意图、开发环境五个维度深度分析代码库，
  最终生成结构化的《项目理解指南》(UNDERSTANDING.md)，并进入交互式答疑模式。
  TRIGGER when: 用户想理解、学习、研究一个 clone 下来的 GitHub 项目，
  或说了"帮我看看这个项目"、"分析这个仓库"、"这个项目怎么工作的"。
  DO NOT TRIGGER when: 用户只想修改或调试某个具体文件。
origin: judge-the-code
version: 1.3.0
---

# Understand Repo — GitHub 项目理解向导

帮助普通技术人快速建立对陌生代码库的完整心智模型。

## 核心理念

普通技术人 clone 一个项目时，最大的困境不是看不懂代码，而是**不知道从哪开始看**。
这个 skill 的目标是：30 分钟内让你对一个陌生项目建立足够的全局认知。

## 调用方式

```
/understand-repo <项目路径>
/understand-repo /path/to/cloned/project
/understand-repo .   (当前目录)
```

---

## Token 使用原则

> **核心策略：用结构推断意图，而不是读源码找答案。**
>
> ⚠️ **作用域**：以下规则**仅适用于 Phase 1**（初始分析阶段）。
> Phase 3 导览模式中可以读取源码文件，但遵循"每步 1 文件、最多 150 行"的独立约束。

**Phase 1 的所有 Agent 必须严格遵守以下规则：**

1. **Glob 优先于 Read**：获取目录结构时只用 Glob 拿文件路径列表，不读文件内容
2. **Grep 优先于 Read**：需要找特定信息时用 Grep 定向搜索，而不是把整个文件读进来
3. **限制读取行数**：
   - 配置文件（`package.json`, `go.mod` 等）：最多读前 **80 行**
   - README.md：最多读前 **150 行**
   - 入口/源码文件：最多读前 **60 行**
   - `.env.example`：最多读前 **50 行**
4. **禁止读取的文件类型**：
   - 业务逻辑实现文件（`services/`, `controllers/` 内的 `.ts/.js/.py/.go` 等）
   - 测试文件（`*.test.*`, `*.spec.*`, `*_test.go`）
   - 生成文件（`dist/`, `build/`, `.next/`, `node_modules/`）
   - 大型静态资源（图片、字体、二进制文件）
5. **每个 Agent 读取文件数上限：10 个**

---

## 执行流程

### Phase 0：确认目标 & 检查历史状态

1. **输入验证**（路径异常时立即停止并提示，不继续执行）：

   - **路径为空** → 提示：
     ```
     ❌ 请提供项目路径，例如：
        /understand-repo .              （当前目录）
        /understand-repo ~/projects/my-repo
     ```
   - **路径不存在** → 提示：
     ```
     ❌ 路径 `{path}` 不存在，请检查路径是否正确。
     ```
   - **路径是文件而非目录** → 提示：
     ```
     ❌ `{path}` 是一个文件，请提供项目根目录路径。
     ```

2. **检查是否存在 `.repo-context/PROGRESS.md`**（之前会话留下的状态）：

   **如果不存在** → 直接进入 Phase 1 全量分析

   **如果存在** → 执行 **代码变更检测**：

   首先判断被分析项目是否为 git 仓库：

   ```bash
   git rev-parse HEAD 2>/dev/null
   ```

   根据结果走不同分支：

   **分支 A：是 git 仓库** → 比较 commit hash：

   - hash 与 `PROGRESS.md` 中 `analyzed_at_commit` 相同 → 直接恢复
   - hash 不同 → 执行 diff 分析：
     ```bash
     git diff --stat {stored_commit}..HEAD
     ```
     交叉比对变更文件 vs `KNOWLEDGE.md` 已探索文件：

     | 类别 | 处理方式 |
     |------|---------|
     | **已探索 & 已变更** | 标记为 ⚠️ STALE，下次访问时重读 diff |
     | **未探索 & 已变更** | 仅记录，不影响现有理解 |
     | **配置文件变更** | 触发对应 Agent 局部重分析 |

   **分支 B：非 git 仓库**（zip 解压、无版本控制）→ 用时间戳判断：

   - 比较 `PROGRESS.md` 中的 `analyzed_at` 时间戳 vs 项目文件的最新修改时间：
     ```bash
     find . -newer .repo-context/PROGRESS.md -type f \
       ! -path './.repo-context/*' ! -path './node_modules/*' \
       ! -path './.git/*' | head -20
     ```
   - 如有文件比上次分析时间更新 → 提示用户："检测到文件有变动（非 git 项目，无法精确 diff）"
   - 提供两个选项：`[R] 直接恢复` 或 `[S] 全量重新分析`
   - 注意：非 git 项目**不提供** `[U] 局部更新`选项（没有 diff 信息）

   **变更摘要展示（git 仓库）：**

   ```
   📂 发现上次的学习记录 + 检测到代码更新

   上次分析: 2026-03-14 16:32（commit a3f7c21）
   当前版本: commit d9e2b84（3天前，共 23 个文件变更）

   对你已理解内容的影响：
   ⚠️  src/controllers/UserController.ts — 已变更（你上次探索过）
   ⚠️  src/services/UserService.ts — 已变更（你上次探索过）
   ✅  src/routes/api.ts — 未变更（你的理解仍然有效）
   📦  package.json — 已变更（依赖有更新）

   建议操作：
   [U] 局部更新：只重新分析 2 个变更文件 + 依赖（推荐）
   [R] 直接恢复：忽略变更，从上次位置继续（风险：部分理解可能过期）
   [S] 全量重新分析（代价最高，适合大版本升级）
   ```

   **选择 [U] 局部更新时的执行逻辑：**

   - 对每个 `⚠️ STALE` 文件：用 `git diff {stored_commit}..HEAD -- {文件路径}` 读取 diff（不读整文件），理解"改了什么"，更新 `KNOWLEDGE.md` 对应条目
   - 若 `package.json` 变更：单独重跑 Agent 4（依赖分析），更新 `KNOWLEDGE.md` 的依赖部分
   - 若目录结构有新增/删除：更新 `PROGRESS.md` 的路径规划
   - **不重跑**未变更维度的 Agent

   > **核心原则**：用 `git diff` 而不是重读整个文件。一个 200 行文件的 diff 通常只有 20-30 行，
   > 成本降低 10 倍，且能精确表达"变化了什么"。

3. 告诉用户当前状态后，继续后续流程

---

### Phase 1：并行 5 Agent 分析

**启动前，确定 skill 目录路径**（供各 Agent 加载详细规格）：

```bash
find ~/.agents/skills ~/.claude/skills -name "SKILL.md" -path "*/understand-repo/*" 2>/dev/null \
  | head -1 | xargs dirname
```

将结果记为 `{SKILL_DIR}`。

**输出进度公告：**

```
🚀 开始分析，同时启动 5 个 Agent...

  Agent 1 — 技术栈探测    ⏳ 分析中
  Agent 2 — 架构分析      ⏳ 分析中
  Agent 3 — 入口追踪      ⏳ 分析中
  Agent 4 — 依赖解析      ⏳ 分析中
  Agent 5 — 开发环境      ⏳ 分析中

预计完成时间：2-4 分钟
```

**同时启动以下 5 个 Explore 子 Agent，全部并行执行：**

每个 Agent 的启动格式：
> 读取 `{SKILL_DIR}/agents/agentN-xxx.md`，按其中的说明执行分析，完成后返回结果。

| Agent | 规格文件 | 分析维度 |
|-------|---------|---------|
| Agent 1 | `{SKILL_DIR}/agents/agent1-stack-detector.md` | 语言、框架、运行时 |
| Agent 2 | `{SKILL_DIR}/agents/agent2-architecture-mapper.md` | 目录结构、架构模式、Mermaid 图 |
| Agent 3 | `{SKILL_DIR}/agents/agent3-entry-point-tracer.md` | 启动入口、请求时序图 |
| Agent 4 | `{SKILL_DIR}/agents/agent4-dependency-analyst.md` | 依赖分类、技术选型解读 |
| Agent 5 | `{SKILL_DIR}/agents/agent5-dev-setup-guide.md` | 本地运行、硬件要求、环境变量 |

---

### Phase 2：综合输出

收集所有 5 个 Agent 的结果后，生成完整的 `UNDERSTANDING.md`：

```markdown
# [项目名] — 项目理解指南

> 生成时间: [date] | 分析工具: understand-repo skill

## TL;DR（30秒了解）
[2-3 句话：这是什么项目，解决什么问题，核心价值是什么]

## 技术栈一览
[来自 Agent 1]

## 架构设计
[来自 Agent 2]

## 程序入口与核心流程
[来自 Agent 3]

## 关键依赖解析
[来自 Agent 4]

## 本地开发指南
[来自 Agent 5]

> 包含"本地可运行性评估"小节，说明是否需要 GPU/特殊硬件，以及生产部署要求与本地开发环境的差距。

## 建议学习路径

对于想深入理解这个项目的开发者，建议按以下顺序阅读：

### 第一步：建立全局认知（30分钟）
1. 阅读 `README.md` 了解项目背景
2. 看 `[入口文件]` 理解启动流程
3. 浏览 `[路由/控制器文件]` 了解功能边界

### 第二步：理解核心业务（1-2小时）
1. 深读 `[最核心的 service 文件]`
2. 理解数据模型 `[model/schema 文件]`
3. 跟着一个核心功能的完整请求链路走一遍

### 第三步：掌握工程细节（按需）
1. 测试文件 — 了解预期行为
2. CI/CD 配置 — 了解质量门禁
3. 数据库迁移文件 — 了解数据演进历史

## 值得关注的设计决策
[分析中发现的有趣/独特的技术决策，解释背后的权衡]

## 常见问题
**Q: 项目的核心数据流是什么？**
A: [简要描述]

**Q: 认证是怎么工作的？**
A: [如果项目有认证的话]

**Q: 如何添加一个新的 API 端点？**
A: [基于架构给出步骤]
```

保存文件到项目根目录的 `UNDERSTANDING.md`。

**保存完成后，输出：**
```
📄 UNDERSTANDING.md 已生成
```

同时初始化状态目录（如不存在则创建）：

```bash
mkdir -p .repo-context
```

写入 `.repo-context/PROGRESS.md`（初始状态）：

```markdown
---
project: [项目名]
analyzed_at: [ISO 时间戳]
analyzed_at_commit: [git rev-parse HEAD 的输出，完整 40 位 hash]
current_path: none
current_step: 0
---

## 初始分析摘要

**技术栈**: [语言] + [框架]
**架构模式**: [MVC / Clean Arch / ...]
**入口文件**: [路径]
**项目类型**: [Web API / 前端 SPA / CLI / ...]

## 已探索文件

（空，等待用户开始导览）

## 学习路径状态

- 路径 A — [名称]: 未开始
- 路径 B — [名称]: 未开始
- 路径 C — [名称]: 未开始
```

写入 `.repo-context/KNOWLEDGE.md`（初始为空，后续增量追加）：

```markdown
---
last_updated: [ISO 时间戳]
files_explored: 0
---

## 文件知识库

（初始分析阶段只读了配置文件，源码文件知识将在导览中逐步积累）

### 配置文件

#### package.json
- **角色**: 项目依赖声明和命令入口
- **关键依赖**: [框架], [ORM], [工具库]
- **可用命令**: dev, build, test, lint
```

写入 `.repo-context/NOTES.md`（记录 Q&A 洞察）：

```markdown
---
last_updated: [ISO 时间戳]
qa_count: 0
---

## 问答记录 & 关键洞察

（会话中的重要问题和结论将记录在这里）
```

---

### Phase 3：渐进式深度学习模式

生成完 `UNDERSTANDING.md` 后，进入**渐进式学习模式**。

#### 3.1 提供学习路径菜单

基于 Phase 1 的分析结果，生成 3-5 条**具体的学习路径**（不是泛泛的建议，而是有序的文件阅读计划）。

> **注意**：以下是示例输出（基于 Node.js/Express 项目）。实际路径和文件名由 Phase 1 分析结果动态生成，Go/Python/Java 等项目的路径会完全不同。

```
✅ 分析完成！UNDERSTANDING.md 已生成。

根据这个项目的结构，我为你准备了几条学习路径，选一条开始？

📍 路径 A — 跟着一个 HTTP 请求走完整流程  ⏱ ~20分钟
   src/routes/api.ts → src/controllers/UserController.ts → src/services/UserService.ts → src/repositories/UserRepo.ts
   适合：想理解数据流和分层架构的人

📍 路径 B — 搞懂认证机制是怎么工作的  ⏱ ~15分钟
   src/middleware/auth.ts → src/services/JwtService.ts → src/models/User.ts
   适合：关注安全和用户系统的人

📍 路径 C — 了解数据库是怎么组织的  ⏱ ~15分钟
   prisma/schema.prisma → src/repositories/ → src/models/
   适合：关注数据层设计的人

📍 路径 D — 自由探索（直接问我任何问题）

输入 A/B/C/D 或直接提问 👇
```

#### 3.2 导览模式（选择 A/B/C 后）

进入**文件逐步导览**：每次只呈现路径中的**一个文件**，解释完再问用户是否继续。

**每一步的标准格式：**

```
📂 第 2 步 / 共 4 步  [路径 A]
已加载: src/controllers/UserController.ts（已读文件: 2个）

[对这个文件的解释]

用通俗语言说明：
- 这个文件是干什么的（一句话）
- 它和上一个文件的关系（承上）
- 它依赖下一个文件做什么（启下）
- 值得关注的设计决策（如果有）

继续下一步？还是有问题想先问？👇
```

**文件加载规则（导览模式）：**
- 每步只加载 **1 个文件**，读取范围最多 **150 行**
- 如果文件超长，优先读：函数签名 + 类定义 + 注释，跳过具体实现细节
- 对于超过 300 行的文件，用 Grep 先找关键结构（类名、函数名），再用 offset/limit 定向读取
- 已在路径中读过的文件**不重复加载**，直接引用已有知识

#### 3.3 问答模式（自由提问 或 导览中途提问）

用户提问时，按以下决策树处理：

```
用户问题
   │
   ▼
能否基于已加载的文件（context 中已有）回答？
   │
   ├─ YES → 直接回答，引用具体文件路径和行号
   │
   └─ NO → 需要加载新文件
              │
              ▼
         Step 1: Grep 搜索（定位相关文件，不读内容）
              │
              ▼
         Step 2: 告知用户"需要读 X 文件来回答这个问题"
              │
              ▼
         Step 3: 读目标文件的相关片段（用 offset+limit 精准定位）
                 原则：最多加载 2 个新文件，每文件最多 100 行
              │
              ▼
         Step 4: 回答，并将该文件加入"已探索列表"
```

**回答质量标准：**
- 引用具体文件路径 + 行号（如 `src/services/UserService.ts:45`）
- 用类比解释复杂概念（"这个 middleware 就像餐厅里的门卫..."）
- 每次回答后，主动提示 1 个"下一步可以看"的关联文件
- 避免过度术语，遇到专业词汇先解释再用

#### 3.4 状态持久化（每次读取新文件后执行）

**每当读取一个新的源码文件后**，立即更新状态文件：

**更新 `.repo-context/PROGRESS.md`**：
```markdown
## 已探索文件

### src/controllers/UserController.ts
- 探索时间: 2026-03-14 16:45
- 所在路径: 路径 A，第 2 步
- 一句话作用: HTTP 请求的入口控制器，解析参数后转发给 UserService

## 学习路径状态

- 路径 A — HTTP 请求流程: 进行中（2/4）
  - ✅ src/routes/api.ts
  - ✅ src/controllers/UserController.ts  ← 当前位置
  - ⬜ src/services/UserService.ts
  - ⬜ src/repositories/UserRepo.ts
```

**更新 `.repo-context/KNOWLEDGE.md`**（追加新文件的理解摘要）：
```markdown
### src/controllers/UserController.ts
- **status**: ✅ current（analyzed at commit a3f7c21）
- **角色**: HTTP 层，接收请求、校验参数、调用 Service
- **核心方法**:
  - `getUser(id)` → 调用 UserService.findById
  - `createUser(body)` → 校验 zod schema → 调用 UserService.create
- **依赖**: UserService（下游）, authMiddleware（上游守卫）
- **设计特点**: 控制器不含业务逻辑，只做参数转换
```

> 每个条目的 `status` 字段由 Phase 0 的变更检测自动维护：
> - `✅ current` — 与当前 HEAD 一致
> - `⚠️ stale (commit d9e2b84 改动了此文件)` — 需要在下次访问时更新

**每当 Q&A 产生重要洞察后**，追加到 `.repo-context/NOTES.md`：
```markdown
### Q: 中间件的执行顺序是怎么决定的？
**时间**: 2026-03-14 16:50 | **触发文件**: src/middleware/
**结论**: Express 中间件按注册顺序执行。这个项目的顺序是：
  cors → rateLimit → authCheck → routeHandler
**关键文件**: src/app.ts:34-41（中间件注册位置）
```

#### 3.5 探索状态展示

在每次回答末尾，显示当前的探索状态（简洁一行）：

```
📊 已探索: 5个文件 | 路径 A 进度: 2/4 | 状态已保存至 .repo-context/
```

#### 3.6 会话恢复流程（Phase 0 选择 R 时）

加载三个状态文件后，直接恢复到中断位置：

1. 读取 `PROGRESS.md`：得知上次路径、步骤、已探索文件列表
2. 读取 `KNOWLEDGE.md`：恢复对已探索文件的完整理解（**不重新读源码**）
3. 读取 `NOTES.md`：恢复 Q&A 上下文
4. 向用户确认："上次你在学习路径 A 的第 2 步，我们从 `UserController.ts` 继续？"
5. 继续执行，**跳过 Phase 1 的全量分析**

> **核心价值**：三个状态文件通常合计 < 500 行，恢复整个上下文只需极少 token，
> 远优于重新运行 Phase 1 的 5 Agent 并行分析。

#### 3.7 深挖触发词

当用户说以下词语时，主动加载更多细节：
- "展开说说" / "详细解释" / "show me the code" → 加载当前话题文件的更多行
- "这个函数具体怎么实现的" → 用 Grep 定位函数，精准读取该函数体
- "有没有测试" → Grep 搜索对应的 `*.test.*` 文件，读前 60 行
- "有没有类似的地方" → Grep 搜索相同模式，列出文件路径（不全读）

---

## .repo-context 目录说明

`.repo-context/` 是 understand-repo skill 的**持久化工作记忆**，存储在被分析项目的根目录。

| 文件 | 内容 | 更新时机 |
|------|------|---------|
| `PROGRESS.md` | 当前路径、步骤、已探索文件列表 | 每次读取新文件后 |
| `KNOWLEDGE.md` | 每个已读文件的理解摘要 | 每次读取新文件后 |
| `NOTES.md` | Q&A 洞察、关键结论 | 每次问答产生重要发现后 |
| `UNDERSTANDING.md` | Phase 2 生成的全局概览 | Phase 2 完成时一次性写入 |

**关于版本控制**：在分析开始时，如果项目有 `.gitignore`，建议（但不强制）追加：
```
# AI 学习辅助文件
.repo-context/
UNDERSTANDING.md
```

---

## 分析质量标准

- **入口文件必须找到**：如果没找到，明确说"未找到明确入口，可能原因是..."
- **架构模式必须有依据**：不能凭感觉猜，要引用具体目录结构证明
- **依赖解释要说"为什么"**：不只是说"这是什么库"，要说"项目用这个库做了什么"
- **本地运行步骤要可执行**：步骤必须是真实可操作的，不能有假命令
- **硬件门槛必须明确说明**：如果项目有 GPU/大内存/专有硬件依赖，必须在"本地可运行性评估"中标注
- **学习路径要有优先级**：必须明确"先看什么，再看什么"

---

## 示例调用

```
/understand-repo .
/understand-repo ~/projects/some-cloned-repo
帮我理解一下这个项目 /path/to/repo
这个项目是怎么工作的？
```
