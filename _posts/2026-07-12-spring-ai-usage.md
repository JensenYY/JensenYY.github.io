---
layout: post
title: "Spring AI 使用方式与用途总结"
date: 2026-07-12 00:00:01 +0800
tags: [Spring AI, Java, LLM]
---

# Spring AI 使用方式与用途总结

---

## 一、Spring AI 是什么？

**Spring AI** 是 Spring 生态面向 AI 应用开发的框架，核心设计理念是 **"将 AI 模型作为 Spring 基础设施的一部分"**，就像 Spring Data 抽象数据库访问一样，Spring AI 抽象了对各种大语言模型（LLM）的调用。它的主要用途包括：

| 用途 | 说明 |
|------|------|
| **统一模型接入** | 一套 API 同时支持 OpenAI、Anthropic、Ollama 等多种模型 |
| **函数调用 (Function Calling)** | 让 LLM 自动调用 Java 方法执行操作 |
| **RAG（检索增强生成）** | 将向量数据库检索与 LLM 结合，实现知识问答 |
| **Agent / 工作流编排** | 构建多步骤、多 LLM 协作的智能代理 |
| **MCP 集成** | 支持 Model Context Protocol 客户端/服务端开发 |
| **流式响应** | 支持 SSE（Server-Sent Events）实时流式输出 |

---

## 二、核心 API：`ChatClient`

`ChatClient` 是 Spring AI 最核心的入口类，几乎每个模块都通过它来调用 LLM。其使用模式非常类似 Spring 的 `RestClient` / `WebClient`：

### 2.1 ChatClient API 分层架构

```
ChatClient                               # 高层 Fluent API（注入 ChatClient.Builder 构建）
  ├── .prompt()                          # 构建 Prompt
  │     ├── .system("...")               # 系统提示词
  │     ├── .user(u -> u.text("...")     # 用户消息（支持模板 + param 参数化）
  │     │              .param("k","v"))
  │     ├── .tools(toolCallbackProvider) # 注册工具集
  │     ├── .advisors(...)               # 注册 Advisor 拦截链
  │     └── .options(ChatOptions)        # temperature, topK, maxTokens 等
  ├── .call()                            # 同步调用
  │     ├── .content()                   # 返回原始 String
  │     ├── .entity(Record.class)        # 结构化输出：LLM JSON → Java Record
  │     └── .chatResponse()              # 完整 ChatResponse 对象
  └── .stream()                          # 流式调用 → Flux<ChatResponse>
```

### 2.2 基础调用（最简单的 Hello World）

```java
// models/chat/helloworld
String joke = chatClient.prompt()
    .user("讲个笑话")
    .call()
    .content();  // 返回纯文本字符串
```

### 2.3 结构化输出（JSON → Java Record）

Spring AI 自动将 LLM 返回的 JSON 反序列化为 Java Record，实现类型安全的输出：

```java
// evaluator-optimizer, orchestrator-workers, routing-workflow
record EvaluationResponse(String evaluation, String feedback) {}
record Generation(String thoughts, String response) {}
record OrchestratorResponse(String analysis, List<Task> tasks) {}
record RoutingResponse(String reasoning, String selection) {}

// 用法
var result = chatClient.prompt()
    .user("评估以下代码...")
    .call()
    .entity(EvaluationResponse.class);  // 自动 JSON → Java 映射
```

**适用场景**：数据提取、分类路由、结构化任务分解、评估打分等任何需要从 LLM 获取结构化数据的场景。

### 2.4 携带系统提示词（System Prompt）

```java
// 方式1：通过 Builder 设置默认系统提示词
var client = ChatClient.builder(chatModel)
    .defaultSystem("你是一个专业的代码审查员...")
    .build();

// 方式2：通过 mutate() 派生新实例（保留共享配置）
var derivedClient = chatClient.mutate()
    .defaultSystem("新的系统提示词")
    .build();
```

### 2.5 参数化提示词

```java
// orchestrator-workers
chatClient.prompt()
    .user(u -> u.text("任务: {task}\n类型: {type}")
              .param("task", taskDescription)
              .param("type", "product_description"))
    .call().content();
```

模板变量通过 `{key}` 占位符 + `.param("key", value)` 注入，适合动态构造提示词的场景。

### 2.6 多工具绑定（Tool Calling）

```java
// 几乎所有 MCP 相关模块
chatClient.prompt()
    .user("查询旧金山的天气")
    .tools(toolCallbackProvider)  // 注入工具集，LLM 自动决定是否调用、调用哪个
    .call().content();
```

工具注册有三种方式（详见 3.7 节）。

### 2.7 顾问链（Advisors 管道）

```java
// recursive-advisor-demo, evaluation-advisor-demo
chatClient.prompt()
    .user("What is the weather in Paris?")
    .advisors(
        new MessageChatMemoryAdvisor(memory),   // 1. 对话记忆
        new ToolCallingAdvisor(),               // 2. 自动工具调用
        new MyLogAdvisor()                      // 3. 自定义日志拦截
    )
    .call();
```

Advisor 是 Spring AI 的 **AOP 拦截器链机制**，按 `getOrder()` 顺序依次执行，每个 Advisor 可在请求前/响应后插入逻辑。

### 2.8 流式输出（SSE）

```java
// misc/openai-streaming-response
@GetMapping(value = "/generateStream", produces = TEXT_EVENT_STREAM_VALUE)
public Flux<ChatResponse> generateStream(@RequestParam String message) {
    return chatModel.stream(new Prompt(new UserMessage(message)));
}
```

---

## 三、工具注册的三种方式

Spring AI 提供了三种方式将 Java 方法暴露给 LLM 调用：

### 方式一：`@Tool` 注解（最常用、最简洁）

```java
// advisors/recursive-advisor-demo, misc/spring-ai-java-function-callback
public class WeatherTools {
    @Tool(description = "Get current weather for a city")
    public String getWeather(@ToolParam(description = "City name") String city) {
        return switch (city.toLowerCase()) {
            case "paris" -> "15°C, Cloudy";
            case "tokyo" -> "10°C, Sunny";
            default -> "Unknown";
        };
    }
}

// 注册为 Bean
@Bean
public ToolCallbackProvider weatherTools() {
    return MethodToolCallbackProvider.builder()
        .toolObjects(new WeatherTools())
        .build();
}
```

### 方式二：`FunctionToolCallback`（函数式风格）

```java
// misc/spring-ai-java-function-callback
Function<WeatherRequest, WeatherResponse> weatherFunc = req -> {
    // 实现逻辑
    return new WeatherResponse(15.0, "C", "Cloudy");
};

@Bean
public ToolCallback weatherFunctionInfo() {
    return FunctionToolCallback.builder("WeatherInfo", weatherFunc)
        .description("Get current weather information for a given city")
        .inputType(WeatherRequest.class)  // 驱动 JSON Schema 生成
        .build();
}
```

适用于已有 `java.util.function.Function` 实现或不想使用注解的场景。

### 方式三：MCP 工具自动发现（零代码桥接）

```java
// model-context-protocol/brave-docker-agents-gateway
// MCP Starter 自动从 MCP Server 发现工具，注入 ToolCallbackProvider
@Bean
CommandLineRunner runner(ChatClient.Builder builder, ToolCallbackProvider tools) {
    var chatClient = builder.defaultTools(tools).build();
    // tools 中包含所有 MCP Server 暴露的工具
}
```

这是最强大的方式——**外部进程（MCP Server）暴露的工具透明地成为 Spring AI 的工具**，无需任何胶水代码。

---

## 四、模块分类详解

### 🧠 1. Agentic Patterns（智能代理模式）— `agentic-patterns/`

基于 Anthropic "Building Effective Agents" 研究，展示 5 种经典 Agent 工作流模式。所有模式的入口均为 `@SpringBootApplication` + `CommandLineRunner`，通过 `ChatClient.Builder` 注入 `ChatClient`：

| 模式 | 模块 | 核心理念 | LLM 调用次数 | 并发性 | 结构化输出 |
|------|------|----------|-------------|--------|-----------|
| **链式工作流** | `chain-workflow/` | 4 步顺序链：提取数值 → 标准化 → 排序 → 格式化。每步的输出是下一步的输入 | 固定（4次） | 顺序 | 无 |
| **评估-优化器** | `evaluator-optimizer/` | 双 LLM 递归循环：生成器产出方案 + 评估器打分反馈（PASS/NEEDS_IMPROVEMENT/FAIL），迭代直到通过 | 可变（直到 PASS） | 顺序 | `.entity(Generation.class)` `.entity(EvaluationResponse.class)` |
| **编排-工人** | `orchestrator-workers/` | 编排 LLM 拆解任务为子任务，多个工人 LLM 执行，合成器汇总 | 1 编排 + N 工人 | 顺序（Stream） | `.entity(OrchestratorResponse.class)` |
| **并行化工作流** | `parallelization-workflow/` | `CompletableFuture` + `Executors.newFixedThreadPool(n)` 实现真正的 LLM 并发调用 | N（可配置） | 并发（线程池） | 无 |
| **路由工作流** | `routing-workflow/` | LLM 先分类输入的类别（reasoning + selection），再路由到对应的专用处理器 | 2（分类 + 处理） | 顺序 | `.entity(RoutingResponse.class)` |

#### 各模式实现细节

**链式工作流（ChainWorkflow）**：
```java
// 核心：for 循环迭代系统提示词数组
public String chain(String userInput) {
    String response = userInput;
    for (String prompt : DEFAULT_SYSTEM_PROMPTS) {
        response = chatClient.prompt()
            .user(String.format("{%s}\n {%s}", prompt, response))
            .call().content();
    }
    return response;
}
```

**评估-优化器（EvaluatorOptimizer）**：
```java
// 核心：递归循环 + 评估枚举终止条件
private RefinedResponse loop(String task, String context,
                              List<String> memory, List<Generation> chainOfThought) {
    Generation generation = generate(task, context);       // 生成器
    EvaluationResponse eval = evaluate(generation, task);  // 评估器
    if (eval.evaluation() == Evaluation.PASS) {
        return new RefinedResponse(generation.response(), chainOfThought);
    }
    // 不通过 → 累积反馈上下文 → 递归重试
    StringBuilder newContext = new StringBuilder(context);
    memory.forEach(m -> newContext.append(m).append("\n"));
    newContext.append("Feedback: ").append(eval.feedback());
    return loop(task, newContext.toString(), memory, chainOfThought);
}
```

**并行化工作流（ParallelizationWorkflow）**：
```java
// 核心：线程池 + CompletableFuture 实现并发
public List<String> parallel(String prompt, List<String> inputs, int nWorkers) {
    ExecutorService executor = Executors.newFixedThreadPool(nWorkers);
    try {
        List<CompletableFuture<String>> futures = inputs.stream()
            .map(input -> CompletableFuture.supplyAsync(
                () -> chatClient.prompt(prompt + "\nInput: " + input).call().content(),
                executor))
            .toList();
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
        return futures.stream().map(CompletableFuture::join).toList();
    } finally {
        executor.shutdown();
    }
}
```

**路由工作流（RoutingWorkflow）**：
```java
// 核心：第一步 LLM 分类 → 第二步路由到专用提示词
public String route(String input, Map<String, String> routes) {
    RoutingResponse route = determineRoute(input, routes);  // LLM 返回 selection
    if (!routes.containsKey(route.selection())) {
        throw new IllegalArgumentException("Unknown route: " + route.selection());
    }
    return chatClient.prompt()
        .user(routes.get(route.selection()) + "\nInput: " + input)
        .call().content();
}
```

---

### 🤖 2. Reflection Agent（自我反思代理）— `agents/reflection/`

- **模式**：Generator（生成器 LLM）→ Critique（审校 LLM）→ 反馈 → 循环改进
- **终止条件**：审校输出的反馈中包含 `<OK>`
- **核心组件**：
  - `MessageChatMemoryAdvisor` — 保留对话上下文（历史记忆）
  - `MessageWindowChatMemory` — 滑动窗口记忆
  - 两个 `ChatClient` 实例共享同一个 `ChatModel` 但有不同的 `defaultSystem`

```java
// ReflectionAgent 核心循环
@Component
public class ReflectionAgent {
    private final ChatClient generateChatClient;   // "生成高质量 Java 代码"
    private final ChatClient critiqueChatClient;   // "审查并给出改进建议，OK 则输出 <OK>"

    public String run(String userQuestion, int maxIterations) {
        String generation = generateChatClient.prompt().user(userQuestion).call().content();
        for (int i = 0; i < maxIterations; i++) {
            String critique = critiqueChatClient.prompt().user(generation).call().content();
            if (critique.contains("<OK>")) break;   // 终止条件
            // 反馈喂回生成器，继续迭代
            generation = generateChatClient.prompt()
                .user("Critique: " + critique + "\nPlease revise the previous response.")
                .call().content();
        }
        return generation;
    }
}
```

---

### 🔌 3. Model Context Protocol（MCP 协议）— `model-context-protocol/`

MCP 是 Spring AI 最具扩展性的能力：**让 AI 模型通过标准化协议调用外部工具、访问资源和数据**。这是项目最大的类别，共 20+ 子模块，展示 MCP 协议完整生态。

#### 3.1 MCP 传输层架构

```
┌──────────────────────────────────────────────────────────────┐
│                       MCP Server                             │
│  ┌──────────┐  ┌──────────────┐  ┌─────────────────────────┐│
│  │  STDIO   │  │ WebFlux SSE  │  │ WebMVC SSE / Streamable ││
│  │  Server  │  │   Server     │  │       HTTP Server       ││
│  └────┬─────┘  └──────┬───────┘  └───────────┬─────────────┘│
│       │               │                      │               │
│  ┌────┴───────────────┴──────────────────────┴────────────┐  │
│  │       McpSyncServer / McpAsyncServer (统一抽象)        │  │
│  │  @McpTool / @McpResource / @McpPrompt / @McpComplete  │  │
│  │  McpSyncServerExchange / McpSyncRequestContext         │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                           │  MCP 协议
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                       MCP Client                             │
│  ToolCallbackProvider ← SyncMcpToolCallbackProvider         │
│  McpClientCustomizer (sampling / progress / logging)         │
│  @McpSampling / @McpLogging / @McpElicitation / @McpProgress │
│  McpClient.sync(transport) / List<McpSyncClient>             │
└──────────────────────────────────────────────────────────────┘
```

#### 3.2 四种传输方式对比

| 传输方式 | 客户端类 | 服务端类 | 适用场景 | 代表模块 |
|---------|---------|---------|---------|---------|
| **STDIO** | `StdioClientTransport` | `StdioServerTransportProvider` | 本地进程间通信，CLI 工具 | `filesystem`, `starter-stdio-server` |
| **WebFlux SSE** | `WebFluxSseClientTransport` | `WebFluxSseServerTransportProvider` | 响应式 Web 服务 | `manual-webflux-server`, `starter-webflux-server` |
| **WebMVC SSE** | `HttpClientSseClientTransport` | Servlet 容器 | 传统 Servlet Web 服务 | `starter-webmvc-server` |
| **Streamable HTTP** | `WebClientStreamableHttpTransport` / `HttpClientStreamableHttpTransport` | Servlet / WebFlux | 新标准 HTTP 传输 | `starter-webflux-server`, `starter-webmvc-server` |

#### 3.3 两种注解体系

项目展示了 Spring AI 的两套工具注解体系：

| 注解 | 来源 | 用途 | 示例 |
|------|------|------|------|
| `@Tool` | Spring AI 原生 | ChatClient 工具调用 | `@Tool(description="...") public String f(@ToolParam String x)` |
| `@ToolParam` | Spring AI 原生 | 工具参数描述 | `@ToolParam(description="City") String city` |
| `@McpTool` | MCP 注解 | MCP Server 能力暴露 | `@McpTool(description="...") public String f(...)` |
| `@McpToolParam` | MCP 注解 | MCP 工具参数 | `@McpToolParam(description="Latitude") double lat` |
| `@McpResource` | MCP 注解 | 资源暴露（URI） | `@McpResource(uri="docs://{id}")` |
| `@McpPrompt` | MCP 注解 | 提示词模板 | `@McpPrompt(name="format")` |
| `@McpComplete` | MCP 注解 | URI/提示词补全 | `@McpComplete(uri="user://{name}")` |
| `@McpArg` | MCP 注解 | 提示词参数 | `@McpArg(description="...") String arg` |

**关键区别**：`@Tool` 是 Spring AI 层面的工具注册（用于 ChatClient），`@McpTool` 是 MCP 协议层面的能力暴露（用于 MCP Server），两者可在同一服务中共存（如 `mcp-annotations-server` 所示）。

#### 3.4 MCP Server 的两种配置风格

**方式一：手动编程配置（manual-webflux-server）**

```java
@Configuration
public class McpServerConfig {
    @Bean
    @ConditionalOnProperty(name = "transport.mode", havingValue = "stdio")
    public StdioServerTransportProvider stdioTransport() { ... }

    @Bean
    @ConditionalOnProperty(name = "transport.mode", havingValue = "sse")
    public WebFluxSseServerTransportProvider sseTransport() { ... }

    @Bean
    public McpSyncServer mcpServer(TransportProvider transport, WeatherApiClient tools) {
        return McpServer.sync(transport)
            .serverInfo("MCP Demo Weather Server", "1.0.0")
            .capabilities(ServerCapabilities.builder().tools(true).logging().build())
            .tools(McpToolUtils.toSyncToolSpecifications(ToolCallbacks.from(tools)))
            .build();
    }
}
```

**方式二：Starter 自动配置（starter-stdio-server）**

```java
// 极简，只需要 @SpringBootApplication + @McpTool 注解
// MCP Starter 自动扫描 @McpTool bean、自动配置传输层
@SpringBootApplication
public class McpServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(McpServerApplication.class, args);
    }
}

@Service
public class WeatherService {
    @McpTool(description = "Get weather forecast by coordinates")
    public String getForecast(@McpToolParam(description = "Latitude") double lat,
                              @McpToolParam(description = "Longitude") double lon) {
        // 调用 Weather API
    }
}
```

#### 3.5 MCP Client 的两种配置风格

**方式一：Starter 自动配置（最常用）**

```java
// brave-docker-agents-gateway, brave-chatbot
// 通过 application.properties 自动配置
@Bean
CommandLineRunner runner(ChatClient.Builder builder, ToolCallbackProvider tools) {
    var chatClient = builder.defaultTools(tools).build();
    // 直接使用，无需关心 MCP 底层细节
}
```

配置示例：
```properties
spring.ai.mcp.client.stdio.connections.weather.command=java
spring.ai.mcp.client.stdio.connections.weather.args=-jar,weather-server.jar
spring.ai.mcp.client.toolcallback.enabled=true
```

**方式二：编程式构建**

```java
// filesystem, weather/manual-webflux-server
var transport = new StdioClientTransport(
    new ServerParameters("npx", List.of("-y", "@modelcontextprotocol/server-filesystem", "/tmp"))
);
var client = McpClient.sync(transport).build();
client.initialize();
// 包装为 Spring AI ToolCallbackProvider
var tools = SyncMcpToolCallbackProvider.builder().mcpClients(List.of(client)).build();
```

#### 3.6 MCP 高级特性

| 特性 | 模块 | 说明 |
|------|------|------|
| **Sampling（采样）** | `sampling/`, `sampling-annotations/` | **Server 反调 Client 的 LLM**：Server 在处理工具调用时，可通过 `McpSyncServerExchange.createMessage()` 让 Client 执行 LLM 调用并返回结果。Client 通过 `McpClientCustomizer.sampling()` 或 `@McpSampling` 处理采样请求，可按 `modelHint` 路由到不同模型 |
| **Elicitation（引导）** | `mcp-annotations/` | Server 通过 `McpSyncRequestContext.elicit(Person.class)` 向 Client 请求结构化表单数据；Client 通过 `@McpElicitation` 注解处理 |
| **Dynamic Tool Update** | `dynamic-tool-update/` | 运行时通过 `mcpSyncServer.addTool()` 动态增删工具，Client 通过 `toolsChangeConsumer` 回调感知变化 |
| **Resources & Prompts** | `mcp-annotations/` | `@McpResource` 暴露 URI 可访问的资源数据（如 `user-profile://{username}`），`@McpPrompt` 暴露提示词模板，`@McpComplete` 提供 URI/提示词的自动补全建议 |
| **Progress & Logging** | `mcp-annotations/` | Server 通过 `McpSyncRequestContext.progress()` / `ctx.info()` 发送进度和日志通知；Client 通过 `@McpProgress` / `@McpLogging` 接收 |
| **Reactive 支持** | `mcp-annotations/` | `AsyncToolProvider` 使用 `McpAsyncRequestContext` + Reactor `Mono` 实现全异步工具提供者 |
| **ToolContext 集成** | `sampling/` | Spring AI 原生 `@Tool` 方法可通过 `ToolContext` 参数获取 MCP 交换上下文：`McpToolUtils.getMcpExchange(toolContext)` |

#### 3.7 MCP 子模块总览

| 子模块 | 作用 | 传输方式 |
|--------|------|---------|
| `client-starter/`（2个） | Spring Boot 自动配置 MCP 客户端 | STDIO, WebFlux SSE |
| `brave-docker-agents-gateway/` | Docker 运行 Brave Search MCP Server → ChatClient 工具调用 | STDIO |
| `filesystem/` | 编程式创建 MCP 客户端，连接文件系统服务 | STDIO |
| `weather/`（4个） | 天气服务：3 种传输方式 × 2 种配置风格（手动 + Starter） | STDIO, WebFlux SSE, WebMVC SSE |
| `web-search/`（2个） | Brave Search：单次查询 + 交互式聊天机器人 | STDIO |
| `mcp-annotations/`（2个） | 全面展示所有 MCP 注解（Tool/Resource/Prompt/Complete 等） | STDIO, SSE, Streamable HTTP |
| `sampling/`（2个） | MCP Sampling：服务端请求客户端 LLM 生成内容 | STDIO, SSE |
| `sampling-annotations/`（2个） | 同上，但使用 `@McpTool` + `@McpSampling` 注解风格 | STDIO, SSE |
| `dynamic-tool-update/`（2个） | 运行时动态添加工具 + Client 感知变化 | STDIO |
| `mcp-apps-server/` | MCP Apps：HTML UI 作为 MCP 资源提供服务 | HTTP |

---

### 🍃 4. Advisors（顾问/拦截器机制）— `advisors/`

Advisor 是 Spring AI 的 **AOP 拦截器链模式**，类似 Spring MVC 的 Interceptor。在 LLM 调用的请求和响应阶段插入横切逻辑：

| 模块 | 核心能力 | 关键技术点 |
|------|---------|-----------|
| `recursive-advisor-demo/` | 自定义 `BaseAdvisor` 实现请求/响应日志记录 | 实现 `before()`/`after()` + `getOrder()` 控制顺序 |
| `tool-argument-augmenter-demo/` | 工具调用增强：LLM 调用工具前输出推理过程 | `AugmentedToolCallbackProvider<AgentThinking>` — 注入 `innerThought`、`confidence`、`memoryNotes` 元数据 |
| `evaluation-recursive-advisor-demo/` | **LLM-as-Judge**：Anthropic 生成 + Ollama 评分（1-4） | `SelfRefineEvaluationAdvisor` 实现 `CallAdvisor` + `StreamAdvisor`；低于阈值自动重试（最多 N 次） |

**LLM-as-Judge 自评估流程**：
```
User Prompt → [生成 LLM] → [评估 LLM 打分] → ≥阈值? → 通过返回
                                              → <阈值? → 追加反馈 → 重试 (最多 N 次)
```

---

### 🎯 5. Prompt Engineering（提示词工程）— `prompt-engineering/`

一个 600+ 行的综合示例，展示 12+ 种提示词工程技术及其在 Spring AI 中的具体 API 映射：

| 类别 | 技术 | Spring AI API 映射 | 关键参数 |
|------|------|-------------------|---------|
| **基础** | Zero-shot | `.entity(Sentiment.class)` | `temperature=0.1, maxTokens=5` |
| | One-shot / Few-shot | 在提示词中内嵌示例 JSON | `maxTokens=250` |
| **系统/角色** | System Prompt | `.system("instruction")` | — |
| | Role Prompt | `.system("act as a travel guide")` | — |
| | Contextual Prompt | `.user(u -> u.text("{context}").param(...))` | — |
| **高级推理** | Step-back | 两步：先问抽象问题 → 答案作上下文 → 再问具体问题 | `chatClient.mutate()` |
| | Chain-of-Thought (CoT) | "Let's think step by step" | `temperature=1.0` |
| | Self-consistency | 5 次采样 + 多数投票 | `temperature=1.0` |
| | Tree of Thoughts | 3 分支生成 → 评估选择最佳 → 深化 | 多次采样 |
| | Auto Prompt Engineering | 10 个提示词变体 → 模型自评最优 | BLEU 评估 |
| **代码** | Write / Explain / Translate | `.system()` + low temperature | `temperature=0.1` |

---

### 🟣 6. Kotlin 支持 — `kotlin/`

| 模块 | 用途 | 关键技术 |
|------|------|---------|
| `kotlin-hello-world` | Spring AI 在 Kotlin 中的最小示例 | `data class` + `entity<Joke>()` 结构化输出 |
| `kotlin-function-callback` | Kotlin Lambda 实现的函数回调 | Kotlin 风格的 `FunctionToolCallback` |
| `rag-with-kotlin` | **完整的 RAG 应用** | `VectorStore` + `QuestionAnswerAdvisor` + 向量检索 + ETL Pipeline |

---

### 🔧 7. Misc（杂项）— `misc/`

#### 7.1 Java Function Callback (`spring-ai-java-function-callback`)

经典的 `java.util.function.Function<Input, Output>` 函数回调模式：
- `WeatherRequest`（`@JsonClassDescription` + `@JsonPropertyDescription` 驱动 JSON Schema）
- `WeatherResponse`（不可变 POJO）
- 通过 `FunctionToolCallback.builder("name", function).inputType(...).build()` 注册

#### 7.2 流式响应 (`openai-streaming-response`)

```java
@RestController
public class ChatController {
    private final OpenAiChatModel chatModel;

    @GetMapping(value = "/generateStream", produces = TEXT_EVENT_STREAM_VALUE)
    public Flux<ChatResponse> generateStream(@RequestParam(defaultValue = "Tell me a joke") String message) {
        return chatModel.stream(new Prompt(new UserMessage(message)));
    }
}
```

直接注入 `OpenAiChatModel`（非 `ChatClient`），使用 `stream()` 返回 `Flux<ChatResponse>`，通过 SSE 推送到客户端。

#### 7.3 Claude Skills 文档生成 (`claude-skills-demo/`)

**完整的企业级 Web 应用**，展示 Anthropic Skills API 与 Spring AI 的深度集成：

- **技术栈**：Spring Boot + Thymeleaf + HTMX + Anthropic Claude Skills
- **支持的文档类型**：XLSX / PPTX / DOCX / PDF（通过 `AnthropicSkill` 枚举映射）
- **核心 API**：
  - `AnthropicChatOptions.builder().skill(AnthropicSkill.XLSX).build()` — 指定 Skill
  - `AnthropicSkillsResponseHelper.extractFileIds(ChatResponse)` — 提取生成文件 ID
  - `anthropicClient.beta().files().retrieveMetadata(fileId)` — 获取文件元数据
  - `anthropicClient.beta().files().download(fileId)` — 下载文件内容
- **异步生成**：PPTX 等长任务通过 `CompletableFuture.runAsync()` 异步执行
- **功能**：文件上传、异步生成、下载、历史管理、水印（自定义 Skill）

---

## 五、Spring AI 使用方式总结

### 5.1 配置层

```properties
# 模型供应商配置
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.anthropic.api-key=${ANTHROPIC_API_KEY}
spring.ai.ollama.base-url=http://localhost:11434

# MCP 客户端自动配置（STDIO）
spring.ai.mcp.client.stdio.connections.weather.command=java
spring.ai.mcp.client.stdio.connections.weather.args=-jar,weather-server.jar
spring.ai.mcp.client.toolcallback.enabled=true

# MCP 服务端自动配置
spring.ai.mcp.server.stdio=true
spring.ai.mcp.server.name=my-server
spring.ai.mcp.server.version=1.0.0
```

### 5.2 API 编程模型

```
ChatClient
  ├── .prompt()
  │     ├── .user() / .system()           — 提示词设置
  │     │     └── .param("key", value)     — 参数化提示词（{key} 占位符）
  │     ├── .tools(toolCallbackProvider)   — 工具绑定
  │     │     ├── @Tool 注解方法 (MethodToolCallbackProvider)
  │     │     ├── FunctionToolCallback
  │     │     └── SyncMcpToolCallbackProvider（MCP 工具代理）
  │     ├── .advisors(...)                 — 顾问拦截链
  │     │     ├── MessageChatMemoryAdvisor（对话记忆）
  │     │     ├── ToolCallingAdvisor（自动工具调用循环）
  │     │     ├── QuestionAnswerAdvisor（RAG 检索增强）
  │     │     └── 自定义 Advisor（日志、评估、增强）
  │     └── .options(ChatOptions)          — 控制 temperature / maxTokens / topK
  └── .call()
        ├── .content()                     — 纯文本输出
        ├── .entity(Record.class)          — 结构化输出（JSON → Java Record）
        └── .chatResponse()                — 完整 ChatResponse 对象
```

### 5.3 统一编程模型

所有模块共享统一的编程范式：

```
@SpringBootApplication
public class Application implements CommandLineRunner {

    @Bean
    CommandLineRunner runner(ChatClient.Builder builder, ToolCallbackProvider tools) {
        var chatClient = builder
            .defaultSystem("系统提示词")             // 全局系统提示词
            .defaultTools(tools)                    // 注册工具
            .defaultAdvisors(                       // 注册 Advisor 链
                new MessageChatMemoryAdvisor(memory),
                new ToolCallingAdvisor(),
                new MyLogAdvisor()
            )
            .build();
        return args -> {
            var response = chatClient.prompt()
                .user("用户输入")
                .call()
                .content();
            System.out.println(response);
        };
    }
}
```

### 5.4 能力矩阵

| 能力 | 核心 API | 应用场景 | 代表模块 |
|------|--------|----------|---------|
| 文本对话 | `chatClient.prompt().call().content()` | 聊天机器人、内容生成 | `helloworld` |
| 结构化输出 | `.call().entity(Record.class)` | 数据提取、分类、表单填充 | `routing-workflow`, `evaluator-optimizer` |
| 函数调用 | `@Tool` / `FunctionToolCallback` | 查询天气、数据库操作、API 调用 | `java-function-callback` |
| RAG 检索 | `VectorStore` + `QuestionAnswerAdvisor` | 知识库问答、文档检索 | `rag-with-kotlin` |
| 对话记忆 | `MessageChatMemoryAdvisor` + `MessageWindowChatMemory` | 多轮对话、Agent 循环 | `reflection`, `brave-chatbot` |
| 流式输出 | `Flux<ChatResponse>` + SSE | 实时打字效果、长文本生成 | `openai-streaming-response` |
| MCP 协议 | `@McpTool`, `SyncMcpToolCallbackProvider` | 跨进程工具调用、微服务 AI | 所有 MCP 模块 |
| 顾问拦截链 | `BaseAdvisor` / `CallAdvisor` | 日志、缓存、评估、增强 | `advisors/` |
| 多模型编排 | 多个 `ChatClient` + `CompletableFuture` | Agent 工作流、复杂推理 | `agentic-patterns/` |
| 多模型混合 | `AnthropicChatModel` + `OllamaChatModel` | LLM-as-Judge、模型路由 | `evaluation-recursive-advisor`, `sampling` |
| 文档生成 | `AnthropicChatOptions.skill()` | Office 文档生成（XLSX/PPTX/DOCX/PDF） | `claude-skills-demo` |

---

## 六、关键 Spring AI 类与接口速查

| Spring AI 类/接口 | 用途 | 代表模块 |
|---|---|---|
| `ChatClient` / `ChatClient.Builder` | 核心 LLM 调用入口（Fluent API） | 所有模块 |
| `ChatModel` | LLM 模型抽象接口（底层） | 底层依赖 |
| `ChatOptions` | 模型参数控制（temperature, maxTokens 等） | prompt-engineering |
| `ChatResponse` / `ChatMemory` | 模型响应 / 对话记忆 | reflection, brave-chatbot |
| `@Tool` / `@ToolParam` | Spring AI 原生工具注解 | MCP 服务端, advisors |
| `FunctionToolCallback` | 将 Java Function 包装为工具回调 | Java/Kotlin function-callback |
| `ToolCallbackProvider` | 工具回调提供者接口 | MCP client-starter |
| `ToolCallingAdvisor` | 自动工具调用循环 Advisor | recursive-advisor-demo |
| `SyncMcpToolCallbackProvider` | MCP 客户端工具代理 | filesystem, web-search |
| `MessageChatMemoryAdvisor` | 对话记忆顾问 | reflection, tool-argument-augmenter |
| `MessageWindowChatMemory` | 滑动窗口记忆实现 | reflection |
| `QuestionAnswerAdvisor` | RAG 检索顾问 | rag-with-kotlin |
| `BaseAdvisor` / `CallAdvisor` / `StreamAdvisor` | 自定义拦截器基类 | recursive-advisor, evaluation-advisor |
| `VectorStore` / `Document` | 向量存储 / 文档抽象 | rag-with-kotlin |
| `McpSyncClient` / `McpServer` | MCP 客户端/服务端 API | 所有 MCP 模块 |
| `McpClientCustomizer` | MCP 客户端定制器（sampling, progress, logging） | sampling, dynamic-tool-update |
| `StdioClientTransport` / `StdioServerTransportProvider` | MCP STDIO 传输 | 所有 MCP STDIO 模块 |
| `WebFluxSseServerTransportProvider` / `WebFluxSseClientTransport` | MCP WebFlux SSE 传输 | weather/manual-webflux-server |
| `HttpClientSseClientTransport` | MCP WebMVC SSE 客户端传输 | weather/starter-webmvc-server |
| `WebClientStreamableHttpTransport` / `HttpClientStreamableHttpTransport` | MCP Streamable HTTP 传输 | weather/starter-webflux-server |
| `McpSyncServerExchange` / `McpSyncRequestContext` | MCP 服务端交换上下文（sampling, elicitation, logging） | mcp-annotations-server |
| `@McpTool` / `@McpToolParam` | MCP 工具注解 | mcp-annotations, weather |
| `@McpResource` / `@McpPrompt` / `@McpComplete` | MCP 资源/提示词/补全注解 | mcp-annotations-server |
| `@McpArg` | MCP 提示词参数注解 | mcp-annotations-server |
| `@McpSampling` / `@McpProgress` / `@McpLogging` / `@McpElicitation` | MCP 客户端功能注解 | mcp-annotations-client, sampling-annotations |
| `@McpProgressToken` | MCP 进度令牌注解 | mcp-annotations-server |
| `AugmentedToolCallbackProvider<T>` | 工具调用增强器（注入元数据） | tool-argument-augmenter-demo |
| `MethodToolCallbackProvider` | 基于 `@Tool` 注解的工具提供者 | 多个模块 |
| `OpenAiChatModel.stream()` | OpenAI 流式输出 | openai-streaming-response |
| `AnthropicChatModel` / `AnthropicChatOptions` | Anthropic 特定能力（Skills） | claude-skills-demo |
| `AnthropicSkillsResponseHelper.extractFileIds()` | Claude Skills 文件 ID 提取 | claude-skills-demo |
| `Media` | 文件附件（PDF 等） | claude-skills-demo |

---

## 七、学习路线图

整个工程是一个 **Spring AI 从入门到精通的完整学习路线图**：

1. **入门** → `models/chat/helloworld`：一个 `ChatClient.prompt().call().content()` 即完成 LLM 调用，理解最基础的编程模型
2. **基础能力** → `misc/spring-ai-java-function-callback` + `misc/openai-streaming-response`：学习工具注册和流式输出的两种核心模式
3. **进阶** → **Agentic Patterns**（5 种模式）：学习如何编排多个 LLM 协作完成复杂任务，理解链式、路由、并行、编排、评估-优化五种工作流
4. **自我改进** → `agents/reflection`：学习双 LLM 自反思循环，理解 Generator-Critique 反馈机制
5. **工程化** → **MCP 系列**（20+ 模块）：学习生产级的工具集成协议，理解 STDIO/SSE/Streamable HTTP 三种传输方式、手动配置与 Starter 自动配置两种风格
6. **横切关注点** → **Advisors**（3 个 Demo）：学习拦截器链模式，包括日志、工具增强、LLM-as-Judge 自评估
7. **精细控制** → **Prompt Engineering**（12+ 模式）：学习从 Zero-shot 到 Tree-of-Thoughts 的提示词技巧，理解 `ChatOptions` 对 `temperature`/`maxTokens`/`topK` 的精细控制
8. **实战** → **RAG** + **Claude Skills**：端到端的知识库问答和文档生成应用
9. **多语言** → **Kotlin 模块**：Spring AI 在 Kotlin 中的一等公民支持，包括 RAG 完整实现

整个工程充分体现了 Spring AI 的核心理念：**让 AI 能力像数据库、消息队列一样，成为 Spring 开发者手中普通的、可组合的基础设施**。
