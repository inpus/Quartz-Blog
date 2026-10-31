# WebSocket 重复异步写触发 Beast 断言

## 涉及文件

| 文件 | 作用 |
| --- | --- |
| `client/src/uploader.cpp` | pending 队列、writing 标志、doWrite 写泵及完成回调 |
| `client/src/uploader.hpp` | 上传器对外接口 |

## 背景与现象（S）

上传器采用一个 Asio IO 线程，所有发送进入 pending 队列。开发时握手回调和入队路径都可能启动写泵，结果触发 Beast `soft_mutex` 断言，错误包含 `id_ != T::id`。

## 要达到的目标（T）

对同一个 WebSocket 连接，任意时刻最多只有一个尚未完成的 `async_write`。后续消息应在前一个写完成后继续发送，不能依赖调用者自行避免重复启动。

## 排查与根因（A）

一个 IO 线程只保证回调代码不会同时执行，不保证异步操作的生命周期不重叠：

```text
回调 A：发起 async_write 后立即返回
回调 B：前一个写尚未完成，再次发起 async_write
稍后：第一个写的完成回调才被调度
```

因此，即使没有两个线程并行进入 `doWrite()`，仍可能在同一 stream 上挂起两个写。真正需要保护的是“在途操作”，而不只是函数执行期间的临界区。

## 如何解决

让写泵本身具备幂等入口。当前实现的关键控制流为：

```cpp
void doWrite() {
    if (writing) return;
    if (!ws || pending.empty() || (stopping && !flushing)) return;
    writing = true;
    // async_write：完成后清 writing、弹出队首，再决定是否继续。
}
```

只在真正发起写之前将 `writing` 置为 true。完成回调清除标志，成功后弹出已发送的队首；队列仍有数据时，由该完成回调启动下一次写。

当前代码还在回调中持有 stream 和 bytes，保证异步写期间对象、缓冲区有效，并用连接 generation 排除旧连接回调对新连接状态的影响。这些措施分别处理操作重叠、资源寿命和重连代次问题。

## 验证与结果（R）

2026-10-04 日志记录此断言问题已经修复，同轮客户端真机上传和服务器归档完成。当前源码仍保留 `if (writing) return` 及相关原因注释。

当日回归为 Windows 16/16、Linux 48/48；这些是整轮结果，不是一个专门测量写泵吞吐的基准。

## 可以复用的经验

异步状态机需要明确“谁拥有下一次操作的启动权”。对于串行发送，最清楚的安排是空闲时入队者起泵，运行后由完成回调接力。单线程执行器不能代替这套状态约束。

## 依据

- `docs/日志记录/day3.md`：“踩坑记录”第 2 项。
- `client/src/uploader.cpp`：`doWrite()`、writing 标志和异步完成回调。
