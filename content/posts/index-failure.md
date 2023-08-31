---
title: "索引失效场景"
date: 2023-08-31T07:56:57+08:00
draft: false
tags: ["MySQL"]
---
## 不等于
```sql
SELECT * FROM users WHERE name != "jack";
```
## 使用函数
```sql
SELECT * FROM users WHERE UPPER(name) = "JACK";
```

## OR 条件
```sql
SELECT * FROM users WHERE name = "jack" or age = 10;
```

## 最左匹配原则
```sql
-- 联合索引 (name, age, gender)
SELECT * FROM users WHERE name = "jack" and age = 10;
```

## 隐式转换
```sql
SELECT * FROM users WHERE age = "10";
```

## LIKE 以通配符开头
```sql
SELECT * FROM users WHERE name LIKE "%jack";
```
