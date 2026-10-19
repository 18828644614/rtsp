# Task Plan: 完善 SDP 会话描述学习章节

## Goal
把 10-topics/04-SDP会话描述.md 扩写成面向 C++ 初学者的完整 SDP 学习教程，并通过可运行示例、练习和检查清单帮助读者独立分析 RTSP DESCRIBE 响应。

## Phases
- [x] Phase 1: 检查知识库规范、现有章节与相关链接
- [x] Phase 2: 规划学习范围、依赖顺序与示例素材
- [x] Phase 3: 编写并覆盖 SDP 主题章节
- [x] Phase 4: 结构检查、链接检查与知识库 refresh/check

## Key Questions
1. 新手先掌握哪些网络、RTSP、RTP 和编码基础，才能读懂 SDP？
2. 如何把字段解释、URL 解析和 C++ 代码串成一个可运行的学习路径？
3. Windows 与 Linux 命令和运行方式如何分别说明？

## Decisions Made
- 以 RFC 8866（替代 RFC 4566）的通用 SDP 语法为主线，按 RTSP 1.0 附录 C 解释 control、端口 0 和基准 URL，再连接 RTP/AVP、H.264/H.265、AAC、G.711 和 Opus。
- 示例优先使用标准库 C++17，避免引入第三方依赖，降低新手运行门槛。
- 用一个完整的 RTSP DESCRIBE 示例贯穿“读取、解析、生成 SETUP URL、检查错误”。

## Errors Encountered
- 本轮最初在项目根目录读取知识库内的 task_plan.md/notes.md，已改用知识库绝对路径。
- 本机 WSL 的 Ubuntu 无法启动（HCS_E_HYPERV_NOT_INSTALLED）；可用 Windows MinGW 验证 C++，Linux 命令保留为读者操作步骤，不声称本机已执行。
- 初次读取 using-superpowers 的旧路径不存在，已根据技能目录搜索到有效路径并读取。
- 初次尝试读取某个不存在的旧章节文件名，已根据实际文件列表修正。

## Status
**Completed** - 章节已写入并通过 Windows MinGW 编译运行、错误输入检查和知识库链接检查。
