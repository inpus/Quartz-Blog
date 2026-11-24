# ListInfoWgt：左侧导航列表的实现

## 控件在主窗口中的位置

`ListInfoWgt` 是客户端主窗口左侧的导航区域，顶部放用户按钮，下方放远程控制、设备设置、高级设置和导播工作台四个列表项。它负责创建这些控件、设置导航样式、处理导航选择，并通过 `sig_Select` 通知主窗口切换页面。

主窗口把 `ListInfoWgt` 放在左侧；`sig_Select` 与 `MainWgt::slot_ItemClicked` 相连。页面索引的具体管理见 [MainWgt.md](./MainWgt.md)，主窗口位置见 [ClientStream主窗口布局.md](./ClientStream主窗口布局.md)。

## 控件层级与布局分工

左侧导航大致由以下控件构成：

~~~mermaid
flowchart TD
    A["ListInfoWgt（左侧容器）"] --> V["QVBoxLayout"]
    V --> B["userBtn_：用户 QPushButton"]
    V --> L["listWgt_：ListWidget（继承 QListWidget）"]
    L --> I0["QListWidgetItem：第 0 行"]
    L --> I1["QListWidgetItem：第 1 行"]
    L --> I2["QListWidgetItem：第 2 行"]
    L --> I3["QListWidgetItem：第 3 行"]
    I0 --> C0["CustomWgt：远程控制"]
    I1 --> C1["CustomWgt：设备设置"]
    I2 --> C2["CustomWgt：高级设置"]
    I3 --> C3["CustomWgt：导播工作台"]
    C0 --> H["QHBoxLayout：图标 + 文字"]
~~~

`ListInfoWgt` 的 `QVBoxLayout` 只排列用户按钮和整个列表控件。每个列表项内部的图标和文字，由对应 `CustomWgt` 自己的 `QHBoxLayout` 排列。

## QListWidgetItem 如何装入 CustomWgt

`ListInfoWgt` 创建四个 `CustomWgt`，设置每个控件的文字、普通图标和高亮图标，再把指针存入 `customWgts_`，供点击后更新高亮状态。随后逐个调用 `listWgt_->AddWidget(...)`。

`ListWidget::AddWidget()` 的核心步骤如下：

~~~cpp
QListWidgetItem* listWidgetItem = new QListWidgetItem(this);
listWidgetItem->setSizeHint(widget->sizeHint());
this->setItemWidget(listWidgetItem, widget);
~~~

`QListWidgetItem` 表示列表中的一行，本身不是用来显示图标和文字的 QWidget。`setItemWidget()` 把 `CustomWgt` 设置为该行实际显示的控件。列表控件负责管理多行，行内的显示内容则由 `CustomWgt` 提供。

## CustomWgt 如何组成图标和文字

`CustomWgt` 继承 `QWidget`。它的 `Init()` 创建两个标签：

- `image_`：显示图标的 `QLabel`，尺寸为 `20 × 20`。
- `name_`：显示文字的 `QLabel`，尺寸为 `80 × 20`。

随后将两个标签加入 `QHBoxLayout`，形成横向的“图标 + 文字”控件。`CustomWgt` 固定尺寸为 `180 × 30`。它不是 `QPushButton`，也没有自己的点击信号；行的点击由外层 `ListWidget` 处理。

`setImageAndText()` 保存普通图标和高亮图标的路径，并设置初始图片与文字。`setHightLight(true/false)` 根据状态选择对应图片，最终由 `setPicture()` 加载图片并显示在 `image_` 上。`setPicture()` 将图片缩放到 `image_` 的尺寸；`Qt::KeepAspectRatio` 保持原图宽高比，避免图标变形，`Qt::SmoothTransformation` 让缩放结果更平滑。这里是按标签尺寸缩放图标，不是针对屏幕分辨率或高 DPI 的适配。

## 样式表如何加载和匹配控件

`ListInfoWgt` 构造函数给列表和用户按钮设置了对象名：

~~~cpp
listWgt_->setObjectName("listWidget");
userBtn_->setObjectName("user_Btn");
~~~

随后调用：

~~~cpp
StyleLoader::getInstance().loadStyle(":/UI/brown/main.css", this);
~~~

这里的 `this` 是 `ListInfoWgt`。`StyleLoader` 通过 `QFile` 从 Qt 资源路径 `:/UI/brown/main.css` 读取整份样式表，然后调用 `this->setStyleSheet(qss)`。它不会主动遍历控件并查找对象名；样式表交给 Qt 后，由 Qt 的样式引擎根据选择器、控件类型、`objectName` 和控件状态进行匹配。样式表设置在 `ListInfoWgt` 上，其子控件也会参与匹配。

资源文件 `res.qrc` 将 `UI/brown/main.css` 编入 Qt 资源系统，所以运行时可以用 `:/UI/brown/main.css` 读取。

左侧导航相关的 QSS 规则在 `ClientStreamLive/UI/brown/main.css`：

| 选择器 | 作用 |
| --- | --- |
| `QPushButton#user_Btn` | 匹配对象名为 `user_Btn` 的用户按钮，设置头像图片。 |
| `QListWidget#listWidget` | 匹配对象名为 `listWidget` 的列表控件，设置透明背景和无边框。 |
| `QListView#listWidget::item` | 设置列表项的高度、边距和圆角。`QListWidget` 继承自 `QListView`。 |
| `QListView#listWidget::Item:hover` | 鼠标悬浮在列表项上时，将该项背景设为 `#353535`。 |
| `QListView#listWidget::Item:selected` | 列表项被选中时设置背景和文字颜色。 |

因此，鼠标悬浮背景变化是 Qt 根据 QSS 的 `:hover` 状态自动绘制的，不需要在 `ListWidget.cpp` 中另写鼠标移动事件。该悬浮规则针对列表项背景，不会自动替换 `CustomWgt` 的图标；图标切换由点击处理触发。

## 点击如何变成页面选择

### 顶部用户按钮：进入登录页

顶部用户头像按钮由 `ListInfoWgt` 创建，设置对象名后由 QSS 显示头像。它的 `clicked` 连接直接发送页面索引 0：

~~~cpp
userBtn_ = new QPushButton(this);
userBtn_->setObjectName("user_Btn");

connect(userBtn_, &QPushButton::clicked, this, [this] {
    emit sig_Select(0);
    for (auto item : customWgts_) {
        item->setHightLight(false);
    }
    listWgt_->clearSelection();
});
~~~

该按钮是登录页的导航入口，不属于 `ListWidget` 中的四个列表项。`MainWgt` 的页面索引 0 对应登录页面；`sig_Select(0)` 经信号槽连接到 `MainWgt::slot_ItemClicked()` 后，最终由 `QStackedWidget` 显示该页面。

### 列表项：选择对应页面

点击列表项时，`ListWidget` 先获取行号并发出信号，`ListInfoWgt` 再把行号映射为页面索引：

~~~mermaid
flowchart LR
    A["鼠标左键点击列表项"] --> B["ListWidget::mousePressEvent"]
    B --> C["itemAt 获取行，row 获取索引"]
    C --> D["发出 itemClicked(index)"]
    D --> E["ListInfoWgt::HandleItemClicked"]
    E --> F["发出 sig_Select(index + 1)"]
    E --> G["选中项图标高亮，其余图标恢复普通状态"]
    F --> H["MainWgt 切换到对应页面"]
~~~

在 `ListWidget::mousePressEvent()` 中，左键按下时先用 `itemAt(event->pos())` 找到鼠标所在的列表项，再通过 `row(item)` 得到行号并发出 `itemClicked(index)`。随后调用基类的 `QListWidget::mousePressEvent(event)`。

`ListInfoWgt` 在构造时把 `ListWidget::itemClicked` 连接到 `HandleItemClicked`。槽函数先检查索引范围，再发出 `sig_Select(index + 1)`，并遍历 `customWgts_`：被点击行调用 `setHightLight(true)`，其他行调用 `setHightLight(false)`。行号与页面索引是两套索引。`ListWidget` 中只有四个列表项，行号为 0 到 3；`HandleItemClicked()` 将行号加一，映射为主页面 `QStackedWidget` 的页面索引 1 到 4。页面索引 0 对应登录页。顶部用户按钮位于列表之外，点击时直接发出 `sig_Select(0)`，所以列表里没有登录页对应的 `QListWidgetItem`。

点击顶部用户按钮时，`ListInfoWgt` 直接发出 `sig_Select(0)`，把所有列表图标恢复为普通状态，并调用 `clearSelection()` 清除列表选中状态。`ClientStream` 将 `ListInfoWgt::sig_Select` 连接到 `MainWgt::slot_ItemClicked`。该槽函数用收到的页面索引调用 `stackWgt_->widget(index)` 取得对应页面；页面有效时再调用 `stackWgt_->setCurrentWidget(widget)` 完成切换。

## 职责速查

| 文件 | 负责内容 |
| --- | --- |
| `ClientStreamLive/UI/list/ListInfoWgt.cpp` | 创建并配置导航控件、建立信号连接、更新图标高亮、加载样式表。 |
| `ClientStreamLive/UI/tool/ListWidget.cpp` | 创建 `QListWidgetItem`、将 `CustomWgt` 放入行内、检测鼠标点击并发出行索引。 |
| `ClientStreamLive/UI/tool/CustomWgt.cpp` | 用两个 `QLabel` 和 `QHBoxLayout` 组成行内容，切换普通/高亮图标。 |
| `ClientStreamLive/UI/tool/StyleLoader.cpp` | 从 Qt 资源读取 QSS，并将样式表设置给传入的 QWidget。 |
| `ClientStreamLive/UI/brown/main.css` | 通过控件类型、`objectName` 和 `:hover`、`:selected` 状态定义视觉样式。 |
| `ClientStreamLive/res.qrc` | 将 QSS 和图标资源编入程序。 |

## 一句话总结

`ListInfoWgt` 用垂直布局摆放用户按钮和 `ListWidget`；`ListWidget` 的每一行由 `QListWidgetItem` 承载一个 `CustomWgt`；QSS 依据 `objectName` 和悬浮/选中状态改变列表外观；点击后由 `ListWidget` 发出行索引，`ListInfoWgt` 更新图标状态并通过 `sig_Select` 通知主窗口切换页面。