---
title: Drogon 服务端实现与流程
---

# Drogon 服务端实现与流程

TacticLens 服务端使用 Drogon 承载 HTTP、WebSocket、静态面板和进程生命周期。项目自己的媒体接收、事件总线、感知、语音、VLM 与推荐逻辑由独立模块处理。

- [服务端流程](流程.md)：用 Mermaid 展示启动、请求路由、WebRTC 信令和媒体下游。
- [源码实现](实现.md)：按入口、控制器、线程边界和关停顺序对照代码。

> 本目录说明当前源码结构，不代表服务器此刻正在运行哪个二进制或配置。服务端启动配置可由 <code>TACTICLENS_CONFIG</code> 覆盖；现场运行状态需要另行核对 systemd、监听端口和日志。
