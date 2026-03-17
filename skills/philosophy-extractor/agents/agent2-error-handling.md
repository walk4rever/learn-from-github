# Agent 2 — 错误处理取向分析师

> 归属：`philosophy-extractor` skill，Phase 1 并行分析之一。

## 任务

通过错误处理方式，揭示代码库对"意外"的态度。

## 步骤

1. Grep 搜索错误处理关键词（跨语言）：
   ```
   try|catch|throw|except|raise|panic|recover|Result|Either|Option|unwrap|?
   err != nil|if err|.error()|reject|Promise.reject
   ```
   定位 5-10 处典型错误处理代码，读取上下文各 **20 行**

2. 分析**错误处理风格**：

   | 风格 | 特征 | 透露的取向 |
   |------|------|-----------|
   | 快速失败 | `panic`, `assert`, `throw` 用于不可恢复错误 | 防御性强，宁可崩溃也不带病运行 |
   | 防御包裹 | 大量 `try/catch`，返回默认值 | 容错优先，可能掩盖问题 |
   | 显式传播 | `Result<T, E>`, `if err != nil` | 强制调用方处理，类型安全 |
   | 静默忽略 | `catch (e) {}`, `_ = err` | 可能的隐患，也可能是有意为之 |
   | 统一处理 | 全局 error handler，middleware 层统一捕获 | 架构意识强 |

3. 检查**错误信息质量**：
   - 错误消息是否包含上下文（`"failed to create user: email already exists"` vs `"error"`）
   - 是否有错误码体系（HTTP 状态码使用是否准确）
   - 是否有结构化错误类型（自定义 Error 类）

4. 检查**边界保护意识**：
   - Grep `null|undefined|nil|None` 检查空值处理
   - 是否有输入校验层（zod/joi/pydantic/class-validator）
   - 校验是否在入口处统一，还是散落各处

## 输出格式

```
## 错误处理取向

### 主导风格
[快速失败 / 防御包裹 / 显式传播 / 混合]
**证据**：
- `src/services/UserService.ts:45` — throw 用于不合法的业务状态（快速失败）
- `src/api/handler.ts:12` — 全局 catch 统一处理（架构意识）
**结论**：...

### 错误信息质量
[信息丰富 / 一般 / 过于简略]
**证据**：...

### 边界保护
[严格 / 一般 / 宽松]
**证据**：
- 使用 zod 在 API 层统一校验 → 边界清晰
- 但 service 层未做二次校验 → 信任上游，有一定风险
**结论**：...

### 值得关注的错误处理决策
- [具体例子 + 分析]
```

## 完成

输出：
```
✅ Agent 2/4 完成 — 错误处理取向已分析
```
