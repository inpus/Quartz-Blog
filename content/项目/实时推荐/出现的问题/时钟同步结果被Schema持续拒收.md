# 时钟同步结果被 Schema 持续拒收

## 涉及文件

- `client/src/uploader.cpp`：收到 pong 后计算偏移与 RTT，发送 clock_offset。
- `schemas/v1/control.json`：控制消息字段类型。
- `server/libs/core/ws_handler.cpp`：控制消息校验和会话时钟更新。

## 问题

客户端一直在发 ping，服务器也有 pong，但时钟偏移没有成功写入会话。服务端日志反复出现：

```text
control rejected: At /offset_us ... unexpected instance type
```

我对照实际上报的数据后发现，`offset_us`、`rtt_us` 是浮点结果，Schema 却要求整数。

## 修改前

Schema 中的旧字段为：

```json
"offset_us": { "type": "integer" },
"rtt_us": { "type": "integer", "minimum": 0 }
```

客户端计算逻辑如下，微秒转换没有把 double 变为整数：

```cpp
const double t_client = j.value("t_client", 0.0);
const double t_server = j.value("t_server", 0.0);
const double now = clock::nowSec();

const double rtt = now - t_client;
const double offset = t_server - t_client - rtt / 2;
enqueueControl({
    {"op", "clock_offset"},
    {"offset_us", offset * 1e6},
    {"rtt_us", rtt * 1e6}
});
```

实际数据可能是 `1305200417126.0002`。不是因为它数值大，而是因为它有非零小数部分，不满足 integer。消息被 Schema 拒收后，后面的时钟状态更新根本不会执行。

## 修改后

我保留计算精度，将契约改成接受 number：

```json
"offset_us": { "type": "number" },
"rtt_us": { "type": "number", "minimum": 0 }
```

客户端计算公式不变，服务端也继续校验，没有绕过 Schema。

`offset_us` 不能加 `minimum: 0`：它表示服务器时钟相对客户端时钟的偏移，正负都可能。`rtt_us` 是往返时长，需要非负。

## 为什么这样改

假设客户端在 `t_client` 发出请求，收到回复时为 `now`，往返时间为 `now - t_client`。NTP-lite 用半个 RTT 近似单程耗时，把服务端回包时间与客户端中点对齐：

```text
客户端中点时间 = t_client + RTT / 2
偏移 offset = t_server - 客户端中点时间
```

这个估计本来就是浮点计算，`× 1e6` 只换单位，不会保证得到整数。字段叫微秒，也不意味着只能表示整数微秒。

JSON Schema 的 number 接受整数和非整数数值，所以原有整数正例仍然合法。若将来需要统一为整型微秒，也应在两端约定舍入规则和范围，再整体修改契约；这次没有必要只为迁就错误类型而截断结果。

另外，两台机器的单调时钟原点不同，offset 数值很大并不直接等于网络延迟大。RTT 是在客户端自己的时钟上做差，不能和跨机 offset 混用。

## 验证

2026-10-04 真机核对后，服务器出现成功保存的记录：

```text
offset_us=1305200406969.9998 rtt_us=907.99
```

当轮 RTT 记录约 1–2ms。这里确认的是消息已通过校验并落到会话状态；NTP-lite 仍假设链路时延近似对称，不因此获得精密时钟同步的保证。

## 总结

我先检查具体拒收字段，再把 Schema 和真实 JSON 对上，最后只修正类型契约。时钟计算能得到结果，不代表服务器已经采用结果；以后检查这条链路，要一直看到接收端的校验和状态更新。测试也要覆盖实际会产生的小数，不能只拿手写整数样例验证。

## 相关记录

- `docs/日志记录/day4.md`：两端体检中 clock_offset 拒收问题。
