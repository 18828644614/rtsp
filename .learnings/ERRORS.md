
## [ERR-20261019-SDP] documentation validation environment

**Logged**: 2026-10-19T12:00:00+08:00
**Priority**: low
**Status**: resolved
**Area**: docs

### Summary
Read paths must be anchored to the vault; local WSL cannot execute Linux validation.

### Error
Get-Content could not find task_plan.md under D:\learn\rtsp. WSL reported HCS_E_HYPERV_NOT_INSTALLED.

### Context
The current vault is D:\learn\rtsp\.knowledge-vault. Windows MinGW g++ is installed; Ubuntu is registered but its virtual machine cannot start.

### Suggested Fix
Use absolute paths for reads and verify portable C++ with MinGW. Document Linux commands separately and state that Linux execution was not validated on this host. Do not change host virtualization settings for a documentation edit.

### Metadata
- Reproducible: yes
- Related Files: .knowledge-vault/10-topics/04-SDP会话描述.md

### Resolution
- **Resolved**: 2026-10-19
- **Notes**: Corrected paths and selected Windows compilation for executable verification.

## [ERR-20261020-H264PATCH] duplicate file operations in apply_patch

**Logged**: 2026-10-20
**Priority**: low
**Status**: resolved
**Area**: docs

### Summary
A patch containing Delete File and Add File for the same path was rejected before any file changes.

### Error
apply_patch verification failed: invalid patch: multiple operations target the same path.

### Suggested Fix
Use Update File with replacement hunks when rewriting an existing note. Keep one operation per path in each patch.

### Metadata
- Reproducible: yes
- Related Files: .knowledge-vault/10-topics/07-H264与RTP封装.md

### Resolution
- **Resolved**: 2026-10-20
- **Notes**: Replaced the rejected patch with Update File; chapter was written and executable examples validated.
