# Notes: SDP 会话描述章节

## Verified structure
- 目标文件：10-topics/04-SDP会话描述.md
- 相关章节：10-topics/03-RTSP协议.md、05-RTP协议.md、07-H264与RTP封装.md、10-C++客户端架构.md
- 知识库要求：保留 YAML frontmatter，补充 wikilinks，完成后运行 vault.py refresh/check。

## Technical scope
- SDP 会话级/媒体级字段、CRLF 行结束、payload type、RTP 映射、fmtp、control URL。
- RTSP DESCRIBE 响应中的 Content-Type 与 Content-Length。
- 视频：H.264/H.265；音频：AAC/G.711/Opus。
- C++17 无第三方依赖解析器，覆盖字段解析、媒体段、属性 Map、URL 解析、错误处理。
- Windows PowerShell/MSVC 与 Linux g++ 的分开编译运行命令。

## References to cite in chapter
- RFC 8866：当前 SDP 语法规范，已下载官方文本核对；替代 RFC 4566。
- RFC 4566：SDP 基本语法与字段定义。
- RFC 2326 / RFC 7826：SDP 在 RTSP DESCRIBE/SETUP 流程中的位置。
- RFC 3551：RTP/AVP payload type 与时钟约定。
- RFC 6184：H.264 over RTP，sprop-parameter-sets/profile-level-id。
- RFC 7798：H.265 over RTP。
- RFC 3640：MPEG-4 generic/AAC fmtp。
- RFC 3551、RFC 4733、RFC 7587：G.711、电话事件与 Opus 相关约定。

## 本轮核对要点（2026-10-19）
- RFC 2326 附录 C.1.1：Content-Base > Content-Location > DESCRIBE 请求 URL；会话级 control:* 继承完整基准 URL。
- RFC 2326 C.1.2：RTSP 单播 SDP 的 m= 端口 0 可表示服务器无端口偏好，不能直接按 SDP offer/answer 的拒绝媒体处理。
- fmtp 必须在第一个等号处分割参数，保留 Base64 尾部的 =；未知属性与未知顶层字段的处理规则不同。
- 示例离线读取 SDP，避免依赖摄像头或重复实现上一章 Socket 客户端；保留纯 SDP 与 RTSP 完整响应的输入边界。

---

# Notes: RTP 零基础学习章节（2026-10-20）

## 范围与来源
- 目标：10-topics/05-RTP协议.md；原正文为提纲，含重复 frontmatter，保留首个有效属性块后重写正文。
- 前置衔接：01-网络与Socket基础、02-音视频基础、03-RTSP协议、04-SDP会话描述。
- 后续衔接：06-RTCP协议、07-H264与RTP封装、08-FFmpeg与解码。
- 规范：RFC 3550 §5、§6.4.1、附录 A；RFC 3551；RFC 2326 §10.12；RFC 6184 §5.1。
- 本机可用 Windows MinGW g++；不预先声称 MSVC 或真实摄像头测试通过。

## 内容与验证设计
- 12 字节固定头、CSRC、扩展、Padding；动态 PT 依赖 SDP；时间戳不是毫秒或收包时间。
- 序号回绕、乱序、重复、重启与 SSRC 分流；重排窗口与等待截止时间。
- 完整通用 RTP 结构解析器；本机 UDP 合成数据；TCP 交织按长度组帧，模拟拆分与合并。
- 测试需包含截短头、CSRC/扩展长度越界、非法 padding、零长度 payload、PT/Marker 分离、回绕和交织半包。

## 本轮核对与实测结果
- 已读取RFC 3550、3551、2326、6184官方文本，核对头布局、扩展长度、Padding、序号源验证、RTSP交织与H.264 Marker规则。
- 从最终章节代码块提取rtp_common.hpp、rtp_demo.cpp、rtp_udp.cpp、interleaved_demo.cpp到系统临时目录验证，不向项目引入应用程序代码。
- Windows MinGW-w64 g++ 15.2.0，C++17，三个程序编译通过，无警告。
- 离线字段与输出一致；UDP真实本机收发通过，畸形数据丢弃、用法错误和接收超时通过；TCP半包、多包及载荷含0x24通过。
- 215项边界与组帧断言通过，包含全部41种单次TCP切分位置、逐字节输入、可变RTP头和非法长度。
- 本轮未运行MSVC、Wireshark图形界面、摄像头/MediaMTX、真实RTSP文本混合接收、媒体解码；章节明确区分作者实测和读者步骤。
