# P4.2 Function 系统

> 目标：纯计算函数，不修改数据，返回结果（路径查找、影响力排序等）。

---

## 依赖

- P1 全部完成
- P3.1 图谱 API（路径查找）

## 任务清单

### Task 1: Function 框架

```python
# backend/app/services/function_engine.py
from abc import ABC, abstractmethod
from typing import Any
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, text, func
from ..models.object import Object
from ..models.link import Link
import uuid

class Function(ABC):
    """Function 基类"""
    name: str
    display_name: str
    description: str
    input_schema: dict
    output_schema: dict

    def __init__(self, db: AsyncSession):
        self.db = db

    @abstractmethod
    async def execute(self, params: dict) -> Any:
        pass


class FindPathsFunction(Function):
    """关系路径查找"""
    name = "find_paths"
    display_name = "查找关系路径"
    description = "查找两个实体之间的所有路径（≤N跳）"
    input_schema = {
        "from_object_id": "UUID",
        "to_object_id": "UUID",
        "max_hops": "int (default 3)"
    }
    output_schema = {"paths": "list[Path]"}

    async def execute(self, params: dict) -> dict:
        from_id = uuid.UUID(params["from_object_id"])
        to_id = uuid.UUID(params["to_object_id"])
        max_hops = params.get("max_hops", 3)

        # 使用递归 CTE
        query = text("""
            WITH RECURSIVE path_search AS (
                SELECT
                    source_object_id, target_object_id, id as link_id,
                    ARRAY[source_object_id, target_object_id] as path,
                    ARRAY[id] as link_path, 1 as depth
                FROM links WHERE source_object_id = :from_id AND status = 'active'

                UNION ALL

                SELECT l.source_object_id, l.target_object_id, l.id,
                    ps.path || l.target_object_id,
                    ps.link_path || l.id, ps.depth + 1
                FROM links l, path_search ps
                WHERE (l.source_object_id = ps.target_object_id
                       OR l.target_object_id = ps.source_object_id)
                  AND l.target_object_id != ALL(ps.path)
                  AND l.status = 'active'
                  AND ps.depth < :max_hops
            )
            SELECT path, link_path, depth FROM path_search
            WHERE target_object_id = :to_id ORDER BY depth LIMIT 10
        """)

        result = await self.db.execute(query, {
            "from_id": from_id, "to_id": to_id, "max_hops": max_hops
        })

        paths = []
        for row in result:
            # 获取路径中每个节点的名称
            node_names = {}
            for nid in row[0]:
                obj = await self.db.execute(select(Object.name).where(Object.id == nid))
                name = obj.scalar()
                node_names[str(nid)] = name

            paths.append({
                "node_ids": [str(n) for n in row[0]],
                "node_names": node_names,
                "hops": row[2],
            })

        return {"paths": paths, "total": len(paths)}


class InfluenceRankingFunction(Function):
    """影响力排序"""
    name = "influence_ranking"
    display_name = "影响力排序"
    description = "按关系数量/质量排序人物影响力"
    input_schema = {"object_type": "string", "top_n": "int (default 10)"}
    output_schema = {"ranking": "list[{object_id, name, score}]"}

    async def execute(self, params: dict) -> dict:
        top_n = params.get("top_n", 10)

        # 简单影响力评分：关系数量 * 权重
        query = text("""
            SELECT
                o.id, o.name,
                COUNT(DISTINCT l.id) as link_count,
                COUNT(DISTINCT CASE WHEN l.source_object_id = o.id THEN l.id END) as outgoing,
                COUNT(DISTINCT CASE WHEN l.target_object_id = o.id THEN l.id END) as incoming
            FROM objects o
            LEFT JOIN links l ON (l.source_object_id = o.id OR l.target_object_id = o.id)
                AND l.status = 'active'
            WHERE o.status = 'active'
            GROUP BY o.id, o.name
            ORDER BY link_count DESC
            LIMIT :top_n
        """)

        result = await self.db.execute(query, {"top_n": top_n})

        ranking = []
        for row in result:
            ranking.append({
                "object_id": str(row[0]),
                "name": row[1],
                "link_count": row[2],
                "outgoing": row[3],
                "incoming": row[4],
                "score": row[2],  # MVP 简单用关系数作为分数
            })

        return {"ranking": ranking}


class TypeStatisticsFunction(Function):
    """类型统计"""
    name = "type_statistics"
    display_name = "类型统计"
    description = "统计各类型的对象数量和关系数量"

    async def execute(self, params: dict) -> dict:
        query = text("""
            SELECT ot.name, ot.display_name, COUNT(o.id) as count
            FROM object_types ot
            LEFT JOIN objects o ON o.object_type_id = ot.id AND o.status = 'active'
            GROUP BY ot.id, ot.name, ot.display_name
        """)
        result = await self.db.execute(query)

        stats = []
        for row in result:
            stats.append({
                "type_name": row[0],
                "display_name": row[1],
                "object_count": row[2],
            })

        # 关系统计
        link_count = await self.db.execute(
            select(func.count()).select_from(Link).where(Link.status == "active")
        )

        return {"types": stats, "total_links": link_count.scalar()}


# Function 注册表
FUNCTION_REGISTRY = {
    "find_paths": FindPathsFunction,
    "influence_ranking": InfluenceRankingFunction,
    "type_statistics": TypeStatisticsFunction,
}
```

---

### Task 2: Function API

```python
# backend/app/api/function.py

@router.post("/functions/execute")
async def execute_function(
    function_name: str,
    params: dict,
    db: AsyncSession = Depends(get_db),
):
    """执行 Function"""
    cls = FUNCTION_REGISTRY.get(function_name)
    if not cls:
        raise HTTPException(400, f"Unknown function: {function_name}")
    fn = cls(db)
    result = await fn.execute(params)
    return result

@router.get("/functions")
async def list_functions():
    """列出所有可用 Function"""
    return [
        {"name": name, "display_name": cls.display_name,
         "description": cls.description, "input_schema": cls.input_schema}
        for name, cls in FUNCTION_REGISTRY.items()
    ]
```

---

### Task 3: Function NL 触发

在 query_parser 中增加 Function 识别：
- "特朗普圈子里影响力最大的5个人" → `{"type": "function", "function": "influence_ranking", "params": {"top_n": 5}}`
- "特朗普和蓬佩奥之间的关系路径" → `{"type": "function", "function": "find_paths", "params": {"from": "特朗普", "to": "蓬佩奥"}}`

---

## 完成标准

```bash
# 影响力排序
curl -X POST "http://localhost:8000/api/v1/functions/execute?function_name=influence_ranking" \
  -d '{"params": {"top_n": 5}}'
# → {"ranking": [{"name": "特朗普", "link_count": 25, "score": 25}, ...]}

# 自然语言触发
curl -X POST "http://localhost:8000/api/v1/query?question=特朗普圈子里影响力最大的5个人"

# 路径查找
curl -X POST "http://localhost:8000/api/v1/query?question=特朗普和马斯克之间的关系路径"
```

## P4 阶段里程碑

> **Action & Function 可用：** 能通过自然语言触发"给这个人打标签""生成简报"；能查"影响力最大的人"和"两人之间的关系路径"。

## 预计产出时间

1-2天
