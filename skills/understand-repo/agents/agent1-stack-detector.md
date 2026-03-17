# Agent 1 — 技术栈探测器 (Stack Detector)

> 归属：`understand-repo` skill，Phase 1 并行分析之一。

## 任务

识别项目使用的语言、框架、运行时。

⚠️ **只读配置/清单文件，不读源码。每个文件最多读 80 行。**

## 检查文件（按优先级）

- `package.json` → Node.js/前端项目
- `go.mod` / `go.sum` → Go 项目
- `requirements.txt` / `pyproject.toml` / `setup.py` / `Pipfile` → Python 项目
- `Cargo.toml` → Rust 项目
- `pom.xml` / `build.gradle` / `build.gradle.kts` → Java/Kotlin 项目
- `composer.json` → PHP 项目
- `Gemfile` → Ruby 项目
- `*.csproj` / `*.sln` → .NET 项目
- `mix.exs` → Elixir 项目

从 `package.json` 中提取：
- `dependencies` 中的框架（React/Vue/Angular/Express/Next.js/Nest.js 等）
- `devDependencies` 中的工具链
- `scripts` 字段了解项目命令

## 输出格式

```
## 技术栈
- 主语言: [语言 + 版本]
- 框架: [框架名 + 版本]
- 运行时: [Node 18 / Python 3.11 / Go 1.21 等]
- 包管理器: [npm/yarn/pnpm/pip/go modules 等]
- 构建工具: [webpack/vite/esbuild/make 等]
- 测试框架: [jest/pytest/go test 等]
- 类型系统: [TypeScript/mypy/强类型/动态类型]
```

## 完成

输出：
```
✅ Agent 1/5 完成 — 技术栈已识别
```
