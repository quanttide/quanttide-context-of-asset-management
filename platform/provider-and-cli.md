# 平台化治理模式

当引入 Provider 服务端集中管理 Catalog 和 Contract 后，GitHub 仓库中的 `contract.yaml` 与 CLI 的角色需要重新定位。

## 两种治理模式

### 模式一：分布式治理（Git 优先）

```
GitHub 仓库
├── .quanttide/asset/contract.yaml    # 契约事实源
└── .quanttide/catalog/<timestamp>.json  # 快照归档

Provider 服务端
├── 读取仓库契约
├── 生成快照并回写仓库
└── 提供 API 查询
```

**特点**：
- 契约存储在 Git 仓库，版本控制
- Provider 是计算服务，不存储数据
- CLI 直接操作本地文件

### 模式二：平台化治理（服务端优先）

```
Provider 服务端
├── Contract 数据库              # 契约事实源
├── Catalog 历史库               # 快照历史
└── 项目-仓库映射关系

GitHub 仓库
├── .quanttide/asset/contract.yaml  # 导出副本（可选）
└── 仅作为代码存储
```

**特点**：
- 契约存储在服务端数据库
- GitHub 仓库可导出契约副本
- CLI 通过 API 操作

## 推荐方案：混合模式

```
┌─────────────────────────────────────────────────────────────┐
│                    Provider 服务端                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   Contract  │  │   Catalog   │  │   项目-仓库映射      │ │
│  │   数据库    │  │   历史库    │  │   (Project-Repo)    │ │
│  │  (事实源)   │  │  (快照历史)  │  │                     │ │
│  └──────┬──────┘  └─────────────┘  └─────────────────────┘ │
│         │                                                   │
│         │  同步                                              │
│         v                                                   │
│  ┌─────────────┐                                            │
│  │  导出/导入   │                                            │
│  │  (可选)     │                                            │
│  └──────┬──────┘                                            │
└─────────┼───────────────────────────────────────────────────┘
          │
          │  webhook / API
          v
┌─────────────────────────────────────────────────────────────┐
│                    GitHub 仓库                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   代码资产   │  │ contract.yaml│  │   catalog/          │ │
│  │   (src/)    │  │  (导出副本)  │  │   (可选归档)        │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
          ^
          │  git push / pull
          │
┌─────────┴───────────────────────────────────────────────────┐
│                      CLI 客户端                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │  本地开发   │  │  缓存契约   │  │   批量操作          │ │
│  │  (离线模式) │  │  (在线模式) │  │   (导入/导出)       │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 契约存储策略

### 事实源：服务端数据库

Provider 服务端存储契约的权威版本：

```python
# Provider 数据库模型
class Project(BaseModel):
    id: str
    name: str
    repo_url: str              # GitHub 仓库地址
    contract: Contract         # 契约 JSON
    contract_version: str
    contract_digest: str       # 内容摘要，用于冲突检测
    updated_at: datetime
    updated_by: str            # 更新者标识
```

### 副本：GitHub 仓库（可选）

仓库中的 `contract.yaml` 是服务端导出的副本：

```yaml
# .quanttide/asset/contract.yaml
# 此文件由 Provider 服务端导出，请勿手动修改
# 最后导出时间: 2026-04-18T10:00:00Z
# 服务端版本: v1.2.0
# 内容摘要: sha256:abc123...

metadata:
  managed_by: provider
  digest: sha256:abc123...
  exported_at: 2026-04-18T10:00:00Z

spec_version: 0.0.1
version: 1.2.0

assets:
  # ... 资产定义
```

### 同步机制

| 触发条件 | 动作 | 方向 |
|---------|------|------|
| 服务端契约更新 | 导出到仓库 | 服务端 → GitHub |
| 仓库推送 | 触发同步检查 | GitHub → 服务端 |
| 手动导入 | 导入仓库契约 | GitHub → 服务端 |

### 冲突解决

当服务端数据库与 GitHub 副本不一致时：

```
场景：开发者手动修改了仓库 contract.yaml，同时服务端也被 Web 界面修改

检测：通过 digest 比对发现冲突
    │
    ├── 策略一：服务端优先（默认）
    │   - 拒绝仓库推送
    │   - 提示执行 `qtcloud-asset contract pull`
    │
    ├── 策略二：仓库优先
    │   - 以仓库版本为准覆盖服务端
    │   - 需显式指定 `--force-from-repo`
    │
    └── 策略三：人工合并
        - 标记冲突状态
        - 开发者手动解决后重新提交
```

**Check API 拦截**：在 PR 阶段验证契约一致性

```yaml
# .github/workflows/contract-check.yml
steps:
  - name: Contract Consistency Check
    uses: quanttide/contract-check-action@v1
    with:
      provider_url: https://provider.qtcloud.io
      project_id: qtcloud-asset
```

## CLI 协调策略

### 在线模式（默认）

CLI 通过 API 与服务端交互：

```bash
# 读取契约（从服务端）
qtcloud-asset contract get --project qtcloud-asset

# 更新契约（提交到服务端）
qtcloud-asset contract update --file contract.yaml

# 触发发现
qtcloud-asset discovery start --project qtcloud-asset

# 查询快照
qtcloud-asset snapshot list --project qtcloud-asset
```

### 离线模式

CLI 直接操作本地文件（原 CLI 行为）：

```bash
# 本地发现
qtcloud-asset discovery start --local

# 本地验证
qtcloud-asset validate --local

# 稍后同步到服务端
qtcloud-asset sync push
```

### 混合模式

CLI 缓存服务端契约，支持离线编辑：

```bash
# 拉取契约到本地缓存
qtcloud-asset contract pull

# 离线编辑...

# 推送更新到服务端
qtcloud-asset contract push

# 服务端发现后，拉取快照
qtcloud-asset snapshot pull
```

## 增强设计

### 增量快照存储

Catalog 历史库采用全量 + 增量策略：

```python
class CatalogSnapshot(BaseModel):
    id: str
    project_id: str
    snapshot_type: str  # full | incremental
    base_snapshot: Optional[str]  # 增量快照依赖的全量快照 ID
    diff: Optional[Dict]  # 差异数据
    status: str  # healthy | drift | broken
    created_at: datetime
```

**归档策略**：
- 数据库存储：最近 30 天全量 + 增量
- GitHub 归档：仅里程碑版本（发布、审计）
- 过期清理：自动删除 90 天前的非里程碑快照

### 治理策略下发

服务端存储 Governance Rules，CLI 本地校验时拉取：

```yaml
# 服务端存储的治理规则
rules:
  - id: required_fields
    name: 必填字段检查
    severity: error
    enabled: true
  
  - id: path_exists
    name: 路径存在性检查
    severity: warning
    enabled: true
```

```bash
# CLI 本地校验时先拉取规则
qtcloud-asset validate --local --sync-rules
```

### 安全与权限

**API Key 体系**：

```bash
# CLI 配置访问令牌
qtcloud-asset config set token $QT_ACCESS_TOKEN
```

**Webhook 安全**：

```python
# 验证 Webhook 签名
import hmac
import hashlib

def verify_webhook(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)
```

## 使用场景

### 场景一：团队协作（在线模式）

- 开发者通过 CLI 或 Web 界面修改契约
- 变更实时同步到服务端
- 服务端触发发现和验证
- 快照历史集中管理

### 场景二：本地开发（离线模式）

- 开发者本地编辑契约
- 本地运行发现和验证
- 完成后批量推送到服务端
- 适合网络不稳定环境

### 场景三：CI/CD 集成

```yaml
# .github/workflows/asset-governance.yml
steps:
  - name: Checkout
    uses: actions/checkout@v4
  
  - name: Install CLI
    run: pip install qtcloud-asset-cli
  
  - name: Import Contract
    run: qtcloud-asset contract import --project qtcloud-asset
  
  - name: Trigger Discovery
    run: qtcloud-asset discovery start --project qtcloud-asset --wait
  
  - name: Check Consistency
    run: qtcloud-asset snapshot verify --project qtcloud-asset --latest
```

## 数据流

```
Developer -> CLI -> Provider API -> Contract DB
                       |
                       v
                Discovery Service
                       |
        ┌──────────────┼──────────────┐
        v              v              v
   Filesystem      GitHub         Feishu
        │              │              │
        └──────────────┼──────────────┘
                       v
                Catalog Snapshot
                       |
                       v
                Catalog History DB
```

## 关键决策

| 决策点 | 选择 | 理由 |
|--------|------|------|
| 契约事实源 | 服务端数据库 | 支持实时协作、权限控制 |
| 仓库契约文件 | 导出副本 | 保留 Git 可见性，但非权威 |
| CLI 默认模式 | 在线模式 | 简化使用，实时反馈 |
| 离线支持 | 缓存机制 | 兼顾网络不稳定场景 |
| 快照存储 | 服务端历史库 | 集中管理，便于查询对比 |
| 冲突解决 | 服务端优先 + Check API | 保证权威性的同时提供反馈 |
| 快照归档 | 全量 + 增量 | 平衡存储成本与查询效率 |
