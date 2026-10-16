---
type: topic
status: draft
created: 2026-10-16
updated: 2026-10-16
tags:
  - network
  - socket
  - tcp
  - udp
---

# 网络与 Socket 基础

## 分层模型
- OSI 与 TCP/IP
- 应用层、传输层、网络层
- RTSP、RTP、RTCP 所在层次

## IP 与端口
- IPv4/IPv6
- 单播、组播、广播
- 端口与服务

## TCP
- 连接建立与关闭
- 字节流语义
- 半包与粘包
- 超时、Keep-Alive、断线
- 阻塞与非阻塞

## UDP
- 数据报语义
- 丢包、乱序、重复
- RTP 为什么常使用 UDP

## C++ Socket
- Winsock 初始化与清理
- Socket 生命周期
- send/recv 与错误码
- select/poll/IOCP 的选择

## RTSP 项目中的应用
- RTSP 控制连接
- RTP/RTCP 媒体连接
- TCP interleaved

## 练习
- [ ] TCP Echo Client/Server
- [ ] UDP 收发程序
- [ ] TCP 半包组包程序
