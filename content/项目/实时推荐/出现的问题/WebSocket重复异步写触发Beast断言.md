# WebSocket 重复异步写触发 Beast 断言

## 涉及文件

- `client/src/uploader.cpp`：pending 队列、writing 标志、doWrite 和连接代次。
- `client/src/uploader.hpp`：上传器接口。

## 问题

上传器只用了一个 Asio IO 线程，仍然触发 Beast 的 `soft_mutex` 断言：`id_ != T::id`。原因是握手回调和入队路径都可能启动 `doWrite()`，前一个 `async_write` 还没完成，下一个又开始了。

## 定位

我原来只考虑了“有没有两个线程同时执行”，没有把“异步操作是否还在进行”单独作为状态。

```text
回调 A：调用 async_write → 立刻返回，写操作仍在进行
回调 B：又调用 async_write → 同一连接出现两个在途写
之后：第一个写的 completion 才被调度
```

一个线程只能保证两段回调代码不同时运行，不能保证它们发起的异步操作不重叠。

## 修改前

按当时故障路径简化：

```cpp
void enqueue(std::vector<uint8_t> bytes) {
    pending.push_back(std::move(bytes));
    doWrite();
}

void doWrite() {
    if (!ws || pending.empty()) return;
    ws->async_write(net::buffer(pending.front()),
        [this](beast::error_code ec, std::size_t) {
            if (ec) return onFail("write", ec);
            pending.pop_front();
            doWrite();
        });
}
```

第一次写发起后，队首需要一直保留到回调完成。此时 enqueue 或握手路径再进入 doWrite，就会针对同一连接重复发起写。仅在 `async_write()` 调用前后用普通局部锁也不够，因为函数返回时异步写尚未结束。

## 修改后

`writing` 表示“已经有一个尚未完成的写”，由写泵自己检查。下面保留当前关键逻辑，关停关闭分支省略：

```cpp
void doWrite() {
    if (writing) return;
    if (!ws || pending.empty() || (stopping && !flushing)) return;
    writing = true;

    auto stream = ws;
    auto bytes = pending.front();
    const auto gen = generation;
    stream->async_write(net::buffer(*bytes),
        [this, stream, bytes, gen](beast::error_code ec, std::size_t) {
            if (gen != generation) return;
            writing = false;
            if (ec) return onFail("write", ec);

            if (!pending.empty()) pending.pop_front();
            if (!pending.empty() && (connected || flushing)) {
                doWrite();
            }
        });
}
```

pending 保存的是 `shared_ptr<vector<uint8_t>>`。入队时，如果正在写就只排队；空闲时才起泵：

```cpp
pending.push_back(
    std::make_shared<std::vector<uint8_t>>(std::move(bytes)));
if (!writing) doWrite();
```

队列满需要淘汰数据时，也不能删掉正在写的队首，当前代码从 `begin() + (writing ? 1 : 0)` 选择可丢弃项。控制模式溢出则直接报连接错误。

## 为什么这样改

修复后，状态顺序固定为：空闲 → 设置 writing → 发起一次写 → 完成回调清 writing → 弹出队首 → 继续下一次写。无论外面多少地方调用 doWrite，只要前一个写还没结束，都不会再次进入发送。

这里还有两个不能省略的细节：

- 回调按值捕获 `stream`、`bytes`，使连接和缓冲区在异步操作期间保持有效。`net::buffer` 本身不拥有数据。
- 回调先判断 `gen`，旧连接完成的写不能清掉新连接的 writing，也不能弹出新队列的队首。generation 是处理重连代次，不是用来串行化当前连接。

writing 只由 IO 线程访问，因此这里不需要把它改成原子变量。真正的约束是所有相关状态修改都在同一执行线程上，且写泵遵守在途标志。

## 验证

2026-10-04 修复后完成了客户端真机上传和服务端归档，当轮 Windows 16/16、Linux 48/48 测试通过。源码中保留了 writing 的幂等检查及原因注释。

## 总结

我通过队列加写泵保证一个连接最多有一个在途写：空闲时入队者起泵，之后由完成回调连续发送。这个问题的关键不是加多少线程锁，而是异步操作的生命周期。单 IO 线程不会自动消除异步操作重叠，缓冲区寿命和重连后的旧回调也要一起考虑。

## 相关记录

- `docs/日志记录/day3.md`：踩坑第 2 项。
