# P5.2 体验优化

> 目标：Agent 采集流式展示、图谱交互优化、移动端适配、加载性能。

---

## 依赖

- P1-P4 全部完成

## 任务清单

### Task 1: Agent 采集流式展示

```python
# 将 Celery 任务轮询改为 WebSocket 推送

# backend/app/api/agent.py
from fastapi import WebSocket

@router.websocket("/ws/tasks/{task_id}")
async def task_websocket(websocket: WebSocket, task_id: str):
    """WebSocket 实时推送任务进度"""
    await websocket.accept()
    from ..celery_app import celery_app

    last_state = None
    while True:
        result = celery_app.AsyncResult(task_id)
        current_state = {
            "state": result.state,
            "meta": result.info if result.info else {},
        }

        if current_state != last_state:
            await websocket.send_json(current_state)
            last_state = current_state

        if result.state in ("DONE", "FAILED"):
            break

        await asyncio.sleep(1)

    await websocket.close()
```

```tsx
// frontend/src/lib/useAgentTask.ts — 改为 WebSocket
export function useAgentTaskWS(taskId: string | null) {
  const [status, setStatus] = useState<any>(null);

  useEffect(() => {
    if (!taskId) return;

    const ws = new WebSocket(`ws://localhost:8000/api/v1/agent/ws/tasks/${taskId}`);
    ws.onmessage = (event) => {
      setStatus(JSON.parse(event.data));
    };
    ws.onclose = () => setStatus(prev => ({ ...prev, isPolling: false }));

    return () => ws.close();
  }, [taskId]);

  return { status };
}
```

---

### Task 2: 图谱交互优化

```
优化项：
1. 节点分层布局（中心人物居中，按关系距离排列）
2. 节点 hover 显示 tooltip（名称 + 关键属性）
3. 双击节点跳转到详情页
4. 单击节点展开下一层关系（动画过渡）
5. 搜索定位（输入名称，高亮并定位到节点）
6. 图例面板（可点击图例筛选关系类型）
7. 导出图片功能（可选）
```

---

### Task 3: 加载性能

```
优化项：
1. API 分页优化（对象列表默认 limit=20）
2. 图谱数据懒加载（先加载1层，点击再展开）
3. 前端数据缓存（SWR / React Query）
4. 图片/资源 CDN（如果有静态资源）
5. API 响应压缩（gzip）
```

```typescript
// 使用 SWR 做前端缓存
import useSWR from 'swr';

export function useObjects(type?: string) {
  const params = type ? `?type=${type}` : '';
  return useSWR(`${API_BASE}/objects${params}`, fetcher, {
    revalidateOnFocus: false,
    dedupingInterval: 5000,
  });
}
```

---

### Task 4: 移动端基本适配

```
适配项：
1. 侧边栏改为底部导航栏（移动端）
2. 对象卡片从网格改为单列
3. 图谱支持触摸缩放/拖拽
4. 审核队列改为上下布局（移动端）
5. 对话界面全屏宽度
```

---

## 完成标准

- [ ] Agent 采集过程实时展示（WebSocket，≤2秒延迟）
- [ ] 图谱 100 节点内操作流畅（无卡顿）
- [ ] 对象列表首屏加载 ≤ 1秒
- [ ] 移动端可正常浏览和操作

## 预计产出时间

2-3天
