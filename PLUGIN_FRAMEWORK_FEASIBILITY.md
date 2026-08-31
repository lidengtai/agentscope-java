# AgentScope「框架 + Agent 插件」可行性落地方案

> 版本：v1.0 | 日期：2026-08-29
> 目标：将 AgentScope 改造为「公共 Controller + Agent 插件包」的插件化框架，每个 Agent 以独立插件形式发布、按需装载。

---

## 一、可行性分析

### 1.1 核心结论

**方案完全可行**。AgentScope 现有代码已具备插件化所需的全部底层机制，本方案不修改 agentscope-core / agentscope-harness 任何源码，仅在其上新增两个轻量模块即可实现。

### 1.2 现有机制与插件化需求的逐项验证

| 插件化需求 | AgentScope 现有机制 | 验证结果 | 证据位置 |
|---|---|---|---|
| 插件自动发现 | `ModelProvider` SPI + `ServiceLoader` 双 ClassLoader 扫描 | ✅ 成熟模式可直接复用 | `agentscope-core/src/main/java/io/agentscope/core/model/spi/ModelProvider.java`、`ModelRegistry.java` |
| 插件注册中心 | `ModelRegistry` 静态注册 + 工厂注册 + SPI 三层优先级 | ✅ 模式已验证 | `ModelRegistry.register() / registerFactory()` |
| 插件与框架解耦 | `ModelCreationContext` 中性上下文（标准字段 + options/components 透明传递） | ✅ 隔离原则已确立 | `agentscope-core/src/main/java/io/agentscope/core/model/ModelCreationContext.java` |
| Agent 可编程装配 | `ReActAgent.builder()` / `HarnessAgent.builder()` Fluent API | ✅ 天然支持工厂式创建 | `agentscope-core/src/main/java/io/agentscope/core/ReActAgent.java`、`agentscope-harness/.../HarnessAgent.java` |
| 工具随插件携带 | `Toolkit.registerTool()` 注解扫描 + Tool Group 分组 | ✅ 每插件独立 Toolkit | `agentscope-core/src/main/java/io/agentscope/core/tool/Toolkit.java` |
| 每插件独立中间件 | `MiddlewareBase` 5 阶段洋葱钩子，构造时注入 Agent | ✅ 天然按 Agent 隔离 | `agentscope-core/src/main/java/io/agentscope/core/middleware/MiddlewareBase.java` |
| Spring Boot 零配置接入 | 12 个 Starter 的 `@AutoConfiguration + @ConditionalOnClass` 模式 | ✅ 模式成熟 | `agentscope-extensions/agentscope-spring-boot-starters/` |
| 统一 Web 入口 | dataagent 的 `/api/agents/{agentId}/chat/stream` 按 agentId 路由 | ✅ 生产验证过的路由模式 | `agentscope-examples/agents/agentscope-dataagent/.../ChatController.java` |
| 插件级权限隔离 | `PermissionEngine` + `PermissionMode`（ALLOW/ASK/DENY/BYPASS） | ✅ 框架级强制 | `agentscope-core/src/main/java/io/agentscope/core/permission/` |
| 会话按 Agent 隔离 | dataagent 的 gateKey（`|x:agentId=` + `|t:conversationId`）编码 | ✅ 生产验证 | `agentscope-dataagent/.../SessionAgentManager.java` |

### 1.3 可行性风险点（均有缓解方案）

| 风险 | 等级 | 缓解方案 |
|---|---|---|
| Java SPI 不支持热插拔（加插件需重启） | 低 | 企业内场景重启成本可接受；预留 `registry.reload()`；如需真热插拔，二期引入独立 ClassLoader 插件目录 |
| 多插件共享同一 Model 实例的并发安全 | 低 | `ReActAgent` 2.0 已线程安全；`Memory`/`Toolkit` 保持 prototype 作用域、按会话懒加载 |
| 插件 JAR 依赖冲突（不同插件引不同版本库） | 中 | Maven Enforcer 强制 provided 传递依赖；公司内统一 BOM；冲突插件用 maven-shade 隔离 |
| 意图路由准确率（自动选 Agent） | 中 | 一期先做显式 `{agentId}` 路由（100% 可靠），意图路由作为二期增强 |

---

## 二、总体架构

### 2.1 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                     前端（Web / IM / OpenAPI）                    │
│         统一入口：POST /api/agents/{agentId}/chat/stream          │
└────────────────────────────┬────────────────────────────────────┘
                             │ SSE / REST
┌────────────────────────────▼────────────────────────────────────┐
│           框架层（agentscope-plugin-spring-boot-starter）         │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ AgentPluginController（公共 Controller，全应用仅 1 个）    │   │
│  │  GET  /api/agents                    → 插件目录           │   │
│  │  POST /api/agents/{id}/chat/stream   → SSE 流式对话       │   │
│  │  POST /api/agents/{id}/chat/send     → 同步对话           │   │
│  │  DEL  /api/agents/{id}/sessions/{k}  → 会话重置           │   │
│  │  GET  /api/agents/health             → 插件健康检查       │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ AgentPluginRegistry（注册中心）                            │   │
│  │  发现优先级：① ServiceLoader SPI ② Spring Bean ③ 手动 API  │   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ AgentPluginContextFactory（上下文工厂）                    │   │
│  │  解析 Model / workspace / 配置 → AgentPluginContext       │   │
│  └──────────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────────┘
                             │ 按 {agentId} 分发
        ┌────────────────────┼────────────────────┐
        ▼                    ▼                    ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│ order-bot    │   │ doc-assistant│   │ data-analyst │   ← 各业务团队独立开发
│ 插件 JAR      │   │ 插件 JAR      │   │ 插件 JAR      │
│ ┌──────────┐ │   │ ┌──────────┐ │   │ ┌──────────┐ │
│ │Plugin实现 │ │   │ │Plugin实现 │ │   │ │Plugin实现 │ │
│ │Toolkit   │ │   │ │Toolkit   │ │   │ │Toolkit   │ │
│ │Middleware│ │   │ │Middleware│ │   │ │Middleware│ │
│ │Skill     │ │   │ │RAG       │ │   │ │Sandbox   │ │
│ └──────────┘ │   │ └──────────┘ │   │ └──────────┘ │
└──────────────┘   └──────────────┘   └──────────────┘
```

### 2.2 核心设计决策

| 决策点 | 选型 | 依据 |
|---|---|---|
| 插件发现机制 | Java SPI（`ServiceLoader`）+ Spring Bean 注入 | 与 AgentScope `ModelProvider` 完全一致，团队心智模型统一 |
| 插件契约 | `AgentPlugin` 接口（agentId + createAgent） | 对标 `ModelProvider` 的 `providerId + create` 最小契约 |
| 路由方式 | URL 路径 `{agentId}` 显式路由 | dataagent 生产验证；意图路由二期增强 |
| 插件隔离边界 | 插件 = 独立 Toolkit + Memory + Middleware +（可选）Workspace | AgentScope 原生 per-Agent 实例化，零改造 |
| 插件依赖方式 | 插件 JAR 以 `provided` 依赖 core/harness，由宿主统一提供 | 避免 JAR 内嵌框架导致类冲突，与 Spring 生态惯例一致 |

---

## 三、模块划分与 Maven 结构

### 3.1 新增模块（2 个）

```
新增：
├── agentscope-plugin-spi/                          ← 模块 A：纯接口（零依赖）
│   └── io.agentscope.plugin
│       ├── AgentPlugin.java                        ← 插件 SPI 接口
│       ├── AgentPluginContext.java                 ← 中性创建上下文
│       └── PermissionConfig.java                   ← 插件级权限声明
│
└── agentscope-plugin-spring-boot-starter/          ← 模块 B：框架宿主
    └── io.agentscope.plugin
        ├── AgentPluginRegistry.java                ← 注册中心（SPI 发现 + 缓存）
        ├── spring/AgentPluginAutoConfiguration.java← Spring 自动装配
        └── web/
            ├── AgentPluginController.java          ← 公共 Controller
            └── AgentPluginContextFactory.java      ← 上下文工厂
```

### 3.2 业务插件模块（N 个，各团队开发）

```
com.mycompany:agentscope-plugin-{agent-id}/         ← 每个插件一个独立仓库/模块
├── src/main/java/.../OrderAgentPlugin.java         ← 实现 AgentPlugin
├── src/main/java/.../OrderToolkit.java             ← @Tool 工具集
├── src/main/resources/
│   └── META-INF/services/
│       └── io.agentscope.plugin.AgentPlugin        ← SPI 声明（一行：实现类全名）
```

### 3.3 宿主应用（主项目）

```
mycompany-agent-platform/                           ← 公司统一 Agent 平台
├── pom.xml                                         ← 引 starter + 全部插件依赖
└── src/main/resources/application.yml              ← 模型 Key / workspace / JWT 配置
```

---

## 四、关键接口契约

### 4.1 AgentPlugin SPI（插件唯一必须实现的契约）

```java
public interface AgentPlugin {
    String agentId();                                  // URL 路由键，全局唯一
    String displayName();                              // 目录展示名
    String description();                              // 目录描述
    default String version() { return "1.0.0"; }
    default String category() { return "通用"; }
    default String icon() { return "🤖"; }
    default boolean requiresAuth() { return true; }
    default PermissionConfig permissionConfig() { return PermissionConfig.defaultAllow(); }
    Agent createAgent(AgentPluginContext context);     // 核心：装配并返回 Agent
    default List<MiddlewareBase> middlewares(AgentPluginContext ctx) { return List.of(); }
    default List<String> capabilities() { return List.of(); }   // 供二期意图路由
    default boolean isHealthy() { return true; }
}
```

### 4.2 插件实现示例（业务团队视角的全部代码）

```java
public class OrderAgentPlugin implements AgentPlugin {
    @Override public String agentId() { return "order-bot"; }
    @Override public String displayName() { return "订单助手"; }
    @Override public String description() { return "订单查询、状态跟踪与审批"; }

    @Override
    public Agent createAgent(AgentPluginContext ctx) {
        Toolkit toolkit = new Toolkit();
        toolkit.registerTool(new OrderToolkit());
        return HarnessAgent.builder()
                .name("order-bot")
                .sysPrompt("你是订单管理助手……")
                .model(ctx.model())          // 模型由框架统一注入
                .toolkit(toolkit)
                .maxIters(ctx.maxIters())
                .build();
    }
}
```

发布 = 打 JAR + 写一行 SPI 声明 + 宿主 pom 加一个依赖。

---

## 五、落地路线图

### 阶段一：最小可用框架（M1，约 1 周）

**目标**：1 个公共 Controller + 2 个示范插件跑通全链路。

| 任务 | 产出 | 验收标准 |
|---|---|---|
| 创建 `agentscope-plugin-spi` 模块 | 3 个接口/类 | 纯接口零依赖，`mvn install` 通过 |
| 创建 `agentscope-plugin-spring-boot-starter` | Registry + Controller + AutoConfig | 宿主引依赖即挂载 `/api/agents/*` |
| 开发 2 个示范插件（echo + 一个真实工具） | 2 个插件 JAR | classpath 自动发现，SSE 流式对话成功 |
| 端到端联调 | curl 脚本 | `POST /api/agents/echo-bot/chat/stream` 返回 SSE 事件流 |

### 阶段二：企业级能力补齐（M2，约 2 周）

| 任务 | 说明 |
|---|---|
| 会话管理 | 移植 dataagent 的 SessionAgentManager 模式（sessionKey + 会话锁 + Agent 缓存驱逐） |
| JWT 鉴权 | 复用 dataagent 的 `JwtService` + `SecurityConfig` 模式，`{agentId}` 级 ACL |
| 插件级权限 | `PermissionConfig` 接入 `PermissionEngine`，敏感工具 ASK 模式触发 HITL 审批 |
| 状态持久化 | 引 `agentscope-extensions-mysql/redis`，服务重启会话可恢复 |
| 插件目录 API | `GET /api/agents` 返回全量插件元数据（名称/图标/分类/健康状态），前端渲染 Agent 广场 |

### 阶段三：规模化治理（M3，按需）

| 任务 | 说明 |
|---|---|
| 意图自动路由 | 用 `capabilities()` + 轻量 LLM 分类，用户免选 Agent |
| 插件热加载 | 独立 ClassLoader 插件目录 + `registry.reload()`，免重启上/下线 |
| 插件市场 | 对接 `skill-git-repository` 扩展，Git 仓库托管插件元数据与版本审批 |
| 多通道接入 | 引 `channel-dingtalk/feishu/wecom` 扩展，IM 消息同样按 agentId 分发 |

---

## 六、API 契约（宿主对外暴露的唯一接口面）

| 方法 | 路径 | 说明 |
|---|---|---|
| GET | `/api/agents` | 插件目录（含元数据、健康状态） |
| POST | `/api/agents/{agentId}/chat/stream` | SSE 流式对话（`text/event-stream`） |
| POST | `/api/agents/{agentId}/chat/send` | 同步对话（返回完整回复） |
| GET | `/api/agents/{agentId}/sessions` | 会话列表 |
| DELETE | `/api/agents/{agentId}/sessions/{sessionId}` | 重置会话 |
| GET | `/api/agents/health` | 各插件健康检查 |

SSE 事件类型直接透传 AgentScope 原生 AgentEvent（`text_block_delta` / `tool_call_start` / `tool_result_end` / `require_user_confirm` / `agent_result` 等），前端按 event 类型渲染，与框架事件体系零转换。

---

## 七、风险与应对总表

| 风险 | 概率 | 影响 | 应对 |
|---|---|---|---|
| SPI 无热插拔 | 确定 | 低 | 一期接受重启发版；三期 ClassLoader 方案 |
| 插件间依赖冲突 | 中 | 中 | provided 依赖 + 统一 BOM + Enforcer 检查 |
| 意图路由误判 | 中 | 中 | 一期显式路由兜底，意图路由仅做推荐 |
| 单插件拖垮全局（慢工具/大循环） | 低 | 高 | 每插件独立 `maxIters` + 车道信号量限并发（dataagent 已验证模式） |
| 团队插件质量参差 | 中 | 中 | 框架内置 `isHealthy()` 健康检查 + 插件审批流程（三期市场） |

---

## 八、结论

1. **技术可行性**：AgentScope 的 SPI 机制（ModelProvider）、Builder 装配（ReActAgent/HarnessAgent）、Tool 分组（Toolkit）、按 agentId 路由（dataagent ChatController）四大基础全部就绪，插件化是对现有模式的上移复用，而非底层改造。
2. **工程可行性**：新增代码集中在 2 个新模块（约 6 个类），不触碰 core/harness 源码，升级 AgentScope 版本零冲突。
3. **组织可行性**：业务团队只需实现 1 个接口 + 1 行 SPI 声明即可发布 Agent，框架由平台团队统一维护，职责边界清晰。
4. **建议**：按 M1 → M2 → M3 递进，M1 一周内可产出可演示的双插件原型，以最低成本验证全链路后再投入企业级加固。
