---
title: Nextjs + HeroUI 工程
date: 2026-06-08 00:00:00 +0800
categories: [nextjs]
tags: [nextjs, heroui]
author: caohongchuan
pin: false
math: true
toc: true
comments: true
mermaid: true
---

> Nextjs 官方文档：https://nextjs.org/docs/app/getting-started/installation#quick-start
>
> HeroUI官方文档：https://heroui.com/en/docs/react/getting-started/quick-start

## 创建Nextjs工程

```bash
npx create-next-app@latest my-app --yes
cd my-app
npm run dev
```

## 添加HeroUI依赖

```bash
npm i @heroui/styles @heroui/react
```

在 `globals.css` 中导入 HeroUI 的CSS

```css
@import "tailwindcss";
@import "@heroui/styles"; 
```

引入测试

```typescript
import { Button } from '@heroui/react';

function App() {
  return (
    <Button>
      My Button
    </Button>
  );
}
```

