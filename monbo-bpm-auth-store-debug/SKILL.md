---
name: monbo-bpm-auth-store-debug
description: monbo-bpm 登录后闪退的根因：两个独立 auth store 导致 token 读写不一致
category: monbo-bpm
---

# monbo-bpm Auth Store 不一致问题

## 问题描述
登录成功，但页面立即闪退到登录页（线上/生产环境必现）。

## 根因
项目存在两个独立的 auth store：

| 文件 | 用途 |
|------|------|
| `src/store/auth.ts` | 登录页写入，Router/ProtectedRoute 读取 |
| `src/lib/stores/authStore.ts` | 另一个独立实现，API client 读取 |

API 请求带的是 `lib/stores/authStore` 的 token（null），所以每次 API 调用都 401 → logout → 跳转登录。

## 修复
修改 `src/lib/api/client.ts`，改为引用 `@/store/auth`（登录写入的那个）：

```typescript
// 动态导入避免循环依赖
async function getAuthToken(): Promise<string | null> {
  const { useAuthStore } = await import('@/store/auth');
  return useAuthStore.getState().token ?? null;
}

async function doLogout() {
  const { useAuthStore } = await import('@/store/auth');
  useAuthStore.getState().logout();
  window.location.href = '/login';
}
```

## 经验
- 401 闪退先查 auth store 引用链：登录写入哪个 → API client 读取哪个
- 多 store 文件是危险信号，应尽早统一
