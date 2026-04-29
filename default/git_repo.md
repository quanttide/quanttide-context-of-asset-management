# 标准文件夹

太棒了，做出决断是架构落地最重要的一步。基于我们达成的共识，结合你选择的 "单体演进形态（单主程序 + Assets/Manifests 分离）"，我为你输出一份可以直接作为项目脚手架使用的完整目录规范。
你可以直接把以下结构复制下来，作为你们单仓的初始骨架（建议配合 tree 命令生成并放入 README 中作为规范）：
your-ai-project/
│
├── 📄 README.md                         # 项目说明文档
├── 📄 package.json                      # (Node/TS) 唯一的顶层依赖管理
├── 📄 pyproject.toml                    # (Python) 唯一的顶层依赖管理
├── 📄 .gitignore
│
│
├── 📂 src/                              # 【核心引擎】唯一的主程序源码
│   ├── 📄 main.ts                       # 程序的唯一启动入口 (如启动 Web 服务或 Agent 死循环)
│   │
│   ├── 📂 core/                         # 核心解析引擎 (负责读取 manifests 并加载 assets)
│   │   ├── 📄 loader.ts                 # 解析 YAML/JSON manifest 文件的逻辑
│   │   ├── 📄 executor.ts               # 根据 manifest 的编排指令，实际调度 skill 的逻辑
│   │   └── 📄 types.ts                  # 整个系统通用的 TypeScript 类型/接口定义
│   │
│   ├── 📂 handlers/                     # 业务处理层 (具体的路由、控制器)
│   │   ├── 📄 agent.handler.ts          # 处理 Agent 对话请求
│   │   └── 📄 workflow.handler.ts       # 处理工作流触发请求
│   │
│   └── 📂 utils/                        # 纯工具函数 (格式化时间、加密解密等，不包含业务逻辑)
│       └── 📄 logger.ts
│
│
├── 📂 assets/                           # 【静态资产层】所有 AI 素材的原材料仓库
│   │
│   ├── 📂 prompts/                      # 提示词模板库
│   │   ├── 📄 system-main.j2            # 主系统提示词 (支持变量注入，如 {{bot_name}})
│   │   ├── 📄 summarizer.md             # 纯静态摘要提示词
│   │   └── 📂 few-shots/                # 复杂的 Few-Shot 样例可以单独建文件夹
│   │       └── 📄 extraction.json
│   │
│   ├── 📂 skills/                       # 技能的具体实现代码库
│   │   ├── 📂 web-search/
│   │   │   ├── 📄 index.ts              # 技能执行逻辑 (调用外部 API 或本地函数)
│   │   │   └── 📄 schema.json           # 该技能的入参/出参 JSON Schema 定义
│   │   ├── 📂 code-review/
│   │   │   └── 📄 index.ts
│   │   └── 📂 db-query/
│   │       └── 📄 index.ts
│   │
│   └── 📂 knowledge/                    # 知识库与静态数据
│       ├── 📂 faq/                      # 结构化的 FAQ 数据
│       │   └── 📄 products.json
│       └── 📂 rules/                    # 静态的业务规则文档
│           └── 📄 refund-policy.md
│
│
├── 📂 manifests/                        # 【声明编排层】系统的中枢神经 (纯配置，零代码)
│   │
│   ├── 📂 registry/                     # 系统标准资产清单 (户口本)
│   │   ├── 📄 skills.yaml               # 声明系统里有哪些标准 skill 及其元数据
│   │   └── 📄 models.yaml               # 声明允许系统调用的 LLM 模型列表及限速配置
│   │
│   ├── 📂 agents/                       # Agent 组装清单
│   │   ├── 📄 main-assistant.yaml       # 主 Agent: (组装了 prompt A + skill B + skill C)
│   │   └── 📄 background-analyst.yaml   # 后台分析 Agent: (组装了 prompt D + skill E)
│   │
│   └── 📂 workflows/                    # 工作流/流水线编排 (DAG 有向无环图)
│       └── 📄 daily-report.yaml         # 编排: [db-query] -> [summarizer] -> [web-search]
│
│
├── 📂 config/                           # 【环境配置层】随部署环境变化的基础设施参数
│   ├── 📄 default.yaml                  # 默认基础配置
│   ├── 📄 development.yaml              # 开发环境覆盖配置 (如本地 mock 数据库)
│   ├── 📄 production.yaml               # 生产环境覆盖配置 (如真实的 Redis 密码)
│   └── 📄 tsconfig.json                 # 代码层面的静态检查配置
│
│
└── 📂 .github/                          # 【工程化层】自动化流水线
    └── 📂 workflows/
        ├── 📄 ci.yml                    # 常规的代码 lint 和单测
        └── 📄 validate-manifests.yml    # 【核心】校验 manifests 中引用的 assets 路径是否真实存在
📝 附：给开发团队的"宪法"说明（建议直接贴进 README）
为了确保这个架构不被破坏，你需要给团队定下几条简单的规矩：
assets 绝对隔离原则：
assets 下的文件（如 .j2, .md, .json）禁止包含任何运行时代码逻辑（如 if/else, import 语句）。它们只是被 src/core/loader 读取的字符串或数据。
manifests 纯引用原则：
编排文件只负责"指认"资产路径（如 ref: assets/skills/web-search），绝对不写具体的调用实现。
不在 manifests 里写死环境变量（如数据库密码），环境变量统一放 config/。
单向依赖原则：
依赖流向必须是严格的单向箭头： src  → 读取 → manifests  →  指向 → assets。
assets 和 manifests 永远不需要、也不允许 import src 里的任何代码。
新增 Skill 的标准动作：
第一步：在 assets/skills/ 下写实现代码和 schema.json。
第二步：在 manifests/registry/skills.yaml 里注册它。
第三步：在 manifests/agents/xxx.yaml 里把它 ref 给某个 Agent。
（这三步缺一不可，CI 脚本会自动检查第 2、3 步指向的路径是否存在）

---

# apps vs src

没问题，这非常关键。架构最怕的就是"生搬硬套"，必须在规范里把边界彻底划定。
我们将这段"演进法则"作为架构决策记录（ADR）直接补充到规范的最前面。你可以把下面这段话直接作为项目 README 的核心准则：
🚦 架构边界准则：src 与 apps 的使用时机
本项目的当前阶段采用的是 "单体核心 + 声明式外挂" 架构，因此顶层目录使用的是 src/。
严禁在当前阶段强行拆分出 apps/ 目录，避免过度设计。
但为了应对未来的系统扩张，在此明确 src 与 apps 的切换红线：
✅ 什么时候必须用 src？（当前阶段）
当你的系统只有一个唯一的启动入口时。
哪怕这个入口内部非常复杂（比如同时启动了 HTTP 路由、WebSocket 连接、后台定时任务），只要它们是打包在同一个进程里、共享同一个内存空间、通过一份 package.json 启动，就必须老老实实放在根目录的 src/ 下。
典型场景：一个 FastAPI 服务、一个 Next.js 全栈应用、一个带 CLI 入口的 Python 脚本。
🚨 什么时候必须重构为 apps/？（触发条件）
当你的系统出现了"生命周期完全不同，且需要独立部署"的多个可执行程序时。
如果你发现以下任何一种情况，就意味着单体结构破裂，必须将当前的 src/ 拆分，升级出 apps/ 目录：
出现了不同的"宿主环境"：
比如：你原本只有 Node.js 写的后端逻辑，现在需要加一个用 Python 写的模型微调脚本，两者无法在一个进程里跑。
出现了不同的"部署节奏"：
比如：你的主 API 服务每天要发版好几次，但你写了一个长驻后台的 Worker 进程，极其脆弱不能随便重启。两者被硬塞在一个 src 里，导致发版互相牵连。
出现了独立的"交互入口"：
比如：你现在的代码是通过 API 给前端调用，但你需要提供一个独立的 MCP Server 给客户端 IDE 直连。它们的启动方式完全不同。
冷启动依赖爆炸：
比如：API 服务只需要 2 秒启动，但因为你把所有东西塞在 src 里，导致它要加载几万个知识库向量文件，启动变成了 2 分钟。
🔄 演进路径预览（仅供参考，当前勿用）
如果未来触发了上述条件，我们的目录结构会发生如下裂变（请注意 src 退居二线的变化）：
# 【未来裂变形态】（触发条件：需要独立部署 API 和 Worker）
your-ai-project/
├── apps/                      # 【裂变产生】顶层变成多应用的集合
│   ├── api-server/            # 独立应用 1：对外提供 HTTP 接口
│   │   └── src/               # 【降级】src 变成了具体应用内部的目录
│   │       └── main.ts
│   └── background-worker/     # 独立应用 2：常驻后台处理工作流
│       └── src/               # 【降级】src 变成了具体应用内部的目录
│           └── main.ts
│
├── packages/                  # 【裂变产生】从旧 src 中抽离出来的被共享的代码库
│   └── core-engine/           # 读取 manifests 和 assets 的公共逻辑
│       └── src/
│
├── assets/                    # 【保持不变】全局共享的静态资产
├── manifests/                 # 【保持不变】全局共享的编排清单
└── config/                    # 【保持不变】环境配置
总结一句话：只要还是"一个锅做饭"，就全扔进 src/；一旦变成"开连锁店，各店有各厨"，就升级为 apps/。
把这段加进去，任何新来的开发人员（或者半年后的你自己）在看目录时，就完全清楚这个架构的边界在哪里了。
