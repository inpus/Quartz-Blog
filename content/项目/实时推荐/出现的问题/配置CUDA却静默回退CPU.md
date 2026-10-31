# 配置 CUDA 却静默回退 CPU

## 涉及文件

| 文件 | 作用 |
| --- | --- |
| `server/libs/perception/radar_track.cpp` | 创建 ORT Session、选择执行后端并记录实际 EP |
| `server/libs/perception/hud_ocr.cpp` | OCR 初始化前建立共享 ORT 环境 |
| `server/libs/perception/ort_env.hpp` | 共享 Ort::Env 访问与生命周期 |
| `config/perception.yaml` | 请求的 GPU 设备及感知参数 |

## 背景与现象（S）

2026-10-07 将 C++ 雷达推理接入 ORT GPU 包后，配置要求 CUDA，程序却表现得像 CPU：GPU 没有预期负载，耗时接近 CPU 对照。初始化路径抛出：

```text
Attempt to use DefaultLogger but none has been registered
```

异常被降级逻辑接住后，功能仍可运行，使性能问题更容易被忽略。原来的后端标签还可能沿用请求值，形成“日志写 CUDA，实际跑 CPU”的误导。

## 要达到的目标（T）

定位 CUDA Session 没有建成的原因，使实际执行后端、日志与耗时证据一致；成功时验证 GPU 推理结果与 CPU 相符。

## 排查与根因（A）

对照不同构造策略并逐项二分后，问题定位到初始化顺序。ORT 1.26 的 EP 框架需要已经注册的全局日志实例，而 `Ort::Env` 承担环境初始化。旧顺序先构造 `Ort::SessionOptions`，再取 Env，因此在 EP 初始化时环境尚未就绪。

同样问题在两套头文件对照中都出现，说明不能仅靠更换头文件解释或修复。GPU 包路径正确，也不等于对象初始化顺序正确。

## 如何解决

把依赖顺序写进 Session 创建路径，先取得共享环境，再创建选项：

```cpp
Ort::Env& env = ortEnv();
auto so = Ort::SessionOptions{};
// 再配置 CUDA EP 并使用 env 创建 Session。
```

同时修改后端标记，使其取自实际建成的 Session：CUDA 尝试失败而 CPU 成功时，明确记录 CPU，不能继续沿用 CUDA 请求值。OCR 接入也在其内部构造 SessionOptions 前初始化共享 Env。

GPU 动态库、CUDA/cuDNN 依赖路径仍需正确，但它们与本次构造顺序根因分别核对。

## 验证与结果（R）

2026-10-07 雷达对照中，实际 `ep=CUDA`，检出逐类与 CPU 一致。42 帧稳态耗时中位约 2.92ms/帧，Python CUDA 对照约 2.89ms/帧；CPU 对照约 30–117ms/帧。

首帧约 820ms，包含 CUDA 上下文及 cuDNN autotune 等一次性成本。因此记录时分开给出冷启动和稳态，不能用其中一个替代另一个。

这是当时雷达模型的性能记录，不外推到 OCR、ASR 或整个系统帧率。Env 的退出阶段生命周期另见 [[项目/实时推荐/出现的问题/模型推理成功但Release进程退出崩溃|模型退出崩溃问题]]。

## 可以复用的经验

有回退的系统需要同时记录请求后端、实际后端、回退原因和性能。排查初始化异常时，除了版本与库路径，还要检查对象之间的构造依赖；异常被兜住不等于目标配置已生效。

## 依据

- `docs/日志记录/day9.md`：“ort-gpu C++ 接入”。
- `server/libs/perception/radar_track.cpp`：Env 先于 SessionOptions 的实现与注释。
