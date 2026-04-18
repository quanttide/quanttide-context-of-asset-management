# 多源资产治理

支持同时治理本地文件、GitHub 仓库、飞书文档等多种来源的数字资产。

## 核心概念

| 概念 | 说明 | 示例 |
|------|------|------|
| **Source** | 资产的存储位置 | GitHub、飞书、本地 |
| **Role** | 副本角色 | primary（主）、mirror（镜像）、draft（草稿） |
| **Sync** | 同步关系 | 从哪个源同步 |
| **Asset** | 逻辑资产 | 可能对应一个或多个 Source |

## 三种多源场景

### 场景一：单源资产（Single Source）

资产只存在于一个平台，这是最简单的情况。

```yaml
assets:
  # 代码只在 GitHub
  src:
    title: 源代码
    type: code
    source:
      type: github
      uri: github://quanttide/qtcloud-asset/src@main
  
  # 商业需求只在飞书
  brd:
    title: 商业需求文档
    type: docs
    source:
      type: feishu
      uri: feishu://doc/brd456
```

### 场景二：多副本资产（Multi-Source Mirror）

同一资产同步到多个平台，需要管理副本关系和一致性。

```yaml
assets:
  # 文档同时在 GitHub 和飞书
  docs:
    title: 产品文档
    type: docs
    sources:
      - type: github
        uri: github://quanttide/qtcloud-asset/docs@main
        role: primary      # 主副本：权威来源
      - type: feishu
        uri: feishu://doc/abc123
        role: mirror       # 镜像副本
        sync_from: github  # 从 GitHub 同步
        sync_mode: bidirectional  # 或 unidirectional
```

### 场景三：跨平台组合（Cross-Platform Portfolio）

不同资产位于不同平台，契约作为统一视图。

```yaml
assets:
  # 代码在 GitHub
  src:
    title: 源代码
    source:
      type: github
      uri: github://quanttide/qtcloud-asset/src@main
  
  # 需求在飞书
  brd:
    title: 商业需求
    source:
      type: feishu
      uri: feishu://doc/brd456
  
  # 设计稿在本地
  design:
    title: 设计稿
    source:
      type: filesystem
      uri: filesystem:///design
```

## 资产标识（URI）

统一使用 URI 标识跨源资产：

| 源类型 | URI 格式 | 示例 |
|--------|---------|------|
| 文件系统 | `filesystem://{path}` | `filesystem:///docs/brd/index.md` |
| GitHub | `github://{owner}/{repo}/{path}@{ref}` | `github://quanttide/qtcloud-asset/docs/brd@main` |
| 飞书文档 | `feishu://{type}/{token}` | `feishu://doc/abc123` |

## 版本映射

### 各源版本标识

| 源类型 | 版本标识 | 说明 |
|--------|---------|------|
| 文件系统 | `mtime` + `checksum` | 修改时间 + 内容哈希 |
| GitHub | `commit_sha` | Git commit hash |
| 飞书 | `revision` | 文档版本号 |

### 统一版本格式

```json
{
  "asset_id": "docs",
  "sources": {
    "github": {
      "type": "git_commit",
      "value": "a1b2c3d...",
      "metadata": {
        "author": "user@example.com",
        "date": "2026-04-18T09:00:00Z"
      }
    },
    "feishu": {
      "type": "revision",
      "value": "18",
      "metadata": {
        "editor": "user@feishu.cn",
        "updated_at": "2026-04-18T08:00:00Z"
      }
    }
  }
}
```

## 发现策略

```yaml
discovery:
  strategies:
    # 单源：直接发现
    - match:
        asset: src
      strategy: direct
    
    # 多副本：发现所有副本，校验一致性
    - match:
        asset: docs
      strategy: multi_source
      consistency_check: true
    
    # 跨平台：分别发现，统一归档
    - match:
        asset: "*"
      strategy: distributed
```

## 校验策略

### 单源校验

| 源类型 | 校验方式 |
|--------|---------|
| 文件系统 | SHA-256 内容哈希 |
| GitHub | Git blob + commit 验证 |
| 飞书 | 文档 token + revision 验证 |

### 多副本一致性校验

```yaml
validation:
  multi_source:
    - asset: docs
      checks:
        - name: content_match
          description: 各副本内容一致
          severity: error
        - name: version_lag
          description: 副本版本延迟不超过 1 小时
          severity: warning
```

## 快照归档

### 三种场景的快照结构

```json
{
  "spec_version": "0.0.1",
  "generated_at": "2026-04-18T10:00:00Z",
  "assets": {
    "src": {
      "title": "源代码",
      "source": {
        "type": "github",
        "uri": "github://...",
        "status": "active",
        "version": "a1b2c3d"
      }
    },
    "docs": {
      "title": "产品文档",
      "sources": {
        "github": {
          "role": "primary",
          "status": "active",
          "version": "a1b2c3d"
        },
        "feishu": {
          "role": "mirror",
          "status": "synced",
          "version": "rev18",
          "sync_from": "github"
        }
      },
      "consistency": "consistent"
    },
    "brd": {
      "title": "商业需求",
      "source": {
        "type": "feishu",
        "status": "active",
        "version": "rev5"
      }
    }
  }
}
```

## 发现适配器接口

```python
class DiscoveryAdapter(ABC):
    @abstractmethod
    def discover(self, source: SourceDef) -> DiscoveredAsset:
        """发现资产"""
        pass
    
    @abstractmethod
    def validate(self, asset: DiscoveredAsset) -> ValidationResult:
        """验证资产"""
        pass
    
    @abstractmethod
    def get_version(self, asset: DiscoveredAsset) -> VersionInfo:
        """获取版本"""
        pass

# 各源适配器
class FilesystemAdapter(DiscoveryAdapter): pass
class GitHubAdapter(DiscoveryAdapter): pass
class FeishuAdapter(DiscoveryAdapter): pass
```
