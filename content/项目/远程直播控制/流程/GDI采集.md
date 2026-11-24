## GDI 采集

### 1. GDI 采集的整体思路

这个工程里的 GDI 采集并不是直接用 Win32 的 `BitBlt` / `GetDIBits` 一帧一帧手工抓屏，而是把桌面采集交给 FFmpeg 的 `gdigrab` 输入设备来做：

- 先注册 FFmpeg 的设备输入模块
- 通过 `gdigrab` 打开桌面采集源
- 读取视频流
- 找到视频解码器并解码成 `AVFrame`
- 再把每一帧 BGRA / RGBA 数据放到内部缓存中，供后续编码和推流使用

也就是说，工程在底层使用了 FFmpeg 的 GDI 采集能力，核心流程本质上是：

- 采集设备：`gdigrab`
- 输入格式：`AVInputFormat`
- 视频流：`AVFormatContext` + `AVStream`
- 解码器：`AVCodecContext`
- 输出图像：`AVFrame`

这种方式的好处是：

- 不需要自己处理 Win32 的桌面句柄和采集细节
- FFmpeg 已经封装了 GDI 采集的底层实现
- 对接到工程里更加简单，后续直接拿到 `AVFrame` 就行

---

### 2. 代码入口：构造函数与初始化

在 `GDIScreenCapture` 的构造函数中，首先注册 FFmpeg 的设备模块：

```c++
GDIScreenCapture::GDIScreenCapture()
    :stop_(false)
    ,is_initialzed_(false)
    ,frame_size_(0)
    ,rgba_frame_(nullptr)
    ,width_(0)
    ,height_(0)
    ,video_index_(-1)
    ,framerate_(25)
    ,input_format_(nullptr)
    ,codec_context_(nullptr)
    ,format_context_(nullptr)
    ,av_frame_(nullptr)
    ,av_packet_(nullptr)
{
    avdevice_register_all();
    av_frame_ = std::shared_ptr<AVFrame>(av_frame_alloc(), [](AVFrame* ptr){ av_frame_free(&ptr); });
    av_packet_ = std::shared_ptr<AVPacket>(av_packet_alloc(), [](AVPacket* ptr){ av_packet_free(&ptr); });
}
```

这里的核心动作有两个：

1. `avdevice_register_all()`
   - 注册 FFmpeg 的所有设备输入模块
   - 包括 `gdigrab`、`dshow` 等
   - 这样后面才能通过 `av_find_input_format("gdigrab")` 找到 GDI 采集输入设备

2. `av_frame_alloc()` 和 `av_packet_alloc()`
   - 为后续解码每一帧画面和读取每一包数据准备内存

> `gdigrab`：FFmpeg 中 Windows GDI 桌面采集的输入格式。
>
> 其本质是通过 Windows 桌面/窗口图形 API 采集图像，但在工程里包装成 FFmpeg 输入设备来使用。

---

### 3. 初始化采集：打开桌面输入

初始化函数 `Init()` 是整个 GDI 采集的重中之重：

```c++
bool GDIScreenCapture::Init(qint64 display_index)
{
    if(is_initialzed_)
    {
        return true;
    }

    AVDictionary *options = nullptr;
    av_dict_set_int(&options, "framerate", framerate_, AV_DICT_MATCH_CASE);
    av_dict_set_int(&options, "draw_mouse", 1, AV_DICT_MATCH_CASE);
    av_dict_set_int(&options, "offset_x", 0, AV_DICT_MATCH_CASE);
    av_dict_set_int(&options, "offset_y", 0, AV_DICT_MATCH_CASE);
    av_dict_set(&options, "video_size", "2560x1440", 1);

    input_format_ = const_cast<AVInputFormat*>(av_find_input_format("gdigrab"));
    if(!input_format_)
    {
        qDebug() << "av_find_input_format failed";
        return false;
    }

    format_context_ = avformat_alloc_context();
    if(avformat_open_input(&format_context_, "desktop", input_format_, &options) != 0)
    {
        qDebug() << "avformat_open_input failed";
        return false;
    }
```

#### 3.1 设置采集参数

```c++
av_dict_set_int(&options, "framerate", framerate_, AV_DICT_MATCH_CASE);
av_dict_set_int(&options, "draw_mouse", 1, AV_DICT_MATCH_CASE);
av_dict_set_int(&options, "offset_x", 0, AV_DICT_MATCH_CASE);
av_dict_set_int(&options, "offset_y", 0, AV_DICT_MATCH_CASE);
av_dict_set(&options, "video_size", "2560x1440", 1);
```

这些参数的含义是：

- `framerate`：采集帧率，当前默认 25 fps
- `draw_mouse`：是否采集鼠标指针，设为 1 表示把鼠标也一起抓进去
- `offset_x` / `offset_y`：采集区域的偏移
- `video_size`：目标采集分辨率，当前写死为 `2560x1440`

> 在 GDI 桌面采集里，`video_size` 很关键，因为它决定了采集区域大小；如果显示器分辨率更大或更小，采集输出尺寸要跟着调整。

#### 3.2 查找输入格式

```c++
input_format_ = const_cast<AVInputFormat*>(av_find_input_format("gdigrab"));
```

- `gdigrab` 是 Windows 下的桌面抓屏输入格式
- FFmpeg 通过它来访问本机桌面帧数据
- `AVInputFormat` 就是 “这个设备输入的描述器”

#### 3.3 打开输入

```c++
format_context_ = avformat_alloc_context();
avformat_open_input(&format_context_, "desktop", input_format_, &options);
```

这里有几个关键点：

- `"desktop"` 是输入源名称，表示采集桌面
- `format_context_` 是 `AVFormatContext`，它承载整个媒体输入上下文
- `avformat_open_input` 会打开这个采集源并准备读取数据

在 FFmpeg 体系里，`AVFormatContext` 本质就是“媒体容器上下文”，它既能管理输入数据，也能帮助搜索流、读取帧。

---

### 4. 读取流信息和找到视频流

初始化的下一步，是获取采集流的信息：

```c++
if(avformat_find_stream_info(format_context_, nullptr) < 0)
{
    qDebug() << "avformat_find_stream_info failed";
    return false;
}

int video_index = -1;
for(uint32_t i = 0; i < format_context_->nb_streams; i++)
{
    if(format_context_->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_VIDEO)
    {
        video_index = i;
    }
}

if(video_index == -1)
{
    return false;
}
```

这一段的意思是：

- `avformat_find_stream_info()`：读取采集流的媒体信息，解析出各个数据流
- 然后遍历所有流，定位 `AVMEDIA_TYPE_VIDEO` 这一条
- 找到后记住它的 `video_index`

这是视频采集最重要的识别动作：

> 采集源里可能有音频流，也可能只有视频流；我们必须先找到那条真正用于桌面画面的视频流。

---

### 5. 创建解码器并打开

找到视频流之后，下一步就是构造解码器：

```c++
AVCodec* codec = const_cast<AVCodec*>(avcodec_find_decoder(
    format_context_->streams[video_index]->codecpar->codec_id));
if(!codec)
{
    return false;
}

codec_context_ = avcodec_alloc_context3(codec);
if(!codec_context_)
{
    return false;
}

codec_context_->pix_fmt = AV_PIX_FMT_RGBA;
avcodec_parameters_to_context(codec_context_, format_context_->streams[video_index]->codecpar);

if(avcodec_open2(codec_context_, codec, nullptr) != 0)
{
    return false;
}
```

#### 5.1 选择解码器

```c++
avcodec_find_decoder(format_context_->streams[video_index]->codecpar->codec_id)
```

这里是根据流中的 `codec_id` 去找对应的解码器。`gdigrab` 输出通常是视频流，因此 FFmpeg 会找到匹配的解码器。

#### 5.2 分配解码器上下文

```c++
codec_context_ = avcodec_alloc_context3(codec);
```

`AVCodecContext` 承载解码器的运行状态，它保存了：

- 视频宽高
- 像素格式
- 帧率
- 解码器状态
- 采集帧输出参数

#### 5.3 设置输出格式

```c++
codec_context_->pix_fmt = AV_PIX_FMT_RGBA;
```

这里明确要求解码输出为 RGBA 格式。这样后续拿到 `AVFrame` 的像素数据时，就很容易转换成工程中真正使用的 RGB / RGBA 缓冲区。

> 这也是为什么后面 `CaptureFrame` 里用 `av_frame->data[0]` 和 `av_frame->linesize[0]` 处理像素数据：最终输出的帧已经被标准化成一块 RGBA 内存。

#### 5.4 打开解码器

```c++
avcodec_open2(codec_context_, codec, nullptr)
```

这一步最后确认解码器可以正常工作，随后即可从采集输入中读取视频帧并解码。

---

### 6. 采集线程：从桌面不断读取帧

初始化完成之后，代码会启动线程：

```c++
video_index_ = video_index;
is_initialzed_ = true;
this->start();
```

线程入口是：

```c++
void GDIScreenCapture::run()
{
    if(is_initialzed_ && !stop_)
    {
        while (!stop_) {
            std::this_thread::sleep_for(std::chrono::milliseconds(1000 / framerate_));
            GetOneFrame();
        }
    }
}
```

这里的逻辑很简单：

- 按固定帧率循环采集
- 每次 sleep 一小段时间
- 调用 `GetOneFrame()` 去读取一帧

也就是说，工程并不是“事件驱动型”的实时采集，而是“固定时钟循环抓帧”的方式。

---

### 7. 读取一帧：`av_read_frame`

`GetOneFrame()` 中非常关键：

```c++
bool GDIScreenCapture::GetOneFrame()
{
    if(stop_)
    {
        return false;
    }

    int ret = av_read_frame(format_context_, av_packet_.get());
    if(ret < 0)
    {
        return false;
    }

    if(av_packet_->stream_index == video_index_)
    {
        Decode(av_frame_.get(), av_packet_.get());
    }
    av_packet_unref(av_packet_.get());
    return true;
}
```

#### 7.1 读取媒体包

```c++
av_read_frame(format_context_, av_packet_.get())
```

这一步表示：

- 从整个输入流中读取一个数据包 `AVPacket`
- 这里的当前输入源就是桌面视频

#### 7.2 过滤视频流

```c++
if(av_packet_->stream_index == video_index_)
```

因为输入流里可能不仅仅是视频，也可能存在其他流；这里根据 `stream_index` 过滤出真正的视频包。

#### 7.3 解码

```c++
Decode(av_frame_.get(), av_packet_.get());
```

一旦拿到视频包，就交给 `Decode()` 负责解码成 `AVFrame`。

---

### 8. 解码流程：从数据包得到画面

核心解码函数：

```c++
bool GDIScreenCapture::Decode(AVFrame *av_frame, AVPacket *av_packet)
{
    int ret = avcodec_send_packet(codec_context_, av_packet);
    if(ret < 0)
    {
        return false;
    }
    if(ret >= 0)
    {
        ret = avcodec_receive_frame(codec_context_, av_frame);
        if(ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
        {
            return false;
        }
        if(ret < 0)
        {
            return false;
        }

        std::lock_guard<std::mutex> lock(mutex_);
        frame_size_ = av_frame->pkt_size;
        rgba_frame_.reset(new uint8_t[frame_size_], std::default_delete<uint8_t[]>());
        width_ = av_frame->width;
        height_ = av_frame->height;

        for(uint32_t i = 0; i < height_; i++)
        {
            memcpy(rgba_frame_.get() + i * width_ * 4,
                   av_frame->data[0] + i * av_frame->linesize[0],
                   av_frame->linesize[0]);
        }
        av_frame_unref(av_frame);
    }
    return true;
}
```

#### 8.1 发送数据包给解码器

```c++
avcodec_send_packet(codec_context_, av_packet);
```

将采集到的原始包喂给解码器。此时 FFmpeg 会按照对应编码器格式，将压缩视频数据还原成像素帧。

#### 8.2 接收解码后的帧

```c++
avcodec_receive_frame(codec_context_, av_frame);
```

这里的 `av_frame` 就是“真正的画面帧”，包含：

- `width`
- `height`
- `data[0]`
- `linesize[0]`
- `pix_fmt`

#### 8.3 复制到内部缓存

```c++
frame_size_ = av_frame->pkt_size;
rgba_frame_.reset(new uint8_t[frame_size_], std::default_delete<uint8_t[]>());
```

和 `CaptureFrame()` 配合，工程在这里把一帧数据拷贝到自己的 `rgba_frame_` 缓冲区中，避免解码器内部对象被后续操作覆盖。

```c++
for(uint32_t i = 0; i < height_; i++)
{
    memcpy(rgba_frame_.get() + i * width_ * 4,
           av_frame->data[0] + i * av_frame->linesize[0],
           av_frame->linesize[0]);
}
```

这里的做法是：

- 每一行按像素行复制
- 目标缓存是连续的 `RGBA` 数据
- 每行长度是 `width * 4` 个字节

> 因为工程将输出格式固定成 `AV_PIX_FMT_RGBA`，所以每个像素占 4 个字节，顺序通常为 `R,G,B,A`。

#### 8.4 释放帧对象

```c++
av_frame_unref(av_frame);
```

这是 FFmpeg 的常规用法：

- `av_frame_unref` 会释放当前 frame 内部引用
- 这样能避免一帧帧缓存堆积
- 让下一帧重新使用这一块内存

---

### 9. 对外输出：`CaptureFrame()`

采集线程不断更新内部缓存后，外部通过 `CaptureFrame()` 读取：

```c++
bool GDIScreenCapture::CaptureFrame(FrameContainer &rgba, quint32 &width, quint32 &height)
{
    std::lock_guard<std::mutex> lock(mutex_);
    if(stop_)
    {
        rgba.clear();
    }

    if(rgba_frame_ == nullptr || frame_size_ == 0)
    {
        rgba.clear();
        return false;
    }
    if(rgba.capacity() < frame_size_)
    {
        rgba.reserve(frame_size_);
    }

    rgba.assign(rgba_frame_.get(), rgba_frame_.get() + frame_size_);
    width = width_;
    height = height_;
    return true;
}
```

这段逻辑的关键点：

1. 加锁：防止线程读取和写入缓存冲突
2. 判断是否停止：若采集停了，就清空缓冲区
3. 判断是否有有效帧：如果没有新帧，直接返回 false
4. `rgba.assign(...)`：把内部缓存里的图片数据复制到调用方的 `std::vector<quint8>` 中

也就是说，外部拿到的是一份完整的 RGBA 帧缓存，可以继续送给编码器、推流器、显示器等模块使用。

---

### 10. 关闭采集：清理解码器和输入

```c++
bool GDIScreenCapture::Close()
{
    if(is_initialzed_)
    {
        StopCapture();
    }
    if(codec_context_)
    {
        avcodec_close(codec_context_);
        codec_context_ = nullptr;
    }
    if(format_context_)
    {
        avformat_close_input(&format_context_);
        format_context_ = nullptr;
    }
    input_format_ = nullptr;
    video_index_ = -1;
    is_initialzed_ = false;
    stop_ = true;
    return true;
}
```

关闭逻辑包括：

- 停止采集线程
- 关闭解码器
- 关闭格式上下文
- 重置内部状态

其中 `StopCapture()` 会：

```c++
void GDIScreenCapture::StopCapture()
{
    if(is_initialzed_)
    {
        stop_ = true;
        if(this->isRunning())
        {
            this->quit();
            this->wait();
        }

        std::lock_guard<std::mutex> lock(mutex_);
        rgba_frame_.reset();
        frame_size_ = 0;
        width_ = 0;
        height_ = 0;
    }
}
```

这里的意图是：

- 停止线程主循环
- 等待线程退出
- 清空缓存，避免残留数据

---

### 11. 总结：GDI 采集的完整链路

从代码看，GDI 采集的完整流程可以概括为：

```text
Windows 桌面 / 显示器
        ↓
FFmpeg gdigrab 输入设备
        ↓
avformat_open_input("desktop", ...)
        ↓
avformat_find_stream_info()
        ↓
找到 AVMEDIA_TYPE_VIDEO
        ↓
avcodec_find_decoder()
        ↓
avcodec_open2()
        ↓
循环：av_read_frame() -> avcodec_send_packet() -> avcodec_receive_frame()
        ↓
得到 AVFrame（RGBA）
        ↓
复制到 rgba_frame_
        ↓
外部调用 CaptureFrame() 取出 RGBA 数据
        ↓
后续送给编码器 / 推流
```

可以把它理解为：

- GDI 层负责“抓桌面”
- FFmpeg 层负责“打包转换成标准视频帧”
- 工程层负责“收帧、缓存、再交给编码器推流”

这是一种很典型的桌面录屏采集架构：

- 抓取源：桌面
- 处理器：FFmpeg `gdigrab`
- 解码器：音视频解码框架
- 输出：RGBA 字节流
- 上层：编码、推流、显示

---

### 12. 关键理解

要理解这个采集流程，最关键的几个概念是：

- `gdigrab`：FFmpeg 的 Windows 桌面采集输入源
- `AVFormatContext`：媒体输入上下文
- `AVCodecContext`：解码器上下文
- `AVFrame`：一帧解码后的图像
- `AVPacket`：一包原始媒体数据
- `AV_PIX_FMT_RGBA`：最终目标像素格式
- `CaptureFrame()`：对外提供缓存帧的接口

这几个组件组合起来，就形成了工程里的 GDI 屏幕采集链路。

---

### 13. 一句话总结

这个项目采用的是“FFmpeg + gdigrab + 解码线程”的 GDI 屏幕采集方案：它不直接操作底层 Windows GDI API，而是借助 FFmpeg 将桌面采集封装成标准的视频流，再由工程自己把每一帧 `AVFrame` 转成 RGBA 缓冲区供后续编码和推流使用。
