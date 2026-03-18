# TODO.md - judge-the-code 路线图

> 更新时间: 2026-03-16
> 使命: 帮助人类在 AI 时代保持对代码的 Judgment 和 Taste

---

## 愿景

AI 写代码,人类判断代码。`judge-the-code` 是人类保持判断力的基础设施。

四层能力:
1. **结构层** - 快速建立代码库全局认知(`code-explore`,已有)
2. **欣赏层** - 提取设计哲学与关键决策(`design-lens`,已有)
3. **判断层** - 发现安全漏洞、性能隐患、设计陷阱(`demon-hunter`,已有)
4. **经济层** - 发现 Token 浪费与隐患(`token-optimize`,已有)

---

## 执行计划

### P1 - 本周(code-explore 可视化升级)

> 目标:让结构层的输出更有力,为后续判断层提供更好的上下文

#### 1. Mermaid 架构图
- **What**: 在 Architecture Mapper (Agent 2) 中生成真实模块依赖的 Mermaid 图,替换现有文本目录树
- **Why**: 结构可视化是判断力的基础--看不清结构,谈不上判断设计好坏
- **How**:
  - Step 1: Glob 获取目录结构(现有逻辑保留)
  - Step 2: 识别 index/barrel 文件,**顶层优先**(根目录 → src/ → 各一级子目录),取满 5 个为止
  - Step 3: Grep 每个 index 文件的 `^import` 前 30 行,提取 `from "..."` 路径
  - Step 4: 推断模块依赖关系
  - Step 5: 生成 Mermaid `graph TD`,**超过 15 个节点自动折叠为 subgraph**(目录级展示)
  - Step 6: 每个节点/subgraph 附一行职责说明文字
- **Mermaid 模板**(固定格式):
  ```
  graph TD
    subgraph routes["Routes 层"]
      R[api.ts - HTTP 路由入口]
    end
    subgraph controllers["Controllers 层"]
      C[UserController - 解析参数]
    end
    routes --> controllers
  ```
- **节点折叠规则**: 同目录下 >3 个文件时合并为 subgraph,subgraph 名用目录名
- **局限性说明**(输出中必须包含): `> 注:依赖关系基于 index 文件 import 推断,不代表完整调用图`
- **验收**: UNDERSTANDING.md 包含可渲染的 Mermaid 图,替代原 ASCII 目录树
- **Effort**: M

#### 2. 数据流动图
- **What**: 在 Entry Point Tracer (Agent 3) 中生成入口到第一层调用的时序图
- **Why**: 一个请求怎么走,是判断架构合理性的起点
- **How**:
  - 读入口文件前 60 行(现有逻辑保留)
  - 识别第一层调用,**不递归追踪深层调用**
  - 生成 Mermaid `sequenceDiagram`,参与者限制为入口 + 第一层模块
- **Mermaid 模板**(固定格式):
  ```
  sequenceDiagram
    participant U as User
    participant R as routes/api.ts
    participant C as UserController
    U->>R: HTTP Request
    R->>C: route handler
    C-->>R: response
    R-->>U: HTTP Response
  ```
- **局限性说明**(输出中必须包含): `> 注:时序图仅覆盖入口到第一层调用,不代表完整请求链路`
- **fallback**: 若入口文件无明显调用链,输出纯文字说明,不强行生成图
- **验收**: UNDERSTANDING.md 包含 sequenceDiagram,并有局限性说明
- **Effort**: M

#### 3. 输入验证与友好提示
- **What**: Phase 0 增加输入验证,路径异常时给出清晰提示
- **Why**: 基本体验保障
- **How**:
  - 路径为空 → 提示用法示例
  - 路径不存在 → 明确报错
  - 路径是文件而非目录 → 提示需要目录路径
- **验收**: 三种异常输入各有清晰的中文提示
- **Effort**: S

---

### P2 - 下月(design-lens)

#### 4. `design-lens` Skill 设计与实现
- **What**: 新 skill,提取代码库的设计哲学、关键决策和值得学习的地方
- **Why**: 这是 judge-the-code 的核心差异化--现在没有任何工具做这件事
- **How**:
  - 4 个并行 Agent:命名与抽象哲学 / 错误处理取向 / 测试与质量信仰 / 架构决策考古
  - 决策评价标签:🔮 精妙 / ✅ 合理 / ⚠️ 存疑 / ❌ 反模式
  - 输出:`PHILOSOPHY.md`(设计决策清单 + 隐含原则 + 值得偷走的模式)
  - 复用 `UNDERSTANDING.md`(如果已运行 code-explore)节省 token
- **验收**: 对已知项目(如 Express、FastAPI)能输出有洞察的设计决策分析
- **Effort**: L
- **Status**: ✅ 完成(v0.3.0)

#### 5. 拆分多文件 skill 结构
- **What**: 将 code-explore 的单一 800+ 行 SKILL.md 拆分为多文件
- **Why**: 加入新 Agent 后文件会更长,维护困难
- **How**:
  - 先调研 Claude Code 是否支持 skill 内多文件引用
  - 若支持:每个 Agent 拆为独立文件,主 SKILL.md 只做调度
  - 若不支持:加内部导航注释提升可读性
- **前置条件**: P1 完成后再启动
- **Effort**: M

#### 6. 进度指示
- **What**: "Agent 3/5 完成,预计还需 2 分钟"
- **Why**: 用户不知道要等多久
- **Effort**: S

---

### P3 - 待定(demon-hunter)

#### 7. `demon-hunter` 设计与实现 ✅ 完成(v0.6.x)
- **What**: Skill + 工具混合,发现安全漏洞、性能隐患、技术债、设计陷阱
- **Why**: AI 时代最迫切的需求--AI 写的代码能跑,但可能埋雷;纯 LLM 扫漏洞不可信,需要真实工具支撑
- **架构**:
  ```
  Go CLI (tool层)                    Skill (Claude层)
  ─────────────────────────────────────────────────────
  调用 semgrep → 安全漏洞             解读扫描结果
  调用 npm audit / pip-audit → 依赖漏洞  结合项目上下文判断优先级
  调用 trivy → 容器/依赖漏洞          解释"为什么危险"
  输出结构化 JSON                     给出具体修复建议
                                      识别设计层面的 demons
                                      (哲学破坏、隐性耦合、陷阱 API)
  ```
- **Demons 类型**:
  - 安全漏洞(注入、越权、未校验输入)← 工具层检测
  - 已知 CVE 依赖漏洞 ← 工具层检测
  - 性能时间炸弹(N+1、无索引大表查询)← Skill 检测
  - 隐性耦合(表面解耦,实际全局状态依赖)← Skill 检测
  - 哲学破坏(违背项目既有设计原则)← Skill 检测(依赖 design-lens 输出)
  - 陷阱 API(容易用错且用错不报错)← Skill 检测
  - 债务炸弹(牵一发动全身的脆弱区域)← Skill 检测
- **分发**: `go install github.com/judge-the-code/demon-hunter@latest`
- **Effort**: XL
- **Depends on**: design-lens 完成 + Go CLI 设计完成

### P4 - 进行中(code-explore 确定性混合架构升级)

#### 8. code-explore 混合架构升级(syft + scc)
- **What**: 为 Agent 2(架构分析)和 Agent 4(依赖分析)引入确定性底层工具,从"纯 LLM 盲猜"升级为"工具收集 + LLM 提炼"的混合架构。
- **Why**: 依赖分析和代码物理分布是 LLM 结构性做不好的两件事--读 `package.json` 前 80 行会截断依赖,Glob 目录名无法量化代码规模。确定性工具可以 100% 解决这两个痛点。
- **决定引入的工具(仅 2 个,总计 ~35MB)**:
  - `syft` (Go, ~30MB): 全球最权威的 SBOM 生成器,Docker 官方底层采用。一键穿透所有子目录和 lockfile,输出完美依赖清单 JSON。替代 Agent 4 的 grep 猜测。
  - `scc` (Go, ~2MB): 世界上最快的代码行数统计器。精确吐出每个目录的真实代码行数,为 Agent 2 画架构图提供量化物理依据。
- **决定不引入的工具(经 review 否决)**:
  - `enry`: 语言识别靠读配置文件已足够,精确百分比对用户来说过度精确。
  - `ast-grep`: 入口追踪靠文件名 + grep 成功率已很高,为每种语言维护 AST 规则的成本远超收益。
  - `yq`: 用途太窄(仅提 YAML 几个字段),grep 足够。
- **How**:
  - Step 1: 编写 `code-explore/setup` 脚本(复用 demon-hunter 模式),下载 `syft` + `scc` 至 `code-explore/bin/`。
  - Step 2: 改造 Agent 2 和 Agent 4 的执行流--先调用工具生成 JSON,再让 LLM 基于确定数据产出报告。
  - Step 3: Agent 1, 3, 5 保持现有 Prompt 模式不变。
  - Step 4: 验证后根据真实反馈决定是否扩展更多工具。
- **Effort**: M
- **Status**: 🚧 规划完成,待实施 (见 `docs/tool-selection-for-code-explore.md`)

---

## 项目结构(最终形态)

```
judge-the-code/
├── skills/
│   └── judge-the-code/          # 单目录安装,cp -r 到 ~/.agents/skills/
│       ├── SKILL.md             # 顶层索引,说明整体工作流
│       ├── code-explore/     # Skill,纯 markdown
│       │   ├── SKILL.md
│       │   └── agents/          # 各维度 Agent 规格(多文件结构)
│       ├── design-lens/ # Skill,纯 markdown
│       │   ├── SKILL.md
│       │   └── agents/
│       └── demon-hunter/        # Skill,调用 Go CLI
│           └── SKILL.md
├── cmd/
│   └── demon-hunter/            # Go CLI,调用 semgrep/trivy/audit
│       └── main.go
├── README.md
└── TODO.md
```

---

## 设计决策记录

### 2026-03-16: 项目重命名
- **决策**: `learn-from-github` → `judge-the-code`
- **理由**: 使命从"学习"升级为"判断",名字要反映这个转变
- **影响**: README、TODO、skill 描述全部更新

### 2026-03-16: 三层架构
- **决策**: 结构层(code-explore)+ 欣赏层(design-lens)+ 判断层(demon-hunter)
- **理由**: 判断力建立在理解之上,理解建立在结构之上,不能跳步

### 2026-03-16: 陪伴式优于文档式
- **决策**: UNDERSTANDING.md 是副产品,陪伴式探索是主产品
- **理由**: 用户真正需要的是"有人陪着判断",而不是"一份报告"

### 2026-03-16: demon-hunter 采用混合架构
- **决策**: demon-hunter = Go CLI(工具层)+ Skill(解读层)
- **理由**: 纯 LLM 扫安全漏洞不可信(漏报、误报);semgrep/trivy 等工具有 CVE 数据库支撑,结果确定性强;Go 单二进制分发,`go install` 一行搞定
- **分工**: 工具找问题(确定性),Claude 解释问题(语义性)

### 2026-03-16: 保持独立发展
- **决策**: 不与 everything-claude-code 合并,保持独立仓库
- **理由**: 独立品牌,定位清晰
- **缓解**: 保持接口兼容,未来可迁移

### 2026-03-18: 四层工作流 + 新增 token-optimize
- **决策**: 将原有的三层工作流扩充为四层，新增"经济层" `token-optimize`。
- **理由**: AI 项目的"隐患"不仅是安全（CVE/漏洞），无意义的 Token 燃烧（钱包黑洞）和冗余上下文导致的注意力污染（引发幻觉）同样是致命缺陷。

### 2026-03-18: code-explore 混合架构升级 — 精准投放而非全面替换
- **决策**: 仅为 Agent 2（架构）和 Agent 4（依赖）引入确定性工具 `scc` + `syft`。Agent 1, 3, 5 保持纯 Prompt 模式。
- **理由**: 经过逐一审视 5 个候选工具后，结论是**只在 LLM 结构性做不好的环节引入工具**。依赖分析（读 80 行会截断）和代码物理分布（Glob 无法量化）是真痛点；而语言识别（读配置即可）、入口追踪（文件名+grep 够用）、环境解析（语义理解是 LLM 强项）不需要工具。
- **否决的工具**: `enry`（收益不足）、`ast-grep`（维护成本过高）、`yq`（用途太窄）。
- **核心原则**: 用真实反馈驱动扩展，不提前过度工程化。验证 `syft` + `scc` 跑通后，再根据实际出错情况决定是否引入更多工具。

---

## 成功指标

| 指标 | 当前 | 目标 | 测量方式 |
|------|------|------|---------|
| 可视化覆盖率 | 0% | 100% | UNDERSTANDING.md 含 Mermaid 图比例 |
| Skill 数量 | 1 | 3 | 代码库统计 |
| demon-hunter 准确率 | - | >80% | 人工评估:发现的问题是否真实存在 |
