---
type: topic
status: active
created: 2026-10-16
updated: 2026-10-19
tags:
  - sdp
  - rtsp
  - beginner
  - cplusplus
  - windows
  - linux
aliases:
  - SDP 入门
  - Session Description Protocol
---

# SDP 会话描述

> 适用对象：会 C++ 基础，但第一次学习 SDP、媒体轨道和 RTP 编码参数的读者。
>
> 学完后的实际产出：拿到一段 SDP，能列出音视频轨道、编码、RTP 时钟、编码参数和每条轨道的 SETUP URL；能在 Windows 或 Linux 上编译一个离线 C++17 解析程序。
>
> 阅读方式：先看第 0 节路线，再依次学习。第 9～11 节是可运行实验，第 14 节有参考答案。无需先准备摄像头，也无需先掌握 FFmpeg。

## 0. 学习规划：这一章到底学哪些东西

### 0.1 给“学会”一个明确标准

完成这一章后，你应该能独立回答：

1. SDP 为什么会出现在 RTSP 的 DESCRIBE 响应中？
2. 一份描述里有几条轨道？视频和音频分别用什么编码？
3. RTP 包头里的数字 `96` 到底代表哪一种格式？
4. `90000` 是不是帧率？`48000/2` 分别是什么意思？
5. `fmtp` 中哪些参数要交给 RTP 解包器，哪些要交给解码器？
6. 为什么同样是 `trackID=0`，换一个基准 URL 就可能请求错地址？
7. 怎样把一份文本整理成 C++ 数据结构，并报告有用的错误？

最终成果是完成这条转换：

```text
DESCRIBE 响应
   ↓ 按 Content-Length 取出消息体
SDP 文本
   ↓ 按行解析，并区分会话级和媒体级
轨道列表 + 每条轨道的格式列表
   ↓ 关联 PT、编码参数，并解析 control URL
SETUP 所需信息 + RTP/解码初始化所需信息
```

### 0.2 完整学习顺序

建议分成 8 次学习，每次 45～90 分钟；编码实验可以拆成两次。时间只是参考，以检查清单为准。

| 阶段 | 阅读位置 | 本阶段内容 | 需要先知道 | 动手产出与通过标准 |
| --- | --- | --- | --- | --- |
| A | 第 1～2 节 | 前置知识、SDP 的作用、输入边界 | C++ 字符串、IP/端口 | 能从 RTSP 响应中指出真正的 SDP |
| B | 第 3～4 节 | 行语法、字段顺序、作用域 | A | 手工标注会话级与每个媒体段 |
| C | 第 5 节 | PT、rtpmap、fmtp、时钟 | B | 按 `(轨道, PT)` 列出格式表 |
| D | 第 6 节 | H.264/H.265/AAC/G.711/Opus | C、基本音视频概念 | 说明各编码需要什么初始化信息 |
| E | 第 7 节 | control 与 URL 解析 | B、URL 基础 | 算对相对 URL、绝对 URL、尾部斜杠的例子 |
| F | 第 8～10 节 | C++ 解析器、样例文件、编译运行 | B～E、C++ 容器 | 在自己的系统上得到预期输出 |
| G | 第 11～12 节 | 接真实流、排错、输入健壮性 | F、上一章 RTSP | 分清网络、SDP、传输、解码问题 |
| H | 第 13～15 节 | 练习、答案、综合验收 | 前面各阶段 | 独立分析一份没见过的 SDP |

### 0.3 学习范围和深度

**本章必须掌握：**

- `v/o/s/c/t/m/a` 的含义，尤其是会话级与媒体级。
- `m=`、`a=rtpmap`、`a=fmtp` 通过 PT 关联起来。
- 会话控制 URL 和单轨道控制 URL 的用途。
- C++ 按行解析、字段校验、保留未知属性、Windows/Linux 运行。
- SDP 不完整、编码不支持、URL 错误时如何定位。

**本章需要认识，后面再深入：**

- H.264 的 SPS/PPS、H.265 的 VPS/SPS/PPS、AAC 的配置数据。
- RTP 时间戳与音视频同步，详见 [[05-RTP协议]]、[[06-RTCP协议]]。
- RTP 分片重组与 FFmpeg 解码，详见 [[07-H264与RTP封装]]、[[08-FFmpeg与解码]]。

**进阶选修：** SDP 的组播、多个时间段、带宽属性、复杂编解码参数。SIP/WebRTC 中的 SDP offer/answer、ICE、DTLS、BUNDLE 属于另一套应用流程；先了解名字，不要把它们的所有规则套到 RTSP 拉流上。

## 1. 前置知识：这些词先用日常语言解释

### 1.1 你已有的 C++ 基础如何派上用场

本章程序只需要：

- `std::string`：保存一行文本，用 `find` 找分隔符、用 `substr` 截取。
- `std::vector`：存放数量不固定的轨道、PT 和属性。
- `std::map`：根据 PT 找编码信息。
- `std::istringstream`：把一行按空格拆成单词。
- `std::ifstream`：读取文件；`throw` / `catch`：把错误交给调用者处理。
- 命令行参数：让同一个程序处理不同的 SDP 文件和基准 URL。

不需要线程、Socket、模板元编程或图形界面。如果 `std::map` 不熟，可以先把它理解为“输入一个键，就能找到对应记录的字典”。

### 1.2 音视频和网络的最小词典

| 词语 | 通俗解释 | 本章中的例子 |
| --- | --- | --- |
| 会话 session | 一次有组织的媒体活动 | 一路直播同时包含视频和声音 |
| 轨道 track / 媒体段 | 会话中的一类媒体描述 | 一条视频轨道、一条音频轨道 |
| 编码 codec | 把原始画面或声音压缩成数据的规则 | H.264、AAC |
| 解码 | 把压缩数据还原成图像或声音 | 压缩 H.264 → 可显示的图像 |
| 载荷 payload | 包中真正装的业务数据 | RTP 头后面的 H.264 数据 |
| PT / Payload Type | RTP 头里用于识别载荷格式的数字 | PT=96 在某条轨道上对应 H.264 |
| RTP 时钟 | 给媒体时间计数的刻度尺 | 每秒 90000 个刻度 |
| 采样率 | 每秒测量声音的次数 | 48000 Hz，即每秒 48000 次 |
| 声道 | 同时记录的声音通道数 | 1=单声道；2 常见为立体声 |
| 参数集 / 配置 | 解码前需要知道的格式说明 | H.264 的 SPS/PPS |
| 单播 / 组播 | 发给一个接收方 / 一个接收组 | 摄像头单播拉流、局域网组播 |

“轨道”是本章对常见 RTSP 媒体段的直观叫法。一个 `m=` 下面可能列出多个格式；它不意味着每个格式各是一条轨道。更复杂的 RTP 会话也可能包含多个发送源，不能把 `m=`、PT、SSRC、一条 Socket 连接全部画等号。

网络基础可回看 [[01-网络与Socket基础]]；编码、帧率和音频基础可回看 [[02-音视频基础]]。

### 1.3 容易混淆的几个“会话编号”

| 出现位置 | 示例 | 用途 |
| --- | --- | --- |
| RTSP 响应头 `Session` | `Session: A19BC` | 后续 RTSP 请求引用服务器建立的会话 |
| SDP 的 `o=` | `o=- 1001 1 IN IP4 127.0.0.1` | 标识描述的来源、会话标识与描述版本 |
| RTP 头的 SSRC | 一个 32 位数字 | 标识 RTP 同步源，后续章节展开 |

三者含义不同，不能把 `o=` 的数字复制到 RTSP 的 `Session:` 头里。

### 1.4 阶段 A 前置检查

- [ ] 我能用一句话解释编码、轨道、PT、时钟、采样率。
- [ ] 我知道一个会话可以同时有视频和音频。
- [ ] 我知道 RTSP Session、SDP 会话标识、RTP SSRC 不是同一个东西。
- [ ] 我能看懂 C++ 的字符串、循环、vector 和 map。

小练习：一台摄像头提供“无声主码流”和“带声音子码流”。它们是否一定出现在同一份 SDP 中？先思考，答案见第 14 节第 1 题。

## 2. SDP 的作用：它是一张媒体说明书

### 2.1 四个协议各做什么

SDP 全称 **Session Description Protocol**，通常译为“会话描述协议”。学习时可先把它当作一种有固定语法的纯文本格式。

| 对象 | 负责回答的问题 | 本章关注的内容 |
| --- | --- | --- |
| RTSP | 请求哪一路？何时开始？怎样建立传输？ | DESCRIBE、SETUP、PLAY |
| SDP | 有哪些媒体？每种媒体怎样解释？ | 轨道、编码、时钟、配置和控制地址 |
| RTP | 媒体数据如何编号、标识格式和计时？ | 包头中的 PT 如何对应 SDP |
| RTCP | 传得好不好？不同媒体的时钟如何对应？ | 后续同步与统计 |

SDP 文本里没有一张张视频画面，也不负责自己建立连接。它可以通过 RTSP、SIP、文件等方式传递；仅有一份 SDP，并不保证它所描述的媒体此刻可达。

### 2.2 它位于 DESCRIBE 响应的消息体

下面是**结构示意**，`<body字节数>` 需要由程序计算，不能原样发送：

```text
RTSP/1.0 200 OK
CSeq: 2
Content-Type: application/sdp
Content-Base: rtsp://127.0.0.1:8554/live/
Content-Length: <body字节数>

v=0
o=- 1001 1 IN IP4 127.0.0.1
s=Beginner SDP Lab
c=IN IP4 0.0.0.0
t=0 0
a=control:*
m=video 0 RTP/AVP 96
a=rtpmap:96 H264/90000
a=control:trackID=0
```

上半部分是 RTSP，空行之后才是 SDP。`Content-Base` 是 RTSP 响应头，**不是 SDP 字段**。线上行末使用 `\r\n`，上面的显示没有把这些控制字符逐个写出来。

正确读取顺序：

1. 收集 TCP 字节，直到找到 RTSP 头末尾的 `\r\n\r\n`。
2. 检查响应状态码；`401` 的错误响应不能当成 SDP 解析。
3. 检查 `Content-Type` 的媒体类型是否为 `application/sdp`，同时保存基准地址相关头。
4. 按 `Content-Length` **字节数**补齐消息体；多收到的字节留给下一条消息。
5. 只把 SDP 消息体传给本章的解析函数。

`recv()` 一次收到的内容可能不完整。上一章的最小客户端只读到 RTSP 头时，不能直接把返回字符串当作完整 SDP，见 [[03-RTSP协议]]。本章离线实验从“完整 SDP 已到手”开始。

### 2.3 一次拉流中它怎样被使用

```text
DESCRIBE → SDP：发现 video/H264、audio/AAC 和各自 control
          ↓
选择客户端支持的轨道与格式
          ↓
视频 SETUP → 保存视频的传输参数
音频 SETUP → 保存音频的传输参数
          ↓
PLAY → 接收 RTP → 按轨道与 PT 找格式 → 解包 → 解码
```

读取到 `H264` 只是知道该用什么规则；真正出画还需要有效 RTP、正确重组、足够的参数和可解码的图像数据。

### 2.4 阶段 A 检查

- [ ] 我能从完整 RTSP 响应中分出状态行、头、空行和 SDP。
- [ ] 我知道 SDP 只描述媒体，不会自动播放。
- [ ] 我知道 `Content-Length` 按字节数计算。
- [ ] 我知道本章程序的输入是 SDP 文本，不是完整 RTSP 响应。

练习：SDP 含有中文标题时，为什么不能简单用“字符个数”填写 `Content-Length`？

## 3. SDP 文本语法：先学会切行与分段

### 3.1 每行长什么样

基本结构是：

```text
一个小写字母=这一字段的内容
```

例如：

```sdp
v=0
s=My Camera
a=control:trackID=0
a=sendonly
```

- `v`、`s`、`a` 是**字段类型**。
- `=` 后面是字段的值；值内部还可能有空格、冒号、斜杠和更多等号。
- 字段字母与 `=` 之间不加空格：`m = video` 不符合语法。
- 规范发送时用 CRLF，即 C++ 字符串中的 `\r\n`；接收时通常还应容忍只有 LF 的文本。
- 文件最后一行最好也有换行；本章程序兼容最后一行没有换行的本地文件。
- SDP 没有通用的 `# 注释` 语法；不要把讲解用的中文批注直接加入待解析文件。
- 一条长 `fmtp` 是一整行，不能因编辑器显示折行就手动拆成两条物理行。

**不要使用“按所有等号切开”的办法。** 比如 Base64 数据可能以 `=` 或 `==` 结尾，后面会用到。

### 3.2 字段有顺序，不能当作随意排列的配置文件

入门先记以下骨架：

```text
v=    协议版本
o=    来源与描述版本
s=    会话名称
...   可选会话信息，按规范顺序出现
c=    可选的会话级连接信息
...   可选带宽信息
t=    时间描述，至少一组
...   可选会话级属性
a=...
m=    第一条媒体描述开始
...   该媒体的连接、带宽、属性
a=...
m=    下一条媒体描述开始
...   下一条媒体的属性
```

`v/o/s/t` 是基础必需字段。对本章可拉流的例子，至少还需要一条媒体描述。`c=` 一般需要在会话级提供，或由每个媒体段各自提供；RTSP 具体连接方式仍以 SETUP 协商为准。

通用 SDP 的完整顺序还包括 `i/u/e/p/b/r/z` 等可选字段，不必一开始就逐项背下来。先把“会话头在前，媒体段在后，`m=` 切换当前媒体段”记牢。

### 3.3 会话级与媒体级：看它出现在什么位置

```sdp
v=0
o=- 1001 1 IN IP4 127.0.0.1
s=Scope Demo
c=IN IP4 0.0.0.0
t=0 0
a=control:*
a=sendonly
m=video 0 RTP/AVP 96
a=rtpmap:96 H264/90000
a=control:trackID=0
m=audio 0 RTP/AVP 0
a=control:trackID=1
```

从第一行到第一个 `m=` 之前，是**会话级**。从一个 `m=` 到下一个 `m=` 之前，是这一条媒体的描述。最后一个媒体段一直延续到文本结束。

因此：

- `a=control:*` 属于整个会话。
- `a=control:trackID=0` 属于视频。
- `a=control:trackID=1` 属于音频。
- 会话级 `c=` 可以被媒体级 `c=` 覆盖。
- 方向属性可以由媒体级覆盖会话级；没有方向属性时通常默认 `sendrecv`。

**不是所有字段都按同一套规则继承。** 尤其不能拿会话级 `control:*` 当作每条轨道的 SETUP 地址；它表示会话聚合控制，见第 7 节。

### 3.4 未知字段与未知属性，要分开处理

```sdp
a=x-vendor-feature:abc
```

如果不认识这个属性，可以保留原文并忽略它对当前功能的影响。属性机制就是为了允许扩展。

```text
x=some-new-top-level-field
```

这属于不认识的**顶层字段类型**。通用 SDP 规范要求不能继续把含这种字段的描述当作有效描述；严格解析器应拒绝。不能把“不认识的所有行都跳过”当成通用规则。

此外，“知道 `a=rtpmap` 的名字，但它的值损坏”也不等于未知属性，应当报告格式错误。

### 3.5 阶段 B：分段练习与检查

练习：在 3.3 的文本中，用横线分出会话区、视频区、音频区，再为每个 `a=` 写出所属区域。把视频段添加 `c=IN IP4 192.0.2.10`，音频的连接信息会跟着改变吗？

- [ ] 我能解释为什么 `a=control` 可以出现多次。
- [ ] 我知道当前媒体段何时切换，何时结束。
- [ ] 我不会把所有 `a=` 都存进一个覆盖旧值的全局 map。
- [ ] 我知道未知 `a=` 属性与未知顶层字段不是一回事。

## 4. 基础字段：把会话和轨道的“外壳”读懂

### 4.1 `v=`：SDP 格式版本

```sdp
v=0
```

这里的 `0` 是 SDP 版本，不是 RTSP 版本，不是视频编码版本，也不是“第 0 条轨道”。当前基本 SDP 仍使用 `v=0`。

### 4.2 `o=`：来源、会话标识和描述版本

```sdp
o=- 1001 3 IN IP4 192.0.2.10
```

| 片段 | 含义 |
| --- | --- |
| `-` | 用户名占位，表示没有提供实际用户名 |
| `1001` | SDP 会话标识，通常作为十进制字符串保存 |
| `3` | 这份会话描述的版本；描述改变时应更新 |
| `IN` | 网络类型，这里是 Internet |
| `IP4` | 地址类型；IPv6 时是 `IP6` |
| `192.0.2.10` | 描述来源的地址 |

这里的地址不是让你无条件连接的“最终媒体服务器地址”。实际 RTSP 请求看 URL，单播媒体传输看 SETUP。大数字不一定装得进 `int`，本章对会话标识和版本先保留字符串。

### 4.3 `s=`：给人看的会话名

```sdp
s=Office Camera
```

它不是 URL，也不能根据它推断用户名或设备路径。名称可以是 UTF-8 文本；没有有意义的名称时，常见占位是 `s=-`。

### 4.4 `t=`：时间描述

```sdp
t=0 0
```

它表示没有限定起止时间的持续会话，实时流中很常见。不是“播放 0 秒”，也不是 Unix 时间戳 0 到 0。

一般的 `t=<start> <stop>` 使用 NTP 时间格式的秒数，其时间基准与 Unix 时间不同。首次学习先会识别 `0 0`，预约会话、重复时间规则以后再展开。

RTSP 中还可能出现：

```sdp
a=range:npt=0-
```

`npt` 是正常播放时间，以媒体播放起点为参考。它与 `t=` 的会话可用时间不是同一件事。`Range` 请求头、点播和拖动可回看 [[03-RTSP协议]]。

### 4.5 `c=`：连接信息

```sdp
c=IN IP4 0.0.0.0
```

结构是 `c=<网络类型> <地址类型> <连接地址>`。

- 会话级出现时，为未单独指定 `c=` 的媒体提供默认连接信息。
- 媒体级出现时，对该媒体使用自己的连接信息。
- RTSP 单播设备常给出 `0.0.0.0` 占位；客户端不要拿它当远端主机直接连接。
- 组播可能给出 `c=IN IP4 239.1.2.3/16`，其中 `/16` 是 IPv4 组播 TTL，**不是 CIDR 子网掩码**。
- IPv6 的表示规则不同，不能把 IPv4 组播尾部的语法硬套过去。

先区分三类地址：`o=` 的来源地址、`c=` 的连接信息、`a=control` 的 RTSP 控制地址。它们可能长得相似，但用途不同。

### 4.6 `m=`：一条媒体描述开始

```sdp
m=video 0 RTP/AVP 96
m=audio 0 RTP/AVP 0 8
```

通用结构：

```text
m=<媒体类型> <端口> <传输配置> <格式1> <格式2> ...
```

| 片段 | 含义 |
| --- | --- |
| `video` / `audio` | 视频 / 音频，也可能有 application、text 等其他媒体类型 |
| `0` | 端口字段，在 RTSP 单播描述中常表示服务器没有端口偏好 |
| `RTP/AVP` | 使用 RTP 的音视频配置，这里 AVP 是 Audio/Video Profile |
| `96` 或 `0 8` | 在 RTP/AVP 语境中，是该媒体列出的 PT |

两个关键区别：

1. **RTSP 单播 SDP 中，端口 0 不代表这条轨道一定不可播放。** RTSP 1.0 附录 C 允许服务器用 0 表示无端口偏好；实际通过 SETUP 的 `Transport` 协商。SDP offer/answer 中端口 0 常用于拒绝媒体，那是另一种上下文。
2. **`RTP/AVP` 不能单独证明实际会走 UDP。** 在常见 RTSP 拉流中，客户端可以在 SETUP 时请求 UDP 或 TCP 交织传输；以服务器接受的 Transport 为准。

`m=audio ... 0 8` 是一条媒体描述列出两个格式，不是两条音频轨道。至于实际发送哪个、客户端是否支持，还要看服务器行为和具体应用流程。RTSP DESCRIBE 不是完整的 offer/answer 编码协商。

完整 SDP 还允许 `端口/端口数量`，例如 `49170/2`。本章程序只支持常见单端口写法，并会对不支持的写法明确报错。

### 4.7 方向属性：谁发送，谁接收

```sdp
a=sendonly
```

| 属性 | 直观含义 |
| --- | --- |
| `sendonly` | 对所描述的媒体角色，仅发送媒体 |
| `recvonly` | 仅接收媒体 |
| `sendrecv` | 发送并接收，未指定时的通常默认值 |
| `inactive` | 不发送也不接收媒体 |

方向的具体视角要结合发布/获取描述的应用约定判断。在常见 RTSP 服务器向拉流客户端描述自己将提供的流时，`sendonly` 通常对应服务器发送、客户端接收；不能看到 `sendonly` 就让自己的拉流客户端开始推视频。更不能把 SIP offer/answer 的双方角色规则直接套过来。

方向属性约束媒体，不能据此断言 RTCP 也只能单向发送。即使只接收媒体，RTP 接收方也可能发送 RTCP 接收报告。

同一作用域最多应出现一个方向属性；媒体级可覆盖会话级。重复或冲突时应报告问题。

### 4.8 其他字段：这一轮先学到“认识”

| 字段 / 属性 | 用途 | 初学处理方式 |
| --- | --- | --- |
| `i=` | 会话或媒体的说明文字 | 可以展示或保留 |
| `u=`、`e=`、`p=` | 相关网页、邮箱、电话 | 与本章核心解析无关 |
| `b=AS:2000` | 带宽提示，AS 常以 kb/s 表示 | 不要把它当成测量到的实时码率；不同修饰词单位不同 |
| `r=`、`z=` | 重复时间与时区调整 | 预约/周期会话进阶内容 |
| `a=framerate:25` | 帧率提示 | 不要用 90000 代替它；实际节奏还要看媒体 |
| `a=ptime:20` | 音频打包时长提示，单位毫秒 | 不等于网络往返延迟 |
| `a=rtcp:`、`a=rtcp-mux` | RTCP 地址/端口或复用扩展 | 识别后需有相应传输实现，不能只解析名字就声称支持 |

老资料还会出现 `k=` 传递密钥的方法，该机制已经过时。本章不实现它，也不要把真实密钥记录进知识库。

## 5. 载荷格式：把 `m=`、`rtpmap`、`fmtp` 连起来

### 5.1 PT 只是号码，不是编码名称

```sdp
m=video 0 RTP/AVP 96
a=rtpmap:96 H264/90000
```

意思是：“本条视频媒体允许 PT 96；在这里，96 代表 H.264，RTP 时钟频率为 90000 Hz。”

在常见 RTP/AVP 配置中：

- 一部分 PT 有标准静态映射，例如 `0 → PCMU/8000/1`、`8 → PCMA/8000/1`。
- `96～127` 是常用的动态 PT 范围，需要从 SDP 等会话描述中获知对应格式。
- 动态 PT 96 可以对应 H.264，也可以在另一份描述里对应 Opus 或其他编码。
- PT 是 RTP 头中 7 位字段，合法数值为 0～127；本章不把所有非动态 PT 的静态表都实现一遍。

正确索引方式是：

```text
媒体轨道 → 该轨道的 PT → 对应编码、时钟、fmtp
```

不要写一个全局 `map<int, Codec>` 覆盖所有轨道。在分开的 RTP 会话里，视频和音频可能使用相同 PT 数字，但含义不同。

### 5.2 `a=rtpmap`：把号码映射到格式

```sdp
a=rtpmap:97 MPEG4-GENERIC/48000/2
```

结构是：

```text
a=rtpmap:<PT> <编码名称>/<RTP时钟频率>[/编码参数]
```

对这里常见的音频编码，最后的编码参数表示声道数。示例可读为：

- PT：97。
- 编码名称：MPEG4-GENERIC，结合后面的参数可知本例是 AAC。
- RTP 时钟：每秒 48000 个刻度。
- 声道数：2。

视频常见 `H264/90000`，没有“视频声道”。不要把省略 `/2` 解释成“视频是单声道”。音频缺省声道通常为 1，但必须注意具体载荷格式的规定，例如 Opus 的 SDP 使用固定 `/2`，见 6.5。

### 5.3 时钟频率不等于帧率

H.264 常用 90000 Hz RTP 时钟。如果稳定 25 fps，一帧间隔对应的时间戳增量约为：

```text
90000 / 25 = 3600
```

若稳定 30 fps，约为 `3000`。`H264/90000` 本身不能告诉你究竟是 25、30 还是可变帧率。

音频例子：PCMU 的时钟为 8000 Hz，一包承载 20 ms 声音，对应时间戳增量：

```text
8000 × 0.020 = 160
```

Opus 使用 48000 Hz RTP 时钟，即使输入音频来自其他采样率，20 ms 对应的 RTP 时间戳增量仍是 `960`。时间戳还可能回绕，音视频间的同步需要后续 RTCP 等机制，本章先只学会读时钟。

### 5.4 `a=fmtp`：格式的进一步说明

```sdp
a=fmtp:96 packetization-mode=1;profile-level-id=42e01f;sprop-parameter-sets=Z0LgHtoCgPaEAAAAwAQAAAMAPI8UKZYA,aM4xUg==
```

结构是 `a=fmtp:<PT> <该载荷格式自己定义的参数文本>`。

- `96` 用来关联这条媒体下的格式 96。
- `packetization-mode`、`profile-level-id` 等由 H.264 的 RTP 载荷规范定义。
- 对本章几种编码，常见参数形式为 `key=value;key=value`，但 **SDP 通用语法不保证所有 fmtp 都是这种形式**。
- `sprop-parameter-sets` 的值内还有逗号和 Base64 的等号，必须保留。

按层次拆分：

```text
先拆 SDP 行：a = fmtp:96 ...
再拆属性：fmtp : 96 ...
再拆 PT 与参数原文：96 空格 参数原文
识别编码后，再按对应载荷规范解释参数
```

对于 `key=value` 型参数，**只在第一个等号处分开**。例如 `sprop-parameter-sets=abc,def==` 的值是 `abc,def==`，不能变成 `abc,def`。

### 5.5 数据缺失和编码不支持是两种问题

- 动态 PT 缺少 `rtpmap`：客户端没有足够的信息判断格式，应报告缺失，不猜它是 H.264。
- 静态 PT 0 没有 `rtpmap`：在 RTP/AVP 的上下文里可以查标准静态表，得到 PCMU。
- `rtpmap` 正确写了 `H265/90000`，但本客户端没有 H.265 解码器：描述可解析，客户端能力不足。
- 不认识的 `fmtp` 参数：先保留；是否可忽略取决于该编码规范和你支持的模式。
- `rtpmap:97 ...` 出现在只列出 PT 96 的媒体段：存在关联错误，不能静默挂到另一条轨道。

“文本解析成功”“格式受支持”“可以实际播放”是三项独立检查。

### 5.6 阶段 C 练习与检查

练习：

1. `H264/90000` 能推出视频宽高和帧率吗？
2. 48 kHz 音频一包 20 ms，时钟增量是多少？如果一包 10 ms 呢？
3. 视频与音频都写 PT=96，应如何组织查找表？
4. `fmtp` 引用一个未在本媒体 `m=` 中列出的 PT，你会怎么处理？

- [ ] 我知道 PT 96 不是 H.264 的全球固定编号。
- [ ] 我能用 `(媒体段, PT)` 关联 `m/rtpmap/fmtp`。
- [ ] 我不会把 90000 当成视频帧率。
- [ ] 我会保留 fmtp 原文，不会破坏 Base64 尾部的等号。
- [ ] 我能区别语法错误、缺少映射和不支持编码。

## 6. 常见编码：看到名字后还要看哪些参数

这一节的目标是建立“能判断下一步查什么”的地图。编码细节和 RTP 分片请继续读 [[07-H264与RTP封装]]。

### 6.1 H.264 / AVC

常见写法：

```sdp
m=video 0 RTP/AVP 96
a=rtpmap:96 H264/90000
a=fmtp:96 packetization-mode=1;profile-level-id=42e01f;sprop-parameter-sets=Z0LgHtoCgPaEAAAAwAQAAAMAPI8UKZYA,aM4xUg==
```

- `H264` 是视频编码名称；`90000` 是 RTP 时钟。
- `packetization-mode=1` 常表示允许通过 FU-A 把一个较大的 NAL 单元拆成多个 RTP 包。具体重组见 [[07-H264与RTP封装]]。
- `profile-level-id` 用十六进制表示 profile、约束标志和 level；它不是分辨率字符串。
- `sprop-parameter-sets` 通常是用逗号分隔的 Base64 SPS、PPS。它们是解码初始化所需的参数集，不能丢掉尾部 `=`。
- 有的设备通过 `sprop-parameter-sets` 提供参数，有的把 SPS/PPS 发送在 RTP 里，有的两者都做。客户端应按实际载荷处理。

### 6.2 H.265 / HEVC

常见写法：

```sdp
m=video 0 RTP/AVP 98
a=rtpmap:98 H265/90000
a=fmtp:98 sprop-vps=QAEMAf//AWAAAAMAkAAAAwAAAwBDoA;sprop-sps=QgEBAWAAAAMAkAAAAwAAAwBDoA==;sprop-pps=RAHBcrA=
```

也可能写成 `HEVC`，要结合设备和 RFC 7798 处理。H.265 通常需要 VPS、SPS、PPS；它们的 Base64 参数仍应原样保留。H.264 和 H.265 都是“视频”，但 NAL 类型、分片规则和解码器并不相同，不能只因字符串前缀相似就复用全部代码。

### 6.3 AAC / MPEG-4 generic

常见写法：

```sdp
m=audio 0 RTP/AVP 97
a=rtpmap:97 MPEG4-GENERIC/48000/2
a=fmtp:97 streamtype=5;profile-level-id=1;mode=AAC-hbr;config=1190;SizeLength=13;IndexLength=3;IndexDeltaLength=3
```

- `MPEG4-GENERIC` 是 RTP 载荷格式；常与 AAC 音频一起出现，但不能只看名称就忽略 `fmtp`。
- `/48000/2` 通常表示 RTP 时钟 48000、2 声道。
- `config=1190` 是十六进制的 AudioSpecificConfig 示例；AAC 解码器初始化时会用它。
- `SizeLength`、`IndexLength` 等告诉接收端如何从 RTP 载荷中取出一个或多个 AAC 访问单元。

`config` 的具体含义取决于 MPEG-4 generic 的配置；不要把它当成“采样率数字 1190”。

### 6.4 G.711：PCMU 与 PCMA

常见静态 PT：

```sdp
m=audio 0 RTP/AVP 0
a=rtpmap:0 PCMU/8000
m=audio 0 RTP/AVP 8
a=rtpmap:8 PCMA/8000
```

- PCMU 常对应 G.711 μ-law，PCMA 常对应 G.711 A-law。
- RTP 时钟为 8000 Hz；一声道是常见默认。
- 静态 PT 0/8 在 RTP/AVP 中有标准约定，但设备仍可能附带 `rtpmap`，以实际描述为准。
- G.711 通常不需要像 H.264 那样的 SPS/PPS；但采样率、声道和每包时长仍影响解码。

### 6.5 Opus

常见写法：

```sdp
m=audio 0 RTP/AVP 111
a=rtpmap:111 opus/48000/2
a=fmtp:111 minptime=10;useinbandfec=1
```

- Opus 在 RTP 中使用 48000 Hz 时钟。
- `/2` 是声道数的默认表达；它不是“内部只能双声道”的全部能力说明。
- `minptime`、`useinbandfec` 等是格式参数；是否启用 FEC、怎样解码要交给 Opus 实现。
- PT 111 是动态值，不是 Opus 永久固定的号码；另一台设备可以选择别的动态 PT。

### 6.6 这些编码的快速判断表

| SDP 片段 | 能得到什么 | 还不能得到什么 |
| --- | --- | --- |
| `H264/90000` | H.264、RTP 时钟 | 分辨率、帧率、是否有 B 帧 |
| `H265/90000` | H.265、RTP 时钟 | VPS/SPS/PPS 是否一定带在 SDP |
| `MPEG4-GENERIC/48000/2` | AAC 常见载荷、时钟、声道 | 仅凭这行不能完成 AAC 解码初始化 |
| `PCMU/8000` | G.711 μ-law、时钟 | 具体每包时长 |
| `opus/48000/2` | Opus、时钟、声道 | FEC 是否启用、实际包时长 |

### 6.7 阶段 D 检查

- [ ] 我知道 H.264 参数集和 AAC `config` 都属于初始化信息，但格式不同。
- [ ] 我知道 H.265 常见 VPS/SPS/PPS。
- [ ] 我不会把动态 PT 当作编码的永久编号。
- [ ] 我能解释为什么 `H264/90000` 不能直接给出帧率。

## 7. `a=control` 与 URL：从 SDP 生成 SETUP 地址

这是 RTSP 拉流中最容易“看起来懂了，实际请求错了”的部分。

### 7.1 会话控制和媒体控制

```sdp
a=control:*
m=video 0 RTP/AVP 96
a=control:trackID=0
m=audio 0 RTP/AVP 97
a=control:trackID=1
```

- 会话级 `a=control:*`：表示整个 presentation 的聚合控制 URL 通常继承完整基准地址。PLAY 常对这个聚合地址发送。
- 媒体级 `a=control:trackID=0`：表示视频轨道的控制 URL。SETUP 通常对这个地址发送。
- 媒体级 URL 可以是相对 URL，也可以是完整的 `rtsp://...` URL。
- 多轨道时，每个媒体段通常必须有自己的 `a=control`，否则无法可靠构造每条 SETUP URL。

### 7.2 解析基准 URL 的优先级

按 RTSP 1.0 的约定，解析相对 control URL 时依次查找：

1. RTSP 响应头 `Content-Base`。
2. RTSP 响应头 `Content-Location`。
3. DESCRIBE 请求使用的 URL。

`Content-Base` / `Content-Location` 是 RTSP Header，不是 SDP 行。保存它们时应去掉首尾空白，但不要随意删除 URL 路径中的斜杠。

例子：

| 基准 | SDP 属性 | 结果 |
| --- | --- | --- |
| `rtsp://cam/live/` | `trackID=0` | `rtsp://cam/live/trackID=0` |
| `rtsp://cam/live` | `trackID=0` | 简化实现要先规范化为目录，避免得到 `rtsp://cam/trackID=0` |
| `rtsp://cam/live/` | `rtsp://other/x` | `rtsp://other/x` |
| `rtsp://cam/live/` | `*`（会话级） | 继承完整基准 URL |
| `rtsp://cam/live/` | `/trackID=0` | 按 URI 解析规则从主机根路径开始，结果为 `rtsp://cam/trackID=0` |

最后一行体现相对 URL 和“以 `/` 开始的路径”的区别。生产代码应使用经过测试的 URI 解析器；本章的示例只覆盖 RTSP 设备最常见的三类写法，并把不支持情况报出来。

### 7.3 一个常见的错误示例

服务器返回：

```text
Content-Base: rtsp://cam/live/main
...
a=control:trackID=0
```

如果客户端直接字符串拼接，可能得到：

```text
rtsp://cam/live/maintrackID=0
```

也可能把 `Content-Base` 当文件名，错误地得到 `rtsp://cam/live/trackID=0`。真正答案取决于 URI 的基准路径规则和服务器的约定，不能只凭肉眼拼接。

建议程序保留三个值，便于排错：

```text
request_url       = DESCRIBE 发出的 URL
content_base      = 响应里的 Content-Base（可空）
control_raw       = SDP 原始 a=control 值
control_resolved  = 最终用于 SETUP/PLAY 的绝对 URL
```

日志中可隐藏用户名、密码和 token，但保留结构化 URL 的主机、路径和解析来源。

### 7.4 没有 session-level `a=control:*` 怎么办

- 单轨道描述中，服务器可能省略会话级 control；媒体级 control 仍可用于 SETUP。
- 多轨道描述中，若没有可用于聚合 PLAY 的会话级 control，设备可能要求对请求 URL 或某个媒体 URL 发 PLAY；应以设备文档和抓包为准。
- 不要看到缺少 `*` 就自动补一个；记录“聚合控制地址未知”，让 RTSP 层决定兼容策略。

### 7.5 阶段 E 检查

- [ ] 我知道 `Content-Base` 不是 SDP 字段。
- [ ] 我能说出相对 control URL 的三层基准优先级。
- [ ] 我能分出会话聚合 URL 与轨道 SETUP URL。
- [ ] 我不会无条件把尾部斜杠加到所有 URL 上。
- [ ] 我能在日志里同时展示原始和解析后的 control 值。

## 8. C++ 解析器设计：先设计数据，再写代码

### 8.1 建议的数据结构

```cpp
struct Format {
    int payload_type = -1;
    std::string encoding;
    int clock_rate = 0;
    int channels = 0;
    std::string fmtp;
};

struct Media {
    std::string kind;              // video / audio
    int port = -1;
    std::string proto;             // RTP/AVP
    std::vector<int> payload_types;
    std::string control;
    std::string resolved_control;
    std::map<int, Format> formats;
};

struct SessionDescription {
    std::string session_name;
    std::string session_control;
    std::string connection;
    std::vector<Media> media;
    std::vector<std::string> warnings;
};
```

设计原则：

1. 轨道是一等对象；不要把 `m=` 的值散落在全局变量。
2. 格式挂在对应的轨道下面；PT 只在本轨道内查找。
3. `fmtp` 先保留原文；识别编码后再解析专属参数。
4. 未知 `a=` 原文可保留，便于兼容厂商扩展。
5. 警告和致命错误分开：未知属性通常是警告，缺少 `v=`、错误的 `m=` 或动态 PT 无法关联则可能是错误。

### 8.2 按行处理的状态机

```text
开始
  ↓
验证每行恰好有一个合法的字段字母和 '='
  ↓
v/o/s/t 等会话字段
  ↓ 遇到 m=
创建 Media，并把 current_media 指向它
  ↓
继续读取该媒体的 c=/a=/b=
  ↓ 再遇到 m=
结束上一 Media，创建下一 Media
  ↓ 文本结束
验证至少有一个 Media、必要字段和 PT 关联
```

`current_media` 是关键状态：读到 `a=rtpmap` 时，必须知道它属于哪条 `m=`。如果 `a=rtpmap` 出现在第一个 `m=` 以前，应报告作用域/顺序问题，而不是塞进某个全局 map。

### 8.3 解析属性值的通用策略

```text
a=名称:值        → name=名称，value=值
 a=名称           → name=名称，value为空（属性存在）
```

- 只在属性值的第一个 `:` 处分割；URL 值中可能还有 `:`。
- 对 `rtpmap` 先取 `PT`，再按 `/` 拆编码、时钟和可选参数。
- 对 `fmtp` 先取 `PT`，其余全部保留；不要先按 `;` 拆成丢失上下文的 map。
- 对 `control` 保存完整值，解析 URL 时再处理 `*`、相对和绝对形式。

### 8.4 错误分类

| 类别 | 例子 | 建议处理 |
| --- | --- | --- |
| 语法错误 | `m=video RTP/AVP 96` 少端口 | 返回带行号的错误 |
| 顺序/作用域错误 | `a=rtpmap` 出现在任何 `m=` 前 | 返回错误或明确不支持 |
| 关联错误 | `rtpmap:97` 不在本媒体 PT 列表 | 返回错误 |
| 能力错误 | 客户端没有 H.265 解码器 | 解析成功，标记不可播放 |
| 兼容性警告 | 未知 `a=x-vendor` | 保留并警告 |
| URL 错误 | control 为空或 URI 不完整 | 返回错误，避免错误 SETUP |

错误消息应该像这样：

```text
line 12: rtpmap payload type 97 is not listed by the current m= section
```

不要只抛出 `parse failed`，否则新手无法定位输入。

## 9. 实验材料：准备一个不依赖摄像头的 SDP 文件

在工作目录新建 `sample.sdp`，内容如下。保存时可以用 UTF-8；协议字段本身使用 ASCII。

```sdp
v=0
o=- 1001 3 IN IP4 192.0.2.10
s=Beginner Camera
c=IN IP4 0.0.0.0
t=0 0  
a=control:*
a=sendonly

m=video 0 RTP/AVP 96
a=rtpmap:96 H264/90000
a=fmtp:96 packetization-mode=1;profile-level-id=42e01f;sprop-parameter-sets=Z0LgHtoCgPaEAAAAwAQAAAMAPI8UKZYA,aM4xUg==
a=control:trackID=0

m=audio 0 RTP/AVP 97
a=rtpmap:97 MPEG4-GENERIC/48000/2
a=fmtp:97 streamtype=5;profile-level-id=1;mode=AAC-hbr;config=1190;SizeLength=13;IndexLength=3;IndexDeltaLength=3
a=control:trackID=1
```

说明：

- 空行只是为了阅读；解析器不能把空行误当成新媒体。
- `t=0 0  ` 的尾部空格用于练习 trim。
- `192.0.2.10` 是文档保留地址，不会指向真实摄像头。
- 这份文件描述了能力和控制地址，不包含任何真实媒体包。

## 10. C++17 示例：读取、解析、打印和生成 SETUP URL

### 10.1 这段示例做什么、暂时不做什么

它会：

- 读取一个 SDP 文件。
- 识别会话名、会话 control、每条 `m=`、PT、`rtpmap`、`fmtp`、媒体 control。
- 对常见相对/绝对 control URL 生成示例 SETUP 地址。
- 用行号报告格式和关联错误。

它不会：

- 连接 RTSP 服务或读取 DESCRIBE 的 TCP 消息体。
- 解析所有 RFC 字段、所有 URI 语法、所有 codec 专属 fmtp。
- 接收 RTP、重组 NAL 或解码画面。

将以下内容保存为 `sdp_inspect.cpp`：

```cpp
#include <algorithm>
#include <cctype>
#include <fstream>
#include <iostream>
#include <map>
#include <sstream>
#include <stdexcept>
#include <string>
#include <vector>

struct Format {
    int pt = -1;
    std::string encoding;
    int clock = 0;
    int channels = 0;
    std::string fmtp;
};

struct Media {
    std::string kind;
    int port = -1;
    std::string proto;
    std::vector<int> pts;
    std::string control;
    std::map<int, Format> formats;
};

struct Description {
    std::string name;
    std::string session_control;
    std::vector<Media> media;
    std::vector<std::string> warnings;
};

static std::string trim(std::string s) {
    auto not_space = [](unsigned char c) { return !std::isspace(c); };
    s.erase(s.begin(), std::find_if(s.begin(), s.end(), not_space));
    s.erase(std::find_if(s.rbegin(), s.rend(), not_space).base(), s.end());
    return s;
}

static std::vector<std::string> words(const std::string& s) {
    std::istringstream in(s);
    std::vector<std::string> out;
    for (std::string word; in >> word;) out.push_back(word);
    return out;
}

static int to_int(const std::string& s, const std::string& what, int line) {
    try {
        size_t used = 0;
        int value = std::stoi(s, &used);
        if (used != s.size()) throw std::invalid_argument("trailing text");
        return value;
    } catch (const std::exception&) {
        throw std::runtime_error("line " + std::to_string(line) +
                                 ": invalid " + what + ": " + s);
    }
}

static bool has_pt(const Media& m, int pt) {
    return std::find(m.pts.begin(), m.pts.end(), pt) != m.pts.end();
}

static void parse_rtpmap(Media& m, const std::string& value, int line) {
    const auto sp = value.find_first_of(" \t");
    if (sp == std::string::npos)
        throw std::runtime_error("line " + std::to_string(line) +
                                 ": rtpmap needs PT and encoding");
    const int pt = to_int(value.substr(0, sp), "payload type", line);
    if (!has_pt(m, pt))
        throw std::runtime_error("line " + std::to_string(line) +
                                 ": rtpmap PT is not listed by m=");
    const auto parts = words(value.substr(sp + 1));
    if (parts.size() != 1)
        throw std::runtime_error("line " + std::to_string(line) +
                                 ": rtpmap has unexpected spaces");
    std::vector<std::string> fields;
    std::stringstream ss(parts[0]);
    for (std::string item; std::getline(ss, item, '/');) fields.push_back(item);
    if (fields.size() < 2 || fields.size() > 3)
        throw std::runtime_error("line " + std::to_string(line) +
                                 ": rtpmap must be name/clock[/parameter]");
    Format& f = m.formats[pt];
    f.pt = pt;
    f.encoding = fields[0];
    f.clock = to_int(fields[1], "RTP clock", line);
    if (fields.size() == 3)
        f.channels = to_int(fields[2], "channels", line);
}

static void parse_fmtp(Media& m, const std::string& value, int line) {
    const auto sp = value.find_first_of(" \t");
    if (sp == std::string::npos)
        throw std::runtime_error("line " + std::to_string(line) +
                                 ": fmtp needs PT and parameter text");
    const int pt = to_int(value.substr(0, sp), "payload type", line);
    if (!has_pt(m, pt))
        throw std::runtime_error("line " + std::to_string(line) +
                                 ": fmtp PT is not listed by m=");
    // Keep the original remainder. Base64 values may contain '=' and ';' is
    // codec-specific; splitting here would silently destroy information.
    m.formats[pt].pt = pt;
    m.formats[pt].fmtp = trim(value.substr(sp + 1));
}

static void parse_attribute(Description& d, Media* current,
                            const std::string& text, int line) {
    const auto colon = text.find(':');
    const std::string name = colon == std::string::npos ? text : text.substr(0, colon);
    const std::string value = colon == std::string::npos ? "" : text.substr(colon + 1);
    if (name == "control") {
        if (current) current->control = value;
        else d.session_control = value;
    } else if (name == "rtpmap") {
        if (!current) throw std::runtime_error("line " + std::to_string(line) + ": rtpmap before m=");
        parse_rtpmap(*current, value, line);
    } else if (name == "fmtp") {
        if (!current) throw std::runtime_error("line " + std::to_string(line) + ": fmtp before m=");
        parse_fmtp(*current, value, line);
    } else if (name == "sendonly" || name == "recvonly" ||
               name == "sendrecv" || name == "inactive") {
        // Direction is useful to inspect, but this small example does not store it.
    } else {
        d.warnings.push_back("line " + std::to_string(line) +
                             ": ignored attribute a=" + text);
    }
}

static Description parse_sdp(const std::string& source) {
    Description d;
    bool saw_v = false, saw_o = false, saw_s = false, saw_t = false;
    Media* current = nullptr;
    std::istringstream input(source);
    std::string raw;
    int line = 0;
    while (std::getline(input, raw)) {
        ++line;
        if (!raw.empty() && raw.back() == '\r') raw.pop_back();
        raw = trim(raw);
        if (raw.empty()) continue;
        if (raw.size() < 2 || raw[1] != '=' ||
            !std::islower(static_cast<unsigned char>(raw[0])))
            throw std::runtime_error("line " + std::to_string(line) + ": expected x=value");
        const char type = raw[0];
        const std::string value = raw.substr(2);
        if (type == 'v') {
            if (value != "0") throw std::runtime_error("line " + std::to_string(line) + ": only v=0 is supported");
            saw_v = true;
        } else if (type == 'o') saw_o = true;
        else if (type == 's') { saw_s = true; d.name = value; }
        else if (type == 't') saw_t = true;
        else if (type == 'm') {
            const auto p = words(value);
            if (p.size() < 4) throw std::runtime_error("line " + std::to_string(line) + ": m= needs media port proto PT");
            Media media;
            media.kind = p[0];
            media.port = to_int(p[1], "media port", line);
            media.proto = p[2];
            for (size_t i = 3; i < p.size(); ++i) {
                const int pt = to_int(p[i], "payload type", line);
                if (pt < 0 || pt > 127) throw std::runtime_error("line " + std::to_string(line) + ": PT outside 0..127");
                media.pts.push_back(pt);
                media.formats[pt].pt = pt;
            }
            d.media.push_back(std::move(media));
            current = &d.media.back();
        } else if (type == 'a') parse_attribute(d, current, value, line);
        else if (type == 'c' || type == 'b' || type == 'i') {
            // Recognized enough for this lab; keep only core fields above.
        } else {
            throw std::runtime_error("line " + std::to_string(line) + ": unsupported top-level field: " + std::string(1, type));
        }
    }
    if (!saw_v || !saw_o || !saw_s || !saw_t)
        throw std::runtime_error("SDP must contain v=, o=, s= and t=");
    if (d.media.empty()) throw std::runtime_error("SDP must contain at least one m=");
    return d;
}

static std::string join_url(const std::string& base, const std::string& control) {
    if (control.empty()) throw std::runtime_error("empty a=control");
    if (control == "*") return base;
    if (control.find("rtsp://") == 0 || control.find("rtsps://") == 0) return control;
    if (base.empty()) throw std::runtime_error("relative control has no base URL");
    const auto scheme = base.find("://");
    const auto slash = base.find('/', scheme == std::string::npos ? 0 : scheme + 3);
    const std::string root = slash == std::string::npos ? base : base.substr(0, slash);
    if (!control.empty() && control[0] == '/') return root + control;
    if (base.back() == '/') return base + control;
    return base + "/" + control; // common RTSP Content-Base behavior
}

int main(int argc, char** argv) {
    if (argc < 2 || argc > 3) {
        std::cerr << "Usage: " << argv[0] << " sample.sdp [base-rtsp-url]\n";
        return 2;
    }
    try {
        std::ifstream file(argv[1], std::ios::binary);
        if (!file) throw std::runtime_error("cannot open SDP file: " + std::string(argv[1]));
        std::ostringstream text; text << file.rdbuf();
        const Description d = parse_sdp(text.str());
        const std::string base = argc == 3 ? argv[2] : "rtsp://127.0.0.1:8554/live/";
        std::cout << "session: " << d.name << "\n";
        std::cout << "aggregate control: " << (d.session_control.empty() ? "<none>" : join_url(base, d.session_control)) << "\n";
        for (size_t i = 0; i < d.media.size(); ++i) {
            const Media& m = d.media[i];
            std::cout << "media[" << i << "]: " << m.kind << " proto=" << m.proto << " PTs=";
            for (int pt : m.pts) std::cout << pt << ' ';
            std::cout << "\n  SETUP: " << (m.control.empty() ? "<missing>" : join_url(base, m.control)) << "\n";
            for (int pt : m.pts) {
                const auto& f = m.formats.at(pt);
                std::cout << "  PT " << pt << ": " << (f.encoding.empty() ? "<static/default>" : f.encoding)
                          << " clock=" << f.clock << " channels=" << f.channels << "\n";
                if (!f.fmtp.empty()) std::cout << "    fmtp=" << f.fmtp << "\n";
            }
        }
        for (const auto& warning : d.warnings) std::cerr << "warning: " << warning << "\n";
    } catch (const std::exception& e) {
        std::cerr << "error: " << e.what() << "\n";
        return 1;
    }
}
```

### 10.2 逐段读懂代码

- `Media` 是遇到 `m=` 才创建的当前媒体段；`current` 保存状态。
- `Format` 用 PT 做键，因此 `fmtp:96` 会挂到当前媒体的 96，而不是覆盖音频的 96。
- `parse_rtpmap` 先确认 PT 出现在 `m=`，再读取编码和时钟。
- `parse_fmtp` 故意只截出 PT 后保留剩余原文；这是为了不破坏 codec 专属语法。
- `join_url` 是**教学用常见规则**，不是完整 RFC URI 库；真实项目应补齐规范化、查询字符串和异常 URL 测试。
- 未知的 `a=` 作为 warning；未知顶层字段作为错误，体现第 3.4 节的区别。

## 11. 运行方法：Windows 和 Linux 分开操作

### 11.1 Windows PowerShell + MinGW g++

在包含 `sample.sdp` 和 `sdp_inspect.cpp` 的目录执行：

```powershell
g++ -std=c++17 -Wall -Wextra -pedantic -O2 sdp_inspect.cpp -o sdp_inspect.exe
.\sdp_inspect.exe .\sample.sdp rtsp://127.0.0.1:8554/live/
```

预期输出的关键部分：

```text
session: Beginner Camera
aggregate control: rtsp://127.0.0.1:8554/live/
media[0]: video proto=RTP/AVP PTs=96
  SETUP: rtsp://127.0.0.1:8554/live/trackID=0
  PT 96: H264 clock=90000 channels=0
media[1]: audio proto=RTP/AVP PTs=97
  SETUP: rtsp://127.0.0.1:8554/live/trackID=1
  PT 97: MPEG4-GENERIC clock=48000 channels=2
```

如果 PowerShell 不允许执行当前目录程序，命令前的 `.\` 是必要的；这不是编译错误。若提示找不到 `g++`，安装 MinGW-w64 或 MSYS2 后，把其 `bin` 目录加入 PATH，再重新打开 PowerShell。

MSVC 也可以：

```powershell
cl /std:c++17 /EHsc /W4 /O2 sdp_inspect.cpp
.\sdp_inspect.exe .\sample.sdp rtsp://127.0.0.1:8554/live/
```

`cl` 命令要在“Developer PowerShell for VS”中执行。若输出中文乱码，先保证源文件 UTF-8；本实验的协议字段和日志主要是 ASCII。

### 11.2 Linux Bash + g++

在同一目录执行：

```bash
g++ -std=c++17 -Wall -Wextra -pedantic -O2 sdp_inspect.cpp -o sdp_inspect
./sdp_inspect ./sample.sdp rtsp://127.0.0.1:8554/live/
```

Debian/Ubuntu 缺少编译器时：

```bash
sudo apt update
sudo apt install build-essential
```

如果文件从 Windows 复制后出现 `^M`，程序已经兼容行尾的 `\r`；若其它工具报错，可以使用：

```bash
sed -i 's/\r$//' sample.sdp
```

Linux 运行时注意大小写：`Sample.sdp` 和 `sample.sdp` 是两个文件。Windows 的盘符路径不能直接当作 Linux 路径；在 WSL 中应使用 `/mnt/d/learn/...`，但需要先确保 WSL 发行版已正确安装。

### 11.3 不依赖网络的自测命令

```text
把 sample.sdp 复制一份为 bad.sdp
将 a=rtpmap:97 改成 a=rtpmap:99
重新运行
预期：退出码为 1，并指出 rtpmap PT 不在 m= 列表中
```

Windows PowerShell 查看退出码：`$LASTEXITCODE`。Linux Bash 查看退出码：`echo $?`。

## 12. 连接真实 RTSP 流：把离线解析接到上一章

### 12.1 推荐接入顺序

1. 用上一章的 Socket 客户端发送 `DESCRIBE`，带 `Accept: application/sdp`。
2. 读取完整 RTSP Header，再按 `Content-Length` 补齐 body。
3. 保存 `Content-Base`、`Content-Location` 和 DESCRIBE request URL。
4. 检查状态码是 200、`Content-Type` 是 `application/sdp`。
5. 把 body 交给 `parse_sdp`，不要把 Header 一起传入。
6. 打印每条媒体的 PT、编码、fmtp 和解析后的 control。
7. 只选择客户端支持的媒体和格式，再进行 SETUP。
8. 每条轨道分别根据解析结果构造 SETUP URL，保存 RTSP Session 和 Transport 响应。

RTSP 方法、Session 和 Transport 的完整过程见 [[03-RTSP协议]]；RTP 接收见 [[05-RTP协议]]。

### 12.2 真实设备可能出现的差异

- `Content-Base` 为空，只有 `Content-Location`。
- `a=control` 可能是 `trackID=1`、`streamid=0`、`h264`，甚至是完整 URL。
- 编码可能写 `H265`、`HEVC`、`MPEG4-GENERIC` 的大小写变体；比较名称时可先做大小写不敏感判断，但保留原文。
- PT 96 在每条媒体中都可能出现；不能跨媒体复用映射。
- 某些设备缺少 `rtpmap`，但使用静态 PT；只有在你确认 RTP profile 的静态表后才能补默认值。
- SDP 可能有厂商扩展属性；先保留并记录，再决定是否影响播放。
- 描述可能只有视频，没有音频；不要假定两者必然成对出现。

### 12.3 一个安全的日志摘要

```text
DESCRIBE status=200 content_type=application/sdp body_bytes=312
media=video pt=96 codec=H264 clock=90000 control=trackID=0
media=audio pt=97 codec=MPEG4-GENERIC clock=48000 channels=2 control=trackID=1
```

不要在日志里写入完整的 `rtsp://user:password@...`、鉴权响应、私有 token 或 `sprop` 之外的敏感内容。密码清理和重连策略见 [[30-runbooks/05-断线重连与故障排查|断线重连与故障排查]]。

## 13. 常见错误：按层排查，不要盲目改代码

| 现象 | 可能原因 | 第一检查项 | 修复方向 |
| --- | --- | --- | --- |
| 收到 200，但解析成“没有 m=” | 把 RTSP Header 传给解析器，或 body 没读完 | 打印 `Content-Length` 与实际 body 字节数 | 先分离并补齐 body |
| 只收到视频，找不到音频 | 设备本来没有音频或 URL 是无声码流 | 查看所有 `m=`，不要只取第一段 | 选择正确的媒体路径 |
| `PT=96` 被认成 AAC | 使用全局 PT map 或写死 96=H.264 | 按媒体段打印 PT | 以当前 SDP 的 rtpmap 为准 |
| `H264/90000` 被当作 90000 fps | 混淆时钟和帧率 | 查看是否有 `framerate` 或 RTP 时间戳 | 用 RTP/RTCP 统计真实节奏 |
| SETUP 返回 404 | 相对 control URL 拼错、Content-Base 尾斜杠错 | 同时打印 raw/base/resolved | 用 URI 解析规则和抓包核对 |
| SETUP 返回 461 | 请求的 Transport 不被支持 | 查看服务器响应的 Transport | 改 UDP/TCP interleaved 或端口策略 |
| SETUP 成功，PLAY 455 | 状态机顺序、Session 或聚合 URL 错 | 检查每条 SETUP 和 PLAY 的 Session | 先完整 SETUP，再用正确控制 URL PLAY |
| RTP 到达但解码器报参数不足 | H.264/H.265 参数集或 AAC config 缺失/损坏 | 检查 fmtp 和 RTP 首包 | 按对应 RFC 还原初始化数据 |
| Linux 解析失败、Windows 正常 | CRLF、路径大小写或编码差异 | 查看行尾和实际路径 | 统一 UTF-8/换行，使用正确路径 |
| 未知 `a=x-...` 直接失败 | 把厂商扩展当成未知顶层字段 | 查看行是否以 `a=` 开头 | 保留扩展并记录 warning |

### 13.1 看到“播放没画面”时的分层清单

```text
RTSP：DESCRIBE / SETUP / PLAY 是否成功？
  ↓
SDP：是否有 video？PT、rtpmap、fmtp、control 是否完整？
  ↓
Transport：客户端 UDP 端口是否收到 RTP？或 TCP interleaved 通道是否正确？
  ↓
RTP：版本、PT、序号、时间戳、Marker 是否合理？是否丢包/乱序？
  ↓
Codec：FU-A/聚合包是否正确重组？参数集是否存在？
  ↓
Decoder：解码器是否支持 profile/level、分辨率和配置？
```

每一层先证明自己的输入成立，再看下一层。否则在“画面黑”时改 SDP 字符串，通常只会掩盖真正的 RTP 或解码问题。

## 14. 练习题与参考答案

### 14.1 基础理解

1. 一台设备的主码流没有声音、子码流有声音，它们一定出现在同一份 SDP 中吗？
2. `m=video 0 RTP/AVP 96` 中的 0、96 各是什么？
3. `H264/90000` 中 90000 是帧率吗？
4. 为什么不能把视频 PT=96 的映射拿去解释音频 PT=96？
5. `Content-Base` 是 SDP 行吗？

### 14.2 解析与 URL

6. 下面哪一行会切换当前媒体：`a=control:x`、`m=audio 0 RTP/AVP 97`、`a=fmtp:97 x=y`？
7. `a=fmtp:96 sprop-parameter-sets=abc==` 的值为什么不能按所有 `=` 切开？
8. 若 `Content-Base` 是 `rtsp://cam/live/`，媒体 control 是 `trackID=1`，常见 SETUP URL 是什么？
9. `a=control:*` 通常用于单轨道 SETUP 还是聚合控制？
10. PT 97 出现在 `rtpmap` 但没有列在本媒体 `m=` 中，你会报告什么？

### 14.3 代码与排错

11. 为什么解析器要保存 `current_media`？
12. 收到 RTSP 200，但 `Content-Length=312` 而程序只读到 150 字节，能开始解析吗？
13. Windows 运行成功、Linux 失败时，列出两个与代码无关的检查点。
14. H.264 `fmtp` 有 `sprop-parameter-sets`，仍然没有画面，下一层该查什么？
15. 设计一个测试：证明程序不会把未知 `a=x-vendor` 当成致命错误。

### 14.4 参考答案（先独立作答）

1. 不一定。设备可以按不同 URL、不同 SDP 或不同 profile 提供；不能假定一份描述必有音频。
2. 0 是 RTSP 单播 SDP 中常见的“服务器不指定媒体接收端口”占位；96 是 PT，必须结合本媒体的 `rtpmap` 解读。
3. 不是。90000 是 RTP 时钟频率；帧率要看媒体时间戳或其它描述。
4. PT 只在所属 RTP 媒体上下文中解释；同一数字可在另一条媒体代表别的编码。
5. 不是。它是 RTSP 响应头；用于解析 SDP 中的相对 control URL。
6. `m=audio...`。它开始一个新的媒体段。
7. Base64 值可以包含 `=` 作为填充；应保留从第一个 `=` 后面的全部文本。
8. `rtsp://cam/live/trackID=1`。
9. 聚合控制，通常用于对整个 presentation 发 PLAY；单轨道 SETUP 使用媒体级 control。
10. 关联错误；严格实现应指出 PT 没有出现在当前 `m=` 的格式列表中。
11. 因为 `a=rtpmap`、`a=fmtp`、`a=control` 的归属由最近的 `m=` 决定。
12. 不能。要继续读满 body，并保留同一次读取中超出的下一条消息字节。
13. 文件名大小写、路径写法、行尾 CRLF、编译器版本、PATH、默认工作目录都可以检查。
14. 检查 RTP 是否到达、FU-A 是否正确重组、参数集是否传给解码器、profile/level 和解码器能力。
15. 在样例中添加 `a=x-vendor:test`；程序应正常打印媒体并给出 warning，而不是退出码 1。

## 15. 阶段检查清单与最终验收

### 阶段 A：前置与定位

- [ ] 我能解释 RTSP、SDP、RTP、RTCP 的边界。
- [ ] 我能从 RTSP 响应中找到 SDP body。
- [ ] 我知道 `Content-Length` 是字节数。

### 阶段 B：语法与作用域

- [ ] 我能解释 `x=value` 和 CRLF。
- [ ] 我能用 `m=` 分出每个媒体段。
- [ ] 我知道会话级和媒体级属性的区别。
- [ ] 我能区别未知 `a=` 与未知顶层字段。

### 阶段 C：格式映射

- [ ] 我能从 `m=` 列出 PT。
- [ ] 我能用 `rtpmap` 解释 PT、编码、时钟和声道。
- [ ] 我能保留并解释 `fmtp` 的作用。
- [ ] 我知道动态 PT 不能写死成某个编码。

### 阶段 D：编码与 URL

- [ ] 我能指出 H.264 的 SPS/PPS、H.265 的 VPS/SPS/PPS、AAC 的 config 可能出现在哪里。
- [ ] 我能算相对 control URL 的常见结果。
- [ ] 我知道 `Content-Base`、`Content-Location`、请求 URL 的优先级。

### 阶段 E：编码实验

- [ ] 我在 Windows 或 Linux 编译运行了示例。
- [ ] 我看到了视频、音频、PT、clock 和 SETUP URL。
- [ ] 我修改一个 PT 后，程序能给出带行号的错误。
- [ ] 我加入未知 `a=x-vendor` 后，程序给出 warning 并继续运行。

### 阶段 F：连接真实流前

- [ ] 我知道本示例没有实现 Socket、认证、RTP、重组和解码。
- [ ] 我能把解析器接在 DESCRIBE body 之后。
- [ ] 我知道 PLAY 200 不等于已收到可解码的视频。
- [ ] 我能按 RTSP、SDP、Transport、RTP、Codec、Decoder 分层排错。

### 最终验收任务

找一份没有见过的 DESCRIBE SDP，写一张表：

| 媒体 | PT | 编码 | RTP 时钟 | 声道 | fmtp 关键项 | 原始 control | 解析后 SETUP URL |
| --- | --- | --- | --- | --- | --- | --- | --- |
|  |  |  |  |  |  |  |  |

然后用不看答案的方式回答：

```text
它有几条媒体？
每条媒体支持哪些 PT？
每个 PT 如何映射？
客户端应该对哪些 URL 发 SETUP？
PLAY 应该使用哪个控制地址？
如果没有 RTP，下一步查哪一层？
```

全部答对，再进入 [[05-RTP协议]]；如果 URL 或 fmtp 仍然不确定，先重做第 7 节和第 14 节练习。

## Related

- [[00-RTSP学习总纲]]
- [[01-网络与Socket基础]]
- [[02-音视频基础]]
- [[03-RTSP协议]]
- [[05-RTP协议]]
- [[06-RTCP协议]]
- [[07-H264与RTP封装]]
- [[08-FFmpeg与解码]]
- [[10-C++客户端架构]]
- [[30-runbooks/02-抓包分析一次RTSP拉流|抓包分析一次 RTSP 拉流]]
- [[30-runbooks/05-断线重连与故障排查|断线重连与故障排查]]

## 参考标准

- [RFC 8866：SDP](https://www.rfc-editor.org/rfc/rfc8866)
- [RFC 2326：RTSP 1.0](https://www.rfc-editor.org/rfc/rfc2326)
- [RFC 7826：RTSP 2.0](https://www.rfc-editor.org/rfc/rfc7826)
- [RFC 3550：RTP/RTCP](https://www.rfc-editor.org/rfc/rfc3550)
- [RFC 3551：RTP/AVP](https://www.rfc-editor.org/rfc/rfc3551)
- [RFC 6184：H.264 over RTP](https://www.rfc-editor.org/rfc/rfc6184)
- [RFC 7798：H.265 over RTP](https://www.rfc-editor.org/rfc/rfc7798)
- [RFC 3640：MPEG-4 generic/AAC over RTP](https://www.rfc-editor.org/rfc/rfc3640)
- [RFC 7587：Opus over RTP](https://www.rfc-editor.org/rfc/rfc7587)
