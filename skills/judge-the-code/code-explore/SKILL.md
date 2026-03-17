---
name: code-explore
description: >-
  帮助普通技术人快速理解一个陌生的 GitHub 项目。通过 5 个并行分析 Agent，从技术栈、
  架构设计、入口流程、依赖意图、开发环境五个维度深度分析代码库，
  最终生成结构化的《项目理解指南》(.judge-the-code/code-explore.md)，并进入交互式答疑模式。
  TRIGGER when: 用户想理解、学习、研究一个陌生代码库，不管是 clone 下来的、
  刚接手的、还是自己写但忘了结构的。触发表达包括："帮我看看这个项目"、
  "分析这个仓库"、"这个项目怎么工作的"、"我刚拿到这个代码不知道从哪开始"、
  "这个 repo 是干嘛的"、"帮我看懂这个"、"给我讲讲这个项目的结构"。
  DO NOT TRIGGER when: 用户只想修改或调试某个具体文件，或只问单个函数的问题。
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
/code-explore <项目路径>
/code-explore /path/to/cloned/project
/code-explore .   (当前目录)
```

---

## Token 使用原则

> **核心策略：用结构推断意图，而不是读源码找答案。**
>
> ⚠️ **作用域**：以下规则**仅适用于 Phase 1**（初始分析阶段）。

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

> 📌 输出语言跟用户对话语言走（中文用户→中文报告，英文用户→英文报告）

## 执行流程

### Phase 0：确认目标 & 检查历史状态

1. **输入验证** — 运行以下 bash 命令，根据返回值决定是否继续：

   ```bash
   TARGET="{用户提供的路径}"

   if [ -z "$TARGET" ]; then
     echo "EMPTY"
   elif [ ! -e "$TARGET" ]; then
     echo "NOT_FOUND"
   elif [ -f "$TARGET" ]; then
     echo "IS_FILE"
   elif [ -d "$TARGET" ]; then
     echo "OK"
   fi
   ```

   **根据输出立即处理，异常情况必须 STOP，不得继续执行后续步骤：**

   - **`EMPTY`** → 输出后停止：
     ```
     ❌ 请提供项目路径，例如：
        /code-explore .              （当前目录）
        /code-explore ~/projects/my-repo
     ```
   - **`NOT_FOUND`** → 输出后停止：
     ```
     ❌ 路径 `{path}` 不存在，请检查路径是否正确。
     ```
   - **`IS_FILE`** → 输出后停止：
     ```
     ❌ `{path}` 是一个文件，请提供项目根目录路径。
     ```
   - **`OK`** → 继续执行步骤 2

2. **检查是否存在 `.judge-the-code/state/progress.md`**（之前会话留下的状态）：

   **如果不存在** → 直接进入 Phase 1 全量分析

   **如果存在** → 执行 **代码变更检测**：

   首先判断被分析项目是否为 git 仓库：

   ```bash
   git rev-parse HEAD 2>/dev/null
   ```

   根据结果走不同分支：

   **分支 A：是 git 仓库** → 比较 commit hash：

   - hash 与 `state/progress.md` 中 `analyzed_at_commit` 相同 → 直接恢复
   - hash 不同 → 执行 diff 分析：
     ```bash
     git diff --stat {stored_commit}..HEAD
     ```
     交叉比对变更文件 vs `state/knowledge.md` 已探索文件：

     | 类别 | 处理方式 |
     |------|---------|
     | **已探索 & 已变更** | 标记为 ⚠️ STALE，下次访问时重读 diff |
     | **未探索 & 已变更** | 仅记录，不影响现有理解 |
     | **配置文件变更** | 触发对应 Agent 局部重分析 |

   **分支 B：非 git 仓库**（zip 解压、无版本控制）→ 用时间戳判断：

   - 比较 `state/progress.md` 中的 `analyzed_at` 时间戳 vs 项目文件的最新修改时间：
     ```bash
     find . -newer .judge-the-code/state/progress.md -type f \
       ! -path './.judge-the-code/state/*' ! -path './node_modules/*' \
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

   - 对每个 `⚠️ STALE` 文件：用 `git diff {stored_commit}..HEAD -- {文件路径}` 读取 diff（不读整文件），理解"改了什么"，更新 `state/knowledge.md` 对应条目
   - 若 `package.json` 变更：单独重跑 Agent 4（依赖分析），更新 `state/knowledge.md` 的依赖部分
   - 若目录结构有新增/删除：更新 `state/progress.md` 的路径规划
   - **不重跑**未变更维度的 Agent

   > **核心原则**：用 `git diff` 而不是重读整个文件。一个 200 行文件的 diff 通常只有 20-30 行，
   > 成本降低 10 倍，且能精确表达"变化了什么"。

3. 告诉用户当前状态后，继续后续流程

---

### Phase 1：并行 5 Agent 分析

**启动前，确定 skill 目录路径**（供各 Agent 加载详细规格）：

```bash
find ~/.agents/skills ~/.claude/skills -name "SKILL.md" -path "*/code-explore/*" 2>/dev/null \
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

收集所有 5 个 Agent 的结果后，生成完整的 `.judge-the-code/code-explore.md`：

```markdown
# [项目名] — 项目理解指南

> 生成时间: [date] | 分析工具: code-explore skill

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

保存文件到`.judge-the-code/code-explore.md`。

**保存完成后，输出：**
```
📄 .judge-the-code/code-explore.md 已生成

运行 view . 打开 dashboard，或直接问我任何关于这个项目的问题。
```

同时初始化状态目录（如不存在则创建）：

```bash
mkdir -p .judge-the-code/state
```

写入 `.judge-the-code/state/progress.md`（初始状态）：

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

（空）
```

写入 `.judge-the-code/state/knowledge.md`（初始为空，后续增量追加）：

```markdown
---
last_updated: [ISO 时间戳]
files_explored: 0
---

## 文件知识库

（初始分析阶段只读了配置文件）

### 配置文件

#### package.json
- **角色**: 项目依赖声明和命令入口
- **关键依赖**: [框架], [ORM], [工具库]
- **可用命令**: dev, build, test, lint
```

写入 `.judge-the-code/state/notes.md`（记录 Q&A 洞察）：

```markdown
---
last_updated: [ISO 时间戳]
qa_count: 0
---

## 问答记录 & 关键洞察

（会话中的重要问题和结论将记录在这里）
```

---

## .judge-the-code/ 目录说明

`.judge-the-code/state/` 是 code-explore skill 的**持久化工作记忆**，存储在被分析项目的根目录。

| 文件 | 内容 | 更新时机 |
|------|------|---------|
| `state/progress.md` | 当前路径、步骤、已探索文件列表 | 每次读取新文件后 |
| `state/knowledge.md` | 每个已读文件的理解摘要 | 每次读取新文件后 |
| `state/notes.md` | Q&A 洞察、关键结论 | 每次问答产生重要发现后 |
| `.judge-the-code/code-explore.md` | Phase 2 生成的全局概览 | Phase 2 完成时一次性写入 |

**关于版本控制**：在分析开始时，如果项目有 `.gitignore`，建议（但不强制）追加：
```
# judge-the-code 分析产物
.judge-the-code/
```

---

## 分析质量标准

- **入口文件必须找到**：如果没找到，明确说"未找到明确入口，可能原因是..."
- **架构模式必须有依据**：不能凭感觉猜，要引用具体目录结构证明
- **依赖解释要说"为什么"**：不只是说"这是什么库"，要说"项目用这个库做了什么"
- **本地运行步骤要可执行**：步骤必须是真实可操作的，不能有假命令
- **硬件门槛必须明确说明**：如果项目有 GPU/大内存/专有硬件依赖，必须在"本地可运行性评估"中标注
- **学习路径建议要有优先级**：输出中必须明确"先看什么，再看什么"

---

## 示例调用

```
/code-explore .
/code-explore ~/projects/some-cloned-repo
帮我理解一下这个项目 /path/to/repo
这个项目是怎么工作的？
```
