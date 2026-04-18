# 架构留白：为演化预留空间

真正的平台化治理不是由架构师拍脑袋定规则，而是**"观察 -> 归纳 -> 规范 -> 自动化"**的演进过程。

## 克制：高级架构师的标志

对"干预"和"评分"的克制，体现了不仅关注技术可行性，更关注**技术的社会学**（开发者文化）和时间维度的演化。

这种"清醒"保证了 qtcloud-asset 不会变成沉重的、充满官僚气息的低效系统，而是一个能够**自我呼吸、随业务增长的动态生态**。

## 避开的坑：过早标准化

**过早标准化（Premature Standardization）**是平台化治理的最大陷阱。

### 反模式

```
架构师定义评分标准 → 强制推行 → 开发者抵触 → 平台废弃
```

### 正模式

```
Provider 运行积累数据 → 观察真实使用模式 → 归纳健康特征 → 生成评分标准 → 开发者认可
```

## 三步演进：从观测到度量

### 第一步：数据积累（望远镜）

Provider 初期角色是"数字资产的物理望远镜"，只观测不评判：

- 开发者在 `contract.yaml` 里最常写什么字段
- 哪些资产被频繁引用
- 哪些资产在 Discovery 阶段经常报错
- 多副本资产的同步延迟分布

```python
# 观测数据模型（不留白，只记录）
class AssetObservation(BaseModel):
    asset_id: str
    project_id: str
    observed_at: datetime
    # 原始数据，不做判断
    contract_fields: List[str]      # 实际使用的字段
    discovery_errors: List[str]     # 发现的错误
    sync_latency_ms: int            # 同步延迟
    access_frequency: int           # 访问频率
```

### 第二步：特征提取（归纳）

通过真实数据，反推什么是"健康的资产"：

- 高频复用的技能是否都有完整的输入输出定义？
- 健康的资产平均有多少个字段？
- 稳定的资产变更频率是多少？

```python
# 特征分析（定期运行，不强制）
async def analyze_asset_health(project_id: str) -> HealthReport:
    observations = await get_observations(project_id, days=30)
    
    # 归纳特征，不评判
    return HealthReport(
        avg_field_count=statistics.mean([o.field_count for o in observations]),
        common_patterns=extract_patterns(observations),
        anomaly_cases=detect_outliers(observations)  # 异常但不做处理
    )
```

### 第三步：标准生成（规范）

基于数据生成评分标准，开发者觉得"这是在帮我"：

```python
# 基于数据生成的评分规则（非强制）
class GovernanceRule(BaseModel):
    id: str
    name: str
    description: str
    source: str  # "derived_from_data" | "manual" | "community"
    confidence: float  # 数据支持的置信度
    severity: str  # suggestion | warning | error（渐进）
    enabled: bool = False  # 默认关闭，由项目启用
```

## 干预分级：从辅助到自动

干预涉及对开发者**主权**（代码库）的侵入，必须分级演化：

| 级别 | 名称 | 描述 | 适用阶段 |
|------|------|------|---------|
| L1 | 信息通知 | 发现不一致，在 CLI 提醒 | 初期 |
| L2 | 建议操作 | 提供 `qtcloud-asset fix` 命令，开发者手动确认 | 中期 |
| L3 | 受控干预 | Provider 自动提交 PR，开发者 Merge | 成熟期 |
| L4 | 自动对齐 | 仅限非核心资产或高度标准化场景 | 完全信任后 |

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
   
2. 为 config 添加 description
   执行：qtcloud-asset fix --asset config --action add-desc

$ qtcloud-asset fix --asset api_doc --action sync --dry-run
预览修复结果...（不实际执行）

$ qtcloud-asset fix --asset api_doc --action sync
确认执行？ [y/N]: y
已修复
```

### L3 示例：受控干预

```python
# Provider 自动提交 PR
async def create_fix_pr(project_id: str, fixes: List[Fix]) -> PullRequest:
    branch = f"qtcloud-asset/fix-{uuid4()[:8]}"
    
    # 创建分支并提交修复
    await create_branch(project.repo_url, branch)
    for fix in fixes:
        await commit_fix(project.repo_url, branch, fix)
    
    # 创建 PR，由开发者 Merge
    pr = await create_pull_request(
        repo=project.repo_url,
        title="[qtcloud-asset] 自动修复资产不一致",
        body=generate_pr_description(fixes),
        branch=branch
    )
    
    # 通知开发者
    await notify(project.owner, f"请审核自动修复 PR: {pr.url}")
    
    return pr
```

## 架构留白：预留扩展点

### Compliance Metadata（合规元数据）

JSONB 字段记录资产与契约的匹配细节，为未来评分引擎存储中间结果：

```python
class AssetCompliance(BaseModel):
    asset_id: str
    project_id: str
    
    # 原始匹配结果，不做判断
    metadata: Dict[str, Any] = Field(default_factory=dict, sa_column=Column(JSONB))
    
    # 示例内容（未来评分引擎使用）
    # {
    #   "field_coverage": 0.85,      # 字段覆盖率
    #   "schema_compliance": true,   # 是否符合 schema
    #   "cross_source_sync": {
    #     "github": "2026-04-18T10:00:00Z",
    #     "feishu": "2026-04-18T08:00:00Z",
    #     "lag_hours": 2
    #   },
    #   "access_patterns": {
    #     "last_read": "2026-04-18T09:00:00Z",
    #     "read_frequency": "daily"
    #   }
    # }
    
    calculated_at: datetime
    calculation_version: str  # 评分算法版本
```

### Activity Logs（活动日志）

详细记录资产变更频率，为未来干预策略提供样本：

```python
class AssetActivity(BaseModel):
    asset_id: str
    project_id: str
    
    # 变更事件
    event_type: str  # created | updated | deleted | synced | validated
    event_source: str  # cli | provider | github_webhook | feishu_webhook
    
    # 变更详情
    before_state: Optional[Dict] = Field(default=None, sa_column=Column(JSONB))
    after_state: Optional[Dict] = Field(default=None, sa_column=Column(JSONB))
    diff: Optional[Dict] = Field(default=None, sa_column=Column(JSONB))
    
    # 时间戳
    timestamp: datetime
    actor: str  # user_id | system | automation
    
    # 上下文
    session_id: Optional[str]  # 关联到操作会话
    correlation_id: Optional[str]  # 分布式追踪
```

## 数据库模型预留

```python
# 核心模型（必须）
class Project(BaseModel): ...
class Contract(BaseModel): ...
class CatalogSnapshot(BaseModel): ...

# 观测模型（留白，初期可为空表）
class AssetObservation(BaseModel): ...  # 观测数据
class AssetCompliance(BaseModel): ...   # 合规元数据
class AssetActivity(BaseModel): ...     # 活动日志
class HealthReport(BaseModel): ...      # 健康报告

# 治理模型（留白，初期不启用）
class GovernanceRule(BaseModel): ...    # 评分规则
class GovernanceScore(BaseModel): ...   # 评分结果
class AutoFixJob(BaseModel): ...        # 自动修复任务
```

## 接口预留

```python
# 核心接口（必须实现）
@app.get("/api/v1/projects/{project_id}/contract")
@app.post("/api/v1/projects/{project_id}/discovery")
@app.get("/api/v1/projects/{project_id}/snapshots")

# 观测接口（留白，返回空或模拟数据）
@app.get("/api/v1/projects/{project_id}/observations")  # 观测数据
@app.get("/api/v1/projects/{project_id}/health")        # 健康报告（初期返回空）

# 治理接口（留白，返回 501 Not Implemented）
@app.get("/api/v1/projects/{project_id}/score")         # 评分（预留）
@app.post("/api/v1/projects/{project_id}/fix")          # 自动修复（预留）
```

## 总结

**架构不是空中楼阁，没有基准的评分是主观的，没有实践的干预是冒进的。**

通过三步演进（观测 -> 归纳 -> 规范）和四级干预（通知 -> 建议 -> 受控 -> 自动），qtcloud-asset 能够在保持开发者信任的同时，逐步建立治理权威。

**留白不是空白，而是为未来的可能性预留空间。**
