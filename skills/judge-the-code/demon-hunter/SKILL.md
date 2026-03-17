---
name: demon-hunter
description: >-
  发现代码库中的安全漏洞、依赖 CVE、密钥泄漏和设计层隐患。
  混合架构：bearer（SAST 数据流分析）+ trivy（依赖漏洞）+ gitleaks（密钥泄漏）
  三件套工具扫描，加 Claude 语义分析，输出 DEMONS.md 报告。
  TRIGGER when: 用户想找安全漏洞、检查依赖风险、审查 AI 生成的代码、
  或说了"帮我找漏洞"、"有没有安全问题"、"这个项目有什么隐患"。
  DO NOT TRIGGER when: 用户只想理解项目结构（用 understand-repo）
  或提炼设计哲学（用 philosophy-extractor）。
origin: judge-the-code
version: 1.0.0
---

# Demon Hunter — 恶魔猎手

> 工具找问题（确定性）+ Claude 解释问题（语义性）

## 技术准则

遵循 judge-the-code 全局准则：**如实 > 速度 > 节省 token**

- **如实**：每个 demon 必须有工具输出的 rule_id/CVE 编号或文件:行号作证据
- **速度**：工具扫描与 Claude 分析尽量并行；工具输出先过滤再传给 Claude
- **节省 token**：只传 critical/high 级别给 Claude，丢弃 low/info 噪音

## 调用方式

```
/demon-hunter .
/demon-hunter /path/to/project
```

---

## 执行流程

### Phase 0：输入验证

运行以下 bash 命令，根据返回值决定是否继续：

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

**根据输出立即处理，异常情况必须 STOP：**

- **`EMPTY`** → 停止：`❌ 请提供项目路径，例如：/demon-hunter .`
- **`NOT_FOUND`** → 停止：`❌ 路径 {path} 不存在，请检查路径是否正确。`
- **`IS_FILE`** → 停止：`❌ {path} 是一个文件，请提供项目根目录路径。`
- **`OK`** → 继续

---

### Phase 1：工具检测 & 初始化

**1. 确定 SKILL_DIR：**

```bash
find ~/.agents/skills ~/.claude/skills \
  -name "SKILL.md" -path "*/demon-hunter/*" 2>/dev/null \
  | head -1 | xargs dirname
```

记为 `{SKILL_DIR}`。

**2. 检测工具可用性：**

```bash
{SKILL_DIR}/bin/find-tools
```

解析输出，得到 `BEARER`、`TRIVY`、`GITLEAKS` 三个变量（路径或 `MISSING`）。

**3. 处理 MISSING 工具：**

- 若**全部 MISSING** → 提示：
  ```
  ⚠️  扫描工具未安装。运行以下命令安装（约 150MB，下载到 skill 目录，不影响系统）：
     {SKILL_DIR}/setup
  ```
  然后 STOP，等待用户安装后重新运行。

- 若**部分 MISSING** → 继续，跳过缺失工具对应的维度，在报告中注明。

**4. 输出启动公告：**

```
🔍 demon-hunter 启动

  ✅ bearer    — SAST 代码漏洞扫描
  ✅ trivy     — 依赖 CVE + 密钥检测
  ✅ gitleaks  — Git history 密钥扫描
  ⚠️ [tool]   — 未安装，跳过（运行 setup 安装）

目标路径: {TARGET}
预计完成时间: 1-3 分钟
```

---

### Phase 2：并行工具扫描

**同时运行以下三个扫描（能并行的尽量并行），输出过滤后的 JSON：**

**Bearer（SAST）：**
```bash
$BEARER scan {TARGET} \
  --format=json \
  --severity=critical,high \
  --quiet \
  2>/dev/null | head -c 100000
# bearer 输出格式：{"critical":[...], "high":[...]}，按 severity 分组
# head -c 100000 防止超大输出撑爆上下文
```

统计摘要（传给 Agent 1 前先提取）：
```bash
python3 -c "
import json, sys
data = json.load(sys.stdin)
c = len(data.get('critical', []))
h = len(data.get('high', []))
print(f'bearer: critical={c}, high={h}')
for sev in ['critical','high']:
    for f in data.get(sev, []):
        print(f'  [{sev}] {f[\"filename\"]}:{f[\"line_number\"]} — {f[\"id\"]}')
"
```

**Trivy（SCA + Secrets + IaC）：**
```bash
$TRIVY fs {TARGET} \
  --format=json \
  --severity=CRITICAL,HIGH \
  --scanners=vuln,secret,config \
  --quiet \
  2>/dev/null | head -c 100000
```

**Gitleaks（Git History Secrets）：**
```bash
# 仅在目标路径是 git 仓库时运行
git -C {TARGET} rev-parse HEAD 2>/dev/null && \
$GITLEAKS detect \
  --source={TARGET} \
  --report-format=json \
  --report-path=- \
  --no-banner \
  2>/dev/null | head -c 50000 \
|| echo '{"leaks":[]}'
```

**工具扫描完成后，输出：**
```
📊 工具扫描完成
  bearer:   [N] 个 critical/high findings
  trivy:    [N] 个 CVE，[M] 个 secrets
  gitleaks: [N] 个 leaks

正在启动 Claude 深度分析...
```

---

### Phase 3：并行 Claude 分析

**同时启动以下 3 个 Explore 子 Agent，将 Phase 2 的 JSON 输出传入：**

| Agent | 规格文件 | 分析维度 |
|-------|---------|---------|
| Agent 1 | `{SKILL_DIR}/agents/agent1-sast-interpreter.md` | bearer 结果解读，去误报，补上下文 |
| Agent 2 | `{SKILL_DIR}/agents/agent2-secrets-sca-interpreter.md` | trivy + gitleaks 结果解读，区分真实泄漏 |
| Agent 3 | `{SKILL_DIR}/agents/agent3-design-analyst.md` | 性能隐患、隐性耦合、陷阱 API（纯 Claude）|

---

### Phase 4：综合输出 DEMONS.md

收集 3 个 Agent 的结果，生成完整报告：

```markdown
# [项目名] — Demon Hunter 报告

> 生成时间: [date] | 工具: bearer + trivy + gitleaks + Claude

## 风险总览

| 级别 | 数量 | 类型 |
|------|------|------|
| 🔴 Critical | [N] | [漏洞类型列表] |
| 🟠 High     | [N] | [漏洞类型列表] |
| 🟡 Medium   | [N] | [漏洞类型列表] |
| 💡 设计隐患 | [N] | 工具无法检测的语义问题 |

**跳过的扫描维度**（工具未安装）：[列出，或"无"】

---

## 🔴 代码安全漏洞（SAST）
[来自 Agent 1]

## 🔑 密钥泄漏
[来自 Agent 2 — Secrets 部分]

## 📦 依赖漏洞（CVE）
[来自 Agent 2 — SCA 部分]

## ⚡ 性能时间炸弹
[来自 Agent 3]

## 🔗 隐性耦合
[来自 Agent 3]

## 💣 哲学破坏
[来自 Agent 3，若有 PHILOSOPHY.md]

## 🪤 陷阱 API
[来自 Agent 3]

---

## 修复优先级建议

### 立即处理（24小时内）
1. [Critical 安全漏洞 / 有效密钥泄漏]

### 近期处理（本迭代）
2. [High CVE / 性能炸弹]

### 计划处理
3. [设计隐患 / 技术债]

---

## 跳过的扫描维度

[若有工具未安装，说明缺失了什么覆盖，及安装方式]
```

保存到项目根目录的 `DEMONS.md`。

**保存完成后，输出：**
```
📄 DEMONS.md 已生成

进入互动模式，你可以继续问：
- "帮我展开说说 [某个漏洞]"
- "这个 CVE 怎么修"
- "有多少是误报"
```

---

## 分析质量标准

- **每个 demon 必须有证据**：工具 rule_id/CVE 编号，或文件:行号
- **区分真实威胁和误报**：测试文件/示例代码中的 finding 必须排除或降级
- **不放大也不缩小**：critical 就是 critical，不因"项目很小"而降级
- **工具缺失不等于无问题**：明确说明哪些维度因工具缺失而未扫描
