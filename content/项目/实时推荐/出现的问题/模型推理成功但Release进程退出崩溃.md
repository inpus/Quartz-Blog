# 模型推理成功但 Release 进程退出崩溃

## 涉及文件

- `server/libs/perception/ort_env.hpp`：共享 Env 与 OrtEnvironmentScope。
- `server/apps/server_main.cpp`：服务入口的环境作用域。
- `server/apps/radar_probe.cpp`：雷达工具的环境作用域。
- `server/tests/test_main.cpp`：实模型测试入口。
- `server/tests/perception_tests.cpp`：OCR/雷达实模型测试。

## 问题

OCR 和雷达测试都能加载模型、完成推理，断言也通过，但 Release 进程退出时出现 `corrupted double-linked list`。只看推理结果，会漏掉这个错误。

## 定位

我把问题缩小到同一个 CUDA 模型，只改变 Env 的持有方式：局部 Env 正常退出，函数静态 Env 则在退出时崩溃。gdb 栈落在 ORT 的进程退出清理阶段。

复现表明，共享静态 Env 的析构晚于 CUDA provider 注册的退出清理，等 Env 再释放资源时，相关共享状态已经进入清理过程。继续修改模型输入或识别代码没有意义，应该调整资源生命周期。

## 修改前

旧设计可以简化为一个函数静态单例：

```cpp
Ort::Env& ortEnv() {
    static Ort::Env env(ORT_LOGGING_LEVEL_WARNING, "tacticlens-perception");
    return env;
}

int main() {
    // 多个模型通过 ortEnv() 共享环境。
    // main 返回后，静态 Env 才进入进程级析构。
}
```

这能解决共享和延迟初始化，却把回收时间交给了进程退出阶段。模型 Session 在局部作用域销毁，不等于它依赖的 Env 一定能在第三方退出清理前销毁。

## 修改后

我保留共享入口，但把 Env 放进可显式 reset 的 unique_ptr。当前持有结构为：

```cpp
struct Environment {
    std::mutex mutex;
    std::unique_ptr<Ort::Env> env;
    std::size_t owners = 0;
};
inline Environment& environment() {
    static Environment value;
    return value;
}

inline Ort::Env& ortEnv() {
    auto& state = detail::environment();
    std::lock_guard lock(state.mutex);
    if (!state.env) {
        state.env = std::make_unique<Ort::Env>(
            ORT_LOGGING_LEVEL_WARNING, "tacticlens-perception");
    }
    return *state.env;
}
```

再用一个 main 作用域对象管理最后释放时间：

```cpp
class OrtEnvironmentScope {
public:
    OrtEnvironmentScope() {
        auto& state = detail::environment();
        std::lock_guard lock(state.mutex);
        ++state.owners;
    }
    ~OrtEnvironmentScope() {
        auto& state = detail::environment();
        std::lock_guard lock(state.mutex);
        if (!--state.owners) state.env.reset();
    }
    OrtEnvironmentScope(const OrtEnvironmentScope&) = delete;
    OrtEnvironmentScope& operator=(const OrtEnvironmentScope&) = delete;
};
```

入口按下面的声明顺序持有对象，示意中的服务名已简化：

```cpp
int main() {
    tl::perception::OrtEnvironmentScope runtime;
    {
        // 创建 Session、模型、服务与线程。
        // 收尾：先停线程并 join，再销毁模型和 Session。
    }
}   // runtime 析构，主动释放共享 Env。
```

server、radar-probe 和 Catch 的自定义入口都要使用这套约束，不能只修其中一个程序。

## 为什么这样改

局部对象按构造的逆序销毁。runtime 先声明，模型与服务后构造；只要它们不越过这个作用域，退出时就会先完成模型、Session 和线程收尾，最后释放 Env。此时还没有离开 main，不必等到进程静态对象清理阶段。

Environment 容器本身仍可以是静态的，但里面的 Env 已经被 reset，后面容器析构时只剩空指针。owners 统计的是环境作用域持有者，不是自动追踪所有 Session 的引用计数；如果有 Session 或后台线程活得比 runtime 更久，这个设计仍然会被破坏。

所以修复有两部分：显式控制 Env 的释放时间，以及保证所有使用者确实先结束。只加一个计数器或互斥锁并不能自动解决后者。

我没有用 `_exit`、泄漏单例或跳过实模型测试来绕过退出路径。

## 验证

2026-10-26，OCR 实模型和雷达实模型用例分别连续 3 次正常通过并退出；完整 Release CTest 116/116 通过，没有排除或跳过。记录保存在 `var/realtime-20261026/` 下的模型退出与测试日志中。

## 总结

我通过最小复现把问题从推理阶段缩小到退出阶段，再用 main 作用域明确资源的销毁顺序：线程和 Session 先结束，Env 后释放，最后才进入进程退出清理。函数静态对象使用方便，但依赖 GPU 动态库时，不能忽略它和第三方退出回调之间的先后关系。

## 相关记录

- `docs/双路实时识别与网页推荐验收.md`：Release 退出异常。
