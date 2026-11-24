# OCR 已识别点位但热身阶段没有推荐

## 涉及文件

- `server/libs/perception/hud_ocr.cpp`、`hud_ocr.hpp`：字段解析和可用性判断。
- `server/libs/perception/perception.cpp`：独立 OCR 观测与 GameState 输入。
- `config/perception.yaml`：点位 ROI。
- `panel/clips-live.js`：点位置信度、有效期及死亡/观战过滤。
- `panel/panel-live.js`：视频查询。

## 问题

截图里的“A 点大道”已经被 RapidOCR 识别出来，网页却没有推荐。热身画面缺少比分或倒计时，也容易出现同样情况。

我看了 `frame_00000244.jpg` 对应输出，点位不是没读到，而是因为缺少 `score_t` 被整体标成 unavailable。继续调 OCR 识别率并不能解决后面的拦截。

## 修改前

只读推荐复用了完整回合的判断条件。下面是原有完整回合条件：

```cpp
bool HudOcr::hasFormalRound(const nlohmann::json& fields) {
    return fields.contains("score_ct") && fields.contains("score_t") &&
           fields.contains("round_time_left");
}
```

原来的发布逻辑可简化为：

```cpp
const bool available = HudOcr::hasFormalRound(ocr_out.fields);
// available 为 false，就把 OCR 观测设为 unavailable。
// 点位存在，但缺一个比分字段，也进不了推荐查询。
```

对于正式 GameState，完整比分和回合信息有用途；但视频检索只是按当前位置找素材，不应该额外要求这些字段齐全。

## 修改后

我把“点位观测有效”单独列出来：

```cpp
bool HudOcr::hasPointObservation(const nlohmann::json& fields) {
    const auto area = fields.find("self_area");
    return area != fields.end() && area->is_string() &&
           !area->get_ref<const std::string&>().empty() && *area != "unknown";
}
```

在组装处分别计算两种条件：

```cpp
const bool ocr_formal_round = ocr_ &&
    HudOcr::hasFormalRound(ocr_out.fields);
const bool ocr_lookup_ready = ocr_formal_round ||
    (ocr_ && HudOcr::hasPointObservation(ocr_out.fields));

// 独立观测使用 lookup 条件。
const json observation = {
    {"type", "ocr_observation"},
    {"status", ocr_lookup_ready ? "published" : "unavailable"},
    {"formal_round", ocr_formal_round},
    {"fields", ocr_out.fields},
    {"confidence", ocr_out.confidence}
    // 会话、帧序号与时间等字段省略。
};

// 只有完整回合才继续进入原来的 GameState 字段聚合。
if (ocr_formal_round) {
    for (auto it = ocr_out.fields.begin(); it != ocr_out.fields.end(); ++it)
        payload[it.key()] = it.value();
    payload["round"] = ocr_out.fields["score_t"].get<int>() +
                       ocr_out.fields["score_ct"].get<int>() + 1;
}
```

published 也不等于直接播放。前端仍检查 OCR 点位置信度、有效期与显式死亡/观战状态，选点的核心条件包括：

```javascript
if (fields.hp === 0 || fields.spectating === true || fields.spectating === 1)
  return {state: "blocked", reason: "当前为死亡或观战状态"};

if (ocr?.status === "published" && typeof area === "string" &&
    area.trim() && area !== "unknown" &&
    Number.isFinite(confidence) && confidence >= 0.45) {
  return {state: "ready", source: "ocr", area, input: {ocr},
          session_id: this.session};
}
```

点位还通过统一区域词典归一化。1440×1080 下的 ROI 同时调为 `(60,267,200,36)` 并放大 3 倍，覆盖长名称；ROI 调整改善输入覆盖，发布条件调整解决已识别结果被拦截，两者分别起作用。

## 为什么这样改

我没有把所有条件一起放宽，而是让两个用途各自使用需要的条件：只读检索消费可靠点位，GameState 继续要求正式回合信息。

这样比分缺失只影响依赖比分的功能，不会让独立存在的当前位置一起失效。无效点位、低置信度、死亡/观战和过期数据仍然不能用于推荐。

修复过程中曾保留“点位+倒计时”的条件，随后进一步取消了只读查询对热身、倒计时和比分的依赖，以上代码按最后的实现记录。

## 验证

2026-10-25，隔离 OCR 验证“A 点大道”“短梯”正常。无倒计时热身“短梯”存档帧回放自动命中 1、2、9、19 号视频；切到“隧道上方”命中 4、5、6，道具筛选和无匹配清空也完成浏览器验证。这轮使用真实存档帧回放。

## 总结

我先确认模型已经识别到点位，再沿发布和消费路径找到错误门控，最后把只读点位查询从完整 GameState 条件里拆出来。感知字段是否可用应该看具体用途，不能因为某个业务需要完整状态，就让所有下游都依赖完整状态。

## 相关记录

- `docs/当前实现与双端对齐.md`：点位 OCR、跨阵营推荐及后续门控调整。
- `docs/道具推荐阶段验收与后续任务.md`：交付规则。
