# Swan KB · AI 知识中台全栈（从后端底座到 RAG 服务）

> 本仓库数据为合成演示数据，与任何真实企业经营数据无关；文档只描述方法论与工程实现。

> 面向连锁门店知识管理场景的 AI 知识中台工程实现（语料为合成演示数据）。
> 本仓库把 **后端服务底座** 与 **检索增强生成（RAG）服务** 收敛为单一代码库，
> 呈现"从 CRUD 到 AI 架构"的完整能力证据链。

本仓库呈现「后端底座 + RAG 服务」的工程实现与方法论：知识维度用于演示知识建模；RAG 架构已用端到端链路验证。
语料为合成演示数据，公开文本只描述方法论与工程实现。

---

## 仓库结构

```
swan_kb/
├── backend-store/        # 后端最小底座（FastAPI + SQLModel + Alembic + Docker）
│   └── README.md         # 模块详情：CRUD、分页、统一错误、配置分离、API Key 鉴权、迁移
├── rag-service/          # RAG 知识问答服务（FastAPI + 混合检索 + 聚合层 + 评测闭环）
│   └── README.md         # 模块详情：数据护栏、wiki 聚合层、双路混合检索、接地测试
└── README.md             # 本文件（总览）
```

- **backend-store/** —— 后端最小项目（纯 CRUD + 鉴权 + 迁移 + Docker），是 RAG 服务的工程底座。
- **rag-service/** —— 在底座之上叠加 AI 应用架构：RAG 问答、数据真相源、自愈索引、评测闭环。

---

## 快速开始

### 1. 后端底座（backend-store）

```bash
cd backend-store
python -m venv venv && source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements-dev.txt
alembic upgrade head
uvicorn app.main:app --reload
```

### 2. RAG 服务（rag-service）

```bash
cd rag-service
pip install -r requirements-dev.txt
pytest tests/ -q                    # 全程 Mock，离线可跑（含数据接地测试）
python seed_corpus.py               # 灌入默认语料
uvicorn app.main:app --reload       # 服务启动；/docs 交互式文档
```

> 默认语料为**合成演示数据**（可公开）。接入私有知识库见下文「与私有知识库的关系」。

---

## 评测与可靠性（不是玄学）

- **评测闭环**：golden-set 评测（检索命中率 + 忠实度），离线 Mock 可跑，接入真实 LLM 后度量生成忠实度。
- **数据护栏**：`store_facts.json` / `business_facts.json` 唯一真相源 + 源—索引自愈合（启动比对 corpus 哈希）+ 接地测试（`test_grounding.py` / `test_business_grounding.py`），pytest 不通过即阻断合并。
- **可靠性文档**：失败模式（FMEA）、兜底路径、可恢复性、可观测性见各模块 `docs/reliability.md`。
- **效果评测报告**：逐题明细 + Badcase 分析 + 成本/延迟估算见 `rag-service/docs/eval-report.md`。

---

## 与私有知识库的关系

本仓库是可公开的**工程实现与方法说明**。私有知识库（含业务数据）**不进入本仓库**：

- 私有知识库是本地数据源（`./corpus/` 形态），通过环境变量 `RAG_CORPUS_DIR` 指向即可让本服务对私有知识问答。
- 仓库内 `data/corpus/` 为合成演示语料，保证公开可复现、无业务数据泄漏。
- 私有数据的更新、索引重建、评测门禁由使用方运维体系承载（单一真相源 + 备份轮换 + 自动评测），本仓库不涉及。

---

## 许可

仓库内知识内容为合成数据，与任何真实企业经营数据无关。