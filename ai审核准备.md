# Qwen3Guard 接入准备清单

## 一、项目基础环境

- [x] NestJS 通用 AI 模块已完成
- [x] AiService 统一入口已实现
- [x] AiOrchestratorService 已实现
- [x] InputGuard / OutputGuard 接口已预留
- [x] NoopGuardAdapter 已实现
- [ ] 确认 Qwen3Guard 模型版本：Qwen3Guard-Gen-0.6B
- [ ] 确定模型部署位置：本地 / 远程服务器

## 二、模型运行环境

- [ ] 确认开发机器操作系统（Windows / Linux / macOS）
- [ ] 确认是否有 NVIDIA GPU
- [ ] 确认 GPU 显存大小
- [ ] 安装 Docker
- [ ] 如采用 NVIDIA GPU Docker 部署，安装 NVIDIA Container Toolkit
- [ ] 确认 Docker 能够访问 GPU
- [ ] 确认服务器能够下载 Qwen3Guard 模型
- [ ] 从 Hugging Face 或 ModelScope 下载模型，或配置自动下载
- [ ] 确认模型文件缓存目录

备注：如果使用远程模型服务，不需要在本机准备 GPU。如果暂时只开发 Adapter，可以跳过本部分的模型部署工作。

## 三、Qwen3Guard 服务部署

- [ ] 部署 Qwen3Guard-Gen-0.6B
- [ ] 使用 vLLM 或其他官方支持的推理方式
- [ ] 配置模型服务端口（建议开发环境使用 8000）
- [ ] 确认模型服务正常启动
- [ ] 确认 `/v1/chat/completions` 接口可以调用
- [ ] 测试普通中文输入能返回有效的安全分类
- [ ] 确认 NestJS 可以访问模型服务
- [ ] 如果远程访问，配置网络访问限制与服务认证

## 四、NestJS 环境变量

需要准备以下配置：

```env
GUARDRAILS_MODE=qwen3guard

QWEN_GUARD_BASE_URL=http://localhost:8000/v1
QWEN_GUARD_MODEL=Qwen/Qwen3Guard-Gen-0.6B
QWEN_GUARD_API_KEY=EMPTY
QWEN_GUARD_TIMEOUT_MS=10000
```

检查事项：

- [ ] 配置审核模型 Base URL
- [ ] 配置模型名称
- [ ] 配置 API Key（本地无鉴权服务可使用 EMPTY）
- [ ] 配置请求超时时间
- [ ] 确认审核配置与业务 AI Provider 配置独立
- [ ] 确认生产环境禁止 NoopGuard

## 五、审核策略

建议第一版使用以下策略：

- [ ] Safe：允许继续调用 AI
- [ ] Unsafe：直接拦截
- [ ] Controversial：进入 REVIEW，暂不放行
- [ ] 审核请求超时：拒绝继续调用
- [ ] 审核服务不可用：拒绝继续调用
- [ ] 模型返回格式异常：拒绝继续调用
- [ ] 输入审核未通过：禁止调用业务 AI
- [ ] 输出审核未通过：禁止返回未经审核的文本

## 六、联调测试数据

- [ ] 准备正常中文问题
- [ ] 准备明显违规问题
- [ ] 准备 Prompt Injection 测试问题
- [ ] 准备存在争议的边界问题
- [ ] 准备安全的 AI 回复样本
- [ ] 准备不安全的 AI 回复样本
- [ ] 测试 Safe / Unsafe / Controversial 分类解析
- [ ] 测试审核服务断开连接
- [ ] 测试审核超时
- [ ] 测试模型返回无效格式
- [ ] 测试输入拒绝后业务 AI 不被调用
- [ ] 测试输出拒绝后原始内容不返回客户端

## 七、交给 Codex 完成的工作

以下工作不需要提前手动实现：

- [ ] Qwen3GuardAdapter 开发
- [ ] InputGuard 与 Qwen3Guard 集成
- [ ] OutputGuard 与 Qwen3Guard 集成
- [ ] 审核结果解析器
- [ ] GuardDecision 映射逻辑
- [ ] 审核异常和超时处理
- [ ] NestJS 依赖注入配置
- [ ] Docker Compose 服务配置
- [ ] Mock 单元测试
- [ ] 真实模型联调脚本
- [ ] 接入文档与启动说明

## 八、最终验收

- [ ] NestJS 能成功连接 Qwen3Guard
- [ ] 用户输入在调用业务 AI 前经过审核
- [ ] AI 输出在返回客户端前经过审核
- [ ] 审核拒绝时正确中断调用
- [ ] 审核故障不会默认放行
- [ ] 现有业务 AI 调用方式不变
- [ ] Qwen3Guard 与业务 AI 使用独立配置
- [ ] 审核单元测试通过
- [ ] 真实模型联调通过
- [ ] 生产环境未使用 NoopGuard

---

## 官方项目地址

GitHub：
https://github.com/QwenLM/Qwen3Guard

模型下载：
https://huggingface.co/Qwen/Qwen3Guard-Gen-0.6B

国内 ModelScope：
https://modelscope.cn/models/Qwen/Qwen3Guard-Gen-0.6B

vLLM 部署文档：
https://docs.vllm.ai/

## 当前最优先的准备顺序

1. 确定模型运行位置。
2. 检查 GPU 和 Docker 环境。
3. 下载并启动 Qwen3Guard 模型。
4. 验证模型 HTTP API。
5. 配置 NestJS 环境变量。
6. 让 Codex 实现 Qwen3GuardAdapter 和审核流程。
7. 运行真实联调测试。
8. 验证异常处理和审核拦截效果。

如果暂时没有 GPU，可以先执行第 5、6 步，通过 Mock HTTP 测试验证适配器；真实模型部署与联调
