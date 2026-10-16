---
type: topic
status: draft
created: 2026-10-16
updated: 2026-10-16
tags:
  - rtcp
  - rtp
---

# RTCP 协议

## RTCP 定位
- RTCP 与 RTP 的关系
- RTCP 是否承载媒体
- RTCP 报告周期

## 常见包类型
- SR
- RR
- SDES
- BYE
- APP
- NACK
- PLI
- FIR

## Sender Report
- NTP Timestamp
- RTP Timestamp
- 发送包数
- 发送字节数
- 音视频同步

## Receiver Report
- Fraction Lost
- Cumulative Lost
- Extended Highest Sequence Number
- Interarrival Jitter
- LSR/DLSR

## 质量指标
- 丢包率
- 抖动
- 往返时间
- 有效吞吐量
- 端到端延迟

## 同步
- NTP 与 RTP 时间映射
- 音视频共同时间轴
- CNAME

## C++ 实现
- RTCP 头解析
- SR/RR 结构
- 统计周期
- 线程模型
- 与 RTP 接收器的关系

## 练习
- [ ] 解析 SR
- [ ] 解析 RR
- [ ] 计算丢包率和抖动
