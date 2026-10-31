# 时钟同步结果被 Schema 持续拒收

## 涉及文件

| 文件 | 作用 |
| --- | --- |
| `client/src/uploader.cpp` | 计算并上报 NTP-lite offset/rtt |
| `schemas/v1/control.json` | clock_offset 控制消息的字段类型 |
| `server/libs/core/ws_handler.cpp` | 控制消息接收与处理 |

## 背景与现象（S）

客户端已经执行 ping/pong 并计算时钟偏移，但 2026-10-04 端到端体检发现，服务端一直报告：`control rejected: At /offset_us ... unexpected instance type`。时钟同步逻辑表面上在运行，偏移却从未成功入库。

## 要达到的目标（T）

让客户端实际发送的数值、公共 Schema 和服务器接收逻辑一致，并通过服务端证据确认时钟结果确实被采用。

## 排查与根因（A）

`offset_us`、`rtt_us` 的单位是微秒，但计算结果是 double。例如偏移可能为 `1305200417126.0002`。

旧 `control.json` 将这两个字段声明为 `integer`。JSON Schema 检查的是数值类型约束；字段名带 `_us` 并不要求它只能是整数。消息进入业务逻辑前，就被公共契约拒收。

定位过程是先看服务端具体拒收字段，再对照客户端序列化结果，最后核查 Schema，而没有先修改时钟公式。

## 如何解决

将两个字段改为 `number`，并保留 RTT 非负约束。当前片段为：

```json
"offset_us": { "type": "number" },
"rtt_us": { "type": "number", "minimum": 0 }
```

`number` 同时接受整数和小数，因此已有整数正例仍然合法。偏移本身允许正负值，不能直接套用 RTT 的非负约束。

这里选择修正契约，使其匹配实际计算语义；没有只为满足旧类型而无说明地截断精度，也没有绕过服务器校验。

## 验证与结果（R）

2026-10-04 真机核对中，服务端出现成功保存记录：

```text
offset_us=1305200406969.9998 rtt_us=907.99
```

当轮 RTT 记录约为 1–2ms，说明消息已通过 Schema 并进入会话时钟状态。这个结果证明上报链路恢复，不单独证明长期时钟估计精度达到某个指标。

## 可以复用的经验

公共契约必须用生产端真实序列化的数据检查。只用手写整数样例，很容易漏掉运行中产生的小数。对跨机链路，应把“计算完成”“发送成功”“校验通过”“状态落地”逐层核对。

## 依据

- `docs/日志记录/day4.md`：“体检揪出的问题”第 1 项。
- `schemas/v1/control.json` 与 `client/src/uploader.cpp` 的 clock_offset 上报。
