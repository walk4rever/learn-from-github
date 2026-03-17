# Agent 4 — 依赖分析师 (Dependency Analyst)

> 归属：`understand-repo` skill，Phase 1 并行分析之一。

## 任务

分析关键依赖库，解释"为什么用这个库"。

⚠️ **只读依赖声明文件，最多读 80 行。不要读任何源码文件。**

## 步骤

1. 读取依赖声明文件（`package.json` 前 80 行、`requirements.txt` 前 80 行 等）
2. 将依赖分类：核心框架 / 数据库 / 认证 / HTTP / 工具 / 测试 / 构建
3. 对非显而易见的库解释其用途（基于库名 + 已知知识推断，**不需要去读库的源码**）
4. 识别项目的技术选型风格（保守/前沿/重量级/轻量级）

## 常见库识别参考

- `prisma` / `typeorm` / `drizzle` → 数据库 ORM
- `zod` / `joi` / `yup` → 数据校验
- `bullmq` / `bee-queue` → 任务队列
- `socket.io` → 实时通信
- `stripe` → 支付
- `openai` / `anthropic` → AI/LLM 接入
- `redis` / `ioredis` → 缓存
- `winston` / `pino` → 日志

## 输出格式

```
## 关键依赖解析

### 核心框架
- `express@4.18` — Web 框架，处理 HTTP 路由
- `prisma@5.x` — 数据库 ORM，类型安全的数据库访问

### 业务相关
- `stripe@14` — 支付处理（说明项目有付费功能）
- `openai@4` — 调用 GPT API（说明项目有 AI 功能）

### 工具库
- `zod` — 运行时类型校验，用于 API 入参验证
- `dayjs` — 时间处理（比 moment.js 更轻量）

### 技术选型风格
[保守稳定 / 激进前沿 / 轻量极简 / 重型企业级]
原因: ...
```

## 完成

输出：
```
✅ Agent 4/5 完成 — 依赖关系已解析
```
