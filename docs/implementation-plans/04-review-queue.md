# P1.4 审核队列

> 目标：候选实体/关系的人工审核界面，支持一键确认/拒绝、冲突对比。

---

## 依赖

- P1.2 API 完成
- P1.3 前端骨架完成

## 任务清单

### Task 1: 审核队列 API

```python
# backend/app/api/review.py

# GET  /api/v1/review/candidates          # 列出候选（支持状态筛选）
# GET  /api/v1/review/candidates/{id}     # 候选详情
# POST /api/v1/review/candidates/{id}/approve   # 确认（候选 → 正式入库）
# POST /api/v1/review/candidates/{id}/reject    # 拒绝
# POST /api/v1/review/candidates/{id}/modify    # 修改后确认

# GET  /api/v1/review/stats              # 统计：待审核/已通过/已拒绝数量
```

**核心逻辑 — approve 接口：**

```python
async def approve_candidate(candidate_id: UUID, db: AsyncSession):
    candidate = await get_candidate(candidate_id, db)

    if candidate.candidate_type == "object":
        # 从候选数据创建正式 Object
        obj = Object(
            object_type_id=candidate.suggested_type_id,
            name=candidate.data.get("name", "未命名"),
            properties=candidate.data,
            properties_meta=_build_meta(candidate),
        )
        db.add(obj)

    elif candidate.candidate_type == "link":
        link = Link(
            link_type_id=candidate.suggested_type_id,
            source_object_id=candidate.data["source_id"],
            target_object_id=candidate.data["target_id"],
            properties=candidate.data.get("properties"),
        )
        db.add(link)

    # 更新候选状态
    candidate.status = "approved"
    candidate.reviewed_at = datetime.utcnow()

    # 更新审核队列
    review = await get_review_by_candidate(candidate_id, db)
    review.status = "resolved"
    review.resolution = "approved"

    await db.commit()
```

**冲突检测逻辑：**

```python
async def detect_conflicts(candidate: EntityCandidate, db: AsyncSession) -> Optional[UUID]:
    """检查候选数据是否与已有数据冲突"""
    if candidate.candidate_type == "object":
        # 按名称查找同名对象
        existing = await db.execute(
            select(Object).where(
                Object.name.ilike(candidate.data.get("name", "")),
                Object.object_type_id == candidate.suggested_type_id
            )
        )
        existing_obj = existing.scalar_one_or_none()
        if existing_obj:
            # 比较属性，检测不一致
            for key, new_value in candidate.data.items():
                old_value = existing_obj.properties.get(key)
                if old_value and old_value != new_value:
                    return existing_obj.id  # 有冲突
    return None
```

---

### Task 2: 审核队列前端页面

```tsx
// frontend/src/app/review/page.tsx
//
// 布局（参考 implementation-report.md 4.4 节设计）：
//
// ┌──────────────────────────────────────────────────┐
// │  审核队列                                        │
// │  待审核：23条   已通过：142条   已拒绝：8条          │
// ├──────────────────────────────────────────────────┤
// │  [筛选: 全部 / 低置信度 / 冲突 / 新实体]           │
// ├───────────────────────────────┬──────────────────┤
// │  候选列表（左侧）              │  详情面板（右侧） │
// │                               │                  │
// │  ○ 特朗普 · 人物 · 0.82       │  姓名: 唐纳德·.. │
// │    来源: Wikipedia             │  职位: 第47任... │
// │    ⚠ 属性冲突                  │  党派: Republican│
// │                               │                  │
// │  ○ 任命关系 · 0.71            │  ── 冲突字段 ──  │
// │    马斯克 → DOGE              │  职位:           │
// │    来源: 官方公告              │  [已有] "CEO"    │
// │                               │  [新值] "DOGE头" │
// │  ...                          │  ○ 保留已有      │
// │                               │  ○ 采用新值      │
// │                               │                  │
// │                               │  [✓ 确认] [✗ 拒绝]│
// └───────────────────────────────┴──────────────────┘
```

**关键组件：**

```tsx
// CandidateList.tsx — 候选列表
// - 显示类型图标（人物/关系/组织）
// - 显示置信度（颜色编码：高/中/低）
// - 冲突标记（黄色警告图标）
// - 支持键盘导航（J/K 上下，Y 确认，N 拒绝）

// CandidateDetail.tsx — 详情面板
// - 显示候选数据的所有字段
// - 冲突字段高亮对比（旧值 vs 新值）
// - 来源信息展示
// - 确认/拒绝按钮

// ConflictResolver.tsx — 冲突解决组件
// - 逐字段对比
// - 每个冲突字段可选择：保留已有 / 采用新值 / 两者都保留
```

**键盘快捷键：**

```typescript
// 全局快捷键处理
useEffect(() => {
  const handler = (e: KeyboardEvent) => {
    if (e.key === 'j') navigateNext();     // 下一条
    if (e.key === 'k') navigatePrev();     // 上一条
    if (e.key === 'y') approveCurrent();   // 确认
    if (e.key === 'n') rejectCurrent();    // 拒绝
  };
  window.addEventListener('keydown', handler);
  return () => window.removeEventListener('keydown', handler);
}, []);
```

---

### Task 3: 候选数据手动创建接口（测试用）

```python
# POST /api/v1/review/candidates/create-test
# 临时接口，用于手动创建候选数据测试审核流程

async def create_test_candidate(data: dict, db: AsyncSession):
    candidate = EntityCandidate(
        candidate_type=data["type"],     # "object" / "link"
        suggested_type_id=data["type_id"],
        data=data["data"],
        confidence=data.get("confidence", 0.8),
        status="pending",
    )
    db.add(candidate)

    # 自动检测冲突
    conflict_id = await detect_conflicts(candidate, db)
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
    await db.commit()
```

---

## 完成标准

- [ ] 创建测试候选数据 → 出现在审核队列列表
- [ ] 点击候选 → 右侧详情面板显示完整数据
- [ ] 点击确认 → 候选消失，正式数据在对象列表中可见
- [ ] 点击拒绝 → 候选消失，不进入正式数据
- [ ] 冲突候选 → 冲突字段高亮，可选择保留/替换
- [ ] 键盘 J/K/Y/N 快捷键工作正常
- [ ] 统计数字（待审核/已通过/已拒绝）实时更新

## 测试脚本

```bash
# 1. 创建测试候选数据
curl -X POST http://localhost:8000/api/v1/review/candidates/create-test \
  -H "Content-Type: application/json" \
  -d '{
    "type": "object",
    "type_id": "<person_type_uuid>",
    "data": {"name": "JD Vance", "title": "副总统", "party": "Republican"},
    "confidence": 0.85
  }'

# 2. 查看待审核列表
curl http://localhost:8000/api/v1/review/candidates?status=pending

# 3. 确认候选
curl -X POST http://localhost:8000/api/v1/review/candidates/<id>/approve

# 4. 验证正式数据
curl http://localhost:8000/api/v1/objects?search=Vance
```

## P1 阶段里程碑

完成 P1.1 ~ P1.4 后，达到第一个里程碑：

> **手动数据能跑通全流程：** 创建类型 → 录入对象 → 建立关系 → 前端浏览 → 字段来源可见 → 审核队列可对候选数据做确认/拒绝。

## 预计产出时间

2天
