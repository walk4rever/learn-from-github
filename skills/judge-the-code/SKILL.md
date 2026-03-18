---
name: judge-the-code
description: >-
  帮助人类在 AI 大量生成代码的时代，保持对代码的 Judgment 和 Taste。
  包含四个渐进式 skill：code-explore（建立结构认知）、
  design-lens（提炼设计哲学）、demon-hunter（发现安全漏洞与设计陷阱）、
  token-optimize（发现 Token 浪费与隐患）。
  TRIGGER when: 用户想理解、评估、学习一个代码库，或想 review AI 生成的代码。
origin: judge-the-code
version: 0.8.0
---

# judge-the-code

> AI 让代码能跑。但能跑不等于好。

这个 skill 套件帮你保持对代码的判断力。

## 四层工作流

```
code-explore  →  design-lens  →  demon-hunter  →  token-optimize
"这个项目长什么样"    "哪里设计得好，为什么"      "哪里有恶魔"        "哪里在烧钱"
     结构层                 欣赏层                   判断层                 经济层
```

### 第一步：建立结构认知

```
/code-explore .
/code-explore ~/projects/some-repo
```

5 个并行 Agent 分析技术栈、架构、入口、依赖、开发环境。
输出 `.judge-the-code/code-explore.md` + 渐进式导览模式。

### 第二步：提炼设计哲学

```
/design-lens .
```

4 个并行 Agent 分析命名风格、错误处理、测试取向、架构决策。
输出 `.judge-the-code/design-lens.md`，每条决策打标签：🔮 精妙 / ✅ 合理 / ⚠️ 存疑 / ❌ 反模式。

### 第三步：猎杀恶魔

```
/demon-hunter .
```

bearer（SAST）+ trivy（依赖 CVE + Secrets + IaC）+ gitleaks（Git history 密钥）+ Claude 语义分析。
首次使用需运行 `{SKILL_DIR}/setup` 安装工具和 dashboard（约 150MB，存入 skill 目录，不影响系统）。

### 第四步：Token 经济学诊断

```
/token-optimize .
```

纯静态分析代码中的大模型交互点，推演 Token 消耗爆炸隐患、长上下文导致的注意力污染，并给出具体的降本、提速、防幻觉建议。

---

## 技术准则

所有 Agent 在做任何分析决策时，遵循以下优先级：

```
如实 > 速度 > 节省 token
```

### 输出语言

**跟用户的对话语言走**，与项目本身的语言无关：
- 用户用中文提问 → 所有报告、注释、说明用中文
- 用户用英文提问 → 所有报告、注释、说明用英文
- 混合语言 → 以用户最后一条消息的语言为准

### 如实（Evidence-based）— 最高优先级

- 每个结论必须有**文件路径 + 行号**作为证据
- 观察不到的东西，不猜测，直接说"未发现"
- 不确定时降级表述（"可能" / "倾向于"），而不是给出确定性结论
- 宁可输出"无法判断"，也不输出无根据的观点

### 速度（Speed）— 次优先

速度指用户从发出命令到看到有用输出的**整体感知耗时**，包含三层：

- **LLM 推理**：token 少 → 推理快；并行 Agent 而非串行
- **工具执行**：grep/bash 命令本身要快；未来混合架构（demon-hunter 调用 semgrep/trivy/Go CLI 等）中，优先选择启动快、增量扫描的工具，避免全量深度扫描
- **感知延迟**：尽早输出中间进度（如进度 banner），让用户知道任务在推进，而不是沉默等待

### 节省 token（Efficiency）— 最低优先

- 使用 `grep` + `sed` 精准提取，而非 `Read` 整个文件
- 用 `wc -l` 统计数量，而非把所有内容返回再手数
- **冲突规则**：如果节省 token 会导致证据不足、结论不可靠，宁可多用 token

---

## 适用场景

- **评估一个库要不要引入** — 不只看功能，还看坑
- **Review AI 生成的代码** — 验证没有破坏设计哲学，没有埋雷
- **学习优秀项目的设计** — 带着批判性眼光，找到真正值得偷的东西
- **接手陌生代码库** — 快速建立判断力，不只是走马观花
