# P1.1 数据库 & ORM 模型

> 目标：建立 PostgreSQL 数据库，定义所有表结构，用 SQLAlchemy ORM 模型映射，确保种子数据可写入和查询。

---

## 前置条件

- Python 3.11+
- PostgreSQL 15+ 运行中
- Redis 7+ 运行中

## 任务清单

### Task 1: 项目初始化

```
1. 创建项目目录结构
2. 创建 backend/requirements.txt:
   - fastapi
   - uvicorn[standard]
   - sqlalchemy[asyncio]
   - asyncpg
   - alembic
   - pydantic
   - pydantic-settings
   - python-dotenv
3. 创建 docker-compose.yml:
   - postgres:15 (端口5432, 用户ontology, 密码ontology123, 库名ontology)
   - redis:7 (端口6379)
4. 创建 backend/.env:
   DATABASE_URL=postgresql+asyncpg://ontology:ontology123@localhost:5432/ontology
   REDIS_URL=redis://localhost:6379/0
```

**测试方法：** `docker compose up -d` 后 `pg_isready` 和 `redis-cli ping` 均成功。

---

### Task 2: 数据库连接 & 配置

```python
# backend/app/config.py
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    DATABASE_URL: str
    REDIS_URL: str
    CLAUDE_API_KEY: str = ""

    class Config:
        env_file = ".env"

settings = Settings()
```

```python
# backend/app/database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase
from .config import settings

engine = create_async_engine(settings.DATABASE_URL, echo=True)
async_session = async_sessionmaker(engine, expire_on_commit=False)

class Base(DeclarativeBase):
    pass

async def get_db() -> AsyncSession:
    async with async_session() as session:
        yield session
```

**测试方法：** 启动 Python，`await engine.connect()` 无报错。

---

### Task 3: ORM 模型定义

#### 3a. ontology_type — Object Type / Link Type 定义

```python
# backend/app/models/ontology.py
import uuid
from sqlalchemy import Column, String, Text, DateTime, JSON, func
from sqlalchemy.dialects.postgresql import UUID
from ..database import Base

class ObjectType(Base):
    """对象类型定义，如"人物""公司"等"""
    __tablename__ = "object_types"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String(100), unique=True, nullable=False)       # "人物"
    display_name = Column(String(200))                             # "政治人物"
    description = Column(Text)
    schema_definition = Column(JSON, nullable=False)               # {"properties": {"name": {"type": "string", "required": true}, ...}}
    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, server_default=func.now(), onupdate=func.now())

class LinkType(Base):
    """关系类型定义，如"任命""盟友"等"""
    __tablename__ = "link_types"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String(100), unique=True, nullable=False)        # "appointed_to"
    display_name = Column(String(200))                             # "任命"
    description = Column(Text)
    source_object_type_id = Column(UUID(as_uuid=True), nullable=False)  # 起点类型
    target_object_type_id = Column(UUID(as_uuid=True), nullable=False)  # 终点类型
    schema_definition = Column(JSON)                                # 关系上的属性定义
    is_directional = Column(Boolean, default=True)                  # 是否有方向
    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, server_default=func.now(), onupdate=func.now())
```

#### 3b. object / link — 实体数据

```python
# backend/app/models/object.py
class Object(Base):
    """具体对象实例"""
    __tablename__ = "objects"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    object_type_id = Column(UUID(as_uuid=True), nullable=False, index=True)
    name = Column(String(500), nullable=False)         # 对象名称（冗余，便于查询）
    properties = Column(JSONB, nullable=False)          # {"name": "特朗普", "party": "Republican", ...}
    properties_meta = Column(JSONB)                     # 字段级元数据: {"name": {"source_url": "...", "confidence": 0.95, "confirmed": true}}
    status = Column(String(20), default="active")       # active / deprecated
    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, server_default=func.now(), onupdate=func.now())

# backend/app/models/link.py
class Link(Base):
    """关系实例"""
    __tablename__ = "links"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    link_type_id = Column(UUID(as_uuid=True), nullable=False, index=True)
    source_object_id = Column(UUID(as_uuid=True), nullable=False, index=True)
    target_object_id = Column(UUID(as_uuid=True), nullable=False, index=True)
    properties = Column(JSONB)                          # 关系上的属性
    properties_meta = Column(JSONB)                     # 字段级元数据
    status = Column(String(20), default="active")
    created_at = Column(DateTime, server_default=func.now())
    updated_at = Column(DateTime, server_default=func.now(), onupdate=func.now())
```

#### 3c. source / candidate / review — Agent 数据管道

```python
# backend/app/models/source.py
class RawSource(Base):
    """Agent 爬取的原始数据"""
    __tablename__ = "raw_sources"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    url = Column(Text, nullable=False)
    site_name = Column(String(200))
    fetched_at = Column(DateTime, server_default=func.now())
    content_type = Column(String(50))                   # html / pdf / api_json
    raw_content = Column(Text)                          # 原始HTML/JSON
    extracted_content = Column(JSONB)                   # LLM 提取的结构化数据
    status = Column(String(20), default="pending")      # pending / extracted / error

# backend/app/models/candidate.py
class EntityCandidate(Base):
    """待确认的实体/关系候选"""
    __tablename__ = "entity_candidates"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    candidate_type = Column(String(20), nullable=False)  # object / link
    suggested_type_id = Column(UUID(as_uuid=True))       # 建议的 object_type_id 或 link_type_id
    data = Column(JSONB, nullable=False)                 # 候选数据
    source_ids = Column(JSONB)                           # 来源 raw_source IDs
    confidence = Column(Float)                           # 提取置信度
    status = Column(String(20), default="pending")       # pending / approved / rejected / conflicted
    conflict_with = Column(UUID(as_uuid=True))           # 冲突的已有对象ID
    review_note = Column(Text)
    created_at = Column(DateTime, server_default=func.now())
    reviewed_at = Column(DateTime)

# backend/app/models/review.py
class ReviewQueue(Base):
    """审核队列"""
    __tablename__ = "review_queue"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    candidate_id = Column(UUID(as_uuid=True), nullable=False)
    review_type = Column(String(30), nullable=False)     # new_entity / conflict / low_confidence
    priority = Column(Integer, default=0)                # 优先级
    status = Column(String(20), default="pending")       # pending / in_review / resolved
    resolved_by = Column(String(100))                    # 审核人
    resolution = Column(String(20))                      # approved / rejected / modified
    resolution_note = Column(Text)
    created_at = Column(DateTime, server_default=func.now())
    resolved_at = Column(DateTime)
```

#### 3d. action_logs

```python
class ActionLog(Base):
    __tablename__ = "action_logs"

    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    action_name = Column(String(100), nullable=False)
    trigger_type = Column(String(20))                    # ui / nl / scheduled
    input_params = Column(JSONB)
    output_result = Column(JSONB)
    status = Column(String(20))                          # success / failed
    created_at = Column(DateTime, server_default=func.now())
```

**测试方法：** 所有模型 import 无报错。

---

### Task 4: Alembic 初始化 + 首次迁移

```
cd backend
alembic init alembic
# 编辑 alembic/env.py，引入 Base.metadata
alembic revision --autogenerate -m "initial schema"
alembic upgrade head
```

**测试方法：** `psql` 中 `\dt` 能看到所有表。

---

### Task 5: 种子数据脚本

编写 `backend/scripts/seed_data.py`，插入以下种子数据：

```python
# 1. 创建 Object Types
object_type_person = {
    "name": "person",
    "display_name": "人物",
    "schema_definition": {
        "properties": {
            "name": {"type": "string", "required": True, "display": "姓名"},
            "english_name": {"type": "string", "display": "英文名"},
            "title": {"type": "string", "display": "职位/头衔"},
            "party": {"type": "string", "display": "党派"},
            "birth_date": {"type": "date", "display": "出生日期"},
            "education": {"type": "string", "display": "教育背景"},
            "background": {"type": "text", "display": "背景描述"},
            "political_stance": {"type": "string", "display": "政治立场"},
        }
    }
}

object_type_org = {
    "name": "government_org",
    "display_name": "政府部门",
    "schema_definition": {
        "properties": {
            "name": {"type": "string", "required": True, "display": "名称"},
            "english_name": {"type": "string", "display": "英文名"},
            "function": {"type": "text", "display": "职能描述"},
            "level": {"type": "string", "display": "层级"},  # federal/state
        }
    }
}

# 2. 创建 Link Types
link_type_appointed = {
    "name": "appointed_to",
    "display_name": "任命",
    "source_object_type_id": person.id,
    "target_object_type_id": org.id,
    "is_directional": True,
    "schema_definition": {
        "properties": {
            "position": {"type": "string", "display": "担任职位"},
            "appointed_date": {"type": "date", "display": "任命日期"},
            "status": {"type": "string", "display": "状态"},  # active/former
        }
    }
}

link_type_ally = {
    "name": "political_ally",
    "display_name": "政治盟友",
    "source_object_type_id": person.id,
    "target_object_type_id": person.id,
    "is_directional": False,
}

link_type_family = {
    "name": "family",
    "display_name": "家族关系",
    "source_object_type_id": person.id,
    "target_object_type_id": person.id,
    "is_directional": False,
    "schema_definition": {
        "properties": {
            "relation": {"type": "string", "display": "关系"},  # 父子/配偶/兄妹
        }
    }
}

# 3. 创建 10-20 个人物对象（种子数据）
seed_persons = [
    {
        "name": "唐纳德·特朗普",
        "english_name": "Donald Trump",
        "title": "美国第47任总统",
        "party": "Republican",
        "background": "...",
    },
    {
        "name": "埃隆·马斯克",
        "english_name": "Elon Musk",
        "title": "DOGE负责人",
        "background": "...",
    },
    # ... 更多人物
]
```

**测试方法：** 运行种子脚本后，`SELECT count(*) FROM objects` 返回 ≥10 条记录。

---

## 完成标准

- [x] docker compose up 后 PostgreSQL 和 Redis 正常运行
- [x] alembic upgrade head 成功，所有表已创建
- [x] 种子数据脚本运行成功，数据可查
- [x] `pytest tests/test_models.py` 通过（基本 CRUD 测试）

## 测试文件

```python
# backend/tests/test_models.py
import pytest
from sqlalchemy import select
from app.models.ontology import ObjectType, LinkType
from app.models.object import Object
from app.models.link import Link

@pytest.mark.asyncio
async def test_create_object_type(db_session):
    ot = ObjectType(
        name="test_type",
        display_name="测试类型",
        schema_definition={"properties": {"name": {"type": "string"}}}
    )
    db_session.add(ot)
    await db_session.commit()

    result = await db_session.execute(select(ObjectType).where(ObjectType.name == "test_type"))
    assert result.scalar_one_or_none() is not None

@pytest.mark.asyncio
async def test_create_object(db_session, seed_object_type):
    obj = Object(
        object_type_id=seed_object_type.id,
        name="测试人物",
        properties={"name": "测试人物", "title": "测试"},
        properties_meta={"name": {"source_url": "https://test.com", "confidence": 1.0, "confirmed": True}}
    )
    db_session.add(obj)
    await db_session.commit()
    assert obj.id is not None

@pytest.mark.asyncio
async def test_create_link(db_session, seed_two_objects, seed_link_type):
    link = Link(
        link_type_id=seed_link_type.id,
        source_object_id=seed_two_objects[0].id,
        target_object_id=seed_two_objects[1].id,
    )
    db_session.add(link)
    await db_session.commit()
    assert link.id is not None
```

## 预计产出时间

半天（AI 辅助生成代码，人工调整 Schema 细节）
