# Profile 数据建模参考

## 设计来源

Profile 资产目录的层次结构参考数据仓库建模体系。

## 层次映射

| 数据仓库 | Databricks Unity Catalog | Profile |
|----------|--------------------------|---------|
| 实例 / Metastore | Metastore | Profile 仓库 |
| 目录 | Catalog | 一级目录（资产目录） |
| 结构定义 | Schema | `schema.json` |
| 数据实例 | Table | `record.json` |

## Metastore 与租户

Metastore（元数据存储）是数据仓库体系中最顶层的容器，管理所有目录、Schema、表的结构信息。在 Unity Catalog 中，Metastore 是**多租户隔离边界**：

- 一个组织 / 云账号通常一个 Metastore
- 权限在 Metastore 或 Catalog 层级授予，向下继承
- 不同 Metastore 之间的资源不共享

对应到量潮资产体系，Metastore 即**租户**概念。现有租户包括：

| 租户 | 对应身份 |
|------|----------|
| `quanttide` | 开源主组织 |
| `quanttide-tech` | 量潮科技内部 |
| `quanttide-archive` | 归档站 |

Profile 仓库本身即扮演 Metastore / 租户级顶层容器，其下的一级资产目录对应 Catalog。

## 术语选择说明

使用 **record** 而非 table，是刻意表达半结构化优先的数据形态：

- Table 隐含固定 schema、严格列对齐的关系模型约束
- Record 对应文档型/半结构化数据，适应资产目录下形态多样的实例数据
- JSON 载体天然支持 Schema-on-read，与 record 语义一致

## 影响

后续工程标准中涉及资产目录结构定义时，应保持此层次映射和术语选择的一致性。
