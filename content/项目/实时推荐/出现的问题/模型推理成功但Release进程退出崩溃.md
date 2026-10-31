# 模型推理成功但 Release 进程退出崩溃

## 涉及文件

| 文件 | 作用 |
| --- | --- |
| `server/libs/perception/ort_env.hpp` | OrtEnvironmentScope 与共享 Env 管理 |
| `server/apps/server_main.cpp` | 在 main 作用域持有环境生命周期 |
| `server/apps/radar_probe.cpp` | 命令行工具使用同一收尾约束 |
| `server/tests/test_main.cpp` | Catch 入口持有环境作用域 |
| `server/tests/perception_tests.cpp` | OCR 与雷达实模型测试 |

## 背景与现象（S）

OCR 和雷达的 Release 实模型测试能够加载模型、完成推理和断言，但进程退出时出现 `corrupted double-linked list`。如果只看断言或模型输出，会误以为测试已经成功。

## 要达到的目标（T）

找出正常推理之后的退出崩溃原因，使会话、GPU 资源、共享 ORT 环境和进程退出清理形成可控顺序，完整跑通 Release 测试。

## 排查与根因（A）

最小复现只改变 Env 的生命周期：同一 CUDA 模型使用局部 Env 时能正常退出，改为函数静态 Env 时推理成功、退出崩溃。gdb 栈位于 ORT 的进程退出清理阶段。

共享静态 Env 的析构晚于 CUDA provider 注册的退出清理。某些共享状态已经被清理后，Env 又开始释放相关资源，触发堆损坏。问题落在资源回收顺序，而不是推理结果。

这与 [[项目/实时推荐/出现的问题/配置CUDA却静默回退CPU|Env 必须先于 SessionOptions 构造]] 属于同一资源的两个时段：前者要求初始化顺序，本文处理退出顺序。

## 如何解决

引入 `OrtEnvironmentScope`，由程序 main 作用域显式持有，并在模型与工作线程对象之前声明。顺序示意为：

```cpp
int main() {
    OrtEnvironmentScope runtime;
    // 创建服务、线程、模型与 Session。
    // 退出时先停止并销毁这些对象。
}   // runtime 最后释放共享 Env，随后才进入进程退出清理。
```

`ort_env.hpp` 维护共享 Env、互斥量和作用域 owner 计数；最后一个作用域退出时执行 `env.reset()`。这种设计的前提是所有会话与使用 Env 的线程都包含在作用域内，不能留有越过 scope 的使用者。

server、radar-probe 和自定义 Catch 入口都遵守这套约束。修复没有通过泄漏 Env、跳过析构、屏蔽实模型测试或强行 `_exit` 隐藏异常。

## 验证与结果（R）

2026-10-26：

- `HudOcr.PpOcrV4ReadsReferenceFrame` 连续 3 次通过并正常退出。
- `Radar.OnnxRealModel.SmokeOnValImage` 连续 3 次通过并正常退出。
- 完整 Release CTest 116/116 通过，没有排除或跳过用例。

上述检查验证了断言与进程收尾；它不提供模型准确率或长时间服务稳定性指标。

## 可以复用的经验

运行成功需要包括退出成功。面对第三方 GPU 库，最小复现应同时比较自动对象、共享对象和静态对象的生命周期。将外部运行环境放到 main 的明确作用域，比依赖跨动态库的静态析构顺序更容易推理和验证。

## 依据

- `docs/双路实时识别与网页推荐验收.md`：“Release退出异常”。
- 服务器 `var/realtime-20261026/model-exit-tests.log`、`tests-release-final.log`、`ort-static-gpu.log` 的文档索引。
- `server/libs/perception/ort_env.hpp` 与三个程序入口。
