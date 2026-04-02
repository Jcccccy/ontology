# P5.3 本体模板 & 演示就绪

> 目标：将"政治人物"本体固化为可复用模板，设计 3 分钟演示路径。

---

## 依赖

- P5.1 数据充实完成
- P5.2 体验优化完成

## 任务清单

### Task 1: 本体模板导出

```python
# backend/scripts/export_template.py
"""
将当前的本体定义导出为可复用的模板 JSON 文件。
未来可以导入到新领域中快速启动。
"""

TEMPLATE = {
    "name": "political_figures",
    "display_name": "政治人物关系网络",
    "version": "1.0",
    "description": "用于追踪政治人物、政府部门及其关系的本体模板",
    "object_types": [
        {
            "name": "person",
            "display_name": "政治人物",
            "schema_definition": {
                "properties": {
                    "name": {"type": "string", "required": True, "display": "姓名"},
                    "english_name": {"type": "string", "display": "英文名"},
                    "title": {"type": "string", "display": "职位/头衔"},
                    "party": {"type": "string", "display": "党派",
                              "enum": ["Democratic", "Republican", "Independent"]},
                    "birth_date": {"type": "date", "display": "出生日期"},
                    "education": {"type": "string", "display": "教育背景"},
                    "background": {"type": "text", "display": "背景描述"},
                    "political_stance": {"type": "string", "display": "政治立场"},
                    "tags": {"type": "array", "display": "标签"},
                }
            }
        },
        {
            "name": "government_org",
            "display_name": "政府部门",
            "schema_definition": {
                "properties": {
                    "name": {"type": "string", "required": True, "display": "名称"},
                    "english_name": {"type": "string", "display": "英文名"},
                    "function": {"type": "text", "display": "职能描述"},
                    "level": {"type": "string", "display": "层级",
                              "enum": ["federal", "state", "local"]},
                }
            }
        },
    ],
    "link_types": [
        {
            "name": "appointed_to",
            "display_name": "任命",
            "source_type": "person",
            "target_type": "government_org",
            "is_directional": True,
            "schema_definition": {
                "properties": {
                    "position": {"type": "string", "display": "担任职位"},
                    "appointed_date": {"type": "date", "display": "任命日期"},
                    "status": {"type": "string", "display": "状态",
                               "enum": ["active", "former", "nominated"]},
                }
            }
        },
        {
            "name": "political_ally",
            "display_name": "政治盟友",
            "source_type": "person",
            "target_type": "person",
            "is_directional": False,
        },
        {
            "name": "political_rival",
            "display_name": "政治对手",
            "source_type": "person",
            "target_type": "person",
            "is_directional": False,
        },
        {
            "name": "family",
            "display_name": "家族关系",
            "source_type": "person",
            "target_type": "person",
            "is_directional": False,
            "schema_definition": {
                "properties": {
                    "relation": {"type": "string", "display": "关系",
                                 "enum": ["配偶", "父子", "父女", "兄妹", "其他"]},
                }
            }
        },
        {
            "name": "business_partner",
            "display_name": "商业伙伴",
            "source_type": "person",
            "target_type": "person",
            "is_directional": False,
        },
    ]
}
```

---

### Task 2: 模板导入 API

```python
# backend/app/api/template.py

@router.post("/templates/import")
async def import_template(template: dict, db: AsyncSession = Depends(get_db)):
    """导入本体模板"""
    engine = OntologyEngine(db)
    results = {"object_types": [], "link_types": []}

    for ot in template.get("object_types", []):
        created = await engine.create_object_type(ObjectTypeCreate(**ot))
        results["object_types"].append({"name": ot["name"], "id": str(created.id)})

    # 需要先创建所有 Object Type 才能创建 Link Type
    for lt in template.get("link_types", []):
        source = await engine._get_type_by_name(lt["source_type"])
        target = await engine._get_type_by_name(lt["target_type"])
        created = await engine.create_link_type(LinkTypeCreate(
            name=lt["name"],
            display_name=lt.get("display_name"),
            source_object_type_id=source.id,
            target_object_type_id=target.id,
            is_directional=lt.get("is_directional", True),
            schema_definition=lt.get("schema_definition"),
        ))
        results["link_types"].append({"name": lt["name"], "id": str(created.id)})

    return results

@router.get("/templates/export")
async def export_template(db: AsyncSession = Depends(get_db)):
    """导出当前本体为模板"""
    # 读取当前所有 object_types 和 link_types，组装为模板格式
    ...
```

---

### Task 3: 演示脚本

```
3 分钟演示路径：

[0:00 - 0:30] 开场
  "这是一个数据自治平台的原型演示。
   核心能力：用自然语言告诉系统你需要什么数据，AI Agent 自动采集和整理。"

[0:30 - 1:30] 亮点：Agent 自动采集
  输入："帮我收集特朗普第二任期的核心团队成员和他们的关系"
  → 观众看到 Agent 规划 → 爬取 → 提取 → 候选入库的全过程
  → 审核队列一键确认

[1:30 - 2:00] 数据浏览
  → 人物列表页浏览
  → 点击特朗普，看详情（属性 + 来源信息）
  → 关系图谱展示

[2:00 - 2:40] 自然语言查询
  → "共和党的鹰派人物有哪些？"
  → "特朗普和马斯克之间是什么关系？"
  → "圈子里影响力最大的5个人是谁？"

[2:40 - 3:00] Action 演示
  → "生成一份特朗普核心圈子关系简报"
  → 展示生成的简报

[3:00] 总结
  "从一句话到完整的关系网络，数据采集、整理、关联全自动。
   这个架构可以复用到任何领域：竞争情报、供应链、学术网络等。"
```

---

## 完成标准

- [ ] 本体模板 JSON 文件可导出
- [ ] 导入模板到空库后能立即开始使用
- [ ] 3 分钟演示路径跑通，无报错
- [ ] 演示数据覆盖 ≥ 50 个人物，关系完整

## P5 阶段里程碑

> **演示就绪：** 3 分钟演示流程顺畅，可以展示给领导和潜在用户。

## 预计产出时间

1-2天
