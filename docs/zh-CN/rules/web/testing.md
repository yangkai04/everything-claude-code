> 本文件在 [common/testing.md](../common/testing.md) 的基础上，补充 Web 专项测试内容。

# Web 测试规则（Web Testing Rules）

## 优先级顺序

### 1. 视觉回归（Visual Regression）

- 截图关键断点：320、768、1024、1440
- 测试 hero 区域、滚动叙事（scrollytelling）区域及重要状态
- 视觉密集型工作使用 Playwright 截图
- 如果存在多个主题（theme），每个都需要测试

### 2. 无障碍（Accessibility）

- 运行自动化无障碍检查
- 测试键盘导航
- 验证减少动画（reduced-motion）行为
- 验证色彩对比度

### 3. 性能（Performance）

- 针对重要页面运行 Lighthouse 或同类工具
- 维持 [performance.md](performance.md) 中的 Core Web Vitals（CWV）目标

### 4. 跨浏览器（Cross-Browser）

- 最低要求：Chrome、Firefox、Safari
- 测试滚动、动效及降级行为（fallback behavior）

### 5. 响应式（Responsive）

- 测试断点：320、375、768、1024、1440、1920
- 验证无溢出
- 验证触摸交互

## E2E 示例

```ts
import { test, expect } from '@playwright/test';

test('landing hero loads', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('h1')).toBeVisible();
});
```

- 避免基于超时的不稳定断言（flaky assertions）
- 优先使用确定性等待（deterministic waits）

## 单元测试（Unit Tests）

- 测试工具函数、数据转换和自定义 Hook
- 对于视觉性很强的组件，视觉回归通常比脆弱的标记断言更有价值
- 视觉回归是覆盖率目标的补充，而非替代
