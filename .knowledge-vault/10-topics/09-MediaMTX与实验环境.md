---
type: topic
status: draft
created: 2026-10-16
updated: 2026-10-16
tags:
  - mediamtx
  - testing
  - rtsp
---

# MediaMTX 与实验环境

## MediaMTX 定位
- RTSP Server
- 推流与拉流
- 多协议分发
- 录制与回放

## 测试源
- 电脑摄像头
- 手机摄像头
- 本地视频文件
- FFmpeg 生成测试流

## 测试拓扑
- 单机回环
- 局域网
- 跨设备
- 网络异常模拟

## 配置项
- RTSP 监听端口
- 路径
- 鉴权
- TCP/UDP
- 日志
- 录制

## 验证工具
- VLC
- FFplay
- FFmpeg
- Wireshark
- MediaMTX 日志

## 实验记录
- [ ] 基础 RTSP 拉流
- [ ] TCP interleaved
- [ ] UDP RTP/RTCP
- [ ] 认证
- [ ] 断网重连
- [ ] 多客户端
