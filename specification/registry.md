# 数字资产快照注册

数字资产快照注册（Catalog）是记录特定时间点资产实际状态的机制，与契约的**声明式注册**互补，提供资产的历史追溯能力。

## 两种注册方式对比

| 维度 | 声明式注册（契约） | 快照注册（目录） |
|------|------------------|----------------|
| 存储位置 | `contract.yaml` | `.quanttide/catalog/<timestamp>.json` |
| 维护方式 | 人工编辑 | 发现/验证后自动生成 |
| 内容 | 资产定义（应该有什么） | 资产快照（特定时间点实际有什么） |
| 版本控制 | 是 | 是（快照文件版本控制） |
| 持久性 | 长期 | 历史归档 |

## 快照注册与发现、验证的关系

```
┌─────────────────────────────────────────────────────────────┐
│                    声明式注册（契约）                         │
│              contract.yaml 中的 assets 清单                  │
│                    「应该有什么」                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              v
┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│   发现过程   │ --> │   验证过程       │ --> │  快照注册    │
│  (Discovery)│     │  (Validation)   │     │  (Catalog)  │
│  「有什么？」 │     │   「对吗？」     │     │「记录状态」  │
└─────────────┘     └─────────────────┘     └─────────────┘
                                                    │
                                                    v
                                         ┌─────────────────┐
                                         │  版本控制归档    │
                                         │  历史追溯       │
                                         └─────────────────┘
```

## 快照注册职责

### 1. 状态记录

记录资产在特定时间点的完整状态：

- `active`：验证通过，正常运行
- `invalid`：验证失败，存在问题
- `missing`：契约有声明但文件不存在
- `unexpected`：文件存在但契约未声明

### 2. 元数据归档

归档运行时产生的完整元数据：

- 快照时间戳
- 发现时间
- 验证时间
- 内容校验和
- 文件统计（数量、大小等）

### 3. 差异追溯

支持跨快照对比：

- 资产增减
- 状态变更
- 内容变化（通过校验和）

## 快照触发条件

| 触发条件 | 说明 | 示例 |
|---------|------|------|
| 版本发布 | 发布时自动归档 | `v1.0.0` 发布快照 |
| 验证完成 | 验证通过后归档 | CI/CD 流水线 |
| 定期归档 | 按计划手动触发 | 月度/季度审计 |
| 人工触发 | 重要节点手动归档 | 架构变更前 |

## 快照文件结构

```json
{
  "spec_version": "0.0.1",
  "generated_at": "2026-04-18T10:00:00Z",
  "snapshot_reason": "v1.0.0 发布归档",
  "source": {
    "contract_version": "1.2.0",
    "contract_checksum": "sha256:abc..."
  },
  "summary": {
    "total_assets": 10,
    "active": 8,
    "invalid": 1,
    "missing": 1,
    "unexpected": 0
  },
  "assets": {
    "brd": {
      "title": "商业需求文档",
      "type": "docs",
      "path": "docs/brd",
      "status": "active",
      "discovered_at": "2026-04-18T09:00:00Z",
      "verified_at": "2026-04-18T09:30:00Z",
      "checksum": "sha256:def...",
      "file_count": 5,
      "total_size": 10240
    }
  },
  "mismatches": {
    "missing": ["qa"],
    "unexpected": [],
    "invalid": [{
      "name": "deprecated_doc",
      "reason": "schema_validation_failed"
    }]
  }
}
```

## 快照配置

```yaml
# .quanttide/asset/contract.yaml
spec_version: 0.0.1
version: 1.0.0

assets:
  # ... 资产定义

# 快照配置（可选）
catalog:
  # 自动快照触发条件
  auto_snapshot:
    on_release: true           # 版本发布时
    on_validation_pass: false  # 验证通过时

  # 快照命名格式
  naming: "{timestamp}.json"   # 或 "{version}.json"

  # 保留策略
  retention:
    max_count: 50              # 最多保留 50 个快照
    max_age_days: 365          # 保留 1 年
    keep_releases: true        # 保留发布版本快照
```

## 与契约的关系

### 对比模式

```
快照 A (2026-04-01)          快照 B (2026-04-18)
┌─────────────┐              ┌─────────────┐
│ brd: active │              │ brd: active │
│ prd: active │   对比       │ prd: active │
│ qa: missing │   ------>    │ qa: active  │  <- 新增
│             │              │ api: invalid│  <- 新增
└─────────────┘              └─────────────┘
```

### 漂移检测

对比最新快照与契约：

```
契约定义          最新快照
┌─────────┐      ┌─────────┐
│ brd ✓   │      │ brd ✓   │
│ prd ✓   │      │ prd ✓   │
│ qa ✓    │  vs  │ qa ✗    │  <- 漂移！
│ ixd ✓   │      │ ixd ✓   │
└─────────┘      └─────────┘
```

## 存储实现

快照文件存储：

```
.quanttide/
├── asset/
│   └── contract.yaml
└── catalog/
    ├── 2026-04-01T000000Z.json
    ├── 2026-04-15T000000Z.json
    ├── 2026-04-18T100000Z.json   <- v1.0.0 发布
    └── latest.json -> 2026-04-18T100000Z.json  (符号链接)
```

- 快照文件纳入版本控制
- `latest.json` 符号链接指向最新快照（不版本控制）
- 支持按时间戳或版本号命名
