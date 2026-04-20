# 开发实现文档

## CLI 到 Provider 的演化路线图

从"本地工具"向"平台服务"的跨越，分四个阶段实现平稳演化。

## 演化的本质

CLI 到 Provider 的演化，本质是**"渐进式平台化"**——在保持现有价值的基础上，逐步将能力从边缘（CLI）迁移到中心（Provider），实现治理的规模化。

### 1. 权力转移：从分散到集中

| 阶段 | 权力分布 | 本质 |
|------|---------|------|
| CLI 时代 | 每个开发者本地决策 | 去中心化治理 |
| 演化过程 | CLI + Provider 双轨运行 | 过渡期妥协 |
| 平台时代 | 服务端统一决策 | 中心化治理 |

### 2. 数据流反转

```
CLI 时代：开发者 -> 本地修改 -> git push -> 仓库
平台时代：开发者 -> 服务端修改 -> 自动同步 -> 仓库
```

### 3. 计算迁移

| 计算任务 | CLI 时代 | 平台时代 |
|---------|---------|---------|
| 发现 | 本地扫描磁盘 | 服务端扫描多源 |
| 验证 | 本地规则校验 | 服务端策略引擎 |
| 快照 | 本地文件存储 | 服务端历史库 |
| 治理 | 人工检查 | 自动化策略执行 |

### 4. 渐进策略

| 风险 | 激进方案 | 渐进方案 |
|------|---------|---------|
| 开发者抵触 | 强制切换 | 双轨运行 |
| 数据丢失 | 一次性迁移 | 增量同步 |
| 网络依赖 | 完全在线 | 保留本地模式 |

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

### 推荐方案：混合模式

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
```

## 第一阶段：API 化与身份认证（建立连接）

让 CLI 具备"感知"服务端的能力，解决"你是谁"的问题。

### 1.1 服务端配置

```yaml
# ~/.quanttide/config.yaml
endpoint: https://provider.qtcloud.io/api/v1
access_token: qt_xxxxxxxxxxxx

# 项目映射
projects:
  qtcloud-asset:
    local_path: ~/projects/qtcloud-asset
    remote_id: proj_abc123
```

### 1.2 身份校验

```bash
# 浏览器 OAuth 登录
qtcloud-asset login

# 或使用 API Key
qtcloud-asset login --token $QT_ACCESS_TOKEN
```

### 1.3 Repository 接口抽象

```python
# repository/base.py
class Repository(ABC):
    @abstractmethod
    def get_contract(self) -> Contract:
        pass
    
    @abstractmethod
    def update_contract(self, contract: Contract) -> None:
        pass

# repository/local.py
class LocalRepository(Repository):
    """操作本地文件（现有逻辑）"""
    def __init__(self, project_path: str):
        self.project_path = project_path
    
    def get_contract(self) -> Contract:
        with open(f"{self.project_path}/.quanttide/asset/contract.yaml") as f:
            return Contract.parse_yaml(f.read())

# repository/remote.py
class RemoteRepository(Repository):
    """调用 Provider API（新增逻辑）"""
    def __init__(self, endpoint: str, token: str, project_id: str):
        self.client = ProviderClient(endpoint, token)
        self.project_id = project_id
    
    def get_contract(self) -> Contract:
        return self.client.get_contract(self.project_id)
```

### 1.4 自动切换逻辑

```python
# CLI 自动选择 Repository
def get_repository(project_path: str) -> Repository:
    config = load_config()
    
    if config.has_remote(project_path):
        return RemoteRepository(
            endpoint=config.endpoint,
            token=config.access_token,
            project_id=config.get_project_id(project_path)
        )
    else:
        return LocalRepository(project_path)
```

## 第二阶段：契约事实源迁移（同步逻辑）

实现"混合模式"，将服务端设为权威，同时保留 Git 可见性。

### 2.1 契约拉取 (Pull)

```bash
qtcloud-asset contract pull
```

### 2.2 契约推送 (Push)

```bash
qtcloud-asset contract push
```

### 2.3 Webhook 闭环

```python
# provider/webhook_handler.py
@app.post("/webhook/github")
async def handle_github_webhook(request: Request):
    payload = await request.json()
    
    if payload["ref"] == "refs/heads/main":
        project = get_project_by_repo(payload["repository"]["full_name"])
        contract_yaml = fetch_contract_from_github(project.repo_url)
        
        contract = Contract.parse_yaml(contract_yaml)
        if validate_contract(contract):
            project.update_contract(contract)
            return {"status": "synced"}
        else:
            create_check_run(project.repo_url, status="failure")
            return {"status": "validation_failed"}
```

### 2.4 冲突解决

```
场景：开发者手动修改了仓库 contract.yaml，同时服务端也被 Web 界面修改

检测：通过 digest 比比发现冲突
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

## 第三阶段：能力重心上移（发现与治理演化）

将重计算任务迁移到服务端，CLI 作为"指令发起者"。

### 3.1 服务端发现 (Remote Discovery)

```bash
# CLI 仅作为指令发起者
qtcloud-asset discovery --project qtcloud-asset

# 输出：
# Starting remote discovery...
# Job ID: job_abc123
# Discovery completed!
# Assets found: 15
# Status: healthy
```

### 3.2 快照版本化

```python
# provider/snapshot_service.py
async def create_snapshot(project_id: str, discovery_result: DiscoveryResult):
    last_snapshot = get_latest_snapshot(project_id)
    diff = compute_diff(last_snapshot, discovery_result)
    
    snapshot = CatalogSnapshot(
        project_id=project_id,
        type="incremental",
        base_snapshot=last_snapshot.id if last_snapshot else None,
        diff=diff,
        status="healthy" if diff.no_critical_changes else "drift"
    )
    
    save_snapshot(snapshot)
```

### 3.3 治理策略下发

```yaml
# 服务端存储的治理规则
governance_rules:
  - id: required_description
    name: 资产必须包含描述
    severity: error
    check: "asset.description is not None"
  
  - id: path_pattern
    name: 路径必须符合规范
    severity: error
    check: "asset.path matches '^[a-z0-9_/-]+$'"
```

```bash
qtcloud-asset validate --local --sync-rules
```

## 第四阶段：治理度量与 CI 集成（质量门禁）

引入契约覆盖率概念，支持 CI 流程中的质量门禁。

### 4.1 契约覆盖率检查

```bash
qtcloud-asset check
qtcloud-asset check --fail-under=80
qtcloud-asset check --asset-coverage --fail-under=90
```

### 4.2 GitHub Check Run 集成

```python
# provider/github_integration.py
async def create_coverage_check_run(
    project_id: str,
    pr_number: int,
    coverage_report: CoverageReport
):
    conclusion = "success" if coverage_report.overall >= 0.8 else "failure"
    
    await github_client.create_check_run(
        repo=project.repo_url,
        name="QuantTIDE Asset Coverage",
        head_sha=pr.head_sha,
        status="completed",
        conclusion=conclusion,
        output={
            "title": f"Asset Coverage: {coverage_report.overall:.1%}",
            "summary": generate_summary(coverage_report)
        }
    )
```

### 4.3 覆盖率趋势追踪

服务端存储历史数据，Dashboard 展示趋势图：

```
资产覆盖率趋势 (最近 30 天)

100% │                                    ╭─╮
 90% │              ╭──╮    ╭──╮    ╭────╯ ╰──╮
 80% │    ╭──╮  ╭──╯  ╰────╯  ╰────╯          ╰── 当前: 87.5%
 70% │────╯  ╰──╯
     └────┬────┬────┬────┬────┬────┬────┬────┬
         4/10  4/12  4/14  4/16  4/18

平均: 85.2%  趋势: ↑ 改善中
```

## CLI 协调策略

### 在线模式（默认）

CLI 通过 API 与服务端交互：

```bash
qtcloud-asset contract get --project qtcloud-asset
qtcloud-asset contract update --file contract.yaml
qtcloud-asset discovery start --project qtcloud-asset
qtcloud-asset snapshot list --project qtcloud-asset
```

### 离线模式

CLI 直接操作本地文件：

```bash
qtcloud-asset discovery start --local
qtcloud-asset validate --local
qtcloud-asset sync push
```

### 混合模式

CLI 缓存服务端契约，支持离线编辑：

```bash
qtcloud-asset contract pull   # 拉取到本地缓存
# 离线编辑...
qtcloud-asset contract push    # 推送更新到服务端
```

## 增强设计

### 增量快照存储

```python
class CatalogSnapshot(BaseModel):
    id: str
    project_id: str
    snapshot_type: str  # full | incremental
    base_snapshot: Optional[str]
    diff: Optional[Dict]
    status: str  # healthy | drift | broken
    created_at: datetime
```

**归档策略**：
- 数据库存储：最近 30 天全量 + 增量
- GitHub 归档：仅里程碑版本
- 过期清理：自动删除 90 天前的非里程碑快照

### 安全与权限

**API Key 体系**：

```bash
qtcloud-asset config set token $QT_ACCESS_TOKEN
```

**Webhook 安全**：

```python
def verify_webhook(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)
```

## 数据流

```
Developer -> CLI -> Provider API -> Contract DB
                       |
                       v
                Discovery Service
                       |
       ┌───────────────┼───────────────┐
       v               v               v
  Filesystem        GitHub          Feishu
       │               │               │
       └───────────────┼───────────────┘
                       v
                Catalog Snapshot
                       |
                       v
                Catalog History DB
```

## 事件流抽象

将所有 Discovery 结果抽象为"事件流"，实现"感知层"的闭环。

```python
class DiscoveryEvent(BaseModel):
    event_type: str  # asset_found | asset_missing | asset_modified | mismatch_detected
    asset_id: str
    project_id: str
    timestamp: datetime
    payload: Dict[str, Any]
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

## 契约版本管理

对每一次 `qtcloud-asset contract push` 进行快照存储，拥有完整的意图演化历史。

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

## 核心建议

### 版本兼容性 (Graceful Degradation)

确保 `--local` 始终可用：

```python
@app.command()
def discovery(
    local: bool = typer.Option(False, "--local", help="本地模式"),
    project: Optional[str] = typer.Option(None, "--project", help="项目ID")
):
    if local:
        repo = LocalRepository(get_project_path())
        result = local_discovery(repo)
    elif project:
        repo = RemoteRepository(get_config())
        result = remote_discovery(repo, project)
    else:
        repo = get_repository_auto()
        result = repo.discovery()
    
    print_result(result)
```

### Schema 优先

Contract JSON Schema 作为共同依赖：

```toml
# pyproject.toml
[dependencies]
quanttide-schemas = "^0.1.0"
```

### 状态反馈环

CLI 异步任务轮询：

```python
def wait_for_job(job_id: str, timeout: int = 300) -> JobResult:
    start = time.time()
    
    while time.time() - start < timeout:
        job = get_job_status(job_id)
        
        progress = job.progress
        bar = "█" * (progress // 5) + "░" * (20 - progress // 5)
        print(f"\r[Job: #{job_id}] {job.status}... [{bar}] {progress}%", end="")
        
        if job.status in ["completed", "failed"]:
            print()
            return job.result
        
        time.sleep(2)
    
    raise TimeoutError(f"Job {job_id} timeout")
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
  
  - name: Check Asset Coverage
    run: qtcloud-asset check --fail-under=80
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

## 演化时间线

| 阶段 | 目标 | 关键交付 | 预计周期 |
|------|------|---------|---------|
| 第一阶段 | 建立连接 | Repository 抽象、登录认证 | 1-2 周 |
| 第二阶段 | 数据迁移 | Pull/Push、Webhook 闭环 | 2-3 周 |
| 第三阶段 | 能力上移 | 远程发现、策略下发 | 3-4 周 |
| 第四阶段 | 质量门禁 | `check` 命令、CI 集成 | 2-3 周 |

**总计**：8-12 周完成从 CLI 到平台的演化。
