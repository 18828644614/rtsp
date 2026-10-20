---
type: topic
status: active
created: 2026-10-16
updated: 2026-10-20
tags:
  - rtp
  - rtsp
  - beginner
  - windows
  - cplusplus
aliases:
  - RTP 入门
  - 实时传输协议
---

# RTP 协议：从零理解实时媒体包

> 适用对象：会一些 C++ 语法，第一次学习音视频网络传输的读者。本文解释用到的术语，不要求你先懂 RTP、Socket、视频编码或 Wireshark。
>
> 学习产出：能手工读懂 RTP 十六进制报文；在 Windows 上编译 C++17 程序，安全提取包头和载荷；完成本机 UDP 收发；理解 TCP 交织；能解释包序号、媒体时间和接收顺序的区别。
>
> 阅读方法：先看 §0，再按阶段学习。三个实验不需要摄像头、RTSP 服务器、FFmpeg 或第三方 C++ 库。示例用于学习传输和解析，后面继续衔接媒体解包、解码和播放。§18 记录实际验证范围，学习检查框由你完成后自行勾选。

## 0. 先规划：完整学习范围和顺序

### 0.1 学完后要能回答什么

1. RTSP、SDP、RTP、RTCP、UDP、TCP 分别负责什么？
2. RTP 头有多长，怎样找到真正的载荷？
3. 包序号、媒体时间戳、SSRC 三个数字为什么不能混用？
4. 为什么 PT=96 在不同轨道上可能代表不同编码？
5. 一张画面拆成三个 RTP 包，哪些字段相同，哪些不同？
6. 收到 `100、102、101`，为什么不能直接说“丢了一个包”？
7. Windows 怎样初始化网络、接收 UDP 并解析字节？
8. TCP 为什么要先恢复消息边界，才能解析 RTP？

### 0.2 按这个顺序学习

建议分 8 次，每次 45～90 分钟；实验可以拆成多次。时间只是参考，以产出和检查清单为准。

| 阶段 | 阅读位置 | 学习范围 | 最少前置 | 产出 / 通过标准 |
| --- | --- | --- | --- | --- |
| A | §1～§2 | 字节、位运算、媒体、网络分层 | C++ 变量、数组、函数 | 手画“编码器→网络→画面”流程 |
| B | §3～§4 | 全部 RTP 头字段、可变头、手工拆包 | A | 算出头长、载荷位置和字段值 |
| C | §5～§6 | 序号、回绕、乱序、时间戳、时钟 | B | 手算顺序、缺口和时间差 |
| D | §7～§8 | PT/SDP、SSRC、Marker、媒体边界 | B～C | 列映射表，区分包、NAL、帧 |
| E | §9～§10 | Windows 环境、离线 C++ 解析 | A～D、C++ 容器 | 编译并解释输出，做错误输入实验 |
| F | §11 | Windows UDP 本机收发 | E、最小 Socket 概念 | 两个终端收 6 个包，解释顺序 |
| G | §12～§14 | TCP 交织、接收器流程、缓冲和抓包 | E～F | 解释半包/多包，画接收器结构 |
| H | §15～§19 | 排错、练习、答案、总验收、资料 | 前面各阶段 | 独立分析新报文，知道后续方向 |

### 0.3 必学、后续和选修

**本章必须掌握：**12 字节固定头、可变头和尾部填充；大端读取；序号和时间戳的区别；动态 PT 查 SDP；SSRC 分流；Windows UDP API 的最小用法；TCP 组帧；长度校验；乱序、重复和暂时缺包。

**理解接口，后续实现：**完整重排缓冲、最终丢包统计、音视频同步、H.264 分片重组、RTSP 控制、解码和播放时钟。衔接 [[06-RTCP协议]]、[[07-H264与RTP封装]]、[[03-RTSP协议]]、[[08-FFmpeg与解码]]、[[10-C++客户端架构]]。

**进阶选修：**SRTP（媒体加密与身份校验）、RTX（另行约定的重传格式）、FEC（冗余信息辅助恢复）、WebRTC 的复用与拥塞控制。第一轮知道用途即可，不能假设普通 RTP 自动提供这些能力。

### 0.4 新手第一遍的具体安排

先读 §1～§2，说清“控制信息”和“媒体数据”；再用纸笔拆 §4 的 16 个字节；然后做 §5～§8 的小练习。动手时先跑离线程序，确认解析正确后再做 UDP，最后学习 TCP 组帧。

每次只修改一个变量，例如先只改 PT，再只改序号。结束时写四句话：“输入是什么、程序做了什么、输出是什么、哪里还不确定”。暂时不用“能不能显示摄像头画面”作为唯一进度指标。

## 1. 前置知识：先把基本词说清楚

### 1.1 画面怎样变成网络数据

**前置：**不需要音视频知识。

视频可以理解为连续播放的一张张图像。一张图像叫一**帧**；30 fps 表示每秒 30 帧。声音按时间不断测量，测一次叫一个**采样**；48000 Hz 表示每秒采样 48000 次。

原始图像和声音很占带宽，通常先按某种规则压缩，这叫**编码**，规则例如 H.264、AAC。接收端按对应规则还原，叫**解码**。压缩数据还可能很大，需要分装进网络包。

RTP 在数据前面加“标签”，说明来源、包顺序和媒体时间。标签之后装的业务数据叫**载荷（Payload）**。

```text
原始图像/声音 → 编码 → 压缩数据 → 媒体打包 → RTP 包 → UDP/TCP
                                                               ↓
画面/声音 ← 解码 ← 完整压缩数据 ← 媒体解包/重组 ← RTP 解析 ← 收包
```

“RTP 解析”只读标签并找到载荷；“媒体解包”要理解 H.264 等格式的分片规则；“解码”才产生像素或声音。三者是不同工作。

### 1.2 IP、端口、Socket、UDP、TCP

| 词语 | 通俗解释 | 本章例子 |
| --- | --- | --- |
| IP 地址 | 找到网络设备的地址 | `127.0.0.1` 表示本机自己 |
| 端口 | 设备上供程序接收数据的编号 | `5004`，本章约定的接收端口 |
| Socket | 操作系统提供的网络通信接口/句柄 | C++ 用它收发字节 |
| UDP 数据报 | 一次发送的一整份消息 | 一次 `sendto` 对应一个数据报 |
| TCP 字节流 | 有序交付的一串字节 | 发送两次，接收方未必分两次收到 |
| 协议 | 双方约定的字节格式和交互规则 | RTP 规定哪些字节表示序号 |
| 会话 | 一次有组织的媒体通信 | 一次直播活动 |
| 轨道 | 常见 RTSP 的一条媒体描述/传输对象 | 视频轨道、音频轨道 |

UDP 保留消息边界，但可能丢失、重复或乱序。TCP 保证连接内字节有序可靠交付，但重传可能造成等待，连接也可能失败；它不保留应用消息边界。

本章 UDP 实验只发到本机，无需路由器和公网设置。更完整的解释见 [[01-网络与Socket基础]]。

### 1.3 位、字节和十六进制

**前置：**C++ 整数和数组。

本章平台上，一个字节有 8 个二进制位，范围 0～255。十六进制的两个数字表示一个字节。

```text
0x80 = 二进制 10000000 = 十进制 128
0x60 = 二进制 01100000 = 十进制  96
```

`80 E0 00 01` 的十六进制转储表示 4 个字节，不是字符 `'8'`、`'0'` 等组成的文本；C++ 里应写 `{0x80, 0xE0, 0x00, 0x01}`。

`std::uint8_t`、`std::uint16_t`、`std::uint32_t` 分别是 8、16、32 位无符号整数，由 `<cstdint>` 提供。16 位范围 0～65535；32 位范围 0～4294967295。

### 1.4 位运算：从字节中取出几位

`>>` 是右移；`&` 是逐位“与”，可遮住不需要的位；`|` 是逐位“或”，可合并不同位置的位。

```cpp
// 解释片段；完整程序在 §10。
std::uint8_t b0 = 0x80;          // 10000000
unsigned version = b0 >> 6;     // 最高 2 位：2
unsigned cc = b0 & 0x0f;        // 低 4 位：0
std::uint8_t b1 = 0xe0;          // 11100000
bool marker = (b1 & 0x80) != 0;  // 最高位是否为 1：是
unsigned pt = b1 & 0x7f;        // 低 7 位：96
```

`0x0f` 是 `00001111`，`0x7f` 是 `01111111`。不要用带符号的 `char` 做这些移位。打印 `uint8_t` 时可转成 `unsigned`，避免被当成字符。

### 1.5 大端序：多个字节怎样合成数字

RTP 多字节字段采用**网络字节序 / 大端序**：高位字节在前。

```text
字节 12 34 → 数值 0x1234 → 十进制 4660
字节 00 01 5F 90 → 0x00015F90 → 十进制 90000
```

两个字节可以理解为“前一个乘 256，再加后一个”：`0x12 * 256 + 0x34 = 4660`。

不要把网络内存直接强转成 C++ 位域结构体或 `uint32_t*`：这会遇到大小端、结构体填充、对齐和对象访问规则问题。先检查长度，再逐字节合并最清楚。

**小练习与检查：**

- [ ] 我能解释编码、解包、解码的区别。
- [ ] 我知道 IP、端口、Socket 各自干什么。
- [ ] 我能算出 `0xe0 & 0x7f = 96`、`0x80 >> 6 = 2`。
- [ ] 我能把 `12 34` 读成 4660。

## 2. RTP 的定位：它负责什么

**前置：**§1。

RTP 全称 **Real-time Transport Protocol，实时传输协议**，为实时媒体提供包序号、媒体时间戳、源标识和载荷类型信息。

| 名称 | 日常比喻 | RTSP 拉流中的任务 |
| --- | --- | --- |
| RTSP | 播放遥控器 | DESCRIBE、SETUP、PLAY、TEARDOWN 等控制 |
| SDP | 媒体说明书 | 轨道、编码、时钟、参数 |
| RTP | 带标签的媒体包 | 携带媒体数据，说明顺序和媒体时间 |
| RTCP | 质量报告和校时信息 | 报告收发质量，建立同步时间关系 |
| UDP/TCP | 承载数据的运输方式 | 把应用字节送到另一端 |

普通 RTP 不自动重传、不保证按时到达、不解码视频，也不自动加密。“实时”表示面向实时媒体的设计，不是零延迟承诺；拥塞控制和恢复机制需要额外设计或协议支持。

经常看到的 `RTP/AVP` 中，**AVP（Audio/Video Profile，音视频配置约定）**是在RTP基础上进一步规定音视频使用规则的一组约定，例如部分PT的默认映射。**载荷格式规范**则进一步说明H.264、Opus等某种媒体怎样放入RTP。因此，理解一个包通常需要“RTP通用头 + 当前配置约定 + 对应媒体载荷规则”三层知识。

### 2.1 RTP over UDP

```text
链路层 → IP 头 → UDP 头 → RTP 头/可选头信息 → 媒体载荷 → 可选填充
```

普通 UDP Socket 的 `recvfrom()` 已由操作系统去掉链路层、IP 和 UDP 头，本章得到的数据从 RTP 第一个字节开始。不要再固定跳过“20 字节 IP 头 + 8 字节 UDP 头”。分析原始抓包时，才按工具展示的各层边界阅读。

### 2.2 RTP over RTSP/TCP

常见的 **TCP 交织（interleaved）** 在 RTP 外加一个短封装，让 RTSP 文本、RTP、RTCP 共用一条 TCP 连接：

```text
RTSP 文本响应 | $ + 通道号 + 长度 + RTP 包 | $ + 通道号 + 长度 + RTCP
```

`$` 不是 RTP 头，必须先拆除外层。其他 RTP over TCP 方案可能采用别的封装，例如 RFC 4571 的两字节长度，不能全部按 `$` 解析。

### 2.3 与前后章节连起来

```text
DESCRIBE → SDP：知道编码和时钟
SETUP → 协商 UDP 端口或 TCP 通道
PLAY → 开始发媒体
收包 → RTP 解析 → 重排/判断缺包 → 媒体解包 → 解码/播放
```

本章用人工数据先跳过控制协商。接入真实服务器时还要完成前面步骤，参见 [[03-RTSP协议]]、[[04-SDP会话描述]]。

**阶段 A 检查：**

- [ ] 我能区分 RTSP、SDP、RTP、RTCP 的任务。
- [ ] 我知道 UDP Socket 返回的内容从哪一层开始。
- [ ] 我知道 RTP 可以由 UDP 或有明确组帧规则的 TCP 承载。

## 3. RTP 头：每个字段逐一讲解

**前置：**字节、位运算、大端序。

### 3.1 总结构和固定头布局

```text
12 字节固定头 → CC 个 CSRC（每个4字节）→ 扩展 → 载荷 → Padding
                  可选                   可选           可选
```

固定头最少 12 字节，整个 RTP 头不一定只有 12 字节。本章使用常见 RTP/AVP 的 M 为 1 位、PT 为 7 位布局。

| 字节偏移 | 内容，从高位到低位 |
| --- | --- |
| 0 | V：2位；P：1位；X：1位；CC：4位 |
| 1 | M：1位；PT：7位 |
| 2～3 | Sequence Number：16位包序号 |
| 4～7 | Timestamp：32位媒体时间戳 |
| 8～11 | SSRC：32位同步源标识 |
| 12 起 | 按标志判断 CSRC、扩展、载荷和填充 |

“偏移 0”就是数组第一个字节 `data[0]`。协议图的排列不意味着能用 C++ 位域直接映射。

### 3.2 V、P、X、CC：第一个字节

| 字段 | 含义 | 用法 |
| --- | --- | --- |
| V：Version | 版本 | 本章要求为 2 |
| P：Padding | 尾部填充标志 | 为 1 时从最后字节读取填充长度 |
| X：Extension | 扩展标志 | 为 1 时在 CSRC 后读扩展头 |
| CC：CSRC Count | 贡献源数量，0～15 | CSRC 列表占 `4×CC` 字节 |

`0x80 = 10000000`：V=2、P=0、X=0、CC=0，没有 CSRC 和扩展，载荷从偏移 12 开始。

`0xb1 = 10110001`：V=2、P=1、X=1、CC=1，要先跳过一个 CSRC、扩展区，再处理尾部填充。

### 3.3 M、PT：第二个字节

**M（Marker，标记位）**的具体事件由媒体规则决定。H.264 over RTP 用它表示对应访问单元的最后一个包；音频可能用它标记一段发声的开始。不能一律解释成“帧结束”。

**PT（Payload Type，载荷类型号）**用于查载荷格式；不是帧类型，不表示是否关键帧。动态 PT 要查 SDP 或其他会话约定。

```text
0x60 → M=0，PT=96
0xE0 → M=1，PT=96
```

直接把 `0xe0` 当 PT 会得到错误的 224，必须先 `& 0x7f`。

### 3.4 Sequence Number：包序号

16 位，正常每发一个 RTP 数据包加 1，65535 之后回到 0。初始值应随机选择，并非固定从 0 开始。它帮助发现乱序、重复和缺口；数的是包，不是视频帧。一个画面拆成 5 个包，就通常占用 5 个连续序号。详见 §5。

### 3.5 Timestamp：媒体时间戳

32 位，表示媒体采样/呈现时间的位置，单位由 RTP 时钟决定。通常不是 Unix 时间、系统时钟或毫秒，初始值通常随机。H.264 每秒 90000 个刻度，时间戳差 3000 对应 `3000/90000` 秒。详见 §6。

### 3.6 SSRC 和 CSRC：来源标识

**SSRC（Synchronization Source，同步源）**是 32 位源标识。接收端用它区分源，为每个源分别维护序号和时间状态。

**CSRC（Contributing Source，贡献源）**是可选的贡献者列表。想象会议混音器把三个人的声音混成一路：一路有混音器的 SSRC，还可以列出三个人的 CSRC。普通摄像头经常 CC=0。

CSRC 不是时间戳，也不是视频分片数量。即使不处理混音，也要正确跳过它。

### 3.7 扩展头：长度单位是 4 字节

X=1 时，在 CSRC 后读取：

```text
16位扩展标识/参数 | 16位长度L | L×4 字节扩展内容
```

L 不包含扩展头本身的 4 字节。L=0 允许，但仍存在 4 字节扩展头。

`BE DE 00 01 10 AA 00 00` 表示标识 `0xBEDE`、内容长 4 字节、整个扩展区长 8 字节。`0xBEDE` 常见于另行定义的扩展元素格式，本章只安全跳过，不逐项解释。

未知扩展在会话规则允许时可跳过；需要其中的信息时再按对应标准解析。不能把它当成视频数据交给解码器。

### 3.8 Padding：在整个包尾部

P=1 时，最后一个字节表示要去掉的填充字节总数，包含计数字节自己：

```text
真实载荷 | 00 00 00 04  → 去掉最后 4 字节
```

计数不能为 0，不能越过头部。P=0 时最后字节属于正常数据，不能随意删掉。

本章解析器区分“字节结构可安全切分”和“具体媒体格式有效”。算出零长度载荷不意味着能送给 H.264 解包器；媒体层还要检查自身最小长度等要求。

### 3.9 载荷位置公式

```text
H = 12 + 4×CC
X=1 时：H 再加 4 + 4×扩展长度字段
P=1 时：Pad=最后一个字节；否则 Pad=0
载荷偏移 = H
载荷长度 = 整个 RTP 包长 - H - Pad
```

每步先确认相关字节存在，再计算并检查是否越界。固定头没有“RTP 总长度”字段：UDP 由数据报边界提供长度；TCP 交织由外层长度字段提供。

## 4. 手工拆包：先不用代码

**前置：**§3 的布局和大端序。

### 4.1 一个 16 字节报文

```text
偏移  00 01 02 03 04 05 06 07 08 09 0A 0B | 0C 0D 0E 0F
字节  80 E0 FF FE 00 01 5F 90 11 22 33 44 | DE AD BE EF
      └────────────12字节头───────────────┘   4字节载荷
```

| 内容 | 计算方法 | 结果 |
| --- | --- | --- |
| 首字节 | `0x80 >> 6`，其他标志为 0 | V=2、P=0、X=0、CC=0 |
| 第二字节 | `0xe0 >> 7`、`0xe0 & 0x7f` | M=1、PT=96 |
| 序号 | 大端读 `FF FE` | 65534 |
| 时间戳 | 大端读 `00 01 5F 90` | 90000 |
| SSRC | 大端读 `11 22 33 44` | `0x11223344` = 287454020 |
| 头长 | `12+0` | 12字节 |
| 载荷 | 从偏移12取 `16-12` 字节 | `DE AD BE EF` |

载荷只是教学字节，不是可解码的 H.264。PT=96 的实际含义仍要查约定。

### 4.2 带全部可选区域的报文

```text
B1 60 00 01 00 00 00 00 11 22 33 44  # 固定头：P=1 X=1 CC=1
01 02 03 04                          # 一个CSRC：4字节
BE DE 00 01                          # 扩展头：内容长1×4
10 AA 00 00                          # 扩展内容：4字节
AA BB                                # 真实载荷：2字节
00 00 00 04                          # 尾部填充：4字节
```

`#` 后是注释，不是网络字节。总长 `12+4+4+4+2+4=30`；H=24，Pad=4，载荷长 `30-24-4=2`。固定跳过12字节会把 CSRC 和扩展当成媒体。

**练习：**把扩展长度 `00 01` 改成 `00 02`，不添加字节，结果会怎样？字段声明更多字节不会创造字节，还要结合总长和 Padding 校验。答案见 §16。

**阶段 B 检查：**

- [ ] 我能指出全部固定头字段，手算以上两个包。
- [ ] 我能解释为什么载荷不总从偏移12开始。
- [ ] 我知道扩展长度单位和 Padding 的位置。
- [ ] 我知道整个 RTP 包长度由谁提供。

## 5. 包序号：顺序、缺口、乱序、重复和回绕

**前置：**知道 Sequence Number 是 16 位的包编号。

### 5.1 先分清五种现象

| 到达顺序 | 当时能判断什么 | 不应急着下什么结论 |
| --- | --- | --- |
| `100、101、102` | 连续到达 | 不能据此知道画面帧数 |
| `100、102` | 暂时没有看到101 | 不能立刻说101永久丢失 |
| `100、102、101` | 101晚到，出现乱序 | 不能把101当成新一轮编号 |
| `100、101、101、102` | 101重复出现 | 不能把重复包交给解包器两次 |
| `65534、65535、0、1` | 正常回绕 | 不能把0当成异常倒退 |

序号是发送顺序的线索，到达顺序是网络交付的结果，两者可能不同。

### 5.2 为什么普通大小比较会出错

`0 > 65535` 在普通整数中是假，但在连续 RTP 序列中，0 可以紧跟 65535。序号在一个大小为 65536 的环上前进。

下面的教学函数求环上的最近差值，返回 -32768～32767：

```cpp
// 假设比较的两个包距离小于半圈，且属于同一连续发送源。
int sequence_delta(std::uint16_t now, std::uint16_t reference) {
    std::uint32_t u =
        (static_cast<std::uint32_t>(now) + 65536u - reference) & 0xffffu;
    return u < 32768u ? static_cast<int>(u)
                      : static_cast<int>(u) - 65536;
}
```

```text
sequence_delta(0, 65535)   =  1：向前一个包
sequence_delta(65535, 0)   = -1：比参考值旧一个包
sequence_delta(103, 100)   =  3：向前三个位置
```

“半圈”是 32768 个包，不是秒。相差恰好半圈时无法仅凭这两个数判断方向；长时间断流、大跳变或源重启也不能只用这个公式。正式接收器需校验源的连续性，参考 RFC 3550 附录 A.1。

### 5.3 参考值要用“已见最高序号”，不能总用上一次到达值

到达 `100、102、101、103` 时，收到101后，已见的最新位置仍然是102。如果把参考值改回101，再收103时就可能多算一次缺口。

最小教学观察规则：

```text
第一次：记住 highest
d = sequence_delta(新序号, highest)
d > 0：推进 highest；若 d>1，只报告出现候选缺口
d = 0：与 highest 同号，疑似重复
d < 0：旧序号，可能是乱序、迟到或旧包重复
```

仅记最高值分不清“第一次晚到的101”和“第二次收到的旧101”。要识别所有重复，还要保存窗口内已收序号集合。本章 §10 只做观察，不实现完整重排器，也不输出伪精确丢包率。

### 5.4 什么时候才能把缺口判成丢包

可先把缺失序号放入“等待名单”。在合理的重排窗口和截止时间内等它：

```text
收到100 → 可交付100，下一包期待101
收到102 → 暂存102，等待101
收到101 → 先交付101，再交付已存的102
```

若到播放截止时间101还没来，就要决定跳过它，并通知媒体重组器当前数据不完整。实时播放不能为一个缺包无限等待。

可用“最多缓存多少个包”和“最多等多少毫秒”一起限制。局域网实验可先尝试几十毫秒，但这只是实验参数，不是 RTP 标准固定值。窗口过小容易把乱序当丢包；过大则增加内存和播放延迟。

### 5.5 统计为什么比“序号差减一”复杂

在同一连续源的一段已确定区间内，教学上可以用：

```text
期望包数 = 最高扩展序号 - 起始扩展序号 + 1
唯一已收包数 = 区间内不同序号的数量
区间缺失数 = 期望包数 - 唯一已收包数
```

**扩展序号**就是把16位环扩展成更大的连续编号，记录已经绕过多少圈。重复包不增加“唯一已收包数”；迟到包可能让区间缺失数减少。

这与“过了播放截止时间、已无法用于播放的包数”不是同一个指标。RTCP 报告还有自己的统计定义，RFC 3550 的接收计数甚至可能因重复包导致累计丢失为负。不要把上述教学公式直接当 RTCP 实现。

**练习：**`10、12、11、12、14` 中，最终见过几个不同包？相对于10～14，哪个仍缺？若13在截止时间后才到，网络接收统计和播放是否可能得出不同结果？答案见 §16。

## 6. 时间戳：媒体的时间刻度

**前置：**知道帧、采样和 Timestamp。

### 6.1 时钟频率不是帧率

RTP 时钟像刻度尺。`90000` 表示一秒分成90000个时间刻度，不是每秒90000帧，也不是每秒发90000个包。

| 格式例子 | RTP 时钟 | 一个简单例子 |
| --- | --- | --- |
| H.264 视频 | 90000 Hz | 恒定30 fps时相邻画面常差3000 |
| PCMU / PCMA | 8000 Hz | 20 ms对应160个刻度 |
| 常见 AAC 配置 | 通常与对应采样率一致，查SDP | 48000 Hz下1024采样对应约21.333 ms |
| Opus | 固定48000 Hz的RTP时钟 | 20 ms对应960个刻度 |

不同编码的规则可能不同。即便实际声音处理用其他采样率，也不能擅自改变 Opus 的 RTP 时钟。AAC 每包也不总是恰好1024采样：要看编码配置和打包方式。

### 6.2 最常用的时间差公式

```text
媒体时间差（秒） = 时间戳差 / RTP时钟频率
媒体时间差（毫秒） = 时间戳差 × 1000 / RTP时钟频率
```

假设同一源、相邻媒体时间向前、未跨越无法判断的长间隔：

```cpp
std::uint32_t before = 90000;
std::uint32_t after  = 93000;
std::uint32_t ticks = after - before; // 无符号减法，按2^32回绕
double milliseconds = 1000.0 * ticks / 90000.0; // 约33.3333
```

`1000.0` 中的小数点让表达式用浮点计算，避免整数除法截断。不要直接把时间戳差当毫秒。

### 6.3 同一画面可以对应多个相同时间戳的包

以简单 H.264 非交织发送为例：

| 包序号 | 时间戳 | M | 通俗含义 |
| --- | --- | --- | --- |
| 100 | 90000 | 0 | 这一画面的部分数据 |
| 101 | 90000 | 0 | 同一画面的后续部分 |
| 102 | 90000 | 1 | 该访问单元的最后一个包 |
| 103 | 93000 | 0 | 下一画面的部分数据 |
| 104 | 93000 | 1 | 下一访问单元的最后一个包 |

因此：包数不是帧数；同一时间戳不表示重复包；时间戳每包加1也通常不正确。真正的媒体分片边界还要看载荷格式。

这只是帮助理解的常见情形。复杂 H.264 打包、隔行内容等要按载荷规范处理，不能写成“所有编码只要时间戳相同就直接拼字节”。

### 6.4 32 位时间戳也会回绕

最大值4294967295之后回到0。90 kHz 时钟绕一圈约13.26小时，48 kHz约24.86小时。不是到了这个时长流就会坏掉，正确实现应处理回绕。

```text
before = 0xFFFFFF00
after  = 0x00000AB8
按32位无符号减法得到差值3000 → 在90kHz下约33.333ms
```

此算法适用于已知向前且真实间隔小于一圈的比较。若拿一个更早的乱序媒体时间减去较新的时间，会得到很大的无符号数；不能把它当成真实巨长间隔。

### 6.5 到达时间不等于媒体时间

摄像头给画面标的时间是媒体时间；你的电脑收到包的时刻是到达时间。网络排队、处理和重传会改变到达间隔，不会自动改变原来的媒体时间。

假设两张图像媒体时间相隔33.3ms，收到它们却只相隔5ms，并不说明视频突然变成200fps；可能是网络把它们攒到一起才交付。

媒体时间戳还可能因 B 帧等编码顺序因素不按包序号单调增长。B帧可先理解为“解码时可能依赖前后画面的压缩帧”；第一轮只需知道发送顺序、解码顺序和显示顺序可能不同。不要用“时间戳倒退”单一条件判坏包。

### 6.6 为什么不能直接用音频时间戳减视频时间戳

视频可能90000Hz、音频48000Hz，单位不同；即使单位相同，随机起点也通常不同。需要 RTCP Sender Report（发送者报告）等把各源的 RTP 时间映射到共同时间轴，再结合源关联信息同步。

因此，仅凭一个RTP时间戳通常无法算绝对拍摄时间或端到端延迟。映射和同步详见 [[06-RTCP协议]]。

**阶段 C 检查：**

- [ ] 我能解释序号跳变为何只能先报告候选缺口。
- [ ] 我能识别65535→0，知道环比较的适用条件。
- [ ] 我能算出3000/90000秒、160/8000秒。
- [ ] 我能区分发送顺序、到达顺序和媒体时间。
- [ ] 我知道音视频时间戳不能直接相减。

## 7. PT 和 SSRC：先找到格式，再维护来源状态

**前置：**§3、§6；不要求记住全部编码。

### 7.1 静态 PT 与动态 PT

PT占7位，数值0～127。在常见 RTP/AVP 中，部分值已有静态映射，如 PT=0 是PCMU/8000，PT=8 是PCMA/8000；96～127用于动态分配。不能说“0～95全部有固定编码”，其中还有未分配或保留值。

**动态**的意思是本次会话另外约定数字和格式的关系。H.264 常见 PT=96，但也可以是其他协商值；96也可以被另一条轨道用于别的编码。

本章实验把96约定为“教学字节”，不把它发送给真实视频解码器。

### 7.2 从 SDP 建立映射表

下面是说明映射的SDP片段，不是完整可用的摄像头配置：

```sdp
m=video 0 RTP/AVP 96
a=rtpmap:96 H264/90000
a=fmtp:96 packetization-mode=1
a=control:trackID=0
m=audio 0 RTP/AVP 97
a=rtpmap:97 MPEG4-GENERIC/48000/2
a=control:trackID=1
```

| 轨道 | PT | 编码 | 时钟 | 接收后交给谁 |
| --- | --- | --- | --- | --- |
| trackID=0 | 96 | H.264 | 90000 | H.264 RTP解包器 |
| trackID=1 | 97 | MPEG4-GENERIC，常用于AAC | 48000 | 相应音频解包器 |

`48000/2` 的2表示声道数；`fmtp` 是额外的格式参数。真正的AAC配置通常还需要具体fmtp参数。RTSP单播的 `m=` 端口0也不表示实验要绑定0端口去等待摄像头；真实端口由SETUP过程协商。详见 [[04-SDP会话描述]]。

接收器应按轨道/会话保存PT表，不要在全程序只写一张永远不变的 `96→H264` 表。未知PT应记录并丢弃或交给上层协商处理，而不是盲猜编码。RTP本身允许来源在合适约定下改变PT，不能把PT当永久不变的流身份。

### 7.3 SSRC 不是这些编号

| 编号 | 用途 | 是否等于SSRC |
| --- | --- | --- |
| RTSP `Session:` | RTSP控制会话标识 | 否 |
| SDP `trackID=0` | 轨道控制URL的一部分 | 否 |
| TCP交织channel=0 | 该TCP连接中的数据通道号 | 否 |
| PT=96 | 本会话的载荷格式号 | 否 |
| SSRC=`0x11223344` | RTP会话中的同步源标识 | 是本字段自身 |

SSRC不是IP地址，不应拿来作身份认证，也不是全球唯一号码。不同RTP会话中可能出现相同SSRC；同一个摄像头的音视频通常使用不同源状态。

### 7.4 多路流与重启怎样处理

状态通常按“RTP会话/轨道上下文 + SSRC”隔离，网络端点和通道等由协商上下文辅助分流。不要把两路流的序号交替丢进同一个缺包检测器。

出现新SSRC时，建立新源状态；不要直接拿新包序号减旧源序号。摄像头重启、冲突处理等可能改变SSRC；有些设备也会在同SSRC下重置序号或时间戳，因此还要识别不连续和重新验证源。仅凭一次大跳变不能准确断言“丢了几万个包”。

## 8. Marker、包、NAL、帧：不要混为一谈

**前置：**§1、§3、§6。

**NAL（网络抽象层单元）**是H.264中的一块有类型的压缩数据；**AU（Access Unit，访问单元）**可先理解为对应一次图像解码的一组相关编码数据。复杂情况留到下一章，第一轮先分清层次：

```text
一个画面的压缩表示：可能有多个NAL
一个NAL：可能放在一个RTP包里，也可能拆成多个包
一个RTP包：有时也能装多个小NAL
```

H.264常见打包规则中，M=1是对应AU最后一个包的提示。它不说明前面所有包都收齐了，不说明该包是关键帧，也不能替代序号检查和载荷重组。规范也要求解码端不能仅依赖Marker判断完整边界。

**常见错误：**收到一个RTP包就调用一次“解码一帧”；只要M=1就认为本帧完整；直接把所有载荷拼成文件。接下来还需要识别单NAL、聚合包和分片包，见 [[07-H264与RTP封装]]。

**阶段 D 检查：**

- [ ] 我能从SDP查PT、编码、时钟，而不把96写死。
- [ ] 我能区分Session、轨道、channel、SSRC。
- [ ] 我能解释“一包不一定一帧”“M=1不保证完整”。

## 9. Windows 实验准备：先能编译，再研究网络

**前置：**能创建文本文件、打开终端；其余按下面做。

### 9.1 准备工作目录

后面所有源文件都放在同一个目录，例如 `C:\rtp-lab`。打开PowerShell，执行：

```powershell
New-Item -ItemType Directory -Force C:\rtp-lab
Set-Location C:\rtp-lab
```

用编辑器创建文件，编码选择UTF-8，扩展名必须是 `.cpp` 或 `.hpp`，不要保存成 `.cpp.txt`。`.hpp` 是供其他源文件包含的公共代码文件，不单独运行。

### 9.2 选择一种编译器即可

**方式一：Visual Studio 2022 / Build Tools，MSVC。** 安装时勾选“使用C++的桌面开发”和Windows SDK，然后打开“Developer PowerShell for VS 2022”或“x64 Native Tools Command Prompt”。前者可直接使用本章PowerShell命令；后者是cmd终端，切目录用 `cd /d C:\rtp-lab`，运行用 `rtp_demo.exe`。

在开发者终端执行 `cl`，能看到编译器版本即可。普通PowerShell不一定认识 `cl`，不要仅凭这一点判断安装失败。

**方式二：MinGW-w64 的 g++。** 安装带Windows头文件和运行库的MinGW-w64工具链，将其bin目录加入PATH；重新打开PowerShell，执行：

```powershell
g++ --version
```

已有可用编译器就继续使用。本章给出两种命令，选择一套执行，不需要两套都安装。

### 9.3 第一个程序不收网络包

§10先使用写在数组中的数据，不用Socket。这样如果字段不对，先查解析逻辑；不必同时排查防火墙、端口和摄像头。

后面共4个文件：

```text
C:\rtp-lab\
  rtp_common.hpp       共同使用的结构解析、造包、打印函数
  rtp_demo.cpp         实验1：离线解析与序号观察
  rtp_udp.cpp          实验2：Windows UDP接收/发送
  interleaved_demo.cpp 实验3：模拟TCP拆分与合并
```

程序中的英文错误信息方便查API和保持终端输出稳定；正文逐项解释含义。

## 10. 实验一：离线解析 RTP，先把字节读对

**目标：**不连接网络，先复现 §4 两个报文的字段结果，再观察序号回绕、乱序和候选缺口。

### 10.1 公共文件 `rtp_common.hpp`

将下面完整代码保存为 `C:\rtp-lab\rtp_common.hpp`。先大致看函数职责，运行成功后再逐行读。

```cpp
#pragma once
#include <cstddef>
#include <cstdint>
#include <iomanip>
#include <iostream>
#include <string>
#include <vector>

struct RtpInfo {
    unsigned version = 0;
    bool padding = false;
    bool extension = false;
    unsigned csrcCount = 0;
    bool marker = false;
    unsigned payloadType = 0;
    std::uint16_t sequence = 0;
    std::uint32_t timestamp = 0;
    std::uint32_t ssrc = 0;
    std::uint16_t extensionProfile = 0;
    std::size_t extensionBytes = 0; // 包含扩展头自身
    std::size_t headerBytes = 0;
    std::size_t paddingBytes = 0;
    std::size_t payloadOffset = 0;
    std::size_t payloadBytes = 0;
};

// 只负责读取；调用者必须事先保证这些字节存在。
inline std::uint16_t read_be16(const std::uint8_t* p) {
    return static_cast<std::uint16_t>(
        (static_cast<std::uint16_t>(p[0]) << 8) | p[1]);
}

inline std::uint32_t read_be32(const std::uint8_t* p) {
    return (static_cast<std::uint32_t>(p[0]) << 24) |
           (static_cast<std::uint32_t>(p[1]) << 16) |
           (static_cast<std::uint32_t>(p[2]) << 8) |
           static_cast<std::uint32_t>(p[3]);
}

inline bool parse_rtp(const std::uint8_t* data, std::size_t size,
                      RtpInfo& out, std::string& error) {
    out = RtpInfo{};
    error.clear();
    if (data == nullptr || size < 12) {
        error = "short fixed header";
        return false;
    }
    out.version = data[0] >> 6;
    if (out.version != 2) {
        error = "unsupported RTP version";
        return false;
    }
    out.padding = (data[0] & 0x20) != 0;
    out.extension = (data[0] & 0x10) != 0;
    out.csrcCount = data[0] & 0x0f;
    out.marker = (data[1] & 0x80) != 0;
    out.payloadType = data[1] & 0x7f;
    out.sequence = read_be16(data + 2);
    out.timestamp = read_be32(data + 4);
    out.ssrc = read_be32(data + 8);

    std::size_t pos = 12 + 4 * out.csrcCount;
    if (pos > size) {
        error = "short CSRC list";
        return false;
    }
    if (out.extension) {
        if (size - pos < 4) {
            error = "short extension header";
            return false;
        }
        out.extensionProfile = read_be16(data + pos);
        std::size_t words = read_be16(data + pos + 2);
        // 先比较容量，再乘4。避免先做不受约束的长度运算。
        if (words > (size - pos - 4) / 4) {
            error = "short extension data";
            return false;
        }
        out.extensionBytes = 4 + words * 4;
        pos += out.extensionBytes;
    }
    if (out.padding) {
        std::size_t count = data[size - 1];
        if (count == 0 || count > size - pos) {
            error = "invalid padding";
            return false;
        }
        out.paddingBytes = count;
    }
    out.headerBytes = pos;
    out.payloadOffset = pos;
    out.payloadBytes = size - pos - out.paddingBytes;
    return true;
}

inline int sequence_delta(std::uint16_t now, std::uint16_t reference) {
    std::uint32_t u =
        (static_cast<std::uint32_t>(now) + 65536u - reference) & 0xffffu;
    return u < 32768u ? static_cast<int>(u)
                      : static_cast<int>(u) - 65536;
}

// 固定SSRC、PT和载荷仅供实验，不是正式媒体发送器。
inline std::vector<std::uint8_t> make_demo_packet(
        std::uint16_t seq, std::uint32_t ts, bool marker) {
    return {
        0x80, static_cast<std::uint8_t>((marker ? 0x80 : 0) | 96),
        static_cast<std::uint8_t>(seq >> 8),
        static_cast<std::uint8_t>(seq),
        static_cast<std::uint8_t>(ts >> 24),
        static_cast<std::uint8_t>(ts >> 16),
        static_cast<std::uint8_t>(ts >> 8),
        static_cast<std::uint8_t>(ts),
        0x11, 0x22, 0x33, 0x44,
        0xde, 0xad, 0xbe, 0xef
    };
}

inline void print_rtp(const RtpInfo& r) {
    std::cout << "PT=" << r.payloadType << " M=" << r.marker
              << " seq=" << r.sequence << " ts=" << r.timestamp
              << " SSRC=0x" << std::hex << std::uppercase
              << std::setw(8) << std::setfill('0') << r.ssrc
              << std::dec << std::setfill(' ')
              << " header=" << r.headerBytes
              << " padding=" << r.paddingBytes
              << " payload=" << r.payloadBytes << '\n';
}
```

**为什么这样写：**

- `const uint8_t* data` 是一段只读字节的起始地址，`size` 是有效字节数；解析器不能靠指针猜长度。
- `RtpInfo& out`、`string& error` 中的 `&` 表示引用，让函数把结果写回调用者的变量。
- `inline` 让这些函数可以定义在公共头文件中，被多个 `.cpp` 包含而不产生重复定义问题。
- `payloadOffset` 与 `payloadBytes` 指出载荷范围，解析器不复制载荷、不做H.264重组。
- 成功才返回 `true`；失败时不要使用部分填好的 `out` 字段继续解码。
- 用偏移值避免返回悬空指针。稍后取载荷时，原字节数组仍必须存在，且不能被改造成别的内容。

这个函数只检查普通明文RTP的结构。它不负责确认UDP来源、已协商PT、完整媒体内容、SSRC连续性，也不能直接验证加密的SRTP认证标签。

### 10.2 主程序 `rtp_demo.cpp`

保存为 `C:\rtp-lab\rtp_demo.cpp`：

```cpp
#include "rtp_common.hpp"

int main() {
    auto first = make_demo_packet(65534, 90000, true);
    std::vector<std::uint8_t> optional = {
        0xb1, 0x60, 0x00, 0x01, 0x00, 0x00, 0x00, 0x00,
        0x11, 0x22, 0x33, 0x44,
        0x01, 0x02, 0x03, 0x04,
        0xbe, 0xde, 0x00, 0x01, 0x10, 0xaa, 0x00, 0x00,
        0xaa, 0xbb, 0x00, 0x00, 0x00, 0x04
    };

    for (const auto& packet : {first, optional}) {
        RtpInfo r;
        std::string error;
        if (!parse_rtp(packet.data(), packet.size(), r, error)) {
            std::cerr << error << '\n';
            return 1;
        }
        print_rtp(r);
        std::cout << "payload hex:";
        for (std::size_t i = 0; i < r.payloadBytes; ++i) {
            unsigned value = packet[r.payloadOffset + i];
            std::cout << ' ' << std::hex << std::uppercase
                      << std::setw(2) << std::setfill('0') << value;
        }
        std::cout << std::dec << std::setfill(' ') << '\n';
    }

    // 改坏一个包：只保留8字节，固定头都不完整。
    auto bad = first;
    bad.resize(8);
    RtpInfo r;
    std::string error;
    if (parse_rtp(bad.data(), bad.size(), r, error)) {
        std::cerr << "expected rejection\n";
        return 1;
    }
    std::cout << "bad packet: " << error << '\n';

    // 只是最高序号观察器，不计算最终丢包数、不执行重排。
    std::vector<std::uint16_t> arrivals = {65534, 0, 65535, 0, 2, 3};
    std::uint16_t highest = arrivals.front();
    std::cout << "first=" << highest << '\n';
    for (std::size_t i = 1; i < arrivals.size(); ++i) {
        auto seq = arrivals[i];
        int d = sequence_delta(seq, highest);
        std::cout << "seq=" << seq << " delta=" << d;
        if (d > 0) {
            std::cout << (d == 1 ? " forward" : " candidate gap");
            highest = seq;
        } else {
            std::cout << (d == 0 ? " same highest" : " old or reordered");
        }
        std::cout << '\n';
    }

    std::uint32_t before = 0xffffff00u;
    std::uint32_t after = 0x00000ab8u;
    std::uint32_t ticks = after - before;
    std::cout << "timestamp ticks=" << ticks
              << " ms=" << 1000.0 * ticks / 90000.0 << '\n';
}
```

`std::vector` 是可变长数组；`.data()`取首地址，`.size()`取元素数；`auto`让编译器根据右侧推断类型；`const auto&`表示只读引用，循环时不修改原元素。`front()`取第一个元素，本例数组非空。

### 10.3 编译和运行

在 `C:\rtp-lab` 中，**MSVC**：

```powershell
cl /nologo /std:c++17 /EHsc /W4 /utf-8 rtp_demo.cpp /Fe:rtp_demo.exe
.\rtp_demo.exe
```

或者 **MinGW-w64**：

```powershell
g++ -std=c++17 -Wall -Wextra -pedantic -finput-charset=UTF-8 rtp_demo.cpp -o rtp_demo.exe
.\rtp_demo.exe
```

编译器把源代码生成 `.exe`；之后运行 `.exe`。PowerShell 当前目录不是默认命令搜索路径，所以使用 `.\` 前缀。

### 10.4 对照预期输出

```text
PT=96 M=1 seq=65534 ts=90000 SSRC=0x11223344 header=12 padding=0 payload=4
payload hex: DE AD BE EF
PT=96 M=0 seq=1 ts=0 SSRC=0x11223344 header=24 padding=4 payload=2
payload hex: AA BB
bad packet: short fixed header
first=65534
seq=0 delta=2 candidate gap
seq=65535 delta=-1 old or reordered
seq=0 delta=0 same highest
seq=2 delta=2 candidate gap
seq=3 delta=1 forward
timestamp ticks=3000 ms=33.3333
```

浮点显示末位可能因编译器设置略有差异。`candidate gap`是候选缺口，`old or reordered`是旧包/乱序包，`same highest`只表示与当前最高序号相同。程序没有维护完整的已收集合，不能据此识别所有重复。

### 10.5 每次只改一处的错误实验

改动后重新编译再运行，完成后还原：

| 改动 | 应观察到什么 | 原因 |
| --- | --- | --- |
| 把生成函数第一个字节从0x80改成0x40 | `unsupported RTP version`，退出1 | 版本变成1 |
| 把 `first` 的首字节设为0x8f | `short CSRC list` | 声明15个CSRC，实际字节不够 |
| 在单独测试包中设P=1，并把尾字节设为0 | `invalid padding` | 填充计数非法 |
| 把可选包扩展长度从1改为0xffff | `short extension data` | 声明的扩展超出报文 |
| 把M从1改成0 | PT仍为96 | 标记位与PT分开 |

若改动的是 `make_demo_packet()`，§11、§12也会受影响；继续其他实验前恢复原代码。正式程序可丢弃一个坏包后继续收下一个，本实验为了清晰展示错误，直接返回失败。

**阶段 E 检查：**

- [ ] 我在自己的Windows终端编译运行成功。
- [ ] 我能把输出与 §4 的字节逐项对上。
- [ ] 我能解释每一次长度校验在保护什么。
- [ ] 我做过至少一个错误输入实验，并能解释拒绝原因。

## 11. 实验二：Windows C++ UDP 本机收发

**目标：**让操作系统真正交付UDP数据报，再复用相同RTP解析器。先启动接收端，再启动发送端。

### 11.1 只需要认识这些 Winsock 函数

Windows网络接口称为 **Winsock**。下面这些是API，也就是程序调用系统功能的接口：

| 调用 | 通俗解释 | 本例何时使用 |
| --- | --- | --- |
| `WSAStartup` | 初始化Winsock | 程序开始时 |
| `socket` | 创建通信句柄 | 创建UDP Socket |
| `bind` | 指定本机接收地址和端口 | 接收端监听127.0.0.1:5004 |
| `recvfrom` | 收一个数据报，并可得知来源 | 接收循环 |
| `sendto` | 把一个数据报发往指定地址 | 发送端 |
| `setsockopt` | 设置Socket选项 | 本例设置接收超时 |
| `WSAGetLastError` | 取得最近一次Winsock错误码 | 调用失败之后立即读取 |
| `closesocket` / `WSACleanup` | 关闭句柄 / 结束Winsock使用 | 程序退出前 |

`htons`把16位主机整数转为网络字节序，`htonl`处理32位。本例把5004放进网络地址结构时必须转换。网络结构里的地址和手工解析RTP头是不同层的工作。

### 11.2 完整文件 `rtp_udp.cpp`

保存为 `C:\rtp-lab\rtp_udp.cpp`，保持 `rtp_common.hpp` 在同一目录：

```cpp
#define WIN32_LEAN_AND_MEAN
#define NOMINMAX
#include <winsock2.h>
#include <ws2tcpip.h>
#include <chrono>
#include <thread>
#include "rtp_common.hpp"

#ifdef _MSC_VER
#pragma comment(lib, "Ws2_32.lib")
#endif

int socket_error(const char* operation) {
    std::cerr << operation << " failed, WSA=" << WSAGetLastError() << '\n';
    return 1;
}

int receive_packets(SOCKET s) {
    // DWORD是Windows定义的32位无符号类型；此选项单位是毫秒。
    DWORD timeout = 5000;
    if (setsockopt(s, SOL_SOCKET, SO_RCVTIMEO,
                   reinterpret_cast<const char*>(&timeout),
                   sizeof(timeout)) == SOCKET_ERROR) {
        return socket_error("setsockopt");
    }
    sockaddr_in local{};
    local.sin_family = AF_INET;
    local.sin_port = htons(5004);
    local.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
    if (bind(s, reinterpret_cast<const sockaddr*>(&local),
             sizeof(local)) == SOCKET_ERROR) {
        return socket_error("bind");
    }
    std::cout << "Listening on 127.0.0.1:5004; start sender now.\n"
              << std::flush;

    // 大于普通IPv4 UDP最大载荷，用来避免本例人为截短数据报。
    std::vector<std::uint8_t> buffer(65536);
    int accepted = 0;
    while (accepted < 6) {
        sockaddr_in source{};
        int sourceBytes = sizeof(source); // 每次调用前重新初始化
        int n = recvfrom(s, reinterpret_cast<char*>(buffer.data()),
                         static_cast<int>(buffer.size()), 0,
                         reinterpret_cast<sockaddr*>(&source),
                         &sourceBytes);
        if (n == SOCKET_ERROR) {
            int e = WSAGetLastError();
            if (e == WSAETIMEDOUT) {
                std::cerr << "receive timeout: no datagram for 5 seconds\n";
            } else {
                std::cerr << "recvfrom failed, WSA=" << e << '\n';
            }
            return 1;
        }
        RtpInfo r;
        std::string error;
        if (!parse_rtp(buffer.data(), static_cast<std::size_t>(n), r, error)) {
            std::cout << "discard: " << error << '\n';
            continue;
        }
        // 本机实验预先约定这两个值；真实客户端按SDP和源状态处理。
        if (r.payloadType != 96 || r.ssrc != 0x11223344u) {
            std::cout << "discard: unexpected demo PT or SSRC\n";
            continue;
        }
        print_rtp(r);
        ++accepted;
    }
    return 0;
}

int send_packets(SOCKET s) {
    sockaddr_in target{};
    target.sin_family = AF_INET;
    target.sin_port = htons(5004);
    target.sin_addr.s_addr = htonl(INADDR_LOOPBACK);

    // 人工构造乱序、重复和缺口，不依赖网络刚好出现故障。
    std::vector<std::uint16_t> sequences = {100, 102, 101, 102, 104, 105};
    for (auto seq : sequences) {
        std::uint32_t ts = seq <= 102 ? 90000u : 93000u;
        bool marker = seq == 102 || seq == 105;
        auto packet = make_demo_packet(seq, ts, marker);
        int n = sendto(s, reinterpret_cast<const char*>(packet.data()),
                       static_cast<int>(packet.size()), 0,
                       reinterpret_cast<const sockaddr*>(&target),
                       sizeof(target));
        if (n == SOCKET_ERROR) {
            return socket_error("sendto");
        }
        if (n != static_cast<int>(packet.size())) {
            std::cerr << "unexpected sent length\n";
            return 1;
        }
        std::cout << "sent seq=" << seq << '\n';
        std::this_thread::sleep_for(std::chrono::milliseconds(20));
    }
    return 0;
}

int main(int argc, char* argv[]) {
    if (argc != 2 ||
        (std::string(argv[1]) != "recv" && std::string(argv[1]) != "send")) {
        std::cerr << "usage: rtp_udp.exe recv | send\n";
        return 1;
    }
    WSADATA data{};
    int startup = WSAStartup(MAKEWORD(2, 2), &data);
    if (startup != 0) {
        // WSAStartup的错误码直接来自返回值。
        std::cerr << "WSAStartup failed, code=" << startup << '\n';
        return 1;
    }
    SOCKET s = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (s == INVALID_SOCKET) {
        int result = socket_error("socket");
        WSACleanup();
        return result;
    }
    int result = std::string(argv[1]) == "recv"
               ? receive_packets(s) : send_packets(s);
    closesocket(s);
    WSACleanup();
    return result;
}
```

### 11.3 怎样读这段代码

先读 `main`：检查参数→初始化→建Socket→选择收/发→释放资源。再读发送端，它只生成6个包并逐个 `sendto`。最后读接收端：绑定地址→接收→按实际返回长度解析→打印。

`reinterpret_cast<char*>` 出现在Winsock调用处，是为了适配API要求的字节缓冲类型；它不把网络数据强转成RTP结构体。`sockaddr_in`是系统定义的IPv4地址结构，API要求传成通用 `sockaddr*`，所以也有对应转换。

`recvfrom` 返回多少有效字节，就只解析多少。缓冲容量65536不代表收到65536字节。返回0在UDP中可以表示零长度数据报，不能照搬TCP“连接关闭”的含义；本例会因不足12字节拒绝它。

接收端没手工设置发送方端口：发送端使用系统分配的临时源端口。目标端口是5004。真实RTSP程序要按SETUP协商结果和来源策略接收，不能把本例固定值直接当标准。

### 11.4 编译与两个终端运行

**MSVC**，在开发者终端、工作目录中：

```powershell
cl /nologo /std:c++17 /EHsc /W4 /utf-8 rtp_udp.cpp /Fe:rtp_udp.exe /link Ws2_32.lib
```

或者 **MinGW-w64**：

```powershell
g++ -std=c++17 -Wall -Wextra -pedantic -finput-charset=UTF-8 rtp_udp.cpp -o rtp_udp.exe -lws2_32
```

终端A执行：

```powershell
Set-Location C:\rtp-lab
.\rtp_udp.exe recv
```

显示 `Listening...` 后，5秒内在终端B执行：

```powershell
Set-Location C:\rtp-lab
.\rtp_udp.exe send
```

如果来不及切换，接收端会超时退出，重新运行即可；也可在代码中把5000改为15000后重新编译。超时是每次接收等待的时限，不是整个程序统一5秒。

### 11.5 预期现象和分析

接收端通常依次打印：

```text
seq=100 ts=90000 M=0
seq=102 ts=90000 M=1
seq=101 ts=90000 M=0
seq=102 ts=90000 M=1
seq=104 ts=93000 M=0
seq=105 ts=93000 M=1
```

上面省略了程序每行其余字段，方便对照；实际顺序由到达决定，UDP不提供顺序保证。

这组结果中：101是人为晚发，用来模拟乱序到达；102重复发送；103没发送，用来模拟缺失；104和105的媒体时间比前一组晚约33.33ms。20ms发送间隔只为便于观察，不代表媒体时间步长。它是故障教学，不是正式的媒体采样和发送调度实现。

接收端按“通过解析的包数”计到6，包括重复包，不等于收到6个不同包，更不等于6帧画面。程序没有重排，也不输出最终丢包统计。

### 11.6 可以马上做的三个小练习

1. 改成连续序号 `100、101、102、103、104、105`，对比输出。
2. 保持序号不变，只把第二组时间戳改成96000，算出媒体间隔。
3. 只开接收端，观察超时。理解“没有包”和“有包但解析失败”的区别。

**阶段 F 检查：**

- [ ] 我能在两个终端完成本机收发。
- [ ] 我能解释为什么程序不需要摄像头或RTSP服务器。
- [ ] 我知道Winsock初始化、绑定、接收和释放的顺序。
- [ ] 我能指出本例中的乱序、重复和缺失。
- [ ] 我不会把6次接收当成6个不同包或6帧。

## 12. 实验三：TCP 交织，先恢复包边界

**前置：**UDP与TCP的区别，§10的解析器。

### 12.1 交织封装具体长什么样

在常见RTSP 1.0 TCP交织中，每个二进制块：

```text
1字节 '$'(0x24) | 1字节channel | 2字节大端长度N | N字节数据
```

长度只计算后面的数据，不包含4字节交织头。一个块携带一个上层协议数据单元，例如一个完整RTP包。长度字段不是ASCII数字文本。

SETUP响应若协商 `interleaved=0-1`，通常0用于该轨道RTP、1用于RTCP；另一轨道可能2和3。**以实际协商为准**，不能永远把偶数通道当视频、奇数当音频。

### 12.2 一次 recv 不等于一个包

假设发送端一次写入20字节：4字节交织头+16字节RTP。接收端可能先读到2字节，再读到18字节；也可能一次读到连续多个块。

正确动作是追加到缓冲区，至少有4字节才读N；至少有 `4+N` 字节才取出一个完整块。消费一个后继续检查剩余字节。不能通过“找下一个 `$`”确定包尾，因为媒体载荷内部也可以有 `0x24`。

### 12.3 完整文件 `interleaved_demo.cpp`

这是**离线模拟TCP分块**：主动把数据分成多次喂给解析器，不建立TCP连接。这样每次都能复现半包和多包。它只接收纯交织数据；真实RTSP混合文本解析见 §12.6。

保存为 `C:\rtp-lab\interleaved_demo.cpp`：

```cpp
#include <algorithm>
#include <stdexcept>
#include "rtp_common.hpp"

class InterleavedReader {
public:
    void feed(const std::uint8_t* data, std::size_t size) {
        // 教学缓冲上限；正式客户端应按自己的处理策略配置。
        constexpr std::size_t maxBuffer = 256 * 1024;
        if (size > maxBuffer - buffer_.size()) {
            throw std::runtime_error("interleaved buffer limit");
        }
        if (size != 0) {
            buffer_.insert(buffer_.end(), data, data + size);
        }
        while (buffer_.size() >= 4) {
            if (buffer_[0] != 0x24) {
                throw std::runtime_error("expected dollar block");
            }
            unsigned channel = buffer_[1];
            std::size_t length = read_be16(buffer_.data() + 2);
            std::size_t total = 4 + length;
            if (buffer_.size() < total) {
                return; // 不够一个完整块，保留等待下次feed
            }
            std::cout << "channel=" << channel << " length=" << length << '\n';
            if (channel == 0) { // 本实验约定0为RTP，不代表所有会话
                RtpInfo r;
                std::string error;
                if (parse_rtp(buffer_.data() + 4, length, r, error)) {
                    print_rtp(r);
                } else {
                    std::cout << "discard RTP: " << error << '\n';
                }
            } else if (channel == 1) {
                std::cout << "RTCP block: dispatch to RTCP parser\n";
            } else {
                std::cout << "unknown demo channel: skip\n";
            }
            // 所有本块数据处理完成后再擦除；不能继续保存内部指针。
            buffer_.erase(buffer_.begin(), buffer_.begin() + total);
        }
    }

    std::size_t pending_bytes() const { return buffer_.size(); }

private:
    std::vector<std::uint8_t> buffer_;
};

std::vector<std::uint8_t> wrap(std::uint8_t channel,
                              const std::vector<std::uint8_t>& packet) {
    if (packet.size() > 65535) {
        throw std::runtime_error("packet too large for interleaved length");
    }
    std::vector<std::uint8_t> result = {
        0x24, channel,
        static_cast<std::uint8_t>(packet.size() >> 8),
        static_cast<std::uint8_t>(packet.size())
    };
    result.insert(result.end(), packet.begin(), packet.end());
    return result;
}

int main() {
    try {
        auto first = make_demo_packet(100, 90000, false);
        first.back() = 0x24; // 载荷内也有'$'，不能用它找包尾
        auto stream = wrap(0, first);
        auto second = wrap(0, make_demo_packet(101, 90000, true));
        stream.insert(stream.end(), second.begin(), second.end());

        InterleavedReader reader;
        std::size_t pos = 0;
        // 总共40字节：先给2字节半头，再3字节，最后35字节。
        for (std::size_t chunk : {2u, 3u, 35u}) {
            auto count = std::min(chunk, stream.size() - pos);
            std::cout << "feed=" << count << '\n';
            reader.feed(stream.data() + pos, count);
            pos += count;
        }
        std::cout << "pending=" << reader.pending_bytes() << '\n';
        if (pos != stream.size() || reader.pending_bytes() != 0) {
            std::cerr << "incomplete input\n";
            return 1;
        }
    } catch (const std::exception& e) {
        std::cerr << e.what() << '\n';
        return 1;
    }
}
```

### 12.4 编译、运行和结果

**MSVC**：

```powershell
cl /nologo /std:c++17 /EHsc /W4 /utf-8 interleaved_demo.cpp /Fe:interleaved_demo.exe
.\interleaved_demo.exe
```

或者 **MinGW-w64**：

```powershell
g++ -std=c++17 -Wall -Wextra -pedantic -finput-charset=UTF-8 interleaved_demo.cpp -o interleaved_demo.exe
.\interleaved_demo.exe
```

预期输出：

```text
feed=2
feed=3
feed=35
channel=0 length=16
PT=96 M=0 seq=100 ts=90000 SSRC=0x11223344 header=12 padding=0 payload=4
channel=0 length=16
PT=96 M=1 seq=101 ts=90000 SSRC=0x11223344 header=12 padding=0 payload=4
pending=0
```

前两次没有完整包，所以不输出RTP；第三次补齐后一次取出两个包。载荷中的 `$` 不影响结果。

### 12.5 C++ 新语法只看这里

`class`把缓冲区和操作放在一起；`private`表示外部不能直接改缓冲；`feed`意思是“给解析器喂入新字节”。`.insert()`追加，`.erase()`消费已经处理的数据。

`throw`抛出错误，`try/catch`在主函数接住，防止程序无说明退出。这里用异常报告流格式错误；普通RTP包结构错误则打印并跳过当前完整块。

这段代码用vector头部删除来保持直观，大量数据时会有移动开销。以后可以换成读游标或环形缓冲，不必第一轮就优化。

### 12.6 真正 RTSP/TCP 客户端还缺什么

实际接收循环中，只有 `recv()` 返回正数时才把这些有效字节交给组帧器；返回0表示TCP对端有序关闭，错误则按系统错误码处理。

同一TCP连接还有RTSP文本。完整状态机在**消息边界**判断：

```text
当前没有正在读取的消息
  ├─ 首字节为'$'：等4字节头 → 按长度读二进制块 → 按通道分发
  └─ 否则：等完整RTSP头（CRLF CRLF）
             → 按Content-Length读完整消息体
             → 消费整个消息，再回到边界判断
```

不要在SDP消息体内部扫描 `$` 并切走；不要把RTCP块丢进RTP解析器。还需消息长度上限、等待超时、通道映射、断连恢复和慢速输入处理。本实验只演示其中的二进制组帧，完整控制见 [[03-RTSP协议]]、[[10-C++客户端架构]]。

**阶段 G 前半检查：**

- [ ] 我知道4字节交织头不属于RTP。
- [ ] 我能解释2+3+35字节为什么最终得到两个包。
- [ ] 我知道不能用载荷里的 `$` 找边界。
- [ ] 我知道通道号来自SETUP协商。

## 13. 从教学程序到真正的 RTP 接收器

**前置：**三个实验；本节主要画流程，第一轮不要求全部实现。

### 13.1 正确的处理顺序

```text
UDP数据报 / 从TCP中提取的完整二进制块
  ↓ 按已协商端口、通道区分RTP和RTCP
完整RTP字节
  ↓ 校验版本、头长、CSRC、扩展、Padding
RTP头信息 + 有效载荷范围
  ↓ 查轨道PT映射、识别SSRC并维护源状态
源内序号检查、去重、有限窗口重排
  ↓ 发现缺失/过期时通知媒体层
H.264/AAC等对应的媒体解包与重组
  ↓ 获得完整压缩单元、保留媒体时间
解码器 → 播放缓冲 → 按媒体时钟显示/播放
```

每层只解决自己的问题。下游报告“解码失败”时，先检查上游是否丢字节、把扩展当载荷、把两路流混在一起。不要一开始就更换解码库。

### 13.2 接收缓冲、重排缓冲、播放缓冲不是一个东西

| 缓冲 | 暂存什么 | 解决什么问题 | 过大有什么代价 |
| --- | --- | --- | --- |
| 操作系统Socket接收缓冲 | 已到达但应用尚未读取的数据 | 应用短暂忙碌时容纳数据 | 内存与积压；不保证不丢包 |
| TCP组帧缓冲 | 尚未组成完整消息的字节 | 半头、半包、多包 | 内存积压，需要长度/时间上限 |
| RTP重排缓冲 | 等待前面缺包的完整RTP包 | 乱序和短暂迟到 | 等待更久，延迟增加 |
| 媒体重组缓冲 | 一个NAL/AU尚不完整的片段 | 形成完整压缩数据 | 大帧或坏长度可撑大内存 |
| 播放缓冲 / 抖动缓冲 | 等待播放时刻的媒体 | 平滑到达间隔变化 | 更稳定但播放更晚 |

学习时它们可以在同一线程或少数容器里实现，但概念上要区分。只把Socket缓冲调大，不能修复错误的分片重组或音画不同步。

### 13.3 抖动：不是每个包都按相同间隔到

**抖动（Jitter）**指包间传输时间变化。网络到达间隔忽长忽短，播放端通常要稍等一小段时间，让数据更平稳地输出；等得越久，增加的延迟越大。

认识RFC 3550的一个估算式即可，第一轮不用背：

```text
R(i)：第i个包的到达时刻，先换算成与RTP相同的时钟单位
S(i)：该包RTP时间戳，比较时处理回绕
D = [R(i)-R(i-1)] - [S(i)-S(i-1)]
J = J + (|D|-J)/16
```

`|D|`是绝对值，J是平滑后的估计。若J用RTP刻度表示，换成毫秒是 `J×1000/clock_rate`。这是包到达抖动估计，不是RTT、丢包率或端到端延迟。

**RTT（往返时间）**是一去一回的耗时；**端到端延迟**是采集到最终播放的耗时。三个指标不要混用。实际计时宜用 `std::chrono::steady_clock` 这样的单调时钟，即不会因用户修改系统时间而突然跳变的计时工具。

### 13.4 RTP 与 RTCP 应怎样分流

常见UDP方式分别使用RTP、RTCP端口；常见TCP交织使用不同通道。传统RTP/AVP常用偶数RTP端口、下一个奇数RTCP端口，但真正客户端按协商结果处理，不能写死。

RTP和RTCP的首字节版本都可能为2，仅检查 `data[0] >> 6` 不足以分流。若协商了同端口复用，则要按相应复用规则分类；不能直接复用本章“这个端口只装RTP”的假设。RTCP见 [[06-RTCP协议]]。

### 13.5 丢包以后怎样恢复

对于某些音频，可以用丢包隐藏技术近似补一小段声音；对于依赖前后数据的视频，一个缺包可能破坏当前NAL、画面或后续参考关系。不能为了“凑够长度”随意填0。

典型做法是标记当前重组不完整、丢弃无法安全使用的单元、等待后续可恢复数据；在协商支持的情况下请求关键帧或重传。反馈消息也需要对方支持，不能只发一个请求就假设一定恢复。下一步学习 [[07-H264与RTP封装]] 和 [[06-RTCP协议]]。

**关键帧**可先理解为有助于重新建立解码参考的图像；H.264中还要进一步认识IDR等具体规则。Marker不是关键帧标志。

### 13.6 MTU：为什么不能把大画面塞进一个 UDP 包

**MTU（最大传输单元）**可先理解为一条链路对单个网络层包大小的限制。网络中最小的有效限制可能影响整条路径。

若某路径MTU为1500，且IPv4头20字节、UDP头8字节、RTP基本头12字节，没有其他选项，则留给RTP载荷的是 `1500-20-8-12=1460` 字节。IPv6、头扩展、隧道等会改变计算。工程上有时选择约1200字节的载荷以留余量，但1200不是所有网络都保证安全的固定值。

不要依赖IP分片运输超大UDP包；其中一个IP片丢失会使整个数据报不能交付。视频应按H.264等载荷标准分片；RTP头自身没有通用的“第几片/共几片”字段。

§11使用65536字节**接收数组**只是为了容量足够，不代表一次要发这么大的RTP包，也不代表绕过MTU。

### 13.7 长度和输入边界的最低要求

把所有网络输入都当成“可能不完整、格式错误或属于别的会话”，逐层检查：

- 固定头至少12字节、版本为2；CC对应的字节必须存在。
- X=1时扩展头与声明内容必须存在；长度单位用对。
- P=1时填充计数非0，并且不侵入头部。
- PT必须能按当前轨道解释；媒体解包器另查自身最小长度。
- SSRC和序号按源状态处理，不跨源拼包；限制可同时维护的源数量。
- 设置包数、字节数和等待时限；媒体重组失败时清理旧状态。
- TCP长度合理但数据迟迟不来时，也需要超时和连接处理。
- 若缓冲要复用或vector会扩容，不把内部指针交给异步任务长期保存；需要时复制有效数据或明确所有权。

解析通过只意味着结构能读，不意味着身份可信、载荷能解码或所有分片收齐。SRTP要由相应实现完成解密和认证处理后，再按正确边界解释数据，不能把认证标签当普通Padding裁掉。

**阶段 G 检查：**

- [ ] 我能画出从完整网络包到画面的处理链。
- [ ] 我能说出至少三种缓冲各自存什么。
- [ ] 我能区分抖动、往返时间、端到端延迟。
- [ ] 我知道MTU、接收数组容量、视频帧大小不是同一个概念。
- [ ] 我能说明坏包、缺片、源变化时要清理哪些状态。

## 14. 可选抓包实验：看见程序收到的字节

**前置：**完成本机UDP实验。抓包工具只是观察工具，不影响前面程序能否运行。

### 14.1 Wireshark 中观察本机 UDP

Wireshark可把网络数据逐层展开。Windows抓包通常使用Npcap；抓本机回环流量时选名称包含Loopback/回环的对应接口，名称随安装版本不同。若没有该接口，先检查抓包驱动的回环支持。

1. 选择回环接口开始抓包。
2. 在**显示过滤器**输入 `udp.port == 5004`，这只过滤显示，不决定程序接收。
3. 按 §11 先开接收端，再开发送端。
4. 找到6个UDP数据报，查看每个UDP载荷是否为16字节。
5. 若没有自动识别RTP，对对应UDP流使用“Decode As / 解码为”选择RTP。
6. 展开RTP字段，对照序号、时间戳、PT、Marker和SSRC。

若界面还把载荷标为未知动态格式，不要紧，本实验没有真实SDP。只验证通用RTP头，不要求解码出视频。

常用显示过滤器，在已经识别为RTP后使用：

```text
udp.port == 5004
rtp
rtp.ssrc == 0x11223344
rtp.seq == 102
```

RTP流分析面板中的缺包、乱序和抖动也应结合统计规则解释，不能把短暂缺口和重复包一概当最终损失。

### 14.2 接入真实 RTSP 时按顺序检查

先完成 [[30-runbooks/02-抓包分析一次RTSP拉流]]，再选择 [[30-runbooks/04-验证UDP传输]] 或 [[30-runbooks/03-验证TCP交织传输]]。

```text
DESCRIBE成功吗？ → SDP中PT和时钟是什么？
SETUP实际选择UDP还是TCP？ → 端口/通道是多少？
PLAY成功吗？ → 对应端口/通道真的有媒体吗？
RTP头能安全解析吗？ → 序号、源、媒体时间是否合理？
载荷重组完整吗？ → 最后再检查解码和播放。
```

本章UDP程序固定绑定127.0.0.1，只用于本机教学。接局域网摄像头时，需要绑定合适的本机接口或按设计使用 `INADDR_ANY`（所有本机接口）、完成SETUP并校验来源。不要只把目标IP改成摄像头地址就期待它主动推流。

TCP方案通常更容易通过某些网络环境，但可能因可靠重传产生排队等待；UDP可能更及时，但要求应用处理缺包和乱序。应根据网络与应用目标选择，不存在所有场景都最好的固定答案。

## 15. 常见错误与排查顺序

**前置：**至少运行过离线实验；不必一次记住整张表，出错时查。

| 现象 | 常见原因 | 第一项检查 / 修正 |
| --- | --- | --- |
| `cl` 无法识别 | 未打开VS开发者终端或未装C++工具链 | 打开对应Developer终端，执行 `cl` |
| `g++` 无法识别 | MinGW-w64未安装或PATH未生效 | 新开终端执行 `g++ --version` |
| 找不到 `rtp_common.hpp` | 文件位置或扩展名错 | `.cpp`和`.hpp`放同目录，查看是否隐藏了`.txt` |
| 中文注释报编码错误 | 源文件编码不一致 | 保存UTF-8，MSVC加 `/utf-8` |
| 链接失败，提示WSAStartup/recvfrom等未定义 | 漏链接Winsock库 | MSVC用Ws2_32.lib；g++末尾加 `-lws2_32` |
| PowerShell找不到刚生成的exe | 漏当前目录前缀 | 用 `.\rtp_demo.exe` |
| bind失败，WSA=10048 | 5004已被占用 | 关闭自己上一次的接收程序，或同时改收发双方端口 |
| WSA=10013 | 访问被拒绝、策略限制或端口受保留等 | 先换一个可用端口并同时改两端，核对具体系统配置 |
| receive timeout，WSA通常为10060 | 5秒内没有下一份数据报 | 先收后发，两端地址端口一致，必要时调长实验超时 |
| WSA=10040 | 数据报大于接收缓冲，已被截断 | 不解析截短数据；增大合理接收容量或约束发送包大小 |
| 发送端显示成功，接收端无数据 | 发往错误地址/端口、接收未运行或策略阻断 | `sendto`成功只说明交给本机网络栈，不是对方已收 |
| PT显示224 | 把整个第二字节当PT | 用 `data[1] & 0x7f`，Marker单独取 |
| 序号/时间戳数值离谱 | 读反字节序、起点不是RTP、解析了RTCP | 检查分层起点，用大端函数读 |
| 总是 version 错 | 把交织头、RTSP文本或别的协议当RTP | 先恢复外层消息边界并分流 |
| 载荷开头不像预期 | 固定跳12字节，漏了CSRC/扩展 | 用实际头长和padding计算范围 |
| 65535→0被判成大倒退 | 直接用普通大小比较 | 用有限窗口下的环比较，维护扩展序号 |
| 乱序后越算“丢包”越多 | 每次用上次到达序号、立刻累计缺口 | 最高序号与已收集合分开，等截止时间再判 |
| 音视频不同步 | 时钟、起点不同，或忽略同步映射 | 查SDP时钟和RTCP映射，不直接减音视频时间戳 |
| 收到M=1却解码失败 | 前面缺片，或未按载荷格式解包 | 查完整性、分片与参数集，不把Marker当完整性保证 |
| TCP有时能播、有时解析失败 | 假设一次recv就是一个包 | 累积字节，按RTSP消息/交织长度消费 |
| 程序收到包但没画面 | 本实验载荷是DE AD BE EF，或还没接媒体解包/解码 | 按下一章建立有效媒体输入和解码链 |
| 长时间运行后内存/延迟上升 | 缺包无限等待或过期数据不清理 | 给每级缓冲设置数量、字节和时间上限 |

查看哪个进程占用本机UDP端口，可在PowerShell执行：

```powershell
Get-NetUDPEndpoint -LocalPort 5004 -ErrorAction SilentlyContinue
```

查看 `OwningProcess` 后，用 `Get-Process -Id 具体PID` 对照程序。优先关闭自己打开的旧终端程序，避免结束不认识的进程。UDP没有TCP那种LISTEN握手状态，不能只用查看TCP连接的方式判断UDP是否绑定。

**建议固定排查次序：**编译→启动→地址/端口/通道→有没有数据→包边界→RTP字段→媒体重组→解码→播放时间。每一步留下“预期/实际”记录，只改一个因素。

## 16. 练习题与参考答案

先独立完成，再展开答案。做错时回对应章节，改完后再算一次；不以背术语为目标。

### 16.1 基础题：用自己的话解释

1. **协议职责（§2）：**RTSP控制播放，为什么还需要RTP？SDP里面装的是视频像素吗？
2. **字节与位（§1、§3）：**首字节0xb2，第二字节0xe1，分别解析出V/P/X/CC/M/PT。
3. **大端序（§1）：**字节 `01 02` 和 `00 00 BB 80` 分别是多少？
4. **源标识（§7）：**两条轨道都出现PT=96，能否用PT把它们分成两路？RTSP Session是否等于SSRC？

<details>
<summary>参考答案：基础题</summary>

1. RTSP发送控制命令，不代替媒体传输；RTP装媒体数据并提供顺序和时间信息。SDP是格式说明书，不是像素。
2. 0xb2=`10110010`：V=2、P=1、X=1、CC=2；0xe1=`11100001`：M=1、PT=97。
3. `0x0102=258`；`0x0000bb80=48000`。
4. 不能。PT表示当前轨道上下文内的载荷格式，不是全局流身份。结合协商的端口/通道、轨道和SSRC维护源状态；Session不等于SSRC。

</details>

### 16.2 报文题：一定要写出计算过程

5. **固定头（§4）：**`80 60 12 34 00 00 BB 80 AA BB CC DD 11 22`。求序号、时间戳、SSRC、M、PT、载荷长度。
6. **可变头（§3）：**一个包总长44字节，CC=2、X=1、扩展长度=2、P=1、尾字节=4。求头长、载荷偏移和载荷长度。
7. **扩展边界（§4.2）：**把30字节可选报文的扩展长度1改成2，不加字节，本章解析器会怎样？
8. **Padding（§3.8）：**P=0时末字节为4，应该删4字节吗？P=1时末字节为0呢？

<details>
<summary>参考答案：报文题</summary>

5. 序号4660，时间戳48000，SSRC=`0xaabbccdd`，M=0，PT=96；无CSRC、扩展和填充，载荷为 `11 22`，长2字节。
6. 头长 `12+2×4+4+2×4=32`，偏移32，载荷长 `44-32-4=8`。
7. 声明扩展内容8字节后，H变成28；包中只剩2字节，尾字节却要求删4字节，报 `invalid padding`。只检查扩展内容是否存在还不够，要继续检查Padding。
8. P=0不删；P=1且计数0为非法Padding，应拒绝。

</details>

### 16.3 顺序与时间题

9. **乱序（§5）：**到达 `10、12、11、12、14`，不同包有几个？10～14中还缺谁？若13晚于播放截止时间才到，会怎样？
10. **回绕（§5）：**`65534、65535、0、1` 是倒退吗？`sequence_delta(0,65535)`是多少？为什么不能拿这个函数解释任意长的断流？
11. **媒体时间（§6）：**H.264时间戳从100000到103600，对应多少毫秒？若每帧都是这个间隔，帧率多少？
12. **音频（§6）：**48kHz时钟的时间戳增加960，对应多少毫秒？G.711/8000的20ms对应多少刻度？
13. **包与帧（§6、§8）：**3个包的序号连续、时间戳相同，是重复包还是同帧数据？最后一个M=1是否保证完整？
14. **同步（§6）：**视频时间戳90000，音频48000，能否说它们正好同步，因为都除以时钟等于1秒？

<details>
<summary>参考答案：顺序与时间题</summary>

9. 4个不同包：10、11、12、14；缺13。迟到的13可能修正网络接收统计，但已经来不及用于当前播放，故播放损失与最终接收缺失可能不同。
10. 是正常回绕；差值1。函数只在同一连续源、距离小于半圈时有方向意义；长断流或重启需重新验证。
11. `3600/90000=0.04秒=40ms`；若恒定每帧40ms，就是25fps。
12. `960/48000=0.02秒=20ms`；8000×0.02=160刻度。
13. 不能仅凭相同时间戳判重复，常见情况是同一画面的多个包；要结合序号和载荷格式。M=1也不保证中间包全到。
14. 不能。两路起点通常不同，单位换算并未消除随机起点；需要共同时间映射和源关联。

</details>

### 16.4 工程题：独立改代码

15. **UDP（§11）：**把实验端口改为5504。只改发送端会出现什么？把解析长度从n改成 `buffer.size()` 为什么错？
16. **TCP（§12）：**把40字节输入改成每次喂1字节；再试一次喂40字节。两个RTP包的解析结果应否相同？
17. **区分协议（§13）：**收到首字节0x80，能否仅凭它认定是RTP？真实RTSP/TCP流中收到 `RTSP/1.0 200 OK` 应该交给谁？
18. **MTU与内存（§13）：**接收数组扩大到1MB，是否就能安全发送1MB的UDP数据报？为什么？
19. **健壮性（§10）：**分别构造不足12字节、CC声明过多、只有半个扩展头、扩展内容越界、非法Padding。让程序报告不同错误，且不崩溃。
20. **综合（§7～§15）：**给你一段实际SDP和抓包，写出轨道、PT、编码、时钟、SSRC、承载方式、端口/通道；再计算两个合理相邻媒体时间的间隔，指出数据应该交给哪个解包器。

<details>
<summary>参考答案与验收方法：工程题</summary>

15. 收发两端的目标/绑定端口都要改；只改发送端，原接收端通常超时。buffer.size()是容量，会把无效或上次遗留字节纳入解析；只用实际n。
16. 结果应相同：得到序号100和101，pending=0；输入分块不应改变上层协议意义。每次1字节时可用循环对 `stream.data()+i` 调 `feed(...,1)`，一次40字节时用 `feed(stream.data(),stream.size())`。
17. 不能，RTCP也有相同版本位，别的字节也可能碰巧相同。先按协商上下文/复用规则分流；RTSP文本交给控制消息解析器，再按Content-Length取消息体。
18. 不能，数组容量不改变UDP协议长度限制和路径MTU。用媒体载荷标准分片，并限制每包大小。
19. 对应 `short fixed header`、`short CSRC list`、`short extension header`、`short extension data`、`invalid padding`。错误时不继续访问越界字段，也不交给解码器。
20. 没有固定数值答案。必须给每个结论附上SDP行、SETUP响应或具体报文。若缺少映射，明确写“仅凭当前包无法确定”，不能猜PT=96就是H.264。

</details>

## 17. 分阶段检查清单与结业小项目

这是学习者自测，不等于作者代码验证记录。能用自己的话解释并留下运行结果，再勾选。

### 17.1 第一关：离开代码也能讲明白

- [ ] 用一句话分别介绍RTSP、SDP、RTP、RTCP。
- [ ] 解释字节、大端序、载荷、时钟、SSRC。
- [ ] 手工拆一个不带扩展的报文，字段都正确。
- [ ] 手工拆一个带CSRC、扩展和Padding的报文。
- [ ] 解释PT为什么需要结合轨道和SDP。

### 17.2 第二关：能处理顺序和时间

- [ ] 不把临时序号缺口立即计成最终丢包。
- [ ] 能解释乱序、重复、回绕、大跳变与重启的区别。
- [ ] 用90kHz和48kHz分别算对一次媒体时间差。
- [ ] 知道同一时间戳可以有多个RTP包。
- [ ] 知道Marker不是关键帧标志，也不是完整性保证。
- [ ] 知道音画同步还需要共同时间轴。

### 17.3 第三关：能动手运行并排错

- [ ] 从空目录保存4个文件，能自行编译3个可执行程序。
- [ ] 离线输出与 §10.4 相同，能解释每个值。
- [ ] 做过畸形长度实验，理解错误信息。
- [ ] 本机UDP收发成功，能指出人工乱序、重复和缺口。
- [ ] 能解释超时、端口占用和链接库错误的排查方式。
- [ ] TCP每字节输入与一次完整输入得到相同包结果。
- [ ] 能解释教学代码尚未实现的功能及其所在章节。

### 17.4 结业小项目：写一个“RTP 观察报告”

先用本章人工数据完成，再对真实流尝试。不要一开始就写播放器。

报告包含：

| 栏目 | 要填写什么 |
| --- | --- |
| 输入来源 | 人工包/UDP/抓包；从哪一层开始 |
| 会话约定 | 轨道、承载、端口或通道、PT表、时钟 |
| 至少6包的观察表 | 到达顺序、seq、timestamp、SSRC、M、头长、载荷长 |
| 顺序结论 | 哪些是乱序、重复、暂缺；等待策略是什么 |
| 时间结论 | 至少一个可解释的媒体时间差与计算式 |
| 边界实验 | 至少三种错误输入和程序响应 |
| 后续接口 | 有效载荷交给谁；缺包或源变化通知谁 |
| 证据与不确定性 | 编译命令、输出；哪些不能仅凭当前数据确定 |

通过标准：字段可复核，计算有单位，结论不超过证据。例如“尚未收到103”比“网络永久丢失103”更符合短时观察。

完成后建议依次读 [[06-RTCP协议]]、[[07-H264与RTP封装]]、[[08-FFmpeg与解码]]。前者补齐报告和同步，中间解决载荷重组，最后把有效压缩数据变成画面。

## 18. 编写依据、验证记录与适用边界

### 18.1 规范依据

本章按下面规范核对通用头布局、时间与序号含义、长度单位和交织边界；它们是查具体规则的依据，不要求第一遍通读RFC。**RFC**可先理解为公开的互联网技术规范文档。

- RFC 3550 §5.1、§5.3.1：RTP固定头、CSRC、扩展和Padding；§6.4.1：接收统计与抖动；附录A.1：序号有效性和源校验。
- RFC 3551：常见RTP/AVP的静态/动态PT、音视频时钟和端口约定。
- RFC 2326 §10.12：本章RTSP 1.0的 `$` 交织格式。RTSP 2.0另见RFC 7826，不能把两个版本的全部流程混用。
- RFC 6184 §5.1：H.264的90kHz时钟、Marker含义和使用边界。
- 特定音频时钟还需结合载荷规范：Opus见RFC 7587；MPEG4-GENERIC见RFC 3640。

### 18.2 作者实际验证与读者操作分开记录

验证日期：2026-10-20。以下按完成后的实际运行结果记录；代码从本章Markdown代码块直接提取，避免文档和测试副本不一致。

| 项目 | 实际结果 |
| --- | --- |
| 编译环境 | Windows，MinGW-w64 g++ 15.2.0，C++17；3个程序使用 `-Wall -Wextra -pedantic` 编译通过，无编译警告 |
| 离线解析 | 两个报文的字段、头长、Padding、载荷内容均与 §4、§10.4 一致；回绕差3000刻度，约33.3333ms |
| 本机UDP | 真实启动接收与发送进程，收到100、102、101、102、104、105；字段与人工构造一致 |
| UDP错误路径 | 额外发送2字节畸形数据报，接收端丢弃后继续；无发送端时5秒超时；错误命令行参数返回失败 |
| TCP交织 | 2+3+35字节输入恢复两个16字节RTP包，载荷包含 `$` 仍正确，最后pending=0 |
| 结构与组帧边界 | 215项断言通过，包含空/截短头、错误版本、0～15个CSRC、扩展头/内容截短、零长度扩展、非法Padding、PT/M分离、序号/时间戳回绕 |
| TCP输入变化 | 覆盖40字节流的全部41种单次切分位置和逐字节输入；残留半包、纯文本误入、交织长度和缓冲上限按预期处理 |

**提供操作步骤但未在本轮实测：**MSVC编译、Wireshark图形界面抓包、真实摄像头/MediaMTX接入、真实RTSP混合TCP文本解析、H.264解码和长时间播放。这些不能由教学包收发成功来替代验证。

### 18.3 这些示例的范围

示例具备普通RTP头结构解析和最小传输观察能力，适合一步步理解字节。它们尚未实现完整RTSP连接、RTCP报告、所有重复识别、正式重排窗口、H.264/AAC重组、SRTP、完整同步、解码和画面显示。

UDP示例固定实验地址、PT、SSRC和6次有效接收；TCP示例只处理纯交织输入和固定通道约定。实际客户端必须按SDP/SETUP配置、限制资源并处理断流。即使实验全部通过，也应继续按相关章节集成和验证真实流。

## 19. 术语速查与继续阅读

| 术语 | 一句话记忆 |
| --- | --- |
| RTP | 带包顺序、媒体时间和源标识的媒体传输格式 |
| RTCP | 质量反馈和同步等控制信息 |
| Payload | 去掉RTP头和Padding后有效的媒体载荷 |
| PT | 在当前媒体上下文中查载荷格式的数字 |
| Sequence Number | 16位包序号，会回绕 |
| Timestamp | 32位媒体时间刻度，不直接等于毫秒 |
| Clock Rate | 每秒多少个RTP时间刻度 |
| SSRC / CSRC | 同步源 / 混合媒体的贡献源 |
| Marker | 含义由所用载荷/配置规则规定的标记位 |
| Interleaved | 在RTSP TCP连接中按通道封装二进制块 |
| Jitter | 包间传输时间的变化 |
| Depacketize | 按媒体规则拆除载荷封装并重组成压缩单元 |
| MTU | 链路/路径能承载的网络层包大小限制 |
| SRTP | 给RTP体系增加加密和认证等能力的协议 |

规范链接，先查相关小节即可：

- [RFC 3550：RTP](https://www.rfc-editor.org/rfc/rfc3550.html)
- [RFC 3551：RTP/AVP](https://www.rfc-editor.org/rfc/rfc3551.html)
- [RFC 2326：RTSP 1.0](https://www.rfc-editor.org/rfc/rfc2326.html)
- [RFC 7826：RTSP 2.0](https://www.rfc-editor.org/rfc/rfc7826.html)
- [RFC 6184：H.264 over RTP](https://www.rfc-editor.org/rfc/rfc6184.html)
- [RFC 7587：Opus over RTP](https://www.rfc-editor.org/rfc/rfc7587.html)
- [RFC 3640：MPEG-4通用RTP载荷](https://www.rfc-editor.org/rfc/rfc3640.html)
- [RFC 8285：RTP头扩展元素格式](https://www.rfc-editor.org/rfc/rfc8285.html)
- [RFC 4571：另一种面向连接的RTP组帧](https://www.rfc-editor.org/rfc/rfc4571.html)

知识库继续阅读：[[00-RTSP学习总纲]]、[[01-网络与Socket基础]]、[[02-音视频基础]]、[[04-SDP会话描述]]、[[06-RTCP协议]]、[[07-H264与RTP封装]]、[[08-FFmpeg与解码]]、[[11-调试与性能优化]]。
