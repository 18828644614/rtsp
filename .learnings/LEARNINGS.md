# Learnings

## [LRN-20261018-MDF] correction

**Logged**: 2026-10-18
**Priority**: medium
**Status**: resolved
**Area**: docs

### Summary
Validate Markdown tables with a table-enabled parser before claiming formatting is verified.

### Details
The user reported seeing pipe-delimited source in the RTSP chapter. The previous check only counted fences and checked wiki links. A subsequent markdown-it-py parse recognized all 12 tables, including section 1.2. The reported display issue was not reproduced by the parser; Obsidian's saved state indicated live preview, which can expose source while editing. Do not claim that delimiter spacing fixes a rendering failure or that the user's current UI has been inspected.

### Suggested Action
Check table tokens, column counts, surrounding blank lines, and fence boundaries. Distinguish valid source from editor display modes. Protect C++ types such as std::vector<char> with inline code so HTML parsing does not hide template arguments.

### Metadata
- Source: user_feedback
- Related Files: .knowledge-vault/10-topics/03-RTSP协议.md
- Tags: markdown, tables, obsidian, validation

### Resolution
- **Resolved**: 2026-10-18
- **Notes**: Parsed all 12 tables successfully; normalized separators, protected C++ declarations, and corrected four network concepts being described as three.
