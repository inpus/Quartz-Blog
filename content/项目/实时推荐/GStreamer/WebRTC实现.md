# WebRTC 客户端主流程与媒体管线实现

> 本页说明客户端从 <code>main()</code>、<code>runMediaClient()</code> 到 Publisher 初始化和 GStreamer 媒体管线的实现。协商信令、ICE candidate、RTP 回调和服务端接收细节移到 [WebRTC信令与回调实现.md](WebRTC信令与回调实现.md)。面向使用者的建链流程见 [WebRTC.md](WebRTC.md)，启动和配置分别见 [启动.md](启动.md) 与 [配置.md](配置.md)。

## 按流程查源码

- 客户端入口、<code>started</code>、<code>begin()</code>、采集启动和收尾：见 [媒体客户端主流程](#client-lifecycle) 与 [WebSocket 控制回调](#websocket-control-hooks)。
- 创建 Publisher、初始化 GStreamer、建立三条媒体轨道：见 [Publisher 管线实现](#publisher-pipeline)。
- <code>negotiate()</code>、<code>offerReady()</code>、ICE gathering 与候选回调：见 [信令与协商回调](WebRTC信令与回调实现.md#signaling-callbacks)。
- 信令入队、<code>poll()</code> 路由、Offer 设置和 Answer/ICE 应用：见 [信令回调实现](WebRTC信令与回调实现.md#signal-queue-routing) 与 [远端信令处理](WebRTC信令与回调实现.md#remote-signaling)。
- 编码器、RTP 元数据、本地接收回调和服务端收流：见 [WebRTC信令与回调实现.md](WebRTC信令与回调实现.md)。

## 实现中的基础术语

- **element** 是 GStreamer 流水线里的处理组件。
- **pad** 是组件连接数据的接口。
- **transceiver** 是 WebRTC 中一条媒体轨道的发送或接收设置。
- **probe** 是在数据经过 pad 时执行的处理钩子。

<a id="client-lifecycle"></a>

## 媒体客户端主流程实现

> 代码范围：<code>client/src/media/media_client.cpp</code> 中的 <code>runMediaClient</code>。下文按运行时序说明入口、控制信令、WebRTC、采集、主循环和收尾。

### 真实远端模式的代码时序：从 main 到 ICE

下面只串联 <code>local_receiver=false</code> 的远端发送路径。每一步给出触发流程的关键代码和源码位置；各函数的内部细节在后面的对应小节展开。

#### 1. main 进入 runMediaClient

客户端选择 WebRTC 传输后，入口调用 <code>runMediaClient()</code>：

~~~cpp
return tl::client::media::runMediaClient(
    cfg,
    g_stop,
    probe || selftest,
    selftest || uplink,
    uplink && argc > 3 ? argv[3] : ""
);
~~~

远端普通模式下，<code>local_receiver</code> 为 <code>false</code>。代码来源：<code>client/src/main.cpp:50-67</code>。

#### 2. WebSocket 收到 started 后调用 begin

远端模式先启动控制连接；服务器回送的控制消息经 <code>post()</code> 放进 <code>incoming</code> 队列，主循环取出 <code>started</code> 并完成协议、传输方式和媒体轮次校验后，才调用 <code>begin(epoch)</code>：

~~~cpp
if (!controls.start(cfg.server_url, cfg.token, "", cfg.heartbeat_s)) {
    throw std::runtime_error("invalid control server URL");
}
~~~

主循环处理 <code>incoming</code> 中的消息时，<code>started</code> 通过校验后调用 <code>begin()</code>；会话启动后收到的 Answer/ICE 则交给 <code>publisher->remote()</code>：

~~~cpp
if (op == "started") {
    // 校验协议版本、WebRTC 传输方式和 media_epoch
    begin(message["media_epoch"].get<uint32_t>());
} else if (publisher) {
    publisher->remote(message);
}
~~~

代码来源：<code>client/src/media/media_client.cpp:164-185</code>；消息入队和主循环分派见 <code>:61-84</code>、<code>:170-210</code>。

#### 3. begin 创建 Publisher 并安装信令回调

<code>begin(epoch)</code> 保存本轮媒体 epoch，创建 <code>WebRtcPublisher</code> 并调用 <code>start()</code>。Publisher 后续产生的 Offer 和 ICE 消息通过这个回调交给控制发送链：

~~~cpp
auto begin = [&](uint32_t epoch) {
    current_epoch = epoch;
    publisher = std::make_unique<WebRtcPublisher>();
    publisher->start(
        cfg, epoch,
        [&, media_session = current_session_id](json message) {
            if (!local_receiver) {
                message["session_id"] = media_session;
                controls.sendControl(std::move(message));
            }
        },
        local_receiver, gpu_capture);
};
~~~

代码来源：<code>client/src/media/media_client.cpp:102-115</code>。

#### 4. WebRtcPublisher::start 注册回调并调用 negotiate

Publisher 创建管线后，先注册协商和 ICE 回调，再将管线切到 <code>PLAYING</code>，随后显式调用 <code>Impl::negotiate()</code>：

~~~cpp
g_signal_connect(self.peer, "on-negotiation-needed",
                 G_CALLBACK(Impl::negotiate), &self);
g_signal_connect(self.peer, "on-ice-candidate",
                 G_CALLBACK(Impl::ice), &self);
g_signal_connect(self.peer, "notify::ice-gathering-state",
                 G_CALLBACK(Impl::gathering), &self);

gst_element_set_state(self.pipeline, GST_STATE_PLAYING);
Impl::negotiate(self.peer, &self);
~~~

<code>on-negotiation-needed</code> 也能触发 <code>negotiate()</code>；显式调用和信号回调共用 <code>offered.exchange(true)</code>，因此只会有一次真正继续创建 Offer。代码来源：<code>client/src/media/webrtc_publisher.cpp:1009-1043</code>。

#### 5. negotiate：向本地 webrtcbin 请求创建 Offer

<code>negotiate()</code> 是 <code>webrtcbin</code> 的 <code>on-negotiation-needed</code> 回调；<code>start()</code> 在管线进入 <code>PLAYING</code> 后也会显式调用它。两种入口传入的 <code>peer</code> 都是本地发送端的 <code>webrtcbin</code>，<code>data</code> 指向它所属的 <code>Impl</code>。函数中的调用顺序如下：

~~~cpp
static void negotiate(GstElement *peer, gpointer data) {
    auto &self = *static_cast<Impl *>(data);
    if (self.offered.exchange(true))
        return;

    auto *promise =
        gst_promise_new_with_change_func(offerReady, &self, nullptr);
    g_signal_emit_by_name(peer, "create-offer", nullptr, promise);
}
~~~

1. <code>offered.exchange(true)</code> 把本轮“已发起协商”标记设为真。如果显式调用和 <code>on-negotiation-needed</code> 都到达这里，后到的一次直接返回，避免重复创建 Offer。
2. <code>gst_promise_new_with_change_func(...)</code> 创建异步结果对象，并把 <code>offerReady</code> 登记为状态变化回调；<code>&amp;self</code> 会作为 <code>data</code> 传回该回调。此时只是准备好接收结果，还没有请求创建 Offer。
3. <code>g_signal_emit_by_name(peer, "create-offer", nullptr, promise)</code> 才是请求的发出点。它通过 GObject 信号机制，在 <code>peer</code> 这个本地 <code>webrtcbin</code> 对象上发出名为 <code>create-offer</code> 的 action。GObject 将 action 分派给 <code>webrtcbin</code> 注册的处理逻辑；第二个参数 <code>nullptr</code> 表示使用默认创建选项，<code>promise</code> 是 GStreamer 写回异步结果的通道。这是进程内调用，不是网络发送，也不是项目里另有一个 C++ <code>createOffer()</code> 函数。
4. <code>webrtcbin</code> 根据本地 PeerConnection 已配置的媒体收发器等信息生成 Offer SDP。操作完成后，它把包含 <code>offer</code> 的回复交给 <code>promise</code>；Promise 状态变化后，已登记的 <code>offerReady()</code> 被调用，继续读取 Offer 并组装信令。

<code>offerReady()</code> 把 Offer SDP 序列化并组成 <code>webrtc_offer</code>，交给 <code>outgoing()</code> 放进 Publisher 的 <code>signals</code> 队列：

~~~cpp
self.outgoing({{"op", "webrtc_offer"},
               {"type", "offer"},
               {"sdp", text},
               {"tracks", mapping}});
~~~

代码来源：<code>client/src/media/webrtc_publisher.cpp:1009-1043</code>（回调注册、管线启动后的显式调用）、<code>:198-205</code>（<code>negotiate()</code>）、<code>:169-195</code>（<code>offerReady()</code>）。GStreamer 对 <code>create-offer</code> action 的定义见 <a href="https://gstreamer.freedesktop.org/documentation/webrtc/#create-offer">webrtcbin create-offer</a>；<code>g_signal_emit_by_name()</code> 在本进程内向目标对象发出该 action。

#### 6. poll 把 Offer 设置到本地 peer，并开始 ICE 候选收集

主循环调用 <code>publisher->stats()</code>，而 <code>stats()</code> 会推进 <code>poll()</code>。<code>poll()</code> 取出 <code>webrtc_offer</code> 后调用：

~~~cpp
if (message.value("op", "") == "webrtc_offer") {
    offered_tracks = message.at("tracks");
    setRemote(peer, "offer", message.at("sdp"), true);
}
~~~

<code>true</code> 让 <code>setRemote()</code> 选择 <code>set-local-description</code>，把本地 Offer 交给客户端的 <code>peer</code>。从这个 action 到 ICE gathering 的内部调用链如下。

GStreamer 在 <code>webrtcbin</code> 的类初始化时，把名为 <code>set-local-description</code> 的 GObject action 注册到内部函数 <code>gst_webrtc_bin_set_local_description()</code>。因此，项目的 <code>g_signal_emit_by_name(peer, "set-local-description", ...)</code> 会分派到这个 handler：

~~~c
gst_webrtc_bin_signals[SET_LOCAL_DESCRIPTION_SIGNAL] =
    g_signal_new_class_handler("set-local-description",
        G_TYPE_FROM_CLASS(klass), G_SIGNAL_RUN_LAST | G_SIGNAL_ACTION,
        G_CALLBACK(gst_webrtc_bin_set_local_description), NULL, NULL, NULL,
        G_TYPE_NONE, 2,
        GST_TYPE_WEBRTC_SESSION_DESCRIPTION, GST_TYPE_PROMISE);
~~~

action handler 校验并复制传入的 Offer，把来源标为 <code>SDP_LOCAL</code>，再把任务排入 <code>webrtcbin</code> 的 PeerConnection 任务队列。下面截取正常入队路径，省略参数无效或 <code>webrtcbin</code> 已关闭时的错误分支：

~~~c
sd = g_new0(struct set_description, 1);
sd->source = SDP_LOCAL;
sd->sdp = gst_webrtc_session_description_copy(local_sdp);
gst_webrtc_bin_enqueue_task(webrtc,
    (GstWebRTCBinFunc) _set_description_task,
    sd, (GDestroyNotify) _free_set_description_data, promise);
~~~

<code>g_signal_emit_by_name()</code> 触发 handler 后，正常路径下 handler 入队便返回；真正的 SDP 操作由排入队列的 <code>_set_description_task()</code> 在 <code>webrtcbin</code> 的 PeerConnection 上下文中执行。它处理 SDP、更新本地协商状态，并从 SDP 媒体行读取 ICE <code>ufrag</code>/<code>pwd</code> 设置到对应 ICE stream。设置完这些 ICE 参数后，任务遍历 ICE stream，调用：

~~~c
if (sd->source == SDP_LOCAL) {
    for (i = 0; i < webrtc->priv->ice_stream_map->len; i++) {
        IceStreamItem *item = &g_array_index(
            webrtc->priv->ice_stream_map, IceStreamItem, i);
        gst_webrtc_ice_gather_candidates(webrtc->priv->ice, item->stream);
    }
}
~~~

这行才是 GStreamer 内部实际请求 ICE 模块开始收集候选的位置。ICE 模块随后异步收集候选，<code>webrtcbin</code> 通过已注册的 <code>on-ice-candidate</code> 通知应用。<code>_set_description_task()</code> 完成本地描述处理并返回后，<code>webrtcbin</code> 的任务执行器再用 <code>gst_promise_reply()</code> 回复 Promise；因此 Promise 表示描述设置任务已处理并发起候选收集，不表示所有候选都已收集完。收集进度和完成状态由 <code>notify::ice-gathering-state</code> 另行报告。

GStreamer 上游源码：<a href="https://github.com/GStreamer/gstreamer/blob/main/subprojects/gst-plugins-bad/ext/webrtc/gstwebrtcbin.c#L9449-L9453">action 注册</a>、<a href="https://github.com/GStreamer/gstreamer/blob/main/subprojects/gst-plugins-bad/ext/webrtc/gstwebrtcbin.c#L7223-L7251">本地描述 action handler</a>、<a href="https://github.com/GStreamer/gstreamer/blob/main/subprojects/gst-plugins-bad/ext/webrtc/gstwebrtcbin.c#L1307-L1355">任务排队</a>、<a href="https://github.com/GStreamer/gstreamer/blob/main/subprojects/gst-plugins-bad/ext/webrtc/gstwebrtcbin.c#L6757-L7173">描述处理任务</a>、<a href="https://github.com/GStreamer/gstreamer/blob/main/subprojects/gst-plugins-bad/ext/webrtc/gstwebrtcbin.c#L7094-L7101">触发候选收集</a>、<a href="https://github.com/GStreamer/gstreamer/blob/main/subprojects/gst-plugins-bad/ext/webrtc/gstwebrtcbin.c#L1246-L1284">任务执行器回复 Promise</a>。

本地 ICE gathering 有两种应用层通知：<code>on-ice-candidate</code> 每产生一个候选就调用 <code>Impl::ice()</code>，将候选做成 <code>webrtc_ice</code> 放进信令队列；收集启动后，<code>ice-gathering-state</code> 通常从 <code>NEW</code> 变为 <code>GATHERING</code>，全部收集完成后变为 <code>COMPLETE</code>。属性变化会触发 <code>notify::ice-gathering-state</code>，从而调用 <code>Impl::gathering()</code>。这个回调只观察状态，不负责启动收集；真正的启动点是前面 GStreamer 内部调用的 <code>gst_webrtc_ice_gather_candidates()</code>。<code>Impl::gathering()</code> 读取当前状态，只有变为 <code>COMPLETE</code> 才放入 end-of-candidates 标记：

~~~cpp
static void gathering(GObject *object, GParamSpec *, gpointer data) {
    auto &self = *static_cast<Impl *>(data);
    GstWebRTCICEGatheringState state;
    g_object_get(object, "ice-gathering-state", &state, nullptr);
    if (state == GST_WEBRTC_ICE_GATHERING_STATE_COMPLETE)
        self.outgoing({{"op", "webrtc_ice"},
                       {"sdp_mline_index", 0},
                       {"candidate", ""},
                       {"end_of_candidates", true},
                       {"local_receiver", GST_ELEMENT(object) == self.receiver}});
}
~~~

这个标记表示本地候选收集结束，不表示 ICE 连通性检查已经完成。每个候选在收集过程中即可通过 <code>Impl::ice()</code> 逐条路由；<code>poll()</code> 设置 Offer 成功后发送 Offer，之后发送已入队的候选和结束标记，因此不需要等待候选全部收集完或服务器 Answer 才发送 Offer。

候选回调的核心代码是：

~~~cpp
static void ice(GstElement *peer, guint line, gchar *candidate,
                gpointer data) {
    auto &self = *static_cast<Impl *>(data);
    self.outgoing({{"op", "webrtc_ice"},
                   {"sdp_mline_index", line},
                   {"candidate", candidate ? candidate : ""},
                   {"end_of_candidates", !candidate || !*candidate},
                   {"local_receiver", peer == self.receiver}});
}
~~~

代码来源：<code>client/src/media/webrtc_publisher.cpp:616-632</code>（<code>poll()</code>）、<code>:559-605</code>（<code>setRemote()</code> 选择并执行描述设置）、<code>:148-156</code>（<code>Impl::ice()</code>）、<code>:157-166</code>（<code>Impl::gathering()</code>）、<code>:1017-1019</code>（发送端回调注册）、<code>:1187-1190</code>（<code>stats()</code> 推进 <code>poll()</code>）。

#### 7. 收到服务器 Answer 后设置远端描述

WebSocket 收到消息后，<code>received</code> 回调只把消息交给 <code>post()</code>；主循环随后调用 <code>publisher->remote(message)</code>。收到 <code>webrtc_answer</code> 时，Publisher 校验返回的轨道映射，再把 Answer 设置为本机 <code>peer</code> 的远端描述：

~~~cpp
if (message.value("op", "") == "webrtc_answer") {
    if (!message.contains("tracks") || message["tracks"] != self.offered_tracks)
        throw std::runtime_error("answer did not confirm the offered mapping");
    self.setRemote(self.peer, "answer", message.at("sdp"));
    self.remote_set = true;
    for (auto &candidate : self.candidates)
        self.iceRemote(self.peer, candidate);
}
~~~

这里省略的参数默认是 <code>local=false</code>，因此 action 是 <code>set-remote-description</code>。它把服务器 Answer SDP 应用为本机 <code>peer</code> 的远端描述，其中包含媒体行和 ICE 参数；如果 SDP 文本本身带有 candidate 属性，GStreamer 也会在解析这份描述时处理它们。但这段代码里另行通过信令到达的 <code>webrtc_ice</code> JSON，不是靠 <code>set-remote-description</code> 加入的。

这些独立的候选消息走 <code>iceRemote()</code>，由 <code>add-ice-candidate</code> action 加到对应的 <code>peer</code>：

~~~cpp
g_signal_emit_by_name(target, "add-ice-candidate", line, candidate.c_str());
~~~

因此，Answer 先成功应用后，代码把之前缓存的候选逐条交给 <code>iceRemote()</code>；Answer 之后才到达的候选则直接走同一函数。可以把两步理解为：<code>set-remote-description</code> 应用远端 SDP，<code>add-ice-candidate</code> 加入单独 trickle 过来的远端候选。代码来源：消息分派 <code>client/src/media/media_client.cpp:177-210</code>；Answer 与候选缓存/刷新 <code>client/src/media/webrtc_publisher.cpp:1089-1112</code>；候选加入 <code>:608-614</code>。

#### 8. WebRTC 报告 connected 后才启动采集源

主循环读取 Publisher 的连接状态，只有 <code>connected</code> 为真才调用 <code>start_sources()</code>：

~~~cpp
const auto connection = publisher->stats();
if (!sources_started && connection["connected"].get<bool>())
    start_sources();
~~~

代码来源：<code>client/src/media/media_client.cpp:216-220</code>；Publisher 从 <code>webrtcbin</code> 读取连接状态见 <code>client/src/media/webrtc_publisher.cpp:693-700</code>。

**时序要点：**创建 Offer 不等于应用 Offer；应用本地 Offer 才触发本地候选收集；Answer 通过 <code>set-remote-description</code> 应用到客户端 PeerConnection；ICE gathering、远端候选加入和连通性检查由 <code>webrtcbin</code>/ICE agent 推进，应用通过候选回调和连接状态读取来路由或启动采集。

### 1. 入口与运行模式

入口在 main.cpp:50–67（源码位置：<code>client/src/main.cpp#L50</code>）。main 根据命令行参数选择配置文件和运行模式，再调用 **runMediaClient(cfg, g_stop, local_receiver, synthetic, fixture_dir)**。

```cpp
return tl::client::media::runMediaClient(
    cfg,
    g_stop,
    probe || selftest,
    selftest || uplink,
    uplink && argc > 3 ? argv[3] : ""
);
```

| 运行方式 | local_receiver | synthetic | 媒体来源与路径 |
|---|---:|---:|---|
| 普通 WebRTC 运行 | false | false | 连接服务器，采集真实视频和音频 |
| --media-probe | true | false | 本地接收，采集真实视频和音频 |
| --media-selftest | true | true | 本地接收，发送合成媒体 |
| --media-uplink-test | false | true | 连接服务器，发送合成媒体或录制素材 |

带测试开关时，配置文件默认是 config/client.webrtc.yaml，也可通过 argv[2] 指定。uplink 测试可再通过 argv[3] 指定素材目录；该目录应包含 cs2.wav、oopz.wav 和 frame.bgra。普通模式使用配置中的媒体传输设置。

g_stop 是停止标志的引用。收到 Ctrl+C 或 SIGTERM 后，主循环会观察到取消状态并退出。函数最终返回的退出码由 main 直接返回给操作系统。

### 2. 运行前准备：校验、队列和对象

对应  media_client.cpp:30–60（源码位置：<code>client/src/media/media_client.cpp#L30</code>）。

函数开头先做有限时长和素材校验：本地接收或合成测试要求 run_seconds 在 1 到 300 秒之间；传入 fixture 时要求它属于远端合成 uplink 测试。音频文件和 BGRA 帧会在启动媒体流程前读入内存。

接着创建媒体队列：

- video 容量为 1，用于缓冲视频帧。
- cs2 与 oopz 是两路音频包队列，容量来自 cfg.media.max_audio_packets。

控制消息走另一条队列：incoming 存 WebSocket 回调交给主循环处理的 JSON。post 用互斥锁保护队列；队列上限是 128 条，满时会将队尾替换成 error 消息。主循环每轮把一批消息从共享队列取到本地，再逐条处理。

消息回调通过 post 安全入队：

```cpp
auto post=[&](json message) {
    std::lock_guard lock(incoming_mutex);
    if (incoming.size()<128) incoming.push_back(std::move(message));
    else incoming.back()={{"op","error"},{"reason","client signaling queue overflow"}};
};
```

这里有两类队列，不要混为一谈：视频/音频队列承载媒体数据，incoming 承载控制信令。

<a id="websocket-control-hooks"></a>

### 3. WebSocket 控制回调

<code>WsUploader</code> 提供 WebSocket 控制通道；<code>setControlHooks()</code> 把连接建立、服务器消息、断线和停止消息的处理交给 <code>media_client</code>。这些回调让网络线程只传递控制事件，媒体会话状态仍由主循环统一修改。

#### 四个回调的签名

源码中的 <code>WsUploader::ControlHooks</code> 定义了四个函数对象（<code>client/src/uploader.hpp:22-27</code>）：

| 回调签名 | 参数和返回值 | 何时调用、负责什么 |
|---|---|---|
| <code>nlohmann::json start_message(const std::string&amp; session)</code> | <code>session</code> 是 WebSocket 握手成功后生成的会话 ID；返回要发给服务器的 JSON。 | 建连成功后调用，构造 <code>start</code> 请求。 |
| <code>void received(const nlohmann::json&amp; message)</code> | <code>message</code> 是服务器发来的 JSON；无返回值。 | 收到控制消息时调用。当前注册为 <code>post</code>，只把消息放入主循环队列。 |
| <code>void disconnected()</code> | 无参数、无返回值。 | 异步连接失败或断开时通知主循环清理当前媒体会话。 |
| <code>nlohmann::json stop_message()</code> | 无参数；返回停止会话时要发出的 JSON。 | <code>WsUploader::stop()</code> 收尾时调用；正常媒体收尾时 JSON 已填入最终统计信息。 |

对应的类型定义是：

```cpp
struct ControlHooks {
    std::function<nlohmann::json(const std::string&)> start_message;
    std::function<void(const nlohmann::json&)> received;
    std::function<void()> disconnected;
    std::function<nlohmann::json()> stop_message;
};
```

<code>setControlHooks(ControlHooks hooks)</code> 必须在 <code>WsUploader::start()</code> 前调用；它保存这组回调并将 uploader 设为控制通道模式。对这个模式，<code>WsUploader::start(url, token, session_id, heartbeat_s, err)</code> 负责启动异步连接；其中 <code>url</code>、<code>token</code>、初始 <code>session_id</code> 和心跳间隔是连接参数，<code>err</code> 用于返回启动参数错误。连接完成后的实际会话 ID 由 uploader 新生成，再传给 <code>start_message</code>。

#### 注册回调并准备主循环队列

回调在 <code>media_client.cpp:61-84</code> 注册。下面保留完整的队列和注册部分，便于对照四个位置：

```cpp
auto post = [&](json message) {
    std::lock_guard lock(incoming_mutex);
    if (incoming.size() < 128)
        incoming.push_back(std::move(message));
    else
        incoming.back() = {{"op", "error"},
                           {"reason", "client signaling queue overflow"}};
};

controls.setControlHooks({
    [&](const std::string& session) {
        post({{"op", "client_session"}, {"session_id", session}});
        return json{
            {"op", "start"},
            {"session_id", session},
            {"request_id", "start-1"},
            {"protocol_version", 2},
            {"media_transport", "webrtc"},
            {"capabilities", {
                {"video_codecs", {"H264"}},
                {"audio_codecs", {"opus"}},
                {"capture_metadata", {"tacticlens-rtp-v1"}}
            }},
            {"audio", {
                {"capture_mode", "process"},
                {"asr_input_mode", "separate"},
                {"tracks", {"cs2", "oopz"}}
            }}
        };
    },
    post,
    [&] { post({{"op", "disconnected"}}); },
    [&] { return stop_message; }
});
```

#### 1. `start_message(session)`：发起媒体会话

WebSocket 握手成功后，客户端的 <code>WsUploader</code> 调用 <code>makeSessionId()</code> 生成 <code>session_id</code>，再把这个值作为 <code>session</code> 参数传给 <code>start_message(session)</code>（<code>client/src/uploader.cpp:223-229</code>）。ID 的格式是 <code>YYYYMMDD-HHMMSS-xxxx</code>：本地日期时间加 4 位小写字母或数字随机后缀，实现在 <code>client/src/session_id.cpp</code>。

调用位置的关键两行是：

```cpp
session_id = makeSessionId();
enqueueControl(hooks.start_message(session_id));
```

这里的 <code>makeSessionId()</code> 不是只给 <code>client_session</code> 生成一个临时值。它生成的是客户端提供给服务器的会话 ID；回调把同一个值分成两路使用：

1. <code>post({"op":"client_session", ...})</code> 将它作为本地事件放入 <code>incoming</code>，主循环据此更新 <code>current_session_id</code>。这条 <code>client_session</code> 只在客户端内部流转，不发给服务器。
2. 回调返回的 <code>start</code> JSON 也带着相同的 <code>session_id</code>，随后由 <code>enqueueControl()</code> 编码并排队发送给服务器。

这里的 <code>client_session</code> 事件和保存的 <code>session_id</code> 不是两种 ID，而是同一个字符串在不同组件里的副本：

| 保存位置 | 用途 |
|---|---|
| <code>WsUploader::Impl::session_id</code> | WebSocket 发送层使用：给控制帧补会话字段、过滤带旧 ID 的消息，并给停止消息加 ID。 |
| <code>media_client::current_session_id</code> | 媒体主循环保存当前会话；创建 Publisher 时复制进信令回调，用于给 Offer、ICE 等消息加上对应 ID。 |
| <code>incoming</code> 中的 <code>client_session</code> | 一条本地队列通知：媒体主循环取出它，把其中的 ID 保存到 <code>current_session_id</code>；处理后这条通知就从队列移除。它不是额外的会话存储，也不会发给服务器。 |
| 服务端的 <code>s.id</code>、<code>s.media->session_id</code> | 从客户端的 <code>start.session_id</code> 保存而来，用于服务端关联会话和校验后续控制消息。 |

具体顺序是：WebSocket 回调调用 <code>post()</code>，把 <code>{"op":"client_session","session_id":sid}</code> 放进 <code>incoming</code>；媒体主循环取出这条 JSON 后执行 <code>current_session_id = sid</code>，然后该队列项就结束了。被保存下来的字符串仍留在 <code>current_session_id</code> 中，供后续 Publisher 信令回调使用。它既不是临时 ID，也不是等服务器再分配正式 ID；<code>WsUploader</code> 同时保留自己的副本，供 WebSocket 发送层使用。

服务器不会另生成一个替代它的 <code>session_id</code>。服务器收到 <code>start</code> 后，取出这个 ID 注册并保存到会话上下文，<code>started</code> 再把相同 ID 回传；响应里的 <code>media_epoch</code> 是另一个字段，用来标记媒体轮次，不是服务器生成的 session ID。服务端代码见 <code>server/libs/core/ws_handler.cpp:41-59</code>。

```cpp
const auto sid = j.value("session_id", "");
// 校验并注册 sid 后：
s.media->session_id = sid;
s.id = sid;
conn->send(json({{"op", "started"}, {"session_id", sid},
                 {"media_epoch", s.media->epoch}}).dump());
```

请求字段的作用：

- <code>op: start</code>：告诉服务端这是新媒体会话的启动请求。
- <code>session_id</code>：客户端生成并提交的会话 ID；服务端用它注册会话，并要求后续控制消息携带同一 ID。
- <code>request_id</code>：标识这次启动请求；服务端在 <code>started</code> 响应中回显它。
- <code>protocol_version: 2</code>、<code>media_transport: webrtc</code>：声明协议版本和选用的媒体传输方式；客户端收到 <code>started</code> 后也会检查服务端确认值。
- <code>capabilities</code>：声明 H.264、Opus 和 TacticLens RTP 采集元数据扩展能力。
- <code>audio</code>：声明进程采集模式、分轨输入，以及 <code>cs2</code>、<code>oopz</code> 两条音轨。

#### `enqueueControl()`：把返回的 JSON 放进 WebSocket 控制发送队列

<code>start_message()</code> 只负责构造并返回 JSON，不直接访问 WebSocket。紧接着的 <code>enqueueControl(hooks.start_message(session_id))</code> 才将它交给 uploader 的发送路径：

```cpp
void enqueueControl(const json& j) {
    auto control = j;
    if (control_only && !control.contains("session_id"))
        control["session_id"] = session_id;
    const std::string text = control.dump();
    const std::vector<uint8_t> payload(text.begin(), text.end());
    enqueue(wire::encode(wire::MsgType::Control, clock::nowUs(), seq++, payload));
    controls_sent.fetch_add(1, std::memory_order_relaxed);
}
```

它接收回调返回的 JSON，必要时补上当前 <code>session_id</code>，序列化为文本，再包装成带类型、时间戳和序号的 Control 帧，放入异步写队列。启动请求本身已经含有 <code>session_id</code>，所以这里不会另造或替换会话 ID；服务端收到并处理这条 Control 帧后，才建立会话上下文并回 <code>started</code>。实现见 <code>client/src/uploader.cpp:78-84</code>。

服务器回 <code>started</code> 时会回显同一个 <code>session_id</code>，并附上 <code>media_epoch</code>。主循环先用本地 <code>client_session</code> 事件更新 <code>current_session_id</code>，再校验 <code>started</code> 的协议版本、传输方式和 <code>media_epoch</code>，最后调用 <code>begin(epoch)</code> 创建 Publisher。当前客户端代码没有再比较响应中的 <code>started.session_id</code>。后续 Offer、ICE 和停止消息继续携带客户端生成的 <code>session_id</code>；<code>media_epoch</code> 则单独随 WebRTC 信令使用。

> [!NOTE]
>
> 当前 <code>uploader.cpp</code> 在每次 WebSocket 握手成功时都会调用 <code>makeSessionId()</code>，因此自动重连成功后会产生一个新 ID。<code>session_id.hpp</code> 的注释写着“重连沿用同一 ID”，与这段实际调用不一致；服务端当前也会拒绝已注册过的 ID。这里按实际执行的客户端和服务端代码说明。

#### 2. `received(message)`：把服务器消息交给主循环

在 <code>uploader.cpp:133-140</code>，<code>WsUploader</code> 解析 WebSocket 文本 JSON，忽略心跳 <code>pong</code>，其余控制消息调用 <code>received(message)</code>；收到 <code>started</code> 时还会取消协商等待定时器。本项目将 <code>received</code> 直接设为 <code>post</code>。因为回调签名接收 <code>const json&amp;</code>、<code>post</code> 按值接收，消息会先复制到 <code>post</code> 参数，再移动进受互斥锁保护的 <code>incoming</code> 队列；Publisher 状态留给主循环处理。

触发回调的代码是：

```cpp
if (control_only && j.value("op", "") != "pong") {
    if (hooks.received) hooks.received(j);
}
```

主循环每轮取出一批消息，再按 <code>op</code> 分派：<code>started</code> 启动会话；<code>webrtc_answer</code> 和 <code>webrtc_ice</code> 交给 <code>publisher->remote(message)</code>；<code>disconnected</code> 清理会话；<code>error</code> 或 <code>media_error</code> 则报告服务端拒绝。队列最多存 128 条，满时把队尾替换为队列溢出错误。

#### 3. `disconnected()`：通知主循环回收会话

这个回调没有参数。<code>WsUploader</code> 的异步失败处理调用它时，本项目只执行 <code>post({{"op", "disconnected"}})</code>。随后主循环停止窗口监视、视频和音频采集，停止并释放 Publisher，清空旧会话媒体队列，再等待新的 WebSocket 会话；重连次数由主循环计数（<code>media_client.cpp:187-206</code>）。

调用位置：

```cpp
if (control_only && hooks.disconnected) hooks.disconnected();
```

#### 4. `stop_message()`：生成最终停止消息

这个回调也没有参数，返回当前保存的 <code>stop_message</code>。媒体客户端在收尾阶段先停止 Publisher 并读取最终统计，再把 <code>media_epoch</code>、最后帧 ID 和两路音频最终 sample index 写入该 JSON（<code>media_client.cpp:267-283</code>）。之后 <code>controls.stop()</code> 调用 <code>stop_message()</code>；<code>WsUploader::stop()</code> 再补上当前 <code>session_id</code>、排队发送并等待控制队列 flush（<code>uploader.cpp:275-292</code>）。

停止时 uploader 的调用顺序如下：

```cpp
auto stop_message = control_only && hooks.stop_message
    ? hooks.stop_message() : json{{"op", "stop"}};
stop_message["session_id"] = session_id;
enqueueControl(stop_message);
doWrite();
```

#### 控制消息的整体流向

1. <code>media_client</code> 在启动 uploader 前调用 <code>setControlHooks()</code> 注册四个回调，再调用 <code>controls.start(...)</code> 启动连接。
2. 握手成功后，<code>start_message(session)</code> 生成并返回 <code>start</code> JSON；uploader 发送它，服务器随后返回 <code>started</code>。
3. 服务器的 Answer、ICE 和状态消息经 <code>received(message)</code> → <code>post()</code> → <code>incoming</code> 队列 → 主循环分派。
4. Publisher 产生 Offer/ICE 时，<code>begin()</code> 安装的信令回调补上本次 <code>session_id</code> 并调用 <code>controls.sendControl(message)</code>。<code>sendControl()</code> 将发送任务投递到 uploader 的 I/O 线程；若消息带有 <code>session_id</code>，会拒绝与当前会话不匹配的消息；连接有效且未进入停止状态时才放入 WebSocket 控制发送队列（<code>uploader.cpp:306-310</code>）。
5. 连接异常走 <code>disconnected()</code>；正常退出走 <code>stop_message()</code> 和 <code>controls.stop()</code>。

### 4. 建立 WebRTC 会话

WebRTC 创建逻辑在  media_client.cpp:97–110（源码位置：<code>client/src/media/media_client.cpp#L97</code>）。

#### 本地接收模式

local_receiver 为 true 时，函数直接调用 begin(1)，不启动 WebSocket 控制连接。这个分支用于本地媒体传输测试。

本地模式和远端模式从这里分开：

```cpp
if (local_receiver) {
    begin(1);
    synthetic_origin=clock::nowUs();
}
else if (!controls.start(cfg.server_url,cfg.token,"",cfg.heartbeat_s))
    throw std::runtime_error("invalid control server URL");
```

#### 远端服务器模式

远端模式调用 controls.start(...)，启动异步 WebSocket 连接。参数包含服务器地址、token、空的初始 session 和心跳间隔。返回 true 表示参数有效且连接流程已启动；连接是否成功要等后台握手完成。

WebSocket 握手完成后，WsUploader 生成 session ID 并调用 start_message。服务器确认后，主循环收到 op=started，检查：

- protocol_version 是 2；
- selected_transport 是 webrtc；
- media_epoch 是非零无符号整数；
- 当前尚未创建 publisher。

校验通过后才调用 begin(media_epoch)。不符合条件就报错，避免在协商未确认时发送媒体。

服务器确认消息的处理分支：

```cpp
else if (op == "started") {
    if (publisher || message.value("protocol_version",0) != 2 ||
        message.value("selected_transport","")!="webrtc" ||
        !message.contains("media_epoch") ||
        !message["media_epoch"].is_number_unsigned() ||
        message["media_epoch"].get<uint32_t>()==0)
        throw std::runtime_error("server did not confirm protocol v2 / WebRTC / a new media epoch; no media sent");
    begin(message["media_epoch"].get<uint32_t>());
}
```

#### begin(epoch) 做什么

begin 保存媒体轮次，创建 WebRtcPublisher 并调用 publisher->start(...)。publisher 产生 WebRTC 信令时，远端模式会给消息加上创建 publisher 时捕获的 session ID，再通过 controls.sendControl(...) 发给服务器；本地接收模式不发送远端控制消息。

begin 的实现把 publisher 创建和信令回调放在一起：

```cpp
auto begin=[&](uint32_t epoch) {
    current_epoch=epoch;
    publisher=std::make_unique<WebRtcPublisher>();
    publisher->start(
        cfg,
        epoch,
        [&,media_session=current_session_id](json message) {
            if (!local_receiver) {
                message["session_id"]=media_session;
                controls.sendControl(std::move(message));
            }
        },
        local_receiver);
};
```

session ID 由 uploader 在 WebSocket 连接成功后生成。对应实现见  uploader.cpp:223–240（源码位置：<code>client/src/uploader.cpp#L223</code>）。

### 5. WebRTC 连通后启动媒体源

启动回调在  media_client.cpp:112–132（源码位置：<code>client/src/media/media_client.cpp#L112</code>）。主循环发现 publisher->stats() 中的 connected 变为 true 后，才调用 start_sources()。

触发条件就在主循环中：

```cpp
const auto connection=publisher->stats();
if (!sources_started && connection["connected"].get<bool>()) {
    start_sources();
    synthetic_origin=clock::nowUs();
}
```

- 真实采集模式：根据 cfg.media.video_backend 选择原生 WGC 或 GStreamer 视频源，再启动 IndependentAudioSource，分别向 cs2、oopz 队列写入音频包。启动失败会抛出异常。

视频后端的选择分支：

```cpp
if (cfg.media.video_backend=="native") {
    std::string error;
    if (!native_video.startRaw(video,cfg.media.target_fps,cfg.window_process,&error))
        throw std::runtime_error(error);
} else {
    gst_video.start(video,cfg);
}
```

- 合成模式：跳过视频和音频采集，但仍将 sources_started 设为 true，允许后续合成媒体逻辑运行。

音频源状态回调只在远端模式下发送 media_status，其中包含媒体轮次、session、音轨名称和状态详情。

halt_sources() 的职责是先请求音频和两种视频源停止，再调用 stop() 等待它们结束。它会在断线处理和函数退出时使用。

### 6. 主循环：处理信令并分发媒体

主循环在  media_client.cpp:157–252（源码位置：<code>client/src/media/media_client.cpp#L157</code>）。循环会在收到取消标志、达到配置的运行时长或发生错误时结束；每轮先处理信令消息，再根据 publisher 状态推进媒体流程。

publisher 已连接后：

- 合成模式按时间生成音频包和视频帧。没有 fixture 时生成测试音调和测试图像；有 fixture 时从已加载的素材中取数据。
- 真实采集模式从视频队列取帧，并分别从 cs2、oopz 队列各取至多一个音频包，交给 publisher。这样单路音频不会独占一次主循环分发。

主循环每轮从队列取媒体并交给 publisher：

```cpp
RawVideoFrame image;
if (video.tryPop(image)) publisher->push(std::move(image));

AudioPacket packet;
if (cs2.tryPop(packet)) publisher->push(std::move(packet));
if (oopz.tryPop(packet)) publisher->push(std::move(packet));
```

主循环还会处理服务器拒绝、publisher 失败、信令协商超时和 ICE/DTLS 连接超时，并定期记录媒体状态。

### 7. 断线、退出与报告

断线事件由主循环处理，而不是由 WebSocket 回调直接停止媒体。处理过程会停止采集源和 publisher，清空媒体队列，重置计数与协商计时，然后等待新的 WebSocket 会话。客户端最多容忍 3 次控制连接重连；超过预算就报错退出。

断线分支负责停止旧会话并清空旧队列：

```cpp
} else if (op=="disconnected") {
    halt_sources();
    if (publisher) {
        publisher->stop();
        publisher.reset();
    }
    sources_started=false; sample=0; frame=0;
    RawVideoFrame discarded_frame;
    while (video.tryPop(discarded_frame)) {}
    AudioPacket discarded_audio;
    while (cs2.tryPop(discarded_audio)) {}
    while (oopz.tryPop(discarded_audio)) {}
}
```

退出时的关键收尾代码：

```cpp
halt_sources();
if (publisher) {
    RawVideoFrame image;
    if (video.tryPop(image)) publisher->push(std::move(image));
    AudioPacket packet;
    while (cs2.tryPop(packet)) publisher->push(std::move(packet));
    while (oopz.tryPop(packet)) publisher->push(std::move(packet));
    publisher->stop();
    report["media"]=publisher->stats();
}
controls.stop();
```

1. 停止采集源。
2. 尽可能把队列中剩余的帧和音频包交给 publisher，再停止 publisher。
3. 整理媒体统计，并生成最终 stop 消息。
4. 停止 WebSocket uploader。
5. 汇总队列丢弃数、重连数和验收结果；如果配置了 report_path，写入 JSON 报告。

返回码为 0 表示本次运行通过相应检查，2 表示失败。报告里的 acceptance 对本地接收模式可以表示传输和媒体身份检查通过；它不代表真实游戏画面或语音识别已经验收。代码也明确将 live_game_and_speech_acceptance 和 audio_sample_continuity_acceptance 设为 false。

### 阅读这段代码时抓住三件事

1. **控制面与媒体面分开：** WebSocket 传 JSON 信令，WebRTC 传音视频。
2. **publisher 启动与采集启动分开：** 先协商并连通，再启动采集。
3. **运行模式是两个开关的组合：** local_receiver 决定本地还是远端控制路径，synthetic 决定真实采集还是合成媒体。


<a id="publisher-pipeline"></a>

## WebRTC Publisher 初始化与 GStreamer 管线

> 入口：<code>client/src/media/webrtc_publisher.cpp:673</code>

### 初始化 GStreamer

> 源码位置：<code>client/src/media/gst_runtime.cpp:25</code>

```c++
    static std::once_flag once;
    std::call_once(once,[] {
        GError* error=nullptr;
        if (!gst_init_check(nullptr,nullptr,&error)) {
            const std::string message=error?error->message:"unknown GStreamer initialization error";
            if (error) g_error_free(error);
            throw std::runtime_error(message);
        }
        guint major,minor,micro,nano; gst_version(&major,&minor,&micro,&nano);
        if (major!=1 || minor<24) throw std::runtime_error("GStreamer >= 1.24 required");
    });
    static GlibLoop loop;
```

#### 保证线程初始化一次

```c++
static std::once_flag once;
```

- `std::once_flag`：一个供 `std::call_once` 使用的标记。它记录“对应的一次性操作是否已经成功完成”。标记的内部状态由 C++ 库管理，你不用自己读写它。
- `static`：让这个标记在函数多次调用之间一直是同一个对象。

它要和下一行一起看：

```c++
std::call_once(once, [] {
    // 初始化 GStreamer
});
```

`std::call_once` 的意思是：“用 `once` 这个标记保护这段代码，让它只成功执行一次。”

假设两个线程同时调用 `initializeGstreamer()`：

1. 两个线程都使用同一个 `once`。
2. 一个线程获得执行机会，运行初始化代码。
3. 另一个线程等待。
4. 初始化成功后，等待的线程继续执行，但不会再运行初始化代码。
5. 以后再调用，也会跳过初始化代码。

这里 `static` 很关键。要是去掉它，每次进入函数都会创建一个新的 `once_flag`；新标记还没记录过操作，初始化代码就可能每次都执行。

可以简单记成：

> [!NOTE]
>
> `static` 让标记一直存在；`once_flag` 让多个线程共享初始化状态；`call_once` 根据这个状态决定要不要执行代码。
>
> **它和 `std::call_once(once, ...)` 配合起来**，用于保证初始化代码只成功执行一次，并且能安全处理多个线程同时调用的情况。`static` 让每次调用函数时都复用同一个 `once` 标记。单独声明这个标记还不会执行初始化或限制调用

如果初始化代码抛出了异常，这次不算成功，后续调用 `call_once` 时还会尝试执行。

#### gstreamer初始化

```c++
gst_init_check(nullptr, nullptr, &error)
```

它负责初始化 GStreamer 库的运行环境，包括设置内部路径、注册内置元素，以及加载标准插件。它返回成功或失败；失败详情写到 `error`，所以程序可以自己报告错误并决定怎么处理。`gst_init_check` 就是适合这种情况的初始化接口。[GStreamer API 文档](https://gstreamer.freedesktop.org/documentation/gstreamer/gst.html)

两个 `nullptr` 表示不把程序的命令行参数交给 GStreamer 处理。`&error` 则让函数能把错误信息写回 `error`。

#### `gst_version(...)` 和版本判断

```
guint major, minor, micro, nano;
gst_version(&major, &minor, &micro, &nano);
```

`gst_version` 查询程序运行时链接到的 GStreamer 版本，并把版本号写进这四个变量。例如运行时版本是 `1.24.5`，大致就是 `major=1`、`minor=24`、`micro=5`。官方文档区分了运行时版本查询和编译时版本宏。[GStreamer 初始化指南](https://gstreamer.freedesktop.org/documentation/application-development/basics/init.html)

```
if (major != 1 || minor < 24)
    throw std::runtime_error("GStreamer >= 1.24 required");
```

这行要求主版本是 `1`，次版本至少是 `24`。因此 `1.24`、`1.25` 能通过；`1.23` 和 `2.0` 会抛出异常。注意判断里没有检查 `micro` 和 `nano`。

这段代码只负责初始化和版本检查；创建具体媒体 pipeline 是后面的 `gst_parse_launch(...)`。

### 注册后续要用的采集标识 webrtc_publisher.cpp:673（源码位置：<code>client/src/media/webrtc_publisher.cpp</code>）

```c++
registerCaptureExtension();
```

```c++
gst_element_register(
    nullptr,
    "rtphdrexttacticlenscapture",
    GST_RANK_MARGINAL,
    capture_extension_get_type());
```

`gst_element_register(...)` 是**向当前程序进程里的 GStreamer 注册一个本地自定义元素**，不联系服务器，也不向远端注册。

1. **`nullptr`：注册到当前程序，而不是某个动态插件**

   GStreamer 把它作为静态 element 注册到当前进程。程序启动后完成这次注册，后续 GStreamer 才能创建这个扩展对象。

2. **`"rtphdrexttacticlenscapture"`：这个 factory 的名字**

   它是 GStreamer 内部创建该类对象用的名字。**你当前代码没有直接用这个字符串去创建对象**；你的代码是按 URI 查找，下面第 4 点会连起来说明。

3. **`GST_RANK_MARGINAL`：factory 的优先级**

   它告诉 GStreamer 这个 factory 的 rank。`MARGINAL` 是较低的 rank；它不决定 RTP 扩展 ID，也不决定扩展 URI。 [GStreamer `gst_element_register` 文档](https://gstreamer.freedesktop.org/documentation/gstreamer/gstelement.html#gst_element_register)

4. **`capture_extension_get_type()`：要注册的 C++ 类类型**

   它返回 `CaptureExtension` 的 GType，告诉 GStreamer：“用这个类来创建刚才登记的 factory 对象。”

#### capture_extension_get_type()

> 在  capture_rtp_extension.cpp (line 11)（源码位置：<code>client/src/media/capture_rtp_extension.cpp:11</code>） 有这行：
>
> ```c++
> G_DEFINE_TYPE(CaptureExtension, capture_extension,
>               GST_TYPE_RTP_HEADER_EXTENSION)
> ```
>
> - `capture_extension`
>
> 宏的第二个参数是 **`capture_extension`**，他是函数名前缀；会找对应的 "**函数名前缀**_class_init"等函数名
>
> 它会生成 `capture_extension_get_type()`，以及一个内部转接函数。转接函数大致如下：
>
> 宏用它拼出 `capture_extension_class_intern_init` 和 `capture_extension_class_init` 这两个名字。生成的桥接函数概念上相当于：
>
> ```
> static void capture_extension_class_intern_init(gpointer klass)
> {
>     capture_extension_class_init(
>         (CaptureExtensionClass *)klass);
> }
> ```
>
> 然后，生成的 `capture_extension_get_type()` 把这个转接函数作为 `GClassInitFunc`，交给 `g_type_register_static_simple(...)`。GObject 类型系统因此记住了：这个类型的类第一次初始化时，要调用哪个回调。这个过程不是按名字查找，而是保存并调用函数指针。[GLib 的 G_DEFINE_TYPE 展开示例](https://raw.githubusercontent.com/GNOME/glib/main/gobject/gtype.h)

#### capture_extension_class_init() capture_rtp_extension.cpp:16（源码位置：<code>client/src/media/capture_rtp_extension.cpp</code>）

##### 设置扩展类型的信息

```C++
auto* klass = GST_RTP_HEADER_EXTENSION_CLASS(type);
```

把 `CaptureExtensionClass*` 转成父类 `GstRTPHeaderExtensionClass*`。之后用 `klass` 设置 RTP 扩展类的 URI 和读写回调。

```c++
gst_element_class_set_static_metadata(
    GST_ELEMENT_CLASS(type),
    "TacticLens capture identity",
    "Network/Extension/RTPHeader",
    "16-byte source identity and capture timestamp",
    "TacticLens");
```

| 参数                                              | 含义                                                         |
| ------------------------------------------------- | ------------------------------------------------------------ |
| `GST_ELEMENT_CLASS(type)`                         | 把扩展的类结构转换为父类 `GstElementClass`，因为这些说明信息属于 GStreamer 元素类。 |
| `"TacticLens capture identity"`                   | 元素的可读名称。                                             |
| `"Network/Extension/RTPHeader"`                   | 元素的分类路径：它属于网络相关的 RTP 头扩展。                |
| `"16-byte source identity and capture timestamp"` | 元素的用途说明。                                             |
| `"TacticLens"`                                    | 作者或维护者名称。                                           |

把这个元素的名称、分类、说明和作者写入 GStreamer 元数据，供 GStreamer 识别和展示。它不会把这些文字放进 RTP 包。

```c++
gst_rtp_header_extension_class_set_uri(
    klass, tl::media::kCaptureExtensionUri);
```

> [!IMPORTANT]
>
> 把这个 RTP 扩展类和 URI 关联起来。之后代码可以按这个 URI 创建扩展实例。

##### 告诉 GStreamer 扩展该如何工作

下面几行是在**登记回调函数**。它们此刻只是设置函数指针；GStreamer 之后需要相关信息时才会调用这些回调，类似于为类实现一组虚函数。[GStreamer RTP header extension 文档](https://gstreamer.freedesktop.org/documentation/rtplib/gstrtphdrext.html)

```c++
klass->get_supported_flags = [](GstRTPHeaderExtension*) {
    return GST_RTP_HEADER_EXTENSION_ONE_BYTE;
};
```

表示此扩展支持 RTP 的 one-byte 扩展格式。GStreamer 文档说明，这种格式每个扩展数据最多能容纳 16 字节；这份实现正好使用 16 字节。

```c++
klass->get_max_size = [](GstRTPHeaderExtension*, const GstBuffer*) -> gsize {
    return 16;
};
```

告诉 GStreamer：需要为这个扩展预留最多 16 字节空间。

```c++
klass->write = []( ... guint8* bytes, gsize size) -> gssize {
    if (size < 16) return -1;
    std::memset(bytes, 0, 16);
    return 16;
};
```

这是发送方向的写回调：

1. 如果预留空间不足 16 字节，就返回 `-1`，表示失败。
2. 空间足够时先写入 16 个零字节。
3. 返回 `16`，告诉 GStreamer 实际写了 16 字节。

这只是先占好扩展数据的位置。注释说明，实际的采集帧 ID 和时间要等编码后的对应关系确定后，才由后面的 RTP 输出探针写入。

```c++
klass->read = []( ... const guint8* bytes, gsize size, ...) -> gboolean {
    return tl::media::decodeCaptureMetadata({bytes, size}).has_value();
};
```

这是读取方向的回调：尝试把收到的扩展数据解码成采集元信息。能解码就返回 `true`，否则返回 `false`。它做的是**有效性判断**，不是这里直接处理视频画面。

##### 这些回调何时工作？

```
类型初始化时：
设置 URI、元数据和回调函数

发送 RTP 时：
GStreamer 调用 get_supported_flags、get_max_size 和 write
之后 RTP 输出探针再把零字节替换成真实采集信息

读取 RTP 时：
GStreamer 的扩展读取路径调用 read 来验证数据
```

> [!NOTE]
>
> 也就是说，`capture_extension_class_init()` 是在**配置扩展类型**；真正处理某个 RTP 包的是之后被调用的回调和 RTP 探针。

#### 流程

> 1. C++ 先调用 `capture_extension_get_type()`。首次调用时，它把这个类型登记到 GObject 类型系统，并返回一个 `GType ID`。登记内容包括父类型 `GST_TYPE_RTP_HEADER_EXTENSION`、类初始化回调等。
>
> 2. `gst_element_register()` 收到这个 `GType ID`，把工厂名 `rtphdrexttacticlenscapture` 和该类型关联起来。
>
> 3. GStreamer 在注册过程中调用 `g_type_class_ref(GType ID)` 用它取得对应的**类结构**。如果类还没初始化，GObject 会调用 `G_DEFINE_TYPE` 生成的转接函数`capture_extension_class_intern_init()`转接函数，再把**类结构指针**传给 `capture_extension_class_init()`[GStreamer 注册代码](https://github.com/GStreamer/gstreamer/blob/main/subprojects/gstreamer/gst/gstelementfactory.c) [GObject 类型系统说明](https://docs.gtk.org/gobject/concepts.html)
>
>    > [!NOTE]
>    >
>    >  **GObject 还没有完成这个类型的类结构初始化**；首次调用时，分配和初始化会在 `g_type_class_ref()` 这条流程里完成。以后再调用，它会返回已初始化的类结构，不会再执行一次 `class_init`
>
> 4. 于是 `capture_extension_class_init()` 执行，设置 URI、元数据以及 `write`、`read` 等回调。
>
> 整体可以记成：
>
> ```
> G_DEFINE_TYPE 在编译时生成 get_type 和初始化回调
>     ↓
> registerCaptureExtension 调用 get_type，拿到 GType ID
>     ↓
> gst_element_register 注册工厂，并取得该类型的类结构
>     ↓
> GObject 自动调用 capture_extension_class_init
> ```

#### **注册后的实际调用关系是：**

```
gst_element_register(...)
    ↓ 登记 factory 和 CaptureExtension 类型
gst_rtp_header_extension_create_from_uri(kCaptureExtensionUri)
    ↓ 按类上设置的 URI 找到并创建扩展对象
gst_rtp_header_extension_set_id(extension, 7)
    ↓ 给 RTP 扩展分配数字 ID 7
add-extension
    ↓ 把扩展对象挂到 payloader 上
```

> [!NOTE]
>
> 注册的作用是让 GStreamer 知道：**自定义 URI `urn:tacticlens:rtp-hdrext:capture-v1` 对应哪段 RTP 扩展实现**。`gst_element_register(..., nullptr, ...)` 会把这个实现作为当前程序内的静态元素注册到 GStreamer 的本地工厂里。[GStreamer API 文档](https://gstreamer.freedesktop.org/documentation/gstreamer/gstelement.html)
>
> 后面的代码会按这个 URI 找到并创建扩展对象，把它加到 RTP payloader 上，并分配扩展编号 `7`，见  webrtc_publisher.cpp (line 786)（源码位置：<code>client/src/media/webrtc_publisher.cpp:786</code>）。随后，代码把采集身份信息写进 RTP 包的扩展区，让媒体包能关联到对应的采集帧或音频片段。
>
> 如果注册失败，按 URI 创建扩展对象会失败，代码会报 `capture extension factory unavailable`。所以这里注册的是**本地实现的可发现性**；采集信息之后才会随 RTP 包发送出去。

> [!TIPS]
>
> factory 名是 GStreamer 本地创建对象用的；URI 用来找到对应的扩展类型；数字 `7` 才是 RTP 包和 SDP 里使用的扩展 ID。

### 注册自定义 buffer 元数据类型

```
static const gchar *tags[] = {nullptr};
```

这是传给元数据注册函数的空标签列表；`nullptr` 表示列表结束。

```c++
static const auto *identity_meta =
    gst_meta_register_custom(kIdentityMeta, tags, nullptr, nullptr, nullptr);
(void)identity_meta;
```

1. **`kIdentityMeta`：给这种 Meta 起名字**

   你的值是 `"TacticLensCaptureMeta"`。后面给 buffer 添加或读取这种 Meta 时，都用这个名字：

   ```
   gst_buffer_add_custom_meta(buffer, kIdentityMeta); // 添加
   gst_buffer_get_custom_meta(buffer, kIdentityMeta); // 查找
   ```

   可以把它理解成：注册一个叫 `TacticLensCaptureMeta` 的标签类型，后续代码才能按这个名字找到它。

2. **`tags`：给 Meta 标类别**

   ```
   static const gchar *tags[] = {nullptr};
   ```

   这是一个空的、以 `nullptr` 结束的类别列表，表示你没有给这个 Meta 标类别。它不是采集编号、时间，也不会被写进 RTP 包。GStreamer 的部分处理流程会参考 Meta 的类别来决定如何处理或传递 Meta。 [GStreamer `gst_meta_register_custom` 文档](https://gstreamer.freedesktop.org/documentation/gstreamer/gstmeta.html#gst_meta_register_custom)

3. **第三个 `nullptr`：没有自定义“变换/复制处理函数”**

   这个位置本来可以传一个函数，让 GStreamer 在 buffer 被复制或变换时，按你指定的方式处理 Meta。你传 `nullptr`，表示使用 GStreamer 的默认规则：没有自定义 transform 回调时，默认处理函数在 copy 变换中复制整个 Meta 及其 GstStructure，并忽略复制区域参数；其他类型的变换不复制该 Meta。 [GStreamer `gst_meta_register_custom` 文档](https://gstreamer.freedesktop.org/documentation/gstreamer/gstmeta.html#gst_meta_register_custom)

4. **第四个 `nullptr`：没有给上述函数准备额外数据**

   如果第三个参数传了自定义函数，这里可以传一份额外数据，供那个函数使用。现在没有自定义函数，所以这个参数也用不上。

5. **第五个 `nullptr`：没有额外数据的清理函数**

   如果第四个参数传了需要释放的额外数据，这里可以传负责清理它的函数。现在没有额外数据需要清理，所以传 `nullptr`。

> [!TIP]
>
> **注册之后，真正给某个 buffer 写采集信息的是后面的代码**：它先用 `gst_buffer_add_custom_meta(buffer, kIdentityMeta)` 添加 Meta，再往它的 `GstStructure` 里写 `source_unit_id` 和 `capture_start_us`。解码回调则用 `gst_buffer_get_custom_meta(buffer, kIdentityMeta)` 查找并读取。 [GStreamer `GstBuffer` 文档](https://gstreamer.freedesktop.org/documentation/gstreamer/gstbuffer.html)

> [!NOTE]
>
> [GStreamer 自定义元数据 API](https://gstreamer.freedesktop.org/documentation/gstreamer/gstmeta.html)
>
> - 本项目用途：
>
> 本地接收并解包后，把采集信息附到 buffer： webrtc_publisher.cpp(line 435)（源码位置：<code>client/src/media/webrtc_publisher.cpp:435</code>）
>
> 解码时读取并核验： webrtc_publisher.cpp(line 378)（源码位置：<code>client/src/media/webrtc_publisher.cpp:378</code>）
>
> - 目的
>
> **在本地回环验收模式下**，就是看媒体包带着标签经过 WebRTC 接收、解包和解码后，解码输出还能否对应到原来的编号和采集时间。
>
> 加 `GstMeta` 是为了让这份信息继续跟着解包、解码后的 `GstBuffer` 走。这样解码回调处理到一帧画面或一段音频时，也能知道它对应哪个采集单元，并确认它确实走到了输出端。单看收到 RTP 包里的编号和时间，不能确认对应内容已经成功解码。

`identity_meta` 接收注册函数的返回值，但这里不再使用它；`(void)identity_meta;` 是为了消除“变量未使用”的编译警告。因为变量是函数内的 `static`，初始化表达式只执行一次。

### **`gst_element_register`和`gst_meta_register_custom`的场景用处**

> [!NOTE]
>
> ```
> 采集帧/音频的 ID 和时间
>   → 写入 RTP 扩展，随媒体包传输
>   → 接收端读取
>   → 放到本机 GstBuffer 元数据上
>   → 解码后核对来源
> ```
>
> - `gst_element_register` 让 GStreamer 能按 URI 创建自定义 RTP 扩展，再把它加到 RTP payloader 上。发送端把采集 ID、采集时间写进 RTP 包，见  扩展注册与添加 (line 786)（源码位置：<code>client/src/media/webrtc_publisher.cpp:786</code>） 和  RTP 元数据注入 (line 199)（源码位置：<code>client/src/media/webrtc_publisher.cpp:199</code>）。
> - `gst_meta_register_custom` 注册本机 `GstBuffer` 元数据类型。启用本地回环接收器时，代码把从 RTP 包读出的采集身份放到解包后的 buffer 上，解码回调再读出来核对，见  添加 buffer 元数据 (line 435)（源码位置：<code>client/src/media/webrtc_publisher.cpp:435</code>） 和  解码时读取 (line 378)（源码位置：<code>client/src/media/webrtc_publisher.cpp:378</code>）。
>
> 这样可以统计媒体有没有到达并成功解码、来源 ID 和时间是否匹配，以及哪些包缺少身份信息。`GstMeta` 是本机 GStreamer buffer 内部的数据；跨网络传递的是 RTP 扩展。

### 生成SSRC

```
for (auto &track : self.tracks)
    track.ssrc = g_random_int();
```

`self.tracks` 有三条轨道：视频、CS2 音频和 OOPZ 音频。这里给每条轨道随机分配一个 32 位 SSRC，用来标识对应的 RTP 流；它不是帧编号，也没有在这里做碰撞检查。

### 管线视频配置 webrtc_publisher.cpp:685（源码位置：<code>client/src/media/webrtc_publisher.cpp</code>）

#### 1. WebRTC 连接元素

```
webrtcbin name=peer bundle-policy=max-bundle latency=50
```

- `webrtcbin`：WebRTC 媒体连接元素，接收后面的 RTP 数据并通过协商好的连接发送。

  > [!NOTE]
  >
  > `webrtcbin` 可以理解成 **GStreamer 里的 WebRTC 通信组件**：
  >
  > - `WebRTC`：实时音视频通信技术。
  > - `bin`：GStreamer 里一种容器型元素；`webrtcbin` 把 WebRTC 连接需要的功能组织在一起。
  >
  > 它负责 WebRTC 连接和媒体传输相关的工作，例如 SDP 协商、ICE 连接，以及收发 RTP 媒体数据。[GStreamer webrtcbin 文档](https://gstreamer.freedesktop.org/documentation/webrtc/)

- `name=peer`：给元素起名；代码后面用 `peer.sink_0` 把视频分支接进来。

- `bundle-policy=max-bundle`：让视频、音频等媒体轨尽量共用一条 WebRTC 传输通道。[GStreamer webrtcbin 文档](https://gstreamer.freedesktop.org/documentation/webrtc/)

  > [!NOTE]
  >
> 在这段代码中，视频、CS2 音频和 OOPZ 音频仍然是三条独立的轨道，分别接到 `peer.sink_0`、`peer.sink_1`、`peer.sink_2`。`max-bundle` 让它们尽量共用一条底层 WebRTC 传输通道，也就是共用 ICE 网络路径和 DTLS/SRTP 传输。媒体轨本身没有合并。[GStreamer 文档](https://gstreamer.freedesktop.org/documentation/webrtc/) [W3C WebRTC 规范](https://www.w3.org/TR/webrtc/#dom-rtcbundlepolicy)
  >
  > ```
  > 视频轨 ─┐
  > CS2 音频 ├── 共用一条 WebRTC 传输通道
  > OOPZ 音频┘
  > ```
  >
  > **它合并的是“怎么通过网络送过去”，不是“送的媒体内容”。** 接收端仍能区分哪份数据是视频、哪份是 CS2 音频或 OOPZ 音频，因为每条轨道在协商和 RTP 数据里仍有各自的信息。
  >
  > 选择 `max-bundle` 的主要原因是减少底层网络连接和 ICE 候选处理，让多个媒体轨通过同一条路径传输。它本身不代表视频会压缩得更多，也不保证端到端延迟更低。
  >
  > 如果对端不支持 BUNDLE，`max-bundle` 的兼容性会受限；W3C 规范描述的策略是只为一条轨道收集 ICE 候选。对于当前项目，服务端和客户端需要按同一套 WebRTC 协商来处理这些轨道。

- `latency=50`：设置 **50 毫秒的接收端 jitter buffer 缓冲时间**。它不是“整条视频发送延迟为 50 毫秒”的保证；如果这个 `peer` 只发送媒体，这项设置对视频编码和发送排队的影响有限。[GStreamer webrtcbin 文档](https://gstreamer.freedesktop.org/documentation/webrtc/)

  > [!NOTE]
  >
  > 这份代码把 `peer` 的媒体轨设为 `SENDONLY`，因此这个值不会让发出的视频“超过 50 毫秒就被丢掉”。它另建的 `receiver` 是接收轨，`latency=50` 才会用于接收端缓冲。
  >
  > 所以，对这条发送管线来说，延迟超过 50 毫秒**不会因这个参数触发特别处理**。发送时的排队和丢帧行为主要由 `appsrc` 和后面的 `queue` 等设置决定。

#### 2. 视频采集入口分支

视频管线的采集入口由 `gpu_capture` 选择。实况配置的 `video_backend: gstreamer` 会走 GStreamer WGC 分支：WGC 源直接位于 Publisher 管线中，画面以 D3D11 GPU buffer 流向编码器。`video_backend: native` 会走原生 WGC 分支：C++ 采集器生成 `RawVideoFrame`，再通过 `video` 这个 `appsrc` 送进管线。合成/测试帧也使用 `appsrc` 入口。

##### 2.1 GStreamer WGC 分支：当前实况默认路径

~~~
d3d11screencapturesrc name=capture capture-api=wgc
  window-capture-mode=client show-cursor=false !
video/x-raw(memory:D3D11Memory),format=BGRA,framerate=.../1 !
queue name=video_in max-size-buffers=1 max-size-time=0 max-size-bytes=0
  leaky=downstream ! d3d11convert !
video/x-raw(memory:D3D11Memory),format=NV12,colorimetry=bt709 !
~~~

这一分支没有视频 `appsrc`，也不把视频帧先转成 C++ `RawVideoFrame` 再排队。`d3d11screencapturesrc` 直接采集窗口；`video_in` 是 GStreamer 内部的单 buffer 队列；`d3d11convert` 将 BGRA 转换为编码器需要的 NV12。采集源锁定/解锁、队列属性和各项 caps 的解释见[采集.md：GStreamer WGC 管线](采集.md#gstreamer-wgc-管线)。

##### 2.2 原生 WGC 分支：`appsrc` 接收 C++ 帧

`media.video_backend=native` 时，客户端的 `WgcFrameSource` 捕获窗口并生成 `RawVideoFrame`。`media_client` 先把帧放入容量为 1 的 C++ `BoundedQueue`，主循环取出后调用 `publisher->push()`；Publisher 再将 BGRA 数据送入以下 `appsrc`：

~~~
appsrc name=video is-live=true format=time block=false
       max-buffers=1 leaky-type=downstream !
video/x-raw,format=BGRA,width=...,height=...,framerate=.../1 !
videoconvert ! video/x-raw,format=NV12,colorimetry=bt709
~~~

- `appsrc`：应用程序向管线送数据的入口。这里的 buffer 来自 `RawVideoFrame`，不是 GStreamer WGC 源。
- `is-live=true`：告诉 GStreamer 这是实时媒体源。
- `format=time`：让管线按时间轴处理帧的时间戳。
- `block=false`：入口队列满时，调用方不等待队列腾出位置。
- `max-buffers=1`：入口最多排队 1 个 buffer。
- `leaky-type=downstream`：满了就丢掉较旧的 buffer，让较新的帧优先通过。拥堵时可能掉帧。[GStreamer appsrc 文档](https://gstreamer.freedesktop.org/documentation/applib/gstappsrc.html)

> [!NOTE]
>
> `max-buffers=1` 限制的是 `appsrc` 自己的入口队列；它和前面的 C++ `BoundedQueue<RawVideoFrame>` 是两个队列。Native 路径的一帧先经过容量为 1 的 C++ 队列，再进入容量为 1 的 `appsrc` 队列。GStreamer WGC 路径不经过这两个视频帧队列，而使用 `video_in` 队列。

#### 3. Native `appsrc` 输入像素格式和尺寸

~~~
video/x-raw,format=BGRA,width=...,height=...,framerate=.../1
~~~

这是 Native `appsrc` 分支的未压缩视频格式声明：

- `video/x-raw`：原始、未编码的视频。
- `format=BGRA`：每像素按 BGRA 格式存放。
- `width`、`height`：取自 `cfg.media.width` 和 `cfg.media.height`。
- `framerate=目标帧率/1`：声明目标帧率，例如 `60/1` 表示 60 帧/秒。代码也会为送入的帧设置 PTS 和帧时长。

GStreamer WGC 分支的输入 caps 不在这里重复说明，见前面的 WGC 分支和[采集说明](采集.md#gstreamer-wgc-管线)。

#### 4. 转换成编码器需要的像素格式

两种采集分支在编码器前都需要 NV12、BT.709 格式，但使用不同的转换元素：Native `appsrc` 使用 `videoconvert`；GStreamer WGC 使用 `d3d11convert`，并维持 D3D11 GPU 内存类型。

Native 分支的管线片段是：

~~~
videoconvert ! video/x-raw,format=NV12,colorimetry=bt709
~~~

- `videoconvert`：执行 Native 系统内存 buffer 的像素格式或颜色格式转换。
- `format=NV12`：要求转换结果为 NV12，一种常见的 YUV 4:2:0 格式，视频编码器通常支持这种输入。[GStreamer 视频格式文档](https://gstreamer.freedesktop.org/documentation/video/video-format.html)
- `colorimetry=bt709`：声明输出颜色按 BT.709 色彩参数解释；它不是分辨率或编码质量设置。[GStreamer 视频颜色文档](https://gstreamer.freedesktop.org/documentation/video/video-color.html)

两条分支的编码器和后续 H.264/RTP/WebRTC 下游相同。

#### 5. H.264 编码器

```
mfh264enc name=encoder bitrate=... low-latency=true gop-size=...
```

- `mfh264enc`：Media Foundation 的 H.264 编码器。
- `name=encoder`：给编码器命名，代码后续会用这个名字查找它。
- `bitrate=...`：目标码率。这里使用 `cfg.media.bitrate_kbps`；该编码器的 `bitrate` 单位是 kbit/s。
- `low-latency=true`：启用编码器的低延迟模式。
- `gop-size=target_fps`：GOP 中的图像数量。比如目标帧率是 60，GOP 大小就是 60 帧，通常大约对应一秒一组关键帧；实际关键帧间隔还受编码器行为影响。[GStreamer mfh264enc 文档](https://gstreamer.freedesktop.org/documentation/mediafoundation/mfh264enc.html)

> [!NOTE]
>
> `mfh264enc` 是编码器的**元素类型**，`name=encoder` 给这个编码器实例取名为 `encoder`；`encoder` 不是固定关键字，可以换成别的名字，但代码里查找它的地方也要同步改。

#### 6. H.264 输出格式约束

```
video/x-h264,stream-format=byte-stream,
alignment=au,profile=constrained-baseline
```

这段声明编码器输出必须符合的 H.264 格式：

- `video/x-h264`：已经编码的 H.264 视频。

- `stream-format=byte-stream`：使用 H.264 byte-stream 格式。

  > [!NOTE]
  >
  > H.264 视频由一段段 NAL 单元组成。`byte-stream` 表示用 Annex B 格式组织它们：在 NAL 单元前加起始码，例如 `00 00 01`，让后续元素能找到边界。

- `alignment=au`：按 Access Unit 对齐，通常一个 buffer 对应一帧的编码单元。(编码后的 H.264 AU buffer )

  > `au` 是 Access Unit（访问单元），通常包含解码一帧所需的 H.264 数据

- `profile=constrained-baseline`：要求使用 constrained-baseline profile，偏向较广的设备兼容性。

  > [!NOTE]
  >
  > H.264 有不同的 profile，可以把它理解成**编码器和解码器约定使用哪些编码功能**。`constrained-baseline` 是 WebRTC 常用的兼容性 profile；WebRTC 视频规范要求设备支持它。[RFC 7742](https://datatracker.ietf.org/doc/html/rfc7742)

这些是格式协商条件，本身不会把不符合条件的视频“转换”成对应格式；编码器需要能输出这种格式。[GStreamer H.264 parser 文档](https://gstreamer.freedesktop.org/documentation/videoparsersbad/h264parse.html)

#### 7. 时间轴整理和 H.264 解析

```
identity name=normalized single-segment=true ! h264parse
```

- `identity`：默认把数据原样传下去。

  > [!NOTE]
  >
  > 管线还需要把这些数据整理成后续 RTP 打包器能处理的形式，所以后面仍有 `h264parse` 和 `rtph264pay`。`single-segment=true` 还会影响时间戳和 segment 事件

- `name=normalized`：给它命名；这份代码在它的输出端挂了探针，用来检查编码后的视频 buffer。

  > [!NOTE]
  >
  > C++ 代码则按 `normalized` 这个名字找到该元素，并在它的输出端装一个“探针”。探针会在 buffer 经过时被调用，但不会把它从管线里拿走。
  >
  > **探针负责建立“编码后 buffer → 原始采集帧”的对应关系**** 探针代码 (line 290)（源码位置：<code>client/src/media/webrtc_publisher.cpp:290</code>）  安装探针的位置 (line 858)（源码位置：<code>client/src/media/webrtc_publisher.cpp:858</code>）

  

  > [!TIPS]
  >
  > 探针是为了在视频被编码后，仍然认得出它来自哪一帧。
  >
  > 比如客户端采集到“帧 ID 123，采集时间 T”，先把这份信息记下来。编码器压缩视频后，探针读取编码输出上的时间信息，把它对应回“帧 ID 123，采集时间 T”。再往后，RTP 探针就能把帧 ID 和采集时间写进 RTP 扩展里。
  >
  > 所以流程是：
  >
  > ```
  > 采集帧及其身份信息
  > → 编码
  > → normalized 探针把编码输出对应回原始帧
  > → RTP 探针把身份信息写入 RTP 扩展
  > ```
  >
  > 这个探针不检查画面内容；它检查的是**编码后 buffer 是否还带着足够的时间信息，能否关联回原始帧**。它负责建立对应关系，真正写 RTP 扩展的是后面的 RTP 探针。

##### `normorlized`探针

> [!NOTE]
>
> 管线创建后，代码找到名为 `normalized` 的元素，取它的输出 pad（`src`），再注册 `Impl::encodedVideo` 作为探针回调：
>
> ```
> auto *element = gst_bin_get_by_name(GST_BIN(self.pipeline), "normalized");
> auto *src = gst_element_get_static_pad(element, "src");
> gst_pad_add_probe(src, GST_PAD_PROBE_TYPE_BUFFER,
>                   Impl::encodedVideo, &self.tracks[0], nullptr);
> ```
>
> 因此，“`normalized` 的探针”只是简称：准确说，是**挂在 `normalized` 元素输出端的 buffer 探针**。每个 buffer 经过那里时，GStreamer 就会调用 `Impl::encodedVideo`。代码见 <code>client/src/media/webrtc_publisher.cpp:878</code>；回调行为见 [Impl::encodedVideo](WebRTC信令与回调实现.md#impl-encoded-video)。

- `single-segment=true`：把时间戳和 segment 事件整理成单段的表现形式。

> [!NOTE]
>
> GStreamer 的 *segment* 是一段媒体的**时间轴规则**，用于解释 buffer 时间戳如何对应播放时间。它不是视频片段、关键帧组，也不是一帧。
>
> ```
> 一个 segment（时间轴规则）
>   ├─ buffer：AU 1，PTS 0 ms
>   ├─ buffer：AU 2，PTS 33 ms
>   ├─ buffer：AU 3，PTS 67 ms
>   └─ ...
> ```

> [!IMPORTANT]
>
> **segment 不是一批要单独解析的视频数据**，而是告诉下游“这些 buffer 的时间戳该怎样解释”的时间轴事件。
>
> 通常是先收到一个 segment 事件，接着收到许多带 PTS 的 H.264 buffer：
>
> ```
> segment 事件 → AU buffer 1 → AU buffer 2 → AU buffer 3 → ...
> ```
>
> `single-segment=true` 的作用是：如果上游送来新的 segment 事件，`identity` 会处理这些变化，让下游看起来仍处在一条连续的时间轴上。`h264parse` 主要是逐个处理后续的 H.264 buffer，不是按 segment 分批解析。
>
> `single-segment=true` 会整理时间戳，并处理 segment 事件，让下游看到一条单段时间轴。对这条管线来说，它处理的是时间轴表现；`identity` 中的 H.264 字节仍继续往下传。[GStreamer identity 文档](https://gstreamer.freedesktop.org/documentation/coreelements/identity.html)

- `h264parse`：解析并整理 H.264 码流，确保后续 RTP 打包器能按预期接收它。[GStreamer identity 文档](https://gstreamer.freedesktop.org/documentation/coreelements/identity.html) [GStreamer H.264 parser 文档](https://gstreamer.freedesktop.org/documentation/videoparsersbad/h264parse.html)

#### 8. H.264 打成 RTP 包

```
rtph264pay name=vpay pt=96 ssrc=... mtu=1200 config-interval=-1
```

- `rtph264pay`：把 H.264 码流封装成 RTP 包。
- `name=vpay`：给打包器命名；代码后续会找到它并设置 RTP 扩展。
- `pt=96`：RTP payload type，告诉接收端这类 RTP 包对应协商的媒体格式。**[自定义] 96：视频格式**
- `ssrc=...`：这条 RTP 媒体流的标识。代码启动时为 track 生成 SSRC。
- `mtu=1200`：把 RTP 包的最大尺寸限制在约 1200 字节。
- `config-interval=-1`：每逢 IDR 关键帧，就把 H.264 的 SPS/PPS 参数一并发送，帮助接收端开始解码或恢复解码。[GStreamer rtph264pay 文档](https://gstreamer.freedesktop.org/documentation/rtp/rtph264pay.html)

#### 9. 声明 RTP 内容和采集信息扩展

```
application/x-rtp,media=video,encoding-name=H264,
payload=96,clock-rate=90000,ssrc=(uint)...,
extmap-7=(string)<URI>
```

这段是告诉 `webrtcbin` 后面收到的 buffer 是什么类型：

- `application/x-rtp`：RTP 格式的数据。
- `media=video`：这是视频轨。
- `encoding-name=H264`：编码格式是 H.264。
- `payload=96`：payload type，与前面的 `pt=96` 对应。
- `clock-rate=90000`：H.264 RTP 时间戳使用每秒 90,000 个时钟单位。
- `ssrc=(uint)...`：用**整数类型**声明这条 RTP 流的 SSRC。
- `extmap-7=(string)...`：声明 RTP 扩展 ID 7 对应的 URI。URI 标识 TacticLens 的采集元信息扩展；代码还会把实际的采集时间和帧 ID 写入 RTP 包。这里的 `(uint)`、`(string)` 是 GStreamer caps 的类型标记。

#### 10. 队列和 WebRTC 视频输入口

```
queue max-size-buffers=8 max-size-time=0 max-size-bytes=0 ! peer.sink_0
```

- `queue`：在发送支路上加一个有界队列，并把上下游处理解耦。
- `max-size-buffers=8`：最多排队 8 个 buffer。因为它位于 RTP 打包器之后，这里通常数的是 RTP buffer/包，不是 8 帧。
- `max-size-time=0`、`max-size-bytes=0`：关闭按排队时间和字节数计算的限制，只保留 8 个 buffer 这个上限。
- 队列默认满时会阻塞上游；它这里没有设置丢弃策略。因此，上游的 `appsrc` 和这条 `queue` 是两层不同的队列，只有前者明确配置了“满时丢旧帧”。[GStreamer queue 文档](https://gstreamer.freedesktop.org/documentation/coreelements/queue.html)
- `peer.sink_0`：连接到名为 `peer` 的 WebRTC 元素的第 0 个输入轨，也就是视频 RTP 轨。

> [!TIPS]
>
> **合起来看**：Native 分支由客户端把 BGRA 帧放进 `video`；GStreamer WGC 分支由 `d3d11screencapturesrc` 直接提供画面。两条分支都会把画面整理为 NV12、编码成 H.264、封成带采集扩展信息的 RTP 包，再送进 WebRTC 视频轨。

### 音频管线配置 webrtc_publisher.cpp:714（源码位置：<code>client/src/media/webrtc_publisher.cpp</code>）

#### 配置类型编号

```c++
const auto name = source == 0 ? "cs2" : "oopz";
const auto payload = std::to_string(97 + source);
```

`std::to_string` 把数字转换成字符串：

- `source == 0`：`payload` 是 `"97"`
- `source == 1`：`payload` 是 `"98"`

#### 1. `appsrc`：把程序中的音频交给 GStreamer

```
appsrc name=cs2 ...
```

`appsrc` 是一个“程序输入口”：它让 C++ 代码可以把采集到的音频数据推入 GStreamer。

这里的 `appsrc` 只负责接收已经采集并整理好的 PCM，不负责采集设备或进程音频。当前实况音频由 `IndependentAudioSource` 通过两路 WASAPI process-loopback 获取，分成 CS2/Oopz 的 20 毫秒音频包，经 C++ 有界队列和 `publisher->push(AudioPacket)` 后分别进入 `cs2` 与 `oopz`。采集和分包过程见[采集.md：WASAPI 音频采集](采集.md#wasapi-音频采集)。

其中：

- `name=cs2`：给这个输入口命名。另一条链会叫 `oopz`。
- `is-live=true`：这是实时数据源，不是一个已经准备好的媒体文件。
- `format=time`：音频缓冲区的时间戳按时间单位处理。
- `block=false`：推送数据时不要因为输入队列满了而一直阻塞。
- `max-buffers=...`：限制 `appsrc` 里最多暂存多少个音频缓冲区。这个数来自 `cfg.media.max_audio_packets`；配置结构体默认值是 10，也可能被 YAML 配置覆盖。
- `leaky-type=downstream`：队列装满时丢掉较旧的数据，让后续处理优先拿到较新的音频。这种设置有助于控制实时传输延迟。

`appsrc` 的输入缓冲区来自程序采集到的音频。该文件后面的代码把音频样本推入对应的 `appsrc`。

#### 2. `audio/x-raw,...`：声明输入是怎样的 PCM 音频

```c++
audio/x-raw,format=S16LE,rate=16000,channels=1,layout=interleaved
```

这段是 GStreamer 的 **caps**，可以理解为“媒体格式说明”，告诉后续环节输入数据是什么格式：

- `audio/x-raw`：未压缩的原始音频，也就是 PCM。
- `format=S16LE`：每个样本是 16 位有符号整数，按小端序存放。
- `rate=16000`：每秒 16,000 个音频样本，即 16 kHz。
- `channels=1`：单声道。
- `layout=interleaved`：多声道音频样本交错排列；这里是单声道，所以这个设置不涉及左右声道交替。

可以把它理解成：**告诉编码器，接下来收到的是 16 kHz、单声道、16 位的 PCM 音频。**

#### 3. `opusenc`：把 PCM 压缩成 Opus

```c++
opusenc name=cs2enc frame-size=20 dtx=false inband-fec=false bitrate=...
```

`opusenc` 是 Opus 编码器。它把未压缩的 PCM 转成体积更小的 Opus 数据，适合网络实时传输。

- `name=cs2enc`：给编码器起名；`oopz` 那条会叫 `oopzenc`。
- `frame-size=20`：每个 Opus 编码帧对应 20 毫秒音频。
- `dtx=false`：关闭不连续传输。静音时也不会通过 DTX 机制省略发送。
- `inband-fec=false`：关闭 Opus 的带内前向纠错数据。
- `bitrate=...`：编码目标码率，单位是 bit/s。

码率这里这样计算：

```c++
cfg.media.audio_bitrate_kbps * 1000
```

配置项以 kbps 为单位，乘以 1000 转成编码器使用的 bit/s。结构体默认是 48 kbps，所以默认生成的数值是 `48000`；实际值可能由配置覆盖。

#### 4. `rtpopuspay`：把 Opus 数据装进 RTP 包

```
rtpopuspay name=cs2pay pt=97 ssrc=... mtu=1200
```

Opus 编码结果还需要套上网络传输使用的 RTP 包头。`rtpopuspay` 负责这个封装。

- `name=cs2pay`：给封装器起名；另一条叫 `oopzpay`。
- `pt=97`：这个 RTP 流使用的 payload number。`oopz` 使用 98。
- `ssrc=...`：这个 RTP 来源的标识。这里取自 `self.tracks[source + 1].ssrc`。
- `mtu=1200`：把 RTP 包大小控制在约 1200 字节以内。

为什么音频轨道用 `source + 1`？这个程序把 `self.tracks[0]` 留给视频，所以两个音频来源使用 `self.tracks[1]` 和 `self.tracks[2]`。`ssrc` 是 RTP 中用于区分数据来源的编号，和前面提到的 payload number 是两种不同的编号。

#### 5. `application/x-rtp,...`：声明输出是怎样的 RTP 流

```c++
application/x-rtp,media=audio,encoding-name=OPUS,
payload=97,clock-rate=48000,ssrc=(uint)...,
extmap-7=(string)...
```

这是另一段 caps，描述 `rtpopuspay` 输出的 RTP 流：

- `application/x-rtp`：数据现在是 RTP 格式。
- `media=audio`：这是音频流。
- `encoding-name=OPUS`：RTP 里的编码格式是 Opus。
- `payload=97`：RTP payload number，要和前面的 `pt=97` 对应。
- `clock-rate=48000`：RTP 时间戳使用 48 kHz 时钟。它和输入 PCM 的 `rate=16000` 说的是不同阶段的时间刻度，所以两者不同并不矛盾。
- `ssrc=(uint)...`：声明 SSRC 是无符号整数，并标明这个 RTP 流的 SSRC。
- `extmap-7=(string)...`：声明 RTP 扩展编号 7 对应哪个扩展 URI。

> [!NOTE]
>
> 这里的扩展 URI 是：
>
> ```
> urn:tacticlens:rtp-hdrext:capture-v1
> ```
>
> 它是 TacticLens 自定义的**采集元数据扩展**。代码的其他位置会把采集相关的身份和时间信息附加到 RTP 包上；这里的 `extmap-7` 则声明“扩展编号 7 对应这类元数据”。这样接收方才能识别这个扩展。

> [!WARN]
>
> `rate=16000`：原始 PCM **每秒有 16,000 个采样点**，这是采样率。
>
> `clock-rate=48000`：RTP 时间戳的刻度是**每秒 48,000 个时间单位**。它不是某个具体时间戳，而是解释时间戳变化速度的标尺。
>
> **20ms 发一次**
>
> 每段PCM 里有 **320 个样本点**：16,000 × 0.02
>
> RTP 时间戳前进 **960 个刻度**：48,000 × 0.02

#### 6. `queue`：在编码后暂存少量 RTP 包

```
queue max-size-buffers=5 max-size-time=0 max-size-bytes=0
```

`queue` 是一个缓冲队列，可以让前后处理环节在一定程度上独立运行。它位于 RTP 封装之后，所以这里排队的是 RTP 数据。

- `max-size-buffers=5`：最多排队 5 个缓冲区。
- `max-size-time=0`：不按时间长度设置额外上限。
- `max-size-bytes=0`：不按字节数设置额外上限。

注意：这里的 `queue` 没有设置 `leaky-type`。前面 `appsrc` 的 `leaky-type=downstream` 才是明确指定“满了时丢旧数据”的位置。

> [!WARN]
>
> 这个 `queue` 满了会让对应的 GStreamer 音频分支暂时停下来处理数据，但不会直接停止音频采集
>
> 按这条链路看，`queue` 在 Opus 编码和 RTP 封装之后。它满时会阻塞上游的推送线程，因此 `rtpopuspay` 暂时无法继续送包，`opusenc` 也就无法继续处理后续音频；等 WebRTC 端取走数据、队列腾出空间后，处理会恢复。GStreamer 的 `queue` 默认就是满时阻塞上游线程。[queue 文档](https://gstreamer.freedesktop.org/documentation/coreelements/queue.html)
>
> 采集线程是独立运行的，仍会继续采集。缓冲积压到上游后，数据可能在别处被丢弃：
>
> - `appsrc` 设置了 `leaky-type=downstream`，自己的缓冲队列满时会丢较旧的数据。
> - 客户端的 `BoundedQueue` 满时也会丢最旧的音频包。(**`BoundedQueue` 自己满，主要表示采集线程产出速度暂时超过了媒体主循环取走的速度**)
>
> 所以更准确地说：**下游变慢时，编码和发送可能暂时等待；采集继续，但持续拥堵可能造成音频丢包或缺口。** [appsrc 文档](https://gstreamer.freedesktop.org/documentation/app/appsrc.html)

#### 7. `peer.sink_...`：交给 WebRTC

```
peer.sink_1
```

`peer` 是这条 GStreamer pipeline 中名为 `peer` 的 `webrtcbin`。`webrtcbin` 负责 WebRTC 的媒体协商和传输。

- `peer.sink_1`：把 `cs2` 音频交给 WebRTC 的第 1 号输入轨道。
- `peer.sink_2`：把 `oopz` 音频交给第 2 号输入轨道。

这和同一文件里的轨道映射一致：视频用 `sink_0`，`cs2` 用 `sink_1`，`oopz` 用 `sink_2`。

#### 整体流程

通过for循环循环两个音轨

```
cs2 音频 → appsrc → PCM 格式声明 → Opus 编码 → RTP 封装 → queue → WebRTC
oopz 音频 → appsrc → PCM 格式声明 → Opus 编码 → RTP 封装 → queue → WebRTC
```

> [!NOTE]
>
>  CS2 和 OOPZ 各建一条独立的音频链，把 16 kHz PCM 转成 Opus，再封装为两条可区分的 RTP 流，最后交给 WebRTC 发送。

### 取出管线的**消息通道**

```c++
self.bus = gst_element_get_bus(self.pipeline);
```

拿到管线的 **Bus**。它传递的是错误、警告、延迟变化等状态消息，**不是音频或视频数据**。后面的代码会从它读取消息，遇到错误时记录失败。

### 获取WebRTC 元件

```c++
self.peer = gst_bin_get_by_name(GST_BIN(self.pipeline), "peer");
```

在管线里查找名为 `peer` 的元件。这个名字来自前面创建管线时的 `webrtcbin name=peer`，所以 `self.peer` 就是负责 WebRTC 连接的 `webrtcbin`。代码之后通过它创建 offer、处理 ICE 候选和 SDP 应答。

> [!NOTE]
>
> `bus` 是状态消息通道，`peer` 是管线里的 WebRTC 通信端

### 给三条流接上 WebRTC 所需的设置



#### 整体流程

三条流的对应关系是：

| `i`  | 输入源  | RTP 发送器 | 媒体       |
| ---- | ------- | ---------- | ---------- |
| 0    | `video` | `vpay`     | H.264 视频 |
| 1    | `cs2`   | `cs2pay`   | Opus 音频  |
| 2    | `oopz`  | `oopzpay`  | Opus 音频  |

整体数据路径可以先记成：

```
video 输入 → H.264 编码 → RTP 打包 ─┐
cs2 音频输入 → Opus 编码 → RTP 打包 ├→ webrtcbin → WebRTC 发送
oopz 音频输入 → Opus 编码 → RTP 打包 ┘
```

这段循环大致依次做五件事：

1. **找到这一轨的输入和 RTP 发送器。**
   `track.source` 保存 `appsrc` 输入源，后续代码会通过它送入视频帧或音频包。`pay` 是把编码后的数据打成 RTP 包的组件。
2. **给 RTP 包加采集标识。**
   它把自定义扩展注册到 RTP 发送器，扩展编号是 7。接着在发送器的输出端安装 `Impl::rtp` probe。可以把 probe 理解成数据经过某个接口时执行的钩子：这里会在 RTP 包发送前补上采集元数据，例如对应的采集时间或源单元编号。远端也需要支持相同扩展，周围代码会检查协商结果。
3. **给音频编码器加一个额外钩子。**
   `i` 为 1 或 2 时会找到 Opus 编码器，并在编码输出处安装 `encodedAudio` probe。它记录编码输出情况，并处理 Opus 编码器的预留采样等时间细节；实际 RTP 元数据是在上一步的 `Impl::rtp` 里写入。
4. **阻止 EOS 过早结束 WebRTC 轨道。**
   EOS 是“数据流结束”事件。音频编码器收到结束信号时，可能还要输出最后几包编码数据；代码在 WebRTC 输入 pad 上丢弃 EOS，让这些尾部数据有机会继续送出。
5. **设置 WebRTC 轨道的发送参数。**
   它把对应 transceiver 设成 `SENDONLY`，并声明媒体类型、编码格式、RTP payload 编号、时钟频率和扩展编号。例如视频使用 H.264、payload 96、90 kHz；两路音频使用 Opus、payload 97/98、48 kHz。这些信息会用于 WebRTC 的媒体协商。

循环结束后，`start()` 还会连接协商和 ICE 信号、启动发送流水线，并发起 WebRTC offer。之后其他代码才会把采集到的帧和音频送进 `appsrc`。所以这段的核心可以概括为：**逐条配置三路媒体流，把采集身份信息挂到 RTP 包上，并准备好 WebRTC 发送协商。**

### 获取输入源

```c++
track.source = gst_bin_get_by_name(
    GST_BIN(self.pipeline),
    i == 0 ? "video" : i == 1 ? "cs2" : "oopz");
```

流水线里按名字查找输入源，并保存到 `track.source`，供后续代码往这个输入源送视频帧或音频数据。

名字由 `i` 决定：

- `i == 0`：找 `video`
- `i == 1`：找 `cs2`
- `i == 2`：找 `oopz`

这不是新建输入源；输入源是在前面构建流水线时创建的。

### 获取RTP打包器

```c++
auto *pay = gst_bin_get_by_name( // 按名字从现有流水线中查找组件。
    GST_BIN(self.pipeline),
    i == 0 ? "vpay" : i == 1 ? "cs2pay" : "oopzpay");
```

RTP 打包器（payloader）。它负责把编码后的视频或音频整理成 RTP 包，下面的代码会在它上面配置 RTP 扩展。

### 创建一个 RTP 头扩展对象

```c++
auto *extension =
    gst_rtp_header_extension_create_from_uri(
        tl::media::kCaptureExtensionUri);
```

根据项目定义的 URI 创建一个 RTP 头扩展对象。后面的代码会把它加到 `pay` 上，让每个 RTP 包可以携带项目自己的采集信息。具体写入信息是在后续的 RTP probe 中

### 设置扩展编号

```c++
gst_rtp_header_extension_set_id(extension, kExtensionId);
```

给这个扩展设置 RTP 头扩展编号。`kExtensionId` 在这份代码里是 `7`。接收端之后也会用这个编号识别扩展。

```c++
g_signal_emit_by_name(pay, "add-extension", extension);
```

通知 `pay`（RTP 打包器）把这个扩展加入自己的配置。此后打包器就知道这路 RTP 流使用编号为 7 的扩展。

```
gst_object_unref(extension);
```

释放当前代码持有的引用。扩展已经交给 `pay` 使用，所以这里释放本地引用不代表扩展立刻失效。

### 添加rtp打包时的探针

```c++
auto *pad = gst_element_get_static_pad(pay, "src");
```

找到 `pay` 的 `src` 输出 pad。可以把 pad 理解成 GStreamer 组件的连接口：`src` 是输出端。

```c++
gst_pad_add_probe(
    pad,
    static_cast<GstPadProbeType>(GST_PAD_PROBE_TYPE_BUFFER |
                                 GST_PAD_PROBE_TYPE_BUFFER_LIST),
    Impl::rtp, &track, nullptr);
```

在这个输出口安装 probe：

- `BUFFER`：数据以单个缓冲区传来时触发。
- `BUFFER_LIST`：数据以一组缓冲区传来时触发。
- `Impl::rtp`：数据到达时要调用的**处理函数**。
- `&track`：作为参数传给处理函数，让它知道当前 RTP 包属于视频、CS2 音频还是 OOPZ 音频。
- 最后的 `nullptr`：没有提供额外的清理回调。

结合 `Impl::rtp` 的实现，这个处理函数会**检查 WebRTC 是否已连接**，并为** RTP 包补上前面配置的采集元数据**；处理失败或尚未连接时，会丢弃该包；具体处理见 [Impl::rtp 与 inject](WebRTC信令与回调实现.md#impl-rtp-inject)。

```c++
gst_object_unref(pad);
gst_object_unref(pay);
```

释放这里临时持有的 `pad` 和 `pay` 引用。它们仍属于流水线；释放引用不会删除组件或移除刚才安装的 probe。

> [!NOTE]
>
> 是在 `pay` 的输出接口上安装一个“检查点”：每个 RTP 包离开打包器时，先经过 `Impl::rtp` 处理，再继续流向后面的队列和 WebRTC 发送端。

### 添加音频编码器的输出端探针

```c++
      auto *encoder = gst_bin_get_by_name(GST_BIN(self.pipeline),
                                          i == 1 ? "cs2enc" : "oopzenc");
      auto *encoded = gst_element_get_static_pad(encoder, "src");
      gst_pad_add_probe(encoded, GST_PAD_PROBE_TYPE_BUFFER, Impl::encodedAudio,
                        &track, nullptr);
      gst_object_unref(encoded);
      gst_object_unref(encoder);
```

```c++
      gst_pad_add_probe(encoded, GST_PAD_PROBE_TYPE_BUFFER, Impl::encodedAudio,
                        &track, nullptr);
```

**`encoded`：要监听的位置**
前面的代码把它设为 Opus 编码器的 `src` 输出口。也就是说，监听编码后的音频经过这里时发生的事情。

**`GST_PAD_PROBE_TYPE_BUFFER`：什么情况触发**
表示有一个单独的缓冲区（`buffer`）经过时，调用回调。这里没设置 `BUFFER_LIST`，所以不监听成组的缓冲区。

**`Impl::encodedAudio`：触发时要调用的函数**
注意这里没有括号 `()`：传进去的是函数本身，告诉 GStreamer“以后触发时调用它”，并不是现在就调用。函数名带 `Impl::`，表示它属于 `Impl` 这个结构体。

**`&track`：额外传给回调的数据**
`&` 是“取地址”，这里把当前轨道 `track` 的地址传进去。这样回调就能知道正在处理哪一路音频。它只是传递指针，不会复制 `track`。

**`nullptr`：清理函数**
如果探针被移除，GStreamer 可以调用清理函数来释放第 4 个参数传入的数据。这里传 `nullptr`，表示没有额外的清理函数；`track` 本身不需要在这里释放。

之后 GStreamer 调用回调时，会把 pad、缓冲区信息和 `&track` 传给它。`encodedAudio` 用这个 `track` 记录编码信息，完成后返回 `GST_PAD_PROBE_OK`，表示缓冲区可以继续沿流水线传递。回调对 Opus clipping metadata 的用途见 [Impl::encodedAudio](WebRTC信令与回调实现.md#impl-encoded-audio)。

### 添加视频编码器的输出端探针

```c++
  auto *encoder = gst_bin_get_by_name(GST_BIN(self.pipeline), "normalized");
  auto *encoded = gst_element_get_static_pad(encoder, "src");
  gst_pad_add_probe(encoded, GST_PAD_PROBE_TYPE_BUFFER, Impl::encodedVideo,
                    &self.tracks[0], nullptr);
```

同上； webrtc_publisher.cpp:868（源码位置：<code>client/src/media/webrtc_publisher.cpp</code>）

### 添加输入`webrtc`接口的探针

> [!NOTE]
>
> ```text
> appsrc（音频输入）
>   → opusenc（编码）
>   → rtpopuspay（打成 RTP 包）
>   → queue（队列）
>   → webrtcbin 的 sink_i（WebRTC 输入）
> ```

**探针就加在最后这个 `sink_i` 上**。代码先用下面这行取得 WebRTC 的输入接口：

```
auto *peerpad = gst_element_get_static_pad(
    self.peer, ("sink_" + std::to_string(i)).c_str());
```

循环中 `i` 是 0、1、2，所以分别取得 `sink_0`、`sink_1`、`sink_2`。接着的 `gst_pad_add_probe(...)` 就把探针装在这个接口上。它监听的是**下游事件**，不是普通音频/RTP 数据包。

运行时 RTP 包经过这里，会继续进入 WebRTC。结束音频流时，`opusenc` 可能先输出编码器内部剩余的尾部数据；这些数据继续经过 RTP 打包器和队列。随后 EOS（流结束）事件也会沿着同一路径到达探针。回调发现这是 EOS，就把 EOS 拦住，让它不要传进 `webrtcbin`，避免 WebRTC 过早结束这条传输。

> [!TIPS]
>
> **它加在 RTP 流进入 WebRTC 的输入口，专门拦截 EOS 结束事件。**

### 获取WebRTC 轨道的设置

```c++
    GstWebRTCRTPTransceiver *transceiver = nullptr;
    g_object_get(peerpad, "transceiver", &transceiver, nullptr);
```

`peerpad` 是 `webrtcbin` 的 `sink_i` 输入接口。这里从它取得关联的 `transceiver`，也就是负责这条 WebRTC 媒体轨道发送或接收设置的对象。

### 配置 WebRTC 轨道的格式

```c++
      const std::string codec =
          "application/x-rtp,media=" + std::string(i == 0 ? "video" : "audio") +
          ",encoding-name=" + (i == 0 ? "H264" : "OPUS") +
          ",payload=" + std::to_string(96 + i) +
          ",clock-rate=" + std::to_string(i == 0 ? 90000 : 48000) +
          ",extmap-7=(string)" + tl::media::kCaptureExtensionUri;
```

- **`application/x-rtp`**：告诉 GStreamer，描述的是 RTP 格式的数据。
- **`media=video` / `media=audio`**：说明这是视频轨道还是音频轨道。`i == 0` 是视频，其他两路是音频。
- **`encoding-name=H264` / `encoding-name=OPUS`**：说明 RTP 包里装的是哪种编码格式。
- **`payload=96+i`**：RTP payload 编号。三条轨道分别是 96、97、98，和前面 RTP 打包器设置的编号对应。
- **`clock-rate=...`**：RTP 时间戳使用的时钟频率。视频是 90000；Opus 是 48000。即使项目送入 Opus 编码器的原始音频是 16000 Hz，RTP 中的 Opus 时间戳仍使用 48000 Hz 时钟。
- **`extmap-7=(string)...`**：声明 RTP 扩展编号 7 对应哪个 URI。这里的 7 和前面调用 `gst_rtp_header_extension_set_id(extension, kExtensionId)` 设置的编号相对应；`(string)` 是 GStreamer 格式描述里的类型标记。

> [!NOTE]
>
> 最后，代码会把这个字符串交给 `gst_caps_from_string` 转成 GStreamer 的 `caps`，再用于配置 WebRTC 轨道的格式。

### 设置接口格式

```c++
g_object_set(transceiver, "direction",
             GST_WEBRTC_RTP_TRANSCEIVER_DIRECTION_SENDONLY,
             "codec-preferences", caps, nullptr);
```

**`transceiver`**：要修改哪个对象。它是前面从 `peerpad` 上取得的 WebRTC 轨道对象。

**`"direction"`**：第一个要修改的属性名，意思是“传输方向”。

**`GST_WEBRTC_RTP_TRANSCEIVER_DIRECTION_SENDONLY`**：`direction` 的值，表示本端只通过这条轨道发送媒体。

**`"codec-preferences"`**：第二个要修改的属性名，意思是“编解码格式偏好”。

**`caps`**：`codec-preferences` 的值。它是前面生成的格式说明，包含当前轨道的媒体类型、编码格式、payload 编号和时钟频率等信息。

**`nullptr`**：结束标记。它告诉 `g_object_set`：“属性和值到这里结束了。”它不是某个属性的值。

### 增加'编码格式'的配置

```
gst_caps_set_simple(caps, "ssrc", G_TYPE_UINT, track.ssrc, nullptr);
```

**`caps`**：要修改的格式说明对象。

**`"ssrc"`**：要添加或设置的字段名。SSRC 是这条 RTP 流的标识。

**`G_TYPE_UINT`**：告诉 GStreamer，接下来这个字段的值是无符号整数类型。

**`track.ssrc`**：字段的实际值，也就是当前轨道自己的 SSRC。

**`nullptr`**：结束标记，表示后面没有更多字段和值。

>  [!NOTE]
>
> 把当前轨道的 `ssrc` 加入格式信息。SSRC 是用来区分 RTP 流的标识；每条轨道在前面的代码中都生成了自己的值。

```c++
gst_pad_send_event(peerpad, gst_event_new_stream_start(...));
gst_pad_send_event(peerpad, gst_event_new_caps(caps));
```

向 `peerpad` 依次发送两个 GStreamer 事件：

1. `stream-start`：告诉它一条媒体流开始了，并给这条流一个名字，例如 `media-0`。
2. `caps`：告诉它这条流的媒体格式，包括编码、payload 编号、扩展和 SSRC 等信息。

### 对webRtc输入接口设置流信息

#### 声明新流信息

```
gst_pad_send_event(
    peerpad,
    gst_event_new_stream_start(
        ("media-" + std::to_string(i)).c_str()));
```

`gst_pad_send_event` 接收两个参数：

1. **`peerpad`**：事件要送到哪个接口。这里是 `webrtcbin` 当前轨道的 `sink_i` 输入接口。

2. **`gst_event_new_stream_start(...)`**：要发送的事件。这个函数先创建一个 `STREAM_START` 事件，再把它作为第二个参数交给 `gst_pad_send_event`。

   > 它接收一个流名称。`std::to_string(i)` 把轨道编号转成文字，再与 `"media-"` 拼接。例如 `i == 0` 时，流名称是 `"media-0"`。`.c_str()` 把它转换成 GStreamer 需要的字符串形式。
   >
   > 因此，整句的意思是：**向当前轨道的 `peerpad` 发送一个 `STREAM_START` 事件，并把这条流命名为 `media-0`、`media-1` 或 `media-2`。**

#### 声明这条流的格式

```
gst_pad_send_event(peerpad, gst_event_new_caps(caps));
```

**`peerpad`**：当前这条轨道的 `webrtcbin` 输入接口 `sink_i`。

**`gst_event_new_caps(caps)`**：根据 `caps` 创建一个 `CAPS` 事件。`caps` 里描述了这条流的媒体类型、编码、RTP payload 编号、时钟频率、扩展信息和 SSRC 等。

> [!NOTE]
>
> **把当前轨道的格式说明发送到 `webrtcbin` 的输入接口**，让它知道后续进入的数据是什么格式。它发送的是格式事件，不是 RTP 数据包。

### 注册 WebRTC 协商回调

```c++
g_signal_connect(self.peer, "on-negotiation-needed",
                 G_CALLBACK(Impl::negotiate), &self);
```

- `self.peer` 是 `webrtcbin`。
- `on-negotiation-needed` 是它发出的信号，表示 WebRTC 需要进行媒体协商。
- `Impl::negotiate` 是收到信号时要调用的函数，它会创建 WebRTC offer。
- `&self` 会传给回调，让回调能访问当前的 `Impl` 状态。

### 注册 ICE 相关回调

```c++
g_signal_connect(self.peer, "on-ice-candidate", G_CALLBACK(Impl::ice), &self);
```

**`self.peer`**：要监听哪个对象。这里是发送端的 `webrtcbin`。

**`"on-ice-candidate"`**：要监听的信号名称。`webrtcbin` 每收集到一个 ICE candidate 时，会发出这个信号。

**`G_CALLBACK(Impl::ice)`**：信号发生时要调用的项目回调函数。`G_CALLBACK` 把函数指针转换成 GObject 信号系统要求的类型。

**`&self`**：传给回调的附加数据。`Impl::ice` 会把它还原成 `Impl` 对象，以便访问当前发布器的状态。

每当 WebRTC 收集到一个 ICE candidate，就调用 `Impl::ice`，把候选连接信息放入信令消息，交给对端。

> [!NOTE]
>
> ICE candidate 可以理解为 **WebRTC 尝试连接时可用的一种网络地址和端口方案**。它不是音视频数据，也不代表连接已经建立

```c++
g_signal_connect(self.peer, "notify::ice-gathering-state",
                 G_CALLBACK(Impl::gathering), &self);
```

ICE 收集状态发生变化时调用 `Impl::gathering`。例如收集完成后，代码会发出“候选已全部发送”的通知。

### 启动 GStreamer 流水线

```c++
if (gst_element_set_state(self.pipeline, GST_STATE_PLAYING) ==
    GST_STATE_CHANGE_FAILURE) {
```

要求整个媒体流水线进入 `PLAYING` 状态，也就是开始运行、准备处理数据。

如果启动失败：

```c++
stop();
throw std::runtime_error("media pipeline failed to start");
```

先调用 `stop()` 清理流水线，再抛出异常，把启动失败报告给上层。

## 下一页

Offer/Answer、ICE candidate、RTP 元数据回调和服务端接收流程见 [WebRTC信令与回调实现.md](WebRTC信令与回调实现.md)。

