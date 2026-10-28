# 头脑风暴总结：Map ESP 实现路径分析

> 日期：2026-02-23  
> 背景：为 LOLM (com.tencent.lolm) 实现小地图 ESP，使敌方英雄在迷雾中可见

## 1. Il2CppDumper 方案 — 已放弃 ❌

### 目标
从内存 dump `libil2cpp.so` + `global-metadata.dat`，交给 `Il2CppDumper.exe` 输出 `script.json` 和 `dump.cs`，获得所有函数 RVA。

### 遇到的困难

| 问题 | 根因 | 解决方式 | 结果 |
|------|------|----------|------|
| `kmalloc(1MB)` 分配失败 | Android 内存碎片，order:8 | 改用 `vmalloc` | ✅ |
| `access_process_vm` 读不了 `r-xp`/`rw-s` | 不用 `FOLL_FORCE` | `get_user_pages_remote + FOLL_FORCE` 回退 | ✅ |
| Maps 解析出错 | `/dev/zero (deleted)` 含空格 | `split(None, 5)` | ✅ |
| Metadata magic 被清零 | ACE 反作弊 | 自动 patch | ✅ |
| Il2CppDumper 报 "protected" | ACE 删除 `.init_array` | 无法绕过 | ❌ |
| 找不到 `CodeRegistration` | 注册函数被混淆 | 模式扫描/指针追踪 | ❌ |
| `EndOfStreamException` | dump 地址重定位不兼容 | — | ❌ |

### 结论
ACE 系统性地破坏了 Il2CppDumper 依赖的所有入口点。**此路径放弃**，但解决技术障碍的过程中改进了内核模块的读取能力（vmalloc、FOLL_FORCE），这些改进对后续方案有价值。

---

## 2. 三条前进路径

### Path 1: 继续利用 IL2CPP 元数据（自己解析）
- 绕开 Il2CppDumper，直接解析 v24 metadata 的 `typeDefinitions`/`methods`/`string` 表
- 通过类名 + 方法名定位 `MethodInfo`，读取 `methodPointer` 字段
- **优势**：原理明确，不依赖第三方工具
- **劣势**：仍受限于 IL2CPP 知识，换游戏/引擎不通用

### Path 2: CE（Cheat Engine）通用逆向思路 ⭐
- 将 CTTCP 完善为 Android 内核版 CE
- 工作流：操作游戏 → 扫描变量变化 → 缩小范围 → HWBP 抓函数 → 分析/修改
- **优势**：完全黑盒，不依赖源码/符号/IL2CPP 知识
- **劣势**：需要大量基础设施建设（多线程、值扫描、前端 UI）

### Path 3: 其他补充方案
- **3a**: HWBP 直接定位 — 找到对象实例，对可见性字段设断点，直接抓函数
- **3b**: 函数 Patch — 找到函数后用 `MOV W0, #1; RET` 覆盖（8 字节）
- **3c**: 网络层截获 — 拦截位置同步数据包（协议加密，难度大）

---

## 3. 战略决策

### Orange Driver 的启示
Orange Driver 的架构验证了 **"通用 CE 后端"** 思路：
- 内核 TCP server 只负责读写内存
- 逻辑全在前端
- 不绑定特定游戏 — 换前端即可对接新游戏

### 最终方向
**建设 CTTCP 为通用 CE 后端 + Path 1 快速验证 + Path 2 长期建设**

优先级：
1. **多线程支持** — 解决单线程瓶颈
2. **值扫描命令** — 内核端原生扫描
3. **Path 1 快速出结果** — 自己解析 metadata 找 3 个目标函数
4. **前端建设** — Python CE 客户端 → 后续 ImGui overlay

---

## 4. CTTCP 需要完善的功能清单

| CE 功能 | 现状 | 优先级 |
|---------|------|--------|
| 读内存 | ✅ 已修复（vmalloc + FOLL_FORCE） | — |
| 写内存 | ✅ | — |
| HWBP (What accesses) | ✅ | — |
| 多连接并发 | ❌ 单线程 | **P0** |
| 值扫描 (first/next scan) | ❌ | **P0** |
| 区域扫描 (region filter) | ❌ | P1 |
| 指针扫描 | ❌ | P2 |
| 前端 UI | ❌ | P2 |
