---
name: mybatis-plus-logic-delete-gotcha
description: MyBatis-Plus @TableLogic 逻辑删除踩坑 — BaseMapper.delete(wrapper) 在逻辑删除表上静默失败的根因和修复
tags: [mybatis-plus, java, bug]
---

# MyBatis-Plus 逻辑删除踩坑

## 问题

实体配置了 `@TableLogic`（逻辑删除）后，`BaseMapper.delete(wrapper)` 会生成：
```sql
UPDATE table SET deleted=1 WHERE <wrapper条件> AND deleted=0
```
这导致**所有软删除表的 delete 操作必定失败**——wrapper 的 WHERE 和 `deleted=0` 永远同时满足，但 `deleted=1` 的行被 `deleted=0` 条件排除在外。

表现为：删除时既不抛错也不报错affected_rows=0，但数据实际未删除。

## 根因

`@TableLogic` 让 MyBatis-Plus 生成 `UPDATE ... SET deleted=1`，而 wrapper 默认追加 `deleted=0`，两者矛盾。

## 解决方案

对需要真正物理删除或清空关联表的场景，用原生 `@Select` 注解绕过 MyBatis-Plus：

```java
@Mapper
public interface UserRoleMapper extends BaseMapper<UserRole> {

    // 用这个替代 delete(wrapper)
    @Select("DELETE FROM mb_user_role WHERE role_id = #{roleId}")
    void deleteByRoleId(@Param("roleId") Long roleId);

    @Select("DELETE FROM mb_user_role WHERE user_id = #{userId}")
    void deleteByUserId(@Param("userId") Long userId);
}
```

## 适用场景

- 中间表（如 `mb_user_role`、`mb_department_user`）的清空操作
- 任何需要真删除（而非逻辑标记）的场景
- 删除前先清理关联数据的需求

## 验证方法

清空表后插入测试数据，调用删除方法，检查 `affected_rows` 是否>0，或查库确认数据已真正删除。

## 预防

在代码评审中检查所有 `mapper.delete(LambdaQueryWrapper/QueryWrapper)` 调用，若实体有 `@TableLogic`，一律要求改用原生 SQL。
