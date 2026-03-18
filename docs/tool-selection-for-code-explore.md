# code-explore 工具化改造决策 (混合架构)

> 决策时间: 2026-03-18
> 适用组件: code-explore

---

## 背景与痛点

当前的 `code-explore` 包含 5 个并行 Agent。它们的核心机制是通过提示词指导大模型使用 `grep`、`glob` 和读取前 80 行等方式去**猜测**项目的语言、架构和入口。

**当前纯 Prompt 模式的关键缺陷：**
1. **依赖分析不可靠**：在 Monorepo、多层嵌套、lockfile 复杂的项目中，大模型读取 `package.json` 前 80 行极易截断关键信息、遗漏子包依赖。
2. **架构判断缺乏物理依据**：大模型用 Glob 拿到目录名后凭"直觉"猜模块职责，无法量化"哪个目录包含多少行真实业务逻辑"，导致架构图主次不分。

**演进方向：混合架构 (Hybrid Architecture)**
像 `demon-hunter` 猎杀安全漏洞一样，`code-explore` 应走向 **"工具主导收集，LLM 主导提炼"**。但必须**精准投放**——只在 LLM 真正做不好的环节引入工具，而非全面替换。

---

## 工具选型决策

### 核心原则
1. **只引入 LLM 真正做不好的工具**：如果当前 Agent 用 grep/glob 已经够用（如找入口文件、读配置），就不引入工具。
2. **零侵入与零依赖**：与 `demon-hunter` 一致，必须是单文件静态二进制，不污染用户的 PATH。
3. **体积可控**：总下载量控制在 ~50MB 以内（demon-hunter 已经 ~150MB，不能再膨胀）。

### ✅ 决定引入的工具（第一批）

| 替换对象 | 选定工具 | 语言 | 二进制体积 | 业界地位 | 核心能力 |
|---------|---------|------|----------|---------|---------|
| **Agent 4** (依赖解析) | [**syft**](https://github.com/anchore/syft) | Go | ~30MB | 全球最权威的 SBOM 生成器，Docker 官方 `docker sbom` 底层调用 | 一键穿透所有子目录和 lockfile，输出完美的依赖清单 JSON |
| **Agent 2** (架构分析) | [**scc**](https://github.com/boyter/scc) | Go | ~2MB | 世界上最快的代码行数与物理结构统计器 | 精确吐出每个目录的真实代码行数，为架构图提供量化物理依据 |

**为什么这两个收益最大：**
- **syft**：依赖分析是 LLM 最容易出错的环节。读 `package.json` 前 80 行可能截断 `dependencies`，完全看不到 `package-lock.json` 或子包的依赖。Syft 解决的是一个**LLM 结构性做不到**的问题。
- **scc**：架构图的质量取决于"知道哪里是重点"。当前 Agent 只看目录名猜职责，无法区分一个 500 行的工具目录和一个 20,000 行的核心业务目录。scc 提供的量化数据让 LLM 画图时能分清主次。

### ❌ 决定不引入的工具（及理由）

| 候选工具 | 原计划用途 | 否决理由 |
|---------|-----------|---------|
| **enry** (语言识别) | 替代 Agent 1 读 `package.json` 猜语言 | **收益不足**。读 `package.json` / `go.mod` 已经能准确识别技术栈，这是 LLM 最擅长的简单推理任务之一。enry 的精确百分比（"TypeScript 85%"）对 code-explore 用户来说是过度精确。 |
| **ast-grep** (AST 搜索) | 替代 Agent 3 用 grep 找入口函数 | **复杂度过高**。入口文件通常是 `main.go`、`server.ts`、`app.py`，文件名 + 简单 grep 的成功率已经很高。ast-grep 要发挥威力需要为每种语言维护一套 AST 规则，维护成本远超收益。 |
| **yq** (YAML 解析) | 替代 Agent 5 用 grep 提 docker-compose 配置 | **用途太窄**。Agent 5 只需提取几个 YAML 字段，grep 足够。且 Agent 5 的核心工作（判断本地能否运行）本质上是语义理解任务，恰好是 LLM 最擅长的。 |

---

## 改造路径

### Phase 1: 基础设施 (`setup` 脚本)
复用 `demon-hunter/setup` 的成熟模式：
- 探测 OS/arch（macOS/Linux, arm64/amd64）
- 从 GitHub Releases 下载 `syft` + `scc` 的静态二进制
- 存入 `skills/judge-the-code/code-explore/bin/`

### Phase 2: Agent 改造
**Agent 2 (架构分析) 改造：**
- 启动时先执行 `scc -f json {TARGET}`
- 将 JSON 结果注入 Agent 的 Prompt
- Agent 不再盲目 glob，而是基于真实代码量数据画 Mermaid 图

**Agent 4 (依赖分析) 改造：**
- 启动时先执行 `syft dir:{TARGET} -o json`
- 将依赖清单注入 Agent 的 Prompt
- Agent 不再读 `package.json` 前 80 行猜依赖，而是基于完整的 SBOM 做技术选型解读

**Agent 1, 3, 5 保持现有 Prompt 模式不变。**

### Phase 3: 验证后再决定是否扩展
在 `syft` + `scc` 跑通"工具出 JSON → LLM 翻译"的完整链路后，评估：
- 如果 Agent 3 (入口追踪) 在实际使用中频繁出错 → 再考虑引入 `ast-grep`
- 如果 Agent 1 (技术栈) 在多语言项目中表现不佳 → 再考虑引入 `enry`
- **不提前引入，用真实反馈驱动决策**

---

## 预期收益

1. **依赖分析准确率从 ~70% 提升至 ~100%**：Syft 穿透 lockfile 和子包，不再有遗漏。
2. **架构图质量显著提升**：scc 提供物理量化依据，LLM 画图有据可依。
3. **Token 成本降低**：Agent 2 和 Agent 4 不再需要读大量配置文件片段。
4. **下载体积可控**：两个工具合计 ~35MB，用户体验可接受。
5. **对低智商模型的容忍度提升**：依赖和代码分布是确定性数据，即使用小模型也能产出可靠的报告。