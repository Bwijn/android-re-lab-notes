# 🔍 LOLM Anti-Cheat Probing History & Technical Pivot Summary

**Date**: 2026-02-21
**Objective**: Identify specific threads accessing `IsActorAlwaysVisibleInMiniMap` (`0x63D6208` in `libil2cpp.so`) to differentiate between Rendering and Anti-Cheat (ACE) logic.

---

## 1. Phase 1: Userspace Probing (Failed)
### Zygisk-Frida Attempt
*   **Method**: Injecting `frida-gadget` into `com.tencent.lolm` using the Zygisk-Frida module.
*   **Symptom**: Game immediate black screens on launch.
*   **Analysis**: 
    1.  **SELinux**: Resetting to `Permissive` helped with basic crashes, but not the black screen.
    2.  **ACE Checksum**: Tencent ACE detects the automatic injection of `.so` files by Zygisk during the process `fork` from Zygote.
    3.  **Process Suspension**: ACE detects the modified memory/maps and intentionally deadlocks the main thread, preventing the Splash screen from loading.
*   **Conclusion**: Any UserMode injection (even Zygisk) is currently detected by ACE's early initialization check.

---

## 2. Phase 2: Kernel HWBP Probing (Plan A - Implemented)
### Hardware Breakpoint Strategy
*   **Method**: Use ARM64 Hardware Breakpoint registers (`DBGBVR`/`DBGBCR`) via the Linux Kernel to monitor memory access without modifying any bytes in the target process.
*   **Implementation**: Developed `cttcp.ko` kernel module providing a command-based protocol (`CTTCP`).

### Challenges & Troubleshooting
1.  **PID Lookup Limitation**: 
    *   *Issue*: `CTTCP_CMD_GET_PID` failed to find `com.tencent.lolm`.
    *   *Cause*: The Linux Kernel truncates `task->comm` to 15 characters, so it was actually `com.tencent.lol`.
    *   *Fix*: Implemented substring matching (`strstr`) in the kernel-side PID lookup.
2.  **System-Wide HWBP Failure**:
    *   *Issue*: `register_wide_hw_breakpoint` returned `-EINVAL`.
    *   *Cause*: System-wide hardware breakpoints in Linux are generally restricted to Kernel addresses. User-space addresses require thread-context breakpoints.
3.  **Per-Thread Attachment**:
    *   *Method*: Refactored code to use `perf_event_create_kernel_counter` attached to every `task_struct` (thread) within the target process pool.
    *   *Result*: Successfully attached to ~140 threads concurrently.

---

## 3. Phase 3: Current Status & Refinement
*   **Breakpoint Type**: Switched from `HW_BREAKPOINT_X` (Execute) to `HW_BREAKPOINT_R | HW_BREAKPOINT_W` (Read/Write) to catch scanners.
*   **Sampling Mode**: Set `sample_period = 1` and called `perf_event_enable`. Implemented an immediate `perf_event_disable` inside the handler to prevent deadlocking the `UnityMain` rendering thread.
*   **Extended Verification (60 Seconds)**:
    *   *Action*: Executed the Read/Write data probe for a full 60 seconds while the player was actively navigating the map with visible enemy units.
    *   *Result*: **0 Hits recorded for Read/Write.** The kernel attached to 195+ threads, but the specific address was never scanned for modifications.
*   **Final Conclusion**: Tencent ACE does *not* perform active, continuous memory integrity checks (CRC) on `IsActorAlwaysVisibleInMiniMap` (`0x63D6208`) in `libil2cpp.so` during active gameplay.

## 4. Next Phase: Stealth Patching
Since ACE does not scan the memory region:
1.  **Objective**: Directly overwrite `0x63D6208` to return `true` (e.g., `MOV W0, #1; RET` -> `20008052 c0035fd6`) from Ring-0.
2.  **Safety**: Bypasses any filesystem or userspace injection checks, avoiding the Zygisk/Frida detection traps.
3.  **Method**: Implement `op_write_mem` or a dedicated `op_stealth_patch` inside `cttcp_ops.c` utilizing `access_process_vm` to bypass read-only page restrictions and safely modify the instructions.

---
*Archived in `artifacts/report/PROBE_HISTORY.md`*
