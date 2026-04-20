# 架构设计文档

从治理哲学、演化策略和系统健壮性三个维度进行专业评估。

### 核心评价：高内聚的"治理闭环"

#### 1. 理论高度：契约驱动 (Contract-First)

通过区分 `Contract`（意图/规范）和 `Catalog`（现实/快照），架构具备了**自我审计**的能力。

#### 2. 演化策略：渐进式平台化 (Progressive Platformization)

"混合模式"极具工程智慧。保留 CLI 离线能力保证了开发体验（DX），引入 Provider 解决了组织层面的合规与协作。

#### 3. 维度广度：多源统一治理

将本地 Git、GitHub 和飞书文档纳入同一套治理框架。

### 技术拆解

| 维度 | 评价等级 | 关键表现 |
| :--- | :--- | :--- |
| **扩展性** | **极高** | 技能定义（SkillConfig）的设计预留了未来接口 |
| **健壮性** | **高** | ETag/Digest 解决分布式冲突 |
| **工程性** | **高** | FastAPI + Pydantic 实现模型复用 |
| **闭环能力** | **极高** | 从发现、验证到快照注册覆盖资产全生命周期 |

## 架构留白

真正的平台化治理不是由架构师拍脑袋定规则，而是**"观察 -> 归纳 -> 规范 -> 自动化"**的演进过程。

### 克制：高级架构师的标志

对"干预"和"评分"的克制，体现了不仅关注技术可行性，更关注**技术的社会学**。

### 避开的坑：过早标准化

**反模式**：
```
架构师定义评分标准 → 强制推行 → 开发者抵触 → 平台废弃
```

**正模式**：
```
Provider 运行积累数据 → 观察真实使用模式 → 归纳健康特征 → 生成评分标准 → 开发者认可
```

### 三步演进：从观测到度量

#### 第一步：数据积累（望远镜）

Provider 初期角色是"数字资产的物理望远镜"，只观测不评判：

- 开发者在 `contract.yaml` 里最常写什么字段
- 哪些资产被频繁引用
- 哪些资产在 Discovery 阶段经常报错

#### 第二步：特征提取（归纳）

通过真实数据，反推什么是"健康的资产"。

#### 第三步：标准生成（规范）

基于数据生成评分标准，开发者觉得"这是在帮我"。

### 干预分级：从辅助到自动

| 级别 | 名称 | 描述 | 适用阶段 |
|------|------|------|---------|
| L1 | 信息通知 | 发现不一致，在 CLI 提醒 | 初期 |
| L2 | 建议操作 | 提供 `qtcloud-asset fix` 命令 | 中期 |
| L3 | 受控干预 | Provider 自动提交 PR | 成熟期 |
| L4 | 自动对齐 | 仅限非核心资产 | 完全信任后 |

### L1 示例：信息通知

```bash
$ qtcloud-asset status
⚠️  发现 2 处不一致：
   1. Asset 'api_doc' 在飞书有更新，GitHub 未同步
   2. Asset 'config' 缺失 description 字段
   
   运行 `qtcloud-asset suggest` 查看修复建议
```

### L2 示例：建议操作

```bash
$ qtcloud-asset suggest
建议修复：
1. 同步 api_doc 到 GitHub
   执行：qtcloud-asset fix --asset api_doc --action sync
```

### L3 示例：受控干预

Provider 自动提交 PR，由开发者 Merge：

```python
async def create_fix_pr(project_id: str, fixes: List[Fix]) -> PullRequest:
    branch = f"qtcloud-asset/fix-{uuid4()[:8]}"
    await create_branch(project.repo_url, branch)
    for fix in fixes:
        await commit_fix(project.repo_url, branch, fix)
    
    pr = await create_pull_request(
        repo=project.repo_url,
        title="[qtcloud-asset] 自动修复资产不一致",
        body=generate_pr_description(fixes),
        branch=branch
    )
    
    await notify(project.owner, f"请审核自动修复 PR: {pr.url}")
    return pr
```

## 数据库模型预留

```python
# 核心模型（必须）
class Project(BaseModel): ...
class Contract(BaseModel): ...
class CatalogSnapshot(BaseModel): ...

# 观测模型（留白，初期可为空表）
class AssetObservation(BaseModel): ...
class AssetCompliance(BaseModel): ...
class AssetActivity(BaseModel): ...

# 治理模型（留白，初期不启用）
class GovernanceRule(BaseModel): ...
class GovernanceScore(BaseModel): ...
```

## 接口预留

```python
# 核心接口（必须实现）
@app.get("/api/v1/projects/{project_id}/contract")
@app.post("/api/v1/projects/{project_id}/discovery")
@app.get("/api/v1/projects/{project_id}/snapshots")

# 观测接口（留白，返回空或模拟数据）
@app.get("/api/v1/projects/{project_id}/observations")
@app.get("/api/v1/projects/{project_id}/health")

# 治理接口（留白，返回 501 Not Implemented）
@app.get("/api/v1/projects/{project_id}/score")
@app.post("/api/v1/projects/{project_id}/fix")
```

## 元模型兼容性

### 资产治理的四根支柱

```
┌─────────────────────────────────────────┐
│           资产治理元模型                  │
│                                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐ │
│  │  标识   │  │  属性   │  │  关系   │ │
│  │(Who)   │  │(What)  │  │(Where) │ │
│  └────┬────┘  └────┬────┘  └────┬────┘ │
│       └─────────────┼─────────────┘     │
│                     │                   │
│                     v                   │
│              ┌─────────┐                │
│              │  验证   │                 │
│              │(How)   │                 │
│              └─────────┘                │
│                                         │
└─────────────────────────────────────────┘
```

### 与 SPDX 的兼容性

量潮架构的 `metadata` 采用开放键值对结构：

```yaml
assets:
  lib_axios:
    type: code
    metadata:
      spdx:
        license_concluded: MIT
        package_checksum: sha256:abc123...
```

### 与 Backstage 的兼容性

`contract.yaml` 定义的是资产的**契约主权**，Backstage 可以看作 Provider 的"展现层插件"：

```python
class BackstageAdapter:
    def convert_to_entity(self, asset: Asset) -> BackstageEntity:
        return BackstageEntity(
            apiVersion="backstage.io/v1alpha1",
            kind=self.map_kind(asset.type),
            metadata={
                "name": asset.name,
                "annotations": {
                    "qtcloud.io/contract-uri": asset.uri
                }
            }
        )
```

### 兼容性的两种做法

| 方式 | 层级 | 特征 |
|------|------|------|
| 打补丁式兼容 | 低级 | 为支持 A、B、C 协议，写一堆适配代码 |
| 协议抽象式兼容 | 高级 | 理解资产治理的本质是"标识、属性、关系、验证" |

## 进阶建议

### 1. 治理成本透明化

CLI 端实时显示 Governance Score：

```bash
$ qtcloud-asset status
Project: qtcloud-asset
Governance Score: 85/100
⚠️  3 assets missing description
```

### 2. Schema 版本语义化

- `spec_version` 遵循语义化版本（SemVer）
- 服务端同时支持多个 spec 版本
- 不兼容时提示升级

### 3. 从"感知"到"干预"

当发现 Catalog 与 Contract 不一致时，服务端自动生成 PR：

```python
async def auto_fix(project_id: str, mismatch: Mismatch):
    if mismatch.type == "missing_in_repo":
        pr = await create_pr(
            repo=project.repo_url,
            title=f"[Auto] Add missing asset: {mismatch.asset_name}",
            changes=[{
                "path": mismatch.expected_path,
                "content": generate_stub(mismatch.asset_type)
            }]
        )
        return pr.url
```

### 4. 事件流抽象

将所有 Discovery 结果抽象为"事件流"：

```python
class DiscoveryEvent(BaseModel):
    event_type: str  # asset_found | asset_missing | asset_modified
    asset_id: str
    project_id: str
    timestamp: datetime
    severity: str  # info | warning | error

class EventBus:
    def subscribe(self, event_type: str, handler: Callable):
        self.handlers.setdefault(event_type, []).append(handler)
    
    async def publish(self, event: DiscoveryEvent):
        for handler in self.handlers.get(event.event_type, []):
            await handler(event)

bus = EventBus()
bus.subscribe("mismatch_detected", notify_feishu)
bus.subscribe("mismatch_detected", update_dashboard)
```

### 5. 契约版本快照

对每一次 `qtcloud-asset contract push` 进行快照存储：

```python
class ContractVersion(BaseModel):
    id: str
    project_id: str
    contract: Contract
    version: str
    digest: str
    change_type: str  # major | minor | patch
    committed_by: str
    committed_at: datetime
    status: str  # active | rolled_back | deprecated
```

```bash
qtcloud-asset contract history --project qtcloud-asset
qtcloud-asset contract diff 1.2.0 1.3.0
qtcloud-asset contract rollback 1.2.0
```
