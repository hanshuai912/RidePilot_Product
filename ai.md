# 任务：NestJS 通用 AI 模块接入 Qwen3Guard 内容安全审核

## 一、任务背景

项目使用 Monorepo 架构，后端为 NestJS + TypeScript。

目前已经完成通用 AI 基础设施模块，包含：

- AiService：统一 AI 调用入口。
- AiOrchestratorService：统一调用编排。
- AiProvider：模型厂商抽象。
- OpenAI-Compatible Provider。
- GuardrailsService：审核编排服务。
- InputGuard 和 OutputGuard 接口。
- NoopGuardAdapter：开发测试用空审核实现。
- 通用请求、响应、错误处理和配置管理。

本次任务是在现有架构基础上，正式接入 Qwen3Guard 内容安全审核模型。

**必须复用现有 AI 基础设施，不允许重新实现一套 AI 系统。**

## 二、核心目标

本次需要实现：

1. Qwen3Guard Adapter。
2. 输入内容安全审核。
3. AI 输出内容安全审核。
4. Qwen3Guard 模型结果解析。
5. 统一安全等级和审核决策。
6. 审核拒绝响应。
7. 审核超时、异常和故障处理。
8. 审核相关配置管理。
9. 审核单元测试和真实服务联调。
10. Docker 开发部署说明。

完成之后，现有 AI 调用流程应当变为：

```text
业务模块
    ↓
AiService
    ↓
AiOrchestratorService
    ↓
InputGuard
    └── Qwen3Guard
           ├── Safe → 继续
           ├── Unsafe → 拒绝
           └── Controversial → 按策略处理
    ↓
AiProvider
    ↓
OutputGuard
    └── Qwen3Guard
           ├── Safe → 返回
           ├── Unsafe → 拒绝
           └── Controversial → 按策略处理
    ↓
AiResponse
```

业务模块不需要修改现有 AI 调用方式。

## 三、实施前检查

先阅读当前仓库中已有的 AI 模块实现，确认：

- GuardrailsService 当前如何调用 InputGuard、OutputGuard。
- NoopGuardAdapter 如何注册。
- Guard 接口和返回类型。
- Orchestrator 如何处理审核决策。
- 当前 ConfigModule 的配置方式。
- 现有 AI Provider 的 HTTP 客户端是否可以复用。
- 错误处理、日志与测试规范。
- Docker Compose 是否已经存在。

基于现有实际代码进行增量开发。

先简述修改计划，然后直接实施，无需等待二次确认。

## 四、Qwen3Guard Adapter

新增 `Qwen3GuardAdapter`，实现现有 Guard 接口，或者使用符合当前项目抽象的适配结构。

选择模型：

`Qwen/Qwen3Guard-Gen-0.6B`

模型官方项目：

https://github.com/QwenLM/Qwen3Guard

模型服务通过 HTTP 访问，使用 OpenAI-Compatible API。

要求：

1. 使用独立的审核模型配置。
2. 不与业务 LLM Provider 的配置混用。
3. 支持自定义 Base URL、Model、API Key、Timeout。
4. 支持审核请求失败和超时处理。
5. 支持异步调用。
6. 不依赖 Python 实现 NestJS Adapter。
7. 不要让 Qwen3Guard 的调用再次经过 AiOrchestratorService，以免递归审核。
8. Adapter 只负责与 Qwen3Guard 通信及解析结果，不包含具体业务逻辑。

配置示例：

```env
GUARDRAILS_MODE=qwen3guard

QWEN_GUARD_BASE_URL=http://localhost:8000/v1
QWEN_GUARD_MODEL=Qwen/Qwen3Guard-Gen-0.6B
QWEN_GUARD_API_KEY=EMPTY
QWEN_GUARD_TIMEOUT_MS=10000
```

环境变量名称可以根据现有规范调整，但要保持含义清晰。

不得将审核模型配置与业务 AI 配置绑定。

## 五、输入安全审核

通过 InputGuard 调用 Qwen3Guard。

对于普通用户输入，按官方 Prompt Moderation 方式构造请求，例如：

```typescript
messages: [
  {
    role: "user",
    content: userMessage,
  },
];
```

要求：

- 正确提取需要审核的用户输入。
- 不将 API Key、内部配置、系统提示词发送给审核模型。
- 不直接把整个 AiRequest 对象序列化为 Prompt。
- 为未来的多轮对话审核保留必要上下文能力。
- 历史上下文的提取必须有明确范围和长度限制。
- 超出审核模型可处理的输入长度时，不能直接截断后默认放行。
- 审核通过后才能调用真正的业务 AI。

对于模型检查结果为 Unsafe 的请求，立即终止调用链。

## 六、输出安全审核

通过 OutputGuard 审核业务 AI 生成的文本。

按照 Qwen3Guard 官方 Response Moderation 方式，将对应用户输入与待审核的 AI 回复一起提交：

```typescript
messages: [
  {
    role: "user",
    content: userMessage,
  },
  {
    role: "assistant",
    content: aiResponse,
  },
];
```

要求：

1. 对最终返回给用户的 AI 文本进行审核。
2. 不把审核模型自己的分类结果当成业务 AI 回复。
3. 输出审核拒绝后，不得返回未经审核的生成文本。
4. 不对审核结果自动执行未经设计的文本修改。
5. 审核失败时返回统一错误。
6. 保持现有 AiResponse 结构兼容。

当前项目只实现普通非流式文本生成审核。

暂时不实现 Qwen3Guard-Stream。

## 七、审核结果解析

Qwen3Guard-Gen 返回的是特定格式的分类文本，不是标准 JSON。

典型输入审核结果：

```text
Safety: Safe
Categories: None
```

或者：

```text
Safety: Unsafe
Categories: Jailbreak
```

输出审核结果还可能包含：

```text
Refusal: Yes
```

需要实现独立的结果解析器。

至少解析：

- Safety Level。
- Categories。
- Refusal（如果存在）。

Safety Level 包含：

- Safe
- Controversial
- Unsafe

要求：

- 严格解析和校验模型返回格式。
- 支持官方定义的安全类别。
- 正确处理多个类别。
- 区分 Categories 和 Safety Level。
- Refusal 字段不能直接等同于 Unsafe。
- 模型返回空字符串、异常格式、未知安全等级时视为审核异常。
- 不允许解析失败时默认返回 ALLOW。
- 不将 Qwen3Guard 原始结果类型直接暴露给业务模块。

解析逻辑必须具有独立单元测试。

## 八、安全等级与审核决策

沿用现有通用 GuardDecision，不要无必要地重构 Guard 接口。

初始策略：

| 安全等级      | 处理   |
| ------------- | ------ |
| Safe          | ALLOW  |
| Unsafe        | REJECT |
| Controversial | REVIEW |

如果当前没有人工复核流程，REVIEW 应在本阶段视为不可继续调用或返回的状态，并提供独立错误码。

不要将 REVIEW 默认转为 ALLOW。

安全等级与最终决策应当分离，方便未来调整策略。

本次只做通用内容安全，不实现具体业务的敏感话题策略。

## 九、审核异常与故障处理

至少处理：

- 服务无法连接。
- 请求超时。
- HTTP 4xx / 5xx。
- API 认证失败。
- 模型不存在。
- 返回体为空。
- 返回结构不符合预期。
- 响应解析失败。
- 审核内容超出模型处理范围。

统一错误处理规则：

1. 审核拒绝与审核服务故障必须使用不同错误码。
2. 输入审核失败不得继续调用业务 AI。
3. 输出审核失败不得直接返回业务 AI 原始输出。
4. 不允许审核失败后自动切换为 NoopGuard。
5. 不允许隐藏异常后继续放行。
6. 限制不必要的重试，避免增加延迟和重复成本。
7. 不向前端暴露审核服务内部异常或敏感配置。

建议预留以下错误码：

```text
AI_INPUT_REJECTED
AI_OUTPUT_REJECTED
AI_GUARD_REVIEW_REQUIRED
AI_GUARD_UNAVAILABLE
AI_GUARD_TIMEOUT
AI_GUARD_INVALID_RESPONSE
```

如果项目已有统一错误码规范，以现有规范为准。

## 十、Guardrails 配置和依赖注入

现有项目已经拥有 NoopGuardAdapter。

本次需要支持：

```text
GuardrailsService
    │
    ├── NoopGuardAdapter
    │
    └── Qwen3GuardAdapter
```

通过 NestJS DI 和环境配置选择实现。

要求：

- development/test 环境可以显式使用 NoopGuard。
- integration 环境可以使用真实 Qwen3Guard。
- production 环境必须启用真实审核。
- 生产环境如果配置 NoopGuard，必须启动失败。
- 生产环境审核模型配置缺失时必须启动失败。
- 审核服务故障不能静默降级为 NoopGuard。
- 切换 Guard Adapter 不需要修改业务模块。

避免直接在业务代码中使用 if/else 判断具体审核模型。

## 十一、日志与监控

复用项目现有日志体系。

至少记录：

- Request ID。
- 审核阶段（Input / Output）。
- 模型名称。
- Safety Level。
- 决策结果。
- 风险类别。
- 审核耗时。
- 错误类型。

默认不记录完整用户输入、AI 回复、API Key 或其他敏感信息。

对于 Qwen3Guard 原始响应，只有在受控测试中才能临时输出调试内容。

## 十二、Docker 与部署

本次需要提供可复现的 Qwen3Guard 开发环境部署方式。

优先采用：

- Docker
- vLLM
- Qwen3Guard-Gen-0.6B

具体要求：

1. 查看现有 Docker Compose 结构。
2. 尽量增加一个独立、可选的 Qwen3Guard 服务。
3. 不影响现有 NestJS、PostgreSQL、Redis 等服务。
4. 模型文件需要支持缓存复用。
5. 模型端口不能默认公开暴露到公网。
6. 模型服务和 NestJS 服务通过 HTTP 通信。
7. 如果 NestJS 在容器内，正确使用容器服务名连接。
8. 提供启动、停止和健康检查说明。
9. 提供不依赖业务 Controller 的联调脚本。

不要假设开发机器一定拥有 NVIDIA GPU。

如果当前机器没有必要的 GPU 运行环境，应当仍然完成 NestJS Adapter、测试及部署配置，并明确标记真实模型联调未完成，不得伪造运行结果。

优先使用经过官方验证、版本兼容的部署方法，不盲目采用过时命令。

## 十三、测试要求

编写单元测试和可选择执行的真实服务集成测试。

至少覆盖：

1. 正常输入 Safe → 调用 Provider。
2. 输入 Unsafe → 不调用 Provider。
3. 输入 Controversial → 进入 REVIEW 处理。
4. 业务 AI 输出 Safe → 正常返回。
5. 业务 AI 输出 Unsafe → 拦截。
6. Qwen3Guard 超时 → 返回统一故障。
7. Qwen3Guard 连接失败 → 不得放行。
8. 模型返回无效格式 → 不得放行。
9. 输入与输出阶段分别调用了正确的审核方法。
10. 不同 Guard Adapter 能通过 DI 替换。
11. 生产环境不能使用 NoopGuard。
12. 业务 AI Provider 配置与审核 Provider 配置互不影响。

所有单元测试使用 Mock HTTP 响应，不依赖实际 GPU。

另外提供一组中文测试样本，覆盖：

- 普通安全问题。
- 明显违规问题。
- Prompt Injection 尝试。
- 边界或争议内容。
- 安全的 AI 回复。
- 不安全的 AI 回复。

真实分类结果以模型实际输出为准，不要假设模型对所有测试样本都会作出预期判断。

## 十四、本次明确不做

严格限制开发范围。

不开发：

- App 业务范围审核 DomainGuard。
- 训练、营养等业务规则。
- AI Agent。
- 业务 Prompt。
- React Native 页面。
- 业务数据库。
- Qwen3Guard-Stream。
- RAG 或向量数据库。
- 复杂 AI 工作流。
- 管理后台或人工审核平台。

不修改现有业务 AI 模型的选择与配置逻辑。

## 十五、最终验收标准

- [ ] Qwen3Guard Adapter 已实现。
- [ ] InputGuard 已能通过 Adapter 审核真实请求。
- [ ] OutputGuard 已能通过 Adapter 审核真实回复。
- [ ] Safe / Unsafe / Controversial 正确解析。
- [ ] 模型返回异常格式时不会默认放行。
- [ ] 审核拒绝能正确中止 AI 调用。
- [ ] 审核故障具有统一错误处理。
- [ ] NoopGuard 与 Qwen3Guard 可配置切换。
- [ ] 生产环境禁止使用 NoopGuard。
- [ ] 提供独立审核配置。
- [ ] 提供 Docker 开发部署配置或明确部署说明。
- [ ] 提供单元测试和真实联调方法。
- [ ] 不修改现有业务模块的 AI 调用方式。
- [ ] 不开发任何具体业务内容。

## 十六、完成后的交付报告

请完成实际代码，不要只给架构说明。

最终汇报：

1. 新增和修改的文件。
2. Qwen3Guard Adapter 设计与位置。
3. Guardrails 的实际调用流程。
4. 环境变量配置。
5. Docker 启动方法。
6. 如何验证 Safe / Unsafe / Controversial。
7. 输入与输出审核测试结果。
8. 真实模型是否已完成联调。
9. 遇到的环境或部署问题。
10. 后续增加 DomainGuard 的扩展位置。

**最终要求：在现有 NestJS AI 基础设施之上增量接入 Qwen3Guard，实现完整的输入与输出安全审核闭环，保持通用、低耦合，不引入业务逻辑，不进行无关重构。**
