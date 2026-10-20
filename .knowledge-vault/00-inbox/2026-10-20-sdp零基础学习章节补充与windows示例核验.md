---
type: session
status: active
created: 2026-10-20
updated: 2026-10-20
tags:
  - session
---

# Session: SDP零基础学习章节补充与Windows示例核验

## Summary

按完全新手需求补充 [[04-SDP会话描述]]，增加 Windows 工具链、最小程序、逐行观察实验、完整解析器阅读步骤、故障练习及阶段验收。以官方 RFC 核对并修复无尾斜杠时无条件追加控制路径的错误。

## Outcomes

- 学习阶段统一为 A～H，章末有 25 道题、参考答案及逐阶段检查清单。
- 从 Markdown 提取三个可运行 C++ 程序，在 Windows MinGW-w64 g++ 15.2.0、C++17 和警告选项下编译成功，无诊断输出。
- 49 组解析与 URL 运行检查通过，另验证 hello 和观察器；22 张表格、162 个表格行、68 个代码围栏通过结构检查。

## Decisions

- 使用规范相对路径合并，区分尾斜杠；厂商追加路径行为单独解释为兼容策略。
- 教学程序限定 RTP/AVP 和 URI 子集；不将解析成功等同于完整标准校验或播放成功。
- 补全常见静态 PT 0/8，检查动态 PT 缺映射、非法数值和关联冲突。

## Knowledge Updated

- [[04-SDP会话描述]]：补充前置知识、Windows 实验及验证记录，修正 H265 标准名称和 Opus 固定 SDP 映射的表述。

## Open Questions

- MSVC、Linux 运行及真实摄像头接入未在本轮实测，留给后续具备相应环境时验证。

## Sources

- `.knowledge-vault/10-topics/04-SDP会话描述.md` 及现有网络、RTSP、RTP、H.264 章节。
- RFC 2326 附录 C、RFC 8866、RFC 7587 §7、RFC 7798 §7 官方正文。
- 系统临时目录 `sdp-chapter-20261020-06f224e0/verify_chapter.py` 与 `verification.json`；临时产物不属于正式项目代码。
