# P3 验证测试报告 — demon-hunter

> 测试时间: 2026-03-17
> 测试项目: longcut（Next.js 15，真实生产项目）

---

## 测试结果总览

| 测试项 | 结果 | 说明 |
|--------|------|------|
| setup 脚本执行 | ✅ 通过（修复后）| latest_version() sed bug → 改用 python3 json 解析 |
| bearer 安装 | ✅ v2.0.1 | darwin/x86_64 |
| trivy 安装 | ✅ v0.69.3 | darwin/x86_64 |
| gitleaks 安装 | ✅ v8.30.0 | darwin/x86_64 |
| find-tools 输出 | ✅ 三工具路径正确 | skill-local bin/ 优先 |
| bearer 扫描输出 | ✅ | JSON 格式 `{critical:[],high:[]}` |
| trivy 扫描输出 | ✅ | JSON 格式 `{Results:[{Vulnerabilities:[]}]}` |
| gitleaks 扫描输出 | ✅ | JSON 数组或空数组 |
| DEMONS.md 生成质量 | ✅ | 发现真实问题，低误报 |
| bin/.gitignore | ✅ | 二进制不进 git |

---

## BUG-01：setup latest_version() 解析失败（已修复）

**严重程度**: 🔴 高（setup 完全无法运行）

**现象**：
`sed 's/.*"v\?\([^"]*\)".*/\1/'` 无法正确提取 GitHub API 返回的版本号。
curl 输出为 `"tag_name": "v2.0.1",...` 多字段 JSON，sed 匹配到错误位置。

**修复**：改用 `python3 -c "import json,sys; print(json.load(sys.stdin)['tag_name'].lstrip('v'))"`

---

## BUG-02：bearer JSON 格式与预期不符（已修复）

**严重程度**: 🟡 中（Agent 1 解析会失败）

**现象**：
预期 `{"findings":[...]}` 实际为 `{"critical":[...],"high":[...]}`。

**修复**：SKILL.md 中更新 JSON 解析示例，Agent 1 规格文件中说明正确格式。

---

## 真实扫描质量评估（longcut）

### bearer 发现质量
- ✅ 1 HIGH：`lib/sanitizer.ts:55` 手动 HTML sanitization
- 误报评估：非测试文件，处于生产路径，**真实风险**
- 亮点：Claude Agent 1 进一步发现项目已引入 dompurify，修复只需一行

### trivy CVE 质量
- ✅ 5 HIGH CVE，均附 fix 版本
- Claude Agent 2 正确区分 direct（axios/next）vs transitive（jws/minimatch/glob）
- 批量修复建议可直接执行

### gitleaks
- ✅ 0 leaks，输出干净
- 空结果处理正常（返回空 JSON 数组而非报错）

### Agent 3 设计分析质量
- ✅ 10 处无超时 fetch()：真实问题，高危路径已标出
- ✅ 61 处分散 process.env：真实耦合问题
- ✅ 1 处 .then() 无 .catch()：轻微但真实

---

## 边界情况覆盖

| 场景 | 处理 |
|------|------|
| 全部工具 MISSING | ✅ 提示运行 setup，STOP |
| 部分工具 MISSING | ✅ 跳过对应维度，继续扫描 |
| 非 git 仓库（gitleaks）| ✅ 跳过 git history 扫描 |
| 网络超时（setup）| ✅ --max-time 120，报错提示重试 |
| GitHub API rate limit | ✅ 捕获错误，给出提示 |
| bearer 输出为空（无 findings）| ✅ 正常处理 |

---

## 遗留问题

| 优先级 | 问题 |
|--------|------|
| P2 | osv-scanner 未集成（计划作为 trivy 补充）|
| P3 | 工具版本更新检测（当前不检查是否有更新版本）|
| P3 | 离线环境支持（无网络时工具已下载可用，但 setup 无法运行）|

---

## 结论

P3 demon-hunter **核心功能验证通过**。发现并修复 2 个 BUG。
真实扫描对 longcut 产生了有价值的、低误报的安全和设计报告。
