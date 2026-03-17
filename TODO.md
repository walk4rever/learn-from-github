# TODO.md — judge-the-code 路线图

> 更新时间: 2026-03-16
> 使命: 帮助人类在 AI 时代保持对代码的 Judgment 和 Taste

---

## 愿景

AI 写代码，人类判断代码。`judge-the-code` 是人类保持判断力的基础设施。

三层能力：
1. **结构层** — 快速建立代码库全局认知（`understand-repo`，已有）
2. **欣赏层** — 提取设计哲学与关键决策（`philosophy-extractor`，规划中）
3. **判断层** — 发现安全漏洞、性能隐患、设计陷阱（`demon-hunter`，规划中）

---

## 执行计划

### P1 - 本周（understand-repo 可视化升级）

> 目标：让结构层的输出更有力，为后续判断层提供更好的上下文

#### 1. Mermaid 架构图
- **What**: 在 Architecture Mapper (Agent 2) 中生成真实模块依赖的 Mermaid 图，替换现有文本目录树
- **Why**: 结构可视化是判断力的基础——看不清结构，谈不上判断设计好坏
- **How**:
  - Step 1: Glob 获取目录结构（现有逻辑保留）
  - Step 2: 识别 index/barrel 文件，**顶层优先**（根目录 → src/ → 各一级子目录），取满 5 个为止
  - Step 3: Grep 每个 index 文件的 `^import` 前 30 行，提取 `from "..."` 路径
  - Step 4: 推断模块依赖关系
  - Step 5: 生成 Mermaid `graph TD`，**超过 15 个节点自动折叠为 subgraph**（目录级展示）
  - Step 6: 每个节点/subgraph 附一行职责说明文字
- **Mermaid 模板**（固定格式）:
  ```
  graph TD
    subgraph routes["Routes 层"]
      R[api.ts — HTTP 路由入口]
    end
    subgraph controllers["Controllers 层"]
      C[UserController — 解析参数]
    end
    routes --> controllers
  ```
- **节点折叠规则**: 同目录下 >3 个文件时合并为 subgraph，subgraph 名用目录名
- **局限性说明**（输出中必须包含）: `> 注：依赖关系基于 index 文件 import 推断，不代表完整调用图`
- **验收**: UNDERSTANDING.md 包含可渲染的 Mermaid 图，替代原 ASCII 目录树
- **Effort**: M

#### 2. 数据流动图
- **What**: 在 Entry Point Tracer (Agent 3) 中生成入口到第一层调用的时序图
- **Why**: 一个请求怎么走，是判断架构合理性的起点
- **How**:
  - 读入口文件前 60 行（现有逻辑保留）
  - 识别第一层调用，**不递归追踪深层调用**
  - 生成 Mermaid `sequenceDiagram`，参与者限制为入口 + 第一层模块
- **Mermaid 模板**（固定格式）:
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
- **局限性说明**（输出中必须包含）: `> 注：时序图仅覆盖入口到第一层调用，不代表完整请求链路`
- **fallback**: 若入口文件无明显调用链，输出纯文字说明，不强行生成图
- **验收**: UNDERSTANDING.md 包含 sequenceDiagram，并有局限性说明
- **Effort**: M

#### 3. 输入验证与友好提示
- **What**: Phase 0 增加输入验证，路径异常时给出清晰提示
- **Why**: 基本体验保障
- **How**:
  - 路径为空 → 提示用法示例
  - 路径不存在 → 明确报错
  - 路径是文件而非目录 → 提示需要目录路径
- **验收**: 三种异常输入各有清晰的中文提示
- **Effort**: S

---

### P2 - 下月（philosophy-extractor）

#### 4. `philosophy-extractor` Skill 设计与实现
- **What**: 新 skill，提取代码库的设计哲学、关键决策和值得学习的地方
- **Why**: 这是 judge-the-code 的核心差异化——现在没有任何工具做这件事
- **How（待设计）**:
  - 分析命名规范、抽象层级、错误处理方式等设计选择
  - 识别"非显而易见"的决策，解释其背后的权衡
  - 对比业界常见做法，指出"这里选择了不一样的路，为什么"
  - 输出：设计决策清单 + 每个决策的评价（精妙/合理/存疑）
- **验收**: 对已知项目（如 Express、FastAPI）能输出有洞察的设计决策分析
- **Effort**: L
- **Depends on**: P1 完成，understand-repo 稳定后启动

#### 5. 拆分多文件 skill 结构
- **What**: 将 understand-repo 的单一 800+ 行 SKILL.md 拆分为多文件
- **Why**: 加入新 Agent 后文件会更长，维护困难
- **How**:
  - 先调研 Claude Code 是否支持 skill 内多文件引用
  - 若支持：每个 Agent 拆为独立文件，主 SKILL.md 只做调度
  - 若不支持：加内部导航注释提升可读性
- **前置条件**: P1 完成后再启动
- **Effort**: M

#### 6. 进度指示
- **What**: "Agent 3/5 完成，预计还需 2 分钟"
- **Why**: 用户不知道要等多久
- **Effort**: S

---

### P3 - 待定（demon-hunter）

#### 7. `demon-hunter` 设计与实现
- **What**: Skill + Go CLI 混合工具，发现安全漏洞、性能隐患、技术债、设计陷阱
- **Why**: AI 时代最迫切的需求——AI 写的代码能跑，但可能埋雷；纯 LLM 扫漏洞不可信，需要真实工具支撑
- **架构**:
  ```
  Go CLI (tool层)                    Skill (Claude层)
  ─────────────────────────────────────────────────────
  调用 semgrep → 安全漏洞             解读扫描结果
  调用 npm audit / pip-audit → 依赖漏洞  结合项目上下文判断优先级
  调用 trivy → 容器/依赖漏洞          解释"为什么危险"
  输出结构化 JSON                     给出具体修复建议
                                      识别设计层面的 demons
                                      （哲学破坏、隐性耦合、陷阱 API）
  ```
- **Demons 类型**:
  - 安全漏洞（注入、越权、未校验输入）← 工具层检测
  - 已知 CVE 依赖漏洞 ← 工具层检测
  - 性能时间炸弹（N+1、无索引大表查询）← Skill 检测
  - 隐性耦合（表面解耦，实际全局状态依赖）← Skill 检测
  - 哲学破坏（违背项目既有设计原则）← Skill 检测（依赖 philosophy-extractor 输出）
  - 陷阱 API（容易用错且用错不报错）← Skill 检测
  - 债务炸弹（牵一发动全身的脆弱区域）← Skill 检测
- **分发**: `go install github.com/judge-the-code/demon-hunter@latest`
- **Effort**: XL
- **Depends on**: philosophy-extractor 完成 + Go CLI 设计完成

---

## 项目结构（最终形态）

```
judge-the-code/
├── skills/
│   ├── understand-repo/     # Skill，纯 markdown
│   │   └── SKILL.md
│   ├── philosophy-extractor/ # Skill，纯 markdown
│   │   └── SKILL.md
│   └── demon-hunter/        # Skill，调用 Go CLI
│       └── SKILL.md
├── cmd/
│   └── demon-hunter/        # Go CLI，调用 semgrep/trivy/audit
│       └── main.go
├── README.md
└── TODO.md
```

---

## 设计决策记录

### 2026-03-16: 项目重命名
- **决策**: `learn-from-github` → `judge-the-code`
- **理由**: 使命从"学习"升级为"判断"，名字要反映这个转变
- **影响**: README、TODO、skill 描述全部更新

### 2026-03-16: 三层架构
- **决策**: 结构层（understand-repo）+ 欣赏层（philosophy-extractor）+ 判断层（demon-hunter）
- **理由**: 判断力建立在理解之上，理解建立在结构之上，不能跳步

### 2026-03-16: 陪伴式优于文档式
- **决策**: UNDERSTANDING.md 是副产品，陪伴式探索是主产品
- **理由**: 用户真正需要的是"有人陪着判断"，而不是"一份报告"

### 2026-03-16: demon-hunter 采用混合架构
- **决策**: demon-hunter = Go CLI（工具层）+ Skill（解读层）
- **理由**: 纯 LLM 扫安全漏洞不可信（漏报、误报）；semgrep/trivy 等工具有 CVE 数据库支撑，结果确定性强；Go 单二进制分发，`go install` 一行搞定
- **分工**: 工具找问题（确定性），Claude 解释问题（语义性）

### 2026-03-16: 保持独立发展
- **决策**: 不与 everything-claude-code 合并，保持独立仓库
- **理由**: 独立品牌，定位清晰
- **缓解**: 保持接口兼容，未来可迁移

---

## 成功指标

| 指标 | 当前 | 目标 | 测量方式 |
|------|------|------|---------|
| 可视化覆盖率 | 0% | 100% | UNDERSTANDING.md 含 Mermaid 图比例 |
| Skill 数量 | 1 | 3 | 代码库统计 |
| demon-hunter 准确率 | — | >80% | 人工评估：发现的问题是否真实存在 |
