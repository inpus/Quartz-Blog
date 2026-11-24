# WebSocket 已连接但服务器收不到媒体

## 涉及文件

- `client/src/uploader.cpp`：握手、WebSocket 消息类型、项目协议封包和发送。
- `server/libs/core/ws_handler.cpp`：区分 Binary/Text 消息并分发。
- `server/libs/core/ingest.cpp`：接收项目协议中的媒体数据。

## 问题

客户端已经连接，也执行了发送，服务器却连续报 `control not valid JSON`，归档里 `frames=0`。这次记录中的 28 条 JSON 错误都来自同一个问题：我把项目二进制帧通过 WebSocket 的 TEXT 消息发出去了。

这篇对应 M0 的 JPEG/PCM 经 WS 上传路径；后来的 WebRTC 分支已经把媒体移到 H.264/Opus 通道，WS 继续承担信令和控制。

## 定位

我先对照客户端发出的字节与服务器接收入口。项目消息已经有 18 字节头，后面才是 JPEG、PCM 或 Control payload。客户端封包没问题，但服务端先根据 WebSocket 消息类型决定用哪条解析路径。

Beast 不会因为传入 `vector<uint8_t>` 就自动发送 Binary；缓冲区只提供字节，Text/Binary 由 WebSocket stream 的选项决定。默认 TEXT 导致服务器把协议头一起送进 JSON 解析。

## 修改前

按故障逻辑简化：

```cpp
// 握手成功，没有设置消息类型。
ws->async_handshake(host, path, [this](beast::error_code ec) {
    if (ec) return onFail("handshake", ec);
    connected = true;
    doWrite();
});

// bytes 已经包含项目头和 payload，但这不决定 WebSocket opcode。
ws->async_write(net::buffer(*bytes), completion);
```

服务端入口可以概括为下面的分支，具体解析细节省略：

```cpp
if (type == WebSocketMessageType::Binary) {
    // 解项目协议头，再按 Frame / AudioChunk / Control 分发。
} else if (type == WebSocketMessageType::Text) {
    // 将整条消息作为 JSON 控制消息解析。
}
```

因此 JPEG 没有机会走到图像解码，错误在更早的入口分发就发生了。

## 修改后

握手完成后，在起写泵之前明确设置 Binary：

```cpp
ws->async_handshake(host, path,
    [this, stream, gen](beast::error_code ec3) {
        if (stopping || gen != generation) return;
        if (ec3) return onFail("handshake", ec3);

        ws->next_layer().expires_never();
        ws->binary(true);
        connected = true;
        connected_flag = true;

        // 之后再封包、入队和启动写泵。
    });
```

Control 消息的封包也沿用项目二进制协议：

```cpp
const std::string text = control.dump();
const std::vector<uint8_t> payload(text.begin(), text.end());
enqueue(wire::encode(
    wire::MsgType::Control, clock::nowUs(), seq++, payload));
```

Control 的 payload 虽然是 UTF-8 JSON，外面仍包着项目协议头，所以整条 WebSocket 消息依然可以是 Binary。服务器先解头，知道类型为 Control 后，再解析内部 JSON。

## 为什么这样改

这里有两层类型：外层 WebSocket 的 Text/Binary，内层项目协议的 Frame/AudioChunk/Control。原来只设置了内层类型，却让外层保持默认值。

`binary(true)` 修正了最外层分发，服务器终于能先拆项目头，再处理对应 payload。我保留了服务端校验，没有让 JSON 解析去容忍任意前缀，否则错误协议还会继续向后传播。

设置顺序同样重要：先设置消息类型，再启动发送。对每次新建的连接都需要完成这项配置。

## 验证

2026-10-04 的 15 秒真机测试中，服务端收到 15 张 JPEG 和 15 秒音频，归档计数正常。验证检查到服务器落盘，而不只看客户端的写回调成功。

## 总结

这个问题让我区分了“传了什么字节”和“按什么协议类型发送”。网络已连接、async_write 已完成，只能说明传输走到了一部分；还要检查外层消息类型、业务头和服务端分发。JSON 报错有时不是 JSON 内容写错，而是根本不该走到 JSON 入口。

## 相关记录

- `docs/日志记录/day3.md`：Beast 上行、踩坑第 1 项和 INT-1 验证。
