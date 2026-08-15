# clickhouse-document

> ClickHouse 文档、学习笔记与实践记录。

## 简介

本仓库用于整理 ClickHouse 相关的技术文档，包括：

- 📚 核心概念与架构笔记
- 🛠️ 运维与部署实践
- ⚡ 性能调优经验
- 📝 SQL 语法与函数速查
- 🧪 使用案例与踩坑记录

## 目录结构

```
clickhouse-document/
├── README.md           # 项目说明
├── CHANGELOG.md        # 变更日志
├── interview/          # 🎯 面试备考资料（下方有详细导航）
├── docs/               # 文档主目录
│   ├── concepts/       # 核心概念
│   ├── operations/     # 运维实践
│   ├── tuning/         # 性能调优
│   ├── sql/            # SQL 与函数
│   └── cases/          # 实践案例
└── examples/           # 示例脚本与配置
```

## 🎯 面试备考资料（interview/）

> 一套完整的 ClickHouse 生产实战面试备考资料：从 CPU 底层原理到系统设计，共 5 份文档，约 20 万字。
>
> 📱 **手机在线阅读（带侧边栏 + 目录跳转 + 全文搜索）**：[qihuavision.github.io/clickhouse-document](https://qihuavision.github.io/clickhouse-document/)

| 资料 | 内容 | 适用场景 |
|---|---|---|
| [生产实战面试题主文档](interview/ClickHouse生产实战面试题_原理与概念.md) | 27 道大题（标准答案+追问+踩分点）+ 12 张原理图解 + 3 道系统设计大题 + 18 条陷阱快查表 | 系统复习主入口 |
| [Q1 深度：列存 CPU 底层原理](interview/Q1深度讲解_列存CPU底层原理.md) | CPU 缓存/向量化执行/SIMD/晚物化/压缩，从硅片讲到 SQL | 搞懂"为什么快" |
| [Q2Q3 深度：稀疏索引与磁盘 IO](interview/Q2Q3深度讲解_稀疏索引与磁盘IO底层.md) | B+Tree 对比、granule 查询字节流、排序键单调性数学 | 索引专题 |
| [图6 深度：复制日志与分布式共识](interview/图6深度讲解_复制日志与分布式共识底层.md) | quorum 数学、Raft、复制状态机、确定性合并 | 分布式专题 |
| [模拟面试复盘：生产实战 5 题](interview/模拟面试复盘_生产实战5题.md) | Too many parts / 副本延迟 / 查询突慢 / Replacing 重复 / 迁移灵魂拷问 | 面前最后突击 |

### 建议复习路径（3 天）

- **第 1 天**：Q1 深度讲解（CPU 层）→ 回看主文档图 2、图 3 巩固
- **第 2 天**：Q2Q3 深度讲解（磁盘 IO 层）→ 重点背第 4 章字节流 + 第 8 章单调性
- **第 3 天**：图 6 深度讲解（分布式层）→ 第 4 章 INSERT 时序 + 第 6 章确定性合并
- **之后**：主文档 27 题 + 系统设计三道题快速过一遍 → 考前只看模拟面试复盘

## 关于 ClickHouse

[ClickHouse](https://clickhouse.com/) 是一个用于在线分析处理（OLAP）的开源列式数据库管理系统，具备高性能、线性可扩展、硬件高效等特点，广泛应用于日志分析、用户行为分析、实时数仓等场景。

---

> 📌 本仓库为个人维护的学习文档项目，欢迎提 Issue 交流。
