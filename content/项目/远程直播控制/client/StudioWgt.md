# StudioWgt：导播工作台构造函数

## StudioWgt 在界面中的位置

`StudioWgt` 是客户端主页面中的导播工作台页面，继承自 `QWidget`。构造函数负责创建预览区、操作按钮、场景设置和直播输出设置，并把这些控件与 `StudioEngine`、场景配置和 RTMP 推流器连接起来。

源码位于 `ClientStreamLive/UI/center/StudioWgt.cpp`。主窗口通过 `MainWgt` 将它加入页面堆栈；用户从左侧导航进入“导播工作台”时，看到的就是这个页面。

构造函数开始时，`engine_` 取得 `StudioEngine::instance()`。后续的画面预览和场景设置都围绕这个引擎进行：控件变化会更新场景，引擎发出的信号也会反过来更新页面。

## 构造了哪些控件

### 页面外框、提示和状态

| 控件 | 用途 |
| --- | --- |
| `StudioWgt`（当前 `QWidget`） | 页面容器。设置对象名 `StudioWgt`、启用样式背景，并固定为 `600 × 510`。 |
| `enabled_`（`QCheckBox`） | 控制是否使用导播画面共享；界面文字说明设置在下次开始共享时生效。 |
| `preview_`（`QLabel`） | 导播工作台预览区域。初始显示操作提示；收到预览帧后显示缩放后的图像。 |
| `status_`（`QLabel`） | 显示导播引擎的状态信息，例如预览、录像等状态。 |
| `publishStatus_`（`QLabel`） | 显示 RTMP 输出状态，例如未推流、连接中或当前码率。 |
| `note`、`layers`、`presetsNote`、`publishNote`（`QLabel`） | 提供采集说明、图层顺序、场景文件和直播输出方面的操作提示。 |

这里的 `QLabel` 不只是显示普通文字。`preview_` 通过 `setPixmap()` 显示引擎送来的画面；其他标签主要用于状态或说明文字，并启用了自动换行。

### 顶部操作按钮

| 控件 | 用途 |
| --- | --- |
| `previewBtn_`（`QPushButton`） | 开始或停止预览。按钮文字会随预览状态切换。 |
| `recordBtn_`（`QPushButton`） | 开始或停止录像。开始时弹出保存文件对话框。 |
| `shot`（`QPushButton`） | 将当前可用的导播画面保存为 PNG 截图。 |
| `folder`（`QPushButton`） | 打开默认录像、截图目录。 |
| `recordAudio_`（`QCheckBox`） | 选择录像时是否录制混音，默认勾选；录像开始后暂时禁用。 |

### “采集与场景”标签页

这一页负责选择主画面来源、摄像头和输出画布，并配置画中画及图像处理选项。

| 控件 | 用途 |
| --- | --- |
| `source_`（`QComboBox`） | 选择主画面来源：屏幕、摄像头、屏幕加摄像头画中画、图片或视频文件。 |
| `camera_`（`QComboBox`） | 选择摄像头设备。 |
| `refresh`（`QPushButton`） | 重新枚举摄像头设备。 |
| `resolution_`（`QComboBox`） | 选择输出画布分辨率，包含横屏、竖屏和 4:3 选项。 |
| `fps_`（`QSpinBox`） | 设置输出帧率，范围为 5–60 fps。 |
| `cameraVisible_`（`QCheckBox`） | 控制摄像头图层是否显示。 |
| `mirror_`（`QCheckBox`） | 控制摄像头画面是否镜像。 |
| `beauty_`（`QCheckBox`） | 控制摄像头柔化效果。 |
| `cameraRect_`（4 个 `QSpinBox`） | 设置摄像头画中画区域的 X、Y、宽度和高度百分比。 |
| `gpuComposition_`（`QCheckBox`） | 请求使用 GPU 合成；不可用时由程序回退到 CPU。 |
| `staticFrames_`（`QCheckBox`） | 启用静态画面时降低编码频率的选项。 |
| `cvSmooth_`、`cvWhiten_`、`cvSharpen_`、`cvContrast_`（`QSpinBox`） | 分别设置磨皮、美白、锐化和对比度。前三项范围是 0–100，对比度范围是 50–150。 |
| `faceTrack_`（`QCheckBox`） | 开启人脸检测和平滑跟随。 |
| `faceSticker_`（`QCheckBox`） | 开启人脸贴纸。 |
| `faceModel_`（`QLineEdit` 加浏览按钮） | 指定 ONNX 人脸模型文件；留空时使用内置 YuNet。 |
| `stickerPath_`（`QLineEdit` 加浏览按钮） | 指定贴纸图片；留空时使用默认皇冠贴纸。 |
| `mediaPath_`（`QLineEdit` 加浏览按钮） | 指定用作主画面的图片或视频文件。 |
| `loop_`（`QCheckBox`） | 控制视频是否循环播放。 |
| `muted_`（`QCheckBox`） | 控制视频是否静音。 |
| `pause`（可切换的 `QPushButton`） | 暂停或继续视频播放。 |
| `restart`（`QPushButton`） | 从头重播视频。 |

`filePicker()` 是构造文件选择行的辅助函数：它将 `QLineEdit` 和“浏览”按钮放进一个水平布局，按钮点击后弹出文件对话框。`rectEditor()` 则构造 X、Y、W、H 四个百分比输入框，用于编辑画面区域。

### “图层与字幕”标签页

| 控件 | 用途 |
| --- | --- |
| `logoVisible_`（`QCheckBox`） | 控制图片或 LOGO 图层是否显示。 |
| `logoPath_`（`QLineEdit` 加浏览按钮） | 选择叠加图片。 |
| `logoRect_`（4 个 `QSpinBox`） | 设置 LOGO 图层的位置和尺寸百分比。 |
| `textVisible_`（`QCheckBox`） | 控制底部字幕是否显示。 |
| `text_`（`QLineEdit`） | 输入字幕内容，最多 500 个字符。 |
| `textSize_`（`QSpinBox`） | 设置字幕字号，范围为 12–160 px。 |
| `textColor_`（`QPushButton`） | 打开颜色选择对话框，选择字幕颜色。 |
| `background_`（`QPushButton`） | 打开颜色选择对话框，选择画布底色。 |
| `layers`（`QLabel`） | 说明图层顺序，以及透明 PNG 等素材用法。 |

### “场景库”标签页

| 控件 | 用途 |
| --- | --- |
| `presets_`（可编辑 `QComboBox`） | 输入或选择场景名称。设置为可编辑，但不允许直接插入新条目；保存操作负责写入场景库。 |
| `save`（`QPushButton`） | 将当前场景按名称保存或覆盖。 |
| `load`（`QPushButton`） | 切换到当前选中的场景。 |
| `remove`（`QPushButton`） | 删除当前选中的场景。 |
| `importBtn`（`QPushButton`） | 从 JSON 文件导入场景。 |
| `exportBtn`（`QPushButton`） | 将当前场景导出为 JSON 文件。 |
| `presetsNote`（`QLabel`） | 说明场景自动保存、导入导出内容和素材文件处理方式。 |

### “直播输出”标签页

| 控件 | 用途 |
| --- | --- |
| `publishUrl_`（`QLineEdit`） | 输入 RTMP 推流地址。构造时从 `QSettings` 读取上次保存的地址；没有保存值时使用 `AppConfig::rtmpUrl(10001)`。 |
| `publishBtn_`（`QPushButton`） | 开始或停止 RTMP 推流；重连期间也用于取消重连。 |
| `publishStatus_`（`QLabel`） | 显示连接、推流和重试状态。 |
| `publishNote`（`QLabel`） | 解释 RTMP 输出与音频设置的关系。 |
| `publishTimer_`（`QTimer`） | 每 200 毫秒检查一次异步连接任务和推流状态；开始连接尝试时启动。 |

## Qt 控件本身的作用和用法

上面各表保留的是“这个变量在导播页面中负责什么”。这一节说明它所使用的 Qt 类本身能做什么，以及构造函数调用的 API 参数怎样影响界面。

### 输入、选择和操作控件

| Qt 类 | Qt 本身提供的作用 | 本页使用的 API、参数和效果 |
| --- | --- | --- |
| `QLabel` | 显示不可直接编辑的文字或图片。 | `QLabel(text)` 用文字初始化；`setText(text)` 更新文字；`setPixmap(pixmap)` 显示图片；`setAlignment(Qt::AlignCenter)` 让预览提示居中；`setFixedHeight(145)` 固定预览区高度；`setWordWrap(true)` 允许说明文字自动换行。 |
| `QPushButton` | 显示可点击的按钮，并通过 `clicked` 信号通知程序。 | `QPushButton(text)` 设置按钮标题。暂停按钮另外调用 `setCheckable(true)`，让它能保持按下/弹起两种状态，并通过 `toggled(bool)` 把状态交给视频暂停逻辑。 |
| `QCheckBox` | 提供一个可勾选/取消的布尔选项。 | `QCheckBox(text)` 同时设置旁边的说明文字；`setChecked(true)` 设置初始勾选状态；`toggled(bool)` 在状态变化时发出新值。本页用它控制共享选项、摄像头属性、字幕显示、视频循环等。 |
| `QLineEdit` | 输入或编辑单行文本。 | `new QLineEdit` 创建单行编辑框；`setPlaceholderText(text)` 设置空内容时显示的提示，不会把提示当成输入值；`setMaxLength(500)` 限制可输入字符数；`setText(text)` 设置实际内容；`editingFinished()` 在编辑结束时通知程序。本页用它填写字幕、文件路径和 RTMP 地址。 |
| `QComboBox` | 从下拉列表中选择一个选项，也可以设置为可编辑。 | `addItems({...})` 加入一组显示文字；`addItem(text, data)` 在显示文字外附带数据，例如分辨率项的 `QSize`；`currentIndexChanged` 通知选择变化；`currentData()` 读取当前选项附带的数据。场景名称框额外调用 `setEditable(true)` 允许输入名称，并用 `setInsertPolicy(QComboBox::NoInsert)` 避免仅输入文字就自动写入选项列表。 |
| `QSpinBox` | 输入有上下限的整数，用户可以键入数值或用箭头增减。 | `setRange(min, max)` 限定允许值；`setSuffix(" fps")` 或 `setSuffix(" px")` 在数字后显示单位；`setPrefix(text)` 在数字前显示标识；`valueChanged` 在数值变化时发出新值。本页用它设置帧率、字号、图像效果强度和位置百分比。 |

### 标签页、滚动区和布局类

| Qt 类/辅助函数 | 本身的职责 | 本页的调用方式和效果 |
| --- | --- | --- |
| `QWidget` | Qt 界面控件的基础容器类型，可以承载子控件和布局。 | `StudioWgt` 继承 `QWidget`，`setFixedSize(600, 510)` 固定页面尺寸；`setObjectName("StudioWgt")` 设置对象名，供样式选择器识别；`setAttribute(Qt::WA_StyledBackground)` 允许控件绘制样式背景；`setStyleSheet(...)` 用 QSS 设置背景、文字、边框和按钮状态样式。 |
| `QTabWidget` | 管理多页界面，一次显示当前选中的页，并提供页签用于切换。 | `new QTabWidget` 创建标签容器；`addTab(page, title)` 把页面控件加入容器并设置页签标题。本页加入“采集与场景”“图层与字幕”“场景库”“直播输出”四页。 |
| `QScrollArea` | 在有限的视口中显示一个子控件；子控件内容超出可视范围时可滚动查看。 | `setWidgetResizable(true)` 让内部页面随视口大小调整；`setFrameShape(QFrame::NoFrame)` 隐藏滚动区边框；`setWidget(body)` 指定要显示和滚动的页面控件。 |
| `QFormLayout` | 按表单形式排列控件，常用于“字段说明 + 输入控件”的设置页。 | `addRow("主画面", source_)` 建立带左侧说明的表单行；`addRow(widget)` 把控件作为整行加入。`setContentsMargins(10, 10, 10, 10)` 设置表单内边距；`setFieldGrowthPolicy(QFormLayout::AllNonFixedFieldsGrow)` 允许非固定大小的输入字段扩展。 |
| `QVBoxLayout` | 沿垂直方向依次排列控件或子布局。 | `new QVBoxLayout(this)` 将根布局安装到当前窗口；随后按顺序加入共享开关、预览区、工具栏、标签页和状态栏。`addWidget(tabs, 1)` 的 `1` 是伸展因子，让标签页在剩余空间中优先扩展。 |
| `QHBoxLayout` | 沿水平方向排列控件或子布局。 | `new QHBoxLayout(parentWidget)` 为指定容器创建水平布局；`addWidget(widget)` 依次加入项目；`setContentsMargins(0, 0, 0, 0)` 去掉容器内边距；`setSpacing(3)` 设置相邻控件的间隔。用于工具栏、摄像头选择行、文件选择行等。 |

### `scrollPage()`：项目封装的滚动表单页

`scrollPage()` **不是 Qt 自带的控件或类名**，而是本项目写的辅助函数。它把 Qt 的 `QScrollArea`、`QWidget` 和 `QFormLayout` 组合起来，再把滚动区域作为一页加入 `QTabWidget`。因此，“滚动页”效果由函数内部创建的 `QScrollArea` 提供。

函数的参数含义如下：

| 参数 | 类型 | 作用 |
| --- | --- | --- |
| `tabs` | `QTabWidget*` | 接收新页面的标签容器。函数最后对它调用 `addTab(scroll, title)`。 |
| `title` | `const QString&` | 新页面显示在标签栏上的标题，例如“采集与场景”。 |
| `form` | `QFormLayout*&` | 对 `QFormLayout*` 的引用。函数创建表单布局后把指针写回调用者，使后续代码可以通过 `form->addRow(...)` 向新页面加控件。 |

这里的 `QFormLayout*&` 可以理解为：把调用者的指针变量按引用传进函数。于是 `form = new QFormLayout(body)` 会把这个指针改为指向新建的布局（口语上可以说“换了指向”）。改变的是指针指向，不是把旧表单的行追加到新表单，也不会删除旧布局。

函数内部的关键步骤可以读成这样：

~~~cpp
auto *scroll = new QScrollArea;
scroll->setWidgetResizable(true);
scroll->setFrameShape(QFrame::NoFrame);

auto *body = new QWidget;
form = new QFormLayout(body);
form->setContentsMargins(10, 10, 10, 10);
form->setFieldGrowthPolicy(QFormLayout::AllNonFixedFieldsGrow);

scroll->setWidget(body);
tabs->addTab(scroll, title);
~~~

例如构造函数中的调用顺序可以简化成：

~~~cpp
QFormLayout *form;

scrollPage(tabs, QStringLiteral("采集与场景"), form); // form 指向第 1 页的布局
form->addRow(...);                                    // 行加到第 1 页
form->addRow(...);

scrollPage(tabs, QStringLiteral("图层与字幕"), form); // form 改为指向第 2 页的布局
form->addRow(...);                                    // 行加到第 2 页
~~~

所以第二次调用时，`form` 的确已经指向一个加过很多行的布局。调用后，调用者的 `form` 改为指向新布局；旧布局仍由它原来的 `body` 持有，而旧 `body` 已经装在第 1 页的 `QScrollArea` 中，因此第 1 页内容保留。第二次调用以后，后续的 `form->addRow(...)` 才会加到第 2 页。若把参数写成普通的 `QFormLayout* form`（不带 `&`），函数只能改自己的局部指针副本，调用者就拿不到新建布局的指针。

### 其他 Qt API 和辅助类

| Qt 类 | Qt 本身的作用 | 本页使用方式 |
| --- | --- | --- |
| `QFileDialog` | 提供系统文件打开或保存对话框。 | `getOpenFileName(parent, caption, directory, filter)` 接收父窗口、标题、起始目录和文件类型筛选，返回选择的路径；用户取消时返回空字符串。`getSaveFileName(...)` 用于选择保存路径。 |
| `QColorDialog` | 提供颜色选择对话框。 | `getColor(initialColor, parent)` 用给定颜色作为初始值并显示对话框；返回颜色调用 `isValid()` 检查，用户取消时不应用更改。 |
| `QTimer` | 在指定时间间隔发出 `timeout()` 信号，用于定时执行检查或更新。 | `new QTimer(this)` 把定时器设为当前窗口的子对象；`setInterval(200)` 将间隔设为 200 毫秒；连接 `timeout` 到 `pollPublish()`，推流尝试开始后启动定时器。重连延迟使用 `QTimer::singleShot(delay, this, callback)` 安排一次性回调。 |
| `QMediaDevices` | 查询系统媒体设备，并在设备列表变化时发出信号。 | `videoInputs()` 用来读取摄像头输入列表；`videoInputsChanged` 连接到 `refreshCameras()`，以便设备变化后刷新下拉框。 |
| `QSettings` | 按组织名、应用名和键保存/读取应用设置。它不是界面控件。 | `QSettings("YunZhuJiao", "ClientStreamLive")` 指定配置归属；`value(key, default)` 读取值并可提供默认值；`setValue(key, value)` 保存 RTMP 地址或场景库 JSON。 |
| `QFile`、`QSaveFile` | `QFile` 提供文件打开和读取；`QSaveFile` 用于写入文件并在提交时完成保存。 | 导入场景时用 `QFile` 只读打开并读取 JSON；导出时用 `QSaveFile` 写入所有 JSON 字节，检查 `commit()` 是否成功。 |
| `QJsonDocument` | 在 JSON 文本/字节和 Qt JSON 对象之间解析、序列化。 | `fromJson(bytes, &error)` 解析文件并返回错误信息；`toJson()` 将当前场景序列化，场景库保存时可指定 `QJsonDocument::Compact` 输出紧凑格式。 |

### 本页自定义的控件组合函数

除了 Qt 类，构造函数还调用了几个本项目的辅助函数。它们不是 Qt 原生 API，但内部使用 Qt 控件来组合界面：

| 函数 | 参数 | 返回/输出 | 组合后的效果 |
| --- | --- | --- | --- |
| `scrollPage(tabs, title, form)` | 目标 `QTabWidget`、页签标题、表单布局指针引用 | 把新建的表单布局写入 `form`，并返回页面 `QWidget*` | 创建一个可滚动的标签页表单。 |
| `filePicker(edit, owner, filter)` | `QLineEdit*&` 用于接收新建的输入框；`owner` 是文件对话框父窗口；`filter` 是文件筛选字符串 | 返回包含输入框和浏览按钮的行控件 | 点击浏览按钮弹出文件选择框，选中的路径写回输入框。 |
| `rectEditor(values)` | `std::array<QSpinBox*, 4>&` 用于接收四个输入框指针 | 返回包含四个百分比输入框的行控件 | 按 X、Y、W、H 编辑画面位置和尺寸。 |
| `effectSpin(title, low, high)` | 表单标题、最小值、最大值 | 返回新建的 `QSpinBox*` | 创建有范围的图像效果数值框，加入当前表单，并将数值变化连接到 `apply()`。 |

`filePicker()` 的实际调用例如 `filePicker(mediaPath_, this, QStringLiteral("素材 (*.png *.jpg ...);;所有文件 (*)"))`：第一个参数让函数把创建好的 `QLineEdit*` 写到 `mediaPath_`，第二个参数 `this` 作为文件对话框的父窗口，第三个参数限定对话框里可选的文件类型。筛选字符串用 `;;` 分隔多个选项，所以用户可以在素材类型和“所有文件”之间切换。

## 控件如何布局

构造函数先设置窗口对象名、背景绘制属性、固定尺寸和 QSS 样式。随后创建页面的根 `QVBoxLayout`，设置边距为左、右 12 像素，上 8 像素、下 8 像素，控件之间留 5 像素间距。

根垂直布局从上到下依次放置：共享开关、预览区、操作工具栏、标签页和状态栏。预览区高度固定为 145 像素；标签页调用 `addWidget(tabs, 1)`，会占用剩余空间。

操作工具栏使用 `QHBoxLayout`，横向排列开始预览、开始录像、截图、录制混音和打开目录。这样一组常用操作集中在预览区下方。

`QTabWidget` 把多组设置组织为四个可切换页面：

1. **采集与场景**：采集设备、画布参数、图像效果、视频播放。
2. **图层与字幕**：LOGO、字幕和画布颜色。
3. **场景库**：保存、切换、删除、导入和导出场景。
4. **直播输出**：RTMP 地址、推流控制和推流状态。

每次调用 `scrollPage(tabs, 标题, form)` 都会创建一个独立的 `QScrollArea`、页面 `QWidget` 和 `QFormLayout`，再把滚动区加入标签页。因此内容较多时可以在对应标签内滚动。`QFormLayout` 将控件按“说明文字 + 输入控件”的表单行排列；调用 `addRow(widget)` 时则添加不带左侧说明文字的整行控件。每页表单设置 10 像素内容边距，并允许非固定字段扩展。

表单中有些控件需要并排显示，因此再套一层水平布局：摄像头下拉框旁放“刷新”按钮；分辨率旁放帧率；摄像头图层选项放在同一行；视频循环、静音、暂停和重播并排显示；场景库的保存/切换/删除以及导入/导出按钮也分别组成横向按钮组。文件选择行和画面区域编辑器同样由辅助函数用 `QHBoxLayout` 组合。

整体层级如下：

~~~mermaid
flowchart TD
    A[StudioWgt 页面] --> B[QVBoxLayout 根布局]
    B --> C[共享开关 enabled_]
    B --> D[QLabel 预览区 preview_]
    B --> E[QHBoxLayout 操作工具栏]
    E --> E1[预览与录像按钮]
    E --> E2[截图与目录按钮]
    E --> E3[录制混音选项]
    B --> F[QTabWidget 标签页]
    F --> G1[采集与场景]
    F --> G2[图层与字幕]
    F --> G3[场景库]
    F --> G4[直播输出]
    G1 --> H1[QScrollArea]
    G2 --> H2[QScrollArea]
    G3 --> H3[QScrollArea]
    G4 --> H4[QScrollArea]
    H1 --> I1[QFormLayout 表单行]
    H2 --> I2[QFormLayout 表单行]
    H3 --> I3[QFormLayout 表单行]
    H4 --> I4[QFormLayout 表单行]
    B --> J[状态栏 status_]
~~~

## `connect()` 如何把控件用途接起来

`connect(sender, signal, receiver, slot)` 把信号发送者、发生的事件和接收处理函数关联起来。构造函数中的连接大致分为三类：用户操作调用页面槽函数或 lambda；表单变化调用 `apply()` 更新场景；引擎信号更新预览和控件状态。

### 预览、录像和素材播放

| 信号 | 处理函数或行为 | 结果 |
| --- | --- | --- |
| `previewBtn_` 的 `clicked` | lambda 调用 `engine_->setPreview(...)` | 切换预览状态，并更新按钮文字；停止预览时把 `preview_` 改为“预览已停止”。 |
| `engine_` 的 `frameReady` | lambda 检查 `engine_->previewing()` 后缩放画面 | 将图像转换成 `QPixmap`，显示在 `preview_` 中并保持宽高比。 |
| `engine_` 的 `statusChanged` | `QLabel::setText` | 将引擎状态显示到 `status_`。 |
| `recordBtn_` 的 `clicked` | `StudioWgt::toggleRecord` | 启动或停止录像。 |
| `shot` 的 `clicked` | `StudioWgt::snapshot` | 保存引擎当前帧。 |
| `folder` 的 `clicked` | 打开 `defaultFolder()` 对应的本地目录 | 快速查看录像和截图文件。 |
| `refresh` 的 `clicked` | `StudioWgt::refreshCameras` | 重新读取摄像头列表。 |
| `QMediaDevices::videoInputsChanged` | `StudioWgt::refreshCameras` | 系统摄像头设备列表变化时自动刷新。 |
| `pause` 的 `toggled` | `StudioEngine::setVideoPaused` | 暂停或继续视频。 |
| `restart` 的 `clicked` | `StudioEngine::restartVideo` | 重播视频。 |
| `engine_` 的 `recordingChanged` | lambda 更新录像按钮和混音选项 | 录像时按钮显示“停止录像”，并禁用 `recordAudio_`；然后调用 `sync()` 更新场景控件。 |

### 场景设置控件

会影响 `StudioScene` 的控件连接到 `StudioWgt::apply()`。包括采集来源、摄像头、分辨率、帧率、摄像头属性、画中画区域、图像效果、人脸处理、素材路径、视频播放选项、LOGO 和字幕设置等。

`apply()` 从控件读取当前值，写入 `StudioScene`，最后调用 `engine_->setScene(s)`。`engine_` 的 `sceneChanged` 信号则连接到 `StudioWgt::sync()`：场景变化时把值回填到各控件。`sync()` 使用 `updating_` 标记避免回填控件时再次触发一轮场景更新。

磨皮、美白、锐化、对比度四个 `QSpinBox` 在创建时就将 `valueChanged` 连接到 `apply()`。其余选项按控件类型成组连接：复选框用 `toggled`，下拉框用 `currentIndexChanged`，数值框用 `valueChanged`，文本字段用 `editingFinished`。

字幕颜色和画布底色按钮会先打开 `QColorDialog`。用户确认了有效颜色后，构造函数中的 lambda 更新颜色值并调用 `apply()`；取消对话框时不改变场景。

### 场景库和 RTMP 输出

| 信号 | 处理函数或行为 | 结果 |
| --- | --- | --- |
| `save` 的 `clicked` | `StudioWgt::savePreset` | 保存当前场景到场景库设置。 |
| `load` 的 `clicked` | lambda 检查当前场景 JSON 后调用 `engine_->setScene(...)` | 将所选场景应用到导播引擎。 |
| `remove` 的 `clicked` | lambda 删除当前名称、写回 `QSettings` 并刷新列表 | 删除场景库中的场景。 |
| `importBtn` 的 `clicked` | `StudioWgt::importPreset` | 打开文件选择框并导入场景 JSON。 |
| `exportBtn` 的 `clicked` | `StudioWgt::exportPreset` | 将当前场景导出为 JSON。 |
| `publishBtn_` 的 `clicked` | `StudioWgt::togglePublish` | 校验 RTMP 地址并开始/停止推流。 |
| `publishTimer_` 的 `timeout` | `StudioWgt::pollPublish` | 轮询异步连接结果、连接状态、码率和重试流程。 |
| `PushSettingsStore::changed` | lambda 调用 `publisher_->applySettings(settings)` | 推流器已创建且连接任务不在执行时，将最新推流设置应用给推流器。 |

构造函数末尾从 `QSettings` 读取保存的场景 JSON，调用 `refreshPresets()` 填充场景下拉框，调用 `refreshCameras()` 枚举摄像头，再调用 `sync()` 将引擎当前场景显示到界面上。

主要信号关系可以概括为：

~~~mermaid
flowchart LR
    Controls[场景表单控件] -->|复选框、下拉框、数值和文本变化| Apply[StudioWgt::apply]
    Apply -->|写入 StudioScene| Engine[StudioEngine]
    Engine -->|sceneChanged| Sync[StudioWgt::sync]
    Sync --> Controls

    PreviewButton[开始预览按钮] -->|clicked| PreviewToggle[setPreview 切换预览]
    Engine -->|frameReady| PreviewLabel[QLabel preview_ 显示画面]
    Engine -->|statusChanged| StatusLabel[QLabel status_ 显示状态]

    RecordButton[录像按钮] -->|clicked| Record[StudioWgt::toggleRecord]
    ScreenshotButton[截图按钮] -->|clicked| Snapshot[StudioWgt::snapshot]
    SceneButtons[场景库按钮] -->|clicked| PresetHandlers[保存、切换、删除、导入、导出]
    PresetHandlers --> Engine

    PublishButton[开始推流按钮] -->|clicked| TogglePublish[StudioWgt::togglePublish]
    TogglePublish --> Publisher[RTMP 推流管理器]
    PublishTimer[200 ms 定时器] -->|timeout| Poll[StudioWgt::pollPublish]
    Poll --> Publisher
    PushSettings[PushSettingsStore 配置变化] -->|changed| Publisher
~~~

## 构造函数之外的函数实现流程

本节按 `StudioWgt.cpp` 中函数的职责说明调用顺序和每一步的结果。匿名命名空间里的辅助函数也列在这里；它们负责准备目录或组合控件，类成员函数则负责更新场景、保存素材和管理输出。

### 控件和路径辅助函数

#### `defaultFolder()`

1. 调用 `QStandardPaths::writableLocation(QStandardPaths::MoviesLocation)` 取得系统“视频/影片”目录。
2. 在该路径后拼接 `ClientStreamLive` 子目录。
3. 调用 `QDir().mkpath(path)` 创建目录（目录已存在时无需重复创建）。
4. 返回最终路径，供录像、截图和“打开目录”按钮使用。

#### `scrollPage(tabs, title, form)`

这个函数在前面的 [Qt 控件本身的作用和用法](#qt-控件本身的作用和用法) 一节说明了参数。实际执行顺序是：创建无边框 `QScrollArea` → 创建内部 `QWidget` → 在内部页面上创建 `QFormLayout` 并设置边距和扩展策略 → 将页面装入滚动区 → 用 `tabs->addTab(scroll, title)` 注册为一个标签页 → 返回内部页面指针。构造函数主要使用输出参数 `form` 添加表单项，返回的页面指针没有单独保存。

#### `filePicker(edit, owner, filter)`

1. 创建行容器和 `QHBoxLayout`，去掉布局边距。
2. 创建 `QLineEdit` 并通过引用参数 `edit` 把指针交给调用方；再创建“浏览”按钮并将宽度限制为 55 像素。
3. 把输入框和按钮放入水平布局，然后连接按钮的 `clicked` 信号。
4. 点击按钮后用 `QFileDialog::getOpenFileName(owner, ..., filter)` 打开文件选择框。`owner` 作为对话框父窗口，`filter` 决定可选文件类型。
5. 如果用户选出了非空路径，先写入输入框，再显式发出 `editingFinished()`，使这个路径像用户完成文本编辑一样进入已有的 `apply()` 连接。
6. 返回整行容器，调用者把它作为一项加入表单。

#### `rectEditor(values)`

创建行容器和水平布局，然后依次创建四个 `QSpinBox`，保存到 `values[0..3]`。四个输入框分别加上 `X `、`Y `、`W `、`H ` 前缀和 `%` 后缀；位置 X/Y 的范围为 0–100，宽高 W/H 的范围为 5–100。函数把它们横向加入布局并返回行容器。`sync()` 和 `apply()` 通过同一组数组指针读写这些数值。

#### `effectSpin(title, low, high)` 局部 lambda

它不是独立成员函数，而是在构造函数里创建效果滑动数值项的局部工厂：创建 `QSpinBox` → 用 `low`、`high` 设置范围 → 用 `title` 将它加入当前 `QFormLayout` → 将 `valueChanged` 连接到 `apply()` → 返回输入框指针。磨皮、美白、锐化和对比度四个控件都由它创建。

### 场景数据和界面同步

#### `sync()`：从引擎场景刷新控件

1. 将 `updating_` 设为 `true`，表示当前是程序回填控件，避免控件变化信号反过来触发 `apply()`。
2. 从 `engine_->scene()` 取得当前 `StudioScene`，将共享开关、采集类型、摄像头、分辨率、帧率、画面效果、人脸设置、素材路径、播放选项、LOGO 和字幕等字段写回对应控件。
3. 如果场景保存的摄像头 ID 或分辨率当前列表里没有，就临时添加一个离线设备项或该分辨率项，再选中它，确保保存的设置仍可见。
4. 将场景中的矩形坐标乘以 100 并四舍五入，填入 X/Y/W/H 百分比输入框；同时回填颜色值。
5. 只有在既不推流/共享也不录像时，才允许修改分辨率和帧率；否则禁用这两个控件。
6. 最后把 `updating_` 设回 `false`，恢复用户操作引起的更新。

#### `apply()`：从控件写回引擎场景

1. 如果 `updating_` 为 `true`，说明控件正在由 `sync()` 回填，立即返回。
2. 先复制当前 `engine_->scene()`，再从界面读取各控件的值，更新对应的 `StudioScene` 字段。先取现有场景可保留页面控件没有编辑的其他字段。
3. 对摄像头画中画和 LOGO 的四个百分比值除以 100，构造成 `QRectF` 的归一化坐标。
4. 调用 `engine_->setScene(s)` 将新的场景交给 `StudioEngine`。场景发生变化后，引擎发出 `sceneChanged`，连接回 `sync()`，再把最终场景显示在控件中。

#### `refreshCameras()`：重新枚举摄像头

1. 先设置 `updating_ = true`，并从当前场景保存摄像头 ID。
2. 清空 `camera_`，先添加“系统默认摄像头”，其附带数据为空 `QByteArray`。
3. 调用 `QMediaDevices::videoInputs()`，逐个把设备描述作为显示文字、设备 ID 作为附带数据加入下拉框。
4. 如果原来保存的设备 ID 不在当前设备列表中，额外加入“已保存设备（当前离线）”，避免丢失场景里的设备选择。
5. 根据原 ID 选中匹配项；没有匹配项时选择索引 0，即系统默认摄像头。
6. 最后清除 `updating_` 标记。

### 录像和截图

#### `toggleRecord()`

1. 如果引擎已经在录像，调用 `engine_->stopRecording()` 后返回。
2. 如果当前没有录像，用 `defaultFolder()` 和当前时间生成 `record_年月日_时分秒_毫秒.mp4` 默认文件名。
3. 调用 `QFileDialog::getSaveFileName(...)` 让用户确认路径，过滤类型为 MP4。
4. 用户没有取消时调用 `engine_->startRecording(path, recordAudio_->isChecked())`；第二个参数决定是否录制混音。

#### `snapshot()`

1. 调用 `engine_->latestFrame()` 取得最近一帧。
2. 如果帧为空，更新 `status_` 提示先开始预览、共享或录像，然后返回。
3. 用 `defaultFolder()` 和时间戳生成 PNG 默认文件名，再打开 PNG 保存对话框。
4. 用户选定路径后调用 `frame.save(path)`；根据保存结果在 `status_` 显示成功路径或失败提示。取消对话框时不写文件。

### 场景库

#### `refreshPresets()`

先保存 `presets_` 当前编辑文字，再清空下拉框，将 `savedScenes_` 的键重新加入选项，最后把刚才的文字写回编辑区。这样刷新列表时不会把用户正在输入或选择的名称清掉。

#### `savePreset()`

1. 读取当前场景名称，去掉首尾空白，并截取最多 80 个字符。
2. 名称为空时在 `status_` 提示用户输入名称并返回。
3. 调用 `engine_->scene().toJson()` 把当前场景转成 JSON，按名称放入 `savedScenes_`。
4. 将整个场景对象序列化为紧凑 JSON，通过 `QSettings` 写到 `studio/presets`。
5. 刷新场景下拉框，重新选中刚保存的名称，并更新状态提示。

#### `importPreset()`

1. 打开 JSON 文件选择框；用户取消时返回。
2. 以只读方式打开文件，并拒绝超过 1 MiB 的文件；打开失败或文件过大时显示错误。
3. 读取文件内容并解析为 `QJsonDocument`。
4. 检查解析错误、顶层是否为对象、`version` 是否为 1，以及是否包含 `source` 字段；任一检查失败就显示无效场景提示并返回。
5. 验证通过后调用 `StudioScene::fromJson(...)` 转成场景并交给 `engine_->setScene(...)`，提示用户可以再命名保存到场景库。导入本身不会自动写入命名场景库。

#### `exportPreset()`

1. 打开 JSON 保存对话框，默认文件名为 `scene.json`；取消时返回。
2. 用 `engine_->scene().toJson()` 取得当前场景，再序列化为 JSON 字节。
3. 通过 `QSaveFile` 打开目标文件、写入全部数据并提交；只有打开、完整写入和提交都成功才视为成功。
4. 根据结果更新 `status_`。`QSaveFile` 的提交方式可避免把未完整写入的临时内容当成最终文件。

### RTMP 推流和重连

#### `togglePublish()`：处理开始、停止和取消

1. 如果异步打开任务仍有效，暂不处理新的点击，避免连接尚未完成时重复操作。
2. 如果已经请求推流或推流器存在，就将 `publishDesired_` 设为 `false`，递增重连代数，调用 `stopPublisher()` 清理资源，并恢复按钮和地址输入框，最后显示已停止。
3. 否则从 `publishUrl_` 读取并去除首尾空格，解析为 `QUrl`。
4. 要求协议为 `rtmp`、主机非空、路径至少含两个非空段；不符合时显示地址格式提示并返回。
5. 将当前场景的 `enabled` 设为 `true` 并提交给引擎；同时把 RTMP 地址保存到 `QSettings`。
6. 设置推流意图，清零重试次数并更新重连代数，然后调用 `startPublishAttempt()`。

#### `startPublishAttempt()`：创建一次连接任务

1. 只有推流意图仍然有效且当前没有连接任务时才继续，否则直接返回。
2. 创建 `RtmpPushManager`，指定当前 `StudioEngine`，并读取 `PushSettingsStore` 的最新推流参数。
3. 禁用地址输入和开始按钮，显示“正在连接服务器…”。
4. 把地址复制到局部变量，通过 `std::async(std::launch::async, ...)` 在后台执行 `publisher_->Open(address)`，结果保存在 `publishTask_`。
5. 启动 200 毫秒轮询定时器；后续由 `pollPublish()` 读取异步结果。

#### `pollPublish()`：轮询连接和推流状态

1. 如果连接任务有效，用 `wait_for(0)` 非阻塞检查是否完成；还没完成就返回，让 UI 线程继续处理界面事件。
2. 任务完成后取出 `Open()` 结果并启用按钮。连接失败时读取 `lastError()`，交给 `retryPublish()`，然后返回；连接成功时把按钮改为“停止推流”，记录建立连接的时间。
3. 没有可用推流器或连接未打开时返回。
4. 检测到链路断开时读取错误原因；如果错误文本为空则使用默认断开提示，然后调用 `retryPublish()` 并返回。
5. 如果已经持续推流超过 10 秒，将连续重试计数清零。
6. 更新 `publishStatus_`：推流中显示码率（kbps）和丢帧数；服务端已连通但尚未推送画面时显示准备状态。

#### `retryPublish(reason)`：执行有限次数的退避重连

1. 先调用 `stopPublisher()` 清除当前连接，再重新启用按钮。
2. 如果当前已不希望推流，或重试次数递增后超过 4 次，就清除推流意图、恢复“开始推流”和地址输入状态，并显示结束原因。
3. 否则按 `500u << (publishRetries_ - 1)` 计算延迟，依次为 500、1000、2000、4000 毫秒；按钮改为“取消重连”，状态文字显示延迟和次数。
4. 用 `QTimer::singleShot()` 安排下一次连接，同时捕获当前 `retryGeneration_`。
5. 定时器到期时，只有推流意图仍有效且代数没有变化才调用 `startPublishAttempt()`。用户停止推流会改变代数，因此旧的重连回调不会再启动连接。

#### `stopPublisher()`

`stopPublisher()` 如果有推流器就调用 `Stop()`，随后释放推流器对象；再将连接状态清零、停止轮询定时器并清除连接成功时间。停止、重试都会复用这段清理逻辑。

#### `~StudioWgt()`

页面销毁时先取消推流意图并递增重连代数，再停止轮询定时器；如果后台 `Open()` 任务仍有效就等待它结束，最后停止推流器。这样对象销毁后不会再由有效的重连流程继续操作页面。

RTMP 相关函数的主流程如下：

~~~mermaid
flowchart TD
    A[点击开始推流] --> B[togglePublish 校验地址并保存]
    B --> C[startPublishAttempt 创建推流器]
    C --> D[后台异步 Open]
    D --> E[200 ms 定时器调用 pollPublish]
    E --> F{连接成功}
    F -->|是| G[检查链路并刷新推流状态]
    F -->|否或链路断开| H[retryPublish]
    H --> I{重试次数不超过4且仍需推流}
    I -->|是| J[按500 1000 2000 4000 ms退避]
    J --> C
    G --> E
    I -->|否| K[停止并恢复按钮与地址]
    G --> L[点击停止]
    L --> M[stopPublisher 清理资源]
~~~

## 一句话总结

`StudioWgt` 用 `QVBoxLayout` 排列预览和主操作，用 `QTabWidget` 分组组织四类设置，每页再由可滚动区域和 `QFormLayout` 管理表单；`connect()` 将表单变化送到 `apply()` 更新 `StudioEngine`，再把引擎的画面、状态和场景变化反馈到预览区与控件。
