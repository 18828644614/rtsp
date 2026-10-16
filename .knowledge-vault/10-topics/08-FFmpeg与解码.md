---
type: topic
status: draft
created: 2026-10-16
updated: 2026-10-16
tags:
  - ffmpeg
  - cplusplus
  - decoding
---

# FFmpeg 与解码

## FFmpeg 组件
- libavformat
- libavcodec
- libavutil
- libswscale
- libavdevice

## 输入方式
- 直接使用 FFmpeg 拉 RTSP
- 自己实现 RTSP 后交给解码器
- AVIOContext 的适用场景

## 解码流程
- 查找解码器
- 创建 Codec Context
- 发送 Packet
- 接收 Frame
- 刷新解码器
- 释放资源

## 像素格式
- YUV420P
- NV12
- RGB
- swscale

## 线程与队列
- 网络线程
- 解包线程
- 解码线程
- 渲染线程
- 背压和丢帧策略

## 资源管理
- AVPacket
- AVFrame
- AVCodecContext
- RAII 封装
- 退出顺序

## 练习
- [ ] 使用 FFmpeg 解码本地 H.264
- [ ] 使用 FFmpeg API 拉取 RTSP
- [ ] 将解码帧转换为 RGB
