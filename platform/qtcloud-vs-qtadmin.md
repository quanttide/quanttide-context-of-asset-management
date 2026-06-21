# qtcloud-asset 与 qtadmin asset 整合思路

## 对比

| 维度 | qtcloud-asset | qtadmin asset |
|------|---------------|---------------|
| 定位 | 数字资产管理基础设施 | 管理后台的资产职能域 |
| 形态 | 独立平台（CLI + Studio + Provider） | CLI 子命令 |
| 技术栈 | Python CLI / Flutter Studio / FastAPI | Rust CLI |
| 核心能力 | 契约定义、归档执行、AI 路线图 | archive / status / quality |
| 交付形式 | 云平台（阿里云 FC/OSS） | 本地 CLI 工具 |

## 功能重叠

**archive** 两者都有：

- qtcloud-asset：按 contract.yaml 配置的路径映射归档产品日志
- qtadmin：按日期（默认 3 天前）扫描 docs/journal/ 下的 .md 文件，归档到 docs/archive/journal/，自动处理 git 子模块提交

## 整合方向

### archive 统一

qtadmin 的 archive 实现更通用（日期驱动、自动 git 操作），可作为底层执行引擎。qtcloud-asset 的 contract.yaml 配置作为上层策略输入。

### status 互补

qtadmin status 已实现 Git 仓库结构合规检查（必需文件、CHANGELOG 格式、提交规范）。可扩展为增加对 `.quanttide/asset/contract.yaml` 的契约-目录一致性校验。

### quality 引入

qtadmin quality 的语义质量评估（叙事/知识/认知三维度）为 qtcloud-asset 所无，可作为新增能力引入。

## 整体闭环

```
定义契约  →  结构合规检查  →  语义质量评估  →  日志归档
   ↑                                            ↓
   └─────────── qtcloud-asset 平台 ─────────────┘
```

qtadmin asset CLI 是执行层，qtcloud-asset 是定义层 + 平台层。
