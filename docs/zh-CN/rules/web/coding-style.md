> 本文件在 [common/coding-style.md](../common/coding-style.md) 的基础上，补充 Web 前端专项内容。

# Web 编码风格（Web Coding Style）

## 文件组织

按功能或页面区域组织，而非按文件类型：

```text
src/
├── components/
│   ├── hero/
│   │   ├── Hero.tsx
│   │   ├── HeroVisual.tsx
│   │   └── hero.css
│   ├── scrolly-section/
│   │   ├── ScrollySection.tsx
│   │   ├── StickyVisual.tsx
│   │   └── scrolly.css
│   └── ui/
│       ├── Button.tsx
│       ├── SurfaceCard.tsx
│       └── AnimatedText.tsx
├── hooks/
│   ├── useReducedMotion.ts
│   └── useScrollProgress.ts
├── lib/
│   ├── animation.ts
│   └── color.ts
└── styles/
    ├── tokens.css
    ├── typography.css
    └── global.css
```

## CSS 自定义属性（CSS Custom Properties）

将设计 token 定义为变量，不要反复硬编码调色板、排版或间距：

```css
:root {
  --color-surface: oklch(98% 0 0);
  --color-text: oklch(18% 0 0);
  --color-accent: oklch(68% 0.21 250);

  --text-base: clamp(1rem, 0.92rem + 0.4vw, 1.125rem);
  --text-hero: clamp(3rem, 1rem + 7vw, 8rem);

  --space-section: clamp(4rem, 3rem + 5vw, 10rem);

  --duration-fast: 150ms;
  --duration-normal: 300ms;
  --ease-out-expo: cubic-bezier(0.16, 1, 0.3, 1);
}
```

## 仅使用合成器友好的动画属性

优先使用合成器友好的运动属性：
- `transform`
- `opacity`
- `clip-path`
- `filter`（谨慎使用）

避免对以下布局相关属性做动画：
- `width`
- `height`
- `top`
- `left`
- `margin`
- `padding`
- `border`
- `font-size`

## 语义化 HTML 优先（Semantic HTML First）

```html
<header>
  <nav aria-label="Main navigation">...</nav>
</header>
<main>
  <section aria-labelledby="hero-heading">
    <h1 id="hero-heading">...</h1>
  </section>
</main>
<footer>...</footer>
```

当存在语义化元素时，不要用通用的 `div` 嵌套来代替。

## 命名规范

- 组件：PascalCase（`ScrollySection`、`SurfaceCard`）
- Hook：`use` 前缀（`useReducedMotion`）
- CSS 类名：kebab-case 或工具类
- 动画时间轴：camelCase 且带意图描述（`heroRevealTl`）
