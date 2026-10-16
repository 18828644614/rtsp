---
type: topic
status: draft
created: 2026-10-16
updated: 2026-10-16
tags:
  - sdp
  - rtsp
---

# SDP 会话描述

## SDP 定位
- SDP 解决什么问题
- SDP 在 DESCRIBE 响应中的位置
- 会话级与媒体级描述

## 基本语法
- type=value
- CRLF
- 字段顺序
- 未知字段处理

## 会话级字段
- v=
- o=
- s=
- t=
- a=control:*

## 媒体级字段
- m=
- c=
- a=rtpmap
- a=fmtp
- a=control
- a=recvonly/sendonly/sendrecv

## 视频轨道
- H.264
- H.265
- Payload Type
- Clock Rate
- SPS/PPS

## 音频轨道
- AAC
- G.711
- Opus
- 采样率与声道

## URL 解析
- 绝对 control URL
- 相对 control URL
- 基础 RTSP URL
- trackID

## C++ 解析器设计
- 行解析
- 当前媒体段
- 数据结构
- 参数 Map
- 错误处理

## 练习
- [ ] 从样例 SDP 提取视频轨道
- [ ] 处理音视频多轨道
- [ ] 生成 SETUP URL
