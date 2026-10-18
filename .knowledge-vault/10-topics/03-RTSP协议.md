---
type: topic
status: draft
created: 2026-10-16
updated: 2026-10-18
tags:
  - rtsp
  - protocol
  - beginner
  - cplusplus
  - windows
  - linux
aliases:
  - RTSP 入门
  - Real Time Streaming Protocol
---

# RTSP 协议

> 适用对象：会 C++ 基础，但没有接触过 RTSP、RTP、SDP 或摄像头协议的初学者。
>
> 本章目标：能看懂一次 RTSP 拉流过程，能手写并发送 OPTIONS、DESCRIBE 请求，理解 SETUP/PLAY 的状态变化，并知道如何继续实现一个 C++ 客户端。
>
> 本章代码用于学习协议和验证连接。它没有实现完整鉴权、SDP 解析、RTP 重组、解码、重连和生产级超时策略。完成本章后，请继续阅读 [[04-SDP会话描述]]、[[05-RTP协议]]、[[06-RTCP协议]] 和 [[10-C++客户端架构]]。

## 0. 先规划：这一章到底要学什么

### 0.1 学完后的能力目标

完成本章后，你应该能够：

- 用自己的话解释 RTSP、RTP、RTCP、SDP 各自负责什么。
- 说清楚 RTSP 为什么像 HTTP，又为什么不能把它当成 HTTP。
- 读懂请求行/状态行、Header、空行、消息体和 CRLF。
- 按正确顺序解释 OPTIONS → DESCRIBE → SETUP → PLAY → TEARDOWN。
- 理解 CSeq、Session、Transport、Content-Length 的作用。
- 区分 UDP 传输、TCP interleaved 传输和 RTSP 控制连接。
- 看懂常见状态码，并知道 401、404、454、461、503 应该先查什么。
- 使用 Windows 和 Linux 编译运行最小 RTSP C++ 客户端。
- 遇到“连接成功但没有画面”时，按层次排查，而不是盲目重试。

### 0.2 建议学习顺序

不要一上来背方法名。每一站都按“概念 → 报文 → 运行 → 故障 → 练习 → 检查”完成。

| 站点 | 学习内容 | 前置 | 动手产出 | 通过标准 |
| --- | --- | --- | --- | --- |
| 第 0 站 | 前置知识和实验环境 | C++、终端、IP/端口 | 编译一个空程序 | 知道 TCP 连接和字节流 |
| 第 1 站 | RTSP 定位和全景 | 第 0 站 | 画 RTSP/RTP/RTCP/SDP 关系 | 说清四者边界 |
| 第 2 站 | 消息格式 | 字符串、换行、TCP | 手写 OPTIONS | 找出 Header 结束位置 |
| 第 3 站 | 方法和状态码 | 第 2 站 | 读 OPTIONS/DESCRIBE | 解释成功或失败原因 |
| 第 4 站 | Session、CSeq、状态机 | 第 3 站 | 画拉流状态图 | 指出每一步前置条件 |
| 第 5 站 | Transport 协商 | UDP/TCP 基础 | 比较 UDP 和 interleaved | 解释端口和通道号 |
| 第 6 站 | 认证、保活、兼容性 | Header、状态码 | 记录一次 401 | 说明 nonce 和 timeout |
| 第 7 站 | C++ 实现 | 前面全部 | 运行 Windows/Linux 示例 | 收到 DESCRIBE 响应 |
| 第 8 站 | 综合练习和验收 | 前面全部 | 抓包和故障分析 | 独立分析一条会话 |

推荐节奏：每站 45～90 分钟。只有完成检查清单，才进入下一站。

### 0.3 本章范围和后续范围

**本章必须掌握：**

- RTSP 控制面和 RTP/RTCP 媒体面的区别。
- RTSP 文本消息格式和 TCP 字节流读取。
- 典型拉流方法顺序。
- CSeq、Session、Transport、Content-Length。
- 状态机、鉴权、保活和常见失败原因。
- 一个能发出请求并收响应的最小 C++ 客户端。

**后续章节深入：**

- SDP 字段和相对 control URL：[[04-SDP会话描述]]。
- RTP 固定头、序号、时间戳和丢包：[[05-RTP协议]]。
- RTCP 的 SR/RR、同步和质量统计：[[06-RTCP协议]]。
- H.264 的 NAL、FU-A、STAP-A：[[07-H264与RTP封装]]。
- 解码、线程、队列和重连：[[08-FFmpeg与解码]]、[[10-C++客户端架构]]、[[11-调试与性能优化]]。

---

## 1. 前置知识：先把地基补齐

### 1.1 你需要的 C++ 知识

本章示例只使用标准 C++ 和系统 Socket API。你至少要能看懂：

- `int main(int argc, char* argv[])`：读取命令行参数。
- `std::string`：保存主机名、URL 和报文。
- `std::vector<char>`：保存接收到的字节。
- 函数返回值和错误处理。
- while 循环、条件判断和简单字符串查找。
- 编译器命令，以及如何检查进程退出码。

暂时不要求会模板元编程、线程池、异步 IO 或 FFmpeg C API。先把一次同步请求做通，再扩展架构。

### 1.2 你需要的网络知识

RTSP 客户端会接触以下四类网络概念：

| 对象 | 通俗解释 | RTSP 中的用途 |
| --- | --- | --- |
| IP 地址 | 找到哪台机器 | 摄像头、服务器或本机 |
| 端口 | 找到机器上的哪个服务 | RTSP 常见默认端口是 554，也可能是 8554、10554 |
| TCP 连接 | 有顺序的字节流 | 传 RTSP 请求/响应，也可承载 interleaved RTP |
| UDP 数据报 | 一封封独立的信 | 常用来传 RTP/RTCP 媒体 |

TCP 的关键特点是：它只保证字节顺序，不保证一次 recv() 就得到完整的一条 RTSP 响应。一次响应可能分成多次收到，也可能一次收到多条消息。这个问题叫半包和粘包，完整背景见 [[01-网络与Socket基础]]。

### 1.3 先理解 URL

一个常见 RTSP URL：

~~~text
rtsp://user:password@192.168.1.20:554/live/main
~~~

可以拆成：

- rtsp：协议方案。
- user:password@：可选的用户名和密码；不要把真实密码写入知识库或日志。
- 192.168.1.20：主机地址。
- 554：端口；省略时通常使用 554。
- /live/main：资源路径，由服务器定义。

注意：/live/main 是资源地址，不是一定代表“主码流”。每个厂商的路径不同；不要仅凭路径猜摄像头型号。

### 1.4 阶段检查清单

- [ ] 我能解释 IP 和端口的区别。
- [ ] 我知道 TCP 是字节流，recv() 不等于一条完整消息。
- [ ] 我能从 RTSP URL 找出主机、端口和资源路径。
- [ ] 我知道 RTSP 端口不一定是 554。
- [ ] 我能在终端编译并运行一个只有 main() 的 C++ 程序。

---

## 2. RTSP 的定位：它像“遥控器”，不是“视频文件”

### 2.1 一句话理解

**RTSP（Real Time Streaming Protocol）负责建立、控制和关闭一个实时媒体会话。**

可以把一场直播想成四个角色：

| 角色 | 它解决的问题 | 常见内容 |
| --- | --- | --- |
| RTSP | “我要播放哪一路？开始、暂停、结束了吗？” | OPTIONS、DESCRIBE、SETUP、PLAY |
| SDP | “这一场流有哪些轨道？编码和参数是什么？” | H.264、采样率、control URL |
| RTP | “音视频包如何编号和计时？” | Sequence Number、Timestamp、Payload |
| RTCP | “传输质量怎样？两端时钟如何对应？” | Sender Report、Receiver Report |

典型链路：

~~~text
客户端 ──RTSP/TCP──> 摄像头或流媒体服务器
客户端 <──SDP─────── 服务器告诉媒体轨道和编码参数
客户端 ──SETUP─────> 协商每条轨道如何传输
客户端 ──PLAY──────> 请求开始发送媒体
客户端 <──RTP/RTCP── 通过 UDP 或 TCP interleaved 到达
客户端 ──TEARDOWN──> 结束会话
~~~

RTSP 通常不直接承载 H.264 视频帧。它控制会话，真正的媒体通常由 RTP 搬运。

### 2.2 RTSP 和 HTTP 的相似点

两者都常见：

- 文本请求和响应。
- 请求方法、Header、状态码。
- CRLF（回车换行）。
- 基于 TCP 的连接。

例如：

~~~text
DESCRIBE rtsp://example/live RTSP/1.0
CSeq: 2
Accept: application/sdp

~~~

### 2.3 RTSP 和 HTTP 的差异

不要把 RTSP 当成“视频版 HTTP”：

1. RTSP 方法和状态机不同：SETUP、PLAY、TEARDOWN 是媒体会话语义。
2. RTSP 的 Session 标识媒体会话；HTTP 常见的是无状态请求。
3. RTSP 的媒体数据通常另走 RTP/RTCP。
4. 一个 RTSP 会话可能包含多个媒体轨道，每条轨道都要 SETUP。
5. 服务器可能长时间保持连接，并要求客户端保活。
6. 同一个 TCP 连接中可能交替出现 RTSP 文本和 $ 开头的 RTP/RTCP 二进制帧。

### 2.4 RTSP 1.0 和 RTSP 2.0

- RTSP 1.0 由 RFC 2326 定义，摄像头和常见流媒体服务器中仍非常普遍。
- RTSP 2.0 由 RFC 7826 定义，改进了语法、错误处理和会话管理。
- 现实设备往往声称支持某个版本，却保留厂商扩展或只实现其中一部分。

本章示例使用 RTSP 1.0 风格，原因是它最容易在真实设备上验证。响应中的 RTSP/1.0 不代表媒体一定使用 RTP/AVP/UDP；还要看 Transport。

### 2.5 阶段练习

1. 用一句话分别解释 RTSP、SDP、RTP、RTCP。
2. 为什么 RTSP 连接成功仍可能没有视频画面？
3. 如果只拿到一个 RTP 包，为什么不能立即把它当成一张图片？
4. 画出“摄像头 → RTSP 协商 → RTP → 解码 → 显示”的箭头图。

### 2.6 阶段检查清单

- [ ] 我知道 RTSP 主要负责控制会话。
- [ ] 我知道 SDP 描述媒体，RTP 传媒体，RTCP 做报告和同步。
- [ ] 我能说出 RTSP 与 HTTP 的至少三个差异。
- [ ] 我知道 RTSP 1.0 是真实设备中的常见兼容目标。

---

## 3. RTSP 消息格式：先学会读一条报文

### 3.1 请求的结构

~~~text
方法 URI 版本 CRLF
Header-1: value CRLF
Header-2: value CRLF
CRLF
可选消息体
~~~

示例：

~~~text
OPTIONS rtsp://192.168.1.20:554/live/main RTSP/1.0\r\n
CSeq: 1\r\n
User-Agent: BeginnerRtspClient/0.1\r\n
\r\n
~~~

这里的 \r\n 是两个字节：

- \r：回车，ASCII 13。
- \n：换行，ASCII 10。

Header 结束的标志是连续两个 CRLF，也就是 \r\n\r\n。不要用单独的 \n\n 代替。

### 3.2 响应的结构

响应第一行是状态行：

~~~text
RTSP/1.0 200 OK\r\n
CSeq: 1\r\n
Public: OPTIONS, DESCRIBE, SETUP, TEARDOWN, PLAY, PAUSE\r\n
\r\n
~~~

有消息体时通常会有 Content-Length：

~~~text
RTSP/1.0 200 OK\r\n
CSeq: 2\r\n
Content-Type: application/sdp\r\n
Content-Length: 174\r\n
\r\n
v=0\r\n
o=- 1 1 IN IP4 192.168.1.20\r\n
s=Camera\r\n
...
~~~

读取策略：

1. 循环 recv()，直到找到 \r\n\r\n。
2. 解析状态行和 Header。
3. 读取 Content-Length 指定的消息体字节数。
4. 同一次 recv() 多收了下一条数据时，保留多余字节。
5. 不能以某次 recv() 返回 0 字节作为普通消息结束；TCP 连接关闭才会返回 0。

### 3.3 Content-Length 的意义

Content-Length 表示消息体的字节数，不是字符数、不是 SDP 行数。收到 SDP 时要按这个长度读取；发送 DESCRIBE 通常没有请求体。

### 3.4 常见格式错误

| 错误 | 结果 |
| --- | --- |
| 只写 \n | 有的服务器能容忍，有的服务器一直等待完整请求 |
| 忘记最后的空行 | 服务器不知道 Header 结束，可能超时 |
| Content-Length 写成字符数 | 客户端提前结束或继续等待 |
| 把多次 recv() 当成多条消息 | SDP 被截断，下一条消息被误解析 |
| 把 send() 一次发送完想当然 | TCP 发送也可能只发送一部分 |
| Header 名后不写冒号 | 服务器可能返回 400 |

### 3.5 手写一条 OPTIONS

~~~text
OPTIONS rtsp://192.168.1.20:554/live/main RTSP/1.0
CSeq: 1
User-Agent: BeginnerRtspClient/0.1

~~~

真正通过 Socket 发送时，行尾必须是 \r\n，最后还要再加一个 \r\n。

### 3.6 阶段检查清单

- [ ] 我能找到 \r\n\r\n。
- [ ] 我理解 Content-Length 是字节数。
- [ ] 我知道发送和接收都要处理部分读写。
- [ ] 我能手写符合格式的 OPTIONS 和 DESCRIBE。

---

## 4. 常用方法：一次拉流到底发生了什么

下面按典型的“客户端拉流”顺序讲解。真实设备可能省略 OPTIONS、改变顺序或要求额外 Header，但核心思想不变。

### 4.1 OPTIONS：询问服务器支持什么

请求：

~~~text
OPTIONS rtsp://192.168.1.20:554/live/main RTSP/1.0
CSeq: 1
User-Agent: BeginnerRtspClient/0.1

~~~

成功响应常见为：

~~~text
RTSP/1.0 200 OK
CSeq: 1
Public: OPTIONS, DESCRIBE, SETUP, PLAY, PAUSE, TEARDOWN

~~~

用途：

- 检查 RTSP 服务是否可达。
- 观察服务器声明支持哪些方法。
- 作为某些设备的保活请求。
- 发现服务器是否需要认证。

Public 是服务器声明支持的方法列表，不保证每个方法在所有 URL 或所有状态下都能成功。

### 4.2 DESCRIBE：索要 SDP

请求：

~~~text
DESCRIBE rtsp://192.168.1.20:554/live/main RTSP/1.0
CSeq: 2
Accept: application/sdp
User-Agent: BeginnerRtspClient/0.1

~~~

成功响应通常包含：

~~~text
RTSP/1.0 200 OK
CSeq: 2
Content-Base: rtsp://192.168.1.20:554/live/main/
Content-Type: application/sdp
Content-Length: 312

v=0
o=- 1 1 IN IP4 192.168.1.20
s=Media Server
t=0 0
a=control:*
m=video 0 RTP/AVP 96
a=rtpmap:96 H264/90000
a=control:trackID=0
m=audio 0 RTP/AVP 97
a=rtpmap:97 MPEG4-GENERIC/48000/2
a=control:trackID=1
~~~

先记住：

- m=video 和 m=audio 表示媒体轨道。
- a=rtpmap 告诉你编码名称、时钟频率和音频声道。
- a=control 给出后续 SETUP 使用的轨道地址。
- SDP 的详细语法见 [[04-SDP会话描述]]。

### 4.3 SETUP：为每条轨道协商传输方式

对于视频轨道，客户端通常发送：

~~~text
SETUP rtsp://192.168.1.20:554/live/main/trackID=0 RTSP/1.0
CSeq: 3
Transport: RTP/AVP;unicast;client_port=5000-5001
User-Agent: BeginnerRtspClient/0.1

~~~

服务器可能响应：

~~~text
RTSP/1.0 200 OK
CSeq: 3
Session: 9A7F23B1;timeout=60
Transport: RTP/AVP;unicast;client_port=5000-5001;server_port=6970-6971;ssrc=12345678
~~~

关键点：

- client_port=5000-5001：客户端希望接收 RTP 和 RTCP 的端口。
- server_port=6970-6971：服务器发送 RTP 和 RTCP 的端口。
- Session：服务器分配的媒体会话标识。
- 第二条轨道的 SETUP 通常必须带上前一条响应里的 Session。
- 多条轨道通常共享一个 RTSP Session，但各自有独立的 RTP/RTCP 流。

### 4.4 PLAY：请求开始发送媒体

~~~text
PLAY rtsp://192.168.1.20:554/live/main/ RTSP/1.0
CSeq: 5
Session: 9A7F23B1
Range: npt=0.000-
User-Agent: BeginnerRtspClient/0.1

~~~

成功响应可能包含：

~~~text
RTSP/1.0 200 OK
CSeq: 5
Session: 9A7F23B1
RTP-Info: url=trackID=0;seq=3456;rtptime=123456,url=trackID=1;seq=7890;rtptime=654321

~~~

RTP-Info 可帮助客户端把 RTSP 播放动作和 RTP 初始序号、时间戳联系起来。RTP 时间轴和同步见 [[05-RTP协议]]、[[06-RTCP协议]]。

### 4.5 PAUSE、GET_PARAMETER、TEARDOWN

PAUSE 通常保留 Session，让客户端稍后再次 PLAY：

~~~text
PAUSE rtsp://192.168.1.20:554/live/main/ RTSP/1.0
CSeq: 6
Session: 9A7F23B1

~~~

GET_PARAMETER 常被用于保活：

~~~text
GET_PARAMETER rtsp://192.168.1.20:554/live/main/ RTSP/1.0
CSeq: 7
Session: 9A7F23B1
Content-Length: 0

~~~

也有设备只接受 OPTIONS。保活方法不是所有服务器都一致，必须观察设备文档或抓包。

TEARDOWN 结束会话：

~~~text
TEARDOWN rtsp://192.168.1.20:554/live/main/ RTSP/1.0
CSeq: 8
Session: 9A7F23B1

~~~

成功后客户端应停止接收线程、关闭 UDP Socket、关闭 RTSP TCP 连接并清理 Session。网络断开时 TEARDOWN 可能发不出去，但本地资源仍要释放。

### 4.6 ANNOUNCE 和 RECORD

- ANNOUNCE 常用于把 SDP 发布给服务器，或更新会话描述。
- RECORD 请求服务器接收客户端发送的媒体。
- 它们更多用于推流或录制，不是普通摄像头拉流的第一组方法。
- Public 中出现 RECORD 不代表可以直接推流；还需要知道媒体编码、RTP 发送、认证和服务器配置。

### 4.7 方法练习

1. 把一次拉流写成 6 行箭头：OPTIONS、DESCRIBE、两次 SETUP、PLAY、TEARDOWN。
2. 为什么有两条轨道时通常要发两次 SETUP？
3. 哪个响应第一次给出 Session？
4. 如果 DESCRIBE 响应没有 Content-Type: application/sdp，你会怎么处理？
5. PLAY 之前为什么通常必须先完成 SETUP？

### 4.8 阶段检查清单

- [ ] 我能解释 OPTIONS、DESCRIBE、SETUP、PLAY、PAUSE、TEARDOWN。
- [ ] 我知道 DESCRIBE 的响应通常包含 SDP。
- [ ] 我知道每条媒体轨道通常都要 SETUP。
- [ ] 我知道 SETUP 之后要保存 Session。
- [ ] 我能说明 PLAY 成功后媒体通常从 RTP 到达。

---

## 5. 状态码：看到数字先定位问题层次

### 5.1 按类别理解

| 类别 | 含义 | 例子 |
| --- | --- | --- |
| 2xx | 请求成功 | 200 OK、201 Created |
| 3xx | 重定向或进一步位置 | 301、302 |
| 4xx | 客户端请求有问题或没有权限 | 400、401、404、454、455、461 |
| 5xx | 服务器暂时或内部失败 | 500、503 |

### 5.2 常见 RTSP 状态码

- 200 OK：当前请求成功。
- 400 Bad Request：报文格式、Header 或 URI 有问题。
- 401 Unauthorized：需要认证，响应通常带 WWW-Authenticate。
- 404 Not Found：路径不存在；检查 URL、尾部斜杠和厂商路径。
- 405 Method Not Allowed：方法不被支持。
- 454 Session Not Found：Session 不存在、过期或 Header 写错。
- 455 Method Not Valid in This State：方法顺序不对，例如未 SETUP 就 PLAY。
- 461 Unsupported Transport：服务器不接受你提出的 Transport。
- 500 Internal Server Error：服务器内部错误。
- 503 Service Unavailable：服务忙、资源不足或暂时不可用。

不同厂商可能使用非标准状态码或在响应体中写错误信息，所以要同时记录状态行、Header 和完整消息体。

### 5.3 按排查层次处理错误

收到错误时按以下顺序排查：

1. 网络层：DNS、路由、防火墙、端口是否可达。
2. RTSP 格式层：方法、URI、CRLF、CSeq、Header。
3. 认证层：是否需要 Basic/Digest，nonce 是否过期。
4. 状态机层：是否先 DESCRIBE、SETUP 后 PLAY。
5. 传输层：UDP 端口、防火墙、Transport 是否被支持。
6. 媒体层：SDP、RTP、编码参数和解码器。

不要看到 454 就只重连。先确认 Session 是否被覆盖、是否把一个轨道的 Session 用错、是否等待太久导致超时。

### 5.4 阶段练习

给出下面错误，写出第一步检查项：

- 401 Unauthorized
- 404 Not Found
- 455 Method Not Valid in This State
- 461 Unsupported Transport
- TCP 连接成功、DESCRIBE 成功、PLAY 也 200，但没有 RTP

参考方向：认证、URL、方法顺序、Transport、服务器端口/客户端防火墙/是否真正收到数据。

### 5.5 阶段检查清单

- [ ] 我能按 2xx/4xx/5xx 解释错误类别。
- [ ] 我知道 401 不等于密码一定错误。
- [ ] 我知道 454 和 455 分别偏向 Session 问题和状态顺序问题。
- [ ] 我会把“RTSP 成功但没有 RTP”继续分层排查。

---

## 6. CSeq、Session 与连接：三个容易混淆的概念

### 6.1 CSeq：请求和响应的流水号

CSeq 用来把响应对应到请求：

~~~text
CSeq: 1
~~~

客户端通常每发一条 RTSP 请求就递增一次：

~~~text
OPTIONS  -> CSeq: 1
DESCRIBE -> CSeq: 2
SETUP    -> CSeq: 3
SETUP    -> CSeq: 4
PLAY     -> CSeq: 5
~~~

响应中的 CSeq 应与对应请求一致。收到不匹配的响应时，先检查：

- 是否有多个线程同时写同一个 RTSP Socket。
- 是否没有保留 TCP 缓冲区中的剩余数据。
- 是否把 interleaved $ 帧误当成 RTSP 响应。
- 是否错误地复用了旧请求的 CSeq。

### 6.2 Session：媒体会话的标识

Session 是服务器为媒体会话分配的 ID：

~~~text
Session: 9A7F23B1;timeout=60
~~~

使用规则：

- SETUP 的第一个响应可能创建 Session。
- 后续轨道的 SETUP、PLAY、PAUSE、GET_PARAMETER、TEARDOWN 通常要带 Session。
- Session 不一定等于 TCP 连接。
- TCP 断开后 Session 通常也就失效，但两者概念不同。
- Session 可能因为超时失效，即使 TCP 连接仍暂时存在。

### 6.3 TCP 连接：承载 RTSP 字节的通道

典型情况下：

~~~text
一个 TCP 连接
  ├─ RTSP 请求和响应
  └─ （可选）TCP interleaved RTP/RTCP
~~~

存在代理、重连或厂商实现导致控制连接和媒体连接关系不同。客户端不要假设“一个 Session 永远只有一个 Socket”，应以服务器响应和 Transport 为准。

### 6.4 Session timeout 和保活

如果响应写着：

~~~text
Session: 9A7F23B1;timeout=60
~~~

通常表示服务器希望在约 60 秒内看到会话活动。实际行为有差异：

- 某些设备要求每 20～30 秒发送一次 OPTIONS。
- 某些设备要求 GET_PARAMETER。
- 某些服务器把 RTP/RTCP 活动也算作保活。
- 某些设备忽略 timeout 参数，按自己的固定时间判断。

建议把保活周期设置得小于 timeout，例如 timeout 为 60 秒时每 20 秒发送一次，并记录请求和响应。不要让多个线程无锁地向 RTSP Socket 写入。

### 6.5 阶段检查清单

- [ ] 我知道 CSeq 是请求/响应匹配号。
- [ ] 我知道 Session 是媒体会话 ID。
- [ ] 我知道 TCP 连接和 Session 不是同一个概念。
- [ ] 我能解释 timeout 和保活的关系。
- [ ] 我知道多个线程不能无锁地写同一个 RTSP Socket。

---

## 7. Transport 协商：UDP、TCP interleaved 和端口

### 7.1 RTP/AVP UDP

常见请求：

~~~text
Transport: RTP/AVP;unicast;client_port=5000-5001
~~~

含义：

- RTP/AVP：使用 RTP/AVP 配置。
- unicast：单播，不是组播。
- client_port=5000-5001：客户端接收 RTP 和 RTCP 的端口范围。

常见响应：

~~~text
Transport: RTP/AVP;unicast;client_port=5000-5001;server_port=6970-6971
~~~

客户端还必须：

- 在发送 SETUP 前绑定相应 UDP 端口。
- 放行本机防火墙。
- 监听 RTP 和 RTCP。
- 记录服务器实际发送的地址和端口。
- 处理 NAT、端口复用和多轨道端口规划。

### 7.2 RTP/AVP/TCP interleaved

当 UDP 不通或需要只使用一个 TCP 连接时，可以请求：

~~~text
Transport: RTP/AVP/TCP;unicast;interleaved=0-1
~~~

之后同一个 TCP 字节流中会出现：

~~~text
$ <channel:1 byte> <length:2 bytes> <binary RTP/RTCP payload>
~~~

解析规则：

1. 读到 $（ASCII 0x24）。
2. 读取 1 字节通道号。
3. 读取网络字节序的 2 字节长度。
4. 再读取指定长度的二进制载荷。
5. 根据 SETUP 中的 interleaved=0-1 判断 RTP/RTCP 通道。

不能把二进制载荷当成字符串，也不能用 strlen() 计算长度。RTSP 文本响应和 $ 帧可能在同一次 recv() 中一起到达，必须在同一个缓冲区里按协议边界解析。

### 7.3 unicast 和 multicast

- unicast（单播）：服务器向某个客户端地址发送，最常见。
- multicast（组播）：多个接收者加入同一个组播地址，网络设备需要支持组播。

初学时先实现 unicast UDP，再实现 TCP interleaved；组播需要额外理解 IGMP、路由和交换机配置。

### 7.4 如何选择 UDP 或 TCP

| 场景 | 优先考虑 |
| --- | --- |
| 局域网、低延迟、UDP 端口可达 | RTP/AVP UDP |
| 公网、NAT 或防火墙限制 | RTP/AVP/TCP |
| 需要简化端口管理和抓包 | TCP interleaved |
| 对丢包敏感但可接受更高延迟 | TCP interleaved，仍要处理拥塞和积压 |
| 设备只支持一种方式 | 按设备能力，失败后尝试另一种 |

TCP 不能消除网络延迟；丢包后 TCP 重传可能造成整体停顿。UDP 不能保证顺序和可靠性；接收端必须处理丢包和乱序。

### 7.5 阶段练习

1. 解释 client_port=5000-5001 中两个端口各自做什么。
2. 为什么 interleaved 通道需要长度字段？
3. 如果 SETUP 返回 interleaved=2-3，客户端能否仍然固定把 0 当视频？
4. 服务器返回 461 时，如何从 UDP 改试 TCP？

### 7.6 阶段检查清单

- [ ] 我理解 RTP 和 RTCP 通常使用相邻的 UDP 端口。
- [ ] 我能读出 client_port、server_port、interleaved。
- [ ] 我知道 $ 帧不是 RTSP 文本消息。
- [ ] 我知道 TCP interleaved 也需要处理半包和粘包。
- [ ] 我能解释 UDP 与 TCP 的延迟、可靠性和部署差异。

---

## 8. 认证：401 之后不要只改密码

### 8.1 Basic Authentication

服务器可能返回：

~~~text
RTSP/1.0 401 Unauthorized
CSeq: 1
WWW-Authenticate: Basic realm="camera"
~~~

Basic 的 Authorization 形式是：

~~~text
Authorization: Basic <base64(username:password)>
~~~

Base64 只是编码，不是加密。不要把用户名密码打印到日志或写入知识库。

### 8.2 Digest Authentication

更常见的挑战：

~~~text
WWW-Authenticate: Digest realm="camera", nonce="abc123", qop="auth"
~~~

客户端根据用户名、密码、realm、nonce、方法和 URI 计算摘要，再发送：

~~~text
Authorization: Digest username="user", realm="camera", nonce="abc123", uri="rtsp://...", response="..."
~~~

关键点：

- nonce 是服务器给本次挑战的随机值或时间相关值。
- uri 必须与请求 URI 的形式一致；有些设备对尾部斜杠很敏感。
- Digest 计算必须严格按服务器声明的算法和字段处理。
- 本章示例不实现 Digest；真实客户端应使用经过验证的库或单独实现并写测试。

### 8.3 认证流程

典型流程：

1. 先发 OPTIONS 或 DESCRIBE。
2. 收到 401 和 WWW-Authenticate。
3. 解析认证方案、realm、nonce 等参数。
4. 重新发送原请求，并带 Authorization。
5. 后续请求是否需要 Authorization，取决于服务器和 nonce 生命周期。
6. nonce 过期时重新处理 401 挑战。

不要把密码直接拼在 URL 后并假设服务器一定能正确处理；也不要把完整 URL（含密码）写入日志。

### 8.4 常见认证错误

| 现象 | 可能原因 |
| --- | --- |
| 第一次 401，第二次仍 401 | Digest 计算错误、URI 不一致、密码错误 |
| OPTIONS 成功，DESCRIBE 401 | 服务器对不同方法使用不同权限 |
| SETUP 401 | 认证 Header 未带到后续请求 |
| 偶尔 401 | nonce 过期或服务器时间/会话策略变化 |
| 服务器要求 Basic | 客户端只实现了 Digest |

### 8.5 阶段检查清单

- [ ] 我知道 401 响应要看 WWW-Authenticate。
- [ ] 我知道 Basic 的 Base64 不是加密。
- [ ] 我知道 Digest 依赖 nonce、realm、方法和 URI。
- [ ] 我不会在日志和知识库中保存真实密码。
- [ ] 我知道最小示例遇到 401 时应记录挑战，而不是假装登录成功。

---

## 9. 拉流状态机：按状态理解方法顺序

### 9.1 推荐状态

~~~text
Disconnected
    │ TCP connect
Connected
    │ OPTIONS / DESCRIBE
Described
    │ SETUP video/audio
Ready
    │ PLAY
Playing
    ├─ PAUSE
    │    ↓
    │  Paused ── PLAY ──> Playing
    │
    ├─ 保活失败/网络断开
    │    ↓
    └─ Reconnecting ──> Connected
Playing ── TEARDOWN ──> Closed
~~~

### 9.2 每个状态的职责

| 状态 | 已有条件 | 可以做的事 |
| --- | --- | --- |
| Disconnected | 没有 TCP 连接 | 建立连接 |
| Connected | TCP 成功 | OPTIONS、DESCRIBE |
| Described | 已拿到 SDP | 解析轨道和控制 URL |
| Ready | 所需轨道 SETUP 成功，有 Session | PLAY、PAUSE、保活 |
| Playing | PLAY 成功 | 收 RTP/RTCP、解码 |
| Paused | Session 仍有效 | 再次 PLAY 或 TEARDOWN |
| Reconnecting | 原连接失效 | 释放旧资源、退避后重新开始 |

状态机的好处是：错误发生时能知道恢复动作。例如：

- DESCRIBE 失败：通常不用直接发 PLAY，应检查认证和 URL。
- 视频 SETUP 成功、音频 SETUP 失败：可以只播放视频，或结束会话重试。
- PLAY 454：Session 失效，需要重新 DESCRIBE/SETUP。
- RTP 长时间没有数据：可能 PLAY 响应虽成功，但传输协商、防火墙或服务器流源有问题。

### 9.3 重连的基本策略

重连不是“无限快速重试”：

1. 停止旧接收线程。
2. 关闭旧 Socket。
3. 清空旧 Session、RTP 序号和解码器状态。
4. 使用 1、2、4、8 秒等退避时间，设置最大值。
5. 重新 OPTIONS/DESCRIBE/SETUP/PLAY。
6. 记录失败原因和重连次数。
7. 连接恢复后再逐渐恢复正常保活和日志级别。

不要复用已失效的 Session，也不要把旧 RTP 包混入新会话。

### 9.4 阶段练习

- 画出“DESCRIBE 失败”“SETUP 461”“PLAY 454”“播放中 TCP 断开”的状态迁移。
- 解释为什么 SETUP 成功后还不能直接显示视频。
- 设计一个最多重试 5 次的退避表。

### 9.5 阶段检查清单

- [ ] 我能画出 Connected、Described、Ready、Playing。
- [ ] 我知道方法顺序错误应在状态机层处理。
- [ ] 我知道重连前要清理旧 Session 和接收状态。
- [ ] 我能解释为什么“收到 200 PLAY”仍需验证 RTP 是否到达。

---

## 10. 从协议到 C++：先实现最小客户端

### 10.1 最小客户端应该做什么

第一版只做四件事：

1. 解析命令行中的主机、端口、资源路径。
2. 建立 TCP 连接。
3. 发送 OPTIONS。
4. 发送 DESCRIBE 并打印响应。

这版程序不做认证计算、SDP 完整解析、SETUP/PLAY、UDP/RTP 接收和视频解码。先验证 TCP、报文格式和服务器地址，能显著降低排错难度。

### 10.2 通用的读取思路

实际代码需要把读取响应分成两层：

- 缓冲区层：不断 recv()，保存所有尚未消费的字节。
- 协议层：从缓冲区中找 \r\n\r\n，解析 Header，再按 Content-Length 取消息体。

本节示例为了保持短小，假设 OPTIONS/DESCRIBE 期间没有 interleaved 二进制帧；实现 PLAY 后必须把 $ 帧解析加入协议层。

---

## 11. Windows 示例：Winsock 最小 RTSP 客户端

### 11.1 保存代码

将下面内容保存为 rtsp_probe_windows.cpp，文件编码使用 UTF-8。

~~~cpp
#define _WIN32_WINNT 0x0601
#include <winsock2.h>
#include <ws2tcpip.h>

#include <iostream>
#include <sstream>
#include <stdexcept>
#include <string>

#pragma comment(lib, "Ws2_32.lib")

namespace {

void send_all(SOCKET sock, const std::string& data) {
    std::size_t sent = 0;
    while (sent < data.size()) {
        const int n = ::send(
            sock,
            data.data() + sent,
            static_cast<int>(data.size() - sent),
            0);
        if (n == SOCKET_ERROR) {
            throw std::runtime_error(
                "send failed: " + std::to_string(WSAGetLastError()));
        }
        if (n == 0) {
            throw std::runtime_error("send returned 0");
        }
        sent += static_cast<std::size_t>(n);
    }
}

std::string recv_until_headers(SOCKET sock) {
    std::string data;
    char buffer[4096];

    for (;;) {
        const int n = ::recv(sock, buffer, sizeof(buffer), 0);
        if (n == SOCKET_ERROR) {
            throw std::runtime_error(
                "recv failed: " + std::to_string(WSAGetLastError()));
        }
        if (n == 0) {
            break;
        }

        data.append(buffer, static_cast<std::size_t>(n));
        if (data.find("\r\n\r\n") != std::string::npos) {
            break;
        }
    }
    return data;
}

std::string make_request(
    const std::string& method,
    const std::string& url,
    int cseq) {
    std::ostringstream out;
    out << method << " " << url << " RTSP/1.0\r\n"
        << "CSeq: " << cseq << "\r\n"
        << "User-Agent: BeginnerRtspClient/0.1\r\n";

    if (method == "DESCRIBE") {
        out << "Accept: application/sdp\r\n";
    }

    out << "\r\n";
    return out.str();
}

SOCKET connect_tcp(const std::string& host, const std::string& port) {
    addrinfo hints{};
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_protocol = IPPROTO_TCP;

    addrinfo* result = nullptr;
    const int rc = ::getaddrinfo(host.c_str(), port.c_str(), &hints, &result);
    if (rc != 0) {
        throw std::runtime_error(
            "getaddrinfo failed: " + std::to_string(rc));
    }

    SOCKET sock = INVALID_SOCKET;
    for (addrinfo* p = result; p != nullptr; p = p->ai_next) {
        sock = ::socket(p->ai_family, p->ai_socktype, p->ai_protocol);
        if (sock == INVALID_SOCKET) {
            continue;
        }

        if (::connect(sock, p->ai_addr,
                      static_cast<int>(p->ai_addrlen)) == 0) {
            break;
        }

        ::closesocket(sock);
        sock = INVALID_SOCKET;
    }

    ::freeaddrinfo(result);

    if (sock == INVALID_SOCKET) {
        throw std::runtime_error(
            "connect failed: " + std::to_string(WSAGetLastError()));
    }
    return sock;
}

} // namespace

int main(int argc, char* argv[]) {
    if (argc != 4) {
        std::cerr
            << "Usage: rtsp_probe_windows.exe <host> <port> <path>\n"
            << "Example: rtsp_probe_windows.exe 192.168.1.20 554 /live/main\n";
        return 2;
    }

    const std::string host = argv[1];
    const std::string port = argv[2];
    const std::string path = argv[3];
    const std::string url = "rtsp://" + host + ":" + port + path;

    WSADATA wsa_data{};
    const int startup_rc = ::WSAStartup(MAKEWORD(2, 2), &wsa_data);
    if (startup_rc != 0) {
        std::cerr << "WSAStartup failed: " << startup_rc << "\n";
        return 1;
    }

    SOCKET sock = INVALID_SOCKET;

    try {
        sock = connect_tcp(host, port);
        std::cout << "Connected to " << host << ":" << port << "\n";

        send_all(sock, make_request("OPTIONS", url, 1));
        std::cout << "\n--- OPTIONS response ---\n";
        std::cout << recv_until_headers(sock);

        send_all(sock, make_request("DESCRIBE", url, 2));
        std::cout << "\n--- DESCRIBE response ---\n";
        std::cout << recv_until_headers(sock);

        ::closesocket(sock);
        ::WSACleanup();
        return 0;
    } catch (const std::exception& ex) {
        std::cerr << "Error: " << ex.what() << "\n";
        if (sock != INVALID_SOCKET) {
            ::closesocket(sock);
        }
        ::WSACleanup();
        return 1;
    }
}
~~~

### 11.2 Windows 编译和运行

Visual Studio Developer PowerShell：

~~~powershell
cl /nologo /std:c++17 /EHsc /W4 rtsp_probe_windows.cpp /Fe:rtsp_probe_windows.exe
if ($LASTEXITCODE -ne 0) { throw "compile failed" }

.\rtsp_probe_windows.exe 192.168.1.20 554 /live/main
if ($LASTEXITCODE -ne 0) { throw "program failed" }
~~~

MinGW-w64：

~~~powershell
g++ -std=c++17 -O2 -Wall -Wextra -pedantic rtsp_probe_windows.cpp -lws2_32 -o rtsp_probe_windows.exe
if ($LASTEXITCODE -ne 0) { throw "compile failed" }

.\rtsp_probe_windows.exe 192.168.1.20 554 /live/main
~~~

把示例 IP、端口和路径换成真实设备值。不要把带真实密码的完整 URL 粘贴到共享终端记录中。

### 11.3 Windows 预期结果

成功时大致会看到：

~~~text
Connected to 192.168.1.20:554

--- OPTIONS response ---
RTSP/1.0 200 OK
CSeq: 1
Public: OPTIONS, DESCRIBE, SETUP, PLAY, PAUSE, TEARDOWN

--- DESCRIBE response ---
RTSP/1.0 200 OK
CSeq: 2
Content-Type: application/sdp
Content-Length: ...
...
~~~

如果 DESCRIBE 返回 401，程序会打印挑战响应后结束；这是预期限制，不代表 Socket 失败。

### 11.4 Windows 常见错误

| 现象 | 处理 |
| --- | --- |
| cl is not recognized | 使用 Visual Studio Developer PowerShell，或改用 MinGW |
| WSAStartup failed | 检查 Winsock 初始化；不要把 Windows 代码拿到 Linux 编译 |
| connect failed: 10061 | 目标端口没有服务、端口错误或被拒绝 |
| connect failed: 10060 | 超时，检查 IP、路由、防火墙 |
| 找不到 winsock2.h | Visual Studio/Windows SDK 未安装完整 |
| 响应只有 Header，没有 SDP | 本示例只读到 Header；真实代码还要按 Content-Length 读取消息体 |
| 连接成功但卡住 | 服务器可能等待认证，或响应被拆包；下一步实现完整读取 |

> 教学限制：示例中的 recv_until_headers() 在发现 \r\n\r\n 后立即返回，可能没有读取完整 SDP。生产代码必须按 Content-Length 补齐 body，并保留多余字节。

---

## 12. Linux 示例：POSIX Socket 最小 RTSP 客户端

### 12.1 保存代码

将下面内容保存为 rtsp_probe_linux.cpp。

~~~cpp
#include <netdb.h>
#include <sys/socket.h>
#include <unistd.h>

#include <cerrno>
#include <cstring>
#include <iostream>
#include <sstream>
#include <stdexcept>
#include <string>

namespace {

void send_all(int sock, const std::string& data) {
    std::size_t sent = 0;
    while (sent < data.size()) {
        const ssize_t n = ::send(
            sock,
            data.data() + sent,
            data.size() - sent,
            MSG_NOSIGNAL);
        if (n < 0) {
            throw std::runtime_error(
                std::string("send failed: ") + std::strerror(errno));
        }
        if (n == 0) {
            throw std::runtime_error("send returned 0");
        }
        sent += static_cast<std::size_t>(n);
    }
}

std::string recv_until_headers(int sock) {
    std::string data;
    char buffer[4096];

    for (;;) {
        const ssize_t n = ::recv(sock, buffer, sizeof(buffer), 0);
        if (n < 0) {
            throw std::runtime_error(
                std::string("recv failed: ") + std::strerror(errno));
        }
        if (n == 0) {
            break;
        }

        data.append(buffer, static_cast<std::size_t>(n));
        if (data.find("\r\n\r\n") != std::string::npos) {
            break;
        }
    }
    return data;
}

std::string make_request(
    const std::string& method,
    const std::string& url,
    int cseq) {
    std::ostringstream out;
    out << method << " " << url << " RTSP/1.0\r\n"
        << "CSeq: " << cseq << "\r\n"
        << "User-Agent: BeginnerRtspClient/0.1\r\n";

    if (method == "DESCRIBE") {
        out << "Accept: application/sdp\r\n";
    }

    out << "\r\n";
    return out.str();
}

int connect_tcp(const std::string& host, const std::string& port) {
    addrinfo hints{};
    hints.ai_family = AF_UNSPEC;
    hints.ai_socktype = SOCK_STREAM;
    hints.ai_protocol = IPPROTO_TCP;

    addrinfo* result = nullptr;
    const int rc = ::getaddrinfo(host.c_str(), port.c_str(), &hints, &result);
    if (rc != 0) {
        throw std::runtime_error(
            std::string("getaddrinfo failed: ") + ::gai_strerror(rc));
    }

    int sock = -1;
    for (addrinfo* p = result; p != nullptr; p = p->ai_next) {
        sock = ::socket(p->ai_family, p->ai_socktype, p->ai_protocol);
        if (sock < 0) {
            continue;
        }

        if (::connect(sock, p->ai_addr, p->ai_addrlen) == 0) {
            break;
        }

        ::close(sock);
        sock = -1;
    }

    ::freeaddrinfo(result);

    if (sock < 0) {
        throw std::runtime_error(
            std::string("connect failed: ") + std::strerror(errno));
    }
    return sock;
}

} // namespace

int main(int argc, char* argv[]) {
    if (argc != 4) {
        std::cerr
            << "Usage: ./rtsp_probe_linux <host> <port> <path>\n"
            << "Example: ./rtsp_probe_linux 192.168.1.20 554 /live/main\n";
        return 2;
    }

    const std::string host = argv[1];
    const std::string port = argv[2];
    const std::string path = argv[3];
    const std::string url = "rtsp://" + host + ":" + port + path;

    int sock = -1;
    try {
        sock = connect_tcp(host, port);
        std::cout << "Connected to " << host << ":" << port << "\n";

        send_all(sock, make_request("OPTIONS", url, 1));
        std::cout << "\n--- OPTIONS response ---\n";
        std::cout << recv_until_headers(sock);

        send_all(sock, make_request("DESCRIBE", url, 2));
        std::cout << "\n--- DESCRIBE response ---\n";
        std::cout << recv_until_headers(sock);

        ::close(sock);
        return 0;
    } catch (const std::exception& ex) {
        std::cerr << "Error: " << ex.what() << "\n";
        if (sock >= 0) {
            ::close(sock);
        }
        return 1;
    }
}
~~~

### 12.2 Linux 编译和运行

~~~bash
g++ -std=c++17 -O2 -Wall -Wextra -pedantic rtsp_probe_linux.cpp -o rtsp_probe_linux
if [ $? -ne 0 ]; then
    echo "compile failed"
    exit 1
fi

./rtsp_probe_linux 192.168.1.20 554 /live/main
~~~

也可以使用 Clang：

~~~bash
clang++ -std=c++17 -O2 -Wall -Wextra -pedantic rtsp_probe_linux.cpp -o rtsp_probe_linux
./rtsp_probe_linux 192.168.1.20 554 /live/main
~~~

### 12.3 Linux 常见错误

| 现象 | 处理 |
| --- | --- |
| Connection refused | 端口没有监听或地址错误 |
| No route to host | 路由、网段或防火墙问题 |
| Permission denied | 不要使用低于 1024 的本地监听端口；客户端通常不需要 root |
| Broken pipe | 服务器已关闭连接；检查请求格式和 Session |
| gai_strerror 错误 | 主机名解析失败，先用 IP 验证 |
| 收到 401 | 需要实现 Basic/Digest，而不是反复发送同一请求 |
| 收到 Header 后没有 SDP | 按 Content-Length 继续读取 body |

> Linux 示例和 Windows 示例做同样的事情，方便比较系统 API。协议层逻辑应保持一致；变化主要在初始化、关闭 Socket 和错误码。

---

## 13. 把最小示例升级成可用客户端

### 13.1 正确读取 Content-Length

更完整的 read_rtsp_response() 应维护持久缓冲区：

~~~text
while 缓冲区没有 \r\n\r\n:
    recv() 追加到缓冲区

解析状态行和 Header
body_length = Content-Length，若不存在则为 0

while 缓冲区中 body 字节不足 body_length:
    recv() 追加到缓冲区

取出 Header + body
保留剩余字节，供下一次解析
~~~

要注意：

- Content-Length 缺失时不能擅自读取到连接关闭，因为服务器可能保持连接。
- 有些响应使用 Content-Base 或 Content-Location，要保存用于解析相对 URL。
- 同一 TCP 流里可能出现 $ interleaved 帧；解析器要先判断当前字节是 RTSP 文本还是 $。
- 生产代码应加入接收超时，避免永久阻塞。

### 13.2 解析 Header 的建议

建议先把每行拆成 name 和 value，再做：

- Header 名转小写。
- 去掉值两侧空格。
- 对 Session 分离 ID 和参数，例如 9A7F23B1;timeout=60。
- 对 Transport 按分号拆参数，再解析 key=value。
- 对未知 Header 保留原文或忽略，不要因为未知 Header 就失败。

### 13.3 URL 和 SDP 的关系

DESCRIBE 可能返回：

- Content-Base: rtsp://host/path/
- a=control:trackID=0
- 或完整的 a=control:rtsp://host/path/trackID=0

客户端要根据 RFC 和服务器实际格式拼出 SETUP URL。不要简单地把 trackID=0 拼到原始 URL 后面；要处理：

- 绝对 URL。
- 相对路径。
- Content-Base。
- a=control:*。
- 尾部斜杠。

完整解析见 [[04-SDP会话描述]]。

### 13.4 SETUP/PLAY 的伪代码

~~~text
connect(rtsp_host, rtsp_port)

OPTIONS(url)
if status == 401:
    处理认证并重试

DESCRIBE(url, Accept: application/sdp)
解析 SDP，得到 video_control、audio_control

为视频创建 UDP RTP/RTCP socket
SETUP(video_control, Transport: RTP/AVP;unicast;client_port=x-y)
保存 Session 和 server_port

为音频创建 UDP RTP/RTCP socket
SETUP(audio_control, Session: same, Transport: ...)
确认 Session 没有改变

PLAY(Content-Base 或聚合 URL, Session: same)
开始接收 RTP/RTCP

循环：
    接收媒体
    处理 RTP 序号、时间戳和重组
    按 timeout 周期发送 OPTIONS 或 GET_PARAMETER
    发现断线则进入重连状态

TEARDOWN(Session)
释放资源
~~~

---

## 14. 保活、超时和线程设计

### 14.1 三种时间不要混淆

- Socket 超时：某次读写最多等待多久。
- Session timeout：服务器多久没有看到会话活动后可能释放 Session。
- 媒体时间戳：RTP 包在媒体时间轴上的时间。

调整 Socket 超时不会自动延长 Session；发送 OPTIONS 也不会修复错误的 RTP 时间戳。

### 14.2 一个简单的线程划分

初学实现可以是：

- RTSP 控制线程：发送请求、读取响应、保活。
- RTP 接收线程：从 UDP 或 TCP interleaved 读取媒体。
- 解码线程：把完整压缩数据交给解码器。
- 播放/输出线程：按时间戳呈现。

线程之间使用有界队列。队列无限增长会让延迟越来越大；队列太小则容易丢帧。线程模型的工程化实现见 [[10-C++客户端架构]]。

### 14.3 常见超时处理

| 超时位置 | 可能原因 | 建议动作 |
| --- | --- | --- |
| TCP connect 超时 | 地址、路由、防火墙 | 检查网络，再换 IP/端口验证 |
| OPTIONS 无响应 | 服务未启动、请求格式、认证 | 抓包确认请求是否离开 |
| DESCRIBE 读取超时 | 未处理 Content-Length 或服务器等待 | 打印完整请求，确认最后空行 |
| PLAY 后无 RTP | UDP 不通、Transport 不匹配、源流无数据 | 改用 TCP interleaved，检查端口 |
| 播放中保活超时 | Session 过期、请求方法不被支持 | 尝试 OPTIONS/GET_PARAMETER，记录响应 |
| 解码线程积压 | 处理速度低于输入速度 | 限制队列、统计延迟、检查解码器 |

---

## 15. 常见错误总表：从现象反推层次

| 现象 | 先查什么 | 不要先做什么 |
| --- | --- | --- |
| 端口连接不上 | IP、端口、路由、防火墙 | 不要先改 SDP |
| 400 | CRLF、请求行、Header 格式 | 不要先重启摄像头 |
| 401 | WWW-Authenticate、认证算法、URI | 不要把密码写进日志 |
| 404 | 资源路径、尾部斜杠、设备文档 | 不要猜路径无限重试 |
| DESCRIBE 200 但没有 SDP | Content-Length、Content-Type、读取逻辑 | 不要直接 SETUP 未知轨道 |
| SETUP 461 | Transport 能力、UDP 端口、TCP 备选 | 不要重复发送相同 Transport |
| PLAY 455 | 方法顺序和 Session | 不要忽略状态机 |
| PLAY 200 但无 RTP | server_port、防火墙、TCP/UDP 模式 | 不要认为 200 就代表视频一定可用 |
| RTP 有包但画面花 | 丢包、乱序、NAL 重组、SPS/PPS | 不要只调播放器窗口 |
| 一段时间后断流 | Session timeout、保活方法、连接状态 | 不要只增加接收缓冲 |
| 延迟越来越大 | 队列积压、处理速度、时钟漂移 | 不要无限增大队列 |
| Windows 代码 Linux 编译失败 | 平台 API 不同 | 不要混用 Winsock 和 POSIX 函数 |
| Linux 代码 Windows 编译失败 | unistd.h、MSG_NOSIGNAL 等差异 | 不要只删除一个头文件就认为完成移植 |

---

## 16. 练习题：从手写报文到抓包分析

### 16.1 基础题

1. 写出一条符合格式的 OPTIONS 请求，CSeq 为 1。
2. 为什么请求末尾需要两个 CRLF？
3. Content-Length: 174 表示什么？
4. Session: abc;timeout=60 中 Session ID 和 timeout 各是什么？
5. CSeq 是否可以每次从 1 开始？在同一个连接中为什么建议递增？
6. 为什么 TCP recv() 返回 0 和本次没有收到完整响应是两回事？

### 16.2 流程题

给定设备有视频和音频两条轨道，写出合理的方法顺序，并在每个请求旁写出必须保存的响应字段。

参考顺序：

~~~text
OPTIONS
DESCRIBE -> SDP、Content-Base
SETUP video -> Session、Transport
SETUP audio -> Session、Transport
PLAY -> RTP-Info
TEARDOWN
~~~

### 16.3 故障题

1. DESCRIBE 返回 401，WWW-Authenticate 是 Digest。下一步需要哪些字段？
2. SETUP 使用 UDP 返回 461，如何改成 TCP interleaved？
3. PLAY 返回 200，但 UDP 端口没有任何包。列出至少四个检查项。
4. 使用 TCP interleaved 时，为什么不能对整个 TCP 缓冲区直接调用字符串解析？
5. 摄像头 60 秒后停止发送，Session timeout 是 60，保活周期设置为 70 秒。问题在哪里？

### 16.4 编程题

按难度顺序完成：

- [ ] 修改最小客户端：从响应中提取状态码和 CSeq。
- [ ] 实现按 Content-Length 读取 DESCRIBE 的 SDP body。
- [ ] 将 Header 名称统一转小写后查找。
- [ ] 实现 Transport 参数解析，提取 server_port 和 interleaved。
- [ ] 实现一个只发送 OPTIONS 的保活循环。
- [ ] 加入连接超时和接收超时。
- [ ] 处理 401：先只打印 WWW-Authenticate，再实现 Basic。
- [ ] 用一个有边界的缓冲区处理 $ interleaved RTP/RTCP 帧。

---

## 17. 分阶段检查清单

### 阶段 A：基础与消息格式

- [ ] 我能解释 TCP 字节流和半包/粘包。
- [ ] 我能从 RTSP URL 提取主机、端口和路径。
- [ ] 我能手写 OPTIONS 和 DESCRIBE。
- [ ] 我能指出请求行、Header、空行和消息体。
- [ ] 我知道 \r\n\r\n 和 Content-Length 的用途。

### 阶段 B：方法与状态机

- [ ] 我能按顺序解释 OPTIONS、DESCRIBE、SETUP、PLAY、TEARDOWN。
- [ ] 我知道多轨道通常需要多次 SETUP。
- [ ] 我能说明 Session 何时创建、何时携带。
- [ ] 我能画出 Connected、Described、Ready、Playing。
- [ ] 我能为 401、404、454、455、461 各写出第一步排查。

### 阶段 C：Transport 与保活

- [ ] 我能解释 UDP client_port/server_port。
- [ ] 我能解释 TCP interleaved 和 $ 帧。
- [ ] 我知道 RTP 和 RTCP 为什么常使用相邻端口。
- [ ] 我能说明 Session timeout 和保活周期的关系。
- [ ] 我能说出 UDP 和 TCP interleaved 的适用场景。

### 阶段 D：代码与工具

- [ ] 我在 Windows 编译并运行了 Winsock 示例。
- [ ] 我在 Linux 编译并运行了 POSIX Socket 示例。
- [ ] 我收到过至少一条 OPTIONS 响应。
- [ ] 我收到过至少一条 DESCRIBE 响应，或记录了明确的 401/404 原因。
- [ ] 我能指出示例中按 Content-Length 读取仍需补强的地方。
- [ ] 我没有把真实密码、Token 或完整敏感 URL 写入日志。

### 阶段 E：进入下一章前的验收

- [ ] 我能在纸上解释一次完整拉流会话。
- [ ] 我能根据 SDP 找出视频和音频轨道的 control URL。
- [ ] 我能解释为什么 PLAY 200 不代表已经收到有效视频。
- [ ] 我能选择 UDP 或 TCP interleaved，并说明原因。
- [ ] 我能把网络错误、RTSP 错误、RTP 错误和解码错误分层。
- [ ] 我已完成本章至少 5 道练习题。

---

## 18. 最后复习：用一句话串起来

~~~text
先通过 TCP 连接 RTSP 服务，
用 OPTIONS 了解能力，
用 DESCRIBE 获取 SDP，
对每条轨道用 SETUP 协商 Transport 并获得 Session，
用 PLAY 开始发送 RTP/RTCP，
用保活维持 Session，
最后用 TEARDOWN 结束，
再把 RTP 交给重组器、解码器和播放器。
~~~

当这条链条已经清楚后，再学习：

- [[04-SDP会话描述]]：如何从 SDP 生成准确的 SETUP URL。
- [[05-RTP协议]]：如何解析序号、时间戳和 RTP 头。
- [[06-RTCP协议]]：如何计算丢包、抖动和音视频时钟关系。
- [[07-H264与RTP封装]]：如何从 RTP 得到 H.264 NAL。
- [[08-FFmpeg与解码]]：如何把压缩数据送入解码器。
- [[30-runbooks/02-抓包分析一次RTSP拉流|抓包分析一次完整拉流]]：如何用 Wireshark 对照每一条请求和响应。

## Related

- [[00-RTSP学习总纲]]
- [[01-网络与Socket基础]]
- [[02-音视频基础]]
- [[04-SDP会话描述]]
- [[05-RTP协议]]
- [[06-RTCP协议]]
- [[10-C++客户端架构]]
- [[30-runbooks/03-验证TCP交织传输|验证 TCP 交织传输]]
- [[30-runbooks/04-验证UDP传输|验证 UDP RTP/RTCP 传输]]

## 参考标准

- [RFC 2326：RTSP 1.0](https://www.rfc-editor.org/rfc/rfc2326)
- [RFC 7826：RTSP 2.0](https://www.rfc-editor.org/rfc/rfc7826)
- [RFC 4566：SDP](https://www.rfc-editor.org/rfc/rfc4566)
- [RFC 3550：RTP/RTCP](https://www.rfc-editor.org/rfc/rfc3550)
