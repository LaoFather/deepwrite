# DeepWrite 功能架构图

> 基于 v1.5.2 代码分析生成。DeepWrite 是面向创作者的本地优先 AI 写作智能体工作台，采用 Electron 多进程架构。

---

## 一、整体进程架构

```mermaid
graph TB
    subgraph Electron 应用
        subgraph Main 主进程
            WINDOW[窗口生命周期<br/>create-desktop-window]
            IPC[IPC 命令路由与安全校验<br/>ipc/*]
            SUPERVISOR[UtilitySupervisor<br/>监管三个子进程]
            CONFIG[配置存储层<br/>*-store.ts]
            EXTRAS[可选能力<br/>云备份/设备同步/市场]
        end

        subgraph Renderer 渲染进程
            SHELL[WorkspaceShell.vue<br/>三栏工作台壳]
            LEFT[左侧资源树]
            MIDDLE[中间智能体对话]
            RIGHT[右侧内容编辑器]
        end

        PRELOAD[Preload 预加载<br/>window.deepwrite 白名单]

        subgraph Core Utility
            CATALOG[FolderCatalogStore<br/>短篇/剧本/素材/技能]
            LONG[LongWorkspaceService<br/>长篇五阶段]
            MATERIAL[MaterialQueryService<br/>素材查询]
            LIBRARY[LibraryManagementService<br/>资料库管理]
            CONV_HIST[对话历史存储]
        end

        subgraph Agent Utility
            PI[PiAgentRuntimeAdapter<br/>智能体运行时]
            PROVIDERS[多模型 Provider<br/>OpenAI/Anthropic/Google/Volc/DeepSeek]
            SUBAGENT[子智能体运行]
        end

        subgraph Tool Utility
            TOOL_BOUNDARY[受控工具执行边界]
        end
    end

    SHELL --> PRELOAD
    PRELOAD --> IPC
    IPC --> SUPERVISOR
    SUPERVISOR --> Core
    SUPERVISOR --> Agent
    SUPERVISOR --> Tool
    IPC --> CONFIG
    IPC --> EXTRAS
    IPC --> WINDOW

    Agent -.内部查询命令.-> Core
```

---

## 二、功能模块总览

```mermaid
graph LR
    subgraph 创作能力
        SHORT[短篇创作<br/>人物/剧情/大纲/正文/分节]
        SCRIPT[剧本创作]
        LONG[长篇创作<br/>世界观/人物/剧情/伏笔/章卡/正文/账本]
        LEARN[学习仿写<br/>风格分析与复用]
        ANALYSIS[书籍分析<br/>短篇/长篇文本分析]
    end

    subgraph 智能体协作
        AGENT_TEAM[智能体团队<br/>主智能体+子智能体]
        PROPOSAL[可审阅修改<br/>Diff 预览/接受/拒绝]
        CONVERSATION[对话管理<br/>历史/持久化/中断恢复]
        MODEL[多模型管理<br/>配置/切换/用量统计]
    end

    subgraph 资源管理
        WORKSPACE[作品管理<br/>新建/打开/导入/导出]
        MATERIAL_LIB[素材库]
        SKILL_LIB[技能库]
        MARKETPLACE[技能市场]
    end

    subgraph 同步与备份
        DEVICE_SYNC[设备同步<br/>WebDAV]
        CLOUD_BACKUP[云备份]
        RECOVERY[草稿与会话恢复]
    end

    subgraph 系统能力
        SETTINGS[设置<br/>外观/通用/模型/智能体]
        UPDATE[应用更新]
        USAGE[模型用量统计]
        MARKETPLACE_AUTH[市场账号]
    end
```

---

## 三、主进程 (Main) 功能模块

```mermaid
graph TB
    subgraph Main 主进程
        subgraph 窗口与生命周期
            W1[窗口创建/显示/隐藏]
            W2[系统托盘]
            W3[优雅关闭<br/>flush 会话+关闭 Utility]
            W4[启动门控]
        end

        subgraph IPC 命令路由
            I1[catalog 作品目录]
            I2[long 长篇工作区]
            I3[manuscript 稿件导出]
            I4[session 智能体会话]
            I5[model 模型配置]
            I6[settings 设置]
            I7[appearance 外观]
            I8[agent-team 智能体团队]
            I9[conversation 对话导出/历史]
            I10[renderer-state 渲染状态同步]
            I11[marketplace 市场]
            I12[device-sync 设备同步]
            I13[cloud-backup 云备份]
        end

        subgraph 配置存储
            S1[ModelConfigStore 模型]
            S2[ModelUsageStore 用量]
            S3[GeneralSettingsStore 通用]
            S4[AppearanceService 外观]
            S5[AgentTeamConfigStore 团队]
            S6[WorkspaceAgentConfigStore 作品智能体]
            S7[LibraryAgentConfigStore 资料库智能体]
            S8[LongAgentConfigStore 长篇智能体]
            S9[WorkspaceDirectoryStore 工作目录]
            S10[ChatAssistantProjectConfigStore]
            S11[LearningImitationConfigStore 仿写]
        end

        subgraph 核心服务
            C1[UtilitySupervisor 进程监管]
            C2[UpdateService 更新]
            C3[MarketplaceClient 市场客户端]
            C4[BookAnalysisServices 书籍分析]
            C5[ManuscriptExport 稿件导出]
            C6[SoftwareTokenUsageReporter 用量上报]
        end
    end
```

---

## 四、Core Utility 功能模块

```mermaid
graph TB
    subgraph Core Utility - 本地项目唯一写入者
        subgraph 短篇/剧本/素材/技能
            FC[FolderCatalogStore]
            FC --> F1[作品目录索引]
            FC --> F2[短篇/剧本作品 CRUD]
            FC --> F3[人物结构变更]
            FC --> F4[剧情结构变更]
            FC --> F5[草稿分节管理]
            FC --> F6[文档读写]
            FC --> F7[素材库/技能库管理]
            FC --> F8[草稿恢复]
        end

        subgraph 长篇工作区
            LW[LongWorkspaceService]
            LW --> L1[长篇列表/打开]
            LW --> L2[世界观设定]
            LW --> L3[人物设定]
            LW --> L4[剧情设计<br/>故事线/卷纲/剧情点/章卡]
            LW --> L5[伏笔管理]
            LW --> L6[正文写作]
            LW --> L7[连续性账本<br/>人物轨迹/世界揭露/伏笔变化]
        end

        subgraph 辅助服务
            MQ[MaterialQueryService 素材查询]
            LM[LibraryManagementService 资料库管理]
            CH[对话历史运行时]
            DS[设备同步核心命令]
            SA[短篇分析数据源]
        end

        subgraph 数据安全
            TX[ProjectTransaction<br/>原子写入/日志/恢复/锁]
            INT[Integrity 完整性校验]
        end
    end
```

---

## 五、Agent Utility 与运行时适配层

```mermaid
graph TB
    subgraph Agent Utility
        PI[PiAgentRuntimeAdapter]
        PI --> RUN[智能体运行循环]
        PI --> STREAM[流式事件输出]
        PI --> ABORT[中断控制]
        PI --> INPUT[用户输入响应]
    end

    subgraph pi-runtime-adapter 适配层
        subgraph 运行时核心
            A1[Provider Runtime<br/>多模型适配]
            A2[Run Lifecycle<br/>运行生命周期]
            A3[Tool Stream<br/>工具调用流]
            A4[User Input Broker<br/>用户输入代理]
            A5[Subagent Runtime<br/>子智能体运行]
        end

        subgraph 提示词构建
            P1[System Prompt 构建]
            P2[User Message 构建]
            P3[长篇目录/导航提示词]
            P4[写作提示词]
        end

        subgraph 受控工具集
            T1[短篇工具<br/>list/read/create/edit/delete]
            T2[长篇工具<br/>世界观/人物/剧情/章卡/正文/账本]
            T3[素材查询工具]
            T4[资料库管理工具]
            T5[学习仿写工具]
            T6[书籍分析工具]
            T7[修订分析工具]
            T8[子智能体创作工具]
        end

        subgraph 模型 Provider
            M1[OpenAI Completions]
            M2[OpenAI Responses]
            M3[Anthropic Messages]
            M4[Google Generative AI]
            M5[Volcengine 火山引擎]
            M6[DeepSeek]
        end
    end

    Agent --> pi-runtime-adapter
```

---

## 六、渲染进程 (Renderer) 功能模块

```mermaid
graph TB
    subgraph Renderer Vue 3 + Pinia
        subgraph 工作台壳 WorkspaceShell
            LAYOUT[三栏布局<br/>左资源树/中对话/右编辑器]
            LAZY[lazyAppComponents<br/>按需加载功能模块]
        end

        subgraph 状态管理 Stores
            ST1[conversationStore<br/>对话控制器/持久化]
            ST2[longWorkspaceStore<br/>长篇工作区状态]
            ST3[catalogIndexStore<br/>目录索引]
            ST4[settingsStore<br/>设置域]
            ST5[layoutStore<br/>布局状态]
        end

        subgraph 短篇写作模块
            SW1[WritingWorkspaceModule]
            SW1 --> SW1a[人物]
            SW1 --> SW1b[剧情]
            SW1 --> SW1c[大纲]
            SW1 --> SW1d[正文]
            SW1 --> SW1e[分节写手]
        end

        subgraph 长篇写作模块
            LW1[LongWorkspaceModule]
            LW1 --> LW1a[世界观]
            LW1 --> LW1b[人物]
            LW1 --> LW1c[剧情]
            LW1 --> LW1d[伏笔]
            LW1 --> LW1e[章卡]
            LW1 --> LW1f[正文]
            LW1 --> LW1g[连续性账本]
        end

        subgraph 智能体对话
            CV[AgentConversation]
            CV --> CV1[流式消息渲染]
            CV --> CV2[工具调用展示]
            CV --> CV3[修改建议 Diff 审阅]
            CV --> CV4[子智能体活动]
            CV --> CV5[模型选择/思考等级]
        end

        subgraph 资源管理
            RES1[作品资源树]
            RES2[素材库管理]
            RES3[技能库管理]
            RES4[草稿恢复]
        end

        subgraph 设置与系统
            SET1[模型配置]
            SET2[外观主题]
            SET3[智能体设置]
            SET4[工作目录]
            SET5[通用设置]
            SET6[学习仿写设置]
        end

        subgraph 可选能力
            EXT1[技能市场]
            EXT2[云备份]
            EXT3[设备同步]
            EXT4[聊天助手]
            EXT5[应用更新]
        end
    end
```

---

## 七、长篇创作五阶段工作流

```mermaid
graph LR
    A[世界观设定] --> B[人物设定]
    B --> C[剧情设计]
    C --> D[正文写作]
    D --> E[连续性账本]

    subgraph 世界观
        A1[规则/势力/地理]
        A2[历史/术语/境界]
        A3[物品/金手指]
    end

    subgraph 人物
        B1[概览索引]
        B2[核心档案<br/>欲望/恐惧/缺陷/秘密]
        B3[人物关系]
        B4[当前状态/历史轨迹<br/>由账本映射]
    end

    subgraph 剧情
        C1[全书故事线]
        C2[分卷大纲]
        C3[剧情点]
        C4[故事情节/故事事件]
        C5[伏笔设计]
        C6[章卡]
    end

    subgraph 正文
        D1[按章卡写正文]
        D2[3000-5000字/章]
        D3[不写章节标题/分析]
    end

    subgraph 账本
        E1[章末状态]
        E2[下一章接续包]
        E3[人物发展状态]
        E4[世界观揭露]
        E5[伏笔兑现核验]
    end

    A -.约束.-> C
    B -.驱动.-> C
    C -.指导.-> D
    D -.记录.-> E
    E -.映射.-> B
```

---

## 八、智能体修改审阅流程

```mermaid
sequenceDiagram
    participant U as 用户
    participant R as Renderer
    participant M as Main
    participant A as Agent Utility
    participant C as Core Utility

    U->>R: 发送创作指令
    R->>M: agent.prompt 命令
    M->>A: 转发到 Agent Utility
    A->>A: PiAgentRuntimeAdapter 运行
    A->>A: 模型生成 + 工具调用
    A->>C: 内部查询命令(经 Main 授权)
    C-->>A: 返回作品内容
    A-->>M: 流式事件(消息/工具/建议)
    M-->>R: 广播事件
    R->>R: 渲染流式消息

    A->>R: 生成修改提案(proposal)
    R->>R: 展示 Diff 差异预览
    U->>R: 接受/拒绝修改
    R->>M: 提交接受结果
    M->>C: 写入作品文件
    C->>C: 原子写入(事务+日志)
    C-->>M: 写入结果
    M-->>R: 同步更新
```

---

## 九、数据流与边界

```mermaid
graph TB
    subgraph 安全边界
        R[Renderer 渲染层]
        P[Preload 白名单]
        M[Main 主进程]
    end

    subgraph 密钥边界
        M -- 模型密钥只在此 --> A[Agent Utility]
        R -- 禁止获取密钥 --> M
    end

    subgraph 写入边界
        A -- proposal 提案 --> M
        M -- 用户审阅后 --> C[Core Utility]
        C -- 唯一写入者 --> FS[(本地文件系统)]
    end

    subgraph 查询边界
        A -- 只读查询令 --> C
        C -- 长篇只查询 --> A
    end

    R --> P --> M
    M --> A
    M --> C
    M --> T[Tool Utility]

    FS[deepwrite.json + UTF-8 Markdown]
    C --> FS
```

---

## 十、技术栈与依赖

| 层级 | 技术选型 |
|------|---------|
| 桌面框架 | Electron (utilityProcess 多进程) |
| 打包工具 | electron-vite + electron-builder |
| 前端框架 | Vue 3 (Composition API) + Pinia |
| UI 组件 | Naive UI |
| 契约校验 | Zod |
| 智能体运行时 | Pi SDK (内部运行时) |
| 语言 | TypeScript |
| 包管理 | pnpm workspace |
| 测试 | Vitest |
| 格式/规范 | Prettier + ESLint |
| 状态管理 | Pinia + Composables |
| 模型协议 | OpenAI/Anthropic/Google/Volcengine/DeepSeek |

---

## 关键设计决策

1. **多进程隔离**：Renderer/Preload/Main/Core/Agent/Tool 六个进程，职责单向依赖，密钥不下发渲染层
2. **可审阅写入**：所有智能体修改先生成 proposal，用户接受后才由 Core 原子落盘
3. **本地优先**：作品以 `deepwrite.json` + UTF-8 Markdown 存储，可用 Git/同步盘管理
4. **契约驱动**：`@deepwrite/contracts` 是协议唯一来源，跨进程通信走 Zod 校验的 Envelope
5. **长篇五阶段**：世界观→人物→剧情→正文→账本，共享设定但各阶段独立写入
6. **受控工具边界**：Agent 只能通过预定义工具访问作品，Tool Utility 提供执行隔离
