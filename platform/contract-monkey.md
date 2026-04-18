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

## 总结

Contract Monkey 将量潮架构从"静态登记"推向"动态韧性"：

- **Drift Injection**：验证发现机制的灵敏度
- **Collision Testing**：验证元治理的防呆能力
- **Cognitive Load Test**：验证验证机制的质量把关

这不仅仅是在写代码，而是在构建一个**具备免疫系统的数字组织**。
