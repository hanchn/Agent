# 目录

* 0. 课程导航与产出
* 1. Monorepo 与脚手架
* 2. 环境与配置（Secrets / 多环境）
* 3. 核心模块设计（LLM 适配、工具、记忆、RAG）
* 4. API 契约与服务编排
* 5. 前端（Node SSR）集成与流式渲染
* 6. 评测（Eval）与回归集
* 7. 观测性（日志/Trace/成本）
* 8. 安全与合规（权限、越权、注入、CORS）
* 9. 部署与运维（无服务器/容器/K8s）
* 10. 成本与性能优化打法
* 11. 实战专题A：电商选品&上新 Agent
* 12. 实战专题B：前端异常诊断 Agent
* 13. 实战专题C：飞书周/月报自动化 Agent
* 14. 项目清单、Checklist 与SLA

---

## 0. 课程导航与产出

**目标**：12 周内完成“从0到上线”的 Agent 能力建设，并可在现有业务中稳定落地三类 Agent（电商/前端运维/协同自动化）。

**最终产出**：

* 可运行的 **Agent 服务**（Node/TS）+ **RAG 知识库**（Redis-Vector）+ **工具集成**（HTTP/DB/Feishu/计算）。
* **API 契约文档**（OpenAPI 3.1）与 **前端 SSR 集成示例**（流式渲染/函数调用）。
* **评测基准**（JSONL 回归集 + 自动打分脚本）与 **监控仪表盘**（请求/成本/工具成功率）。
* **SOP**（开发、发布、回滚、应急、权限开通、密钥轮换、成本治理）。

**里程碑（建议）**：

* W1–W2：最小 Agent + 工具调用 Demo；
* W3–W4：RAG/记忆打通；
* W5–W6：SSR 集成 + 观测接入；
* W7–W8：评测体系与安全栅栏；
* W9–W10：业务专题 A/B 上线试运营；
* W11：工作流与人机协同；
* W12：性能与成本冲刺、复盘。

---

## 1. Monorepo 与脚手架

**技术栈**：pnpm + Nx/Turborepo（任选），TypeScript 全量类型约束。

**仓库结构（示例）**：

```
repo/
  package.json
  pnpm-workspace.yaml
  apps/
    agent-service/            # Node/TS（Fastify or Nest）
    web-ssr/                  # Node SSR（现有 C 端）
    tooling-scripts/          # 评测/数据/迁移脚本
  packages/
    llm-adapter/              # 模型抽象层（OpenAI/Claude/本地）
    rag-engine/               # RAG 管线（切分/嵌入/检索/重排）
    tool-sdk/                 # 工具封装（HTTP/SQL/Feishu/计算）
    shared-schema/            # zod/pydantic-ts Schema&Types
    obs-sdk/                  # 观测封装（logger/trace/cost）
  infra/
    docker/                   # Dockerfile 与 compose
    k8s/                      # Helm/Manifests
    nginx/                    # 反向代理/灰度配置
    scripts/                  # CI/CD、密钥轮换、备份
```

**快速初始化（指令建议）**：

* `pnpm dlx create-nx-workspace@latest` 或 Turborepo；
* 统一 `tsconfig.json` 与 `eslint`、`prettier`、`lint-staged`；
* Husky + commitlint（约定式提交，自动版本/CHANGELOG）。

---

## 2. 环境与配置（Secrets / 多环境）

**环境**：`local` / `dev` / `staging` / `prod`

**配置载体**：`.env` + KMS（云密钥）/ Vault；不在代码库存储明文密钥。

**关键变量**：

* LLM：`OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `AZURE_OPENAI_*`
* 向量：`REDIS_URL`（Redis Stack：RediSearch + Vector）
* 数据：`MYSQL_URL`（业务库只读帐号）
* 缓存/会话：`REDIS_URL`、`SESSION_TTL`
* 观测：`OTLP_ENDPOINT`（Jaeger/Tempo），`PROM_PUSHGATEWAY`
* 安全：`ALLOWED_ORIGINS`、`TOOL_WHITELIST`
* 飞书：`LARK_APP_ID`、`LARK_APP_SECRET`、`LARK_BOT_WEBHOOK`

**密钥轮换SOP**：按季度轮换；灰度重载；最小权限；审计日志。

---

## 3. 核心模块设计（LLM 适配、工具、记忆、RAG）

### 3.1 LLM 适配层（packages/llm-adapter）

* 能力：文本/多模态、函数调用、JSON 模式、流式。
* 接口：`generate()`, `stream()`, `toolCall()`, `embed()`。
* 降级：主模型不可用时自动切备份；上下文>32k时自动摘要。
* 输出：统一 Result 对象（tokens/latency/cost/finish\_reason）。

### 3.2 工具（packages/tool-sdk）

* 规范：纯函数签名 + zod Schema 校验 + 幂等设计。
* 通用工具：

  * `http.get/post()`（带重试/超时/熔断）；
  * `db.query()`（MySQL 只读，限制白名单 SQL）；
  * `calc.run()`（沙箱 Python/JS，资源配额）；
  * `file.read()`（受控目录）；
  * `search.web()`（可选）；
  * `feishu.*`（群消息、成员、工时查询）。
* 权限：`TOOL_WHITELIST` + 细粒度 Acl（哪类 prompt 能调哪些工具）。

### 3.3 记忆（短/长）

* 短期：会话窗/摘要；
* 长期：`user_profile`（MySQL）、`conversation_memory`（Redis Hash）；
* 策略：15 分钟活跃会话自动扩展；超长会话切片+摘要归档。

### 3.4 RAG 引擎（packages/rag-engine）

* 切分：基于 Markdown/段落/句向量；
* 嵌入：`llm.embed()`（批量 + 缓存）；
* 索引：Redis Vector（HNSW/Flat，1536/3072 维）+ 元数据 Hash；
* 检索：Top-k + 语义/关键词混合召回 + 业务过滤；
* 重排：轻量重排（交叉编码或规则）；
* 生成：带引文与拒答策略（低置信度→追问/拒答）。

**Redis Vector 结构（示例）**：

* 索引：`kb:vec:index`（schema: `vector`, `doc_id`, `chunk_id`, `tags`）
* 元数据：`kb:meta:{doc_id}:{chunk_id}` → `{text, url, title, ver}`

**摄取流水线**：

1. 清洗（去重/拆分）→ 2) 嵌入（批量）→ 3) 入索引 → 4) 验证抽检 → 5) 发布可用。

---

## 4. API 契约与服务编排

### 4.1 服务结构（apps/agent-service）

* 框架：Fastify/Nest（二选一，建议 Fastify + `@fastify/websocket`）。
* 主要路由：

  * `POST /v1/agent/chat`：对话（可流式 SSE/WebSocket）。
  * `POST /v1/agent/tools/:name`：工具直调（供内部 E2E 测试）。
  * `POST /v1/rag/ingest`：知识库摄取（需管理员权限）。
  * `GET  /v1/health`：健康检查。
* OpenAPI：`/docs` 提供 Swagger UI，契约版本化。

### 4.2 Chat 请求/响应（SSE 示例）

**Request**

```json
{
  "session_id": "abc",
  "user_id": "u_123",
  "query": "帮我评估这款泳衣的上新时机",
  "context": {"site": "sg", "locale": "zh-CN"},
  "tools": ["search_trend", "mysql_ro", "feishu_report"],
  "mode": "plan-act-reflect"
}
```

**SSE Events**：`plan` → `retrieve` → `tool_call`(n) → `final` → `metrics`。

**Final Payload（带引文）**

```json
{
  "answer": "……",
  "citations": [{"doc_id": "sku123", "chunk": 5}],
  "metrics": {"latency_ms": 1820, "tokens": 1200, "tool_succ": 0.9}
}
```

### 4.3 工具函数签名（TypeScript）

```ts
export const searchTrend = defineTool({
  name: "search_trend",
  desc: "查询站点/类目近30天搜索热度",
  input: z.object({ site: z.string(), keyword: z.string() }),
  output: z.object({ series: z.array(z.tuple([z.string(), z.number()])) }),
  run: async (input, ctx) => { /* … */ }
});
```

### 4.4 计划-执行-反思（策略）

* **Plan**：用 `system+developer` 提示产出子任务列表（含所需工具）。
* **Act**：按子任务执行，失败→指数退避重试；
* **Reflect**：根据失败模式库修正计划/提示（例如“检索为空→改写查询”）。

---

## 5. 前端（Node SSR）集成与流式渲染

* **调用层**：BFF（现有 Java 或 Node）→ 统一代理到 `agent-service`；
* **安全**：BFF 注入 `user_id`/`scope`，前端不得直连 Agent；
* **渲染**：SSE 流式拼接到 UI，显示阶段性状态（计划/检索/工具/最终）。

**前端事件协议**：

```ts
interface AgentEvent {
  type: 'plan'|'retrieve'|'tool'|'final'|'metrics'|'error';
  data: any;
}
```

**降级策略**：Agent 超时→回退纯 RAG 或 FAQ；工具异常→展示可操作建议。

---

## 6. 评测（Eval）与回归集

**指标**：

* 任务成功率（目标达成/判定规则）；
* 事实正确率（含引文匹配）；
* 工具调用成功率/冗余率；
* 令牌/成本、P95 延迟；
* 审核类（越权、敏感输出、注入触发率）。

**基准集（JSONL 示例）**：

```json
{"id":"ecom_001","query":"对比两款泳衣材质并给出上新建议","gold":"包含材质差异、库存/季节窗口、广告建议"}
{"id":"feishu_001","query":"生成本周SSR项目周报","gold":"包含进度%、异常、工时、下周计划"}
```

**自动评测脚本（要点）**：

* 模版：`tooling-scripts/eval.ts`；
* 评分：规则 + LLM-as-a-judge（带裁判提示/对齐标准）；
* 报表：存 MySQL `agent_eval_result`，Grafana 可视化。

---

## 7. 观测性（日志/Trace/成本）

**日志结构**：

```json
{ "ts": 1733731200, "trace_id":"t1", "route":"/v1/agent/chat", "user":"u_1", "latency":1820, "model":"gpt-X", "tokens_in":800, "tokens_out":400, "tool_calls":2, "cost_usd":0.012, "ok":true }
```

**Trace**：Span 粒度切分：plan/retrieve/tool(n)/final；OTLP 上报。

**仪表盘**：

* 概览：QPS、P50/P95、错误率、成本/千次；
* 质量：任务成功率、工具成功率、引文命中；
* 失败面板：超时/重试次数/常见错误码。

---

## 8. 安全与合规

* **身份**：BFF 注入用户身份与 `scope`；
* **授权**：工具分级授权（读/写/跨系统）；
* **CORS**：仅允许主站 `www.*` 与子域白名单（au./sg./…）；
* **注入**：提示隔离（system 不暴露）、RAG 内容净化；
* **越权**：工具参数白名单/正则过滤；
* **速率**：用户/IP/会话三级限流；
* **审计**：高风险工具全量留存请求/响应摘要。

---

## 9. 部署与运维

* **形态**：

  * 轻量：Vercel/CF Workers（仅推理/编排，不做大计算/DB）
  * 标准：Docker + K8s（有状态：Redis/MySQL 托管）
* **Nginx**：流式代理、超时与断点续传、蓝绿/金丝雀；
* **CI/CD**：分支保护、预览环境、自动回滚（失败阈值触发）；
* **备份**：Redis AOF、MySQL binlog；**演练**：故障日脚本。

---

## 10. 成本与性能优化打法

* 缓存：提示模板、嵌入、RAG 命中、工具结果（TTL/标签）；
* 上下文压缩：检索前摘要、结构化摘要、会话切片；
* 并行：可并发的工具统一并发执行；
* 模型混用：检索/草案用小模型，裁判/最终用大模型；
* Token 策略：模板去冗余、严格 JSON 模式、函数调用优先；
* P95 目标：< 2.5s（无重工具），< 5s（1\~2 次外部工具）。

---

## 11. 实战专题A：电商选品&上新 Agent

**能力图谱**：趋势检索（站点/类目）→ 库存与毛利 → 季节窗口 → 广告建议 → 上新决策。

**数据/工具**：

* `mysql_ro`: sku/库存/毛利表（只读视图）
* `search_trend`: 站点关键词趋势（内部/三方）
* `calc.run`: 促销方案/毛利敏感性分析

**Prompt（裁剪版）**：

```
You are an e-commerce merchandising agent. Goals: … Constraints: 引文必须来自工具结果；如关键数据缺失→列出需要补充的数据。
输出 JSON：{decision, reasons[], risks[], promo_plan}
```

**验收**：

* 用 20 条历史上新记录回放；
* 指标：命中实际“成功上新”的比例≥70%，错误建议≤10%。

---

## 12. 实战专题B：前端异常诊断 Agent

**能力**：聚合 APM/前端日志 → 归因 → 修复建议 → 生成 PR 描述。

**工具**：

* `http.get(APM)`：查询某 Trace/JS 错误 TopN；
* `rag`：团队最佳实践库（监控、SSR 指南、组件规范）；
* `git.diff`（可选，只读）

**输出**：`{root_causes[], hotspots[], fix_suggestions[], risk_level}`。

**评测**：对比人工排障结论的重叠度≥80%。

---

## 13. 实战专题C：飞书周/月报自动化 Agent

**流程**：

1. 每周五 16:00 拉取本周 Epic/Story 进度、工时、异常；
2. 生成周报（进度%、本周完成/未完成、风险与下周计划）；
3. 群内发送卡片消息 + 附带链接；
4. 异常项（未评审/超期/停滞）单独 @责任人。

**工具**：`feishu.members`, `feishu.issues`, `feishu.postMessage`。

**卡片字段**：项目、进度%、完成/未完成/进行中、异常清单、下周计划。

**风控**：仅机器人可见密钥；白名单群；消息频率限制。

---

## 14. 项目清单、Checklist 与SLA

**开发 Checklist**：

* [ ] OpenAPI3.1 契约与 Mock 服务；
* [ ] 工具函数 Schema 校验 + 幂等；
* [ ] RAG 引文/拒答策略落地；
* [ ] 观测（日志/Trace/成本）完备；
* [ ] 评测集 ≥ 50 条 + 自动化脚本；
* [ ] 权限与限流；
* [ ] 蓝绿发布脚本与回滚演练。

**SLA（建议）**：

* 可用性 ≥ 99.5%；
* 成本：\$X/千次内；
* 质量：任务成功率 ≥ 80%，工具成功率 ≥ 95%；
* 延迟：P95 < 2.5s（无重工具）。

---

### 附录A：示例代码片段

**Fastify 路由（SSE）**

```ts
app.post('/v1/agent/chat', async (req, reply) => {
  reply.raw.setHeader('Content-Type', 'text/event-stream');
  const ctx = buildCtx(req);
  const plan = await planner(req.body, ctx);
  sse(reply, { type: 'plan', data: plan });
  for (const step of plan.steps) {
    const res = await execStep(step, ctx);
    sse(reply, { type: 'tool', data: res });
  }
  const final = await summarize(ctx);
  sse(reply, { type: 'final', data: final });
  sse(reply, { type: 'metrics', data: ctx.metrics });
  reply.raw.end();
});
```

**工具定义（zod 校验）**

```ts
export function defineTool<TIn extends z.ZodTypeAny, TOut extends z.ZodTypeAny>(
  cfg: { name: string; input: TIn; output: TOut; run: (i: z.infer<TIn>, ctx: Ctx) => Promise<z.infer<TOut>> }
) { /* … */ }
```

**Redis Vector（创建索引）**

```bash
FT.CREATE kb:vec:index ON HASH PREFIX 1 kb:meta: SCHEMA \
  vector VECTOR HNSW 6 TYPE FLOAT32 DIM 1536 DISTANCE_METRIC COSINE \
  doc_id TAG chunk_id NUMERIC tags TAG
```

**飞书发送卡片（Webhook）**

```ts
await axios.post(process.env.LARK_BOT_WEBHOOK!, {
  msg_type: 'interactive',
  card: { config:{wide_screen_mode:true}, elements:[/* … */], header:{title:{content:'本周项目周报',tag:'plain_text'}} }
});
```

**Eval 裁判提示（片段）**

```
You are a strict judge. Compare `answer` with `gold`. Score 0-1 by: goal coverage, factuality (with citations), actionability.
Return JSON: {score: number, issues: string[]}
```

> 以上为“可直接落地”的大纲+SOP骨架。后续我可以：
>
> 1. 生成空仓库（Nx/Turbo）与最小可运行代码；
> 2. 按你的电商数据库结构映射 `mysql_ro` 视图；
> 3. 将周/月报 Agent 对接到你现有飞书群并给出 CRON 作业示例。
