# 数据发现

如果你认为 AWS Glue 的 Crawler（爬虫）设计得比较“笨重”或“黑盒”，那么微软 Purview 的 **扫描规则集 (Scan Rule Sets)** 确实代表了另一种更精细化、更具治理导向的设计思路。

在微软的设计中，发现（Discovery）不是简单的“探测”，而是一个**规则驱动的映射过程**。以下是微软 Purview 扫描规则集的官方参考资料及核心设计逻辑，这应该正是你寻找的“发现活动配置文件”的高阶参考。

### 1\. 官方资料链接

  * **[Microsoft Purview 扫描规则集概念 (Official)](https://www.google.com/search?q=https://learn.microsoft.com/zh-cn/azure/purview/concept-scans-and-ingestion)**：详细介绍了如何定义扫描、摄取和分类规则。
  * **[如何创建扫描规则集 (Step-by-Step)](https://www.google.com/search?q=https://learn.microsoft.com/zh-cn/azure/purview/create-a-scan-rule-set)**：包含了不同数据源（如 SQL, ADLS Gen2）的可选配置项。
  * **[Purview 自动分类规则 (Classification Rules)](https://learn.microsoft.com/zh-cn/azure/purview/concept-classification)**：这是发现活动中最核心的“逻辑配置”，定义了如何通过模式匹配发现资产属性。

-----

### 2\. 微软“扫描规则集”配置文件核心结构

微软将发现活动解耦为：**数据源 (Source) + 扫描规则集 (Scan Rule Set) + 扫描作业 (Scan Job)**。

#### A. 文件类型选择 (File Types)

与 Glue 试图自动识别一切不同，Purview 允许你在配置文件中硬性规定扫描范围：

  * **支持的扩展名:** 显式指定 `.csv`, `.parquet`, `.json`, `.avro` 等。
  * **自定义类型:** 允许定义特定的定界符或自定义文件解析规则。

#### B. 分类规则映射 (Classification Rules)

这是 Purview 的核心优势，也是建立高质量 Catalog 的关键：

  * **系统规则:** 自动发现 200+ 种全球敏感信息（如身份证、信用卡）。
  * **自定义规则:** \* **正则表达式 (Regex):** 比如 `^CUST-[0-9]{8}$` 匹配客户编码。
      * **字典匹配:** 匹配特定的业务词汇表。
  * **置信度阈值 (Threshold):** 只有当样本中匹配比例超过特定值（如 60%）时，才在 Catalog 中自动标注该属性。

#### C. 扫描频率与增量逻辑 (Scanning Behavior)

  * **增量扫描 (Incremental):** 仅扫描自上次成功发现以来更改过的文件。
  * **采样 (Sampling):** 配置文件可以指定每个文件夹扫描的前 $N$ 个文件或每个文件的前 $M$ 行，以平衡发现深度与资源成本。

-----

### 3\. 为什么这种设计更符合“发现活动”的需求？

微软的设计逻辑实际上是将 **"发现"** 从一个 **"工程探测任务"** 提升到了 **"策略审计任务"**：

1.  **确定性高：** 你可以预判扫描后 Catalog 会产生哪些标签，而不是等 Glue Crawler 跑完后去清理它乱撞出来的“未知表名”。
2.  **解耦设计：** \* **Glue:** 一个 Crawler 绑定一个目标、一组逻辑和一套运行环境。
      * **Purview:** 扫描规则集是独立的“配置文件实体”，可以复用到多个不同的数据源上，确保全公司对“敏感数据”的发现标准是一致的。
3.  **结果导向：** 它不仅发现“这里有个文件”，还发现“这里有符合 GDPR 要求的个人信息”，直接完成了从物理资产到合规资产的跃迁。

### 4\. 建议：参考 Purview 设计你的 Profile 字段

如果你在设计自己的发现活动配置文件，可以借鉴 Purview 的以下 JSON 结构片段（简化版）：

```json
{
  "scanRuleSetName": "Financial_Data_Discovery_Standard",
  "sourceType": "AzureStorage",
  "fileCriteria": {
    "includePatterns": ["finance/**/*.parquet"],
    "excludePatterns": ["**/temp/**"]
  },
  "classificationRules": {
    "useSystemRules": true,
    "customRuleIds": ["Tax_ID_Rule", "Internal_Project_Code"]
  },
  "sampling": {
    "type": "Full",
    "thresholdLines": 1000
  }
}
```


基于 **Microsoft Purview** 和 **Google Dataplex** 的设计逻辑，一个成熟的“发现活动配置文件”不再只是简单的连接参数，而是一套**元数据推断与分类治理的策略引擎**。

为了建立高质量的 Catalog，你的发现配置文件（Discovery Profile）应包含以下核心字段：

---

### 1. 扫描范围与过滤字段 (Scope & Filter)
定义“去哪里找”以及“避开哪里”，这是确保扫描性能的关键。
* **Include/Exclude Patterns (包含/排除模式):** 使用通配符（如 `bucket/logs/**`）排除临时文件、归档数据或日志。
* **Asset Depth (扫描深度):** 规定扫描的目录层级。
* **Integration Type (集成类型):** 是全量扫描（Full Scan）还是增量扫描（Incremental Scan，仅处理自上次发现后的变更）。

### 2. 元数据推断规则 (Metadata Inference Rules)
决定“如何理解”非结构化或半结构化数据。
* **Schema Inference (架构推断):** 是否允许系统通过读取文件头部来自动生成字段名和类型。
* **Partition Discovery (分区发现):** 规定如何将类似 `/year=2024/month=03/` 的路径识别为表的分区，而不是成千上万个独立文件。
* **File Grouping (文件分组):** 定义“多文件即一张表”的逻辑（如将一个文件夹下所有的 Parquet 文件视为一个数据集）。

### 3. 自动分类与打标规则 (Classification & Tagging)
这是发现活动的“灵魂”，直接决定了 Catalog 的搜索体验。
* **Classification Rulesets (分类规则集):**
    * **System Rules:** 调用内置规则（如身份证号、手机号、信用卡号的正则模型）。
    * **Custom Rules:** 自定义业务规则（如公司内部的 `Project_ID` 格式）。
* **Confidence Threshold (置信度阈值):** > 例如：设定为 **60%**。意味着如果一个列中超过 60% 的数据符合“手机号”模式，Catalog 才自动将其标注为“敏感信息”。
* **Classification Depth (采样深度):** 每张表采样多少行进行内容匹配（如前 1000 行）。

### 4. 目录同步与冲突策略 (Catalog Sync Strategy)
发现结果如何“写入”现有的 Catalog。
* **Schema Evolution (架构演变策略):** * `Allow`: 发现新字段自动加入 Catalog。
    * `Block`: 发现变更时不更新 Catalog，仅报错。
* **Stale Data Policy (陈旧数据处理):** 如果源头数据已删除，Catalog 中的资产是直接删除、标记为“已过期”，还是保留元数据？
* **Owner Assignment (归属分配):** 发现资产后，根据路径自动映射负责人（如 `/hr/**` 下的资产自动分配给 HR 部门。

---

### 5. 发现活动元数据 (Discovery Activity Metadata)
记录活动本身的审计信息。
* **Trigger Type (触发类型):** 手动执行、定时任务、或由 Data Event（数据变更事件）触发。
* **Scan Identity (执行身份):** 运行发现活动的 Service Account 权限。
* **Resource Limits (资源限制):** 限制发现任务占用的计算单元（如 Dataplex 的 DCU 限制），防止扫描导致源库宕机。

---

### 对比参考表：设计你的配置项

| 字段类别 | 核心字段示例 (Schema Key) | 作用描述 |
| :--- | :--- | :--- |
| **基础信息** | `discovery_rule_id`, `source_type` | 标识规则及数据源（如 S3, SQL Server） |
| **规则选择** | `scan_ruleset_name` | 引用预定义的规则集（如“财务审计扫描集”） |
| **采样控制** | `sampling_rate`, `row_limit` | 控制成本，避免全量扫描带来的高额费用 |
| **结果处理** | `auto_publish_to_catalog` | 发现结果是直接上线，还是进入“待审核”状态 |
| **质量预检** | `dq_profile_enabled` | 在发现的同时是否进行空值率、基数等统计分析 |

**建议设计思路：**
不要把所有的逻辑都塞进一个大 JSON。参考微软，将 **"扫描规则"（Scan Ruleset）** 定义为一个独立实体，而在具体的 **"扫描任务"（Scan Job）** 中去引用它。这样当你更新了“敏感数据识别规则”时，所有关联的发现活动都会同步更新。



这个问题可能需要一个具体的参考表，而不是一篇论述文。我试着把“发现配置”变成一张可以直接对着填的清单。以下配置的核心设计逻辑，完全基于你的数据湖思想来源——**Schema-on-Read**，即在发现资产时通过多维元数据标签动态构建视图，而非依赖事先锁定的文件夹路径。

---

### 发现配置清单 (可直接用 Markdown 表格做模板)

#### 1. 发现入口层配置 (Discovery Entry)

决定用户从什么维度进来搜索。

| 配置项 | 字段名 (示例) | 数据湖思想说明 |
| :--- | :--- | :--- |
| **发现范围** | `discovery.scope` | 默认发现“所有已编目资产”（类似数据湖允许全量扫描），也可限定于特定数据域（Data Zone）。 |
| **发现通道** | `discovery.channels` | **多模态索引**：支持同时搜索“文本（聊天记录/文档）、图像、代码库、会议音频转写等”。通道之间逻辑是“OR”。 |
| **过滤钩子** | `discovery.filters` | **标签预筛选**：支持按“项目、部门、负责人、资产密级”等标签快速过滤搜索结果。 |
| **排序权重** | `discovery.sort` | **智能加权**：优先显示被高频引用的案例（热度高），或最近更新的资产（新鲜度高），或AI语义相似度高的资产。 |

#### 2. 关系发现层配置 (Relationship Discovery)

决定搜索结果出来后，系统自动“联想”出什么关联信息。

| 配置项 | 字段名 (示例) | 数据湖思想说明 |
| :--- | :--- | :--- |
| **显式关联** | `relationship.explicit` | **主动引用**：本资产在正文中直接链接或提及了哪些其他资产ID（如“请参考 QD-2026-001 号判例”）。 |
| **隐式关联** | `relationship.implicit` | **血缘与谱系**：基于 ETL 数据血缘自动推导依赖关系（如“生成此报告的上游数据表/程序”）。 |
| **业务关联** | `relationship.business` | **术语/口径绑定**：将资产与集团统一的标准业务术语关联（如“客户活跃度”指向资产A和B）。 |
| **AI 上下文** | `relationship.context` | **大模型 RAG 推理**：AI 分析资产内容后判断强相关（如“此营销案例与当时某条例修订相关”），写入 AI 上下文索引。 |

#### 3. 发现结果层配置 (Discovery Result)

决定找到的资产条目展示哪些关键字段。

| 配置项 | 字段名 (示例) | 数据湖思想说明 |
| :--- | :--- | :--- |
| **技术元数据** | `meta.tech` | **系统自动采集**：存储路径（如 `lakehouse/catalog/schema/asset_name`）、文件类型、大小、创建时间、SHA256。 |
| **业务元数据** | `meta.business` | **人工/流程注册**：资产名称、简介、**自定义标签集**（`owner=张三`, `case_type=判例`, `module=财务`）。 |
| **管理元数据** | `meta.governance` | **权限与合规**：访问权限、密级分类（如 L3 机密）、数据保留周期（如 10 年）、数据责任人。 |
| **关系元数据** | `meta.relation` | **上下游依赖**：预览显示“上游：2 个数据表；下游：3 个报表模型”，帮助快速理解影响力。 |

#### 4. 反馈闭环配置 (Feedback Loop)

决定如何利用用户行为优化未来的发现结果。

| 配置项 | 字段名 (示例) | 数据湖思想说明 |
| :--- | :--- | :--- |
| **用户动作** | `feedback.action` | **记录元数据**：将“点击、收藏、复制链接、引用”等行为作为特殊元数据写入资产描述中。 |
| **发现优化** | `feedback.optimize` | **协同过滤推荐**：基于用户行为日志实现“看过此资产的人也看过……”，影响后续发现排序。 |

你可以把这张表直接作为 **Markdown 配置文档** 用。最关键的一步是，在资产入库注册时就通过自定义模板强制填写或自动采集这些元数据。一旦这些“标签地基”打好了，发现程序的动态构建能力才会被真正激活。
