# Java（Spring Boot）是什么

**Spring Boot** 是 Java 生态中最主流的后端开发框架，用来快速构建后端服务/API。

## 关系

```
Java       = 编程语言（类似 Python）
Spring     = Java 的后端框架（类似 Python 的 Django）
Spring Boot = Spring 的简化版（类似 Django 开箱即用的体验）
```

## 类比 Python

| Java 生态 | Python 对应 | 作用 |
|-----------|------------|------|
| Java | Python | 语言本身 |
| Spring Boot | Django / FastAPI | Web 后端框架 |
| Maven / Gradle | pip | 包管理 |
| MyBatis / JPA | SQLAlchemy / Django ORM | 数据库操作 |

## Spring Boot 能做什么

- REST API（给前端/小程序提供接口）
- 微服务（大型系统拆分成多个小服务）
- 后台管理系统的后端
- 定时任务、消息队列消费等

## 为什么这么流行

1. **企业级标配** — 国内大部分公司后端都用 Java + Spring Boot
2. **生态最完善** — 安全、数据库、缓存、消息队列，全都有现成方案
3. **招聘市场大** — Java 后端岗位数量远超其他语言
4. **稳定可靠** — 适合大型系统，经过大量生产验证

## 和 Python 后端比

| | Java + Spring Boot | Python + Django/FastAPI |
|--|-------------------|----------------------|
| **性能** | 更高 | 一般 |
| **开发速度** | 较慢（代码量多） | 快 |
| **代码风格** | 严谨冗长 | 简洁灵活 |
| **适合场景** | 大型企业系统、高并发 | 中小项目、AI/数据、快速原型 |
| **学习曲线** | 陡（Java 本身就比 Python 复杂） | 平缓 |

## 代码对比

Python FastAPI：
```python
@app.get("/hello")
def hello(name: str):
    return {"message": f"Hello, {name}"}
```

Java Spring Boot：
```java
@GetMapping("/hello")
public Map<String, String> hello(@RequestParam String name) {
    return Map.of("message", "Hello, " + name);
}
```

做的事情一样，Java 就是更啰嗦一些。

## 总结

**Spring Boot = Java 世界的 Django/FastAPI，是国内企业后端的第一选择。** 如果你将来做后端开发进公司，大概率会碰到它。
