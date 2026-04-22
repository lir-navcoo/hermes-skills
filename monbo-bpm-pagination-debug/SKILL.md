---
name: monbo-bpm-pagination-debug
category: monbo-bpm
description: monbo-bpm 前端列表"暂无数据"问题排查——MyBatis-Plus分页返回格式与前端接口不匹配
---

# monbo-bpm 分页数据调试

## 症状
前端所有列表页面显示"暂无数据"，但API直接调用返回有数据（total>0, records有值）。

## 根本原因
MyBatis-Plus 分页返回格式：
```json
{"total": 6, "records": [...], "size": 10, "current": 1, "pages": 1}
```

前端 PageData 接口错误地定义为：
```typescript
// 错误 ❌
interface PageData<T> {
  list: T[];
  total: number;
  page: number;
  pageSize: number;
}
```

## 修复方法

### 1. 后端：添加 MybatisPlusConfig（如果不存在）
```java
package com.monbo.bpm.config;

import com.baomidou.mybatisplus.annotation.DbType;
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```

### 2. 前端：修正 PageData 接口
```typescript
// 正确 ✅
interface PageData<T> {
  records: T[];
  total: number;
  size: number;
  current: number;
  pages: number;
}
```

### 3. 前端：修正引用处
```typescript
// 改为：
setUsers(resp.data.records || []);
setTotal(resp.data.total || 0);
```

## 调试命令

```bash
# 测试登录
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"123456"}'

# 测试分页API（需带token）
curl http://localhost:8080/api/users?pageNum=1&pageSize=10 \
  -H "Authorization: Bearer <token>"

# 关键检查点
# 1. resp.data.total > 0 但 resp.data.records 为空 → MybatisPlusConfig未配置
# 2. resp.data.total === 0 → 数据库本身无数据或查询条件不对
# 3. resp.data.records 有值但前端不显示 → 前端字段名不匹配
```

## 相关文件
- 前端：`src/lib/api/user.ts` — PageData 接口定义
- 前端：`src/pages/users/UserListPage.tsx` — 数据引用处
- 后端：`src/main/java/com/monbo/bpm/config/MybatisPlusConfig.java`

## 预防
所有新增的列表页面，API返回的分页格式必须与 PageData 接口一致，或为新接口单独定义类型。
