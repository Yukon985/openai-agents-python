# OpenAI 风格 与 Anthropic（Claude）风格：Agent/SDK 级别架构对比

> **摘要**：本文从 Agent/SDK 层面，系统梳理 OpenAI API 风格与 Anthropic Claude API 风格在架构设计上的核心异同，重点分析两者在消息模型、函数调用、流式响应、错误处理、认证配置、计费与上下文窗口以及提示安全性等方面的关键差异，并给出可落地的设计建议，供工程师在多模型适配或架构选型时参考。

---

## 1. 高层总体架构对比

### 1.1 共同核心模块

两种风格的 Agent/SDK 在以下模块上高度相似，均提供：

- **模型调用层**：封装 HTTP 请求、响应解析与重试逻辑。
- **工具/函数调用**：允许模型在推理过程中调用外部工具并将结果返回给模型。
- **会话/上下文管理**：在多轮对话中维护历史消息，控制上下文窗口。
- **流式输出**：通过 Server-Sent Events（SSE）实时推送 token 或事件。
- **安全与防护层**：包括输入过滤、输出验证和内容策略。

### 1.2 OpenAI 风格特点

| 维度 | 说明 |
|---|---|
| **消息格式** | 采用 `role`（`system` / `user` / `assistant` / `tool`）+ `content` 的扁平列表，结构简洁。 |
| **函数调用** | 通过 `tools` 字段声明工具，模型返回 `tool_calls` 数组，SDK 负责执行后以 `role: tool` 的消息回传结果。 |
| **流式协议** | 使用 `delta` 增量字段，每帧仅传输新增内容，带宽效率较高。 |
| **Assistants/Responses API** | 提供服务端托管对话（`thread`、`run`、`conversation_id`），可减少客户端状态管理负担。 |
| **模型多样性** | 覆盖 GPT 系列、o 系列推理模型（o1、o1-mini、o3、o3-mini 等）、Realtime API、Embeddings、图像等多模态统一入口。 |

**设计建议**：
- 若已基于 OpenAI SDK，优先使用 `Responses API` 的 `conversation_id` 管理多轮对话，可避免手动拼接历史消息带来的 bug。
- 工具结果回传时务必保证 `tool_call_id` 与请求严格对应，否则模型会拒绝继续推理。

### 1.3 Anthropic（Claude）风格特点

| 维度 | 说明 |
|---|---|
| **消息格式** | `messages` 列表只接受 `user` 和 `assistant` 两种 role；`system` 提示作为顶层独立字段传入，不混入消息列表。 |
| **函数调用** | 工具定义结构与 OpenAI 类似，但模型返回 `tool_use` 类型的 content block，结果以 `tool_result` block 的形式放入下一条 `user` 消息。 |
| **流式协议** | 以 `content_block_delta` 事件驱动，区分文本流与工具调用流；事件语义更细粒度，解析逻辑稍复杂。 |
| **宪法 AI / 安全层** | Claude 在模型层面内嵌了 Constitutional AI 原则，默认拒绝范围更广；安全行为不完全可通过参数覆盖。 |
| **Extended Thinking** | 支持 `thinking` 类型 content block，公开推理链，便于调试复杂任务。 |

**设计建议**：
- 在 SDK 适配层中，将 `system` 提示单独抽离为顶层参数，而非将其作为第一条 `user` 消息的前缀，以符合 Claude 规范并避免截断。
- 处理流式响应时，需同时监听 `content_block_start`、`content_block_delta`、`content_block_stop` 三种事件，建议封装专用的流解析器。

---

## 2. 架构设计上的具体差异与影响

### 2.1 消息 / Role Schema

- **差异**：OpenAI 支持 `system`、`user`、`assistant`、`tool` 四种 role 混合排列；Claude 要求 `system` 独立、消息列表严格交替（`user → assistant → user …`）。
- **影响**：多模型适配器若直接转发同一消息列表，会导致 Claude 报 `invalid_request_error`。
- **建议**：在适配层中，提取并合并所有 `system` role 消息为单一 `system` 字段，并验证 `user/assistant` 交替顺序，必要时插入占位消息。

### 2.2 函数调用 / Tools

- **差异**：OpenAI 的工具结果以独立的 `tool` role 消息回传；Claude 将工具结果内嵌于 `user` 消息的 `content` 数组中（`tool_result` block）。
- **影响**：共享工具执行逻辑时，需要在消息组装阶段做模型感知（model-aware）的分支处理，否则两侧均会报解析错误。
- **建议**：将工具结果封装为统一的内部数据结构，在序列化阶段按目标模型类型转换，保持业务逻辑层无感知。

### 2.3 流式响应

- **差异**：OpenAI 使用 `choices[0].delta` 结构；Claude 使用事件类型区分（`text_delta`、`input_json_delta` 等）。
- **影响**：直接复用同一流解析器会导致字段缺失或工具调用参数累积错误。
- **建议**：为每种 API 风格实现独立的流解析适配器，对外暴露统一的异步生成器接口（如 `AsyncGenerator[StreamEvent, None]`），上层消费者无需关心底层差异。

### 2.4 错误、速率限制与重试

- **差异**：OpenAI 使用 HTTP 429 + `Retry-After` 头；Claude 同样使用 429，但错误体结构（`type`、`error.type`）不同，且有 `overloaded_error` 这一特有类型。
- **影响**：通用重试中间件若仅匹配 HTTP 状态码，可能遗漏 Claude 的 `overloaded_error`（有时以 500 返回）。
- **建议**：重试策略应同时检查 HTTP 状态码和响应体中的错误类型字段，对 `overloaded_error` 使用指数退避（建议初始间隔 1 秒，最大 60 秒）。

### 2.5 认证与配置

- **差异**：OpenAI 使用 `Authorization: Bearer <api_key>`；Claude 使用 `x-api-key: <api_key>` + `anthropic-version` 请求头。
- **影响**：共享 HTTP 客户端时，需为不同 provider 注入不同的请求头，若配置混用会导致认证失败。
- **建议**：将认证逻辑封装于各自的 Provider 类中，禁止在通用 HTTP 层硬编码 header 名称，通过依赖注入传递。

### 2.6 Token、计费与上下文窗口

- **差异**：两者的分词器（tokenizer）不同，相同文本消耗 token 数存在差异（通常在 5%–15% 范围内）；Claude 的上下文窗口最大可达 200K tokens，但实际有效推理质量在超长上下文末尾有所下降。
- **影响**：基于 token 数的预算估算、截断策略若直接复用，可能导致超额收费或上下文截断位置不准确。
- **建议**：接入各模型官方 tokenizer（OpenAI 的 `tiktoken`，Anthropic 的 `tokenizer` 接口）进行精确计数；截断策略优先保留最新消息和 `system` 提示，中间历史视重要性滑动窗口删减。

### 2.7 提示工程与安全性

- **差异**：Claude 的宪法 AI 层对越狱、有害内容的拒绝阈值更保守，且部分安全行为无法通过 API 参数关闭；OpenAI 提供 `moderation` API 作为独立的内容审核端点，灵活性更高。
- **影响**：同一系统提示在 Claude 上可能被拒绝，而在 OpenAI 上正常执行，导致 Agent 行为不一致。
- **建议**：在多模型场景下维护模型感知的提示模板库；将内容策略测试纳入 CI 流程，覆盖两类模型的边界场景，确保行为一致性。

---

## 结论

OpenAI 风格与 Anthropic Claude 风格在 Agent/SDK 层面的核心差异集中于**消息 schema、工具调用回传格式和流式事件模型**三个维度，其余差异（认证、重试、计费）属于工程实现层面的适配问题，改造成本相对可控。

### 推荐短期行动项

1. **统一消息规范化层**：在 SDK 内部引入 `MessageNormalizer`，负责将内部消息列表按目标模型规范序列化，隔离业务逻辑与模型差异。
2. **拆分流解析适配器**：为 OpenAI 和 Claude 各实现一套流解析器，对外提供统一的 `StreamEvent` 抽象，减少上层耦合。
3. **增强重试中间件**：扩展错误类型匹配范围，覆盖 Claude 特有的 `overloaded_error`，并统一退避策略配置接口。
4. **Token 计数精确化**：集成各 provider 的官方 tokenizer，将 token 预算估算误差控制在 2% 以内。
5. **提示兼容性测试**：在 CI 中针对两类模型运行提示安全性与功能一致性测试，尽早发现因模型安全策略差异导致的行为分歧。
