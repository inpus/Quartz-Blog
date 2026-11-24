# 配置 CUDA 却静默回退 CPU

## 涉及文件

- `server/libs/perception/radar_track.cpp`：Session 创建、CUDA/CPU 回退和 ep_ 标记。
- `server/libs/perception/hud_ocr.cpp`：OCR Session 初始化前准备共享环境。
- `server/libs/perception/ort_env.hpp`：共享 Ort::Env。
- `config/perception.yaml`：设备编号和模型配置。

## 问题

配置要求使用 CUDA，模型却跑得像 CPU，GPU 没有预期负载。初始化日志中有：

```text
Attempt to use DefaultLogger but none has been registered
```

异常被回退逻辑接住，模型仍能输出结果；如果后端标签还保留请求的 CUDA，就会出现“日志显示 CUDA，实际运行 CPU”的情况。

## 定位

我对照不同初始化顺序，并排除了单纯更换头文件的影响。最终定位到 `Ort::SessionOptions` 构造早于 `Ort::Env`。这次使用的 ORT 1.26 路径依赖已经建立的全局日志环境，顺序不对时 CUDA 初始化失败。

模型文件存在、GPU 包可找到、代码调用了 AppendExecutionProvider_CUDA，三者都不能证明 CUDA Session 已经成功建立。

## 修改前

按故障顺序简化：

```cpp
Ort::SessionOptions so;  // 环境还未准备好就先创建选项。
Ort::Env& env = ortEnv();
so.AppendExecutionProvider_CUDA(cuda_options);
// 创建 Session；失败后走 CPU 回退。
```

另一个错误是把请求值直接当成实际结果：

```cpp
ep_ = requested_cuda ? "CUDA" : "CPU";
// 即使 CUDA 创建失败、后面 CPU 成功，ep_ 也没有对应改正。
```

## 修改后

当前代码把创建过程集中到 buildSession，先取 Env，再创建选项；只有 Session 建成后才保存结果：

```cpp
const auto buildSession = [&](bool cuda, std::string* caught) -> bool {
    try {
        Ort::Env& env = ortEnv();
        auto so = Ort::SessionOptions{};
        so.SetIntraOpNumThreads(2);
        so.SetGraphOptimizationLevel(GraphOptimizationLevel::ORT_ENABLE_ALL);
        if (cuda) {
            OrtCUDAProviderOptions opts{};
            opts.device_id = rc.device_id;
            so.AppendExecutionProvider_CUDA(opts);
        }

        auto s = std::make_unique<Session>();
        s->session = Ort::Session(env, rc.model_path.c_str(), so);
        // 输入输出名和输入尺寸的保存代码省略。
        sess_ = std::move(s);
        ep_ = cuda ? "CUDA" : "CPU";
        return true;
    } catch (const Ort::Exception& e) {
        if (caught) *caught = e.what();
        return false;
    }
};
```

外层仍然可以先尝试 CUDA，失败再建 CPU Session；每次调用都重新构造 SessionOptions，避免将失败选项直接混入下一次尝试。OCR 的初始化入口也先调用 `ortEnv()`，再让其内部构建 SessionOptions。

## 为什么这样改

先创建 Env，使后续选项和 CUDA EP 初始化需要的环境已经存在。将赋值放到 Session 创建成功之后，只有真正成功的构建路径才会写入 sess_ 和 ep_。

这里 ep_ 表示“这次成功创建了带 CUDA EP 的 Session”或“走了 CPU 回退路径”，并不等于整张图的每个算子都在 GPU 上。若要判断算子分配，仍需要 ORT profile，不能只用一个字符串下结论。

这个修复针对初始化顺序。动态库找不到、CUDA/cuDNN 版本不兼容等失败仍要分别处理，不能认为调整顺序能修好所有 CUDA 错误。

## 验证

2026-10-07，雷达实际使用 CUDA，逐类检出与 CPU 一致。42 帧稳态耗时中位约 2.92ms/帧，Python CUDA 约 2.89ms/帧，CPU 对照为约 30–117ms/帧。首帧约 820ms，包含上下文和 autotune 等初始化成本，因此冷启动与稳态分别记录。

## 总结

我先修正 Env 与 SessionOptions 的构造顺序，再让后端标记跟着成功构建的路径更新。有自动回退的模块不能只记录想用什么，还要知道实际用了什么、为什么回退，并用耗时和 profile 交叉检查。能出结果不代表加速已经生效。

## 相关记录

- `docs/日志记录/day9.md`：ort-gpu C++ 接入。
- [[项目/实时推荐/出现的问题/模型推理成功但Release进程退出崩溃|Env 的退出生命周期问题]]。
