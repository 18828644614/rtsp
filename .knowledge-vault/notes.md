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
