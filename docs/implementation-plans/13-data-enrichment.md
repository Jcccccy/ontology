# P5.1 数据充实

> 目标：Agent 补充爬取到 50-100 个政治人物，关系网络完善。

---

## 依赖

- P2 全部完成（Agent 采集可用）

## 任务清单

### Task 1: 批量采集脚本

```python
# backend/scripts/enrich_data.py
"""
分批次运行 Agent 采集，扩展数据到 50-100 个政治人物。

批次设计：
  批次1: 特朗普第二任期内阁成员（约15人）
  批次2: 特朗普家族成员和商业伙伴（约10人）
  批次3: 关键顾问和外部盟友（约10人）
  批次4: 民主党核心人物（约10人）
  批次5: 国际关系人物（约5人）
"""

BATCH_INTENTS = [
    "收集特朗普第二任期(2025年起)全部内阁成员的详细信息，包括国务卿、国防部长、财政部长、司法部长、商务部长等",
    "收集特朗普家族成员信息，包括梅拉尼娅、小唐纳德、伊万卡、埃里克、蒂芙尼、贾里德·库什纳",
    "收集特朗普的重要顾问和幕僚信息，包括Susie Wiles、Stephen Miller、Tom Homan等",
    "收集与特朗普关系密切的商界人物，包括马斯克、彼得·蒂尔等",
    "收集民主党核心人物信息，包括拜登、哈里斯、舒默、佩洛西",
]

async def run_enrichment():
    """运行批量采集"""
    from app.services.planner import PlannerService
    from app.services.extractor import ExtractorService
    from app.services.ontology_engine import OntologyEngine
    from app.database import async_session

    async with async_session() as db:
        engine = OntologyEngine(db)

        for i, intent in enumerate(BATCH_INTENTS):
            print(f"\n=== 批次 {i+1}/{len(BATCH_INTENTS)} ===")
            print(f"意图: {intent}")

            # 使用 Agent 采集管道
            from app.tasks.agent_tasks import _collect_data
            # 直接调用（不经过 Celery，方便调试）
            # 或者通过 API 触发

            # 方式1: 直接调用
            # result = await _collect_data(...)

            # 方式2: 通过 API 触发（推荐，走完整流程）
            import httpx
            async with httpx.AsyncClient() as client:
                resp = await client.post(
                    "http://localhost:8000/api/v1/agent/collect",
                    params={"intent": intent}
                )
                task_id = resp.json()["task_id"]
                print(f"任务已启动: {task_id}")

                # 等待完成
                import asyncio
                while True:
                    await asyncio.sleep(5)
                    resp = await client.get(f"http://localhost:8000/api/v1/agent/tasks/{task_id}")
                    status = resp.json()
                    print(f"  状态: {status['state']}")
                    if status["state"] in ("DONE", "FAILED"):
                        break

            print(f"批次 {i+1} 完成")

        # 打印统计
        types_stats = await engine.get_objects()
        print(f"\n总计: {len(types_stats)} 个对象")
```

---

### Task 2: 关系补充

```python
# 在数据充实后，运行关系发现任务
# 使用 LLM 分析已有对象，发现潜在关系

BATCH_RELATIONSHIP_INTENTS = [
    "帮我梳理特朗普内阁成员之间的政治关系",
    "帮我找出特朗普家族成员和商界的关系",
    "帮我梳理马斯克和特朗普政府中其他人的关系",
]
```

---

### Task 3: 标签体系

```python
# 手动 + Action 批量打标签
TAG_RULES = [
    {"tag": "鹰派", "objects": ["Marco Rubio", "Mike Waltz", "Pete Hegseth"]},
    {"tag": "科技界", "objects": ["Elon Musk", "David Sacks"]},
    {"tag": "MAGA核心", "objects": ["Donald Trump", "Stephen Miller", "Tom Homan"]},
    {"tag": "家族", "objects": ["Melania Trump", "Donald Trump Jr.", "Ivanka Trump"]},
]
```

---

## 完成标准

- [ ] 对象总数 ≥ 50 个
- [ ] 关系总数 ≥ 100 条
- [ ] 至少覆盖 5 种关系类型
- [ ] 每个人物有 ≥ 3 个属性填充
- [ ] 标签体系覆盖 ≥ 80% 的人物

## 预计产出时间

2天（主要是 Agent 运行时间，人工审核也需要时间）
