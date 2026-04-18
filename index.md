# 量潮数字资产云 - 语境文档

本文档记录 qtcloud-asset 项目的设计语境、架构决策和演化思路。

## 核心设计

### 数字资产治理

围绕**数字资产契约**建立的一整套治理机制：

- [契约结构](specification/contract/schema.md) - 契约文件格式与字段定义
- [资产定义](specification/contract/asset.md) - 资产的组成与属性
- [技能定义](specification/contract/skill.md) - 可复用能力的封装
- [发现机制](specification/discovery.md) - 自动识别和提取资产
- [验证机制](specification/validation.md) - 确认资产完整性
- [快照注册](specification/registry.md) - 记录资产历史状态

### 多源资产治理

支持同时治理本地文件、GitHub 仓库、飞书文档等多种来源：

- [多源资产](specification/multi-source-assets.md) - 单源、多副本、跨平台组合
- [Catalog vs Contract](specification/catalog-vs-contract.md) - 快照与契约的区别

### 平台化治理

从 CLI 本地工具向 Provider 平台服务的演化：

- [Provider 与 CLI](platform/provider-and-cli.md) - 混合治理模式设计
- [演化路线图](platform/evolution-roadmap.md) - 三阶段演化计划
- [演化本质](platform/evolution-essence.md) - 从个人工具到组织基础设施
- [架构评估](platform/architecture-evaluation.md) - 第三方专业评估（A+ 级）
- [架构留白](platform/architecture-leave-blank.md) - 为演化预留空间的设计哲学
- [成熟度评估](platform/maturity-model-assessment.md) - 工业级治理架构典范（9.3/10）
- [层级评估](platform/architecture-level-assessment.md) - L7 巅峰，向 L8 突破（行业定义者）
- [元模型兼容性](platform/meta-model-compatibility.md) - 超越行业标准的治理主权
- [确定性哲学](platform/determinism-philosophy.md) - 认知模型的确定性（与 Borg、Chaos Engineering 并列）
- [Contract Monkey](platform/contract-monkey.md) - 治理混沌工程（动态韧性测试）

## 设计原则

1. **契约即事实源** - 契约定义资产的"应该是什么"
2. **渐进式平台化** - 保持 CLI 可用，逐步迁移能力到服务端
3. **多源统一治理** - 本地、GitHub、飞书等多种来源统一管理
4. **快照历史追溯** - Catalog 记录资产变更历史，支持审计

## 关键决策

| 决策 | 选择 | 理由 |
|------|------|------|
| 契约事实源 | 服务端数据库 | 支持实时协作、权限控制 |
| CLI 默认模式 | 在线模式 | 简化使用，实时反馈 |
| 快照存储 | 服务端历史库 | 集中管理，便于查询对比 |
| 冲突解决 | 服务端优先 | 保证权威性 |

## 相关文档

- [产品需求文档](../../docs/prd/) - 产品层面的需求定义
- [架构设计文档](../../docs/add/) - 技术实现细节
