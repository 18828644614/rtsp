---
type: topic
status: draft
created: 2026-10-16
updated: 2026-10-16
tags:
  - rtp
  - rtsp
---

# RTP 协议

## RTP 定位
- RTP 解决什么问题
- RTP 与 UDP/TCP 的关系
- RTP 与 RTSP 的关系

## RTP 固定头
- Version
- Padding
- Extension
- CSRC Count
- Marker
- Payload Type
- Sequence Number
- Timestamp
- SSRC

## Sequence Number
- 包顺序
- 丢包检测
- 乱序处理
- 16 位回绕
- RTP 包序号与视频帧编号的区别

## Timestamp
- 媒体时间
- 视频 90000 时钟
- 音频时钟
- 同一帧多个包的时间戳
- Timestamp 回绕

## Payload Type
- 静态 Payload Type
- 动态 Payload Type
- SDP 映射

## SSRC
- 同步源
- 多路媒体
- SSRC 变化处理

## RTP 接收器
- UDP 接收
- TCP interleaved 接收
- 缓冲
- 重排
- 丢包统计

## RTP 安全与边界
- 长度校验
- 版本校验
- 非法头部
- 异常 Payload

## 练习
- [ ] 解析 RTP 固定头
- [ ] 检测丢包和乱序
- [ ] 计算 RTP 时间间隔
