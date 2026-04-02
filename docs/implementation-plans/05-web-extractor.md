# P2.1 网页提取器

> 目标：给定 URL，自动获取网页内容并用 LLM 提取结构化数据。

---

## 依赖

- P1.1 数据库模型（raw_sources 表）
- Claude API Key 或 OpenAI API Key

## 任务清单

### Task 1: Prompt 文件

```yaml
# backend/prompts/agent/extractor_v1.yaml
name: entity_extractor
version: v1
description: 从网页内容中提取结构化实体信息
model: claude-sonnet-4-6
temperature: 0.1
system: |
  你是一个结构化信息提取专家。你的任务是从网页内容中提取特定类型的实体信息。

  提取规则：
  1. 只提取明确出现在文本中的信息，不要推测
  2. 如果某个字段无法确定，设为 null
  3. 对每个提取的字段，评估置信度（0-1）
  4. 如果发现多个实体，分别提取
  5. 返回 JSON 格式

user_template: |
  目标本体类型：{{object_type}}
  类型定义：{{schema_definition}}

  网页内容：
  {{content}}

  请提取所有符合该类型的实体，返回 JSON 数组：
  ```json
  [
    {
      "data": { "属性名": "值", ... },
      "confidence": 0.0-1.0,
      "source_snippet": "提取依据的原文片段"
    }
  ]
  ```

few_shots:
  - input: |
      目标本体类型：person
      网页内容：Elon Musk is the CEO of Tesla and SpaceX...
    output: |
      ```json
      [
        {
          "data": {
            "name": "Elon Musk",
            "english_name": "Elon Musk",
            "title": "CEO of Tesla and SpaceX",
            "background": "Entrepreneur and business magnate"
          },
          "confidence": 0.95,
          "source_snippet": "Elon Musk is the CEO of Tesla and SpaceX"
        }
      ]
      ```
```

---

### Task 2: Prompt 加载器

```python
# backend/app/services/prompt_loader.py
import yaml
from pathlib import Path
from string import Template

PROMPTS_DIR = Path(__file__).parent.parent.parent / "prompts"

class PromptTemplate:
    def __init__(self, name: str, version: str, model: str, temperature: float,
                 system: str, user_template: str, few_shots: list):
        self.name = name
        self.version = version
        self.model = model
        self.temperature = temperature
        self.system = system
        self.user_template = user_template
        self.few_shots = few_shots

    def render(self, **kwargs) -> str:
        template = Template(self.user_template.replace("{{", "${").replace("}}", "}"))
        return template.safe_substitute(**kwargs)

def load_prompt(path: str) -> PromptTemplate:
    """加载 Prompt YAML 文件"""
    full_path = PROMPTS_DIR / path
    with open(full_path) as f:
        data = yaml.safe_load(f)
    return PromptTemplate(
        name=data["name"],
        version=data["version"],
        model=data["model"],
        temperature=data["temperature"],
        system=data["system"],
        user_template=data["user_template"],
        few_shots=data.get("few_shots", []),
    )
```

---

### Task 3: LLM 调用封装

```python
# backend/app/services/llm.py
import anthropic
import json
from .prompt_loader import PromptTemplate
from ..config import settings

async def call_claude(prompt: PromptTemplate, user_message: str) -> str:
    """调用 Claude API，返回文本响应"""
    client = anthropic.AsyncAnthropic(api_key=settings.CLAUDE_API_KEY)

    messages = []

    # 添加 few-shot 示例
    for shot in prompt.few_shots:
        messages.append({"role": "user", "content": shot["input"]})
        messages.append({"role": "assistant", "content": shot["output"]})

    messages.append({"role": "user", "content": user_message})

    response = await client.messages.create(
        model=prompt.model,
        max_tokens=4096,
        temperature=prompt.temperature,
        system=prompt.system,
        messages=messages,
    )
    return response.content[0].text

async def call_claude_json(prompt: PromptTemplate, user_message: str) -> dict | list:
    """调用 Claude 并解析 JSON 响应"""
    text = await call_claude(prompt, user_message)
    # 提取 ```json ... ``` 中的内容
    if "```json" in text:
        json_str = text.split("```json")[1].split("```")[0].strip()
    elif "```" in text:
        json_str = text.split("```")[1].split("```")[0].strip()
    else:
        json_str = text.strip()
    return json.loads(json_str)
```

---

### Task 4: 网页获取器

```python
# backend/app/services/fetcher.py
import httpx
from bs4 import BeautifulSoup
from typing import Optional

# 白名单数据源
ALLOWED_DOMAINS = [
    "wikipedia.org",
    "wikidata.org",
    "whitehouse.gov",
    "congress.gov",
    "britannica.com",
    "bbc.com",
    "reuters.com",
]

async def fetch_page(url: str, use_playwright: bool = False) -> dict:
    """获取网页内容"""
    if use_playwright:
        return await _fetch_with_playwright(url)
    return await _fetch_with_httpx(url)

async def _fetch_with_httpx(url: str) -> dict:
    """静态页面获取"""
    async with httpx.AsyncClient(follow_redirects=True, timeout=30) as client:
        resp = await client.get(url, headers={
            "User-Agent": "OntologyBot/0.1 (research bot)"
        })
        resp.raise_for_status()

        soup = BeautifulSoup(resp.text, "html.parser")

        # 移除无关元素
        for tag in soup(["script", "style", "nav", "footer", "header"]):
            tag.decompose()

        content = soup.get_text(separator="\n", strip=True)

        # 限制长度（避免超出 LLM 上下文）
        if len(content) > 50000:
            content = content[:50000]

        return {
            "url": url,
            "content": content,
            "title": soup.title.string if soup.title else "",
            "content_type": "html",
        }

async def _fetch_with_playwright(url: str) -> dict:
    """JS 渲染页面获取（按需使用）"""
    from playwright.async_api import async_playwright

    async with async_playwright() as p:
        browser = await p.chromium.launch(headless=True)
        page = await browser.new_page()
        await page.goto(url, wait_until="networkidle")
        content = await page.content()
        title = await page.title()
        await browser.close()

        soup = BeautifulSoup(content, "html.parser")
        text = soup.get_text(separator="\n", strip=True)
        if len(text) > 50000:
            text = text[:50000]

        return {
            "url": url,
            "content": text,
            "title": title,
            "content_type": "html",
        }

def is_url_allowed(url: str) -> bool:
    """检查 URL 是否在白名单中"""
    from urllib.parse import urlparse
    domain = urlparse(url).netloc.lower()
    return any(allowed in domain for allowed in ALLOWED_DOMAINS)
```

---

### Task 5: 提取器服务

```python
# backend/app/services/extractor.py
from .prompt_loader import load_prompt
from .llm import call_claude_json
from .fetcher import fetch_page, is_url_allowed
from ..models.source import RawSource
from sqlalchemy.ext.asyncio import AsyncSession

class ExtractorService:
    def __init__(self, db: AsyncSession):
        self.db = db
        self.prompt = load_prompt("agent/extractor_v1.yaml")

    async def extract_from_url(
        self, url: str, object_type_name: str, schema_definition: dict
    ) -> list[dict]:
        """从 URL 提取结构化数据"""

        # 1. 获取网页内容
        page = await fetch_page(url)

        # 2. 保存原始数据
        raw = RawSource(
            url=url,
            site_name=self._extract_site_name(url),
            raw_content=page["content"][:10000],  # 保存前10000字符
            content_type=page["content_type"],
            status="pending",
        )
        self.db.add(raw)
        await self.db.commit()
        await self.db.refresh(raw)

        # 3. LLM 提取
        user_message = self.prompt.render(
            object_type=object_type_name,
            schema_definition=str(schema_definition),
            content=page["content"],
        )
        results = await call_claude_json(self.prompt, user_message)

        # 4. 更新原始数据状态
        raw.extracted_content = results
        raw.status = "extracted"
        await self.db.commit()

        # 5. 附加来源信息
        for item in results:
            item["source_url"] = url
            item["raw_source_id"] = str(raw.id)

        return results

    def _extract_site_name(self, url: str) -> str:
        from urllib.parse import urlparse
        return urlparse(url).netloc

    async def extract_from_urls(
        self, urls: list[str], object_type_name: str, schema_definition: dict
    ) -> list[dict]:
        """批量提取"""
        all_results = []
        for url in urls:
            if not is_url_allowed(url):
                continue
            try:
                results = await self.extract_from_url(url, object_type_name, schema_definition)
                all_results.extend(results)
            except Exception as e:
                print(f"Failed to extract from {url}: {e}")
        return all_results
```

---

### Task 6: 提取器 API

```python
# backend/app/api/extractor.py
from fastapi import APIRouter, Depends
from ..services.extractor import ExtractorService

router = APIRouter(prefix="/api/v1/extractor", tags=["extractor"])

@router.post("/extract")
async def extract_from_url(
    url: str,
    object_type_name: str,
    db: AsyncSession = Depends(get_db),
):
    """从单个 URL 提取数据"""
    engine = OntologyEngine(db)
    ot = await engine._get_type_by_name(object_type_name)
    extractor = ExtractorService(db)
    results = await extractor.extract_from_url(url, object_type_name, ot.schema_definition)
    return {"url": url, "entities": results, "count": len(results)}

@router.post("/extract-batch")
async def extract_from_urls(
    urls: list[str],
    object_type_name: str,
    db: AsyncSession = Depends(get_db),
):
    """批量提取"""
    engine = OntologyEngine(db)
    ot = await engine._get_type_by_name(object_type_name)
    extractor = ExtractorService(db)
    results = await extractor.extract_from_urls(urls, object_type_name, ot.schema_definition)
    return {"count": len(results), "entities": results}
```

---

## 完成标准

```bash
# 给定 Wikipedia URL 能提取出结构化数据
curl -X POST "http://localhost:8000/api/v1/extractor/extract?url=https://en.wikipedia.org/wiki/Elon_Musk&object_type_name=person"

# 期望返回：
{
  "url": "https://en.wikipedia.org/wiki/Elon_Musk",
  "entities": [
    {
      "data": {
        "name": "Elon Musk",
        "english_name": "Elon Musk",
        "title": "CEO of Tesla, SpaceX",
        "background": "..."
      },
      "confidence": 0.92,
      "source_snippet": "...",
      "source_url": "https://en.wikipedia.org/wiki/Elon_Musk"
    }
  ],
  "count": 1
}
```

## 预计产出时间

2-3天
