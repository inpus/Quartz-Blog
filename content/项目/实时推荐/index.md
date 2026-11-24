---
title: 实时推荐项目笔记
---

# 实时推荐项目笔记

TacticLens 面向 CS2 场景，将 Windows 画面与双路音频采集、Linux 感知推理、SQLite 道具视频库和 Web 副屏接成实时推荐链路，并扩展雷达位置可视化与战术分析。

问题笔记：[[项目/实时推荐/出现的问题/index|开发中遇到的问题]]。

每篇记录我遇到的问题、定位过程、修改前后的代码、改动原理和验证结果，最后用“总结”整理重点。源码路径以项目仓库根目录为起点：Windows 为 `D:/csevent`，服务器为 `/home/lth/csevent`。

LangGraph 回合复盘的[流程与实现笔记](LangGraph/index.md)按现行源码拆分了 Runtime 调度、StateGraph 节点、checkpoint 和 shadow 发布路径。

道具视频的[模块流程与实现笔记](道具视频/index.md)记录素材入库、OCR/VLM 查询、T/CT 候选合并和浏览器 Range 播放。
