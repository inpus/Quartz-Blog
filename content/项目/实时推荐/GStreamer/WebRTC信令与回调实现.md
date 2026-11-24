# WebRTC 信令与回调实现

> 本页说明协商信令、ICE candidate、RTP 元数据回调和服务端接收实现。客户端主流程和 Publisher/GStreamer 管线见 [WebRTC实现.md](WebRTC实现.md)；面向流程的建链图见 [WebRTC.md](WebRTC.md)。

## 按流程查实现

- Offer 创建与信令入队：见 [协商回调](#signaling-callbacks)、[Offer/Answer 步骤索引](#offer-answer-full-chain)。
- Offer SDP 安装到本地 PeerConnection：见 [本地 SDP 设置](#local-sdp-apply)。
- Answer 应用和远端 candidate：见 [处理服务器的信令](#remote-signaling)、[加入远端 ICE candidate](#remote-ice-apply)。
- RTP 发送前添加采集身份：见 [发送端媒体回调](#impl-sender-callbacks)。
- 本地接收回调和服务端收流：见 [本地接收回调](#impl-local-receiver-callbacks)、[服务端媒体接收](#server-media-receive)。

<a id="signaling-callbacks"></a>

## Impl相关回调实现

### 信令与协商回调

<a id="signal-queue-routing"></a>

#### `Impl::outgoing()`：给信令消息补媒体轮次并入队

<code>outgoing(json message)</code> 是 Publisher 内部的信令入队函数。它接收 Offer、ICE 或状态回调组装好的 JSON，补上当前 <code>media_epoch</code>，再放入受互斥锁保护的 <code>signals</code> 队列：

| 函数 | 职责 | 输入/输出 |
|---|---|---|
| <code>outgoing(message)</code> | 生产并入队 | 输入一条信令 JSON；补 <code>media_epoch</code> 后写入 <code>signals</code>。 |
| <code>poll()</code> | 取出并交接 | 从 <code>signals</code> 取出已排队消息；远端模式交给 <code>signal</code> 回调。 |

一句话：<code>outgoing()</code> 把消息放进去，<code>poll()</code> 再把消息取出来交给下一步。两者之间的 <code>signals</code> 是进程内队列；真正的 WebSocket 发送在之后由 <code>media_client</code>/<code>WsUploader</code> 完成。

```c++
void outgoing(json message) {
  message["media_epoch"] = epoch;
  std::lock_guard lock(signal_mutex);
  if (signals.size() >= 128) {
    error("signaling queue overflow");
    return;
  }
  signals.push_back(std::move(message));
}
```

- <code>message</code>：调用方构造的 JSON，例如 <code>webrtc_offer</code> 或 <code>webrtc_ice</code>。函数按值接收，以便补字段后移动进队列。
- <code>epoch</code>：当前 Publisher 的媒体轮次；入队时写入 <code>media_epoch</code>，让服务端能识别消息属于哪一轮媒体会话。
- <code>signal_mutex</code> 和 <code>signals</code>：保护信令队列。GStreamer 回调可以在 GLib 线程中产生消息，主循环则通过 <code>poll()</code> 取消息。
- 队列上限是 128 条；满时调用 <code>error()</code> 标记 Publisher 失败，不会静默丢掉某一条信令。

<code>outgoing()</code> 只负责补标记和入队，**不发送 WebSocket，也不生成 <code>session_id</code>**。在远端运行中，后续由 <code>poll()</code> 取出消息并调用 Publisher 的信令回调；<code>media_client</code> 再补上 <code>session_id</code>，交给 <code>WsUploader::sendControl()</code>。Offer 的 <code>offerReady()</code>、ICE 的 <code>Impl::ice()</code>/<code>gathering()</code> 和媒体状态回调都会复用这个入口。

#### `Impl::poll()`：本地处理 Offer，再交给信令回调

<code>poll()</code> 不是 WebSocket 回调，也不直接操作 WebSocket；它是 Publisher 内部的信令队列处理函数。主循环读取 Publisher 状态时调用 <code>publisher->stats()</code>，而 <code>stats()</code> 会调用 <code>poll()</code>。下面只展开远端真实发送路径；本地 <code>receiver</code> 自测分支省略：

```c++
void poll() {
  std::deque<json> messages;
  {
    std::lock_guard lock(signal_mutex);
    messages.swap(signals);
  }
  for (auto &message : messages) {
    if (message.value("op", "") == "webrtc_offer") {
      offered_tracks = message.at("tracks");
      setRemote(peer, "offer", message.at("sdp"), true);
    }
    if (!receiver) {
      message.erase("local_receiver");
      if (signal)
        signal(std::move(message));
      continue;
    }
    // local_receiver 自测分支
  }
}
```

`messages.swap(signals)` 在持锁时把队列中的消息移到局部批次，随后释放锁；新到的 ICE 消息留待下一轮 `poll()`。处理 `webrtc_offer` 时，`poll()` 先调用 `setRemote()` 完成本地 SDP 设置，随后在远端运行模式下调用 `signal()` 把消息交给发送回调。下面分两个小节，分别讲这两段连续流程。`poll()` 由 <code>WebRtcPublisher::stats()</code> 推进，源码位置见 <code>client/src/media/webrtc_publisher.cpp:616-630</code> 和 <code>:1187-1190</code>。

<a id="local-sdp-apply"></a>

##### 将 Offer SDP 配置到本地 PeerConnection

这里按实际调用顺序，从 `poll()` 取出 `webrtc_offer` 开始，说明 Offer SDP 怎样经过 `setRemote()`、`description()` 和 `peerAction()`，交给本地 `peer`（`webrtcbin`）设置。Promise 负责回报 action 的完成结果；设置成功返回后，`poll()` 才继续把同一条 Offer 信令交给 WebSocket 发送回调。

###### 1. `poll()`：取出 Offer 并调用本地 SDP 设置

`poll()` 从信令队列取出 `webrtc_offer` 后，保存它的轨道映射，再把 SDP 文本交给 `setRemote()`：

```c++
if (message.value("op", "") == "webrtc_offer") {
  offered_tracks = message.at("tracks");
  setRemote(peer, "offer", message.at("sdp"), true);
}
```

`message.at("sdp")` 是 Offer 的 SDP 文本；第四个参数 `true` 表示把这份描述设置到本机 `peer`。尽管辅助函数叫 `setRemote()`，此处实际选择的是 `set-local-description`。下面依次展开它如何解析 SDP、发出 action 并等待结果。

###### 2. `setRemote()`：把 SDP 文本解析并包装成会话描述

这次调用中，四个参数分别表示目标 `webrtcbin`、SDP 类型、JSON 中的 SDP 文本，以及是否把描述设置为本地描述。`setRemote()` 是通用辅助函数：`local=true` 时设置本地描述，默认 `local=false` 时才设置远端描述。

| 参数 | 本次调用传入的值 | 用途 |
|---|---|---|
| `target` | `peer` | 目标 GStreamer `webrtcbin`，即客户端自己的 PeerConnection。 |
| `type` | `"offer"` | SDP 描述类型；本次是 Offer，服务端 Answer 处理时传 `"answer"`。 |
| `text` | `message.at("sdp")` | `offerReady()` 放入信令 JSON 的 SDP 文本。 |
| `local` | `true` | 选择把这份描述设置为 `target` 的本地描述；`false` 时使用远端描述动作。 |

函数先限制文本长度，并把 JSON 中的 SDP 文本解析成 GStreamer 结构：

```c++
GstSDPMessage *sdp = nullptr;
if (gst_sdp_message_new(&sdp) != GST_SDP_OK)
  throw std::runtime_error("SDP allocation failed");

if (gst_sdp_message_parse_buffer(
        reinterpret_cast<const guint8 *>(text.data()),
        static_cast<guint>(text.size()), sdp) != GST_SDP_OK) {
  gst_sdp_message_free(sdp);
  throw std::invalid_argument("invalid SDP");
}
```

- `gst_sdp_message_new(&sdp)`：创建空的 `GstSDPMessage`，供解析器填入 SDP 字段。
- `reinterpret_cast<const guint8 *>(text.data())`：把 C++ 字符串的字符缓冲区按 GStreamer API 要求转换为字节指针。
- `static_cast<guint>(text.size())`：传入缓冲区长度，解析器按这段长度读取 SDP 文本。
- `sdp`：作为输出目标，接收解析后的媒体行、属性等结构化 SDP 内容。
- `gst_sdp_message_parse_buffer(...)`：完成文本到结构化 SDP 的解析；它只解析内容，还没有把 SDP 应用到 `peer`。
- 解析失败时先释放刚创建的 `sdp`，再抛异常，后续 action 不会执行。

然后函数把“SDP 内容”和“Offer/Answer 类型”包装为 WebRTC 会话描述，并选择要执行的动作：

```c++
auto *value = gst_webrtc_session_description_new(
    std::string(type) == "offer" ? GST_WEBRTC_SDP_TYPE_OFFER
                                 : GST_WEBRTC_SDP_TYPE_ANSWER,
    sdp);
description(target,
            local ? "set-local-description" : "set-remote-description",
            value);
gst_webrtc_session_description_free(value);
```

`gst_webrtc_session_description_new()` 让 GStreamer 知道这是 Offer 还是 Answer；这一步仍只是构造参数对象。真正执行设置是在接下来的 `description()` / `peerAction()` 中。当前发送路径的 `type` 是 `offer`、`local` 是 `true`，所以最终动作名为 `set-local-description`。处理服务器 Answer 时则使用 `type="answer"`、默认 `local=false`，动作名是 `set-remote-description`。

###### 3. `description()`：把描述交给动作调用层

`description()` 本身很薄，只把目标端点、动作名和会话描述交给 `peerAction()`，并在操作结束后释放 Promise：

```c++
void description(GstElement *peer, const char *action,
                 GstWebRTCSessionDescription *sdp) {
  auto *promise = peerAction(peer, action, sdp);
  gst_promise_unref(promise);
}
```

这里的 `peer` 是 GStreamer 的 `webrtcbin` 元件，也就是本地 WebRTC PeerConnection 的 GStreamer 接口；`action` 指定要执行的操作；`sdp` 是已解析并包装好的 Offer 或 Answer。客户端发送 Offer 的调用中，参数分别是本机 `self.peer`、`"set-local-description"` 和 Offer 描述。`description()` 不自己解析 SDP，也不通过网络发送 SDP。

###### 4. `peerAction()`：发出 GObject action 并等待 GStreamer 返回结果

`description()` 把目标、动作和会话描述交给 `peerAction()`。`peerAction()` 的关键实现如下：

```c++
GstPromise *peerAction(GstElement *peer, const char *action,
                       gpointer argument) {
  using Completion = std::shared_ptr<std::promise<GstPromiseResult>>;
  auto completion = std::make_shared<std::promise<GstPromiseResult>>();
  auto future = completion->get_future();

  auto *promise = gst_promise_new_with_change_func(
      [](GstPromise *value, gpointer data) {
        (*static_cast<Completion *>(data))->set_value(gst_promise_wait(value));
      },
      new Completion(completion),
      [](gpointer data) { delete static_cast<Completion *>(data); });

  g_signal_emit_by_name(peer, action, argument, promise);

  if (future.wait_for(std::chrono::seconds(10)) != std::future_status::ready) {
    gst_promise_interrupt(promise);
    gst_promise_unref(promise);
    throw std::runtime_error(std::string(action) + " timed out");
  }
  if (future.get() != GST_PROMISE_RESULT_REPLIED) {
    gst_promise_unref(promise);
    throw std::runtime_error(std::string(action) + " interrupted");
  }

  const auto *reply = gst_promise_get_reply(promise);
  if (reply && gst_structure_has_field(reply, "error")) {
    GError *error = nullptr;
    gst_structure_get(reply, "error", G_TYPE_ERROR, &error, nullptr);
    const std::string message =
        error ? error->message : "GStreamer peer action failed";
    if (error)
      g_error_free(error);
    gst_promise_unref(promise);
    throw std::runtime_error(message);
  }
  return promise;
}
```

这几个参数组成一次对本地 `webrtcbin` 的动作调用：

| 参数 | 本次 Offer 设置中的值和作用 |
|---|---|
| `peer` | 本机 `self.peer`，即管线里的 `webrtcbin` 元件。GObject action 会发给这个对象。 |
| `action` | 字符串 `"set-local-description"`。这是 `webrtcbin` 注册的 action 名称。 |
| `argument` | `GstWebRTCSessionDescription *`，封装了 Offer 类型和解析后的 SDP 内容。 |
| `promise` | 传给 action 的完成通知对象。`webrtcbin` 操作完成后，通过它回复成功或错误。 |

这段函数的执行顺序和每一步的作用如下：

1. **先准备 C++ 等待端**：`std::promise` / `std::future` 是本项目用来等待结果的 C++ 工具。`completion` 保存结果，`future` 留在当前调用线程，之后用它等待。
2. **创建 GStreamer Promise，并把结果接到 C++ 等待端**：`gst_promise_new_with_change_func()` 创建初始状态为 `PENDING` 的 GStreamer Promise，同时注册“Promise 状态结束时要执行什么”。这里的回调会在 Promise 离开 `PENDING` 时执行一次；可能的结果包括 `REPLIED`、`INTERRUPTED` 或 `EXPIRED`。

   ```c++
   [](GstPromise *value, gpointer data) {
     (*static_cast<Completion *>(data))->set_value(gst_promise_wait(value));
   }
   ```

   - `value` 是发生状态变化的 `GstPromise` 对象。这里不是本项目额外设置了“先改状态、再回调”的开关，而是 GStreamer API 对调用时机的约定：`gst_promise_new_with_change_func()` 注册的回调，只会在 Promise 从 `PENDING` 转到其他状态时执行；`gst_promise_wait()` 的约定则是等待它离开 `PENDING`，若调用时它已经离开，就立即返回当前状态。因此回调运行到 `gst_promise_wait(value)` 时，Promise 已经有最终状态，这次 wait 会立即返回，不会在回调里继续等 SDP 设置。可分别对照 [change callback 的触发条件](https://gstreamer.freedesktop.org/documentation/gstreamer/gstpromise.html#gst_promise_new_with_change_func) 和 [wait 的返回条件](https://gstreamer.freedesktop.org/documentation/gstreamer/gstpromise.html#gst_promise_wait)。
   - `data` 是传给回调的用户数据。`new Completion(completion)` 在堆上保存一份 `shared_ptr` 副本，让异步回调期间 C++ `std::promise` 的共享状态仍然有效。回调通过它的 `set_value(...)` 把 GStreamer 的结果交给 C++ `future`，唤醒正在 `future.wait_for(10秒)` 的 `peerAction()`。
   - 第三个参数 `[](gpointer data) { delete static_cast<Completion *>(data); }` 是清理函数。GStreamer 不再需要这份用户数据时会调用它，释放前面 `new` 出来的 `Completion`。

   因此，这一步不是执行 `set-local-description`；它是在执行 action 之前先搭好“GStreamer Promise → 回调 → C++ future”的结果通知通路。真正发出 action 的是下一步 `g_signal_emit_by_name(...)`，而 `future.wait_for(...)` 在后面等待这条通路送回结果。GStreamer 对回调触发时机和清理参数的定义见 [GstPromise API](https://gstreamer.freedesktop.org/documentation/gstreamer/gstpromise.html#gst_promise_new_with_change_func)。
3. **在这一行立即调用 action**：`g_signal_emit_by_name(peer, action, argument, promise)` 按 `action` 字符串，在 `peer` 对象上发出 GObject action。代入当前实参，它等价于：

   ```c++
   g_signal_emit_by_name(peer, "set-local-description",
                         offer_description, promise);
   ```

   这不是 C++ 直接调用一个名为 `set_local_description()` 的项目函数，而是让 GStreamer 的 `webrtcbin` 处理它注册的 `set-local-description` action。**action 是在执行这行时立即发出的**；传进去的 `promise` 此时是待处理的完成通知对象，不是调用 action 的前置条件。`webrtcbin` 收到 action 后执行/安排本地描述设置；操作结束后，它才把结果写入 Promise，状态变化回调再唤醒 C++ `future`。顺序是“发出 action → GStreamer 处理 → Promise 状态变化 → future 收到结果”，不是“先等 Promise 变化 → 再调用 action”。这也不是网络发送。
4. **最多等待 10 秒**：`future.wait_for(10秒)` 会阻塞当前调用线程，直到前面注册的 change callback 把 GstPromise 的结果写进 C++ `future`，或者 10 秒到期。它等的是这个 `future`，不是应用代码循环检查 GstPromise；Promise 状态变化时，回调会通知 `future`，等待线程据此醒来。这里等待的是 `webrtcbin` 对本地描述设置 action 的回复；不等待 ICE 候选全部收集完、不等待服务端 Answer，也不等待 WebSocket 写入。ICE 收集由 `on-ice-candidate` / `gathering` 回调另行通知。

   **为什么异步操作里还有等待？** `webrtcbin` 的 action 会异步完成，结果稍后通过 GstPromise 通知；但本项目的 `peerAction()` 把这段异步操作包装成了“最多等 10 秒、完成后再返回”的同步调用。因此，GStreamer 内部可以异步处理，调用 `peerAction()` 的 C++ 线程仍会在这里阻塞。用回调通知而不是忙轮询，是因为状态变化时可以直接唤醒等待者，不必反复读取状态；用 C++ `future.wait_for()` 则可以给等待设置 10 秒上限。若要让 `peerAction()` 本身完全异步，就不能在这里等待，而要把成功、错误和后续流程都改为完成回调中的处理。GStreamer 的 `gst_promise_wait()` 本身没有超时参数，见 [GstPromise API](https://gstreamer.freedesktop.org/documentation/gstreamer/gstpromise.html#gst_promise_wait)。
5. **超时处理并检查结果**：如果 10 秒内没有收到回调结果，`wait_for()` 返回的状态不是 `ready`，代码就调用 `gst_promise_interrupt(promise)`，把 Promise 标记为 `INTERRUPTED`，告知 GStreamer 这一侧已不再等待该结果；随后释放本调用方持有的引用并抛出超时异常。收到结果时，状态不是 `REPLIED` 也会抛出异常。注意，`REPLIED` 只表示 GStreamer 已经回复，不单独等于成功，所以代码还会检查回复结构：如果含有 `error`，就提取 `GError` 文本并抛出。只有状态为 `REPLIED` 且回复中没有 `error` 时，`peerAction()` 才走到 `return promise;`。这里返回的是它之前创建并传给 action 的同一个 `GstPromise *`，不是新的 Promise，也不是 SDP。

`description()` 收到这个指针后调用 `gst_promise_unref()`，意思是释放本调用方持有的 Promise 引用，避免引用计数泄漏；它不是“把 Promise 设为成功”，也不是释放 SDP。之后回到 `setRemote()`，才调用 `gst_webrtc_session_description_free(value)` 释放会话描述。这个应用层调用链只负责把 SDP 交给 GStreamer 并确认“设置动作完成”；真正应用 SDP、更新 PeerConnection 状态和驱动 ICE 的实现属于 GStreamer `webrtcbin`。API 定义见 [webrtcbin 文档](https://gstreamer.freedesktop.org/documentation/webrtc/)，Promise 语义见 [GstPromise 文档](https://gstreamer.freedesktop.org/documentation/gstreamer/gstpromise.html)。本项目源码见 <code>client/src/media/webrtc_publisher.cpp:31-69</code>、<code>:559-605</code>。

###### 5. GStreamer 内部实际调用哪个函数

`g_signal_emit_by_name()` 是通用的 GObject 分发函数。它根据 `peer` 对象的类型和传入的 action 名称，找到该对象注册的处理器。`webrtcbin` 在类初始化时把 `set-local-description` 这个 action signal 绑定到了内部函数 `gst_webrtc_bin_set_local_description()`：

```c
gst_webrtc_bin_signals[SET_LOCAL_DESCRIPTION_SIGNAL] =
    g_signal_new_class_handler(
        "set-local-description",
        G_TYPE_FROM_CLASS(klass),
        G_SIGNAL_RUN_LAST | G_SIGNAL_ACTION,
        G_CALLBACK(gst_webrtc_bin_set_local_description),
        NULL, NULL, NULL,
        G_TYPE_NONE, 2,
        GST_TYPE_WEBRTC_SESSION_DESCRIPTION,
        GST_TYPE_PROMISE);
```

因此，这次调用实际沿着下面的关系进入 GStreamer：

```text
项目调用 g_signal_emit_by_name(peer, "set-local-description", desc, promise)
    ↓ GObject 根据 peer 类型和 action 名分发
gst_webrtc_bin_set_local_description(webrtc, local_sdp, promise)
    ↓ 校验并复制描述，标记来源为 SDP_LOCAL
enqueue _set_description_task(..., promise)
    ↓
_set_description_task() 处理 SDP、更新协商状态并写入 ICE credentials
    ↓ 调用 gst_webrtc_ice_gather_candidates()
ICE 模块异步收集候选，webrtcbin 发出 on-ice-candidate
    ↓ task 返回后由执行器回复 Promise
```

`gst_webrtc_bin_set_local_description()` 是 GStreamer 插件里的内部静态处理函数，不是项目源码直接调用的公开 C 函数。它会检查描述是否合法，把描述复制到内部任务数据中，并将来源标为 `SDP_LOCAL`，再把 `_set_description_task()` 和 Promise 一起交给 `gst_webrtc_bin_enqueue_task()`。任务处理 Offer 时保存 pending local description、更新 signaling state，并从 SDP 读取 ICE credentials 写入对应 stream；随后遍历 ICE stream，实际调用 `gst_webrtc_ice_gather_candidates()` 启动候选收集。该调用发起异步收集；任务返回后，`webrtcbin` 的任务执行器才回复 Promise，所以 Promise 不代表候选已经全部产生。此处正是从“应用本地 SDP”转入“ICE 开始收集候选”的触发链。源码对应 [action 注册](https://github.com/GStreamer/gstreamer/blob/main/subprojects/gst-plugins-bad/ext/webrtc/gstwebrtcbin.c#L9449-L9453)、[内部 action handler](https://github.com/GStreamer/gstreamer/blob/main/subprojects/gst-plugins-bad/ext/webrtc/gstwebrtcbin.c#L7223-L7251)、[`_set_description_task()`](https://github.com/GStreamer/gstreamer/blob/main/subprojects/gst-plugins-bad/ext/webrtc/gstwebrtcbin.c#L6757-L7173)、[候选收集调用](https://github.com/GStreamer/gstreamer/blob/main/subprojects/gst-plugins-bad/ext/webrtc/gstwebrtcbin.c#L7094-L7101) 和[任务执行器回复 Promise](https://github.com/GStreamer/gstreamer/blob/main/subprojects/gst-plugins-bad/ext/webrtc/gstwebrtcbin.c#L1246-L1284)。

收集过程中产生的本地 candidate 由 `webrtcbin` 的 `on-ice-candidate` 通知应用；收集完成则由 `notify::ice-gathering-state` 报告，`Impl::gathering()` 再发送 end-of-candidates 标记。

##### `signal`：将 Offer SDP 发送到服务端

本地 SDP 设置成功返回后，`poll()` 继续处理同一条 `webrtc_offer`。远端发送分支的关键代码是：

```c++
if (!receiver) {
  message.erase("local_receiver");
  if (signal)
    signal(std::move(message));
  continue;
}
```

- `receiver` 为空表示当前走远端发送路径。`local_receiver` 只用于本地分支分流，发送前从 JSON 删除。`signal(std::move(message))` 把 Offer 消息交给 `media_client` 在 `publisher->start()` 时传入的回调；此处仍是进程内函数调用，没有直接写 WebSocket。

`media_client` 注册的回调负责补充会话标识并交给控制发送器：

```c++
[&, media_session = current_session_id](json message) {
  if (!local_receiver) {
    message["session_id"] = media_session;
    controls.sendControl(std::move(message));
  }
}
```

1. `json message`：接收 `poll()` 移交的 `webrtc_offer` JSON，其中包含 `op`、SDP 文本、轨道映射等字段。
2. `media_session = current_session_id`：创建回调时捕获当前会话 ID；回调运行时将它写入消息的 `session_id`，供服务端关联会话。
3. `controls.sendControl(std::move(message))`：把补好 `session_id` 的信令消息交给控制发送器；之后 `WsUploader` 负责将控制消息写入 WebSocket。

因此，这段信令路径依次是：`poll()` 调用 `signal` 回调 → 回调补 `session_id` → `sendControl()` 交接消息 → `WsUploader` 写 WebSocket。`set-local-description` 已在前一小节把 Offer 应用到本地 PeerConnection；这条 `signal` 路径才把同一份 Offer SDP 作为信令传给服务端。双方通过信令交换 Offer、Answer 和 ICE 候选并建立连接后，音视频 RTP 才由 `webrtcbin` 的 WebRTC 媒体传输通道发送。源码位置见 <code>client/src/media/webrtc_publisher.cpp:616-630</code>、<code>client/src/media/media_client.cpp:104-115</code>；GStreamer 的 action 说明见 [webrtcbin 文档](https://gstreamer.freedesktop.org/documentation/webrtc/)。

#### Impl::ice：整理 ICE 候选信息

##### 回调代码

~~~c++
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

##### 在哪里注册、何时触发

发布端在 <code>WebRtcPublisher::start()</code> 中把 <code>Impl::ice</code> 注册到发送端 <code>self.peer</code> 的 <code>on-ice-candidate</code> 信号上：

~~~c++
g_signal_connect(self.peer, "on-ice-candidate",
                 G_CALLBACK(Impl::ice), &self);
~~~

启用本地接收器的自测模式时，也会把同一个回调注册到 <code>self.receiver</code> 上。注册时传入的 <code>&self</code> 会在回调触发时作为 <code>data</code> 返回。<code>g_signal_connect()</code> 只建立连接，不会当场调用 <code>ice()</code>；之后 GStreamer 每产生本地 ICE candidate 并发出 <code>on-ice-candidate</code> 信号，才调用这个回调。注册位置见 <code>client/src/media/webrtc_publisher.cpp:771</code> 和 <code>:882</code>。

##### 回调收到什么

| 参数 | 含义 |
|---|---|
| <code>peer</code> | 发出信号的 <code>webrtcbin</code>。它可能是发送端 <code>self.peer</code>；本地接收器自测时也可能是 <code>self.receiver</code>。 |
| <code>line</code> | 该 candidate 对应的 SDP media line 序号，写入 <code>sdp_mline_index</code>，供对端把 candidate 加到对应媒体行。 |
| <code>candidate</code> | GStreamer 收集到的候选地址信息字符串；为空或空指针时表示候选收集结束标记。 |
| <code>data</code> | 注册时传入的用户数据，本项目传的是 <code>&self</code>；回调将它还原为当前的 <code>Impl</code>。 |

##### 为什么这是本地候选

这里的“本地”是相对于发出 `on-ice-candidate` 信号的 `webrtcbin` 来说的：回调参数 `peer` 指明了是哪一个 WebRTC 端点收集到了候选，所以该候选属于这个端点一侧，不是从远端 SDP 中读出来的远端候选。

这不是通过 WebSocket 把 Offer 发给自己。客户端的 `peer` 先创建 Offer；`Impl::poll()` 从 Publisher 的本地 `signals` 队列取出这条 `webrtc_offer`，调用 `setRemote(peer, "offer", sdp, true)`，在同一个本机 `peer` 上执行 GStreamer 的 `set-local-description`。这个辅助函数名叫 `setRemote`，但 `local=true` 会让它设置本地 SDP，所以这里的名字容易造成误解。

本地描述设置完成后，如果运行在远端模式（没有本地 `receiver`），`poll()` 才把这条 Offer 交给信令回调；之后 `media_client` 添加 `session_id`，通过 WebSocket 发给服务端。与此同时，`peer` 的 ICE agent 按这份本地 SDP 的媒体行和 ICE 参数收集客户端一侧的候选，并通过 `on-ice-candidate` 通知应用。对应代码见 <code>client/src/media/webrtc_publisher.cpp:603-604</code> 和 <code>:623-630</code>。

因此，`line` 用来标明候选对应哪一条 SDP 媒体行。候选地址由 ICE agent 收集，不是这段回调从 SDP 文本中解析或生成的。远端候选则通过信令到达后，由 `Impl::iceRemote()` 调用 `add-ice-candidate` 交给对应的 `webrtcbin`。

当前客户端创建 `webrtcbin` 时只设置了 `bundle-policy` 和 `latency`，没有配置 `stun-server` 或 `turn-server`；因此常见的是本机网卡地址对应的 `host` 候选。若配置了 STUN 或 TURN，回调拿到的仍是本端候选，但类型也可能是映射地址 `srflx` 或中继地址 `relay`。

##### `candidate` 字符串是什么

`candidate` 不是完整 SDP，而是 ICE 的 candidate 属性值。一个示意字符串如下（实际地址、端口和数字由运行环境决定）：

```text
candidate:842163049 1 udp 1677729535 192.168.1.20 54400 typ host
```

| 片段 | 含义 |
|---|---|
| `candidate:` | 表示后面是 ICE candidate 属性值。完整 SDP 中通常写成 `a=candidate:...`；这里通过回调单独传递时不带 `a=`。 |
| `842163049` | foundation，用于标识可归为同一类的候选。 |
| `1` | component ID；RTP 媒体通常为 `1`。 |
| `udp` | 候选使用的传输协议。 |
| `1677729535` | 候选优先级，ICE 用它安排连通性检查顺序。 |
| `192.168.1.20 54400` | 候选地址和端口。 |
| `typ host` | 候选类型；`host` 表示本机接口地址。还可能看到 `srflx`、`prflx` 或 `relay`。 |

映射或中继候选后面还可能带 `raddr`、`rport` 等相关地址字段。该格式承载的是 ICE 连通性检查所需的传输地址和候选属性，具体字段格式见 [RFC 8839 的 candidate 属性定义](https://www.rfc-editor.org/rfc/rfc8839.html#section-5.1)。

##### 回调组出什么、起什么作用

回调把收到的参数整理为 <code>webrtc_ice</code> JSON 消息：

| 字段 | 来源与作用 |
|---|---|
| <code>op</code> | 固定为 <code>webrtc_ice</code>，标明这是一条 ICE 信令。 |
| <code>sdp_mline_index</code> | 取自 <code>line</code>，指出 candidate 属于哪条 SDP 媒体行。 |
| <code>candidate</code> | 候选字符串；空指针会转换为空字符串，避免把 GStreamer 指针传出回调。 |
| <code>end_of_candidates</code> | candidate 为空或空指针时设为 <code>true</code>，表示这是候选结束标记。 |
| <code>local_receiver</code> | 由 <code>peer == self.receiver</code> 得出，标明 candidate 来自本地自测的接收端。 |

随后 <code>self.outgoing()</code> 给消息补上当前 <code>media_epoch</code>，放入 <code>signals</code> 队列；队列最多容纳 128 条，超限会报告信令队列错误。**因此，<code>Impl::ice</code> 的工作是把本地 candidate 转成带会话标记的信令并入队，不是在回调里直接发 WebSocket。** <code>session_id</code> 也不是在此处添加：远端运行时由 <code>media_client</code> 的信令回调补上后发送。外部发送路径见 [媒体客户端主流程](WebRTC实现.md#client-lifecycle)。

##### 入队之后的路由

<code>poll()</code> 取出队列消息后有两种处理方式：

- **远端运行**：没有本地接收器时，<code>poll()</code> 删除内部路由字段 <code>local_receiver</code>，再调用信令回调；<code>media_client</code> 添加 <code>session_id</code> 并经 WebSocket 发送。
- **本地接收器自测**：两端都在本进程时，<code>local_receiver</code> 用来选择对端的 <code>webrtcbin</code>。来自 <code>self.receiver</code> 的 candidate 加到发送端 <code>peer</code>；来自发送端 <code>peer</code> 的 candidate 加到 <code>receiver</code>。随后由 <code>iceRemote()</code> 校验媒体行编号和 candidate 长度，再调用 GStreamer 的 <code>add-ice-candidate</code>。

真正到达远端服务端的客户端 candidate，则由服务端 ICE 信令处理链应用，见 [服务端 ICE 信令处理实现](#server-ice)。客户端收到服务端 candidate 后的缓存和应用见 [处理服务器的信令](#remote-signaling)。

candidate 为空时，<code>Impl::ice</code> 会发出结束标记；下节的 <code>gathering</code> 回调也会在 GStreamer 报告收集完成时把结束标记放入同一信令队列。后者说明的是收集状态完成，两者都会经过 <code>outgoing()</code> 和 <code>poll()</code>。

#### `gathering` 回调

```c++
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
```

##### 注册位置、触发时机和参数

<code>WebRtcPublisher::start()</code> 分别把 <code>Impl::gathering</code> 注册到发送端 <code>self.peer</code> 和自测模式的接收端 <code>self.receiver</code> 的 <code>notify::ice-gathering-state</code> 属性通知上。注册位置见 <code>client/src/media/webrtc_publisher.cpp:1017-1019</code> 和 <code>:906-908</code>。当 GStreamer 改变 ICE 收集状态时，会调用这个回调；注册本身不会立即触发它。

- <code>object</code>：状态发生变化的 <code>webrtcbin</code>。
- <code>pspec</code>：发生变化的属性说明，本回调未使用。
- <code>data</code>：注册时传入的 <code>&self</code>，在回调中还原为 <code>Impl</code>。

回调读取当前 <code>ice-gathering-state</code>。只有状态变为 <code>COMPLETE</code> 时才调用 <code>outgoing()</code>，放入一条 end-of-candidates 信令：<code>candidate</code> 为空、<code>end_of_candidates</code> 为 <code>true</code>，并根据发出通知的是不是 <code>self.receiver</code> 设置内部路由字段 <code>local_receiver</code>。消息之后仍由 <code>poll()</code> 路由。

> [!NOTE]
>
> ICE candidate 收集完成后，发送 end-of-candidates 标记，表示后续没有更多候选。

#### Impl::negotiate：请求本地 webrtcbin 异步创建 Offer

##### 回调代码

~~~c++
static void negotiate(GstElement *peer, gpointer data) {
  auto &self = *static_cast<Impl *>(data);
  if (self.offered.exchange(true))
    return;
  auto *promise =
      gst_promise_new_with_change_func(offerReady, &self, nullptr);
  g_signal_emit_by_name(peer, "create-offer", nullptr, promise);
}
~~~

##### 注册位置、触发时机和参数

<code>WebRtcPublisher::start()</code> 把回调注册到发送端 <code>self.peer</code> 的 <code>on-negotiation-needed</code> 信号上；管线进入 <code>PLAYING</code> 后，<code>start()</code> 还会直接调用一次同一个回调。两条入口都经过 <code>self.offered.exchange(true)</code>，因此只有第一次会继续创建 Offer。注册和直接调用位置见 <code>client/src/media/webrtc_publisher.cpp:874-875</code> 与 <code>:908</code>。

<code>peer</code> 是发出协商信号的发送端 <code>webrtcbin</code>；<code>data</code> 是注册或直接调用时传入的 <code>&self</code>。异步创建完成后，GStreamer 会调用 <code>offerReady</code>，并将这个 <code>data</code> 继续传给它。

##### 回调作用

该回调通过 Promise 请求 <code>peer</code> 异步创建 Offer。结果准备好后，GStreamer 调用 <code>offerReady</code> 处理；协商入口由 <code>offered</code> 保证只发起一次。

##### Promise 参数与创建请求

这一行建立 Promise，并指定异步结果完成后要运行的函数：

~~~c++
auto *promise =
    gst_promise_new_with_change_func(offerReady, &self, nullptr);
~~~

- <code>offerReady</code>：Offer 创建结果准备好之后运行的回调。
- <code>&self</code>：传给回调的附加数据。它指向现有的 <code>Impl</code>，回调据此访问 Publisher 状态；这里只传递指针，不复制或接管对象。
- <code>nullptr</code>：没有额外数据清理回调。这里没有为 <code>data</code> 单独分配内存需要清理。

这里传入 <code>GstPromise</code>，是因为 <code>create-offer</code> action 通过这个参数交还异步生成的 Offer；Offer 不是从 <code>g_signal_emit_by_name()</code> 的返回值取得。<code>offerReady</code> 已经注册为这个 Promise 的 change callback，所以 GStreamer 完成创建并更新 Promise 后，会调用它处理结果。想在此回调里接收创建结果，就需要把带有该回调的 Promise 传给 action。C++ <code>future</code> 是另一层可选的等待机制；当前 <code>negotiate()</code> 直接使用 GStreamer 回调，不需要它。[<code>create-offer</code> action 参数](https://gstreamer.freedesktop.org/documentation/webrtc/#create-offer) · [GstPromise 回调机制](https://gstreamer.freedesktop.org/documentation/gstreamer/gstpromise.html#gst_promise_new_with_change_func)

Promise 创建好后，下面这行向 <code>peer</code> 发出创建请求：

~~~c++
g_signal_emit_by_name(peer, "create-offer", nullptr, promise);
~~~

<code>peer</code> 是发送端 <code>webrtcbin</code>；信号名 <code>create-offer</code> 表示请求创建 Offer；第二个 <code>nullptr</code> 使用默认创建选项；<code>promise</code> 接收异步结果。Promise 只负责登记完成回调和承接结果，真正的创建请求由这行信号发出。完成后的 SDP 处理见 [Impl::offerReady](#impl-offer-ready)。
##### 为什么这里不使用 C++ `future` 等待

`negotiate()` 的职责是发出创建请求，Offer 准备好之后由 `offerReady()` 接着处理：读取结果、序列化 SDP、组成 `webrtc_offer` 并放入 `signals` 队列。调用 `negotiate()` 的代码不需要在当前线程拿到 Offer 后才能继续，所以这里采用 Promise 回调作为异步后续流程，不需要再用 C++ `future` 阻塞等待。

这和前面的 `set-local-description` 用法不同：`poll()` 要先等本地 `peer` 应用 Offer 成功，才能继续把同一条 Offer 信令交给 `signal` 回调；因此 `peerAction()` 用 change callback 把结果交给 C++ `future`，再用 `wait_for()` 限时等待。是否用 `future` 取决于调用方是否需要在当前调用中等待结果，不是 Promise 本身改变了用途。

当前 `offerReady()` 也没有通过 `future.get()` 或显式比较 `GST_PROMISE_RESULT_REPLIED` 来判断成功。它从 Promise 回复中尝试取出 `"offer"` 字段：取到非空 Offer 就继续打包入队；没有取到就调用 `self.error("GStreamer did not create an offer")`。所以这段实现是按结果中是否有 Offer 来继续或报错，失败信息是通用错误。

##### 总流程

~~~
negotiate
→ 请求 webrtcbin 创建 offer
→ 创建完成后调用 offerReady
→ offerReady 把 SDP 整理成 webrtc_offer 信令消息
→ 消息进入 signals 队列
~~~

<a id="impl-offer-ready"></a>

#### Impl::offerReady：整理本地 Offer 并交给信令路由

##### 回调代码

~~~c++
static void offerReady(GstPromise *promise, gpointer data) {
  auto &self = *static_cast<Impl *>(data);
  const auto *reply = gst_promise_get_reply(promise);
  GstWebRTCSessionDescription *offer = nullptr;
  if (reply)
    gst_structure_get(reply, "offer", GST_TYPE_WEBRTC_SESSION_DESCRIPTION,
                      &offer, nullptr);
  if (offer) {
    gchar *text = gst_sdp_message_as_text(offer->sdp);
    json mapping = json::array();
    for (guint i = 0; i < gst_sdp_message_medias_len(offer->sdp); ++i) {
      const auto *media = gst_sdp_message_get_media(offer->sdp, i);
      const char *mid = gst_sdp_media_get_attribute_val(media, "mid");
      mapping.push_back({{"mid", mid ? mid : ""},
                         {"stream_id", i == 0   ? "game_video"
                                       : i == 1 ? "cs2"
                                                : "oopz"}});
    }
    self.outgoing({{"op", "webrtc_offer"},
                   {"type", "offer"},
                   {"sdp", text},
                   {"tracks", mapping}});
    g_free(text);
    gst_webrtc_session_description_free(offer);
  } else
    self.error("GStreamer did not create an offer");
  gst_promise_unref(promise);
}
~~~

##### 注册、触发与输入

<code>offerReady</code> 的定义见 <code>client/src/media/webrtc_publisher.cpp:164</code>。<code>negotiate()</code> 在 <code>client/src/media/webrtc_publisher.cpp:198</code> 将它传给 <code>gst_promise_new_with_change_func()</code>；<code>create-offer</code> 的 Promise 返回结果时，GStreamer 调用这个回调。<code>promise</code> 携带异步结果，<code>data</code> 是传入的 <code>&self</code>。回调从 Promise 取出 SDP Offer，建立 MID 到项目流名的映射，调用 <code>outgoing()</code> 入队，并释放 Promise 和临时 SDP 对象。队列消息随后由 <code>poll()</code> 按运行模式路由：远端运行时发往服务端 WebRTC 对端；本地接收自测时交给本进程的 <code>receiver</code>。
##### 从 Promise 读取 offer

```c++
const auto *reply = gst_promise_get_reply(promise);
```

从 `promise` 中取出异步操作返回的结果结构，存入 `reply`。这个 `reply` 是 `GstStructure`，不是状态码；它可能为空。这里 `const` 表示回调只读取它，不修改它。

```c++
GstWebRTCSessionDescription *offer = nullptr;
```

先声明一个 offer 指针，并初始化为空。稍后尝试从 `reply` 里取出 offer；如果没取到，它就保持为空。

```c++
if (reply)
  gst_structure_get(reply, "offer", GST_TYPE_WEBRTC_SESSION_DESCRIPTION,
                    &offer, nullptr);
```

如果有结果结构，就从里面读取名为 `"offer"` 的字段：

- `reply`：从哪个结果结构中读取。
- `"offer"`：要读取的字段名。
- `GST_TYPE_WEBRTC_SESSION_DESCRIPTION`：期望这个字段的类型是 WebRTC 会话描述。
- `&offer`：把取出的值写入 `offer`。这里传的是 `offer` 指针的地址，因此函数可以修改它。
- `nullptr`：结束可变参数列表，表示字段参数到此为止。

如果读取成功，`offer` 就指向创建好的 WebRTC 会话描述；接下来的 `if (offer)` 会检查这一点，然后把它的 SDP 转成文本。如果结果为空或提取失败，`offer` 仍是空指针，函数就会报告创建 offer 失败。

> [!NOTE]
>
> 这里的成功判断是能否从回复中取到 `offer`，不是像 `peerAction()` 那样显式读取并比较 `GstPromiseResult`。GStreamer 文档规定，change callback 可在 Promise 离开 `PENDING` 进入其他终态时触发，而 `gst_promise_get_reply()` 要求 Promise 处于 `REPLIED`。当前 `offerReady()` 没有区分 `REPLIED`、`INTERRUPTED` 或 `EXPIRED`，而是按正常收到回复的路径读取 `offer`；取不到就报告通用错误。若要分别处理非 `REPLIED` 状态或读取回复里的详细错误，需要在回调中先检查 Promise 状态，再读取回复。[回调触发条件](https://gstreamer.freedesktop.org/documentation/gstreamer/gstpromise.html#gst_promise_new_with_change_func) · [`get_reply()` 的状态要求](https://gstreamer.freedesktop.org/documentation/gstreamer/gstpromise.html#gst_promise_get_reply)

##### 把创建好的 SDP offer 转成文本、建立媒体轨道映射，再组成信令消息

##### 1.把 SDP 结构转成文本

```
gchar *text = gst_sdp_message_as_text(offer->sdp);
```

`offer->sdp` 是 GStreamer 内部表示的 SDP。这个函数把它转换成文本形式，方便放入 JSON 信令消息中。返回的 `text` 是新分配的内存，后面要用 `g_free` 释放。

> [!NOTE]
>
> **Offer 的生成、打包与设置是三个不同步骤：**
>
> 1. <code>create-offer</code> 根据本地 <code>peer</code> 已建立的 WebRTC 轨道与能力生成 SDP。<code>offerReady()</code> 里的 <code>gst_sdp_message_as_text(offer-&gt;sdp)</code> 只是把结构化 SDP 序列化成新分配的文本；遍历媒体行、读取 <code>mid</code> 也只是读取操作。它没有修改 SDP，也没有把 SDP 设置到 <code>peer</code>。
> 2. <code>mapping</code> 是项目另建的 JSON 数组，用于把 SDP 的 <code>mid</code> 对应到 <code>game_video</code>、<code>cs2</code>、<code>oopz</code>，不是 SDP 配置，也不是网络端点。
> 3. <code>offerReady()</code> 把 SDP 文本和 <code>mapping</code> 组成 <code>webrtc_offer</code>，交给 <code>outgoing()</code> 入队。之后 <code>poll()</code> 才在本机 <code>peer</code> 上调用 <code>set-local-description</code>，让本地 WebRTC 会话采用这份 Offer 并启动/推进 ICE 候选收集；再调用信令回调，经 WebSocket 发给服务端。因此，这不是把 Offer 发给自己，而是先在本地应用，再发送给远端。

##### 2. 创建一个空的映射数组

```c++
json mapping = json::array();
```

创建一个空 JSON 数组，用来记录 SDP 中每条媒体行对应项目里的哪路媒体。

##### 3. 遍历 offer 里的媒体行

```
for (guint i = 0; i < gst_sdp_message_medias_len(offer->sdp); ++i) {
```

`gst_sdp_message_medias_len(...)` 返回 SDP 里媒体行的数量。循环按顺序逐条处理它们。

> [!NOTE]
>
> 这个项目的 offer 预期有 **3 条媒体行**，分别对应发送到 `peer.sink_0`、`peer.sink_1`、`peer.sink_2` 的视频和两路音频。

```c++
const auto *media = gst_sdp_message_get_media(offer->sdp, i);
```

根据下标 `i` 取出当前媒体行。

```
const char *mid = gst_sdp_media_get_attribute_val(media, "mid");
```

从当前媒体行读取 `mid` 属性。`mid` 是 SDP 用来标识媒体行的 ID。

> [!NOTE]
>
> **GStreamer 生成 `mid`**：创建 SDP offer 时，它会给媒体行分配 MID；
>
> 然后下面代码再自行映射对应的流名字

```
mapping.push_back({{"mid", mid ? mid : ""},
                   {"stream_id", ...}});
```

向数组添加一条映射：

- `"mid"`：当前 SDP 媒体行的 ID；如果没有取到，就用空字符串。

  > [!NOTE]
  >
  > `mid` 是 SDP 中**某一条媒体行的标识符**。每个 SDP 媒体行，也就是每个 `m=` 区块，可以带有类似 `a=mid:0` 的属性。

- `"stream_id"`：按媒体行顺序映射到项目中的流名：第 0 条是 `game_video`，第 1 条是 `cs2`，之后是 `oopz`。

因此，预期生成类似这样的内容：

```
[
  {"mid": "0", "stream_id": "game_video"},
  {"mid": "1", "stream_id": "cs2"},
  {"mid": "2", "stream_id": "oopz"}
]
```

##### 4. 把 offer 和轨道映射放入信令队列

```
self.outgoing({{"op", "webrtc_offer"},
               {"type", "offer"},
               {"sdp", text},
               {"tracks", mapping}});
```

构造一条 JSON 信令消息，包含消息类型、SDP 文本，以及 SDP 媒体行到项目流名的映射。`self.outgoing` 会补上 `media_epoch` 并将消息放入队列；之后 `poll()` 再把它交给外部信令回调发送，或交给本地接收端处理。

##### 5. 释放临时对象

```
g_free(text);
gst_webrtc_session_description_free(offer);
```

SDP 文本和 offer 描述都由 GStreamer 创建，因此分别用对应的 GStreamer 释放函数清理。放进 `json` 时，文本内容已经被复制进消息，不依赖这个临时 `text` 指针继续存在。

##### 总流程

这段代码把创建好的 SDP Offer 整理成信令消息，并通过 `self.outgoing(...)` 放入本地队列。之后 `poll()` 取出 `webrtc_offer`，先调用 `setRemote(peer, "offer", sdp, true)`；其中 `true` 使辅助函数在客户端本地 `peer` 上执行 `set-local-description`，再由信令回调把消息交给 `media_client` 的发送路径。

流程是：

1. 从 `promise` 的结果里取出 GStreamer 创建的 `offer`。

2. 把 `offer` 中的 SDP 转成文本：

   ```
   gchar *text = gst_sdp_message_as_text(offer->sdp);
   ```

3. 遍历 SDP 里的媒体行，记录每个 `mid` 对应项目里的轨道名：视频对应 `game_video`，两路音频对应 `cs2` 和 `oopz`。

4. 把 SDP 和轨道映射打包成 `webrtc_offer` JSON，交给 `self.outgoing(...)`：

   ```
   create-offer 的结果
   → 转成 SDP 文本
   → 包成 webrtc_offer JSON
   → 放入信令队列
   → 后续交给服务器或本地接收端处理(服务器收到 offer，把它设为远端 SDP，创建 answer，把 answer 设为本地 SDP，再发回客户端。)
   ```

所以这段是在发送**协商用的 SDP offer**，不是发送音视频数据。音视频媒体数据会在 WebRTC 连接建立后走媒体通道。

<a id="offer-answer-full-chain"></a>

#### Offer/Answer 步骤索引：从本地协商到服务端应答

远端模式的逐步代码时序见 [客户端主流程](WebRTC实现.md#client-lifecycle)。本节保留函数索引，避免重复展开相同调用链：

1. <code>WebRtcPublisher::start()</code> 注册并调用 <code>Impl::negotiate()</code>；<code>negotiate()</code> 向本地 <code>webrtcbin</code> 发出 <code>create-offer</code>。
2. <code>offerReady()</code> 读取 Offer、生成 tracks 映射并交给 <code>outgoing()</code>；<code>poll()</code> 入队取出后设置本地 SDP，再调用信令回调。
3. 客户端通过 WebSocket 发出 Offer；服务端 <code>WebRtcReceiver::offer()</code> 应用远端 Offer、创建 Answer 并回送，服务端 ICE 处理见 [服务端 ICE 信令处理实现](#server-ice)。
4. 客户端 <code>WebRtcPublisher::remote()</code> 应用 Answer；单独到达的 <code>webrtc_ice</code> 消息先缓存或直接交给 <code>iceRemote()</code>，后者调用 <code>add-ice-candidate</code>。
5. ICE/DTLS 连通后，客户端主循环才调用 <code>start_sources()</code>，启动真实音视频采集。

具体实现分别见 [协商与 Offer 回调](#signaling-callbacks)、[Offer 本地设置和 WebSocket 发送](#local-sdp-apply)、[远端 Answer/ICE 分派](#remote-signaling) 和 [加入远端 candidate](#remote-ice-apply)。

> [!NOTE]
>
> <code>local_receiver=true</code> 的自测模式走 <code>poll()</code> 的本地分支：Offer 交给本进程的 <code>receiver</code>，由它创建 Answer，再在两个本地 <code>webrtcbin</code> 之间应用 Answer 和 ICE candidate。这条路径不经过服务端信令发送回调。代码见 <code>client/src/media/webrtc_publisher.cpp:579-603</code>。


<a id="impl-sender-callbacks"></a>

### 发送端媒体回调

> 这组回调位于发送管线：编码器输出处记录编码信息，RTP 打包器输出处补上采集身份扩展，之后 buffer 才继续流向 WebRTC 轨道。

<a id="impl-encoded-video"></a>

#### Impl::encodedVideo：关联编码视频与采集身份

##### 注册位置、触发时机和参数

回调注册在 <code>normalized</code> 的输出 pad 上。该 element 位于 H.264 编码器之后、RTP 打包器之前；每个编码后的视频 buffer 到达此处时触发。<code>info</code> 提供输出 buffer，<code>data</code> 是视频轨道的 <code>Track*</code>。注册位置见 <code>client/src/media/webrtc_publisher.cpp:878</code>。

~~~c++
static GstPadProbeReturn encodedVideo(GstPad *, GstPadProbeInfo *info,
                                      gpointer data) {
  auto &track = *static_cast<Track *>(data);
  auto *buffer = GST_PAD_PROBE_INFO_BUFFER(info);
  ++track.encoded_units;
  track.encoded_pts = GST_BUFFER_PTS(buffer);
  auto *stamp = gst_buffer_get_reference_timestamp_meta(buffer, nullptr);
  if (!stamp)
    return GST_PAD_PROBE_OK;
  std::lock_guard lock(track.mutex);
  for (const auto &[pts, identity] : track.identities)
    if (identity.capture_start_us == stamp->timestamp / GST_USECOND) {
      track.identities[GST_BUFFER_PTS(buffer)] = identity;
      break;
    }
  return GST_PAD_PROBE_OK;
}
~~~

##### 回调作用与后续流向

<code>push()</code> 先按输入 PTS 保存采集时间和源单元编号。这里读取编码输出 buffer 上的 reference timestamp，再按采集时间找到对应身份，并把这份身份信息登记到编码后的 PTS 下。这样后面的 RTP 回调可以按编码包时间戳查回源身份，不需要假设“第 N 个编码帧就是第 N 个采集帧”。

回调会增加 <code>encoded_units</code> 并记录最后的 <code>encoded_pts</code>。若 reference timestamp 不存在，它仍返回 <code>GST_PAD_PROBE_OK</code> 让媒体继续流动；身份缺失会在后续 RTP 注入阶段被统计。详细注入过程见 [RTP发送回调与身份扩展](#impl-rtp-inject)。

<a id="impl-encoded-audio"></a>

#### Impl::encodedAudio：记录 Opus 编码输出的预留采样

##### 注册位置、触发时机和参数

<code>start()</code> 在 <code>cs2enc</code> 和 <code>oopzenc</code> 的输出 pad 上注册此 probe；每个编码后音频 buffer 到达时触发。<code>data</code> 是对应音频轨道的 <code>Track*</code>。注册位置见 <code>client/src/media/webrtc_publisher.cpp:832</code>。

~~~c++
static GstPadProbeReturn encodedAudio(GstPad *, GstPadProbeInfo *info,
                                      gpointer data) {
  auto &track = *static_cast<Track *>(data);
  auto *buffer = GST_PAD_PROBE_INFO_BUFFER(info);
  ++track.encoded_units;
  const auto *clipping = gst_buffer_get_audio_clipping_meta(buffer);
  if (clipping && clipping->start) {
    const auto ticks =
        clipping->format == GST_FORMAT_TIME
            ? gst_util_uint64_scale_round(clipping->start, 48000, GST_SECOND)
            : clipping->start;
    track.pending_preskip_ticks = ticks;
    track.codec_preskip_ticks = ticks;
  }
  return GST_PAD_PROBE_OK;
}
~~~

##### 回调作用与后续流向

Opus 编码器可能在开头带有 codec priming。回调从 <code>GstAudioClippingMeta</code> 读取裁剪起点；若格式是时间单位，就换算成 Opus 的 48 kHz RTP 时钟刻度，然后保存到 <code>pending_preskip_ticks</code> 和 <code>codec_preskip_ticks</code>。它不丢弃 buffer，始终返回 <code>GST_PAD_PROBE_OK</code>。RTP 注入时会消费 pending 值并校正首包时间戳，见 [RTP发送回调与身份扩展](#impl-rtp-inject)。

<a id="impl-rtp-inject"></a>

#### Impl::rtp 与 inject：给 RTP 包写入采集身份

##### 注册位置、触发时机和输入

每条轨道的 RTP payloader 输出 pad 都安装了 probe，监听单个 buffer 和 buffer list；到达此 pad 的 RTP 包会触发 <code>Impl::rtp</code>。probe 的 <code>data</code> 是当前轨道的 <code>Track*</code>。注册位置见 <code>client/src/media/webrtc_publisher.cpp:820</code>。

~~~c++
static GstPadProbeReturn rtp(GstPad *, GstPadProbeInfo *info, gpointer data) {
  auto &track = *static_cast<Track *>(data);
  if (!track.owner->connected.load())
    return GST_PAD_PROBE_DROP;
  if (GST_PAD_PROBE_INFO_TYPE(info) & GST_PAD_PROBE_TYPE_BUFFER_LIST) {
    auto *list =
        gst_buffer_list_make_writable(GST_PAD_PROBE_INFO_BUFFER_LIST(info));
    GST_PAD_PROBE_INFO_DATA(info) = list;
    for (guint i = 0; i < gst_buffer_list_length(list); ++i)
      if (!inject(gst_buffer_list_get_writable(list, i), track))
        return GST_PAD_PROBE_DROP;
    return GST_PAD_PROBE_OK;
  }
  auto *buffer = gst_buffer_make_writable(GST_PAD_PROBE_INFO_BUFFER(info));
  GST_PAD_PROBE_INFO_DATA(info) = buffer;
  return inject(buffer, track) ? GST_PAD_PROBE_OK : GST_PAD_PROBE_DROP;
}
~~~

##### rtp 的作用与返回值

WebRTC 未连接时直接返回 <code>GST_PAD_PROBE_DROP</code>。连接后，回调先取得可写 buffer，再调用辅助函数 <code>inject()</code>；buffer list 中每个 RTP 包都要成功注入。成功返回 <code>GST_PAD_PROBE_OK</code>，身份缺失、RTP buffer 无法映射或扩展写入失败则返回 <code>GST_PAD_PROBE_DROP</code>。因此这个回调只在发送前检查和修改 RTP 包，不负责直接发送网络数据。

##### inject 如何找到并写入采集身份

视频通常按 RTP buffer 的 PTS 精确查 <code>track.identities</code>。Opus 编码器会因 lookahead 调整输出 PTS；音频查找不到精确项时，代码会取不晚于当前 PTS 的上一条身份记录，只在偏差不超过 20 ms 时按偏移修正源单元编号和采集时间：

~~~c++
auto found = track.identities.find(GST_BUFFER_PTS(buffer));
if (found == track.identities.end() && track.index != 0) {
  found = track.identities.upper_bound(GST_BUFFER_PTS(buffer));
  if (found != track.identities.begin()) {
    --found;
    const auto delta = GST_BUFFER_PTS(buffer) - found->first;
    if (delta <= 20 * GST_MSECOND) {
      metadata = found->second;
      metadata.source_unit_id +=
          gst_util_uint64_scale_round(delta, 16000, GST_SECOND);
      metadata.capture_start_us += delta / GST_USECOND;
    } else
      found = track.identities.end();
  } else
    found = track.identities.end();
}
~~~

找到身份后，<code>inject()</code> 将其编码为项目定义的 16 字节采集元数据，写入 RTP 扩展 ID 7。音频首包还会取出 <code>pending_preskip_ticks</code> 校正 Opus RTP 时间戳，并检查后续 20 ms 包是否按 960 个 48 kHz 时钟刻度递增。处理成功后更新 RTP 包和字节计数；这些字段供发布报告判断身份缺失、编码流量和时钟连续性。


<a id="impl-local-receiver-callbacks"></a>

### 本地接收自测回调

> 以下回调只在 <code>local_receiver=true</code> 的本地接收验证模式建立第二个 <code>webrtcbin</code> 时使用。远端服务器的接收实现见 [服务端 ICE 信令处理实现](#server-ice)。

<a id="impl-incoming"></a>

#### Impl::incoming：为新到达的 WebRTC pad 建接收支路

##### 注册位置、触发时机和参数

自测模式创建 <code>self.receiver</code> 后，将此回调注册到它的 <code>pad-added</code> 信号。收到远端媒体并出现新的 source pad 时，GStreamer 调用 <code>incoming()</code>；<code>pad</code> 是新 pad，<code>data</code> 是 <code>&self</code>。正常远端 Run 不创建这个本地接收器，因此不走此回调。注册位置见 <code>client/src/media/webrtc_publisher.cpp:775</code>。

回调先检查 pad 方向和 caps，再用媒体类型与 RTP payload 判断轨道。下面是选择接收轨道的关键代码段：

~~~c++
const char *media = gst_structure_get_string(structure, "media");
gint payload = 0;
gst_structure_get_int(structure, "payload", &payload);
const int index = media && std::string(media) == "video" ? 0
                  : payload == 97                        ? 1
                                                         : 2;
~~~

视频轨建立 H.264 depay/parse/decode/appsink 支路；两路音频建立 Opus depay/decode/16 kHz 单声道/appsink 支路。接着回调把新支路加入 receiver pipeline、将 pad 连到支路入口，并同步到父 pipeline 的状态。它在每个远端轨道 pad 到达时完成一次动态接线。

<a id="impl-received-rtp"></a>

#### Impl::receivedRtp：读取并验证 RTP 身份扩展

##### 注册位置、触发时机和参数

<code>incoming()</code> 在 depayloader 的 sink pad 上注册此 buffer probe，注册代码见 <code>client/src/media/webrtc_publisher.cpp:482-487</code>；它看到的是已收到、尚未 depayload 的 RTP 包。<code>info</code> 提供 RTP buffer，<code>data</code> 是对应轨道的 <code>Track*</code>。回调完整代码如下：

~~~c++
static GstPadProbeReturn receivedRtp(GstPad *, GstPadProbeInfo *info,
                                     gpointer data) {
  auto &track = *static_cast<Track *>(data);
  auto *buffer = GST_PAD_PROBE_INFO_BUFFER(info);
  GstRTPBuffer rtp = GST_RTP_BUFFER_INIT;
  if (!gst_rtp_buffer_map(buffer, GST_MAP_READ, &rtp))
    return GST_PAD_PROBE_DROP;
  gpointer bytes = nullptr;
  guint size = 0;
  const auto found = gst_rtp_buffer_get_extension_onebyte_header(
      &rtp, kExtensionId, 0, &bytes, &size);
  const auto metadata =
      found ? tl::media::decodeCaptureMetadata(
                  {static_cast<const uint8_t *>(bytes), size})
            : std::nullopt;
  gst_rtp_buffer_unmap(&rtp);
  if (!metadata) {
    ++track.missing;
    return GST_PAD_PROBE_DROP;
  }
  {
    std::lock_guard lock(track.mutex);
    track.received_identities[GST_BUFFER_PTS(buffer)] = *metadata;
    while (track.received_identities.size() > 256)
      track.received_identities.erase(track.received_identities.begin());
  }
  return GST_PAD_PROBE_OK;
}
~~~

该回调验证 RTP 扩展 ID 7 并解码出采集时间、源单元编号等身份；扩展缺失或内容无效时计入 missing 并丢弃该包。通过验证后按接收端 PTS 暂存身份，最多保留 256 条，供 depayloader 和 decoder 阶段关联使用。

<a id="impl-depayloaded"></a>

#### Impl::depayloaded：把身份信息带到解码 buffer

##### 注册位置、触发时机和参数

<code>incoming()</code> 在 depayloader 的 src pad 上安装此 probe，注册代码见 <code>client/src/media/webrtc_publisher.cpp:488-492</code>。它看到的是已经从 RTP 解出的压缩媒体 buffer；<code>data</code> 是对应轨道。回调先将 buffer 变为可写，再按接收端 PTS 查找 <code>received_identities</code>：

~~~c++
static GstPadProbeReturn depayloaded(GstPad *, GstPadProbeInfo *info,
                                     gpointer data) {
  auto &track = *static_cast<Track *>(data);
  auto *buffer = gst_buffer_make_writable(GST_PAD_PROBE_INFO_BUFFER(info));
  GST_PAD_PROBE_INFO_DATA(info) = buffer;
  std::lock_guard lock(track.mutex);
  auto found = track.received_identities.find(GST_BUFFER_PTS(buffer));
  if (found != track.received_identities.end()) {
    auto *meta = gst_buffer_add_custom_meta(buffer, kIdentityMeta);
    gst_structure_set(gst_custom_meta_get_structure(meta), "source_unit_id",
                      G_TYPE_UINT64, found->second.source_unit_id,
                      "capture_start_us", G_TYPE_UINT64,
                      found->second.capture_start_us, "encoded_pts",
                      G_TYPE_UINT64, GST_BUFFER_PTS(buffer), nullptr);
  }
  return GST_PAD_PROBE_OK;
}
~~~

找到映射时，回调把源单元编号和采集时间写入自定义 GstMeta，随压缩 buffer 一起经过 decoder；没有映射时不伪造身份，仍让媒体继续流动。这个回调只传递元数据，不执行解码。

<a id="impl-decoded"></a>

#### Impl::decoded：统计解码结果并核对身份

##### 注册位置、触发时机和参数

<code>incoming()</code> 将 <code>decoded</code> 注册到 appsink 的 <code>new-sample</code> 信号，注册代码见 <code>client/src/media/webrtc_publisher.cpp:493-496</code>。每有一个已解码 sample 可取时调用；<code>sink</code> 是 appsink，<code>data</code> 是轨道 <code>Track*</code>。回调先拉取 sample、增加 decoded 计数；sample 为空时返回 <code>GST_FLOW_EOS</code>。

音频轨还会统计解码样本数、非零样本和过零次数，用于本地音调与轨道自测。随后回调优先读取 depayload 阶段附加的自定义身份。下面展示这一分支的关键代码；完整回调还包括音频样本统计和后面的 PTS 查找兜底：

~~~c++
auto *capture = gst_buffer_get_custom_meta(buffer, kIdentityMeta);
guint64 capture_us = 0, unit_id = 0, encoded_pts = 0;
if (capture) {
  const auto *structure = gst_custom_meta_get_structure(capture);
  gst_structure_get_uint64(structure, "capture_start_us", &capture_us);
  gst_structure_get_uint64(structure, "source_unit_id", &unit_id);
  gst_structure_get_uint64(structure, "encoded_pts", &encoded_pts);
  const auto delta = track.index && GST_BUFFER_PTS(buffer) >= encoded_pts
                         ? GST_BUFFER_PTS(buffer) - encoded_pts
                         : 0;
  ++track.verified;
  track.last_id =
      unit_id + gst_util_uint64_scale_round(delta, 16000, GST_SECOND);
  track.last_capture = capture_us + delta / GST_USECOND;
}
~~~

如果自定义 GstMeta 没有到达解码输出，回调再用 RTP 阶段保存的 <code>received_identities</code> 按 PTS 查找；音频允许在 20 ms 内用相邻时间偏移修正源单元编号和采集时间。两种方式都无法关联时增加 missing 并记录告警。最后释放 sample 并返回 <code>GST_FLOW_OK</code>。结果进入 Publisher 的本地接收统计，用来验证本机回环是否解码、身份是否保留；它不参与远端生产服务的处理。

<a id="remote-signaling"></a>

## 处理服务器的信令 media_client.cpp:209（源码位置：<code>client/src/media/media_client.cpp</code>）

```c++
else if (publisher) publisher->remote(message);
```

WebSocket 收到的 Answer 和 ICE JSON 先由 <code>post()</code> 放入客户端 <code>incoming</code> 队列；主循环取出消息后才调用 <code>publisher->remote(message)</code>。因此这个函数处理的是 WebRTC 控制信令，不负责接收 RTP 媒体包。远端运行时它主要分派两类消息：<code>webrtc_answer</code> 和 <code>webrtc_ice</code>。

### 处理过程 webrtc_publisher.cpp:1089（源码位置：<code>client/src/media/webrtc_publisher.cpp</code>）

```c++
  auto &self = *impl_;
  // 处理来自服务器的远程消息
  if (!self.pipeline)
    return;
  if (message.value("media_epoch", 0u) != self.epoch)
    return;
  if (message.value("op", "") == "webrtc_answer") {
    if (!message.contains("tracks") || message["tracks"] != self.offered_tracks)
      throw std::runtime_error(
          "answer did not confirm the offered MID/source mapping");
    self.setRemote(self.peer, "answer", message.at("sdp"));
    self.remote_set = true;
    for (auto &candidate : self.candidates)
      self.iceRemote(self.peer, candidate);
    self.candidates.clear();
  } else if (message.value("op", "") == "webrtc_ice") {
    if (self.remote_set)
      self.iceRemote(self.peer, message);
    else if (self.candidates.size() < 128)
      self.candidates.push_back(message);
    else
      self.error("ICE candidate cache overflow");
  }
```

### `WebRtcPublisher::remote()`：应用 Answer 并处理服务器候选

函数先确认 Publisher 管线还存在，再检查消息的 <code>media_epoch</code> 是否属于当前媒体会话。旧会话的消息会被忽略。

#### 服务端 Answer：`local=false` 的设置流程

远端运行时，服务器返回的 <code>webrtc_answer</code> 由 <code>media_client</code> 交给 <code>publisher->remote(message)</code>。该分支先确认服务器返回的轨道映射与 Offer 中的 MID/流映射一致，然后调用：

```c++
self.setRemote(self.peer, "answer", message.at("sdp"));
```

这里省略了第四个参数，所以使用默认值 <code>local=false</code>。流程如下：

1. **把服务器 Answer 解析为 SDP 对象**：<code>setRemote()</code> 把 JSON 中的 SDP 文本解析成 <code>GstSDPMessage</code>。因为本次目标是 <code>self.peer</code>，类型是 <code>answer</code>，辅助函数还会检查 Answer 是否包含项目要求的三条媒体行和 ID 为 7 的采集扩展；校验失败就抛异常，不继续设置。
2. **构造 Answer 描述并选择远端动作**：函数用 <code>GST_WEBRTC_SDP_TYPE_ANSWER</code> 包装 SDP。<code>local=false</code> 使动作名选为 <code>set-remote-description</code>。这里的 <code>peer</code> 仍是客户端本机的 <code>webrtcbin</code>；“remote”指这份 Answer 描述的是协商对端，不是说代码找到了一个远端 GStreamer 对象。
3. **交给本机 webrtcbin 应用**：<code>description()</code> 把 Answer 描述交给 <code>peerAction()</code>。它创建 GstPromise 和 C++ future，再执行：

   ```c++
   g_signal_emit_by_name(peer, "set-remote-description",
                         answer_description, promise);
   ```

   这行立即向本机 <code>webrtcbin</code> 发出 action；Promise 用于接收操作结果。<code>peerAction()</code> 随后最多等待 10 秒，并检查回复状态和错误。等待机制与本地 Offer 的 <code>set-local-description</code> 相同，具体见[本地 Peer 配置流程](#local-sdp-apply)。
4. **应用完成后再处理 ICE**：<code>setRemote()</code> 成功返回后，<code>remote()</code> 才设置 <code>remote_set = true</code>，并把先到、暂存的服务器 ICE candidate 逐条交给 <code>iceRemote()</code>。之后到达的 <code>webrtc_ice</code> 如果发现 <code>remote_set</code> 已为真，就直接应用；否则放入最多 128 条的候选队列。队列已满时报告 ICE candidate cache overflow。

Answer 和候选可能以任意先后顺序到达。先应用远端 Answer，再调用 <code>add-ice-candidate</code>，可确保客户端 PeerConnection 已经有远端描述。

> [!NOTE]
>
> 客户端没有自己实现一套完整的 WebRTC/ICE 状态机。应用层维护的是几个流程标记：<code>offered</code> 防止重复发起 Offer，<code>remote_set</code> 标记 Answer 是否已应用，<code>connected</code> 缓存从 <code>peer</code> 读取的连接状态，<code>failed</code> 记录应用错误。真正的 signaling、ICE gathering 和连通性检查状态由 <code>webrtcbin</code>/ICE agent 管理；应用通过 <code>on-ice-candidate</code>、<code>ice-gathering-state</code>、<code>connection-state</code> 等通知或属性读取这些状态。收到远端 Answer 和 candidate 后，应用把它们交给 GStreamer，ICE agent 会继续内部连通性检查。

<a id="remote-ice-apply"></a>

### `Impl::iceRemote()`：校验并加入远端 ICE candidate

```c++
void iceRemote(GstElement *target, const json &message) {
  const auto candidate = message.value("candidate", "");
  const auto line = message.value("sdp_mline_index", 0u);
  if (candidate.size() > 4096 || line > 2)
    throw std::invalid_argument("invalid ICE candidate");
  g_signal_emit_by_name(target, "add-ice-candidate", line, candidate.c_str());
}
```

远端运行时，<code>target</code> 是客户端发布端 <code>self.peer</code>。要区分管线容器和这个目标元素：<code>self.pipeline</code> 是 GStreamer 管线；<code>self.peer</code> 则由 <code>gst_bin_get_by_name(GST_BIN(self.pipeline), "peer")</code> 从管线中取出，是管线描述里 <code>webrtcbin name=peer</code> 创建的子元素。因此，查找 <code>self.peer</code> 属于管线元素访问；下面的 <code>add-ice-candidate</code> 才是发给该 <code>webrtcbin</code> 的 WebRTC 操作。元素创建和命名见 [Publisher 管线](WebRTC实现.md#publisher-pipeline)。

函数从信令 JSON 取候选字符串和对应的 SDP media line 序号，拒绝超过 4096 字节的候选或大于 2 的行号；本项目协商三条媒体行，因此有效行号为 0、1、2。检查通过后，<code>g_signal_emit_by_name(..., "add-ice-candidate", ...)</code> 调用 <code>webrtcbin</code> 的 GStreamer API，把服务端候选交给客户端 PeerConnection 参与 ICE 连通性检查。空候选会作为候选收集结束标记传入。实现位置：<code>client/src/media/webrtc_publisher.cpp:608-614</code>。

<a id="server-ice"></a>

## 服务端 ICE 信令处理实现

服务端接收和转发 ICE 信令的路径如下：

1. <code>server/libs/core/ws_handler.cpp:77-79</code> 校验 session 与 media_epoch 后，把 <code>webrtc_offer</code> 或 <code>webrtc_ice</code> 放入媒体控制队列。
2. <code>server/libs/core/media_service.cpp:122-147</code> 消费命令；Offer 交给 <code>WebRtcReceiver::offer()</code>，ICE 候选交给 <code>WebRtcReceiver::ice()</code>。
3. <code>server/libs/media/webrtc_receiver.cpp:186-231</code> 校验 SDP Offer、配置三条接收轨、在 <code>:213</code> 注册 <code>on-ice-candidate</code> 回调、设置远端 Offer 并创建 Answer。
4. 同一文件 <code>:97-99</code> 的 <code>Impl::candidate</code> 回调把服务端本地产生的候选（空候选表示收集结束）放进信令队列；<code>:238-241</code> 的 <code>poll()</code> 再把队列消息交给 WebSocket 发送回调。
5. <code>:233-236</code> 的 <code>WebRtcReceiver::ice()</code> 校验收到的候选和媒体行编号，再调用 <code>add-ice-candidate</code> 加到服务端 PeerConnection。

这一段对应信令的两个方向：<code>candidate()</code> 产生服务端本地候选并发回客户端；<code>ice()</code> 接收客户端候选并交给服务端 PeerConnection。

<a id="server-media-receive"></a>

## 服务端真实媒体接收与处理

以下是远端生产链路的服务端收包路径，说明客户端发出的媒体如何离开 <code>webrtcbin</code> 并变成服务端业务样本。

### 1. Offer 建立三条接收轨

<code>WebRtcReceiver::offer()</code> 校验收到的三条媒体行、payload type、方向和 MID 映射，创建服务端 <code>webrtcbin</code>，并为视频、CS2 音频和 Oopz 音频添加三个 <code>RECVONLY</code> transceiver。服务端应用远端 Offer 并创建 Answer 后，客户端的媒体包就可以沿协商出的 WebRTC 通道到达这个接收端。代码见 <code>server/libs/media/webrtc_receiver.cpp:186-231</code>。

### 2. `Impl::incoming()` 按 RTP payload 接入解码支路

服务端 <code>webrtcbin</code> 收到某条媒体轨后发出 <code>pad-added</code>；<code>Impl::incoming()</code> 从新 pad 的 caps 读 payload type，并按约定路由：96 是 H.264 视频，97 是 CS2 Opus，98 是 Oopz Opus。随后为该 pad 建立解包、解码和 appsink 支路，并将支路连接到新 pad。代码见 <code>server/libs/media/webrtc_receiver.cpp:152-180</code>。

视频支路使用 <code>rtph264depay → h264parse → openh264dec → videoconvert → appsink</code>；两路音频分别使用 <code>rtpopusdepay → opusdec → appsink</code>，输出统一为 16 kHz、单声道 S16LE PCM。启用录像时，视频还会从编码后的数据分出录像支路。

### 3. 恢复 RTP 采集元数据，`Impl::sample()` 生成业务样本

客户端把采集时间和源单元编号放进 RTP header extension。服务端注册的 TacticLens RTP 扩展读取器将其解码为 buffer 上的 <code>CaptureMeta</code>，并记录接收时间。代码见 <code>server/libs/media/webrtc_receiver.cpp:32-47</code>。

每当 appsink 的 <code>new-sample</code> 信号触发，<code>Impl::sample()</code> 拉取一个已解码的 sample：视频分支映射 BGR 帧并生成含帧 ID、采集时间、接收时间、解码时间和尺寸的 <code>VideoSample</code>；音频分支生成带 <code>cs2</code>/<code>oopz</code> 来源、采集时间和 PCM 的 <code>AudioSample</code>。随后分别调用 <code>WebRtcReceiver</code> 构造时传入的视频或音频回调。代码见 <code>server/libs/media/webrtc_receiver.cpp:100-126</code>。

### 4. `media_service` 把样本交给下游

接收器回调将视频样本放入帧队列，将两路音频放进各自队列。<code>media_service::drain()</code> 把有效尺寸的视频帧交给 <code>ingestPipeline().onMediaFrame()</code>；音频样本交给对应的 <code>AudioAssembler</code>，组装出的块发布到 EventBus，并按配置归档。代码见 <code>server/libs/core/media_service.cpp:51-79</code>、<code>:131-148</code>。

因此，远端数据的服务端路径是：**SRTP 到达服务端 webrtcbin → RTP 轨道 pad → 解包/解码 → `Impl::sample()` → `VideoSample` / `AudioSample` → `media_service` → 感知与语音处理**。
