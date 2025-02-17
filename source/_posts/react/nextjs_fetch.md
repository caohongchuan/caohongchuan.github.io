---
title: Nextjs前后端交互
category: 
  - frontend
---

> 前后端交互使用Javascript。目前底层原生工具有`fetch`和`XMLHttpRequest`。开发常用的`axios`是对`fetch`的封装。这两种通信方式都是HTTP/HTTPS协议，即无状态协议。

跨域问题：

`Fetch` 如果后端不支持CORS，前端是无法实现跨域访问的。

```typescript
const headers = new Headers();
headers.set("Content-Type", "application/json");
const request = new Request(process.env.NEXT_PUBLIC_LOGIN_API!, {
    method: "POST",
    mode: "cors",
    headers: headers,
});
const response = await fetch(request);
if (!response.ok) {
    throw new Error(`Response status ${response.status}`);
}
const json = await response.json();
```

