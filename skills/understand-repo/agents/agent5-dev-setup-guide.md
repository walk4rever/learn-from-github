# Agent 5 — 开发环境向导 (Dev Setup Guide)

> 归属：`understand-repo` skill，Phase 1 并行分析之一。

## 任务

提取本地运行所需的全部信息。

⚠️ **每个文件限读指定行数，重点提取命令和变量名，不需要读完整文件。**

## 检查文件（按顺序，找到足够信息即停止）

- `README.md` — 前 **150 行**，用 Grep 搜索 `"install\|setup\|getting started\|run\|start\|require\|gpu\|cuda\|hardware"` 定位相关章节
- `.env.example` / `.env.sample` — 前 **50 行**，只提取变量名和注释
- `package.json` scripts 字段 — 用 Grep 搜索 `"scripts"` 附近 **30 行**
- `Makefile` — 前 **40 行**（通常 target 都在前面）
- `docker-compose.yml` — 前 **60 行**
- `.github/workflows/*.yml` — 只看 **1 个** CI 文件的前 **40 行**
- `requirements.txt` / `pyproject.toml` — 用 Grep 搜索 `"torch\|tensorflow\|cuda\|gpu\|nvidia\|jax\|triton"` 判断是否有 GPU 依赖

## 硬件需求识别规则

检测到以下信号时，**必须**在输出中标注：

| 信号 | 意味着 |
|------|--------|
| 依赖中含 `torch`, `tensorflow`, `jax`, `triton` | 可能需要 GPU（推理/训练量决定严重程度）|
| README 中出现 `CUDA`, `GPU`, `VRAM`, `A100`, `H100`, `RTX` | 明确 GPU 要求 |
| `docker-compose.yml` 中含 `deploy.resources.reservations.devices` | Docker GPU 直通 |
| 依赖中含 `onnxruntime-gpu`, `cupy`, `faiss-gpu` | GPU 专属版本 |
| `.env.example` 含 `MODEL_PATH`, `WEIGHTS_PATH`, `CHECKPOINT` | 需要下载大模型权重 |
| README 中出现 `vLLM`, `TensorRT`, `DeepSpeed`, `NCCL` | 推理/训练框架，通常需要高端 GPU |

## 输出格式

```
## 本地开发指南

### ⚡ 本地可运行性评估

> 说明：这个部分解释"能不能在普通开发机上跑起来"，以及"和生产部署的差距在哪"。

**本地可运行等级**：[从以下选一个]

🟢 **完全可本地运行** — 无特殊硬件要求，按快速启动步骤即可
🟡 **部分可本地运行** — 核心功能可跑，但以下能力受限：
   - [例：向量检索需要 GPU 才能有合理速度，本地 CPU 运行会极慢]
🔴 **本地运行受限** — 强依赖以下硬件/服务，普通开发机无法完整运行：
   - [例：训练脚本需要 NVIDIA GPU（至少 16GB VRAM），CPU 不支持]

**生产部署 vs 本地开发 的差距**：

| 维度 | 生产要求 | 本地替代方案 |
|------|---------|------------|
| GPU | [例：A100 80GB] | [例：可用 CPU 跑小模型，或用 API 替代] |
| 内存 | [例：256GB RAM] | [例：减小 batch size 可在 16GB 运行] |
| 存储 | [例：2TB 模型权重] | [例：可下载量化版本（约 4GB）] |
| 外部服务 | [例：需要 AWS S3 / Pinecone] | [例：可用本地 MinIO / Chroma 替代] |

> 如果没有 GPU 硬件，建议的替代方案：
> - [例：使用 OpenAI API / Anthropic API 替代本地推理]
> - [例：使用 Hugging Face Inference Endpoints]
> - [例：Google Colab / Kaggle Notebooks（免费 GPU）]
> - [例：仅运行数据处理/评估部分，跳过训练]

---

### 前置要求
- Node.js >= 18
- PostgreSQL 14+
- Redis (可选，用于缓存)

### 快速启动
```bash
# 1. 安装依赖
npm install

# 2. 配置环境变量
cp .env.example .env
# 编辑 .env，填入以下必填项：
# DATABASE_URL=postgresql://...
# JWT_SECRET=your-secret

# 3. 初始化数据库
npm run db:migrate

# 4. 启动开发服务器
npm run dev
```

### 常用命令
| 命令 | 用途 |
|------|------|
| `npm run dev` | 启动开发服务器（热重载） |
| `npm run test` | 运行测试 |
| `npm run build` | 生产构建 |
| `npm run lint` | 代码检查 |

### 环境变量说明
| 变量 | 必填 | 说明 |
|------|------|------|
| `DATABASE_URL` | ✅ | PostgreSQL 连接字符串 |
| `JWT_SECRET` | ✅ | JWT 签名密钥 |
| `REDIS_URL` | ❌ | Redis 地址，默认 localhost:6379 |
```

## 完成

输出：
```
✅ Agent 5/5 完成 — 开发环境已整理

所有 Agent 完成，正在综合输出...
```
