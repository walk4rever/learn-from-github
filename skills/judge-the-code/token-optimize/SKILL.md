---
name: token-optimize
description: >-
  分析目标项目中的 LLM 和 Agent 交互点，预估 Token 消耗并发现"钱包黑洞"和"注意力污染"隐患。
  提供从模型降级、上下文裁剪到启用缓存等具体的 Token 优化建议。
  TRIGGER when: 用户想优化大模型 API 消耗、检查 Token 成本、"找找哪里浪费了 token"或想降低大模型延迟时。
origin: judge-the-code
---

# Token Economist — 提示词狙击手

> 寻找 AI 时代的独有隐患：无意义的 Token 燃烧与导致幻觉的垃圾上下文。

## 核心理念

大模型调用不应是盲目的黑盒。把整个文件或未截断的历史记录丢进 Prompt，会导致：
1. **钱包出血**：无意义的 Input Token 累积。
2. **中间丢失（Lost in the middle）**：过长的无关上下文稀释模型的注意力，引发幻觉。
3. **延迟激增**：首字返回时间（TTFB）被超大 Input 拖慢。

这个 skill 会扫描整个代码库，找出所有的 LLM 交互点，推演并优化其 Token 消耗模式。

## 调用方式

```
/token-optimize .
/token-optimize /path/to/project
```

---

## Token 使用原则

> **核心策略：纯静态追踪，找到调用点和变量来源。**

1. **Grep 定位**：通过特征词（如 `openai`, `anthropic`, `messages.create`, `invoke`, `langchain`）快速定位大模型 API 调用点。
2. **数据流回溯**：针对找到的每个调用点，向上追踪传入 `messages` 或 `prompt` 的变量。重点关注数组的 `map`/`join`、文件读取 `readFile` 等可能无限膨胀的操作。
3. **严格审视**：不放过任何一个隐式的大文本注入。

---

> 📌 输出语言跟用户对话语言走（中文用户→中文报告，英文用户→英文报告）

## 执行流程

### Phase 0：输入验证

1. 运行 bash 检查路径：

   ```bash
   TARGET="{用户提供的路径}"  # 实际替换时必须加双引号，如 TARGET="/path/to/project"

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
   - **`EMPTY`** → 停止：`❌ 请提供项目路径，例如：/token-optimize .`
   - **`NOT_FOUND`** → 停止：`❌ 路径 {path} 不存在，请检查路径是否正确。`
   - **`IS_FILE`** → 停止：`❌ {path} 是一个文件，请提供项目根目录路径。`
   - **`OK`** → 继续执行

2. 确定 SKILL_DIR：
   ```bash
   find ~/.agents/skills ~/.claude/skills -name "SKILL.md" -path "*/token-optimize/*" 2>/dev/null \
     | head -1 | xargs dirname
   ```
   记为 `{SKILL_DIR}`。

---

### Phase 1：寻找 LLM 交互点 (Discovery)

使用 **Grep 工具**（而非 bash grep/rg）查找代码库中的 LLM 触点。
搜索特征包含但不限于：
- 核心 SDK 调用：`openai.`, `anthropic.messages`, `gemini`, `bedrock`
- 框架封装：`langchain`, `llamaindex`, `dify`, `ai_agent`
- 提示词相关：`system_prompt`, `PromptTemplate`
- 网络请求：寻找向 `api.openai.com`, `api.anthropic.com` 发送的 HTTP 请求。

使用 Grep 工具，pattern 示例：
```
(chat\.completions|messages\.create|langchain|llamaindex|SystemMessage|api\.openai\.com|api\.anthropic\.com)
```
glob 设为 `**/*`，path 设为 `{TARGET}`，排除 `node_modules`。

---

### Phase 2：深度追踪与推演 (Estimation & Risks)

对找出的核心交互点，读取其周围的上下文代码（用 `sed` 或 `read` 工具，切忌读取整个大文件）。
进行针对性分析：

1. **钱包黑洞**：
   - 数组/列表插值：`messages.map(...)` 或 `history.join()` 是否有长度限制？
   - 循环调用：LLM 调用是否被包裹在 `for/map` 循环中（N+1 LLM 调用反模式）？
   - 杀鸡用牛刀：是否对简单的格式化、短文本分类也使用了 `GPT-4o` 或 `Opus` 等旗舰模型？

2. **注意力污染**：
   - 裸奔大文本：是否将未过滤的 `readFile` 直接塞给模型？
   - Prompt Injection 风险：用户输入是否与系统指令混在一个字符串中，没有使用边界符隔离？

3. **缓存丢失**：
   - 长篇 System Prompt（如几百上千字）是否被正确加上了 `ephemeral` 或对应缓存标签？

---

### Phase 3：综合输出

将分析结果整合，生成完整的 `.judge-the-code/token-optimize.md` 报告。报告结构如下：

```markdown
# [项目名] — Token 优化诊断报告

> 生成时间: [date] | 分析工具: token-optimize skill

## 📊 交互点全局盘点

本项目共发现 **[N]** 个与 LLM 交互的核心触点。

| 交互场景 | 所在位置 | 调用的模型 | Token 膨胀风险 |
|---------|---------|-----------|--------------|
| [如: 用户问答] | `src/...:行号` | [模型名] | 🔴 高 / 🟡 中 / 🟢 低 |

---

## 🔍 深度分析与优化建议

### 1. 场景：[场景名称]
- **📍 位置**: `[文件路径]` (行 XX)
- **⚙️ 交互逻辑**: [简要描述输入内容和目的]
- **📈 Token 预估**: [评估 Input 和 Output 的规模，推演最坏情况下的 Token 爆炸点]
- **⚠️ 隐患**: [说明如果数据变大，会不会有 OOM、截断或超高费用]
- **💡 优化建议**:
  1. **[标签] [具体动作]**：[例如：**[高优/降本] 降级为 gpt-4o-mini**：由于仅做标题生成，无需复杂推理。]
  2. **[标签] [具体动作]**：[例如：**[高优/防幻觉] 实施滑动窗口**：只保留最近 10 轮对话历史，或使用 tiktoken 限制。]

[补充其他场景...]
```

保存到`.judge-the-code/token-optimize.md`。

**保存完成后，使用 bash 检查并执行 dashboard：**

```bash
if [ -x "{SKILL_DIR}/bin/view" ]; then
  "{SKILL_DIR}/bin/view" .
else
  echo "SKIP_VIEW"
fi
```

- 输出 `SKIP_VIEW`：跳过 dashboard，提示用户"如需可视化报告，请先运行 `/demon-hunter` 安装 view 工具"。
- 否则：dashboard 已生成并在浏览器打开。

**执行完成后，输出：**
```
📄 .judge-the-code/token-optimize.md 已生成
📊 Dashboard 已更新并在浏览器中打开。

进入互动模式，你可以继续问：
- "帮我重构一下 [某个场景] 的 Prompt"
- "如何给 [某个接口] 加上 Prompt Caching"
- "推荐一个更便宜的平替模型"
```

---

## 分析质量标准

- **绝不无病呻吟**：如果代码已经做了滑动窗口截断，要给予肯定，而不是盲目挑刺。
- **具体的替代方案**：说"换个便宜模型"不够，要明确指出换成哪个模型（如 claude-3-haiku）。
- **必须提供具体代码位置**：不能只说"某个文件里调用了 LLM"，必须提供路径和行号。
