# Tool Call Display Naming 接手说明

## 背景

产品目标是：tool call 展示时，用户能看懂 AI 正在做什么、做完了什么，但不要暴露内部 tool 名、参数、实现细节，避免被逆向。

展示文案应该是用户语言，不是工程语言。例如不要直接显示 `ConnectorStatus`、`Mcp`、`DeviceInvoke`，而是显示 `Checking Slack`、`Using Notion MCP`、`Using Mac`。

## 已确认的产品原则

1. 文案要短、清楚、像系统状态。
2. 不展示未知 provider、device、internal tool id。
3. provider/device 是 input 字段，不是 tool name 的一部分。
4. 如果 provider/device 可识别，展示用户能理解的名字。
5. 有执行中和完成态，但只改主文案，不加副文案。
6. preview HTML 只是临时对比工具，不要提交进 PR。

## 基础映射

| Tool | Running 文案 | Done 文案 |
| --- | --- | --- |
| `Bash` | `Running a command` | `Ran a command` |
| `Memory` | `Checking memory` | `Checked memory` |
| `SpawnAgent` / `create_task` | `Starting a sub-agent task` | `Started a sub-agent task` |
| `AutomationApply` | `Updating a routine` | `Updated a routine` |
| `AutomationList` | `Checking routines` | `Checked routines` |
| `Write` | `Writing a file` | `Wrote a file` |
| `Edit` | `Editing a file` | `Edited a file` |

## Provider 动态文案

provider 来自 tool input，例如：

```ts
{ toolName: 'ConnectorStatus', input: { provider: 'notion' } }
```

应该显示：

| 状态 | 文案 |
| --- | --- |
| running | `Checking Notion` |
| done | `Checked Notion` |

```ts
{ toolName: 'Mcp', input: { provider: 'notion' } }
```

应该显示：

| 状态 | 文案 |
| --- | --- |
| running | `Using Notion MCP` |
| done | `Used Notion MCP` |

provider 必须走白名单，例如：

| provider id | 展示名 |
| --- | --- |
| `notion` | `Notion` |
| `slack` | `Slack` |
| `github` | `GitHub` |
| `google_maps` | `Google Maps` |

如果 provider 不在白名单里，不要展示原始 id，回退到泛化文案：

| Tool | Running fallback | Done fallback |
| --- | --- | --- |
| `ConnectorStatus` | `Checking a connected app` | `Checked a connected app` |
| `Mcp` | `Using MCP` | `Used MCP` |

## Device 动态文案

device 也来自 input 字段，不是 tool name。

```ts
{ toolName: 'DeviceInvoke', input: { deviceId: 'd_mac_1' } }
```

应该显示：

| 状态 | 文案 |
| --- | --- |
| running | `Using Mac` |
| done | `Used Mac` |

```ts
{ toolName: 'FileImportToDevice', input: { deviceId: 'd_iphone_1' } }
```

应该显示：

| 状态 | 文案 |
| --- | --- |
| running | `Sending a file to an iPhone` |
| done | `Sent a file to an iPhone` |

```ts
{ toolName: 'FileExportFromDevice', input: { deviceId: 'd_mac_1' } }
```

应该显示：

| 状态 | 文案 |
| --- | --- |
| running | `Getting a file from a Mac` |
| done | `Got a file from a Mac` |

device 字段优先级建议：

1. `deviceName` / `device` / `name`
2. `platform` / `clientPlatform` / `devicePlatform`
3. `deviceId` / `id` 中可识别的 mac、iphone、ipad、android 信息

未知 device 不要展示原始 id，回退到：

| Tool | Running fallback | Done fallback |
| --- | --- | --- |
| `DeviceInvoke` | `Using a connected device` | `Used a connected device` |
| `FileImportToDevice` | `Sending a file to a device` | `Sent a file to a device` |
| `FileExportFromDevice` | `Getting a file from a device` | `Got a file from a device` |

## 状态逻辑

当前需要两个主状态文案：

| 状态 | 含义 |
| --- | --- |
| `in_progress` / `running` | 正在执行 |
| `completed` | 执行完成 |

兼容旧状态：

| 输入状态 | 归一化状态 |
| --- | --- |
| `running` | `in_progress` |
| `failed` | `error` |

`error` / `aborted` 暂时可以继续用 running 文案或 fallback，不建议为了这次改动扩大范围。

## 合并 Tool Call 信息

产品希望把两个 assistant message 之间连续的 tool call 合并成一条摘要，而不是一个 tool call 一个 bubble。

目标示例：

```text
Explored 1 file, 1 search, ran 1 command
```

```text
Ran 3 commands
```

建议分类：

| Tool | 摘要分类 |
| --- | --- |
| `Bash` | command |
| `Read` / `ImageView` / 文件读取类 | file exploration |
| `WebSearch` | search |
| `WebFetch` | web page |
| 其他无法归类 tool | generic work，不暴露内部名 |

完成态摘要文案建议：

| 场景 | 文案 |
| --- | --- |
| 1 个 command | `Ran 1 command` |
| 3 个 command | `Ran 3 commands` |
| 混合类型 | `Explored 1 file, 1 search, ran 1 command` |
| 混合类型复数 | `Explored 2 files, 3 searches, ran 1 command` |

注意大小写：整句开头大写，中间动词小写。

## 推荐实现位置

核心文案建议放在公共库：

```text
libs/ts/tool-call-display/src/index.ts
```

这里负责：

1. 单个 tool call 的 display text、icon、status。
2. 一组 tool calls 的 summary text、icon、status。

BFF 里做聚合调用：

```text
services/backend-service/src/routes/messages-v2.ts
```

现在 `toTypedMessage` 会遍历 `msg.intermediates` 并逐条转成 `turn`。这里可以把连续 `messageType === 'tool_call'` 的 intermediates 收集起来，调用公共库生成一个 summary turn。

## 重要实现细节

tool_call 行的 metadata 里已有这些信息：

```ts
metadata: {
  toolName,
  toolCallId,
  args,
  status,
}
```

BFF 调 display lib 时，应该把：

```ts
metadata.args
```

传成：

```ts
input
```

否则历史消息里的 `Checking Notion`、`Using Mac` 这类动态文案拿不到 provider/device 字段。

## 不要做的事

1. 不要展示原始 tool name。
2. 不要展示未知 provider/device id。
3. 不要把 `tool-call-display-preview.html` 提交进 PR。
4. 不要加两行文案。状态必须体现在同一行主文案里。
5. 不要把所有 tool 都强行细分，未知的宁愿显示 `Working on it`。

## 验收标准

至少补这些测试：

1. 单个 tool call：
   - `WebSearch completed` -> `Searched the web`
   - `Bash completed` -> `Ran a command`

2. provider：
   - `ConnectorStatus + provider:notion` -> `Checking Notion` / `Checked Notion`
   - unknown provider -> `Checking a connected app`

3. device：
   - `DeviceInvoke + deviceId:d_mac_1` -> `Using Mac` / `Used Mac`
   - unknown device -> `Using a connected device`

4. merge summary：
   - 3 个 `Bash completed` -> `Ran 3 commands`
   - `Read + WebSearch + Bash completed` -> `Explored 1 file, 1 search, ran 1 command`

5. BFF：
   - 连续 tool_call intermediates 应该合并成一个 `turn`
   - 普通 text intermediate 仍然正常显示
   - tool_call 的 `metadata.args` 要传给 display lib 作为 `input`
