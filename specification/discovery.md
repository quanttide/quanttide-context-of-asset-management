# 数字资产发现

数字资产发现是指系统通过扫描和分析项目结构，自动识别和提取数字资产的过程。

## 发现机制

### 发现触发条件

发现过程可由以下事件触发：

- **项目初始化**：新项目首次加载时自动扫描
- **文件变更**：监控文件系统变化，增量发现
- **手动触发**：用户主动发起发现请求
- **定时扫描**：按计划周期性扫描

### 发现流程

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   触发条件   │ -> │  扫描目录   │ -> │  解析契约   │ -> │  注册资产   │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

### 发现策略

#### 1. 契约优先发现

以 `.quanttide/asset/contract.yaml` 为事实源：

- 读取契约文件定义的资产清单
- 验证资产路径是否存在
- 标记缺失或新增的资产

#### 2. 文件系统发现

扫描项目目录结构：

- 识别标准资产目录（如 `docs/`、`src/`）
- 检测未在契约中声明的资产
- 生成候选资产列表供人工确认

#### 3. 混合发现模式

结合契约和文件系统：

1. 首先加载契约定义
2. 扫描实际文件系统
3. 对比差异并生成报告
4. 支持双向同步（契约 -> 文件系统，文件系统 -> 契约）

## 发现事件

发现过程中产生的事件：

| 事件 | 描述 |
|------|------|
| `asset.discovered` | 发现新资产 |
| `asset.missing` | 契约中定义但文件系统缺失 |
| `asset.modified` | 资产内容变更 |
| `asset.validated` | 资产验证完成 |

## 发现配置

发现配置统一放在 `.quanttide/asset/contract.yaml` 中，符合 Data Contract 风格，保持单一事实源。

### 单源发现配置（文件系统）

```yaml
# .quanttide/asset/contract.yaml
spec_version: 0.0.1
version: 1.0.0

assets:
  # ... 资产定义

discovery:
  # 扫描路径（相对于项目根目录）
  scan_paths:
    - docs
    - src
    - assets

  # 忽略模式
  ignore_patterns:
    - "*.tmp"
    - "node_modules/"
    - ".git/"

  # 自动发现开关
  auto_discovery: true

  # 定时扫描间隔（分钟）
  scan_interval: 60

  # 发现策略
  strategy: hybrid  # strict | filesystem | hybrid
```

### 多源发现配置（跨平台）

支持同时发现本地文件、GitHub 仓库、飞书文档等多源资产：

```yaml
# .quanttide/asset/contract.yaml
spec_version: 0.0.1
version: 1.0.0

# 多源资产定义
assets:
  # 本地文件系统资产
  brd_local:
    title: 商业需求文档（本地）
    type: docs
    category: brd
    source:
      type: filesystem
      path: docs/brd

  # GitHub 资产
  brd_github:
    title: 商业需求文档（GitHub）
    type: docs
    category: brd
    source:
      type: github
      repo: quanttide/qtcloud-asset
      path: docs/brd
      ref: main

  # 飞书文档资产
  prd_feishu:
    title: 产品需求文档（飞书）
    type: docs
    category: prd
    source:
      type: feishu
      doc_token: abc123
      doc_type: docx

discovery:
  # 发现适配器配置
  adapters:
    - type: filesystem
      enabled: true
      config:
        root: .

    - type: github
      enabled: true
      config:
        token: ${GITHUB_TOKEN}
        api_version: 2022-11-28

    - type: feishu
      enabled: true
      config:
        app_id: ${FEISHU_APP_ID}
        app_secret: ${FEISHU_APP_SECRET}
```

### 配置与契约的关系

```
┌─────────────────────────────────────────────────────────────┐
│              .quanttide/asset/contract.yaml                 │
│                                                             │
│  ┌─────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐  │
│  │  assets │  │ discovery │  │validation │  │  catalog  │  │
│  │(是什么) │  │(怎么发现) │  │(怎么验证) │  │(怎么归档) │  │
│  └────┬────┘  └─────┬─────┘  └───────────┘  └───────────┘  │
│       │             │                                        │
│       │      ┌──────┴──────┐                                 │
│       │      │  发现适配器  │                                 │
│       │      │  ├─ filesystem                               │
│       │      │  ├─ github                                   │
│       │      │  └─ feishu                                   │
│       │      └──────┬──────┘                                 │
│       │             │                                        │
│       └─────────────┼────────────────┐                       │
│                     v                v                       │
│            ┌─────────────┐    ┌─────────────┐               │
│            │  GitHub 资产 │    │  飞书文档   │               │
│            └─────────────┘    └─────────────┘               │
└─────────────────────────────────────────────────────────────┘
```

单一配置文件符合 Data Contract 理念：契约不仅定义数据结构，也定义数据产生的方式。多源发现配置支持跨平台资产统一治理。
