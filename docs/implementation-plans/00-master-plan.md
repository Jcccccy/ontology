# 数据自治平台 MVP — 实施总纲

> 基于 implementation-report.md 和 product-report.md 拆解为可执行、可测试的小任务。
> 每个子阶段独立成文件，完成后产出测试报告。

---

## 项目结构

```
ontology-platform/
├── backend/                    # Python/FastAPI
│   ├── app/
│   │   ├── main.py             # FastAPI 入口
│   │   ├── config.py           # 配置管理
│   │   ├── database.py         # DB 连接
│   │   ├── models/             # SQLAlchemy ORM 模型
│   │   │   ├── ontology.py     # Object Type / Link Type
│   │   │   ├── object.py       # 对象实体
│   │   │   ├── link.py         # 关系
│   │   │   ├── source.py       # 原始数据源
│   │   │   ├── candidate.py    # 候选实体/关系
│   │   │   └── review.py       # 审核队列
│   │   ├── api/                # API 路由
│   │   │   ├── ontology.py     # /types CRUD
│   │   │   ├── objects.py      # /objects CRUD
│   │   │   ├── links.py        # /links CRUD
│   │   │   ├── review.py       # 审核队列
│   │   │   ├── agent.py        # Agent 相关
│   │   │   └── query.py        # 自然语言查询
│   │   ├── services/           # 业务逻辑
│   │   │   ├── ontology_engine.py
│   │   │   ├── agent_engine.py
│   │   │   ├── extractor.py
│   │   │   ├── planner.py
│   │   │   ├── ontology_infer.py
│   │   │   └── nl_interface.py
│   │   └── schemas/            # Pydantic 模型
│   ├── prompts/                # Prompt YAML 文件
│   ├── tests/                  # 测试
│   ├── alembic/                # 数据库迁移
│   ├── requirements.txt
│   └── Dockerfile
├── frontend/                   # Next.js
│   ├── src/
│   │   ├── app/
│   │   │   ├── page.tsx        # 首页
│   │   │   ├── objects/        # 对象浏览
│   │   │   ├── objects/[id]/   # 对象详情
│   │   │   ├── review/         # 审核队列
│   │   │   ├── graph/          # 关系图谱
│   │   │   └── chat/           # 对话界面
│   │   ├── components/
│   │   └── lib/
│   └── package.json
├── docker-compose.yml          # PostgreSQL + Redis
└── README.md
```

## 阶段依赖关系

```
P1.1 DB & Models ──→ P1.2 API ──→ P1.3 前端 ──→ P1.4 审核队列
                                                      │
                                                      ▼
P2.1 网页提取器 ──→ P2.2 规划器 ──→ P2.3 推断器 ──→ P2.4 采集管道
                                                      │
                    ┌─────────────────────────────────┘
                    ▼
P3.1 关系图谱 ──→ P3.2 NL查询
                    │
                    ▼
P4.1 Action ──→ P4.2 Function
                    │
                    ▼
P5.1 数据充实 ──→ P5.2 体验优化 ──→ P5.3 本体模板
```

## 阶段文件索引

| 文件 | 阶段 | 状态 |
|------|------|------|
| [01-database-and-models.md](01-database-and-models.md) | P1.1 数据库 & 模型 | 待实施 |
| [02-ontology-engine-api.md](02-ontology-engine-api.md) | P1.2 Ontology Engine API | 待实施 |
| [03-frontend-skeleton.md](03-frontend-skeleton.md) | P1.3 前端骨架 | 待实施 |
| [04-review-queue.md](04-review-queue.md) | P1.4 审核队列 | 待实施 |
| [05-web-extractor.md](05-web-extractor.md) | P2.1 网页提取器 | 待实施 |
| [06-agent-planner.md](06-agent-planner.md) | P2.2 Agent 规划器 | 待实施 |
| [07-ontology-inferrer.md](07-ontology-inferrer.md) | P2.3 本体推断器 | 待实施 |
| [08-collection-pipeline.md](08-collection-pipeline.md) | P2.4 采集管道 | 待实施 |
| [09-graph-visualization.md](09-graph-visualization.md) | P3.1 关系图谱 | 待实施 |
| [10-nl-query.md](10-nl-query.md) | P3.2 自然语言查询 | 待实施 |
| [11-action-system.md](11-action-system.md) | P4.1 Action 系统 | 待实施 |
| [12-function-system.md](12-function-system.md) | P4.2 Function 系统 | 待实施 |
| [13-data-enrichment.md](13-data-enrichment.md) | P5.1 数据充实 | 待实施 |
| [14-experience-polish.md](14-experience-polish.md) | P5.2 体验优化 | 待实施 |
| [15-ontology-template.md](15-ontology-template.md) | P5.3 本体模板 | 待实施 |
