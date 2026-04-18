# Contract Monkey：治理混沌工程

受 Netflix Chaos Monkey 启发，引入主动"扰动"机制，验证治理系统的稳健性。

## 核心理念

> **一个伟大的标准不仅要能描述"对"的情况，更要能优雅地处理"错"的情况。**

Chaos Monkey 通过随机关闭服务器验证系统容错能力。

Contract Monkey 通过随机"扰动"资产状态，验证治理系统的**不可降级性**。

## 三种玩法

### 1. 自动漂移检测 (Drift Injection)

Provider 随机选择资产，模拟属性变更，测试 Discovery 引擎能否精准捕捉偏差。

```python
class DriftInjection:
    """漂移注入器"""
    
    async def inject_drift(self, project_id: str) -> DriftRecord:
        # 随机选择资产
        assets = await self.get_active_assets(project_id)
        target = random.choice(assets)
        
        # 模拟变更类型
        drift_types = [
            "modify_description",   # 修改描述
            "change_owner",         # 变更负责人
            "fake_deletion",        # 模拟删除
            "add_fake_dependency",  # 添加虚假依赖
        ]
        drift_type = random.choice(drift_types)
        
        # 在服务端数据库注入漂移（不改变实际资产）
        drift = await self.create_drift_record(
            project_id=project_id,
            asset_id=target.id,
            drift_type=drift_type,
            original_value=getattr(target, drift_type),
            injected_value=self.generate_fake_value(drift_type)
        )
        
        # 记录期望：Discovery 应该在下次扫描时检测到这个漂移
        return drift
    
    async def verify_detection(self, drift: DriftRecord) -> bool:
        # 触发 Discovery
        result = await run_discovery(drift.project_id)
        
        # 验证是否检测到漂移
        detected = any(
            d.asset_id == drift.asset_id and d.drift_type == drift.drift_type
            for d in result.drifts
        )
        
        return detected
```

**Dashboard 展示**：

```
┌─────────────────────────────────────────┐
│  Drift Injection Test                   │
│                                         │
│  本周注入漂移: 12 次                      │
│  成功检测: 11 次 (91.7%)                 │
│  漏检: 1 次                               │
│                                         │
│  最近注入:                                │
│  - api_docs: modify_description (2h前)   │
│    状态: ✅ 已检测                        │
│  - config_prod: fake_deletion (5h前)     │
│    状态: ✅ 已检测                        │
│  - mobile_app: change_owner (1d前)       │
│    状态: ❌ 漏检                          │
└─────────────────────────────────────────┘
```

### 2. 模拟契约冲突 (Collision Testing)

故意注入与现有 RFC 冲突的子标准，测试元治理层的"防呆"能力。

```python
class CollisionTester:
    """冲突测试器"""
    
    async def inject_collision(self, project_id: str) -> CollisionRecord:
        # 生成冲突的契约定义
        collision_types = [
            "duplicate_asset_id",      # 重复资产ID
            "circular_dependency",     # 循环依赖
            "invalid_schema",          # 无效 Schema
            "conflicting_rules",       # 冲突规则
        ]
        
        collision_type = random.choice(collision_types)
        fake_contract = self.generate_colliding_contract(collision_type)
        
        # 尝试注入（应该在验证阶段被拦截）
        try:
            await self.contract_service.update_contract(
                project_id=project_id,
                contract=fake_contract
            )
            # 如果成功，说明防呆机制失效
            return CollisionRecord(
                type=collision_type,
                intercepted=False,  # 危险！
                severity="critical"
            )
        except ContractValidationError as e:
            # 被正确拦截
            return CollisionRecord(
                type=collision_type,
                intercepted=True,
                error_message=str(e)
            )
```

**Dashboard 展示**：

```
┌─────────────────────────────────────────┐
│  Collision Test                         │
│                                         │
│  本周注入冲突: 8 次                       │
│  成功拦截: 8 次 (100%)                   │
│                                         │
│  拦截详情:                                │
│  - circular_dependency: ✅ 已拦截         │
│  - invalid_schema: ✅ 已拦截              │
│  - conflicting_rules: ✅ 已拦截           │
│                                         │
│  系统防呆能力: 健康 ✅                     │
└─────────────────────────────────────────┘
```

### 3. 认知负荷压力测试 (Cognitive Load Test)

随机推送描述模糊、不符合语境文档的资产，观察验证机制能否拦截"低认知质量"信息。

```python
class CognitiveLoadTester:
    """认知负荷测试器"""
    
    def generate_low_quality_assets(self) -> List[Asset]:
        """生成低质量资产"""
        return [
            Asset(
                name="thing",  # 过于笼统
                description="",  # 空描述
                type="stuff",  # 无效类型
            ),
            Asset(
                name="temp_2024_backup_copy_final_v2",  # 命名混乱
                description="This is a asset for doing things",  # 无意义描述
                type="docs",
            ),
            Asset(
                name="api",  # 与现有资产重复
                description="API documentation",  # 描述冲突
                type="code",  # 类型错误
            ),
        ]
    
    async def test_validation(self, project_id: str) -> ValidationResult:
        low_quality_assets = self.generate_low_quality_assets()
        
        results = []
        for asset in low_quality_assets:
            # 尝试通过验证
            result = await self.validation_service.validate_asset(
                project_id=project_id,
                asset=asset
            )
            results.append({
                "asset": asset.name,
                "passed": result.passed,
                "issues": result.issues
            })
        
        return results
```

**Dashboard 展示**：

```
┌─────────────────────────────────────────┐
│  Cognitive Load Test                    │
│                                         │
│  本周注入低质量资产: 15 个                │
│  成功拦截: 14 个 (93.3%)                 │
│  漏网: 1 个                               │
│                                         │
│  拦截原因分布:                            │
│  - 描述过于笼统: 5 次                     │
│  - 命名不规范: 4 次                       │
│  - 类型错误: 3 次                         │
│  - 与现有资产冲突: 2 次                    │
│                                         │
│  漏网资产: temp_backup (已标记复查)        │
└─────────────────────────────────────────┘
```

## 资产健康漂移率

Dashboard 核心指标：实时显示资产的"健康漂移率"。

```python
class HealthDriftMetrics:
    """健康漂移指标"""
    
    def calculate_drift_rate(self, project_id: str) -> DriftRate:
        # 获取最近 7 天的快照
        snapshots = self.get_snapshots(project_id, days=7)
        
        # 计算漂移率
        total_assets = len(snapshots[-1].assets)
        drifted_assets = len([
            a for a in snapshots[-1].assets.values()
            if a.status in ["drifted", "missing", "inconsistent"]
        ])
        
        drift_rate = drifted_assets / total_assets if total_assets > 0 else 0
        
        # 趋势分析
        trend = self.analyze_trend(snapshots)
        
        return DriftRate(
            current=drift_rate,
            trend=trend,  # improving | stable | worsening
            threshold=0.1,  # 10% 警戒线
            status="healthy" if drift_rate < 0.1 else "warning"
        )
```

**Dashboard 主界面**：

```
┌─────────────────────────────────────────────────────────────┐
│  项目：qtcloud-asset                    资产健康漂移率      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                                                     │   │
│  │   8%  ████████████████████░░░░░░░░░░░░░░░░░░░░░░   │   │
│  │       ↑ 趋势: 改善中 (上周 12%)                      │   │
│  │       状态: 健康 ✅                                  │   │
│  │                                                     │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  指标说明:                                                   │
│  - 健康: < 10% 资产处于漂移状态                              │
│  - 警告: 10%-20% 资产处于漂移状态                            │
│  - 危险: > 20% 资产处于漂移状态                              │
│                                                             │
│  本周漂移事件:                                               │
│  - Contract Monkey 注入: 12 次                               │
│  - 自然漂移: 3 次                                           │
│  - 人工修复: 8 次                                           │
│                                                             │
│  [查看详情]  [运行测试]  [导出报告]                          │
└─────────────────────────────────────────────────────────────┘
```

## 为什么加速 L7 到 L8 的进化？

### 1. 证明系统的不可降级性

如果治理系统能经受住 Contract Monkey 的随机抽检，就证明了量潮的资产治理不是靠开发者的"自觉"，而是靠**系统的必然**。

### 2. 沉淀干预的最佳实践

通过模拟各种不一致场景，观察开发者如何反应，为"自动化干预"积累最真实的决策数据。

```
场景: api_docs 缺失
    ↓
开发者反应:
    - 70% 运行 qtcloud-asset fix
    - 20% 手动创建
    - 10% 忽略
    ↓
数据沉淀:
    - L2 干预（建议修复）最有效
    - L3 干预（自动 PR）接受度 70%
    - L4 干预（自动创建）可能过度
```

### 3. 评分标准的最早原型

资产健康漂移率，就是未来评分标准的最早原型。

```
漂移率 < 5%   → A 级 (卓越)
漂移率 5-10%  → B 级 (健康)
漂移率 10-20% → C 级 (需关注)
漂移率 > 20%  → D 级 (危险)
```

## 实施建议

### 阶段一：内部测试（当前）

- 在量潮内部项目运行 Contract Monkey
- 积累数据和经验
- 优化检测算法

### 阶段二：可选开启（未来）

- 让外部用户选择是否开启
- 提供"沙盒模式"，不影响生产
- 分享最佳实践

### 阶段三：行业标准（远景）

- 将 Contract Monkey 作为治理系统的标准测试
- 类似 Chaos Engineering 的地位
- 成为 L8 的行业标准之一

## 关键洞察

> **静态的治理是脆弱的，动态的韧性才是强大的。**

> **Contract Monkey 不是破坏，而是免疫系统的疫苗。**

> **通过主动引入混乱，验证秩序的稳健性。**

## 契约覆盖率 (Contract Coverage)

类比测试覆盖率，契约覆盖率衡量资产治理的完整程度。三层语义：

### 第一层：资产覆盖率 (Asset Coverage)

物理资产被契约"签收"的比例。

```python
class AssetCoverageCalculator:
    """资产覆盖率计算器"""
    
    def calculate(self, project_id: str) -> AssetCoverage:
        # 获取契约定义
        contract = self.get_contract(project_id)
        
        # 获取实际资产（通过 Discovery）
        actual_assets = self.run_discovery(project_id)
        
        # 计算覆盖
        contracted_ids = {a.id for a in contract.assets}
        actual_ids = {a.id for a in actual_assets}
        
        covered = len(contracted_ids & actual_ids)
        uncovered = len(actual_ids - contracted_ids)
        orphaned = len(contracted_ids - actual_ids)  # 契约有但实物已删除
        
        total = len(actual_ids) if actual_ids else 1  # 避免除零
        
        return AssetCoverage(
            covered=covered,
            uncovered=uncovered,
            orphaned=orphaned,
            coverage_rate=covered / total,
            details={
                "covered_assets": list(contracted_ids & actual_ids),
                "uncovered_assets": list(actual_ids - contracted_ids),
                "orphaned_assets": list(contracted_ids - actual_ids)
            }
        )
```

**示例输出**：

```
资产覆盖率: 85.7% (12/14)

已覆盖资产 (12):
  ✓ api_docs, backend_code, frontend_code, ...

未覆盖资产 (2):
  ⚠ legacy_script (未在契约中定义)
  ⚠ temp_backup (未在契约中定义)

孤儿资产 (1):
  ✗ old_api (契约定义存在但实物已删除)
```

### 第二层：属性匹配率 (Property Match Rate)

已覆盖资产的属性一致性。

```python
class PropertyMatchCalculator:
    """属性匹配率计算器"""
    
    def calculate(self, asset_id: str) -> PropertyMatch:
        contract_asset = self.get_contract_asset(asset_id)
        actual_asset = self.get_actual_asset(asset_id)
        
        mismatches = []
        
        # 比较关键属性
        for prop in ["type", "owner", "description", "tags"]:
            contract_val = getattr(contract_asset, prop)
            actual_val = getattr(actual_asset, prop)
            
            if contract_val != actual_val:
                mismatches.append(PropertyMismatch(
                    property=prop,
                    expected=contract_val,
                    actual=actual_val
                ))
        
        total_props = len(["type", "owner", "description", "tags"])
        matched = total_props - len(mismatches)
        
        return PropertyMatch(
            asset_id=asset_id,
            match_rate=matched / total_props,
            mismatches=mismatches,
            status="consistent" if not mismatches else "drifted"
        )
```

**示例输出**：

```
属性匹配率: 75.0% (3/4 属性一致)

资产: api_docs

匹配属性:
  ✓ type: documentation
  ✓ owner: @zhangsan
  ✓ tags: ["api", "public"]

不匹配属性:
  ✗ description
    契约: "API 接口文档"
    实际: "API docs (updated 2024)"
```

### 第三层：发现完整性 (Discovery Integrity)

Discovery 扫描结果与 Registry 快照的一致性。

```python
class DiscoveryIntegrityChecker:
    """发现完整性检查器"""
    
    def check(self, project_id: str) -> IntegrityReport:
        # 最新 Discovery 结果
        discovery_result = self.run_discovery(project_id)
        
        # 最新 Registry 快照
        registry_snapshot = self.get_latest_snapshot(project_id)
        
        # 对比
        discovery_assets = {a.id: a for a in discovery_result.assets}
        registry_assets = {a.id: a for a in registry_snapshot.assets}
        
        inconsistencies = []
        
        for asset_id in set(discovery_assets.keys()) | set(registry_assets.keys()):
            if asset_id not in discovery_assets:
                inconsistencies.append(f"{asset_id}: Registry 存在但 Discovery 缺失")
            elif asset_id not in registry_assets:
                inconsistencies.append(f"{asset_id}: Discovery 发现但 Registry 未记录")
            elif discovery_assets[asset_id].hash != registry_assets[asset_id].hash:
                inconsistencies.append(f"{asset_id}: 哈希不一致（内容已变）")
        
        total = len(set(discovery_assets.keys()) | set(registry_assets.keys()))
        consistent = total - len(inconsistencies)
        
        return IntegrityReport(
            integrity_rate=consistent / total if total > 0 else 1.0,
            inconsistencies=inconsistencies,
            last_discovery=discovery_result.timestamp,
            last_snapshot=registry_snapshot.timestamp
        )
```

## CI 集成：`qtcloud-asset check`

CLI 提供 `check` 命令，支持在 CI 流程中强制执行治理门禁。

### 命令设计

```bash
# 基础检查
qtcloud-asset check

# 指定覆盖率阈值（失败时退出码非零）
qtcloud-asset check --fail-under=80

# 检查特定维度
qtcloud-asset check --asset-coverage --fail-under=90
qtcloud-asset check --property-match --fail-under=95
qtcloud-asset check --discovery-integrity --fail-under=100

# 输出格式
qtcloud-asset check --format=json > coverage-report.json
qtcloud-asset check --format=markdown > coverage-report.md
```

### 实现示例

```python
# cli/commands/check.py
import typer
from typing import Optional
import sys

app = typer.Typer()

@app.command()
def check(
    fail_under: Optional[float] = typer.Option(None, "--fail-under", help="覆盖率阈值，低于此值则失败"),
    asset_coverage: bool = typer.Option(True, "--asset-coverage/--no-asset-coverage", help="检查资产覆盖率"),
    property_match: bool = typer.Option(True, "--property-match/--no-property-match", help="检查属性匹配率"),
    discovery_integrity: bool = typer.Option(True, "--discovery-integrity/--no-discovery-integrity", help="检查发现完整性"),
    format: str = typer.Option("text", "--format", help="输出格式: text, json, markdown"),
):
    """检查契约覆盖率，支持 CI 集成"""
    
    project_path = get_project_path()
    
    # 运行检查
    results = {}
    
    if asset_coverage:
        results["asset_coverage"] = AssetCoverageCalculator().calculate(project_path)
    
    if property_match:
        results["property_match"] = PropertyMatchCalculator().calculate(project_path)
    
    if discovery_integrity:
        results["discovery_integrity"] = DiscoveryIntegrityChecker().check(project_path)
    
    # 计算综合得分
    scores = [
        r.coverage_rate if hasattr(r, "coverage_rate") else r.match_rate if hasattr(r, "match_rate") else r.integrity_rate
        for r in results.values()
    ]
    overall_score = sum(scores) / len(scores) if scores else 0
    
    # 输出报告
    if format == "json":
        print(json.dumps({k: v.__dict__ for k, v in results.items()}, indent=2))
    elif format == "markdown":
        print(generate_markdown_report(results, overall_score))
    else:
        print(generate_text_report(results, overall_score))
    
    # 判断是否失败
    if fail_under is not None and overall_score < fail_under:
        typer.echo(f"\n❌ 检查失败: 综合覆盖率 {overall_score:.1%} 低于阈值 {fail_under:.1%}", err=True)
        sys.exit(1)
    
    typer.echo(f"\n✅ 检查通过: 综合覆盖率 {overall_score:.1%}")
```

### GitHub Actions 集成示例

```yaml
# .github/workflows/asset-governance.yml
name: Asset Governance Check

on:
  pull_request:
    paths:
      - '.quanttide/**'
      - '**/README.md'
      - 'docs/**'

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup QuantTide CLI
        uses: quanttide/setup-qtcloud-asset@v1
        with:
          version: '0.2.0'
      
      - name: Check Asset Coverage
        run: qtcloud-asset check --fail-under=80 --format=markdown >> $GITHUB_STEP_SUMMARY
        
      - name: Upload Coverage Report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage-report.md
```

### 报告示例

```markdown
# 契约覆盖率报告

项目: qtcloud-asset  
检查时间: 2026-04-18 10:00:00  
综合得分: 87.5% ✅

## 资产覆盖率: 85.7%

| 指标 | 数值 |
|------|------|
| 已覆盖 | 12 |
| 未覆盖 | 2 |
| 孤儿资产 | 1 |

### 未覆盖资产
- `legacy_script` - 建议添加到契约或删除
- `temp_backup` - 临时文件，建议清理

## 属性匹配率: 91.7%

平均匹配率 across 12 个资产

### 漂移资产
| 资产 | 不匹配属性 |
|------|-----------|
| api_docs | description |

## 发现完整性: 100.0%

Discovery 与 Registry 完全一致 ✅

---

**结论**: 综合覆盖率 87.5% 达到阈值 80%，检查通过。
```

### 演进路线

| 阶段 | 功能 | 说明 |
|------|------|------|
| v0.2.0 | 基础检查 | `qtcloud-asset check` 仅支持本地检查 |
| v0.3.0 | 阈值控制 | 增加 `--fail-under` 参数 |
| v0.4.0 | 多维度 | 支持三层语义分别检查 |
| v0.5.0 | 远程报告 | 结果上报 Provider，生成趋势图 |
| v0.6.0 | 智能建议 | 根据未覆盖资产类型推荐契约模板 |

## 从"事后观测"到"准入控制"

CI 门禁的引入完成了治理体系的**关键闭环**：

> **"不合规，不合流"** —— 契约成为与 Unit Test、Lint 同等地位的工程纪律。

### 架构意义

| 维度 | 事前观测 | 准入控制 |
|------|---------|---------|
| 反馈时机 | 资产已腐化后 | 代码入库前 |
| 干预成本 | 高（需追溯修复） | 低（即时修正） |
| 认知负担 | 大（上下文丢失） | 小（现场反馈） |
| 组织惯性 | 抗拒（已成型） | 接受（未定型） |

### 渐进式实施策略

**第一步：温和启动（Soft Launch）**

```yaml
# .github/workflows/asset-governance.yml
- name: Check Asset Coverage (Non-blocking)
  run: qtcloud-asset check --format=markdown >> $GITHUB_STEP_SUMMARY
  continue-on-error: true  # 初期仅报告，不阻断
```

**第二步：渐进收紧**

```
v0.2.0: 警告模式（continue-on-error: true）
   ↓ 2周后
v0.3.0: 宽松强制（--fail-under=60, continue-on-error: false）
   ↓ 1月后  
v0.4.0: 标准强制（--fail-under=80）
```

**第三步：内部验证 Contract Monkey**

在 `quanttide-handbook` 等内部仓库开启 Drift Injection，观察：
- 开发者对"契约漂移"通知的反应时间
- 修复漂移的平均耗时
- 哪些类型的漂移最容易被忽略

### 量潮内部试点仓库

| 仓库 | 黑盒问题 | 治理价值 |
|------|---------|---------|
| `quanttide-handbook` | 文档资产散落在多个飞书目录，版本与代码不同步 | 验证多源发现 + 飞书集成 |
| `qtcloud-asset` 自身 | CLI 与 Provider 的接口契约未显式定义 | 验证自举（Bootstrap）能力 |
| 数据平台相关仓库 | 数据 Schema 与业务逻辑分离，血缘关系模糊 | 验证 Data Contract 扩展 |

### 工业美感的本质

> **最好的治理不是"管理"人，而是让"正确的行为"成为阻力最小的路径。**

当 `qtcloud-asset check` 成为 CI 的默认环节，开发者不会觉得被"管控"，而是像写单元测试一样自然——**不通过测试就不能合并，这是天经地义的工程纪律。**

契约覆盖率 80% 的门禁，本质上是在说：
> "我们接受 20% 的探索性灰色地带，但核心资产必须有明确的身份和归属。"

这种**有弹性的严格**，比 100% 的强制更符合创新组织的本质。

## 总结

Contract Monkey 将量潮架构从"静态登记"推向"动态韧性"：

- **Drift Injection**：验证发现机制的灵敏度
- **Collision Testing**：验证元治理的防呆能力
- **Cognitive Load Test**：验证验证机制的质量把关
- **Contract Coverage**：量化治理完整度，支持 CI 门禁

这不仅仅是在写代码，而是在构建一个**具备免疫系统的数字组织**。
