# P1.2 Ontology Engine API

> 目标：实现本体和数据的全部 CRUD API，Postman/curl 可测试所有接口。

---

## 依赖

- P1.1 完成（数据库 + ORM 模型 + 种子数据可运行）

## 任务清单

### Task 1: Pydantic Schema 定义

```python
# backend/app/schemas/ontology.py
from pydantic import BaseModel
from uuid import UUID
from datetime import datetime
from typing import Optional

# --- Object Type ---
class ObjectTypeCreate(BaseModel):
    name: str
    display_name: Optional[str] = None
    description: Optional[str] = None
    schema_definition: dict

class ObjectTypeResponse(BaseModel):
    id: UUID
    name: str
    display_name: Optional[str]
    description: Optional[str]
    schema_definition: dict
    created_at: datetime
    updated_at: datetime

class ObjectTypeUpdate(BaseModel):
    display_name: Optional[str] = None
    description: Optional[str] = None
    schema_definition: Optional[dict] = None

# --- Link Type ---
class LinkTypeCreate(BaseModel):
    name: str
    display_name: Optional[str] = None
    description: Optional[str] = None
    source_object_type_id: UUID
    target_object_type_id: UUID
    schema_definition: Optional[dict] = None
    is_directional: bool = True

class LinkTypeResponse(BaseModel):
    id: UUID
    name: str
    display_name: Optional[str]
    source_object_type_id: UUID
    target_object_type_id: UUID
    schema_definition: Optional[dict]
    is_directional: bool
    created_at: datetime

# --- Object ---
class ObjectCreate(BaseModel):
    object_type_name: str          # 通过名称引用，前端更友好
    name: str
    properties: dict
    properties_meta: Optional[dict] = None

class ObjectResponse(BaseModel):
    id: UUID
    object_type_id: UUID
    object_type_name: Optional[str] = None  # 响应时附带
    name: str
    properties: dict
    properties_meta: Optional[dict] = None
    status: str
    created_at: datetime
    updated_at: datetime

class ObjectUpdate(BaseModel):
    name: Optional[str] = None
    properties: Optional[dict] = None
    properties_meta: Optional[dict] = None
    status: Optional[str] = None

# --- Link ---
class LinkCreate(BaseModel):
    link_type_name: str
    source_object_id: UUID
    target_object_id: UUID
    properties: Optional[dict] = None
    properties_meta: Optional[dict] = None

class LinkResponse(BaseModel):
    id: UUID
    link_type_id: UUID
    link_type_name: Optional[str] = None
    source_object_id: UUID
    target_object_id: UUID
    properties: Optional[dict]
    status: str
    created_at: datetime
```

---

### Task 2: Service 层 — Ontology Engine

```python
# backend/app/services/ontology_engine.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from sqlalchemy.orm import selectinload
from ..models.ontology import ObjectType, LinkType
from ..models.object import Object
from ..models.link import Link
from ..schemas.ontology import *

class OntologyEngine:
    def __init__(self, db: AsyncSession):
        self.db = db

    # --- Object Type ---
    async def create_object_type(self, data: ObjectTypeCreate) -> ObjectType: ...
    async def get_object_types(self) -> list[ObjectType]: ...
    async def get_object_type(self, id: UUID) -> ObjectType: ...
    async def update_object_type(self, id: UUID, data: ObjectTypeUpdate) -> ObjectType: ...
    async def delete_object_type(self, id: UUID) -> bool: ...

    # --- Link Type ---
    async def create_link_type(self, data: LinkTypeCreate) -> LinkType: ...
    async def get_link_types(self) -> list[LinkType]: ...
    async def get_link_type(self, id: UUID) -> LinkType: ...

    # --- Object ---
    async def create_object(self, data: ObjectCreate) -> Object:
        """通过 object_type_name 查找 ObjectType，然后创建 Object"""
        ot = await self._get_type_by_name(data.object_type_name)
        obj = Object(
            object_type_id=ot.id,
            name=data.name,
            properties=data.properties,
            properties_meta=data.properties_meta or {},
        )
        self.db.add(obj)
        await self.db.commit()
        await self.db.refresh(obj)
        return obj

    async def get_objects(self, type_name: Optional[str] = None,
                          search: Optional[str] = None,
                          skip: int = 0, limit: int = 50) -> list[Object]:
        """支持按类型过滤和名称搜索"""
        query = select(Object).where(Object.status == "active")
        if type_name:
            ot = await self._get_type_by_name(type_name)
            query = query.where(Object.object_type_id == ot.id)
        if search:
            query = query.where(Object.name.ilike(f"%{search}%"))
        query = query.offset(skip).limit(limit).order_by(Object.updated_at.desc())
        result = await self.db.execute(query)
        return list(result.scalars().all())

    async def get_object(self, id: UUID) -> Optional[Object]:
        """获取对象详情，含关联的所有关系"""
        result = await self.db.execute(
            select(Object).where(Object.id == id)
        )
        return result.scalar_one_or_none()

    async def get_object_links(self, object_id: UUID) -> dict:
        """获取与某对象关联的所有关系"""
        outgoing = select(Link).where(Link.source_object_id == object_id)
        incoming = select(Link).where(Link.target_object_id == object_id)
        # 执行并返回 {outgoing: [...], incoming: [...]}
        ...

    async def update_object(self, id: UUID, data: ObjectUpdate) -> Object: ...
    async def delete_object(self, id: UUID) -> bool:
        """软删除：标记 status = deprecated"""
        ...

    # --- Link ---
    async def create_link(self, data: LinkCreate) -> Link: ...
    async def get_links(self, object_id: Optional[UUID] = None,
                        type_name: Optional[str] = None) -> list[Link]: ...
    async def delete_link(self, id: UUID) -> bool: ...

    # --- Helper ---
    async def _get_type_by_name(self, name: str) -> ObjectType:
        result = await self.db.execute(
            select(ObjectType).where(ObjectType.name == name)
        )
        ot = result.scalar_one_or_none()
        if not ot:
            raise ValueError(f"ObjectType '{name}' not found")
        return ot
```

---

### Task 3: API 路由

```python
# backend/app/api/ontology.py
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.ext.asyncio import AsyncSession
from ..database import get_db
from ..services.ontology_engine import OntologyEngine
from ..schemas.ontology import *

router = APIRouter(prefix="/api/v1", tags=["ontology"])

def get_engine(db: AsyncSession = Depends(get_db)) -> OntologyEngine:
    return OntologyEngine(db)

# --- Object Types ---
@router.post("/types", response_model=ObjectTypeResponse, status_code=201)
async def create_object_type(data: ObjectTypeCreate, engine=Depends(get_engine)): ...

@router.get("/types", response_model=list[ObjectTypeResponse])
async def list_object_types(engine=Depends(get_engine)): ...

@router.get("/types/{type_id}", response_model=ObjectTypeResponse)
async def get_object_type(type_id: UUID, engine=Depends(get_engine)): ...

@router.patch("/types/{type_id}", response_model=ObjectTypeResponse)
async def update_object_type(type_id: UUID, data: ObjectTypeUpdate, engine=Depends(get_engine)): ...

# --- Link Types ---
@router.post("/link-types", response_model=LinkTypeResponse, status_code=201)
async def create_link_type(data: LinkTypeCreate, engine=Depends(get_engine)): ...

@router.get("/link-types", response_model=list[LinkTypeResponse])
async def list_link_types(engine=Depends(get_engine)): ...

# --- Objects ---
@router.post("/objects", response_model=ObjectResponse, status_code=201)
async def create_object(data: ObjectCreate, engine=Depends(get_engine)): ...

@router.get("/objects", response_model=list[ObjectResponse])
async def list_objects(
    type: Optional[str] = None,
    search: Optional[str] = None,
    skip: int = 0, limit: int = 50,
    engine=Depends(get_engine)
): ...

@router.get("/objects/{object_id}", response_model=ObjectDetailResponse)
async def get_object(object_id: UUID, engine=Depends(get_engine)):
    """返回对象详情 + 所有关联关系 + 来源信息"""
    ...

@router.patch("/objects/{object_id}", response_model=ObjectResponse)
async def update_object(object_id: UUID, data: ObjectUpdate, engine=Depends(get_engine)): ...

@router.delete("/objects/{object_id}")
async def delete_object(object_id: UUID, engine=Depends(get_engine)): ...

# --- Links ---
@router.post("/links", response_model=LinkResponse, status_code=201)
async def create_link(data: LinkCreate, engine=Depends(get_engine)): ...

@router.get("/links", response_model=list[LinkResponse])
async def list_links(
    object_id: Optional[UUID] = None,
    type: Optional[str] = None,
    engine=Depends(get_engine)
): ...

@router.delete("/links/{link_id}")
async def delete_link(link_id: UUID, engine=Depends(get_engine)): ...
```

---

### Task 4: FastAPI 入口 & CORS

```python
# backend/app/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from .api.ontology import router as ontology_router

app = FastAPI(title="Ontology Platform", version="0.1.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(ontology_router)

@app.get("/health")
async def health():
    return {"status": "ok"}
```

---

## 完成标准

用以下 curl 命令逐一验证：

```bash
# 1. 健康检查
curl http://localhost:8000/health

# 2. 创建 Object Type
curl -X POST http://localhost:8000/api/v1/types \
  -H "Content-Type: application/json" \
  -d '{"name": "person", "display_name": "人物", "schema_definition": {"properties": {"name": {"type": "string"}}}}'

# 3. 列出所有 Object Types
curl http://localhost:8000/api/v1/types

# 4. 创建 Link Type
curl -X POST http://localhost:8000/api/v1/link-types \
  -H "Content-Type: application/json" \
  -d '{"name": "ally", "display_name": "盟友", "source_object_type_id": "<uuid>", "target_object_type_id": "<uuid>"}'

# 5. 创建 Object
curl -X POST http://localhost:8000/api/v1/objects \
  -H "Content-Type: application/json" \
  -d '{"object_type_name": "person", "name": "特朗普", "properties": {"name": "唐纳德·特朗普", "party": "Republican"}}'

# 6. 按类型查询 Objects
curl "http://localhost:8000/api/v1/objects?type=person"

# 7. 获取 Object 详情
curl http://localhost:8000/api/v1/objects/<uuid>

# 8. 创建 Link
curl -X POST http://localhost:8000/api/v1/links \
  -H "Content-Type: application/json" \
  -d '{"link_type_name": "ally", "source_object_id": "<uuid1>", "target_object_id": "<uuid2>"}'

# 9. 删除 Object（软删除）
curl -X DELETE http://localhost:8000/api/v1/objects/<uuid>
```

## 测试文件

```python
# backend/tests/test_api.py
import pytest
from httpx import AsyncClient, ASGITransport
from app.main import app

@pytest.mark.asyncio
async def test_health():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        resp = await client.get("/health")
        assert resp.status_code == 200

@pytest.mark.asyncio
async def test_object_type_crud():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        # Create
        resp = await client.post("/api/v1/types", json={
            "name": "test_type",
            "schema_definition": {"properties": {"name": {"type": "string"}}}
        })
        assert resp.status_code == 201
        type_id = resp.json()["id"]

        # List
        resp = await client.get("/api/v1/types")
        assert resp.status_code == 200
        assert len(resp.json()) >= 1

        # Get
        resp = await client.get(f"/api/v1/types/{type_id}")
        assert resp.status_code == 200

@pytest.mark.asyncio
async def test_object_crud():
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
        # 先创建 type
        resp = await client.post("/api/v1/types", json={
            "name": "test_obj_type",
            "schema_definition": {"properties": {"name": {"type": "string"}}}
        })
        type_id = resp.json()["id"]

        # 创建 object
        resp = await client.post("/api/v1/objects", json={
            "object_type_name": "test_obj_type",
            "name": "测试对象",
            "properties": {"name": "测试"}
        })
        assert resp.status_code == 201
        obj_id = resp.json()["id"]

        # 查询
        resp = await client.get(f"/api/v1/objects/{obj_id}")
        assert resp.status_code == 200
        assert resp.json()["name"] == "测试对象"

        # 按类型查询
        resp = await client.get("/api/v1/objects?type=test_obj_type")
        assert resp.status_code == 200
```

## 预计产出时间

1-2天
