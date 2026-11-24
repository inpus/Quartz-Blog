---
title: LangGraph 流程与实现
---

# LangGraph 流程与实现

本文记录 TacticLens Python Agent 的 LangGraph 回合复盘工作流。流程入口、异步调度和证据包构建由 Agent Runtime 负责；LangGraph 只编排回合结束后的复盘、校验、失败降级与提交。

- [回合复盘流程](流程.md)：从回合结束、证据冻结到复盘结果写入的完整运行路径。
- [源码实现](实现.md)：按 `review.py`、`runtime.py`、`context.py` 对照节点、状态、重试和 checkpoint。

源码位于 `D:/csevent/agent/tactical_agent/`。服务器工作树可能有独立修改；本次记录已核对服务器与 Windows 两端相关源码哈希一致，并分别检查了工作树状态。运行证据与代码事实分开说明，详见[流程页](流程.md#现场部署检查点)。
