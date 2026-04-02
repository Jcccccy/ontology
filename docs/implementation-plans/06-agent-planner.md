# P2.2 Agent 规划器

> 目标：给定用户意图，LLM 生成采集计划（目标 URL 列表 + 采集策略）。

---

## 依赖

- P2.1 网页提取器
- Claude API

## 任务清单

### Task 1: 规划器 Prompt

```yaml
# backend/prompts/agent/planner_v1.yaml
name: collection_planner
version: v1
description: 根据用户意图生成数据采集计划
model: claude-sonnet-4-6
temperature: 0.3
system: |
  你是一个数据采集规划专家。用户会给你一个数据采集意图，你需要制定一个采集计划。

  你可以使用的工具：
  1. web_search(query) — 搜索互联网获取相关信息和URL
  2. 你的知识库 — 对知名人物/组织，你可以建议 Wikipedia 等权威来源 URL

  输出格式为 JSON，包含采集步骤和建议的 URL 列表。

  规则：
  1. 优先使用 Wikipedia、官方机构网站等权威来源
  2. 每个步骤说明要采集什么数据
  3. 标注每个 URL 是否需要 JS 渲染
  4. 估算每个步骤的预计数据量
  5. 如果需要搜索，提供搜索关键词

user_template: |
  用户意图：{{intent}}

  已有本体类型：
  {{existing_types}}

  请生成采集计划：

few_shots:
  - input: |
      用户意图：帮我收集特朗普第二任期的内阁成员信息
      已有本体类型：person, government_org
    output: |
      ```json
      {
        "plan_summary": "收集特朗普第二任期(2025起)内阁成员的个人信息和任命关系",
        "steps": [
          {
            "step": 1,
            "description": "获取特朗普第二任期内阁完整名单",
            "urls": [
              {
                "url": "https://en.wikipedia.org/wiki/Second_presidency_of_Donald_Trump#Cabinet",
                "needs_js": false,
                "purpose": "获取内阁成员名单和职位"
              }
            ],
            "target_type": "person",
            "estimated_entities": 15
          },
          {
            "step": 2,
            "description": "逐人爬取详细背景信息",
            "urls": [
              {
                "url": "https://en.wikipedia.org/wiki/Marco_Rubio",
                "needs_js": false,
                "purpose": "国务卿 Rubio 详细信息"
              },
              {
                "url": "https://en.wikipedia.org/wiki/Pete_Hegseth",
                "needs_js": false,
                "purpose": "国防部长 Hegseth 详细信息"
              }
            ],
            "target_type": "person",
            "estimated_entities": 15
          },
          {
            "step": 3,
            "description": "获取政府部门信息",
            "urls": [
              {
                "url": "https://en.wikipedia.org/wiki/United_States_federal_executive_departments",
                "needs_js": false,
                "purpose": "联邦部门概况"
              }
            ],
            "target_type": "government_org",
            "estimated_entities": 15
          }
        ],
        "suggested_link_types": [
          {"name": "appointed_to", "description": "人物被任命到某个政府部门"}
        ],
        "total_estimated_entities": 45
      }
      ```
```

---

### Task 2: 搜索工具集成

```python
# backend/app/services/search.py
"""简单的搜索工具，用于规划器获取搜索结果"""

import httpx
from typing import Optional

async def web_search(query: str, num_results: int = 5) -> list[dict]:
    """
    搜索互联网获取结果。

    可选实现：
    1. 使用 SerpAPI / Google Custom Search API
    2. 使用 DuckDuckGo（免费，无需 API Key）
    3. 使用 Tavily API
    """
    # 方案A：DuckDuckGo（零成本启动）
    from duckduckgo_search import DDGS
    results = []
    with DDGS() as ddgs:
        for r in ddgs.text(query, max_results=num_results):
            results.append({
                "title": r["title"],
                "url": r["href"],
                "snippet": r["body"],
            })
    return results

    # 方案B：Tavily（效果好，需 API Key）
    # async with httpx.AsyncClient() as client:
    #     resp = await client.post("https://api.tavily.com/search", json={
    #         "query": query,
    #         "max_results": num_results,
    #         "api_key": settings.TAVILY_API_KEY,
    #     })
    #     return resp.json().get("results", [])
```

---

### Task 3: 规划器服务

```python
# backend/app/services/planner.py
from .prompt_loader import load_prompt
from .llm import call_claude_json
from .search import web_search
from ..services.ontology_engine import OntologyEngine
from sqlalchemy.ext.asyncio import AsyncSession
import json

class PlannerService:
    def __init__(self, db: AsyncSession):
        self.db = db
        self.prompt = load_prompt("agent/planner_v1.yaml")
        self.engine = OntologyEngine(db)

    async def plan(self, intent: str) -> dict:
        """根据用户意图生成采集计划"""

        # 1. 获取已有本体类型
        types = await self.engine.get_object_types()
        existing_types = ", ".join([f"{t.name}({t.display_name})" for t in types])
        type_schemas = {t.name: t.schema_definition for t in types}

        # 2. LLM 生成采集计划
        user_message = self.prompt.render(
            intent=intent,
            existing_types=existing_types,
        )
        plan = await call_claude_json(self.prompt, user_message)

        # 3. 搜索补充（如果计划中提到的搜索关键词）
        # 规划器可能需要搜索来找到更多 URL
        plan = await self._enrich_plan_with_search(plan, intent)

        # 4. 验证 URL 可达性（可选，快速 HEAD 请求）
        plan = await self._validate_urls(plan)

        return plan

    async def _enrich_plan_with_search(self, plan: dict, intent: str) -> dict:
        """用搜索结果补充计划中的 URL"""
        search_results = await web_search(intent, num_results=5)
        # 将搜索结果中高质量的 URL 添加到计划中
        for result in search_results:
            url = result["url"]
            # 检查是否已在计划中
            existing_urls = set()
            for step in plan.get("steps", []):
                for u in step.get("urls", []):
                    existing_urls.add(u["url"])

            if url not in existing_urls and "wikipedia" in url.lower():
                # 添加到第一步
                if plan.get("steps"):
                    plan["steps"][0]["urls"].append({
                        "url": url,
                        "needs_js": False,
                        "purpose": result["snippet"][:100],
                    })
        return plan

    async def _validate_urls(self, plan: dict) -> dict:
        """快速验证 URL 可达性"""
        import httpx
        async with httpx.AsyncClient(timeout=5) as client:
            for step in plan.get("steps", []):
                for url_info in step.get("urls", []):
                    try:
                        resp = await client.head(url_info["url"], follow_redirects=True)
                        url_info["reachable"] = resp.status_code < 400
                    except:
                        url_info["reachable"] = False
        return plan
```

---

### Task 4: 规划器 API

```python
# backend/app/api/agent.py
from fastapi import APIRouter, Depends
from ..services.planner import PlannerService

router = APIRouter(prefix="/api/v1/agent", tags=["agent"])

@router.post("/plan")
async def create_plan(intent: str, db: AsyncSession = Depends(get_db)):
    """根据意图生成采集计划"""
    planner = PlannerService(db)
    plan = await planner.plan(intent)
    return plan
```

---

## 完成标准

```bash
# 输入意图，返回采集计划
curl -X POST "http://localhost:8000/api/v1/agent/plan?intent=帮我收集特朗普第二任期的内阁成员信息"

# 期望返回：
{
  "plan_summary": "收集特朗普第二任期内阁成员...",
  "steps": [
    {
      "step": 1,
      "description": "获取内阁完整名单",
      "urls": [
        {"url": "https://en.wikipedia.org/wiki/...", "needs_js": false, "purpose": "..."}
      ],
      "target_type": "person",
      "estimated_entities": 15
    },
    ...
  ],
  "total_estimated_entities": 45
}
```

## 预计产出时间

1-2天
