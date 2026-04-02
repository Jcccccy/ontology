# P1.3 前端骨架

> 目标：Next.js 前端，能浏览对象列表、查看对象详情（属性+关联关系）。

---

## 依赖

- P1.2 完成（API 可用）

## 任务清单

### Task 1: Next.js 项目初始化

```bash
npx create-next-app@latest frontend --typescript --tailwind --app --src-dir
cd frontend
npm install
```

**目录结构：**
```
frontend/src/
├── app/
│   ├── layout.tsx          # 全局布局（侧边栏导航）
│   ├── page.tsx            # 首页（跳转到对象列表）
│   ├── objects/
│   │   ├── page.tsx        # 对象列表（按类型筛选）
│   │   └── [id]/
│   │       └── page.tsx    # 对象详情（属性卡片 + 关联关系）
│   ├── types/
│   │   └── page.tsx        # 本体类型管理（简单列表）
│   └── globals.css
├── components/
│   ├── Sidebar.tsx         # 侧边导航
│   ├── ObjectCard.tsx      # 对象卡片
│   ├── PropertyTable.tsx   # 属性表格（含来源信息）
│   ├── RelatedLinks.tsx    # 关联关系列表
│   └── TypeBadge.tsx       # 类型标签
├── lib/
│   ├── api.ts              # API 调用封装
│   └── types.ts            # TypeScript 类型定义
└── ...
```

---

### Task 2: API 调用封装

```typescript
// frontend/src/lib/api.ts
const API_BASE = process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000/api/v1';

export async function fetchObjectTypes() {
  const res = await fetch(`${API_BASE}/types`);
  return res.json();
}

export async function fetchObjects(params?: { type?: string; search?: string }) {
  const query = new URLSearchParams();
  if (params?.type) query.set('type', params.type);
  if (params?.search) query.set('search', params.search);
  const res = await fetch(`${API_BASE}/objects?${query}`);
  return res.json();
}

export async function fetchObject(id: string) {
  const res = await fetch(`${API_BASE}/objects/${id}`);
  return res.json();
}

export async function createObject(data: any) {
  const res = await fetch(`${API_BASE}/objects`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(data),
  });
  return res.json();
}

export async function deleteObject(id: string) {
  await fetch(`${API_BASE}/objects/${id}`, { method: 'DELETE' });
}
```

---

### Task 3: 全局布局 + 侧边栏

```tsx
// frontend/src/app/layout.tsx
// 左侧固定侧边栏 + 右侧内容区
// 侧边栏导航项：首页、对象浏览、类型管理、审核队列（P1.4 激活）

// frontend/src/components/Sidebar.tsx
// 导航项：
// - 📊 首页
// - 📋 对象浏览 → /objects
// - 🔗 关系图谱 → /graph (P3 激活)
// - 💬 对话 → /chat (P3 激活)
// - ✅ 审核队列 → /review (P1.4 激活)
// - ⚙️ 类型管理 → /types
```

---

### Task 4: 对象列表页

```tsx
// frontend/src/app/objects/page.tsx
//
// 布局：
// ┌──────────────────────────────────┐
// │ [类型下拉筛选]  [搜索框]  [+ 新建]│
// ├──────────────────────────────────┤
// │  卡片网格 (3-4列)                │
// │  ┌──────┐  ┌──────┐  ┌──────┐  │
// │  │ 特朗普│  │ 马斯克│  │ 蓬佩奥│ │
// │  │ 人物  │  │ 人物  │  │ 人物  │ │
// │  │ R     │  │ DOGE  │  │ 前国务│ │
// │  └──────┘  └──────┘  └──────┘  │
// └──────────────────────────────────┘
//
// 功能：
// - 顶部类型下拉筛选（从 /types 获取）
// - 搜索框（名称模糊搜索）
// - 每张卡片显示：名称、类型、关键属性摘要
// - 点击卡片跳转详情页
```

---

### Task 5: 对象详情页

```tsx
// frontend/src/app/objects/[id]/page.tsx
//
// 布局：
// ┌──────────────────────────────────────────┐
// │ ← 返回列表   特朗普   [编辑] [删除]       │
// ├──────────────────────────────────────────┤
// │                                          │
// │  ┌─ 基本信息 ──────────────────────────┐ │
// │  │ 姓名: 唐纳德·特朗普    来源: wiki  ✓  │ │
// │  │ 英文名: Donald Trump   来源: wiki  ✓  │ │
// │  │ 党派: Republican       来源: wiki  ✓  │ │
// │  │ 职位: 第47任总统       来源: 官网  ✓  │ │
// │  │ 背景: ...              来源: wiki     │ │
// │  └─────────────────────────────────────┘ │
// │                                          │
// │  ┌─ 关联关系 ──────────────────────────┐ │
// │  │                                      │ │
// │  │  → 任命 → 白宫    (source: 官方公告)  │ │
// │  │  ↔ 盟友 ↔ 马斯克  (source: 媒体报道)  │ │
// │  │  ↔ 家族 ↔ 小唐纳德                  │ │
// │  │                                      │ │
// │  │  [+ 添加关系]                        │ │
// │  └─────────────────────────────────────┘ │
// │                                          │
// │  ┌─ 来源信息 ──────────────────────────┐ │
// │  │ 共 5 个属性，3 个来源                 │ │
// │  │ 来源: Wikipedia, 白宫官网, CNN        │ │
// │  │ 最后更新: 2026-03-28                  │ │
// │  └─────────────────────────────────────┘ │
// │                                          │
// └──────────────────────────────────────────┘
//
// 关键点：
// - 每个属性右侧显示来源和确认状态（可信信息展示）
// - 关联关系可点击跳转到关联对象
// - 来源信息面板汇总所有数据来源
```

---

### Task 6: 新建对象弹窗

```tsx
// frontend/src/components/CreateObjectModal.tsx
//
// 功能：
// 1. 选择 Object Type（下拉）
// 2. 根据 schema_definition 动态生成表单字段
// 3. 提交到 POST /objects
//
// 关键：表单根据 schema 动态渲染
// schema_definition.properties 中每个字段 → 一个输入框
// type=string → text input
// type=date → date picker
// type=text → textarea
```

---

## 完成标准

- [ ] `npm run dev` 启动无报错
- [ ] 侧边栏导航可切换页面
- [ ] 对象列表页：类型筛选和搜索工作正常
- [ ] 对象详情页：属性和关联关系正确展示
- [ ] 新建对象弹窗：可创建并立即在列表中看到
- [ ] 前后端联调成功（API 返回数据正确渲染）

## 预计产出时间

2-3天
