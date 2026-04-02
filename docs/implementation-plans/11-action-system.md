# P4.1 Action 系统

> 目标：通过自然语言或 UI 按钮触发操作（标记标签、生成简报等）。

---

## 依赖

- P1 全部完成
- P3 NL 查询完成

## 任务清单

### Task 1: Action 框架

```python
# backend/app/services/action_engine.py
from abc import ABC, abstractmethod
from typing import Any
from ..models.object import Object
from ..models.link import Link
from ..models.source import ActionLog
from sqlalchemy.ext.asyncio import AsyncSession
import uuid

class Action(ABC):
    """Action 基类"""
    name: str
    display_name: str
    description: str

    def __init__(self, db: AsyncSession):
        self.db = db

    @abstractmethod
    async def execute(self, params: dict) -> dict:
        """执行 Action，返回结果"""
        pass

    async def log(self, trigger_type: str, input_params: dict, output_result: dict, status: str = "success"):
        """记录操作日志"""
        log = ActionLog(
            action_name=self.name,
            trigger_type=trigger_type,
            input_params=input_params,
            output_result=output_result,
            status=status,
        )
        self.db.add(log)
        await self.db.commit()


class TagObjectAction(Action):
    """标记/打标签"""
    name = "tag_object"
    display_name = "标记对象"
    description = "给对象添加标签"

    async def execute(self, params: dict) -> dict:
        object_id = uuid.UUID(params["object_id"])
        tag = params["tag"]

        result = await self.db.execute(
            select(Object).where(Object.id == object_id)
        )
        obj = result.scalar_one_or_none()
        if not obj:
            raise ValueError(f"Object {object_id} not found")

        # 添加标签到 properties
        tags = obj.properties.get("tags", [])
        if tag not in tags:
            tags.append(tag)
            obj.properties["tags"] = tags
            await self.db.commit()

        await self.log("nl", params, {"tags": tags})
        return {"object_id": str(object_id), "tags": tags}


class GenerateBriefAction(Action):
    """生成关系简报"""
    name = "generate_brief"
    display_name = "生成关系简报"
    description = "选一组人物，LLM生成分析文档"

    async def execute(self, params: dict) -> dict:
        object_ids = [uuid.UUID(oid) for oid in params["object_ids"]]

        # 获取所有对象及其关系
        objects_data = []
        for oid in object_ids:
            obj = await self.db.execute(select(Object).where(Object.id == oid))
            obj = obj.scalar_one_or_none()
            if obj:
                objects_data.append({
                    "name": obj.name,
                    "properties": obj.properties,
                })

        # 获取对象间关系
        links = await self._get_links_between(object_ids)

        # LLM 生成简报
        from .llm import call_claude
        brief = await call_claude(
            system="你是一个政治分析专家。根据提供的人物和关系数据，生成一份简洁的关系分析简报。",
            prompt=f"人物数据：{json.dumps(objects_data, ensure_ascii=False)}\n关系数据：{json.dumps(links, ensure_ascii=False)}\n\n请生成关系分析简报（500字以内）。"
        )

        await self.log("nl", params, {"brief_length": len(brief)})
        return {"brief": brief, "objects": len(objects_data), "links": len(links)}


class RefreshDataAction(Action):
    """触发数据刷新"""
    name = "refresh_data"
    display_name = "刷新数据"
    description = "派遣 Agent 重新爬取该对象的最新数据"

    async def execute(self, params: dict) -> dict:
        object_id = uuid.UUID(params["object_id"])
        # 触发 Agent 采集任务
        from ..tasks.agent_tasks import collect_data_task
        task = collect_data_task.delay(
            f"刷新对象 {object_id} 的最新数据"
        )
        return {"task_id": task.id, "status": "started"}


# Action 注册表
ACTION_REGISTRY = {
    "tag_object": TagObjectAction,
    "generate_brief": GenerateBriefAction,
    "refresh_data": RefreshDataAction,
}

def get_action(name: str, db: AsyncSession) -> Action:
    cls = ACTION_REGISTRY.get(name)
    if not cls:
        raise ValueError(f"Unknown action: {name}")
    return cls(db)
```

---

### Task 2: Action NL 触发

```python
# 添加到 backend/app/services/nl_interface.py

# 在 query_parser_v1.yaml 的 few_shots 中增加 Action 识别：
# "把蓬佩奥标记为对华鹰派" → {"type": "action", "action": "tag_object", "params": {"object_name": "蓬佩奥", "tag": "对华鹰派"}}
# "生成特朗普和马斯克的关系简报" → {"type": "action", "action": "generate_brief", "params": {"object_names": ["特朗普", "马斯克"]}}

async def _execute_action(self, query: dict) -> dict:
    """执行 Action"""
    action_name = query.get("action")
    params = query.get("params", {})

    # 将名称解析为 ID
    if "object_name" in params:
        objs = await self.engine.get_objects(search=params["object_name"], limit=1)
        if objs:
            params["object_id"] = str(objs[0].id)
    if "object_names" in params:
        ids = []
        for name in params["object_names"]:
            objs = await self.engine.get_objects(search=name, limit=1)
            if objs:
                ids.append(str(objs[0].id))
        params["object_ids"] = ids

    action = get_action(action_name, self.db)
    result = await action.execute(params)
    return result
```

---

### Task 3: Action API

```python
# backend/app/api/action.py

@router.post("/actions/trigger")
async def trigger_action(
    action_name: str,
    params: dict,
    trigger_type: str = "ui",
    db: AsyncSession = Depends(get_db),
):
    """触发 Action"""
    action = get_action(action_name, db)
    result = await action.execute(params)
    return result

@router.get("/actions")
async def list_actions():
    """列出所有可用 Action"""
    return [
        {"name": name, "display_name": cls.display_name, "description": cls.description}
        for name, cls in ACTION_REGISTRY.items()
    ]
```

---

## 完成标准

```bash
# 列出可用 Actions
curl http://localhost:8000/api/v1/actions

# 标记对象
curl -X POST "http://localhost:8000/api/v1/actions/trigger?action_name=tag_object" \
  -d '{"params": {"object_id": "<uuid>", "tag": "鹰派"}}'

# 自然语言触发
curl -X POST "http://localhost:8000/api/v1/query?question=把蓬佩奥标记为对华鹰派"
# → 自动识别为 Action 并执行

# 生成简报
curl -X POST "http://localhost:8000/api/v1/actions/trigger?action_name=generate_brief" \
  -d '{"params": {"object_ids": ["<uuid1>", "<uuid2>"]}}'
```

## 预计产出时间

1-2天
