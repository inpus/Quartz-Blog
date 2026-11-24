# json记录

## 结束报告['media_client.cpp:140'](D:\csevent\client\src\media\media_client.cpp)

```c++
    json report={
        {"mode",synthetic ? (local_receiver?"synthetic_local_webrtc":"synthetic_remote_webrtc") : local_receiver?"live_capture_local_webrtc":"live_webrtc"},
        {"dependencies",inspectGstreamer()},
        {"audio_packet_ms",20},
        {"client_cross_source_wait_ms",0},
        {"video_backend",cfg.media.video_backend},
        {"acceptance","not_run"}
    };
```

**`mode`**：运行模式，由 `synthetic` 和 `local_receiver` 决定：

- 合成媒体 + 本地接收：`synthetic_local_webrtc`
- 合成媒体 + 远端上传：`synthetic_remote_webrtc`
- 真实采集 + 本地接收：`live_capture_local_webrtc`
- 真实采集 + 远端上传：`live_webrtc`

**`dependencies`**：`inspectGstreamer()` 返回的 GStreamer 依赖和插件检查结果。

**`audio_packet_ms: 20`**：报告中记下的音频包时长为 20 毫秒。

**`client_cross_source_wait_ms: 0`**：客户端在处理 `cs2` 和 `oopz` 两路音频时，不额外等待另一条音轨的数据。

**`video_backend`**：本次配置的视频后端，例如 `native` 或 `gstreamer`。

**`acceptance: "not_run"`**：先标记验收尚未完成。
