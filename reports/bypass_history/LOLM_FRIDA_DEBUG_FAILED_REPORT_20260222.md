# LOLM (英雄联盟手游) 动态调试环境对抗实验总结 (2026-02-22)

**日期**: 2026年2月22日
**目标包名**: `com.tencent.lolm`
**核心任务**: 在 KernelSU + Zygisk Next 环境下重新建立基于 `Zygisk-Frida` 的动态调试链路。
**最终结论**: **调试方案废弃 (PASSED)**。现阶段该方案在不配合内核级隐藏模块的情况下已达到对抗瓶颈，建议切换回内核级 (LKM) 或硬件调试。

---

## 1. 实验环境 (Experiment Environment)
*   **Device**: Android 16 (SDK 36).
*   **Hardware**: Snapdragon 8 Elite (SM8750), 12GB RAM.
*   **Kernel**: 6.6.89-android15.
*   **Root**: KernelSU 3.1.0 (Next generation).
*   **Injection**: Zygisk Next + Zygisk-Frida (v1.9.0).
*   **Gadget Version**: Frida-Gadget 16.x.x。

---

## 2. 关键实验记录与对抗表现

### 2.1 注入时机测试 (Start-up Delay)
*   **3000ms**: 注入成功，UI 加载后立即卡死。
    *   *分析*: 此时 Unity/腾讯引擎已初始化渲染，但 Frida 的符号扫描或 Hook 逻辑触发了 ACE 的实时监控。
*   **5000ms - 7000ms**: 注入表现为“静默失败”。
    *   *分析*: 虽然能进入游戏，但 `netstat` 无法观测到端口，内存 `maps` 中无相关 `.so` 映射。可能是由于延迟过长，系统已锁定链接器命名空间。

### 2.2 二进制特征混淆 (Binary Patching)
*   **操作**: 使用随机文件名（如 `libi1bjtcxl.so`），并将二进制内所有 `frida` / `Frida` 字符串替换为 `txace` / `TxAce`。
*   **结果**: **无效**。ACE 依然能够通过代码流指纹（Code Fingerprinting）识别 Frida 核心引擎。

### 2.3 Zygisk Next 模式对抗
*   **Disabled 模式**: 游戏开启即黑屏。
    *   *原因*: Zygisk Next 在禁用强制模式时可能留下了可被 ACE 检测的内核/进程间通信特征。
*   **Enforced 模式**: 游戏正常进入，但注入被拦截。
    *   *原因*: 系统强制执行 SELinux 策略，`untrusted_app` 域无法加载位于 `/data/local/tmp` 的库文件。即便修正 `chcon` 标签，依然受限于挂载空间隔离。

---

## 3. 失败原因归纳 (Root Cause Analysis)
1.  **特征点过载**: 官方 `frida-gadget` 拥有太多无法通过简单字符串替换抹除的特征（导出符号、内联 Hook 指纹、特定的系统调用序列）。
2.  **SELinux 强对抗**: 在 KernelSU 的 Enforced 模式下，非标准的动态库注入极易触发 AVC Denied。
3.  **ACE 运行时监控**: 腾讯反作弊系统在 2.13 报告后的更新中，显著增强了对 Zygisk 派生线程的扫描强度，任何非合法的 `r-xp` 段都会导致主线程挂起。

---

*Report archived by opencode. Final status: DEBUGGING PATH ABANDONED.*
