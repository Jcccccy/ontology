# P3.2 自然语言查询

> 目标：用户用自然语言提问，系统返回正确结果。

---

## 依赖

- P1 数据库 + API
- P2 有足够的数据可供查询

## 任务清单

### Task 1: 查询解析 Prompt

```yaml
# backend/prompts/nl_interface/query_parser_v1.yaml
name: query_parser
version: v1
description: 将自然语言查询转换为内部查询指令
model: claude-sonnet-4-6
temperature: 0.1
system: |
  你是一个查询解析器。将用户的自然语言问题转换为结构化查询指令。

  支持的查询类型：
  1. filter — 按属性筛选对象
     → {"type": "filter", "object_type": "...", "conditions": [{"field": "...", "op": "eq/gt/lt/contains", "value": "..."}]}
  2. graph — 图遍历查询（找关系）
     → {"type": "graph", "from": "实体名", "to": "实体名"（可选）, "max_hops": N, "relation_type": "..."}
  3. aggregate — 聚合统计
     → {"type": "aggregate", "object_type": "...", "group_by": "...", "metric": "count/sum/avg"}
  4. search — 语义搜索（模糊查找）
     → {"type": "search", "query": "...", "object_type": "..."}

  无法理解的问题返回 {"type": "unknown", "original": "..."}

user_template: |
  已有本体类型：
  {{object_types}}

  已有关系类型：
  {{link_types}}

  用户问题：{{question}}

  请输出查询指令 JSON：

few_shots:
  - input: "共和党的内阁成员有哪些？"
    output: '{"type": "filter", "object_type": "person", "conditions": [{"field": "party", "op": "eq", "value": "Republican"}]}'

  - input: "谁和马斯克有直接关系？"
    output: '{"type": "graph", "from": "马斯克", "max_hops": 1}'

  - input: "特朗普和蓬佩奥之间有什么关系？"
    output: '{"type": "graph", "from": "特朗普", "to": "蓬佩奥", "max_hops": 3}'

  - input: "鹰派人物有哪些？"
    output: '{"type": "filter", "object_type": "person", "conditions": [{"field": "tags", "op": "contains", "value": "鹰派"}]}'
```

---

### Task 2: 结果摘要 Prompt

```yaml
# backend/prompts/nl_interface/result_summary_v1.yaml
name: result_summary
version: v1
description: 将查询结果转换为自然语言摘要
model: claude-sonnet-4-6
temperature: 0.3
system: |
  你是一个查询结果摘要生成器。将结构化查询结果转换为简洁、准确的中文回答。

  规则：
  1. 直接回答问题，不要说"根据查询结果"
  2. 如果结果是列表，用简洁的列表格式
  3. 如果结果为空，说明未找到相关数据
  4. 如果是关系路径，用 A → B → C 的格式描述

user_template: |
  用户问题：{{question}}
  查询结果：{{result}}

  请生成中文摘要：
```

---

### Task 3: NL 查询服务

```python
# backend/app/services/nl_interface.py
from .prompt_loader import load_prompt
from .llm import call_claude_json, call_claude
from ..services.ontology_engine import OntologyEngine
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import text, select
from ..models.object import Object
from ..models.link import Link
import json

class NLInterface:
    def __init__(self, db: AsyncSession):
        self.db = db
        self.parser_prompt = load_prompt("nl_interface/query_parser_v1.yaml")
        self.summary_prompt = load_prompt("nl_interface/result_summary_v1.yaml")
        self.engine = OntologyEngine(db)

    async def query(self, question: str) -> dict:
        """自然语言查询主入口"""

        # 1. 获取本体信息
        types = await self.engine.get_object_types()
        link_types = await self.engine.get_link_types()

        types_str = json.dumps([
            {"name": t.name, "display_name": t.display_name,
             "fields": list(t.schema_definition.get("properties", {}).keys())}
            for t in types
        ], ensure_ascii=False)

        links_str = json.dumps([
            {"name": lt.name, "display_name": lt.display_name,
             "source": lt.source_object_type_id, "target": lt.target_object_type_id}
            for lt in link_types
        ], ensure_ascii=False)

        # 2. 解析查询
        parse_msg = self.parser_prompt.render(
            object_types=types_str,
            link_types=links_str,
            question=question,
        )
        query_instruction = await call_claude_json(self.parser_prompt, parse_msg)

        # 3. 执行查询
        if query_instruction["type"] == "filter":
            result = await self._execute_filter(query_instruction)
        elif query_instruction["type"] == "graph":
            result = await self._execute_graph(query_instruction)
        elif query_instruction["type"] == "search":
            result = await self._execute_search(query_instruction)
        else:
            result = {"error": "无法理解的查询", "type": "unknown"}

        # 4. 生成摘要
        summary_msg = self.summary_prompt.render(
            question=question,
            result=json.dumps(result, ensure_ascii=False)[:3000],
        )
        summary = await call_claude(self.summary_prompt, summary_msg)

        return {
            "question": question,
            "parsed_query": query_instruction,
            "result": result,
            "summary": summary,
        }

    async def _execute_filter(self, query: dict) -> list[dict]:
        """执行属性过滤查询"""
        type_name = query.get("object_type", "person")
        conditions = query.get("conditions", [])

        objects = await self.engine.get_objects(type_name=type_name)
        results = []

        for obj in objects:
            match = True
            for cond in conditions:
                field = cond["field"]
                op = cond["op"]
                value = cond["value"]
                actual = obj.properties.get(field)

                if actual is None:
                    match = False
                    break

                if op == "eq" and str(actual) != str(value):
                    match = False
                elif op == "contains" and str(value).lower() not in str(actual).lower():
                    match = False
                elif op == "gt" and float(actual) <= float(value):
                    match = False
                elif op == "lt" and float(actual) >= float(value):
                    match = False

            if match:
                results.append({
                    "id": str(obj.id),
                    "name": obj.name,
                    "properties": obj.properties,
                })

        return results

    async def _execute_graph(self, query: dict) -> dict:
        """执行图遍历查询"""
        from_name = query.get("from", "")
        to_name = query.get("to")
        max_hops = query.get("max_hops", 2)

        # 查找起始对象
        from_objs = await self.engine.get_objects(search=from_name, limit=1)
        if not from_objs:
            return {"error": f"未找到 '{from_name}'"}

        from_obj = from_objs[0]

        if to_name:
            # 路径查询：A → B
            to_objs = await self.engine.get_objects(search=to_name, limit=1)
            if not to_objs:
                return {"error": f"未找到 '{to_name}'"}
            to_obj = to_objs[0]

            # 用图谱 API 查找路径
            paths_result = await self._find_paths(from_obj.id, to_obj.id, max_hops)
            return {
                "from": {"id": str(from_obj.id), "name": from_obj.name},
                "to": {"id": str(to_obj.id), "name": to_obj.name},
                "paths": paths_result,
            }
        else:
            # 邻居查询：A 的直接关系
            links = await self.engine.get_object_links(from_obj.id)
            neighbors = []
            for link in links["outgoing"]:
                target = await self.engine.get_object(link.target_object_id)
                lt = await self.engine.get_link_type(link.link_type_id)
                neighbors.append({
                    "name": target.name,
                    "relation": lt.display_name,
                    "direction": "outgoing",
                })
            for link in links["incoming"]:
                source = await self.engine.get_object(link.source_object_id)
                lt = await self.engine.get_link_type(link.link_type_id)
                neighbors.append({
                    "name": source.name,
                    "relation": lt.display_name,
                    "direction": "incoming",
                })
            return {
                "center": {"id": str(from_obj.id), "name": from_obj.name},
                "neighbors": neighbors,
            }

    async def _execute_search(self, query: dict) -> list[dict]:
        """执行搜索（MVP 用简单的 LIKE 匹配）"""
        q = query.get("query", "")
        type_name = query.get("object_type")
        objects = await self.engine.get_objects(type_name=type_name, search=q)
        return [
            {"id": str(o.id), "name": o.name, "properties": o.properties}
            for o in objects
        ]
```

---

### Task 4: 查询 API

```python
# backend/app/api/query.py

@router.post("/query")
async def nl_query(question: str, db: AsyncSession = Depends(get_db)):
    """自然语言查询"""
    interface = NLInterface(db)
    result = await interface.query(question)
    return result
```

---

### Task 5: 前端对话界面增强

```tsx
// 在 P2.4 的 chat 页面基础上增加查询功能
// Agent 完成采集后，同一个对话界面支持查询

// 新增消息类型：
// - "user_query" — 用户提问
// - "query_result" — 查询结果（含 summary + 对象卡片 + 迷你图谱）

// 结果展示组件：
// QueryResultCard.tsx — 展示查询摘要
// ObjectInlineCard.tsx — 内联对象卡片（可点击跳转）
// MiniGraph.tsx — 迷你关系图谱（小尺寸版本）
```

---

## 完成标准

```bash
# 属性过滤查询
curl -X POST "http://localhost:8000/api/v1/query?question=共和党的内阁成员有哪些"
# → {"summary": "以下是共和党内阁成员：\n1. 马科·卢比奥 - 国务卿\n2. 皮特·赫格塞斯 - 国防部长\n...", ...}

# 关系查询
curl -X POST "http://localhost:8000/api/v1/query?question=谁和马斯克有直接关系"
# → {"summary": "与马斯克有直接关系的人物：\n1. 特朗普 - 任命关系\n...", ...}

# 路径查询
curl -X POST "http://localhost:8000/api/v1/query?question=特朗普和蓬佩奥之间有什么关系"
# → {"summary": "特朗普 → 任命 → 蓬佩奥 为国务卿", ...}
```

## P3 阶段里程碑

> **图谱 + 问答可用：** 打开关系图谱能看到核心人物网络；问属性过滤和一跳/两跳关系类问题能返回正确答案。

## 预计产出时间

2-3天
