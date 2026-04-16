# 资产一致性审查记录

## 2026-04-17 平台资产与语境资产对齐

### 操作概述

对平台资产（qtcloud-asset）与语境资产（docs/context）进行对比和同步，确保语境文档与产品定义保持一致。

### 活动一：PRD 对比

| 资产 | 路径 | 角色 |
|------|------|------|
| 平台资产 | `apps/qtcloud-asset/docs/prd/*` | 产品 PRD，用户故事驱动，包含验收标准和业务规则 |
| 语境资产 | `docs/context/platform/prd.md` | 对 BRD/Tutorial 代沟的分析文章 |

**结论**：平台 PRD 已从描述性文档重构为用户故事格式（Given-When-Then），每个模块对应语境 PRD 提出的三层代沟问题。两者互补：语境 PRD 是"为什么做"，平台 PRD 是"做什么"。

### 活动二：BRD 同步

| 资产 | 路径 | 角色 |
|------|------|------|
| 平台资产 | `apps/qtcloud-asset/.agents/skills/product-brd/SKILL.md` | BRD 编写技能规范，定义三改变、问题四要素、假设句式 |
| 语境资产 | `docs/context/platform/brd.md` | BRD 上下文文档 |

**操作**：将 SKILL.md 中的 BRD 标准结构（三改变、边界、问题四要素、假设句式、角色定义）注入 brd.md，使其从"只分析代沟"升级为"既定义标准又分析代沟"。

### 活动三：资产重命名

将 `docs/context/platform.md` 重命名为 `docs/context/platform/brd.md`，建立清晰的资产分类边界。

### 资产关系

```
platform/
├── brd.md  ← 业务需求定义（来自 product-brd SKILL）
└── prd.md  ← 产品需求实现（对应 apps/qtcloud-asset/docs/prd/*）
```
