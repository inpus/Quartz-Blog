# OCR 抖动导致推荐视频反复重载

## 涉及文件

- `panel/clips-live.js`：观测去重、来源选择和有效期。
- `panel/panel-live.js`：稳定位置、道具确认、请求版本和结果签名。
- `panel/panel.js`：播放器接线。
- `tools/test_panel_live.mjs`：漏识别、重复帧、换区和迟到响应测试。

## 问题

人还站在同一区域，视频却可能闪一下、重新加载或从头开始。OCR 单帧漏识别、低置信度、误读区域，以及 OCR/VLM 之间切换，都会引起输入变化，但这些变化不一定意味着应该更换视频。

我把问题拆成三层：观测怎么确认、请求结果是否还有效、播放器是否需要更新。

## 修改前

原有问题可以概括为下面的直接驱动方式，函数名为示意：

```javascript
async function onObservation(observation) {
  if (!observation.area) {
    clearVideos();
    return;
  }
  const clips = await queryClips(observation.area);
  renderVideos(clips);  // 相同结果也可能重建元素、重新设置 src。
}
```

它有三个后果：漏一帧就清空；错一帧就换区；相同列表反复设置媒体源。若 A 区请求慢、B 区请求快，A 还可能在 B 之后返回，把画面换回旧区域。

## 修改后

### 1. 用不同帧确认换区

第一次可靠位置可以建立状态；已有位置切到另一区域时，先记 candidate，两个不同帧确认后才更新 stable。当前 `selectStable()` 的换区分支如下：

```javascript
const row = this.positions[selected.source];
const evidence = row.message.seq;
const next = {...selected, seenAt: row.received};

if (!this.stable || selected.area === this.stable.area ||
    now - this.stable.seenAt >= this.positions.maxAgeMs) {
  this.stable = next;
  this.candidate = null;
} else {
  if (this.candidate?.area !== selected.area ||
      now - this.candidate.seenAt >= 2500) {
    this.candidate = {
      area: selected.area, evidence, seenAt: row.received, count: 1
    };
  } else if (evidence !== this.candidate.evidence) {
    this.candidate = {...this.candidate, evidence,
      seenAt: row.received, count: this.candidate.count + 1};
  }
  if (this.candidate.count >= 2) {
    this.stable = next;
    this.candidate = null;
  }
}
```

不能按 tick 次数增加 count，因为一次观测可能被重复读取；OCR 和 VLM 读取同一帧也只能算一次证据。seq 用来区分帧，candidate 超过 2.5 秒则重新确认。

单帧漏识别时，最近有效位置最多保留 4 秒。确认死亡/观战、停止、断线或超期仍要清空。OCR/VLM 来源切换但 area 相同，不算换区。手持道具也按不同 OCR 帧确认，不能用库存栏代替当前选中道具。

### 2. 相同结果不操作播放器

`show()` 比较当前可见列表的 ID 和 URL：

```javascript
show(clips, recommendations = []) {
  const visible = clips.slice(0, this.limit);
  const signature = JSON.stringify(visible.map(c => [c.clip_id, c.url]));
  if (signature === this.signature) return;
  this.signature = signature;
  this.render(visible, recommendations);
}
```

10 秒周期刷新或重复观测即使触发查询，只要最终列表没变，也不重新 render。签名保留列表顺序，因此槽位顺序变化仍然会触发更新。

### 3. 请求取消与版本校验一起用

失效时取消请求、递增版本并清空结果：

```javascript
invalidate() {
  this.version++;
  this.pending?.abort();
  this.pending = null;
  this.key = "";
  this.show([]);
}
```

发起查询时保存本次版本，结果回来后再判断。下面是当前逻辑的压缩写法：

```javascript
const current = ++this.version;
const controller = new AbortController();
this.pending?.abort();
this.pending = controller;

// 请求使用 signal: controller.signal；此处省略 T/CT 两路查询和合并。
const results = await requestResults(controller.signal);
if (current !== this.version) return;
this.show(mergeClips(results));
```

`requestResults`、`mergeClips` 是为了缩短示例抽出的名字，实际实现在 query 内。采集停止还会记录 stoppedSessions，拦住停止会话迟到的观测，防止它再发起一轮新查询。

## 为什么这样改

双帧确认处理观测噪声，4 秒有效期限制旧状态能保留多久，结果签名避免重复操作媒体源，版本号处理异步响应乱序。这几层不能互相替代。

AbortController 只负责尽量停止无用请求；请求可能已经完成或结果已进入后续代码，所以是否能修改 UI 最终由版本号决定。播放器 loop 自己回到开头则是正常循环，要和设置 src 引起的重载分别观察。

## 验证

2026-10-25，105 帧回放中插入漏识别和低置信度帧，浏览器连续约 21.939 秒没有额外 loadeddata 换片事件。另验证了换区、道具过滤、无匹配和失效清空。

2026-10-26 的整轮前端 25 项测试包含相同结果不重载、迟到响应、媒体停止、OCR 优先和端口解析，均通过。

## 总结

我没有直接拿每条识别结果控制播放器，而是先确认稳定位置和道具，再查询，最后比较真正要显示的视频列表。异步结果必须证明自己仍属于当前版本，才能更新页面。这样既减少误识别闪切，也避免相同视频反复重载和旧请求回填。

## 相关记录

- `docs/道具推荐阶段验收与后续任务.md`：播放稳定性。
- `docs/双路实时识别与网页推荐验收.md`：前端验证。
