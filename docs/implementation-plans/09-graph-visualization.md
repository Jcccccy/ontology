# P3.1 关系图谱可视化

> 目标：以选定人物为中心展开关系网络，支持点击展开、缩放拖拽。

---

## 依赖

- P1 全部完成（有对象和关系数据）
- P2 全部完成（Agent 能补充数据）

## 任务清单

### Task 1: 图谱数据 API

```python
# backend/app/api/graph.py

router = APIRouter(prefix="/api/v1/graph", tags=["graph"])

@router.get("/neighborhood/{object_id}")
async def get_graph_neighborhood(
    object_id: UUID,
    depth: int = 1,          # 展开层数（MVP: 1-2）
    max_nodes: int = 50,     # 最大节点数
    db: AsyncSession = Depends(get_db),
):
    """
    获取以某对象为中心的关系网络数据。

    返回格式适配 D3 force-graph:
    {
      "nodes": [{"id": "...", "name": "...", "type": "person", ...}],
      "edges": [{"source": "...", "target": "...", "type": "appointed_to", ...}]
    }
    """
    nodes = []
    edges = []
    visited = set()

    async def expand(oid: UUID, current_depth: int):
        if current_depth > depth or len(nodes) >= max_nodes or oid in visited:
            return
        visited.add(oid)

        # 获取对象信息
        obj = await engine.get_object(oid)
        if not obj:
            return

        # 添加节点
        ot = await engine.get_object_type(obj.object_type_id)
        nodes.append({
            "id": str(obj.id),
            "name": obj.name,
            "type": ot.name,
            "properties": {k: v for k, v in obj.properties.items()
                          if k in ["name", "title", "party"]},  # 只返回关键属性
        })

        # 获取所有关联关系
        links = await engine.get_object_links(oid)
        for link in links["outgoing"] + links["incoming"]:
            # 确定对端
            peer_id = link.target_object_id if link.source_object_id == oid else link.source_object_id
            lt = await engine.get_link_type(link.link_type_id)

            edges.append({
                "source": str(link.source_object_id),
                "target": str(link.target_object_id),
                "type": lt.name,
                "display_name": lt.display_name,
                "id": str(link.id),
            })

            # 递归展开对端
            await expand(peer_id, current_depth + 1)

    await expand(object_id, 0)
    return {"nodes": nodes, "edges": edges}

@router.get("/paths")
async def find_paths(
    from_id: UUID,
    to_id: UUID,
    max_hops: int = 3,
    db: AsyncSession = Depends(get_db),
):
    """查找两个实体之间的路径（递归 CTE 实现）"""
    # SQL: 使用 PostgreSQL 递归 CTE 查找路径
    query = text("""
        WITH RECURSIVE path_finder AS (
            -- 起点
            SELECT
                source_object_id AS node,
                target_object_id AS next_node,
                id AS link_id,
                ARRAY[source_object_id, target_object_id] AS path,
                ARRAY[id] AS link_path,
                1 AS depth
            FROM links
            WHERE source_object_id = :from_id AND status = 'active'

            UNION ALL

            -- 递归扩展
            SELECT
                l.source_object_id,
                l.target_object_id,
                l.id,
                pf.path || l.target_object_id,
                pf.link_path || l.id,
                pf.depth + 1
            FROM links l, path_finder pf
            WHERE (l.source_object_id = pf.next_node OR l.target_object_id = pf.next_node)
              AND l.target_object_id != ALL(pf.path)  -- 避免环路
              AND l.status = 'active'
              AND pf.depth < :max_hops
        )
        SELECT path, link_path, depth
        FROM path_finder
        WHERE next_node = :to_id
        ORDER BY depth
        LIMIT 10;
    """)

    result = await db.execute(query, {"from_id": from_id, "to_id": to_id, "max_hops": max_hops})
    paths = []
    for row in result:
        paths.append({
            "node_ids": [str(nid) for nid in row[0]],
            "link_ids": [str(lid) for lid in row[1]],
            "hops": row[2],
        })
    return {"paths": paths}
```

---

### Task 2: 前端图谱组件

```tsx
// frontend/src/components/RelationshipGraph.tsx
//
// 使用 react-force-graph-2d 或 d3-force 实现
//
// npm install react-force-graph-2d

"use client";
import ForceGraph2D from "react-force-graph-2d";
import { useState, useEffect, useCallback } from "react";

interface GraphProps {
  centerObjectId: string;
  depth?: number;
}

export default function RelationshipGraph({ centerObjectId, depth = 2 }: GraphProps) {
  const [graphData, setGraphData] = useState({ nodes: [], links: [] });
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    loadGraph(centerObjectId, depth);
  }, [centerObjectId, depth]);

  const loadGraph = async (objectId: string, d: number) => {
    setLoading(true);
    const resp = await fetch(`${API_BASE}/graph/neighborhood/${objectId}?depth=${d}&max_nodes=100`);
    const data = await resp.json();

    // 转换为 force-graph 格式
    setGraphData({
      nodes: data.nodes.map(n => ({
        id: n.id,
        name: n.name,
        type: n.type,
        val: n.id === centerObjectId ? 20 : 10, // 中心节点大
        color: getColorByType(n.type),
      })),
      links: data.edges.map(e => ({
        source: e.source,
        target: e.target,
        label: e.display_name,
        color: getColorByLinkType(e.type),
      })),
    });
    setLoading(false);
  };

  const handleNodeClick = useCallback((node: any) => {
    // 点击节点：展开该节点的关系（追加到图谱）
    loadGraph(node.id, 1);
  }, []);

  return (
    <div style={{ width: "100%", height: "600px" }}>
      {loading ? (
        <div>加载中...</div>
      ) : (
        <ForceGraph2D
          graphData={graphData}
          onNodeClick={handleNodeClick}
          nodeLabel="name"
          nodeVal="val"
          linkLabel="label"
          linkDirectionalArrowLength={3}
          linkDirectionalArrowRelPos={1}
          nodeCanvasObject={(node, ctx, globalScale) => {
            // 自定义节点渲染：圆形 + 名称标签
            const label = node.name as string;
            const fontSize = Math.max(12 / globalScale, 4);
            ctx.font = `${fontSize}px Sans-Serif`;
            ctx.textAlign = "center";
            ctx.textBaseline = "middle";
            ctx.fillStyle = node.color as string;
            ctx.beginPath();
            ctx.arc(node.x!, node.y!, 5, 0, 2 * Math.PI);
            ctx.fill();
            ctx.fillStyle = "#333";
            ctx.fillText(label, node.x!, node.y! + 8);
          }}
        />
      )}
    </div>
  );
}

function getColorByType(type: string): string {
  const colors: Record<string, string> = {
    person: "#4A90D9",
    government_org: "#E8913A",
  };
  return colors[type] || "#999";
}

function getColorByLinkType(type: string): string {
  const colors: Record<string, string> = {
    appointed_to: "#E8913A",
    political_ally: "#4A90D9",
    family: "#D94A4A",
  };
  return colors[type] || "#CCC";
}
```

---

### Task 3: 图谱页面

```tsx
// frontend/src/app/graph/page.tsx
//
// 布局：
// ┌──────────────────────────────────────────┐
// │  [搜索对象...]  [选择中心人物 ▼]          │
// │  [展开层数: 1 / 2 / 3]                   │
// ├──────────────────────────────────────────┤
// │                                          │
// │        ┌── 特朗普 ──┐                    │
// │       /    |    \     \                  │
// │   马斯克  蓬佩奥  卢比奥 万斯             │
// │     |      |                              │
// │   DOGE  国务院                            │
// │                                          │
// │  图例：● 人物 ● 部门 ─ 任命 ─ 盟友        │
// │                                          │
// │  提示：点击节点展开更多关系                 │
// └──────────────────────────────────────────┘
```

---

## 完成标准

- [ ] 图谱 API 返回正确的 nodes + edges 数据
- [ ] 前端能渲染出关系网络图
- [ ] 点击节点能展开下一层关系
- [ ] 关系类型用不同颜色区分
- [ ] 缩放和拖拽功能正常
- [ ] 100 节点内加载 < 2 秒

## 预计产出时间

2-3天
