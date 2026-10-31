# WebSocket 已连接但服务器收不到媒体

## 涉及文件

| 文件 | 作用 |
| --- | --- |
| `client/src/uploader.cpp` | Beast WebSocket 握手、消息类型和异步发送 |
| `server/libs/core/ws_handler.cpp` | 根据 WebSocket 消息类型分发接入数据 |
| `server/libs/core/ingest.cpp` | 处理项目二进制协议与媒体输入 |

## 背景与现象（S）

M0 使用 WebSocket 上传 JPEG 和 PCM。客户端能够连接服务器，也执行了发送，但服务端持续报告 `control not valid JSON`，媒体归档 `frames=0`。当时出现的 28 条 JSON 错误都来自这条错误路径。

## 要达到的目标（T）

让已经按项目协议组装的二进制消息，进入服务端的二进制解析入口，最终形成真实的帧和音频归档。

## 排查与根因（A）

项目消息由固定 18 字节协议头和 payload 构成。客户端虽然发送的是字节缓冲区，但 Beast 默认将出站消息设为 TEXT。

“缓冲区里放着二进制字节”与“WebSocket 消息类型为 BINARY”是两件事。服务端收到 TEXT 后，按文本控制消息解析，首先遇到的却是二进制协议头，自然无法得到合法 JSON。

```text
客户端：项目二进制协议 → Beast 默认 TEXT
服务器：收到 TEXT → 交给文本控制解析 → JSON 失败
结果：数据确实发出，入口分发却已经走错
```

## 如何解决

在握手成功后、开始发送队列之前，显式设置：

```cpp
ws->binary(true);
```

旧分帧协议中的 Control 也可以作为项目二进制帧的 payload 发送，因此应依据整体线协议确定 WebSocket 消息类型，不能只看到 payload 中存在 JSON 就判断为 TEXT。

修复时保留服务端校验，让类型错误继续暴露；没有通过放宽 JSON 解析或把任意字节强行转成字符串来绕过问题。

## 验证与结果（R）

2026-10-04 的修复后真机烟测中，服务器收到 15 张 JPEG 和 15 秒音频，归档结构与计数符合预期。连接成功、发送完成、服务端解析成功和归档成功由此形成了完整的检查链。

这篇记录的是 M0 旧媒体传输故障。后续 WebRTC 分支使用 H.264/Opus 传媒体，WebSocket 承担信令与控制；不能把旧方案的修复理解为恢复 WS 媒体上传。

## 可以复用的经验

排查网络协议先核对分层：TCP 是否连通、WebSocket 是否握手、消息 opcode 是否正确、业务头是否符合协议、payload 是否通过校验。上层的 JSON 错误可能由更早的消息类型错误引起。

## 依据

- `docs/日志记录/day3.md`：“T1.4 Beast WS 上行”及“踩坑记录”第 1 项。
- `client/src/uploader.cpp`：握手后的 `ws->binary(true)`。
