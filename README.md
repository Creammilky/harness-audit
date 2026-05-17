# harness-audit

下面这个 prompt 可以直接复用。把目标 Agent Framework / Harness 的关键代码贴进去，它会按模型请求层、响应解析层、执行层、状态管理层、测试层去找问题。

```
你是一个资深 Agent Harness / LLM Systems / Benchmark Infrastructure 工程 reviewer。

我会给你一段 Agent Framework 或 Agent Harness 的代码。请你像审查真实生产系统一样，系统性找出其中的工程坑点、隐含假设、模型兼容性问题、response parsing 问题、retry 问题、stop token 问题、tool execution 安全问题、benchmark reproducibility 问题，以及测试覆盖缺口。

请严格基于我提供的代码分析。不要泛泛而谈。每个问题都要指出对应代码片段、触发条件、影响、修复方案和建议测试。

重点审查这些方面：

1. 模型请求层
- 是否混淆不同 provider / endpoint / SDK 的参数名，例如 stop、stop_sequences、max_tokens、max_completion_tokens、temperature。
- 是否硬编码模型名判断，例如 if "o1" in model_name。
- 是否有模型能力配置，例如 supports_stop_sequences、supports_temperature、is_reasoning_model、supports_tools。
- 是否正确处理 reasoning model 的特殊行为，例如不可见 reasoning tokens、temperature 限制、stop 不支持、token budget 被 reasoning 消耗。
- 是否区分 HELM / non-HELM / OpenAI / Azure / Anthropic / local model 的 response schema。

2. Retry 和错误处理
- 是否把所有 HTTPError 都重试，导致 400/401/403/404 这种非 transient error 被无意义重试。
- 是否只重试 429、500、502、503、504、connection reset、timeout 等 transient failures。
- 是否设置了合理的最大重试次数、总超时时间、指数退避和 jitter。
- 是否记录 request id、provider response id、raw request、raw response。
- 是否考虑重复请求导致重复扣费、非确定性输出、benchmark 不可复现。

3. Stop token / stop sequence
- STOP_TOKEN 是否只是普通字符串，而不是特殊 token。
- prompt 是否要求模型输出 stop token，API 是否又传了 stop sequence。
- provider 是否会从返回文本中移除 stop sequence。
- 如果 API 不支持 stop，代码是否有 fallback。
- 是否存在 stop token 太短、误截断、出现在正常命令里的风险。
- 是否手动追加 STOP_TOKEN，追加后是否会影响 parser。

4. Response parsing
- 是否用自然语言标签硬解析，例如 Command:、Action:、Thought:。
- 是否只找第一个 Command，多个 Command 时会怎样。
- 是否可能把解释文本、Please note、Markdown、代码块、系统消息泄漏一起解析成命令。
- 是否处理空命令、超长命令、多行命令、多个 action、缺失 action。
- 是否有 schema validation。
- 是否返回 parse error 原因，而不是简单 Optional / None。
- 是否保留 raw_response、cleaned_response、parsed_action 方便 debug。
- 是否应该改成 JSON schema、tool call、或显式 <COMMAND>...</COMMAND> block。

5. Hallucination / prompt leakage 清理
- 是否只是简单字符串替换。
- 是否可能漏掉其他 role marker、system prompt、assistant message wrapper。
- 是否可能误删合法内容。
- 是否清理前后都有日志。
- 是否应该把这类情况作为 invalid output 触发 regenerate，而不是静默清理。

6. Tool / shell command 执行安全
- parser 是否直接把模型输出交给 shell。
- executor 是否设置 sandbox、cwd、env、timeout、resource limit。
- 是否使用 shell=True；如果使用，是否有必要，是否隔离。
- 是否有 allowlist/denylist。
- 是否阻止危险命令、系统目录写入、越权访问、外网访问、秘密泄露。
- 是否记录 stdout/stderr/exit code/duration。
- 是否防止命令输出过大导致 prompt/context 爆炸。

7. Agent loop / 状态管理
- observation 是否截断、摘要、去重。
- 长上下文是否有 compaction 策略。
- 是否会把旧错误、旧计划、旧 hallucination 反复传回模型。
- 是否区分 task state、iteration state、model state、environment state。
- 是否有 max iterations、early stop、failure reason。
- 是否处理 flag found / submit action / no-op / wait / retry action 等不同 action type。

8. Benchmark reproducibility
- 是否固定 random seed，或至少记录无法固定的来源。
- 是否记录模型版本、deployment name、temperature、token limit、prompt hash、response hash。
- mock_calls 是否真实覆盖 parser 和 executor。
- mock 文件路径是否依赖当前工作目录。
- 是否有 golden tests。
- 是否能重放一次完整 agent trajectory。

9. Observability
- 日志是否包括 raw prompt、raw response、cleaned response、parsed command、parse error、finish_reason、usage、latency、retry count。
- 是否避免把 API key、secrets、flag 等敏感信息写入日志。
- 是否有结构化日志。
- 是否能定位失败来自 request、parse、execution、environment、还是 model behavior。

10. 测试建议
- 请给出具体测试用例，不要只说“增加测试”。
- 包括正常 case、缺失 Command、多 Command、Markdown fence、trailing explanation、空命令、stop token 被移除、stop token 未被移除、模型返回 tool call、finish_reason=length、content_filter、网络重试、mock path 错误等。

请按以下格式输出：

## 总体判断
用 3-5 句话说明这段代码属于 prototype、benchmark harness、还是接近 production，以及最大风险在哪里。

## 问题清单
用表格输出：
| 严重程度 | 位置/代码片段 | 问题 | 触发条件 | 影响 | 修复方案 | 建议测试 |

严重程度分为：Critical / High / Medium / Low。

## 推荐重构方案
给出分层方案：
- ModelAdapter
- ResponseNormalizer
- ActionParser
- Executor
- StateManager
- Logger / TraceRecorder
- Test Harness

## 可直接替换的代码示例
如果代码里有 parser、retry、model capability config、mock path 等问题，请给出更稳的替代代码。

## 最小测试集
给出 pytest 风格的测试用例。

下面是要审查的代码：

[把 Agent Harness 代码贴在这里]
```
