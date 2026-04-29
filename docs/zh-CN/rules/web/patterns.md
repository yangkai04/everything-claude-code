> 本文件在 [common/patterns.md](../common/patterns.md) 的基础上，补充 Web 专项模式。

# Web 模式（Web Patterns）

## 组件组合（Component Composition）

### 复合组件（Compound Components）

当关联 UI 共享状态和交互语义时，使用复合组件：

```tsx
<Tabs defaultValue="overview">
  <Tabs.List>
    <Tabs.Trigger value="overview">Overview</Tabs.Trigger>
    <Tabs.Trigger value="settings">Settings</Tabs.Trigger>
  </Tabs.List>
  <Tabs.Content value="overview">...</Tabs.Content>
  <Tabs.Content value="settings">...</Tabs.Content>
</Tabs>
```

- 父组件持有状态
- 子组件通过 Context 消费状态
- 对于复杂交互组件，优先使用此模式而非 prop drilling

### 渲染属性（Render Props）/ 插槽（Slots）

- 当行为共享但标记结构需要变化时，使用渲染属性或插槽模式
- 将键盘交互、ARIA 和焦点逻辑保留在无头层（headless layer）

### 容器与展示组件分离（Container / Presentational Split）

- 容器组件（Container）负责数据加载和副作用
- 展示组件（Presentational）接收 props 并渲染 UI
- 展示组件应保持纯函数（pure）

## 状态管理（State Management）

分类管理各类状态：

| 关注点 | 工具 |
|---------|---------|
| 服务端状态（Server state） | TanStack Query、SWR、tRPC |
| 客户端状态（Client state） | Zustand、Jotai、signals |
| URL 状态（URL state） | search params、route segments |
| 表单状态（Form state） | React Hook Form 或同类工具 |

- 不要将服务端状态复制到客户端 store
- 优先派生（derive）值，而非存储冗余的计算状态

## URL 即状态（URL As State）

将可共享的状态持久化到 URL：
- 筛选条件（filters）
- 排序顺序（sort order）
- 分页（pagination）
- 当前激活的 tab
- 搜索关键词（search query）

## 数据获取（Data Fetching）

### 过时数据重验证（Stale-While-Revalidate）

- 立即返回缓存数据
- 在后台重新验证（revalidate）
- 优先使用现有库，不要手动实现

### 乐观更新（Optimistic Updates）

- 快照当前状态
- 应用乐观更新
- 失败时回滚（roll back）
- 回滚时向用户展示可见的错误反馈

### 并行加载（Parallel Loading）

- 并行获取相互独立的数据
- 避免父子请求瀑布流（waterfall）
- 在合理时预取（prefetch）可能访问的下一个路由或状态
