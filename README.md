# dsh-sub2api-provider

> GitHub：<https://github.com/lldwb/dsh-sub2api-provider>（公开）。

把 **Sub2API** 注册为 [DeepSeek Harness (dsh)](https://github.com/deepseek-ai/deepseek-harness) 的第三方提供商，并支持 **Off / Low / High / Max 思考强度**。

本仓库是一个 dsh **bundle**（配置型插件）：它通过 profile 的 Cordis patch 层，把 `sub2api` 这条路由写进 dsh 随附的 `@deepseek-ai/dsh-llm-pi-ai` 适配器。

> 为什么是配置而不是适配器代码：在 dsh 里，Anthropic Messages 的协议实现由随附的 `dsh-llm-pi-ai` 提供，"第三方提供商"本身是**配置**。自己再写一个适配器会重复实现流式解析、工具调用、思考块与图片处理，且更容易出错。这里沿用官方适配器，只补上提供商定义。

## 安装

需要 dsh 桌面版，且 `plugin_manager` 可用。

```jsonc
// 让 agent 执行（或手动在 dsh 中安装该目录）：
plugin_manager { "action": "install_bundle", "target": "E:\\dsh-plugins\\dsh-sub2api-provider" }
```

安装后重启 dsh（替换已安装包需要重启才会加载新的 JS 模块代际；纯新增 bundle 通常走 HMR 生效）。

装好后打开 **设置 → 模型**，`Sub2API` 会作为第三方提供商出现，可直接编辑。

## 配置

在 provider 配置里改这几项：

| 字段 | 说明 |
|---|---|
| `baseURL` | 你的 Sub2API（或任何 Anthropic Messages 兼容）端点。**仓库里的默认值是我自己的网关，请改成你的。** |
| `apiKeyEnv` | 凭据引用名，默认 `SUB2API_API_KEY`。密钥不存在配置文件里，而是存在 dsh 凭据存储中；也可以在「设置 → 模型」里填入。 |
| `models` | 该路由对外提供的模型清单。每个条目支持 `contextWindow`、`maxTokens`、`input`、`reasoningEfforts`。 |

`api` 固定为 `anthropic-messages`。

## 思考强度

`reasoningEfforts` 的**键**是选择器公布的档位，**值**是发给网关的 effort 拼写：

| 选择 | 实际发送 |
|---|---|
| Off | `thinking: { type: "disabled" }` |
| Low / High / Max | `thinking: { type: "adaptive" }` + `output_config: { effort: "low" / "high" / "max" }` |

### 为什么走 effort 而不是 budget

`anthropic-messages` 默认走 budget 路径（`thinking.budget_tokens`）。实测本类网关**不执行** budget：

| 请求 `budget_tokens` | 实际思考量 |
|---|---|
| 128 | 10038 字符（约 2500 token） |
| 1024 | 20123 字符 |
| 20000 | 18769 字符 |

`budget_tokens: 128` 仍产出约 2500 token 思考，说明它只当开关用。而且在 budget 路径下，pi-ai 会把 `max`/`xhigh` **并入** `high`，两者发出的字节完全相同——那样 `Max` 会是个假档位。

因此这里用 `compat.forceAdaptiveThinking: true` 切到 effort 语义，档位名会原样进入 `output_config.effort`，`Max` 才真正独立。GLM 上实测到单调递增：low 9353 → high 10364 → max 12137 字符。

### `off` 需要按模型取舍

`glm-5.3-flash` 对 `thinking.type=disabled` 返回 **HTTP 400**（deepseek-flash 正常返回 200）。pi-ai 的规则是：

- **声明** `off` → 选中时发送 `disabled`
- **不声明** `off` → 完全不发送 thinking 参数

所以 `glm-5.3-flash` 故意不声明 `off`：它少一个 Off 档位，而不是多一个一点就报错的档位。若你的网关接受 `disabled`，给它补上 `off:` 即可。

## 模型发现

路由**不在** pi-ai 已安装目录里时，dsh 会向端点询问可用模型，供「设置 → 模型」采纳：

```
GET {baseURL}/v1/models?limit=1000     (x-api-key + anthropic-version)
```

`baseURL` 带不带结尾 `/v1` 都可以。发现结果只是候选，**不会自动写入配置**——需要你在界面上确认采纳。

## 已知限制

- **配置整块替换**：Cordis 的 id 定向 patch 会替换该行的整个 `config`，不做深合并。如果你自己的 `cordis.patch.yml` 里也有 `llm-pi-ai` 行，它层级更高，会**整体覆盖**本 bundle 的配置。此时请把 provider 写进你自己的那一行，或删掉自己的行交给本 bundle 管。
- 同理，本 bundle 会替换 `llm-pi-ai` 行的全部 provider。若你还要在这条适配器上跑别的提供商，请把它们一并写进同一份 `providers`。
- 提供商目录不校验远端是否真的接受某个模型，实际调用仍可能失败。

## 适配与验证记录（DSH 0.2.2-rc.1，2026-11-06）

**无配置改动**。0.2.2-rc.1 给 `llm-pi-ai` 的每个 provider 新增了两个**可选**字段：

| 字段 | 默认 | 作用 |
|---|---|---|
| `maxRequestImages` | `50` | 单次请求保留的图片数上限；超出的最老图片被换成占位文本（记录进会话，不回退） |
| `imageOffloadCountQuantum` | `10`（不超过上面的上限） | 每次计数溢出时批量卸载多少张 |

本 bundle **不写这两项**，因此走 0.2.2 的默认值 —— 这正是想要的行为：默认值由适配器维护，
不在这里另抄一份常量（抄了就会在官方调默认值时变成陈旧配置）。二者只影响多图长会话的降级路径，
文本/单图请求与之前完全一致。

### 验证

| 验证 | 结果 |
|---|---|
| 单测 | 无（纯配置 bundle，仓库不含测试） |
| 合成核对（devkit headless `sub2api-provider-config`，`inheritModel: false` 以免真实 profile 的 `llm-pi-ai` 覆盖层盖住本 bundle） | `dsh --profile devkit-sub2api-provider-config --dump-config` 里本 bundle 提供的 `llm-pi-ai` 行**完整保留**：`providers.sub2api` 的 `displayName`/`apiKeyEnv`/`api: anthropic-messages`/`baseURL`/`compat.forceAdaptiveThinking`/两个模型的 `contextWindow`/`input`/`reasoningEfforts`（含 glm 故意不声明 `off` 的形状）逐项在树中，未被 0.2.2 的新 schema 拒绝或裁剪 |
| 真实调用（devkit headless `sub2api-provider-regress`，模型 overlay 指向 sub2api 路由） | task「回复两个字：收到」→ **exit 0，返回「收到」**，即 0.2.2 下 sub2api 路由可正常派发 |
| 本 bundle 自定义端点的实际调用（`sub2api-provider-config` + 只把 `agent-default-model` 指向 sub2api 的临时 patch） | 失败于 `dsh: TRANSPORT: Connection error.` —— 是**网络层**错误：仓库里默认的 `baseURL`（`150.241.207.58:8080`）当前返回 **HTTP 502**，而真实 profile 覆盖用的 `150.241.207.60:8080` 可达。属环境/网关状态，不是 0.2.2 兼容问题；配置本身已被接受（否则会报配置校验错，而不是传输错误） |

> 默认值 `baseURL` 本来就是占位，请按「配置」一节改成你自己的端点。真实 desktop profile 里
> `llm-pi-ai` 行由 profile 自己的 patch 覆盖（含可达的 `.60` 端点与 4 个模型）—— 这是本插件
> 文档「已知限制」里写明的整块替换行为，符合预期。

## 适配与验证记录（DSH 0.2.2-rc.2，2026-11-12）

**无配置改动**。rc.1 → rc.2 之间，`llm-pi-ai` 的 **provider 配置 schema 零改动**：

- `packages/llm/llm-pi-ai/src/config.ts` 在 `dsh-v0.2.2-rc.1..dsh-v0.2.2-rc.2` 之间 **diff 为空**——
  本 bundle 用到的 `displayName` / `apiKeyEnv` / `api` / `baseURL` / `compat.forceAdaptiveThinking` /
  `models[].{id,name,contextWindow,input,reasoningEfforts}` 全部仍是合法字段。
- 同包 `src/index.ts` 只新增 `onThinkingOmitted` 日志回调，`src/provider.ts` 新增
  `google-generative-ai` 协议；两者都不影响 `api: anthropic-messages`。

### 验证（devkit 沙箱，真实 Loader）

| 验证 | 结果 |
|---|---|
| 干净 profile 合成（`inheritModel: false`，避免真实 profile 的 `llm-pi-ai` 覆盖层） | 合成树里 `- id: llm-pi-ai` 的 config **逐字段等于本 bundle 的 patch**（`providers.sub2api` 的 displayName/apiKeyEnv/api/baseURL/compat 与两个模型），即无更高层覆盖时本 bundle 是唯一事实来源 |
| 干净 profile 默认路由（`deepseek-official/deepseek-flash`）跑一个 task | **exit 0**，`pwsh echo` 正常返回——装上本 bundle 后 rc.2 Loader 正常启动 |
| 干净 profile + 临时 patch 把 `agent-default-model` 指向本 bundle 的 `sub2api/deepseek-flash` | 报 `dsh: TRANSPORT: Connection error.`——适配器**已注册**并真的发出了请求；仓库默认 `baseURL`（`150.241.207.58:8080`）在本机 TCP 不可达，属占位端点/网络环境问题 |
| 负对照：`agent-default-model` 指向不存在的 provider（临时 patch） | 报 `dsh: NO_ADAPTER: no adapter registered for provider "no-such-provider"`——错误类型与上一条**不同**，反证 `sub2api` 路由确实注册成功 |
| 真实 profile 覆盖层（继承真实 desktop 的 `llm-pi-ai`） | 有效 config 由 profile 层覆盖（`.60` 端点、4 个模型），符合「已知限制」的整块替换语义；该层下真实跑完一个 task（agent 运行在 `deepseek-flash` 上，**exit 0**） |

单测：无（纯配置 bundle，仓库不含测试）。

## 许可

MIT
