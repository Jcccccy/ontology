# P2.3 本体推断器

> 目标：从提取的结构化数据中，推断出合适的 Object Type、属性定义和关系类型。

---

## 依赖

- P2.1 网页提取器（有提取结果可供分析）
- P2.2 规划器（有采集计划上下文）

## 任务清单

### Task 1: 本体推断 Prompt

```yaml
# backend/prompts/agent/ontology_infer_v1.yaml
name: ontology_inferrer
version: v1
description: 从提取的数据中推断本体结构
model: claude-sonnet-4-6
temperature: 0.2
system: |
  你是一个本体建模专家。你会收到一组从网页中提取的结构化数据，需要：

  1. 分析数据中的实体类型
  2. 为每种实体类型建议属性定义（字段名、类型、是否必填）
  3. 识别实体间可能的关系类型
  4. 尽量复用已有的本体类型，只在必要时建议新类型

  属性类型支持：string, text, number, date, enum, boolean
  返回 JSON 格式。

user_template: |
  已有本体类型：
  {{existing_types}}

  已有关系类型：
  {{existing_link_types}}

  提取的数据样本：
  {{extracted_data}}

  用户原始意图：{{intent}}

  请分析以上数据，输出本体建议：

few_shots:
  - input: |
      已有本体类型：无
      提取的数据样本：
      [
        {"name": "Marco Rubio", "title": "Secretary of State", "party": "Republican"},
        {"name": "Pete Hegseth", "title": "Secretary of Defense", "party": "Republican"},
        {"name": "Scott Bessent", "title": "Secretary of Treasury", "party": "Republican"}
      ]
      用户原始意图：收集特朗普内阁成员信息
    output: |
      ```json
      {
        "suggested_object_types": [
          {
            "name": "person",
            "display_name": "政治人物",
            "description": "政府官员和政治人物",
            "schema_definition": {
              "properties": {
                "name": {"type": "string", "required": true, "display": "姓名"},
                "english_name": {"type": "string", "display": "英文名"},
                "title": {"type": "string", "display": "职位/头衔"},
                "party": {"type": "string", "display": "党派", "enum": ["Democratic", "Republican", "Independent"]},
                "background": {"type": "text", "display": "背景描述"},
                "education": {"type": "string", "display": "教育背景"},
                "birth_date": {"type": "date", "display": "出生日期"}
              }
            }
          },
          {
            "name": "government_org",
            "display_name": "政府部门",
            "description": "联邦政府部门和机构",
            "schema_definition": {
              "properties": {
                "name": {"type": "string", "required": true, "display": "名称"},
                "function": {"type": "text", "display": "职能描述"}
              }
            }
          }
        ],
        "suggested_link_types": [
          {
            "name": "appointed_to",
            "display_name": "任命",
            "source_type": "person",
            "target_type": "government_org",
            "is_directional": true,
            "schema_definition": {
              "properties": {
                "position": {"type": "string", "display": "担任职位"},
                "status": {"type": "string", "display": "状态", "enum": ["active", "former", "nominated"]}
              }
            }
          },
          {
            "name": "political_ally",
            "display_name": "政治盟友",
            "source_type": "person",
            "target_type": "person",
            "is_directional": false
          }
        ],
        "reuse_existing_types": [],
        "notes": "建议创建 person 和 government_org 两种类型，并用 appointed_to 关系连接"
      }
      ```
```

---

### Task 2: 本体推断服务

```python
# backend/app/services/ontology_infer.py
from .prompt_loader import load_prompt
from .llm import call_claude_json
from ..services.ontology_engine import OntologyEngine
from sqlalchemy.ext.asyncio import AsyncSession
import json

class OntologyInferrer:
    def __init__(self, db: AsyncSession):
        self.db = db
        self.prompt = load_prompt("agent/ontology_infer_v1.yaml")
        self.engine = OntologyEngine(db)

    async def infer(self, extracted_data: list[dict], intent: str) -> dict:
        """从提取数据推断本体结构"""

        # 1. 获取已有类型信息
        existing_types = await self.engine.get_object_types()
        existing_link_types = await self.engine.get_link_types()

        types_str = json.dumps([
            {"name": t.name, "display_name": t.display_name,
             "properties": list(t.schema_definition.get("properties", {}).keys())}
            for t in existing_types
        ], ensure_ascii=False)

        links_str = json.dumps([
            {"name": lt.name, "display_name": lt.display_name}
            for lt in existing_link_types
        ], ensure_ascii=False)

        # 2. 准备数据样本（限制长度）
        data_sample = json.dumps(extracted_data[:10], ensure_ascii=False)
        if len(data_sample) > 10000:
            data_sample = data_sample[:10000]

        # 3. LLM 推断
        user_message = self.prompt.render(
            existing_types=types_str,
            existing_link_types=links_str,
            extracted_data=data_sample,
            intent=intent,
        )
        suggestion = await call_claude_json(self.prompt, user_message)

        return suggestion

    async def apply_suggestion(self, suggestion: dict, approved_items: list[str] = None) -> dict:
        """用户确认后，将建议的本体结构写入数据库"""

        results = {"created_types": [], "created_link_types": [], "skipped": []}
        approved = set(approved_items or [])

        # 创建 Object Types
        for ot in suggestion.get("suggested_object_types", []):
            if approved and ot["name"] not in approved:
                results["skipped"].append(ot["name"])
                continue

            try:
                created = await self.engine.create_object_type(ObjectTypeCreate(
                    name=ot["name"],
                    display_name=ot.get("display_name"),
                    description=ot.get("description"),
                    schema_definition=ot["schema_definition"],
                ))
                results["created_types"].append({"name": ot["name"], "id": str(created.id)})
            except Exception as e:
                results["skipped"].append(f"{ot['name']}: {str(e)}")

        # 创建 Link Types
        for lt in suggestion.get("suggested_link_types", []):
            if approved and lt["name"] not in approved:
                results["skipped"].append(lt["name"])
                continue

            try:
                # 查找 source/target 类型
                source_ot = await self.engine._get_type_by_name(lt["source_type"])
                target_ot = await self.engine._get_type_by_name(lt["target_type"])

                created = await self.engine.create_link_type(LinkTypeCreate(
                    name=lt["name"],
                    display_name=lt.get("display_name"),
                    source_object_type_id=source_ot.id,
                    target_object_type_id=target_ot.id,
                    is_directional=lt.get("is_directional", True),
                    schema_definition=lt.get("schema_definition"),
                ))
                results["created_link_types"].append({"name": lt["name"], "id": str(created.id)})
            except Exception as e:
                results["skipped"].append(f"{lt['name']}: {str(e)}")

        return results
```

---

### Task 3: 推断器 API

```python
# 添加到 backend/app/api/agent.py

@router.post("/infer-ontology")
async def infer_ontology(
    extracted_data: list[dict],
    intent: str,
    db: AsyncSession = Depends(get_db),
):
    """从提取数据推断本体结构"""
    inferrer = OntologyInferrer(db)
    suggestion = await inferrer.infer(extracted_data, intent)
    return suggestion

@router.post("/apply-ontology-suggestion")
async def apply_ontology_suggestion(
    suggestion: dict,
    approved_items: list[str] = None,
    db: AsyncSession = Depends(get_db),
):
    """确认后写入本体结构"""
    inferrer = OntologyInferrer(db)
    results = await inferrer.apply_suggestion(suggestion, approved_items)
    return results
```

---

## 完成标准

```bash
# 1. 提取 Wikipedia 数据
curl -X POST "http://localhost:8000/api/v1/extractor/extract?url=https://en.wikipedia.org/wiki/Second_presidency_of_Donald_Trump&object_type_name=person"
# → 返回提取的实体列表

# 2. 将提取结果交给推断器
curl -X POST http://localhost:8000/api/v1/agent/infer-ontology \
  -H "Content-Type: application/json" \
  -d '{
    "extracted_data": [<上面返回的entities>],
    "intent": "收集特朗普内阁成员信息"
  }'

# 期望返回：
{
  "suggested_object_types": [
    {"name": "person", "display_name": "政治人物", "schema_definition": {...}},
    {"name": "government_org", "display_name": "政府部门", "schema_definition": {...}}
  ],
  "suggested_link_types": [
    {"name": "appointed_to", "display_name": "任命", ...}
  ],
  "notes": "..."
}

# 3. 确认并写入
curl -X POST http://localhost:8000/api/v1/agent/apply-ontology-suggestion \
  -d '{"suggestion": <上面的结果>, "approved_items": ["person", "government_org", "appointed_to"]}'
```

## 预计产出时间

1-2天
