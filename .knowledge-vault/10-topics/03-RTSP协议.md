---
type: topic
status: draft
created: 2026-10-16
updated: 2026-10-16
tags:
  - rtsp
  - protocol
---

# RTSP 协议

## 协议定位
- RTSP 解决什么问题
- RTSP 与 HTTP 的相似点和差异
- RTSP 与 RTP/RTCP 的关系

## RTSP 消息格式
- 请求行
- 状态行
- Header
- 空行
- 消息体
- CRLF
- Content-Length

## 常用方法
- OPTIONS
- DESCRIBE
- SETUP
- PLAY
- PAUSE
- GET_PARAMETER
- TEARDOWN
- ANNOUNCE、RECORD 的适用场景

## 状态码
- 2xx
- 3xx
- 4xx
- 5xx
- 摄像头常见错误

## 拉流状态机
- 初始状态
- DESCRIBE 状态
- SETUP 状态
- PLAYING 状态
- PAUSED 状态
- TEARDOWN 状态
- 异常与重连

## Session 与 CSeq
- CSeq 的作用
- Session 的作用
- TCP 连接与 RTSP Session 的关系
- Session timeout

## Transport 协商
- RTP/AVP UDP
- RTP/AVP/TCP
- unicast 与 multicast
- client_port、server_port
- interleaved 通道

## 认证
- Basic
- Digest
- 认证挑战
- Authorization
- 认证失败处理

## 保活
- OPTIONS 保活
- GET_PARAMETER 保活
- timeout
- 不同设备的差异

## 规范与兼容性
- RTSP 1.0
- RTSP 2.0
- 厂商扩展
- 常见兼容性问题

## 练习
- [ ] 手写 OPTIONS 请求
- [ ] 手写 DESCRIBE 请求
- [ ] 记录一次完整 RTSP 交互
