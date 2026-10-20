---
type: topic
status: active
created: 2026-10-16
updated: 2026-10-20
tags:
  - rtcp
  - rtp
  - beginner
  - windows
  - cplusplus
aliases:
  - RTCP 入门
  - RTP 控制协议
---

# RTCP 协议：从零读懂质量报告与音画同步

> 适用对象：会一些 C++ 变量、循环、函数和数组语法，第一次学习音视频传输的读者。本章会解释专业名词，不要求先会写播放器。
>
> 学习产出：读懂 SR、RR 和 SDES；手算丢包、抖动、RTT；解释音画同步；在 Windows 编译 C++17 程序，完成离线解析和本机 UDP 双向报告实验。
>
> 阅读顺序：先看 §0 的路线，再逐阶段学习。三个实验不需要摄像头、MediaMTX、FFmpeg 或第三方 C++ 库。示例使用人工数据，作者实测范围见 §19；检查框留给你完成后自行勾选。

## 0. 先规划：这一章需要学什么、按什么顺序学

### 0.1 最终要能回答的问题

1. RTP 已经能传视频，为什么还要 RTCP？它与 RTSP 有什么关系？
2. 谁发 SR，谁发 RR？只看视频的客户端要不要发 RTCP？
3. 一次收到的 UDP 数据为什么可能包含多个 RTCP 小包？
4. 怎么从十六进制字节读出长度、SSRC 和统计数据？
5. “累计缺几包”和“最近一段的丢包率”有什么区别？
6. 抖动为什么不直接等于毫秒？它为什么不是延迟？
7. 怎样用 LSR、DLSR 算往返时间？为什么不能直接除二当单程延迟？
8. 音频和视频时间戳不同，怎样放到共同时间轴？
9. NACK、PLI、FIR 分别请求什么？发了请求为什么不一定解决问题？
10. 程序怎样校验坏包、保存源状态和安排发送？

### 0.2 八阶段学习路线

建议每阶段45～90分钟，实验可以拆成多次；以“能解释并做出来”为准。

| 阶段 | 阅读顺序 | 学习范围 | 最低前置 | 动手产出 / 通过标准 |
| --- | --- | --- | --- | --- |
| A：搭地基 | §1～§2 | 字节、时钟、Socket、协议职责 | 一些 C++ 语法 | 画出媒体和报告的两个方向 |
| B：认报文 | §3～§4 | 公共头、长度、复合包、包类型 | A | 手算长度，区分 PT、RC、FMT |
| C：读发送报告 | §5～§6 | SR、NTP/RTP 映射、CNAME、同步 | B | 计算音视频对应的共同时间 |
| D：读接收报告 | §7～§10 | RR、丢包、回绕、抖动、RTT、码率 | C、四则运算 | 算出各统计值并写对单位 |
| E：理解策略 | §11～§12 | 反馈、报告周期与带宽规则 | B～D | 说明何时反馈及协商条件 |
| F：离线编程 | §13～§15 | Windows 环境、解析生成、统计 | A～E、函数 | 编译前两个实验，修改坏包观察错误 |
| G：实际收发 | §16～§17 | UDP 双向报告、抓包、集成 | F | 两个终端完成 SR→RR，解释 RTT |
| H：独立验收 | §18～§20 | 排错、练习、答案、总检查 | 前面各阶段 | 独立完成 RTCP 观察报告 |

### 0.3 必学、继续实现和选修

**第一轮必学：**RTCP不装普通视频帧；SR/RR/SDES结构；复合包按长度遍历；大端读取和边界检查；两种丢包指标；抖动单位；LSR/DLSR的1/65536秒；NTP/RTP同时刻映射；SSRC与CNAME；Windows编译和UDP收发。

**理解接口、后续实现：**真实RTP源验证与序号展开、每个SSRC的统计状态、RTCP调度、多轨道同步播放、成员与SSRC冲突管理。结合 [[05-RTP协议]]、[[10-C++客户端架构]]、[[08-FFmpeg与解码]] 逐步实现。

**进阶选修：**AVPF早期反馈、RTX重传、FEC冗余恢复、XR扩展报告、SRTCP加密认证、RTP/RTCP同端口复用、Reduced-Size RTCP（精简RTCP）、拥塞控制、NTP年代和长时间回绕。第一遍知道用途与约定条件即可，文末有入口。

### 0.4 每次的学习方法

先用自己的话解释，再用纸笔算数，最后运行代码。每次只改一个变量，例如先改丢包数，再改时钟频率。记录“输入、计算、输出、疑问”四项。

先把“能解释一个报告”作为成果。解码与显示继续看 [[07-H264与RTP封装]]、[[08-FFmpeg与解码]]；RTCP实验成功不能替代播放器验证。

## 1. 前置知识：先把词语说清楚

### 1.1 媒体、帧、包、载荷和会话

**前置：**不需要音视频知识。

视频是连续图像，一张图像叫一**帧**；音频是不断采集的声音数值。先按H.264、AAC等规则压缩，叫**编码**；接收端还原叫**解码**。压缩数据分装进网络包。

RTP包前面的标签叫**包头**，后面装的媒体数据叫**载荷（Payload）**。一帧可能拆成多个RTP包，所以“丢2包”不等于“丢2帧”。

**会话（Session）**是一组相关通信活动。常见RTSP拉流有音频、视频两条**轨道（Track）**，各有参数和传输约定。一个RTP会话内也可能有多个来源，不能永远把端口、轨道和SSRC画等号。

### 1.2 IP、端口、UDP、TCP和Socket

| 词语 | 通俗解释 | 本章例子 |
| --- | --- | --- |
| IP地址 | 找到哪台设备 | `127.0.0.1` 是本机自己 |
| 端口 | 找到设备上的通信入口 | UDP 5005、5007 |
| Socket | 向操作系统申请的收发接口 | Windows使用Winsock |
| UDP数据报 | 一次发送的一整份消息 | 一次sendto发一份复合RTCP |
| TCP字节流 | 有顺序的一长串字节 | 一次recv可能只有半份消息或有多份 |
| 网络字节序 | 多字节整数高位字节在前 | `00 07` 是数字7 |

UDP保留消息边界，但可能丢失、乱序、重复。TCP在连接内可靠有序交付字节，但重传会等待，应用仍需恢复消息边界。RTCP可以承载在UDP或约定的TCP封装中。

普通UDP `recvfrom()` 已去掉IP/UDP头，缓冲区从RTCP首字节开始，不能再固定跳过28字节。详见 [[01-网络与Socket基础]]。

### 1.3 位、字节、十六进制和大端读取

本章Windows平台1字节是8位。`81 C9 00 07` 是4个十六进制字节，不是11个文本字符，C++写成 `{0x81, 0xc9, 0x00, 0x07}`。

```cpp
// 解释片段；完整可运行程序从 §14 开始。
std::uint8_t b = 0x81;           // 二进制10000001
unsigned version = b >> 6;      // 右移，取最高两位：2
bool padding = (b & 0x20) != 0; // &保留指定位置：这里是false
unsigned count = b & 0x1f;      // 取最低五位：1
```

`std::uint8_t/16_t/32_t/64_t` 是指定宽度无符号整数，来自 `<cstdint>`；`std::int32_t` 可表示负数。`&` 像遮罩，`|` 合并不同位置的位，`<<` 左移。

大端读取 `12 34` 是 `0x12*256+0x34=4660`。不要把网络内存直接强转成结构体：会遇到字节序、结构体填充、对齐等问题。先检查长度，再逐字节读取最清楚。

### 1.4 三种时间先分开

| 时间 | 含义 | 工具 / 单位 |
| --- | --- | --- |
| 墙上时间（Wall Clock） | 某个共同时间轴上的时刻，类似钟表 | NTP表示法、system_clock |
| 经过时间 | 从收到SR到现在等了多久 | steady_clock，不因系统校时跳动 |
| 媒体时间 | 声音/图像在媒体轴上的位置 | RTP时间戳，单位由时钟决定 |

**时钟频率（Clock Rate）**是一秒有多少时间刻度。H.264常用90000刻度/秒，Opus的RTP时钟是48000刻度/秒。它不是帧率，也不是每秒发包数。

`秒数=RTP时间戳差/时钟频率`。视频差9000刻度，在90000Hz下是0.1秒。频率通常来自SDP，见 [[04-SDP会话描述]]。

**前置检查：**

- [ ] 我知道一帧可能拆成多包。
- [ ] 我能把 `00 07` 读成7，解释位运算。
- [ ] 我知道RTP时间戳不是当前毫秒数。
- [ ] 我能区分墙上时间和经过时间。

## 2. RTCP定位：质量报告与共同时间参照

**前置：**§1。

RTCP全称 **RTP Control Protocol，RTP控制协议**。把RTP想成带标签的货物，RTCP就是质量报表和时钟对照表。

| 协议 | 负责的事情 | 记忆方法 |
| --- | --- | --- |
| RTSP | 请求播放、暂停、建立传输和结束会话 | 播放遥控器，见 [[03-RTSP协议]] |
| SDP | 描述轨道、编码和时钟 | 媒体说明书 |
| RTP | 携带媒体、标识顺序、媒体时间与来源 | 装媒体的运输包 |
| RTCP | 报告质量、标识参与者、提供同步参照、约定反馈 | 报告和对照表；不解码视频 |

```text
发送端                            接收端
  ├──────── RTP音视频 ─────────────→│
  ├──── RTCP SR：时间映射/发送量 ───→│
  │←─── RTCP RR：接收质量/时间反馈 ──┤
  └───── 两端都可发送SDES等信息 ─────┘
```

**SR（Sender Report，发送者报告）**由近期发送过RTP的参与者按规则发送；它也能带接收别人的统计。**RR（Receiver Report，接收者报告）**供没有相应发送者信息要报告的参与者报告接收情况。

双向通话者能同时发收RTP；SR/RR是报告角色，不能当成永久设备类型。只拉流的客户端通常也需按会话规则发送RR+SDES。“媒体只收”不等于“RTCP完全单向”。RTCP还通过报告帮助估计参与者数量，控制报告频率。

### 2.1 端口与TCP交织

传统RTP/AVP常见RTP用偶数端口、RTCP用下一个奇数端口，例如5004/5005。这是常见约定，真实客户端须服从SDP与SETUP协商，不能对任何RTP端口盲目加一。

```text
Transport: RTP/AVP;unicast;client_port=5004-5005;server_port=6970-6971
```

此常见RTSP 1.0例子中，客户端对应端口为5004/5005，服务器为6970/6971；客户端RTCP通常发往协商的服务器RTCP地址及端口。防火墙、网络地址转换（NAT）影响实际可达性。

TCP交织（Interleaved）可能约定 `interleaved=0-1`，0为RTP、1为RTCP。外壳为 `$ + 1字节通道 + 2字节大端长度 + 内容`。先接齐并去掉外壳，再解析内容。若协商 `interleaved=4-5`，RTCP就是通道5，不能固定写1。

扩展 `a=rtcp-mux` 在协商并满足分流规则时允许同端口承载RTP/RTCP，见RFC 5761。不要把所有数据不加区分地交给RTCP解析器。

## 3. 公共头、长度、复合包与填充

**前置：**§1的位运算与大端序。

### 3.1 四字节公共头

```text
字节偏移     0                 1            2              3
         V(2) P(1) count(5)    PT(8)     length高8位    length低8位
```

| 字段 | 位数 | 含义 |
| --- | --- | --- |
| V | 2 | 版本，目前为2 |
| P | 1 | 是否有整包Padding（填充） |
| count | 5 | SR/RR是报告块数RC，SDES/BYE是源数量SC，APP是子类型，反馈包是FMT |
| PT | 8 | RTCP类型，200=SR、201=RR等 |
| length | 16 | **当前RTCP子包**的32位字数量减一，包含头和整包填充 |

RTCP的PT是完整8位，与RTP载荷类型的意义、布局不同，不能照搬 `b1 & 0x7f`，也不要把PT=200去SDP查编码。

### 3.2 长度先加一，再乘四

```text
当前RTCP包总字节数 = (length + 1) × 4
length=7 → 32字节；length=1 → 8字节
```

`length=0` 在公共头层面是4字节，但SR至少28字节、RR至少8字节，不能据此说这个SR/RR合法。**公共长度合法和类型结构合法是两次检查。**

偏移（Offset）是从缓冲区起点数过去几个字节。剩余至少4字节才能读公共头；算完总长，还要确认不超过剩余输入。

### 3.3 一次运输包含多份报表

一个 **RTCP packet** 是有自己公共头的子包；**compound RTCP packet（复合RTCP包）**把多个子包首尾相连，通常放进同一个UDP数据报：

```text
一次UDP数据报内容： [ SR或RR ][ SDES/CNAME ][ 可选BYE等 ]
每份有自己的公共头和length，中间没有分隔字符。
```

循环：读头→算当前长度→解析→偏移增加当前长度→继续，直到刚好消费输入。不能只看第一个子包，也不能把第一个length当整份UDP内容长度。

RFC 3550经典复合规则包括以SR/RR开头，以及包含报告发送者的SDES CNAME。没有接收统计也能用RC=0的RR。CNAME要与正确的报告者对应，不能仅凭“出现任意CNAME”认定合规。

RFC 5506的 **Reduced-Size RTCP** 在约定条件下允许精简发送，例如省去部分常规信息的反馈，但仍需其规则要求的常规复合报告。不能默认所有设备接受裸PLI。本章函数解析结构，**不是完整会话合规验证器**；发送实验用SR/RR+对应CNAME。

### 3.4 两种补齐方式

1. **整包Padding：**P=1时，末字节是填充总字节数，包含它自身；复合包中只有最后一个子包可以带此填充。先检查计数非零、未吞掉公共头，再扣掉填充解析正文。
2. **SDES内部对齐：**每个描述块以END结束，再补零到4字节边界。这是内部结构，不需要置公共头P位。

不能把任意多余尾字节都当填充。

**阶段B检查：**

- [ ] 我能解释 `(length+1)*4` 是当前子包长度。
- [ ] 我知道低五位在不同PT里的含义不同。
- [ ] 我能画出SR+SDES和RR+SDES。
- [ ] 我能分清整包Padding和SDES对齐。

## 4. 常见包类型全景

**前置：**§2～§3。

| 名称 | PT / 子类型 | 用途 | 第一轮要求 |
| --- | --- | --- | --- |
| SR | 200 | 发送量、NTP/RTP映射，还能带接收报告块 | 会解析 |
| RR | 201 | 对一个或多个源的接收质量 | 会计算 |
| SDES | 202 | Source Description，源描述，含CNAME | 会识别CNAME |
| BYE | 203 | 指定来源离开，可带原因文本 | 知道它不同于超时 |
| APP | 204 | 应用自定义数据，含SSRC、4字节名称、约定内容 | 知道需双方约定 |
| RTPFB | 205，FMT=1为Generic NACK | 传输反馈，例如列出缺失序号 | 会读PID/BLP |
| PSFB | 206，FMT=1为PLI，FMT=4为FIR | 图像损坏或刷新请求 | 区分用途 |
| XR | 207 | Extended Reports，扩展测量 | 选修，RFC 3611 |

NACK、PLI、FIR不是三个新顶层PT，必须一起看 **PT+FMT**。例如PT=205,FMT=4不是FIR。

BYE是源离开RTP会话，不是RTSP TEARDOWN请求；BYE本身可能丢失，仍需活跃超时判断。APP内容按应用约定解析，不能当通用文本或JSON。

### 4.1 BYE：明确告诉别人“这个源离开了”

BYE正文先放SC个32位来源标识，可再带“1字节原因长度+对应长度的原因文本”，最后按结构补齐到4字节。原因可省略，不是C字符串，也没有SDES式条目END。

```text
81 CB 00 01  11 22 33 44
V=2，SC=1，PT=203，总长8字节，表示0x11223344离开。
```

这是单个BYE子包的格式示意。按经典复合规则发送时应放进相应复合报文，通常最后发送，不能因为它本身8字节就默认可以单独发到任何会话。接收方收到后更新源状态和资源，不能把“一个视频源离开”一概当成整个RTSP会话已销毁。

### 4.2 APP：应用之间另行约定的控制信息

APP在公共头后放SSRC、4字节应用名称，再放自定义数据；count五位此时叫subtype（子类型）。例如：

```text
80 CC 00 03  11 22 33 44  44 45 4D 4F  01 02 03 04
PT=204，subtype=0，长16字节，SSRC=0x11223344，名称为ASCII的DEMO。
```

最后4字节的意义只能由DEMO的应用规范定义，普通RTCP解析器不知道它是计数还是状态码。收到不支持的APP通常按长度安全跳过，而不是把它误认为媒体。XR则是已有规范定义的扩展报告，例如更详细的丢包和延迟测量；需要时再学习具体报告块，第一轮不实现它。

## 5. Sender Report：发送统计与时间对照

**前置：**§1时间、§3公共头。

### 5.1 SR字段布局

| 从SR起点的偏移 | 字节数 | 字段 | 含义 |
| --- | --- | --- | --- |
| 0 | 4 | 公共头 | PT=200，count是RC |
| 4 | 4 | 发送者SSRC | 谁发的SR |
| 8 | 8 | NTP Timestamp | 共同参考时间的时刻 |
| 16 | 4 | RTP Timestamp | 与上述NTP **同一时刻**的媒体时间 |
| 20 | 4 | Sender's Packet Count | 当前源开始发送以来的RTP包数 |
| 24 | 4 | Sender's Octet Count | RTP **载荷**总字节数，排除RTP头和填充 |
| 28 | 每个24 | RC个接收报告块 | 与RR报告块相同 |

**SSRC（Synchronization Source，同步源标识）**是32位数字，例如 `0x11223344`。它不是IP、端口、编码或账号。真实实现要随机选择，并处理冲突和重启；本章固定值只方便核对。

RC统计“我接收别人RTP”的报告块，不包括自己的发送信息。因此RC=0的SR合法，最小28字节、length=6；一个报告块至少52字节。特定配置可能定义尾部扩展，需按其规则解释。

### 5.2 NTP的64位怎么理解

NTP在这里首先是**时间表示法**，不是“每次发SR都访问一次校时服务器”。

```text
高32位S：整数秒；低32位F：小数秒
时间 = S + F/4294967296 秒      （4294967296=2^32）
F=0x80000000是0.5秒；F=0x40000000是0.25秒。
```

常见绝对NTP以1900-01-01为起点，Unix以1970-01-01为起点；在当前年代整数秒相差2208988800。32位秒字段2036年会回绕，长期绝对日期解释需要年代上下文。

RFC 3550允许没有绝对时间的发送端使用一致的相对时钟，完全没有相关能力时可能填零。看到非零NTP不等于发送端已与接收端精确校时。单个发送参与者的音画映射需要共同、一致的参考轴；跨设备绝对延迟还需校时及误差分析。

### 5.3 从媒体时间映射到共同参考时间

假设视频SR给出简化教学值：参考时间T0=1000.0秒，对应RTP时间R0=900000，clock_rate=90000Hz。后来包R=909000：

```text
T = T0 + (R-R0)/clock_rate
  = 1000.0 + (909000-900000)/90000
  = 1000.1秒
```

SR中的RTP时间不一定等于最近发送包的时间戳；应按媒体时钟推算到NTP对应时刻，不能把两个不同时刻的值硬拼。

RTP时间戳以2^32为模回绕。近邻差可做32位模减再解释为带符号差，前提是同一连续源、相隔小于半个周期。长时间运行要展开时间轴，不能无限外推。

### 5.4 两份SR估算发送载荷码率

两份SR隔2秒，载荷字节从100000增加到300000：

```text
载荷码率 = (300000-100000)*8/2 = 800000bit/s = 800kbit/s
```

这里kbit/s按1000bit/s计。它不是网络全部开销，也不是接收后的有效画面吞吐。计数32位会回绕；确认同一SSRC、无重启且采样跨度足够短后才做模差。

## 6. 音画同步与CNAME

**前置：**§5；会算时间戳差/时钟频率。

### 6.1 为什么不能直接比较音频和视频时间戳

两条媒体起始时间戳通常独立，时钟频率也不同。视频900000和音频48000，单看数值无法判断谁更晚。

取得同一参与者相关媒体的SR：

| 媒体 | T0 | R0 | clock_rate | 待播放包R |
| --- | --- | --- | --- | --- |
| 视频 | 1000.0秒 | 900000 | 90000Hz | 909000 |
| 音频 | 1000.0秒 | 48000 | 48000Hz | 52800 |

分别换算，两者都是1000.1秒。实际SR的NTP时刻可以不同，各用各自映射点落到同一个参考轴即可。

### 6.2 SDES和CNAME识别参与者

**SDES（Source Description，源描述）**按SSRC组织信息；**CNAME（Canonical Name，规范名称）**是类型值1的条目，用于稳定标识参与者、关联其相关RTP会话。它不是频道名称或密码。

```text
一个源描述块：SSRC(4) → 条目类型(1) → 条目长度(1) → 内容
            → 可继续其他条目 → END(一个0字节) → 补0到4字节边界
```

条目长度按字节，不带C字符串的结束符；END是下一个条目类型为0。本章ASCII `demo@local` 长10。真实CNAME可按RFC 7022建议生成，不要求公开真实账号或IP，并应在需要关联的范围保持稳定。

同一参与者音频/视频可用相同CNAME关联，但CNAME解决“属于谁”，**SR映射**才解决媒体时间关系。SSRC变化时，也需正确处理关联与状态。

### 6.3 建立时间轴之后还需要播放调度

播放器还要处理乱序、缓冲、解码耗时、声卡输出、显示刷新和时钟漂移。常见方法以音频播放时钟为主，让视频根据共同时间决定等待、显示或丢弃过晚帧。

没有SR时可结合RTSP RTP-Info、约定时间基准或本地到达时间降级；到达时间含网络波动，不能假装具有同样精度。新SR映射也应检查异常、平滑调整，避免画面突然跳时。

**阶段C检查：**

- [ ] 我能把一个新RTP时间戳换算成参考时刻。
- [ ] 我知道音视频分别除以自己的clock_rate。
- [ ] 我知道SSRC标识源、CNAME关联参与者、SR映射时间。
- [ ] 我能区分共同时间轴和按时播放。

## 7. Receiver Report：接收统计逐项理解

**前置：**§3长度、§5 SSRC。

### 7.1 外层与24字节报告块

```text
RR = 公共头(4) + 报告者SSRC(4) + RC×报告块(24) + 可选配置扩展
RC=0至少8字节；RC=1至少32字节；RC最多31。
```

超过31个源要按规范拆出更多报告。外层SSRC表示“谁写报告”，报告块里的SSRC表示“报告谁”，不要混淆。SR的接收报告块布局相同。

| 块内偏移 | 字节数 | 字段 | 意思 |
| --- | --- | --- | --- |
| 0 | 4 | Source SSRC | 被观察的RTP源 |
| 4 | 1 | Fraction Lost | 上个报告区间丢包比例，以1/256表示 |
| 5 | 3 | Cumulative Lost | 从统计起点累计丢包，**有符号24位** |
| 8 | 4 | Extended Highest Sequence Number | 已确认序号状态下的最高扩展序号 |
| 12 | 4 | Interarrival Jitter | 包间传输时间变化的平滑估计，单位RTP刻度 |
| 16 | 4 | LSR | 最近收到该源SR的NTP中间32位 |
| 20 | 4 | DLSR | 收到该SR到发报告的等待时间，单位1/65536秒 |

### 7.2 手工拆32字节RR

```text
81 C9 00 07  55 66 77 88
11 22 33 44  05 00 00 02
00 00 00 C7  00 00 00 B4
56 78 80 00  00 00 40 00
```

1. `81`：V=2、P=0、RC=1；`C9`=201，是RR。
2. `00 07`：总长32字节。
3. 报告者 `0x55667788`，报告对象 `0x11223344`。
4. Fraction=5，即5/256≈1.953125%，有量化误差。
5. 累计丢包2，最高扩展序号199，Jitter=180刻度。
6. LSR=`0x56788000`；DLSR=16384，即0.25秒。

若媒体90000Hz，180刻度是2ms；若8000Hz则是22.5ms。只看报告里的抖动数字不够，要知道媒体时钟。以上仅一个RR子包，经典发送仍需报告者对应的SDES CNAME。

## 8. 丢包：累计值、区间值、回绕与重复

**前置：**§7；RTP序号可复习 [[05-RTP协议]]。

### 8.1 先展开序号，再算预期包数

16位RTP序号 `65534、65535、0、1` 正常连续，扩展后为 `65534、65535、65536、65537`。

```text
extended_highest = cycles + max_seq
expected = extended_highest - base_seq + 1
cumulative_lost = expected - received
```

`base_seq`是接收方经过源验证的有效统计起点，不是发送者生涯第一包。`received`按RFC统计，包含迟到和重复；所以累计丢包可减小甚至为负，不是永远增加的最终计数。

从100到199预期100包，收到98包则累计缺2包。仅凭已有最高序号，不能知道其后多少包尚在路上或已丢失。

真实状态不能用“序号变小就回绕”实现：也可能是乱序、旧重复包或重启。需源试用期、合理跳变、回绕和重启判断，参考RFC 3550附录A.1，各SSRC分别统计。

### 8.2 Fraction Lost是最近区间

```text
expected_interval = expected_now - expected_previous
received_interval = received_now - received_previous
lost_interval = expected_interval - received_interval

若expected_interval<=0或lost_interval<=0：fraction=0
否则：fraction=floor(lost_interval*256/expected_interval)
```

字段范围0～255，极端上界限制为255。第一次用上次快照0计算，发送报告后更新快照。

例：上次expected=100、received=98；本次expected=150、received=147。区间预期50、收到49、缺1，字段floor(1*256/50)=5，解读约1.953125%，精确值是2%。

若最高序号未增加但补到2个旧包，区间expected=0，字段为0，累计丢包减少2。不能除零或往无符号字段里写负比例。

### 8.3 24位有符号数

```text
00 00 02 → 2；FF FF FF → -1
范围：-8388608～8388607
```

先读24位无符号数，若符号位0x800000为1，就减0x1000000。生成报告时超界应限制，不能直接截低24位让正数变负数。

媒体重组的去重计数与RFC接收报告计数有不同用途，不要合成“收到有效帧数”。小练习：预期10包、收到11包且含重复，累计值为-1，Fraction不能编码负数。

## 9. 抖动：到达节奏变化，不是总延迟

**前置：**§1.4时间单位、§7 Jitter字段。

### 9.1 同时比较媒体间隔和接收间隔

本来发两包就隔40ms，接收也隔40ms，网络可能很稳定；不能说抖动40ms。

记S_i为第i个到达包的RTP时间，R_i为本地到达时间，先统一为同一种RTP刻度：

```text
D(i-1,i) = (R_i-R_(i-1)) - (S_i-S_(i-1))
J_i = J_(i-1) + (abs(D)-J_(i-1))/16
```

abs是绝对值，1/16让新变化只占部分权重，称为平滑。第一包只保存参照，J初始0。按**实际到达顺序**更新，不能先排序。

### 9.2 手算例子

发送每包20ms，接收间隔为20ms、25ms、15ms。为便于计算，用1000Hz时钟，1刻度=1ms：

| 包对 | 接收间隔 | 媒体间隔 | D | 更新后J |
| --- | --- | --- | --- | --- |
| 1→2 | 20 | 20 | 0 | 0 |
| 2→3 | 25 | 20 | 5 | 0.3125 |
| 3→4 | 15 | 20 | -5 | 0.60546875 |

内部估计约0.605ms，写入32位报告字段会量化为整数。视频180刻度在90000Hz下是 `180*1000/90000=2ms`。

RTP与本地时钟起点可不同，差分会消去固定偏移；仍须处理回绕并用单调时钟避免本地时间跳动。

### 9.3 指标边界

固定延迟100ms但稳定的网络，Jitter可以很小。抖动不是RTT、单程延迟或缓冲长度。同帧多包共用时间戳、编码器批量发送、操作系统调度、乱序也会影响该估计，不能把它解释成“最大晚到时间”。

## 10. LSR、DLSR、RTT与其他质量指标

**前置：**§5 NTP、§7报告块。

### 10.1 先画往返过程

**RTT（Round-Trip Time，往返时间）**是去一趟再回来的时间估计，也受排队和处理影响。

```text
发送端A                                 接收端B
发SR，记录发送时间 ───── 网络去程 ─────→ 收SR
                                          │等待DLSR
收反馈，记录A_time ←──── 网络回程 ───── 发RR/SR反馈块

RTT = A_time - LSR - DLSR
```

要扣掉接收端等待；DLSR不是网络去程，也不是常规报告间隔。

### 10.2 NTP中间32位与时间单位

```text
完整NTP：12 34 56 78 80 00 00 00
中间32位：   56 78 80 00  → 0x56788000
compact = ((ntp_seconds & 0xffff)<<16) | (ntp_fraction>>16)
```

高16位为整数秒低16位、低16位为小数，单位1/65536秒。DLSR相同单位：0.25秒=16384，1.5秒=98304。

未收到该源SR时，LSR和DLSR都填0，不能套公式。收到SR后立刻反馈，DLSR允许为0。

### 10.3 手算与正确的计算端

LSR对应本端发送SR的100.0秒，DLSR=0.25秒，本端收到反馈时A_time=100.5秒，则RTT=0.25秒=250ms。三个量实际必须统一到紧凑时间单位。

**原SR发送者**在收到“报告自己”的反馈块时计算RTT。第三台旁观机器不能把自己的当前时间代入别人LSR。

经典算法将A_time转为本端NTP中间32位，做32位模减后除65536；紧凑秒字段约18.2小时回绕，墙上时钟跳变也会影响结果，必须匹配近期SR并检查合理范围。

本章UDP用等价的**本地单调时钟方法**：发SR记录steady_clock，收匹配LSR的反馈时用本地经过时间减 `DLSR/65536`。不要求两端精确校时，也避开本端墙上时间跳变。真实实现需保存近期多份SR匹配记录，不能拿旧报告随便相减。

去回程可能不对称，RTT/2不保证是单程延迟。采集到显示的**端到端延迟**还含编码、缓冲、解码、显示等，普通SR/RR不会直接给出完整值。

### 10.4 质量指标对照

| 指标 | 回答什么 | 数据来源 | 注意 |
| --- | --- | --- | --- |
| 区间丢包率 | 最近一段缺多少包 | Fraction Lost | 1/256量化 |
| 累计丢包 | 当前序号统计窗口缺多少 | 有符号24位累计值 | 可因迟到/重复减小或为负 |
| 抖动 | 到达节奏变化 | Jitter | 用媒体clock_rate换算 |
| RTT | 报告往返多久 | 匹配SR与反馈 | 不是单程或播放延迟 |
| 发送载荷码率 | 发出多少媒体字节/秒 | SR计数差、时间差 | 排除协议开销 |
| 有效吞吐量 | 收后真正可用多少 | 本地去重/重组与经过时间 | 说明统计层次 |
| 端到端延迟 | 采集到播放多久 | 可追踪采集时间、播放测量及校时 | 不能只读普通RR |

**阶段D检查：**

- [ ] 我能计算累计和区间丢包，解释负累计值。
- [ ] 我能把视频180刻度抖动换成2ms。
- [ ] 我知道RTT的计算端、三个时间量及其单位。
- [ ] 我不会把抖动、RTT、端到端延迟混为一谈。

## 11. NACK、PLI、FIR反馈扩展

**前置：**§4 PT/FMT，RTP序号与视频用途。

反馈包通常为：公共头4字节+反馈者SSRC4字节+媒体源SSRC4字节+可选 **FCI（Feedback Control Information，反馈控制信息）**。具体含义须再查PT/FMT。

### 11.1 NACK列出缺失包

Generic NACK为 `PT=205,FMT=1`。每个FCI条目4字节：PID是16位缺失包序号，BLP是后16个序号的缺失位图。

`PID=100,BLP=0x0005` 指缺100、101、103：位0对应PID+1，位1对应PID+2，位2对应PID+3，序号按16位回绕。

NACK本身只描述缺包，发送端需缓存和双方约定的恢复机制，重传封装可用RTX（RFC 4588）。重传来得太晚也可能无法播放。RR丢包指标不是明确的逐包重传请求。

### 11.2 PLI提示图像信息损坏

**PLI（Picture Loss Indication，图像丢失指示）**为 `PT=206,FMT=1`，无额外FCI，不计整包填充共12字节。它表示图像丢失、可能无法正确解码，通常促使编码器考虑恢复画面。

它不列具体缺哪些包，也不保证马上发H.264 IDR。IDR先理解为一种便于重新开始解码的图像，详见 [[07-H264与RTP封装]]。实际反应受编码器、带宽、限速策略影响。

### 11.3 FIR显式请求解码刷新点

**FIR（Full Intra Request，完整帧内刷新请求）**为 `PT=206,FMT=4`。FCI每条8字节：目标SSRC4字节+命令序号1字节+保留零位3字节。新命令加序号，重复同一命令保持序号，它不是RTP序号。

FIR公共反馈头的媒体源SSRC按规范填0，目标在FCI。不能照抄PLI的目标位置。PLI偏向损坏提示，FIR显式请求刷新；刷新点取决于编码格式，二者都不等于保证立即返回某个特定关键帧。频繁请求还会造成码率突发。

### 11.4 先看配置与能力

**AVP**是基本音视频配置；**AVPF**增加反馈时序等规则。教学SDP片段：

```sdp
m=video 5004 RTP/AVPF 96
a=rtpmap:96 H264/90000
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtcp-fb:96 ccm fir
```

这不是完整SDP。使用哪些反馈、多久发，需遵守当前配置、反馈属性与协商；普通RTP/AVP老摄像头不一定支持。未声明支持也不能依赖其响应。WebRTC的加密配置等还有其他要求，不能把此片段当完整WebRTC会话。

## 12. RTCP报告周期与带宽

**前置：**§2双向报告、§11反馈。

经典RTCP按会话带宽、成员数、活跃发送者数、平均报文大小计算周期并随机化，避免多人一起报告挤占媒体。RFC 3550常见建议为RTCP分配会话带宽约5%；发送者较少时，再把RTCP预算约25%给发送者、75%给接收者。

这些是默认建议与算法输入，不是每条流无条件多发5%。配置可改变预算，成员与发送者比例也影响分配分支。

经典固定最小间隔建议5秒，但初次发送、允许缩放、单播、反馈配置另有规则，最终还有随机化、补偿。不能硬编码“必须每5秒发”，也不能每收到一个RTP就回一个RR。

完整流程：

1. 收发时更新成员、源统计和平均报告大小。
2. 常规定时器到期，决定本次SR还是RR。
3. 取统计快照、生成报告块，附上自己的SDES CNAME。
4. 按协商传输发送，更新区间快照与下次定时器。
5. 启用AVPF时，按早期反馈、抑制、预算算法处理反馈事件。

本章UDP“收SR后等200ms发RR”是展示LSR/DLSR的单次受控实验，没有实现报告周期、成员管理或反馈调度，不能直接变成长时间RTCP引擎。

**阶段E检查：**

- [ ] 我能区分NACK、PLI、FIR，读出PID=100/BLP=5。
- [ ] 我知道反馈要有对应配置和能力。
- [ ] 我不把实验等待时间或5秒建议当固定规范周期。
- [ ] 我知道RTCP不对每个RTP逐包确认。

## 13. Windows C++实验准备

**前置：**能创建文件，了解变量、函数、循环；容器可以边读边学。

### 13.1 四个文件，三个实验

创建目录，例如 `E:\rtcp-lab`，把完整代码保存到同一个目录：

| 文件 | 来源 | 用途 |
| --- | --- | --- |
| rtcp_common.hpp | §14.1 | 共享字节读写、SR/RR/SDES解析生成 |
| rtcp_demo.cpp | §14.2 | 离线报告解析，故意输入坏包 |
| rtcp_stats.cpp | §15.1 | 离线丢包和抖动计算 |
| rtcp_udp.cpp | §16.1 | 本机双向SR/RR和RTT |

`.hpp` 是多个 `.cpp` 引用的头文件，不单独生成可执行程序。保存为UTF-8，检查没有误加 `.txt`。

`std::vector`是可变长数组，`std::string`是字符串，`const`表示不修改，参数里的 `&` 表示引用。`throw`在发现错误时抛出异常，`catch`捕获后打印原因。程序用它把坏输入与正常流程分开。

### 13.2 编译器选一种即可

**MSVC：**安装Visual Studio或Build Tools，选“使用C++的桌面开发”。从开始菜单打开 **x64 Native Tools Command Prompt for VS**：

```bat
cd /d E:\rtcp-lab
cl /nologo /std:c++17 /EHsc /W4 /utf-8 rtcp_demo.cpp /Fe:rtcp_demo.exe
cl /nologo /std:c++17 /EHsc /W4 /utf-8 rtcp_stats.cpp /Fe:rtcp_stats.exe
cl /nologo /std:c++17 /EHsc /W4 /utf-8 rtcp_udp.cpp /Fe:rtcp_udp.exe /link ws2_32.lib
```

`/EHsc`设置异常支持，`/W4`提高警告等级，`ws2_32.lib`链接Winsock。这里是cmd环境，`cd /d`跨盘切目录。

**MinGW-w64 g++：**确认其bin目录在PATH，在PowerShell执行：

```powershell
Set-Location E:\rtcp-lab
g++ -std=c++17 -Wall -Wextra -pedantic rtcp_demo.cpp -o rtcp_demo.exe
g++ -std=c++17 -Wall -Wextra -pedantic rtcp_stats.cpp -o rtcp_stats.exe
g++ -std=c++17 -Wall -Wextra -pedantic rtcp_udp.cpp -o rtcp_udp.exe -lws2_32
```

不要混用两种编译器参数。后续运行命令按PowerShell书写，当前目录下的程序前面要加 `.\`。

## 14. 实验一：离线读写SR、RR、SDES

**前置：**§3、§5、§7，已准备编译器。

先证明能读字节，再加网络。程序解析普通SR/RR、SDES；其他类型只按公共长度跳过，**未完成反馈、BYE、APP等全部结构校验**。解析成功不代表满足所有会话和安全规则。

### 14.1 保存为rtcp_common.hpp

```cpp
#pragma once
#include <algorithm>
#include <cstdint>
#include <iostream>
#include <stdexcept>
#include <string>
#include <utility>
#include <vector>

using Bytes = std::vector<std::uint8_t>;

inline void need(bool ok, const char* message) {
    if (!ok) throw std::runtime_error(message);
}

// Call only after checking that enough bytes remain.
inline std::uint16_t read16(const std::uint8_t* p) {
    return (std::uint16_t(p[0]) << 8) | p[1];
}
inline std::uint32_t read32(const std::uint8_t* p) {
    return (std::uint32_t(p[0]) << 24) |
           (std::uint32_t(p[1]) << 16) |
           (std::uint32_t(p[2]) << 8) | p[3];
}
inline void put16(Bytes& b, std::uint16_t x) {
    b.push_back(std::uint8_t(x >> 8)); b.push_back(std::uint8_t(x));
}
inline void put32(Bytes& b, std::uint32_t x) {
    b.push_back(std::uint8_t(x >> 24)); b.push_back(std::uint8_t(x >> 16));
    b.push_back(std::uint8_t(x >> 8)); b.push_back(std::uint8_t(x));
}

struct Report {
    std::uint32_t source = 0;
    std::uint8_t fraction = 0;
    std::int32_t lost = 0;
    std::uint32_t highest = 0, jitter = 0, lsr = 0, dlsr = 0;
};

struct Packet {
    unsigned pt = 0, count = 0;
    std::uint32_t sender = 0;
    std::uint64_t ntp = 0;
    std::uint32_t rtp = 0, packets = 0, octets = 0;
    std::vector<Report> reports;
    std::vector<std::pair<std::uint32_t, std::string>> cnames;
};

inline std::uint32_t compactNtp(std::uint64_t ntp) {
    return std::uint32_t(ntp >> 16); // Middle 32 bits.
}

inline std::vector<Packet> parseRtcp(const Bytes& data) {
    need(!data.empty(), "empty input");
    std::vector<Packet> result;
    std::size_t offset = 0;
    while (offset < data.size()) {
        need(data.size() - offset >= 4, "truncated RTCP header");
        const auto* p = data.data() + offset;
        need((p[0] >> 6) == 2, "wrong RTCP version");
        const std::size_t total = (std::size_t(read16(p + 2)) + 1) * 4;
        need(total <= data.size() - offset, "length exceeds input");
        std::size_t bodyEnd = total;
        if ((p[0] & 0x20) != 0) {
            need(offset + total == data.size(), "padding before final packet");
            const unsigned padding = p[total - 1];
            need(padding != 0 && padding <= total - 4, "bad padding count");
            bodyEnd -= padding;
        }

        Packet packet;
        packet.pt = p[1]; packet.count = p[0] & 0x1f;
        if (packet.pt == 200 || packet.pt == 201) {
            const std::size_t fixed = packet.pt == 200 ? 28 : 8;
            need(bodyEnd >= fixed + 24 * packet.count, "short SR/RR body");
            need((bodyEnd - fixed) % 4 == 0, "unaligned SR/RR extension");
            packet.sender = read32(p + 4);
            if (packet.pt == 200) {
                packet.ntp = (std::uint64_t(read32(p + 8)) << 32) | read32(p + 12);
                packet.rtp = read32(p + 16);
                packet.packets = read32(p + 20); packet.octets = read32(p + 24);
            }
            for (unsigned i = 0; i < packet.count; ++i) {
                const auto* q = p + fixed + 24 * i;
                Report r;
                r.source = read32(q); r.fraction = q[4];
                const std::uint32_t raw = (std::uint32_t(q[5]) << 16) |
                                         (std::uint32_t(q[6]) << 8) | q[7];
                r.lost = (raw & 0x800000) ? std::int32_t(raw) - 0x1000000
                                        : std::int32_t(raw);
                r.highest = read32(q + 8); r.jitter = read32(q + 12);
                r.lsr = read32(q + 16); r.dlsr = read32(q + 20);
                packet.reports.push_back(r);
            }
            // Extra profile-specific words are not interpreted here.
        } else if (packet.pt == 202) {
            std::size_t pos = 4;
            for (unsigned i = 0; i < packet.count; ++i) {
                need(bodyEnd - pos >= 4, "short SDES source");
                const auto source = read32(p + pos); pos += 4;
                bool ended = false;
                while (pos < bodyEnd) {
                    const unsigned type = p[pos++];
                    if (type == 0) { ended = true; break; }
                    need(pos < bodyEnd, "missing SDES item length");
                    const unsigned size = p[pos++];
                    need(size <= bodyEnd - pos, "short SDES item");
                    if (type == 1) {
                        packet.cnames.push_back({source,
                            std::string(reinterpret_cast<const char*>(p + pos), size)});
                    }
                    pos += size;
                }
                need(ended, "missing SDES END");
                while (pos % 4 != 0) {
                    need(pos < bodyEnd && p[pos] == 0, "bad SDES alignment");
                    ++pos;
                }
            }
            need(pos == bodyEnd, "extra SDES bytes");
        }
        result.push_back(packet); offset += total;
    }
    return result;
}

inline Bytes makeSR(std::uint32_t source, std::uint64_t ntp,
                    std::uint32_t rtp, std::uint32_t packets, std::uint32_t octets) {
    Bytes b{0x80, 200, 0, 6}; // RC=0, 28 bytes.
    put32(b, source);
    put32(b, std::uint32_t(ntp >> 32)); put32(b, std::uint32_t(ntp));
    put32(b, rtp); put32(b, packets); put32(b, octets);
    return b;
}

inline Bytes makeRR(std::uint32_t reporter, const Report& r) {
    Bytes b{0x81, 201, 0, 7}; // RC=1, 32 bytes.
    put32(b, reporter); put32(b, r.source); b.push_back(r.fraction);
    const auto limited = std::clamp(r.lost, std::int32_t(-8388608), std::int32_t(8388607));
    const auto raw = std::uint32_t(limited) & 0xffffff;
    b.push_back(std::uint8_t(raw >> 16)); b.push_back(std::uint8_t(raw >> 8));
    b.push_back(std::uint8_t(raw));
    put32(b, r.highest); put32(b, r.jitter); put32(b, r.lsr); put32(b, r.dlsr);
    return b;
}

inline Bytes makeSdes(std::uint32_t source, const std::string& cname) {
    need(!cname.empty() && cname.size() <= 255, "CNAME needs 1..255 bytes");
    Bytes b{0x81, 202, 0, 0}; // SC=1; length filled later.
    put32(b, source); b.push_back(1); b.push_back(std::uint8_t(cname.size()));
    b.insert(b.end(), cname.begin(), cname.end()); b.push_back(0); // END.
    while (b.size() % 4 != 0) b.push_back(0); // SDES alignment; P stays zero.
    const auto length = std::uint16_t(b.size() / 4 - 1);
    b[2] = std::uint8_t(length >> 8); b[3] = std::uint8_t(length);
    return b;
}

inline void append(Bytes& a, const Bytes& b) { a.insert(a.end(), b.begin(), b.end()); }

inline void show(const std::vector<Packet>& packets) {
    for (const auto& p : packets) {
        std::cout << "PT=" << p.pt << " count=" << p.count << '\n';
        if (p.pt == 200 || p.pt == 201) {
            std::cout << "  reporter=0x" << std::hex << p.sender << std::dec << '\n';
            if (p.pt == 200) {
                std::cout << "  NTP=0x" << std::hex << p.ntp << std::dec
                          << " RTP=" << p.rtp << " packets=" << p.packets
                          << " payload_bytes=" << p.octets << '\n';
            }
            for (const auto& r : p.reports) {
                std::cout << "  source=0x" << std::hex << r.source << std::dec
                          << " fraction=" << unsigned(r.fraction)
                          << " lost=" << r.lost << " highest=" << r.highest
                          << " jitter=" << r.jitter << '\n';
                std::cout << "  LSR=0x" << std::hex << r.lsr << std::dec
                          << " DLSR=" << r.dlsr << '\n';
            }
        } else if (p.pt == 202) {
            for (const auto& item : p.cnames) {
                std::cout << "  CNAME source=0x" << std::hex << item.first
                          << std::dec << " value=" << item.second << '\n';
            }
        } else {
            std::cout << "  body skipped (unsupported type)\n";
        }
    }
}
```

先抓四条主线：read16/read32把字节变数字；parseRtcp按长度走；Report保存24字节块的含义；makeSR/makeRR/makeSdes把数字写回字节。

read16/read32自身不检查长度，调用者要先检查。本章解析器先检查再读取；不能在别处随意用它们读取任意指针。SDES的reinterpret_cast只用于按已验证长度读取字符内容，和把网络内存强转成结构体不同。

**不会一下读完187行也没关系，按下面顺序拆读：**

| 阅读步骤 | 对照协议做什么 | 你可以先观察什么 |
| --- | --- | --- |
| 1 | 看Bytes、read16、read32 | 4字节怎样按大端变成32位数字 |
| 2 | 看Report和Packet | 字段名与§5、§7的表格一一对应 |
| 3 | 看makeSR | 从固定28字节报告开始，不涉及循环 |
| 4 | 看makeRR | 除累计丢包外，主要是依次写32位数字 |
| 5 | 看parseRtcp外层while | offset走到哪里、total保护什么 |
| 6 | 看SR/RR分支 | fixed区分28/8字节，for逐个读24字节块 |
| 7 | 再看SDES分支 | 条目长度、END和对齐为什么分三步处理 |
| 8 | 最后看show和异常处理 | std::hex只改变显示格式，std::dec恢复十进制 |

`auto`让编译器推导变量类型；`const auto* p`表示不通过该指针修改字节；`p+4`就是从当前位置向后4字节。`size_t`用于数组长度和下标，先验证再减法，避免无符号数下溢。`push_back`添加一个元素，`insert`添加一段元素。打印uint8_t时转unsigned，防止被当成字符。

### 14.2 保存为rtcp_demo.cpp

```cpp
#include "rtcp_common.hpp"

int main() {
    try {
        const std::uint32_t mediaSource = 0x11223344;
        const std::uint32_t receiver = 0x55667788;
        const std::uint64_t ntp = 0x1234567880000000ULL; // Synthetic time.

        Bytes sr = makeSR(mediaSource, ntp, 900000, 100, 100000);
        append(sr, makeSdes(mediaSource, "demo@local"));
        std::cout << "SR compound bytes=" << sr.size() << '\n';
        show(parseRtcp(sr));

        Report r;
        r.source = mediaSource; r.fraction = 5; r.lost = 2;
        r.highest = 199; r.jitter = 180;
        r.lsr = compactNtp(ntp); r.dlsr = 16384;
        Bytes rr = makeRR(receiver, r);
        append(rr, makeSdes(receiver, "recv@local"));
        std::cout << "RR compound bytes=" << rr.size() << '\n';
        show(parseRtcp(rr));
        std::cout << "loss_percent=" << r.fraction * 100.0 / 256
                  << " jitter_ms=" << r.jitter * 1000.0 / 90000 << '\n';

        Bytes broken = rr;
        broken[2] = 0xff; broken[3] = 0xff;
        try {
            (void)parseRtcp(broken);
            std::cerr << "ERROR: bad length was accepted\n";
            return 1;
        } catch (const std::exception& e) {
            std::cout << "Bad packet rejected: " << e.what() << '\n';
        }
    } catch (const std::exception& e) {
        std::cerr << e.what() << '\n';
        return 1;
    }
}
```

### 14.3 运行与预期输出

按§13编译后：

```powershell
.\rtcp_demo.exe
```

```text
SR compound bytes=52
PT=200 count=0
  reporter=0x11223344
  NTP=0x1234567880000000 RTP=900000 packets=100 payload_bytes=100000
PT=202 count=1
  CNAME source=0x11223344 value=demo@local
RR compound bytes=56
PT=201 count=1
  reporter=0x55667788
  source=0x11223344 fraction=5 lost=2 highest=199 jitter=180
  LSR=0x56788000 DLSR=16384
PT=202 count=1
  CNAME source=0x55667788 value=recv@local
loss_percent=1.95312 jitter_ms=2
Bad packet rejected: length exceeds input
```

52=28字节SR+24字节SDES；56=32字节RR+24字节SDES。固定NTP是教学时间，不是当前日期。这里的两个CNAME分别属于发送参与者与接收参与者；同一参与者内部相关音视频才需要稳定关联。

### 14.4 三个修改练习

1. `r.lost=2` 改成-1，应输出lost=-1，而不是16777215。
2. `r.fraction=5` 改成128，应输出loss_percent=50。
3. 对构造完的rr副本执行pop_back()，应报输入不足，不能越界读取。

**阶段F解析检查：**

- [ ] 我能保存文件、编译并得到上述输出。
- [ ] 我能指出每项检查保护了哪次读取。
- [ ] 我理解拒绝畸形输入是预期行为。
- [ ] 我知道结构解析不等于完整会话实现。

## 15. 实验二：最小丢包和抖动计算

**前置：**§8～§9，函数、浮点数。

### 15.1 保存为rtcp_stats.cpp

输入是**已经验证和展开的累计快照**，让注意力集中在公式；不从原始RTP序号自动判断回绕，不是完整接收统计器。

```cpp
#include <algorithm>
#include <cmath>
#include <cstdint>
#include <iostream>

struct LossInterval {
    std::int64_t previousExpected = 0, previousReceived = 0;
    void report(std::int64_t expected, std::int64_t received) {
        const auto e = expected - previousExpected;
        const auto r = received - previousReceived;
        const auto lost = e - r;
        std::int64_t fraction = 0;
        if (e > 0 && lost > 0) fraction = std::min<std::int64_t>(255, lost * 256 / e);
        std::cout << "cumulative=" << expected - received
                  << " interval_expected=" << e
                  << " interval_lost=" << lost << " fraction=" << fraction << '\n';
        previousExpected = expected; previousReceived = received;
    }
};

struct Jitter {
    bool first = true;
    double previousArrival = 0, value = 0;
    std::uint32_t previousStamp = 0;
    void packet(double arrivalSeconds, std::uint32_t stamp, double clockRate) {
        const double arrivalTicks = arrivalSeconds * clockRate;
        if (!first) {
            // Interpret a nearby 32-bit modular timestamp difference as signed.
            const std::uint32_t raw = stamp - previousStamp;
            const std::int64_t delta = raw < 0x80000000u ? std::int64_t(raw)
                : std::int64_t(raw) - 0x100000000LL;
            const double d = (arrivalTicks - previousArrival) - double(delta);
            value += (std::abs(d) - value) / 16.0;
        }
        first = false; previousArrival = arrivalTicks; previousStamp = stamp;
        std::cout << "jitter_ticks=" << value
                  << " jitter_ms=" << value * 1000.0 / clockRate << '\n';
    }
};

int main() {
    LossInterval loss;
    loss.report(100, 98);  // First interval: 2 / 100.
    loss.report(150, 147); // Next interval: 1 / 50.
    loss.report(150, 149); // Two late packets; no new highest sequence.
    loss.report(150, 151); // Duplicates can make cumulative loss negative.

    Jitter j;
    j.packet(0.000, 0, 1000);
    j.packet(0.020, 20, 1000);
    j.packet(0.045, 40, 1000);
    j.packet(0.060, 60, 1000);
}
```

int64_t用64位累计计数；double保留抖动小数。实际接收用会话相对单调时间记录到达秒数，源重启则重置对应状态。此例假定clock_rate始终是同一源的固定正数。

### 15.2 运行与解释

按§13编译后：

```powershell
.\rtcp_stats.exe
```

```text
cumulative=2 interval_expected=100 interval_lost=2 fraction=5
cumulative=3 interval_expected=50 interval_lost=1 fraction=5
cumulative=1 interval_expected=0 interval_lost=-2 fraction=0
cumulative=-1 interval_expected=0 interval_lost=-2 fraction=0
jitter_ticks=0 jitter_ms=0
jitter_ticks=0 jitter_ms=0
jitter_ticks=0.3125 jitter_ms=0.3125
jitter_ticks=0.605469 jitter_ms=0.605469
```

第三行展示迟到减少累计缺口，第四行展示重复可使累计为负。抖动输出对应§9手算。

练习：第三包到达0.045改为0.040，第四包仍0.060，J应一直为0。再把时钟与所有RTP时间戳乘90，J刻度乘90，但毫秒结果不变。

## 16. 实验三：Windows UDP双向SR→RR

**前置：**离线实验通过；bind选择本地端口，sendto指定目标，recvfrom收一份数据报。

```text
发送角色127.0.0.1:5005 ─── SR+SDES ──→ 接收角色127.0.0.1:5007
                       ←── RR+SDES ───
```

5005/5007都是实验约定的RTCP端口，不是同一轨道的RTP/RTCP端口对。程序不发RTP，不测真实丢包：SR的包数、字节数、RTP时间是合成数据，RR的丢包与抖动填0。真实测量仅有等待时间及本机报告往返过程。

为单独观察LSR/DLSR，实验手工放入一个合成接收报告块。**真实接收方不能只因收到SR，就编造该源的RTP接收统计。**还没收到该源RTP、没有符合规则的统计时，不应照抄这个报告块；没有任何源可报告时可发RC=0的RR+SDES。接入真实客户端时，报告块必须由实际接收状态和当前报告规则产生。

### 16.1 保存为rtcp_udp.cpp

```cpp
#define WIN32_LEAN_AND_MEAN
#define NOMINMAX
#include <winsock2.h>
#include <ws2tcpip.h>
#include <chrono>
#include <thread>
#include "rtcp_common.hpp"

using Clock = std::chrono::steady_clock;

struct Winsock {
    Winsock() {
        WSADATA data{};
        const int error = WSAStartup(MAKEWORD(2, 2), &data);
        if (error != 0) throw std::runtime_error("WSAStartup error=" + std::to_string(error));
    }
    ~Winsock() { WSACleanup(); }
};

struct UdpSocket {
    SOCKET value = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    UdpSocket() { need(value != INVALID_SOCKET, "socket failed"); }
    ~UdpSocket() { closesocket(value); }
    UdpSocket(const UdpSocket&) = delete;
    UdpSocket& operator=(const UdpSocket&) = delete;
};

void socketOK(bool ok, const char* operation) {
    if (!ok) throw std::runtime_error(std::string(operation) + " WSA=" +
                                      std::to_string(WSAGetLastError()));
}

sockaddr_in loopback(unsigned short port) {
    sockaddr_in address{};
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
    address.sin_port = htons(port);
    return address;
}

void sendPacket(SOCKET socket, const Bytes& data, const sockaddr_in& target) {
    const int sent = sendto(socket, reinterpret_cast<const char*>(data.data()),
        int(data.size()), 0, reinterpret_cast<const sockaddr*>(&target), sizeof(target));
    socketOK(sent == int(data.size()), "sendto");
}

Bytes receivePacket(SOCKET socket, sockaddr_in& from) {
    Bytes data(2048); // Lab packets are under 100 bytes.
    int addressSize = sizeof(from);
    const int n = recvfrom(socket, reinterpret_cast<char*>(data.data()), int(data.size()),
        0, reinterpret_cast<sockaddr*>(&from), &addressSize);
    socketOK(n != SOCKET_ERROR, "recvfrom");
    data.resize(std::size_t(n));
    return data;
}

std::uint64_t currentNtp() {
    using namespace std::chrono;
    const auto ns = duration_cast<nanoseconds>(system_clock::now().time_since_epoch()).count();
    const std::uint64_t unixSeconds = std::uint64_t(ns / 1000000000LL);
    const std::uint64_t remainder = std::uint64_t(ns % 1000000000LL);
    const auto seconds = std::uint32_t(unixSeconds + 2208988800ULL);
    const auto fraction = std::uint32_t((remainder << 32) / 1000000000ULL);
    return (std::uint64_t(seconds) << 32) | fraction;
}

int main(int argc, char** argv) {
    try {
        need(argc == 2 && (std::string(argv[1]) == "send" || std::string(argv[1]) == "recv"),
             "usage: rtcp_udp.exe send|recv");
        const bool senderMode = std::string(argv[1]) == "send";
        Winsock winsock;
        UdpSocket udp; // Created after WSAStartup, closed before WSACleanup.
        const auto local = loopback(senderMode ? 5005 : 5007);
        socketOK(bind(udp.value, reinterpret_cast<const sockaddr*>(&local), sizeof(local)) !=
                 SOCKET_ERROR, "bind");
        DWORD timeout = 5000; // Windows SO_RCVTIMEO is milliseconds.
        socketOK(setsockopt(udp.value, SOL_SOCKET, SO_RCVTIMEO,
            reinterpret_cast<const char*>(&timeout), sizeof(timeout)) != SOCKET_ERROR,
            "setsockopt");

        if (senderMode) {
            const auto ntp = currentNtp();
            Bytes data = makeSR(0x11223344, ntp, 900000, 100, 100000);
            append(data, makeSdes(0x11223344, "sender@local"));
            const auto sentAt = Clock::now();
            sendPacket(udp.value, data, loopback(5007));
            std::cout << "SR sent; waiting for RR..." << std::endl;
            sockaddr_in from{};
            const Bytes reply = receivePacket(udp.value, from);
            const auto arrivedAt = Clock::now();
            need(from.sin_addr.s_addr == local.sin_addr.s_addr && from.sin_port == htons(5007),
                 "unexpected peer");
            const auto packets = parseRtcp(reply);
            show(packets);
            bool matched = false;
            for (const auto& p : packets) {
                if (p.pt != 201 || p.sender != 0x55667788) continue;
                for (const auto& r : p.reports) {
                    if (r.source != 0x11223344 || r.lsr != compactNtp(ntp) || r.lsr == 0) continue;
                    const double elapsed = std::chrono::duration<double>(arrivedAt - sentAt).count();
                    const double rtt = elapsed - r.dlsr / 65536.0;
                    need(rtt >= 0 && rtt < 5, "implausible RTT");
                    std::cout << "RTT_ms=" << rtt * 1000 << '\n';
                    matched = true;
                }
            }
            need(matched, "no matching RR block");
        } else {
            std::cout << "Listening on 127.0.0.1:5007 (5 second timeout)..." << std::endl;
            sockaddr_in from{};
            const Bytes input = receivePacket(udp.value, from);
            const auto receivedAt = Clock::now();
            need(from.sin_addr.s_addr == local.sin_addr.s_addr && from.sin_port == htons(5005),
                 "unexpected peer");
            const auto packets = parseRtcp(input);
            show(packets);
            need(!packets.empty() && packets[0].pt == 200 && packets[0].sender == 0x11223344,
                 "expected demo SR first");
            std::this_thread::sleep_for(std::chrono::milliseconds(200)); // Lab delay only.
            Report r;
            r.source = packets[0].sender;
            r.lsr = compactNtp(packets[0].ntp);
            const double held = std::chrono::duration<double>(Clock::now() - receivedAt).count();
            r.dlsr = std::uint32_t(held * 65536.0);
            Bytes reply = makeRR(0x55667788, r);
            append(reply, makeSdes(0x55667788, "receiver@local"));
            sendPacket(udp.value, reply, from);
            std::cout << "RR sent; DLSR_seconds=" << r.dlsr / 65536.0 << '\n';
        }
    } catch (const std::exception& e) {
        std::cerr << e.what() << '\n';
        return 1;
    }
}
```

### 16.2 运行顺序与预期结果

按§13编译，在两个PowerShell终端都切到 `E:\rtcp-lab`。

终端A先监听：

```powershell
.\rtcp_udp.exe recv
```

看到监听提示后，**5秒内**在终端B执行：

```powershell
.\rtcp_udp.exe send
```

可以在两个窗口先输入好命令，A回车后马上B回车。来不及就重新运行；或把timeout=5000改为15000，重新编译，给自己15秒。

A应打印收到的SR/SDES及 `RR sent; DLSR_seconds=...`；B打印RR/SDES及 `RTT_ms=...`。等待约0.2秒，但RTT已扣掉等待，**不应固定为200ms**。

系统负载、调度、量化影响数字，NTP/LSR每次也会变化。RTT含少量构造和系统处理开销，例子帮助理解关系，不是精密测量仪。

### 16.3 每个Windows API的任务

| 调用 | 目的 | 易错点 |
| --- | --- | --- |
| WSAStartup | 初始化本进程Winsock | 不要在它之前创建Socket |
| socket | 创建IPv4 UDP接口 | 不要误用TCP常量 |
| bind | 固定本地地址端口 | 检查端口占用 |
| setsockopt | 设置接收超时 | Windows此选项用毫秒DWORD，不是Linux的timeval |
| sendto | 指定对方并发一份数据报 | 区分本地和目标地址 |
| recvfrom | 收数据并取得来源 | 检查错误后才能使用长度 |
| closesocket / WSACleanup | 释放接口与Winsock环境 | 不用Linux的close，也不提前清理Winsock |

小结构体的析构函数自动清理资源，这叫 **RAII**：对象离开作用域释放所拥有资源。Winsock先构造最后析构，Socket先关闭。程序单次实验遇坏包报错退出，真实服务应隔离坏输入并按策略继续。

### 16.4 故意制造错误

1. 只开接收端：约5秒后 `recvfrom WSA=10060`，是超时，不是解析失败。
2. 同时开两个接收端：第二个通常 `bind WSA=10048`，端口被占用；等第一个退出或关闭后重试。
3. 不带参数运行 `.\rtcp_udp.exe`：输出用法，以非零状态结束。
4. 暂不接摄像头。固定SSRC、端口和合成统计不匹配真实RTSP，先按§17替换协商及统计来源。

只绑定环回，无需开放公网。2048字节缓冲区足够本实验；真实UDP过大时Windows可能返回WSAEMSGSIZE=10040，应报告并调整策略，不能解析截断内容。

**阶段G检查：**

- [ ] 我能在两个终端完成SR和RR双向交换。
- [ ] 我能指出5005/5007分别属于哪个角色。
- [ ] 我能解释DLSR约200ms但RTT通常明显较小。
- [ ] 我知道实验没发真实RTP，没测出真实丢包/抖动。
- [ ] 我做过超时和参数错误并会恢复。

## 17. 接入真实RTSP客户端与抓包

**前置：**实验通过；参考 [[03-RTSP协议]] SETUP与 [[05-RTP协议]] 包头。

### 17.1 推荐数据与线程结构

```text
RTSP/SDP配置 → 轨道、clock_rate、端口/通道、反馈能力
                         │
UDP数据报 / TCP组帧 → 按协商分流
                         ├─ RTP解析 → 按SSRC更新统计 → 媒体解包 → 解码/播放
                         └─ RTCP解析 → 保存SR映射/到达时刻、报告、CNAME
定时器 → 取统计快照 → 生成SR/RR+SDES → 按协商传输发出
播放时钟 ← SR映射 + 媒体时间 + 缓冲策略
```

初期可用单线程事件循环处理收包和定时器，减少共享状态。分成网络、解码、播放线程后，用短时间加锁复制统计快照，在锁外构造和发送；不让解码阻塞RTCP收发。

每源至少记录：会话/媒体上下文及SSRC、clock_rate、有效起始/最高扩展序号、接收计数、上次报告快照、上次到达时间与RTP时间、Jitter、最近SR的NTP/RTP及到达单调时间、CNAME、活跃时间。发送者保存近期SR的LSR与本地发送时刻以匹配RTT。

相同数字SSRC在不同RTP会话不能自动合并，状态键要包含正确会话上下文。源重启、SSRC变更或碰撞要按规则重新统计和映射，不能把旧流时间差带入新流。

### 17.2 逐步替换教学数据

1. 保留独立字节解析器，不在Socket读取时强转结构体。
2. 从SDP取时钟，从SETUP响应取真实地址、端口/通道。
3. 用真实RTP更新源状态，先做好源验证、回绕、乱序、重复、重启。
4. 收SR保存映射和本地到达时刻，用来生成LSR/DLSR。
5. 实现常规RTCP调度，正确更新报告区间快照。
6. 再做同步播放，最后按能力添加反馈和重传。

只收媒体通常发RR；自己发RTP时按规则发SR，可附接收别人的报告块。TCP承载不意味着RTCP同步和统计全无用途，也不能保证播放前从不因积压丢弃数据。

### 17.3 Wireshark抓本机实验

**Wireshark**是显示网络字段的工具，Windows环回抓包通常需要支持环回捕获的Npcap。

1. 选择对应环回捕获接口，例如Adapter for loopback traffic capture，名称随版本变化。
2. 开始捕获，再运行两个实验角色。
3. 用显示过滤器 `udp.port == 5005 || udp.port == 5007`。
4. 找到两个方向数据。若未自动识别，可右键 **Decode As（解码为）**，按当前版本选项选择RTCP或对应RTP/RTCP解析器。
5. 核对一份数据报内SR+SDES，另一份RR+SDES。
6. 核对V=2、SR RC=0、RR RC=1、两种SSRC、LSR中间32位及DLSR约0.2秒。

`rtcp`过滤器仅在已识别为RTCP时有效，无结果先按UDP端口找。真实流同时对照DESCRIBE/SDP、SETUP、RTP和RTCP，不能用实验5005推断真实端口。

继续看 [[02-抓包分析一次RTSP拉流]]、[[04-验证UDP传输]]、[[03-验证TCP交织传输]]。

## 18. 常见错误、练习与参考答案

### 18.1 按现象排查

| 现象 | 常见原因 | 优先检查 / 修复 |
| --- | --- | --- |
| 找不到cl/g++ | 工具未装或终端环境不对 | MSVC开发者命令提示符；MinGW检查PATH |
| 找不到rtcp_common.hpp | 目录/扩展名错误 | 检查四文件同目录且没有.txt |
| WSAStartup链接错误 / LNK2019 | 未链接Winsock | g++加-lws2_32；MSVC加ws2_32.lib |
| std::min/max附近报错 | Windows宏污染 | 包含Windows头前定义NOMINMAX |
| bind WSA=10048 | 端口占用 | 关闭冲突进程，改端口需两端一起改 |
| recvfrom WSA=10060 | 超时或地址端口不匹配 | 先看启动顺序，真实流再查协商与防火墙 |
| recvfrom WSA=10054 | 部分Windows路径反馈UDP目标不可达 | 确认对端已监听、目标正确后重试 |
| wrong RTCP version | 把RTP/RTSP文本/TCP外壳送进解析器 | 检查分流、起始偏移，先去外壳 |
| length exceeds input | length单位错、截短或TCP半包 | 用(length+1)*4，先接齐TCP内容 |
| 只见SR无CNAME | 只读复合包首个或来源不含预期信息 | 按长度遍历，核对会话配置 |
| lost=16777215 | 24位有符号数读错 | 0xffffff应符号扩展为-1 |
| 丢包率过大 | 忘除256或百分号换算错误 | 字段/256为比例，乘100为百分比 |
| 抖动异常 | 秒/毫秒/刻度混用，时钟跳动 | 查clock_rate、模差和单调时间 |
| RTT巨大 | 无有效LSR、错源、旧报告、单位错 | 匹配近期SR，检查合理范围 |
| 有报告仍花屏 | RTCP不自动修复媒体 | 查RTP、H.264重组、解码参考和反馈支持 |
| 音画渐渐不同步 | 映射、时钟漂移或播放调度 | 分别核对时钟、SR、主播放时钟 |

排查顺序：传输收到没有→边界是否正确→字段是否正确→统计单位→媒体解码/播放。继续看 [[05-断线重连与故障排查]]、[[11-调试与性能优化]]。

### 18.2 先独立做，再看答案

**概念与报文：**

1. RTSP/RTP/RTCP各干什么？RTCP里有普通H.264视频帧吗？
2. 只看视频的客户端为什么仍可能发RTCP？
3. `81 C9 00 07` 的V、RC、PT、总字节数分别是多少？
4. SR RC=0为什么合法？最小字节数和length多少？
5. 28字节SR和24字节SDES拼成一个UDP内容共多大？首个length变吗？
6. PT=205,FMT=4是不是FIR？
7. 报告者SSRC甲、块内Source SSRC乙，指标在说谁？
8. CNAME相同就能直接比较音视频RTP时间戳吗？

**手工计算：**

9. 起始1000、最高扩展1099、received=97，expected和累计丢包多少？
10. 上次expected=100/received=97，本次200/195，区间Fraction字段多少？
11. `FF FF FE` 的累计丢包是多少？Fraction能写负值吗？
12. Jitter=240、clock_rate=48000，是多少ms？
13. 上次J=16刻度，本次abs(D)=32，更新后多少？
14. NTP=0x1234567880000000，LSR多少？DLSR=32768代表多久？
15. 发SR到收匹配反馈经过0.8秒，DLSR=32768，RTT多少？单程能确定吗？
16. 视频T0=10秒/R0=180000/clock_rate=90000，包R=225000映射到几秒？
17. 两SR隔4秒，载荷计数增加400000字节，码率多少kbit/s？
18. NACK PID=65535、BLP=0x0003，列出哪些16位序号？

**代码与综合：**

19. 实验一lost改-1、输入截掉1字节，分别应怎样？
20. 抖动实验全部到达间隔20ms，J多少？为什么？
21. 实验三RR丢包0为什么不能证明网络没丢RTP？
22. TCP只交付交织头前3字节，能调用本章解析器吗？
23. 接收端等200ms，发送端算RTT约1ms，是否一定错误？
24. 收到SR却没画面，下一步检查什么？

### 18.3 参考答案

| 题号 | 答案要点 |
| --- | --- |
| 1 | RTSP控制播放，RTP传媒体，RTCP报告/参照/约定反馈；无普通视频帧 |
| 2 | 用RR报告质量、SDES标识参与者，并参与会话控制 |
| 3 | V=2、RC=1、PT=201、32字节 |
| 4 | 自身发送统计不占RC；RC=0为无接收报告块；28字节，length=6 |
| 5 | 52字节；首个length仍6，只描述SR自己 |
| 6 | 不是；FIR必须PT=206,FMT=4 |
| 7 | 乙；甲是报告撰写者 |
| 8 | 不能，还要各自SR映射和clock_rate |
| 9 | expected=1099-1000+1=100，lost=3 |
| 10 | 区间预期100、收到98、缺2，floor(2*256/100)=5 |
| 11 | -2；Fraction按规则填0，不编码负比例 |
| 12 | 240*1000/48000=5ms |
| 13 | 16+(32-16)/16=17刻度 |
| 14 | LSR=0x56788000，32768/65536=0.5秒 |
| 15 | RTT=0.8-0.5=0.3秒=300ms；单程不一定150ms |
| 16 | 10+(225000-180000)/90000=10.5秒 |
| 17 | 400000*8/4=800000bit/s=800kbit/s，排除协议开销 |
| 18 | 65535、0、1，按16位回绕 |
| 19 | 输出lost=-1；截短输入应报错，不越界 |
| 20 | 一直0；接收间隔等于媒体间隔，D为0 |
| 21 | 没发送RTP，该指标是教学固定值 |
| 22 | 不能，先接齐4字节外壳，再按长度收内容，按通道分流 |
| 23 | 可以正确，RTT扣除了DLSR的等待 |
| 24 | 查RTP到达、PT/时钟/SSRC、缺包、H.264重组/参数集、解码和播放 |

## 19. 编写依据、实测记录和示例边界

### 19.1 具体问题查哪里

**RFC**是公开互联网技术规范，第一遍不必通读，按问题查小节：

- RFC 3550 §6.1～§6.3：复合RTCP、预算和调度；§6.4.1～§6.4.2：SR/RR、丢包、Jitter、LSR/DLSR；§6.5～§6.7：SDES/BYE/APP；附录A.1/A.3/A.8：序号验证、接收统计、抖动算法。
- RFC 4585 §6.1、§6.2.1、§6.3.1：反馈头、NACK、PLI；§3～§4：反馈时序和SDP能力。
- RFC 5104 §4.3.1：FIR的PT/FMT、目标SSRC和命令序号。
- RFC 5506：精简RTCP；RFC 5761：RTP/RTCP复用。
- RFC 2326 §10.12：本章RTSP 1.0交织；RTSP 2.0另见RFC 7826，不能直接混用全部流程。

### 19.2 作者实际验证

验证日期：2026-10-20。代码从本章最终Markdown提取后测试，避免文档和测试副本不同。

补充范围与验证来源记录于 [[2026-10-20-rtcp零基础学习章节补充与windows示例核验]]。

| 项目 | 实际结果 |
| --- | --- |
| 编译 | Windows，MinGW-w64 g++ 15.2.0，C++17；3个程序均使用-Wall -Wextra -pedantic无警告编译通过 |
| 离线报文 | SR复合52字节、RR复合56字节，字段和§14输出逐项一致；坏length被拒绝 |
| 丢包/抖动 | §15的四次快照和四个抖动输出与手算一致 |
| 结构与统计边界 | 524项检查通过，覆盖SR/RR的RC=0～31、截短输入、错误版本/长度、24位符号/限制、合法/非法Padding、SDES条目/END/对齐、多源SDES、1～255字节CNAME、未知类型跳过、RTP时间戳回绕和乱序差值 |
| 真实本机UDP | 启动两个进程，完成SR+SDES→RR+SDES，正确匹配Source SSRC及LSR；本次DLSR约0.214981秒、RTT约0.461921ms，数字随负载变化 |
| UDP错误路径 | 无参数输出用法；端口占用10048、无发送端约5秒超时10060、2字节坏报文拒绝、4096字节超大数据报10040，均实测符合预期 |

**本轮没有实测：**MSVC、Wireshark图形界面、摄像头/MediaMTX、真实RTSP混合TCP流、完整RTCP调度、音画同步播放。前文提供相应学习路径或操作方法，不将其写成已通过测试的结果。

### 19.3 示例的适用边界

解析器支持SR/RR/SDES结构，跳过其他类型正文；未实现所有RTCP类型校验、经典复合合规、认证、成员/冲突管理、完整RTP序号状态、报告调度、反馈处理和播放。

UDP固定地址/SSRC/端口、单次收发、超时退出；来源检查不是安全认证。真实网络需要协商、资源限制和所需认证/加密，例如SRTCP，不能把IP或SSRC当身份凭据。

抖动程序适用于连续同源短时间差；NTP转换针对当前Windows常见Unix纪元实现和现代日期，不处理2036年代解释。真实SR必须用同一时刻的NTP/RTP映射和真实发送计数，RR需真实接收统计。

## 20. 总检查、结业练习与继续阅读

### 20.1 第一关：能通俗解释

- [ ] 我能解释RTSP、SDP、RTP、RTCP职责。
- [ ] 我知道SR、RR、SDES、BYE、APP用途。
- [ ] 我知道NACK/PLI/FIR要同时看PT/FMT和协商。
- [ ] 我知道报告丢包不等于自动重传，SR不等于自动同步播放。

### 20.2 第二关：能读报文和算数

- [ ] 我能独立拆解§7的32字节RR。
- [ ] 我能算子包长度，并遍历完整复合包。
- [ ] 我能区分报告者和被报告者。
- [ ] 我能算累计/区间丢包，解释负累计值。
- [ ] 我能换算Jitter、LSR/DLSR并算RTT。
- [ ] 我能把音视频分别放到共同参考轴。

### 20.3 第三关：能运行和排错

- [ ] 我能从空目录保存4文件，编译运行3程序。
- [ ] 我做过非法长度、负丢包和规则到达实验。
- [ ] 我能完成UDP双向报告并解释自己的RTT。
- [ ] 我能排查超时、端口占用和链接库错误。
- [ ] 我能说明哪些是合成数据、哪些是真实测量。
- [ ] 我知道接入真实流还需协商、源状态、调度和播放时钟。

### 20.4 结业练习：一份RTCP观察报告

先用本章数据，再尝试真实流：

| 栏目 | 填写什么 |
| --- | --- |
| 输入来源 | 合成、本机UDP、真实RTSP；从哪一层起始 |
| 会话配置 | 轨道、时钟、端口/通道、反馈能力 |
| 结构 | 各子包PT、count、length与字节数 |
| 身份 | 报告者、被报告者SSRC及CNAME关系 |
| 指标 | 原始字段、换算公式、最终单位，含丢包/抖动/RTT |
| 同步 | 至少一组SR映射手算，说明音视频关联 |
| 故障 | 三种错误、预期反应与恢复 |
| 证据边界 | 编译命令、实际输出、当前无法判断的事 |

通过标准：数值可复算、单位不混淆，不把合成数据当实测，不根据RTT宣称知道端到端播放延迟。

### 20.5 术语速查

| 术语 | 一句话记忆 |
| --- | --- |
| SR / RR | 发送者 / 接收者报告；SR也能带接收统计 |
| SSRC | RTP来源的32位标识 |
| SDES / CNAME | 源描述 / 标识参与者并关联来源的名称 |
| RC / SC / FMT | count五位在不同包里的不同含义 |
| Compound | 多个RTCP子包首尾相连 |
| NTP | 共同参考时刻的64位表示 |
| Clock Rate | 每秒多少RTP刻度 |
| Jitter | 相对媒体节奏的到达变化估计 |
| LSR / DLSR | 最近SR紧凑时间 / 收到SR后等待多久 |
| RTT | 本端观察的报告往返时间 |
| FCI | 反馈控制信息，格式随PT/FMT变化 |
| NACK / PLI / FIR | 包缺失描述 / 图像丢失提示 / 解码刷新请求 |
| AVPF | 带反馈时序等规则的RTP音视频配置 |
| SRTCP | RTCP加密、认证等保护机制 |

建议继续：[[07-H264与RTP封装]] → [[08-FFmpeg与解码]] → [[10-C++客户端架构]]。复习入口：[[00-RTSP学习总纲]]、[[01-网络与Socket基础]]、[[04-SDP会话描述]]、[[05-RTP协议]]。

规范链接：

- [RFC 3550：RTP/RTCP](https://www.rfc-editor.org/rfc/rfc3550.html)
- [RFC 4585：AVPF、NACK与PLI](https://www.rfc-editor.org/rfc/rfc4585.html)
- [RFC 5104：FIR等编码控制](https://www.rfc-editor.org/rfc/rfc5104.html)
- [RFC 5506：精简RTCP](https://www.rfc-editor.org/rfc/rfc5506.html)
- [RFC 5761：RTP/RTCP复用](https://www.rfc-editor.org/rfc/rfc5761.html)
- [RFC 7022：CNAME建议](https://www.rfc-editor.org/rfc/rfc7022.html)
- [RFC 3611：RTCP XR](https://www.rfc-editor.org/rfc/rfc3611.html)
- [RFC 4588：RTP重传格式](https://www.rfc-editor.org/rfc/rfc4588.html)
- [RFC 3711：SRTP/SRTCP](https://www.rfc-editor.org/rfc/rfc3711.html)
- [RFC 2326：RTSP 1.0](https://www.rfc-editor.org/rfc/rfc2326.html)
- [RFC 7826：RTSP 2.0](https://www.rfc-editor.org/rfc/rfc7826.html)
