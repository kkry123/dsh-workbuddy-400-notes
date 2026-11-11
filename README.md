# WorkBuddy 国内版 400 / code 11128 排查记录

> 2026-11-11 实测 · Windows 11 x64 · DSH 内核 `0.2.1-rc.2` · 插件 `dsh-workbuddy-connect@0.7.9`

## TL;DR

国内版网关（`copilot.tencent.com`）会对请求里的**特定文字指纹**做渠道校验，命中就返回：

```
HTTP 400
{"code":11128,"msg":"Illegal API invocation from an unapproved channel",
 "displayMsg":{"zh":"请求被拦截，请重新发送"},
 "displayTips":{"zh":"这次调用没通过渠道校验，可能不是官方客户端发出的。"}}
```

DSH 的系统提示里默认带**四处**这样的指纹，所以要四处一起改。国际版（`www.workbuddy.ai`）不做这个校验。

## 环境

| 项 | 值 |
|---|---|
| 插件 | `dsh-workbuddy-connect@0.7.9` |
| DSH 内核 | `0.2.1-rc.2`（Desktop 内置版） |
| 系统 | Windows 11 x64 |
| 区域 | CN（`domain: www.workbuddy.cn`） |
| 模型 | `deepseek-v4.1-flash`，effort `high` |

账号已登录、模型目录 17 个、积分正常。`status` 与推理档位探针都能 `200`，**只有完整对话请求会挂**。

## 现象

会话里第一次请求就 400，重试无效（上游 `retry_policy.retry: false`）。

## 我的定位过程（以及第一次的错误结论）

### 第一步：二分请求体

同一凭证、同一套请求头、同一模型，**只改 body**，一次一个变量：

| 变体 | 结果 |
|---|---|
| 原始请求体（8 条消息，`roles: system,user,user,...`，41 个工具） | **400 / 11128** |
| 去掉 `tools` + `tool_choice` | **400 / 11128** |
| 去掉 `tool` 角色消息 | **400 / 11128** |
| **去掉 `system` 角色消息** | **200 OK** |

### 第二步：一个错误的推论

看到上表，我当时的结论是「网关拒绝含 `system` 角色的请求体」，修法是**把 system 提示折叠进第一条 user 消息**。改完确实通了。

**但这个结论是错的。**

去掉整条 system 消息，等于把 system 里的所有文字一起删了——其中就包含真正触发校验的那几段。我删的是「载体」，却以为删的是「原因」。这是典型的把相关当因果。

正确的说法是：**网关校验的是 system（和 assistant）消息里的特定文字指纹，不是 `system` 这个角色本身。**

## 真正的触发源（四处）

社区已有更精确的逐词消融实验，见文末「参考」。四处指纹如下：

### 1. 文件引用提示段

`@deepseek-ai/dsh-file-reference` 注入的那段：

```
Tokens prefixed with @ are paths the user explicitly referenced.
Relative paths resolve from the workspace root; ...
```

不是「出现 `@` 就拦」。逐词删词消融显示：删掉这段前半部分的**任何一个词**，请求立刻变 `200`；只换掉 `@` 字形（改写成 `(at)`）也变 `200`。也就是说网关匹配的是**前半段连续词组 + `@` 字形**这个组合。

### 2. 内核身份句

```
You are an AI agent powered by DeepSeek Harness.
```

（`DeepSeek` 与 `Harness` 之间有一个零宽空格 U+200B。）

只要这句**位于 system 消息开头**就被拦；放到句中、放到尾部、前面垫一行别的文字都 `200`。加普通空格、加 BOM、保留零宽字符都绕不过去。

### 3. personaPrefix 里的 coding agent 句

```
You are a coding agent powered by the {model} model.
```

网关认的是 `coding agent powered by` 这个**句式**，换模型名救不了。

### 4. Web GUI 段

```
You are interacting with the user through the DeepSeek Harness Web GUI ...
Current DeepSeek Harness Web GUI URL: "..."
```

这两段在 system 的**尾部**，所以只做「开头锚定」的修法够不到它们。

## 本机实测的指纹位置

把失败请求的 system 文本 dump 出来核对，四处**全在**：

| 指纹 | 偏移 | 是否存在 |
|---|---|---|
| 身份句 `You are an AI agent powered by …` | 0（开头） | ✅ |
| coding agent 句 | 50 | ✅ |
| 文件引用段 | 116 | ✅ |
| Web GUI 段 | 8606 | ✅ |

system 总长 9553 字符，全文只有 2 个 `@`，都在文件引用段里。

## 触发范围不止 system

社区实测（同一账号、同一请求头，只换 body）：

| 场景 | 结果 |
|---|---|
| `user` / `tool` 消息里放同样内容 | `200`（不扫描） |
| `assistant` 消息里放文件引用段（任意位置） | **400** |
| `assistant` 消息里放身份句（**仅开头**） | **400** |
| 多轮里只有第 2 条 assistant 带指纹 | **400**（逐条都查） |

结论：`system` 和 `assistant` 都要处理，`user` 与 `tool` 不用动。

这也说明**只改 system 不够**——历史会话里已经冻结的 assistant 消息仍可能触发。

## 修复方案

### 方案 A：独立 guard 插件（推荐）

在宿主进程包一层 `globalThis.fetch`，只对发往 `copilot.tencent.com` 的请求做语义等价改写：

| 改什么 | 改成什么 |
|---|---|
| 文件引用段里的 `@` | `(at)` |
| `powered by DeepSeek Harness.` | `powered by DeepSeek-Harness.` |
| `You are a coding agent powered by …` | `You are a coding agent running on …` |
| `… the DeepSeek Harness Web GUI …` | `… this Web GUI …` |

优点：不改插件本体，插件升级不受影响；有 `dryRun`、`verify-live.mjs`、`health-check.mjs` 可自检。

参考实现：[`curo22/dsh-wb400-guard`](https://github.com/curo22/dsh-wb400-guard)（MIT，v0.1.7）

### 方案 B：折叠 system 到 user

我最初的做法。能在**只涉及 system** 的场景下通过，因为指纹随 system 一起变成了 user 消息，而 user 不被扫描。

```ts
// 把 system 文本前置到第一条 user 消息，文字原样保留，只改角色
function foldCnSystemIntoUser(source: string): string { /* ... */ }
```

**缺陷**：`assistant` 消息不在处理范围内，历史会话里的旧 assistant 消息仍可触发 400；且指令优先级下降，长会话行为可能漂移。**不推荐长期使用。**

### 方案 C：社区修复版整包

[`LJY-YL/dsh-workbuddy-connect`](https://github.com/LJY-YL/dsh-workbuddy-connect)（`0.7.9-p1`，内置 `CHANNEL_PHRASE_REWRITES` 改写表）、[`Hourz26/dsh-workbuddy-connect-patched`](https://github.com/Hourz26/dsh-workbuddy-connect-patched)（补丁 fork）。

缺点：替换插件本体；两个仓库都注明上游 `corrinehu/dsh-workbuddy-connect` 已停止维护，且明确提示这是「猫鼠游戏」，不保证长期可用。

## 为什么插件自带的几处规避都没命中

- `normalizeDeveloperRole()` 把 `developer` 改写成 `system` —— 方向正好相反。它修的是另一个 `11128` 变体（`developer` 角色被拒），却制造出带指纹的 system。
- `conversationHeaders()`（`X-Conversation-*` 三件套）修的是 `conversation id missing` 那个变体，无关。
- `prepareInternationalChatBody()` 给国际版**补**一条开头 system —— 那边正确，若跑到 CN 通道就是致命的。

`prepareCnChatBody()` 当时只剥 `reasoning_effort: "off"`，没有任何指纹处理。

## 诊断方法（可复用）

插件吞掉了上游原始错误体，只留中文 `displayMsg`，所以第一次排查只能靠猜。有效做法是在 `chatStream()` 的非 2xx 分支里临时记录：

- 上游原始响应（前 1500 字符）
- 请求摘要：模型、消息角色序列、工具数量、`reasoning_effort`、`max_tokens`
- 并顺势重发几个变异体（去 tools / 去 system / 去 tool 消息）做对照

一轮就能把范围从「整个请求体」缩到「某类消息」。

## 已知限制

- 这是对服务端渠道校验的**客户端绕行**，上游随时可能再收紧。实现应能优雅降级，而非假设规则不变。
- 每新增一处 DSH 内核措辞，就可能新增一个指纹。`anchor` 模式只认已知锚点，内核改文案时需要回退到 `mode=all`（全量替换）。
- 指纹规则来自社区实测，我本人只验证了「四处指纹在本机 system 里全部存在」以及「去掉 system 后 200」，**没有**逐条独立复现每一处指纹的单独触发效果。
- `stream: false` 会被国内网关拒为 `11101`，与本问题的 `11128` 不是一回事。

## 参考

- [`curo22/dsh-wb400-guard`](https://github.com/curo22/dsh-wb400-guard) —— 四触发源的完整逐词消融实验表
- [`LJY-YL/dsh-workbuddy-connect`](https://github.com/LJY-YL/dsh-workbuddy-connect) —— `0.7.9-p1` 社区修复版
- [`Hourz26/dsh-workbuddy-connect-patched`](https://github.com/Hourz26/dsh-workbuddy-connect-patched) —— 补丁 fork + `PATCH-NOTES.md`
- [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) —— 宿主本体
