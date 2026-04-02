# P2.4 采集管道（完整 Agent 流程）

> 目标：一句话触发 → 规划 → 提取 → 推断 → 候选入库 → 等待审核。整个流程自动化。

---

## 依赖

- P2.1 网页提取器
- P2.2 规划器
- P2.3 本体推断器
- Redis + Celery（异步任务）

## 任务清单

### Task 1: Celery 配置

```python
# backend/app/celery_app.py
from celery import Celery
from .config import settings

celery_app = Celery(
    "ontology_agent",
    broker=settings.REDIS_URL,
    backend=settings.REDIS_URL,
)

celery_app.conf.update(
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="Asia/Shanghai",
    task_track_started=True,
    task_time_limit=600,         # 10分钟超时
    task_soft_time_limit=540,    # 9分钟软超时
)
```

---

### Task 2: Agent 任务

```python
# backend/app/tasks/agent_tasks.py
from ..celery_app import celery_app
from ..database import async_session
from ..services.planner import PlannerService
from ..services.extractor import ExtractorService
from ..services.ontology_infer import OntologyInferrer
from ..services.ontology_engine import OntologyEngine
from ..models.candidate import EntityCandidate
from ..models.review import ReviewQueue
import json

@celery_app.task(bind=True)
def collect_data_task(self, intent: str):
    """完整的 Agent 采集任务"""
    import asyncio
    return asyncio.run(_collect_data(self, intent))

async def _collect_data(task, intent: str):
    async with async_session() as db:
        try:
            # === 阶段 1: 规划 ===
            task.update_state(state="PLANNING", meta={"step": "规划采集策略"})

            planner = PlannerService(db)
            plan = await planner.plan(intent)

            task.update_state(state="PLANNED", meta={
                "step": "规划完成",
                "total_steps": len(plan.get("steps", [])),
                "plan_summary": plan.get("plan_summary", ""),
            })

            # === 阶段 2: 本体推断 ===
            # 先用第一步的数据推断/确认本体结构
            task.update_state(state="INFERING_ONTOLOGY", meta={"step": "推断本体结构"})

            all_extracted = []

            # 执行第一步提取（用于本体推断）
            first_step = plan["steps"][0] if plan.get("steps") else None
            if first_step:
                engine = OntologyEngine(db)
                type_name = first_step.get("target_type", "person")
                try:
                    ot = await engine._get_type_by_name(type_name)
                    schema_def = ot.schema_definition
                except ValueError:
                    schema_def = {"properties": {"name": {"type": "string"}}}

                extractor = ExtractorService(db)
                for url_info in first_step.get("urls", [])[:3]:  # 先取3个URL做推断
                    try:
                        results = await extractor.extract_from_url(
                            url_info["url"], type_name, schema_def
                        )
                        all_extracted.extend(results)
                    except Exception as e:
                        print(f"Extract error for {url_info['url']}: {e}")

            # 推断本体
            if all_extracted:
                inferrer = OntologyInferrer(db)
                suggestion = await inferrer.infer(all_extracted, intent)
                # 注意：这里不自动写入，等用户确认
                # 但如果已有类型匹配，直接继续
            else:
                suggestion = {}

            # === 阶段 3: 批量提取 ===
            task.update_state(state="EXTRACTING", meta={
                "step": "批量提取数据",
                "extracted": len(all_extracted),
            })

            for step in plan.get("steps", [])[1:]:  # 跳过第一步（已提取）
                type_name = step.get("target_type", "person")
                try:
                    ot = await engine._get_type_by_name(type_name)
                    schema_def = ot.schema_definition
                except ValueError:
                    continue

                for url_info in step.get("urls", []):
                    try:
                        results = await extractor.extract_from_url(
                            url_info["url"], type_name, schema_def
                        )
                        all_extracted.extend(results)

                        task.update_state(state="EXTRACTING", meta={
                            "step": f"提取: {url_info['url'][:50]}",
                            "extracted": len(all_extracted),
                        })
                    except Exception as e:
                        print(f"Extract error: {e}")

            # === 阶段 4: 数据清洗 & 候选入库 ===
            task.update_state(state="CLEANING", meta={
                "step": "数据清洗和去重",
                "total_entities": len(all_extracted),
            })

            engine = OntologyEngine(db)
            candidates_created = 0

            for entity in all_extracted:
                # 去重：检查是否已存在同名对象
                data = entity.get("data", {})
                name = data.get("name") or data.get("english_name")
                if not name:
                    continue

                existing = await engine.get_objects(search=name, limit=1)
                if existing:
                    continue  # 跳过重复

                # 获取或使用推断的类型
                type_name = "person"  # 默认，实际应根据推断结果

                # 创建候选
                candidate = EntityCandidate(
                    candidate_type="object",
                    suggested_type_id=(await engine._get_type_by_name(type_name)).id,
                    data=data,
                    source_ids={"urls": [entity.get("source_url", "")]},
                    confidence=entity.get("confidence", 0.5),
                    status="pending",
                )
                db.add(candidate)
                await db.flush()

                # 冲突检测
                conflict_id = await _detect_conflict(candidate, engine)
                if conflict_id:
                    candidate.status = "conflicted"
                    candidate.conflict_with = conflict_id
                    review_type = "conflict"
                elif candidate.confidence < 0.7:
                    review_type = "low_confidence"
                else:
                    review_type = "new_entity"

                review = ReviewQueue(
                    candidate_id=candidate.id,
                    review_type=review_type,
                    status="pending",
                )
                db.add(review)
                candidates_created += 1

            await db.commit()

            # === 完成 ===
            task.update_state(state="DONE", meta={
                "step": "采集完成",
                "candidates_created": candidates_created,
                "total_extracted": len(all_extracted),
                "ontology_suggestion": suggestion,
            })

            return {
                "status": "completed",
                "candidates_created": candidates_created,
                "total_extracted": len(all_extracted),
                "ontology_suggestion": suggestion,
            }

        except Exception as e:
            task.update_state(state="FAILED", meta={"error": str(e)})
            raise

async def _detect_conflict(candidate, engine):
    """检测候选数据与已有数据的冲突"""
    name = candidate.data.get("name")
    if not name:
        return None
    existing = await engine.get_objects(search=name, limit=1)
    if existing:
        return existing[0].id
    return None
```

---

### Task 3: Agent API（完整流程）

```python
# 添加到 backend/app/api/agent.py

import uuid
from ..tasks.agent_tasks import collect_data_task

# 存储 task_id → intent 的映射（生产环境用 Redis）
_task_registry = {}

@router.post("/collect")
async def start_collection(intent: str):
    """启动 Agent 采集任务"""
    task = collect_data_task.delay(intent)
    _task_registry[task.id] = intent
    return {
        "task_id": task.id,
        "intent": intent,
        "status": "started",
    }

@router.get("/tasks/{task_id}")
async def get_task_status(task_id: str):
    """查询采集任务状态"""
    from ..celery_app import celery_app
    result = celery_app.AsyncResult(task_id)

    response = {
        "task_id": task_id,
        "state": result.state,
    }

    if result.info:
        response["meta"] = result.info

    if result.ready():
        response["result"] = result.result

    return response

@router.post("/confirm-ontology")
async def confirm_ontology(
    task_id: str,
    approved_items: list[str],
    db: AsyncSession = Depends(get_db),
):
    """确认 Agent 推断的本体结构"""
    from ..celery_app import celery_app
    result = celery_app.AsyncResult(task_id)

    if not result.ready():
        raise HTTPException(400, "Task not completed yet")

    task_result = result.result
    suggestion = task_result.get("ontology_suggestion", {})

    inferrer = OntologyInferrer(db)
    apply_result = await inferrer.apply_suggestion(suggestion, approved_items)
    return apply_result
```

---

### Task 4: 前端 Agent 交互面板

```tsx
// frontend/src/app/chat/page.tsx
//
// Agent 交互界面（也是未来的对话查询界面）
//
// 布局：
// ┌──────────────────────────────────────────┐
// │  🤖 Agent 采集助手                        │
// ├──────────────────────────────────────────┤
// │                                          │
// │  你: 帮我收集特朗普第二任期内阁成员          │
// │                                          │
// │  Agent: 🔄 正在规划采集策略...             │
// │  Agent: ✅ 规划完成，共3步，预计45个实体    │
// │  Agent: 🔄 正在提取数据... (12/45)         │
// │                                          │
// │  Agent: 📋 本体建议                       │
// │    建议创建：人物(government_org)          │
// │    建议关系：任命(appointed_to)            │
// │    [✓ 确认本体建议]                       │
// │                                          │
// │  Agent: 🔄 数据清洗和入库...              │
// │  Agent: ✅ 采集完成！                      │
// │    提取 42 个实体 → 38 个候选入库          │
// │    → [查看审核队列] (4条待审核)            │
// │                                          │
// ├──────────────────────────────────────────┤
// │  [输入意图...                    ] [发送]  │
// └──────────────────────────────────────────┘

// 关键实现：
// 1. 轮询 GET /agent/tasks/{id} 获取状态
// 2. 根据状态展示进度（PLANNING/EXTRACTING/CLEANING/DONE）
// 3. DONE 且有 ontology_suggestion 时展示确认面板
// 4. 用户确认后调用 POST /agent/confirm-ontology
```

**轮询逻辑：**

```typescript
// frontend/src/lib/useAgentTask.ts
export function useAgentTask(taskId: string | null) {
  const [status, setStatus] = useState<any>(null);
  const [isPolling, setIsPolling] = useState(false);

  useEffect(() => {
    if (!taskId) return;
    setIsPolling(true);

    const interval = setInterval(async () => {
      const resp = await fetch(`${API_BASE}/agent/tasks/${taskId}`);
      const data = await resp.json();
      setStatus(data);

      if (data.state === 'DONE' || data.state === 'FAILED') {
        setIsPolling(false);
        clearInterval(interval);
      }
    }, 2000); // 每2秒轮询

    return () => clearInterval(interval);
  }, [taskId]);

  return { status, isPolling };
}
```

---

## 完成标准

```bash
# 完整流程测试
# 1. 启动采集
curl -X POST "http://localhost:8000/api/v1/agent/collect?intent=帮我收集特朗普第二任期的内阁成员"
# → {"task_id": "xxx", "status": "started"}

# 2. 查询状态（轮询直到 DONE）
curl http://localhost:8000/api/v1/agent/tasks/<task_id>
# → {"state": "PLANNING", "meta": {"step": "规划采集策略"}}
# → {"state": "EXTRACTING", "meta": {"extracted": 12}}
# → {"state": "DONE", "result": {"candidates_created": 38, ...}}

# 3. 确认本体建议
curl -X POST http://localhost:8000/api/v1/agent/confirm-ontology \
  -d '{"task_id": "xxx", "approved_items": ["person", "government_org", "appointed_to"]}'

# 4. 前端审核队列中看到候选数据
# 5. 确认候选后，对象列表中可见
```

## P2 阶段里程碑

完成 P2.1 ~ P2.4 后，达到第二个里程碑：

> **Agent 自动采集 → 候选入库：** 输入一句话意图，Agent 自动规划、爬取、提取、推断本体，候选数据进入审核队列，用户确认后入库。

## 回退方案

如果全自动采集不稳定，降级为：
1. 用户输入意图 → 系统返回建议 URL 列表
2. 用户确认 URL → Agent 提取
3. 用户审核入库

## 预计产出时间

3-4天
