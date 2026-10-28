# CTTCP 后端 PRD — Android Kernel-Level Cheat Engine

## 1. 前因后果

### 1.1 为什么要做这个？

在 Android 游戏逆向工程中（以 LOLM 为例），常规的用户态工具（Frida、ptrace、process_vm_readv）全部被 ACE 反作弊拦截。我们需要一个 **Ring-0 级别的内存读写后端**，类似于 PC 端的 Cheat Engine，但更隐蔽——以内核模块形式运行，通过 TCP socket 暴露接口。

### 1.2 前世——Orange Driver 的验证

逆向分析 "橘子驱动"（Orange Driver）发现其架构就是此方案的成功实践：
- 内核 TCP server（端口绑定 127.0.0.1）
- `CTTCP` 自定义二进制协议
- 前端 ImGui overlay，每帧通过 TCP 读取游戏内存、绘制 ESP
- **不绑定特定游戏** — 后端通用，前端可替换

### 1.3 今生——我们的 CTTCP

基于 Orange Driver 思路，我们从零实现了 CTTCP 内核模块。当前已实现：
- ✅ Kernel TCP server（端口 12345）
- ✅ 读内存（`access_process_vm` + `FOLL_FORCE` 回退）
- ✅ 写内存
- ✅ 获取 PID / 基地址
- ✅ 硬件断点（HWBP）

### 1.4 遇到的瓶颈

在尝试用 Il2CppDumper 逆向 LOLM 的 IL2CPP 二进制时，发现 **IL2CPP-specific 的工具链在重保护下不可靠**。头脑风暴后决定转向 **通用 CE 思路**——不依赖源码/符号，纯黑盒逆向。这要求后端具备 CE 级别的功能。

---

## 2. 产品定义

### 2.1 是什么

**CTTCP** (ConnecT TCP) 是一个 Android Linux Kernel Module (.ko)，作为通用内存调试器后端运行在 Ring-0。它通过 localhost TCP 暴露标准化的二进制协议，允许任何前端（Python 脚本、ImGui overlay、Web UI）连接并执行内存读写、扫描、断点等操作。

### 2.2 不是什么

- 不是前端 UI — 只是后端 engine
- 不是游戏特定工具 — 不包含任何游戏逻辑
- 不是 rootkit — 不做主动隐藏（隐蔽性通过 TCP 无设备节点天然获得）

---

## 3. 功能需求

### 3.1 现有功能（v1.0 — 已实现）

| CMD | 功能 | 协议 |
|-----|------|------|
| `0x00` PING | 心跳 | 无 payload |
| `0x01` READ_MEM | 读进程内存 | `{pid, addr, size}` → `{data[]}` |
| `0x02` WRITE_MEM | 写进程内存 | `{pid, addr, data}` → `{status}` |
| `0x03` GET_PID | 按名称获取 PID | `{name[64]}` → `{pid}` |
| `0x04` GET_BASE_ADDR | 获取模块基地址 | `{pid, name[64]}` → `{addr, size}` |
| `0x05` SET_HWBP | 设置 / 取消硬件断点 | `{pid, addr, type}` → `{status}` |
| `0x06` BP_WAIT | 等待断点命中挂起 | `{timeout}` → `{status, tid}` |
| `0x07` GET_REGS | 获取挂起线程寄存器上下文 | `{tid}` → `{regs[]}` |
| `0x08` SET_REG | 修改挂起线程寄存器 | `{tid, reg_idx, value}` → `{status}` |
| `0x09` BP_CONT | 恢复挂起线程继续执行 | `{tid, step_mode}` → `{status}` |

### 3.2 V2.0 需求 — CE 核心功能

#### P0: 多连接支持
```
现状: 单线程 accept → process_client 循环
问题: 前一个连接不关闭，新连接永远阻塞
目标: kthread worker pool，支持至少 4 个并发连接
```

#### P0: 值扫描 (Value Scan)
```
CMD: 0x10 SCAN_INIT    — 首次扫描
CMD: 0x11 SCAN_NEXT    — 过滤扫描
CMD: 0x12 SCAN_RESULTS — 获取结果
CMD: 0x13 SCAN_RESET   — 清空结果

首次扫描:
  Input:  {pid, value, value_size(1/2/4/8), scan_type(exact/bigger/smaller/changed/unchanged)}
  Action: 遍历 task_struct -> mm 的 VMA 树结构，筛选所有可读区域 (VM_READ)，匹配值
  Output: {result_count}
  
过滤扫描:
  Input:  {pid, new_value, scan_type}
  Action: 在上次结果集中重新读取并过滤
  Output: {result_count}
  
获取结果:
  Input:  {offset, count}
  Output: {addr[], value[]} (分页返回)
```

#### P1: 内存区域列表
```
CMD: 0x14 LIST_REGIONS
  Input:  {pid}
  Output: {regions[]} — 每个 region: {start, end, perms, name}
```

#### P1: 完整调试器挂起机制 (Full Debugger Suspension)
```
现状: HWBP 仅作为探针 (Probe) 记录 TID，不挂起线程，防止游戏卡死。
问题: 无法像传统调试器或 rwMem 那样修改执行流和上下文。
目标: 
  1. 通过 wait_queue_head_t 将命中断点的目标线程挂起在内核态 (TASK_INTERRUPTIBLE)。
  2. 保存命中现场的 struct pt_regs，供前端通过 GET_REGS / SET_REG 查看和修改。
  3. 支持前端发送 BP_CONT (继续) 和 BP_STEP (单步) 指令唤醒线程。
```

#### P1: 批量读取
```
CMD: 0x15 READ_MULTI
  Input:  {pid, count, addrs[], sizes[]}
  Output: {data_chunks[]}
  用途: 前端一次请求读多个地址，减少 RTT
```

#### P2: 指针扫描
```
CMD: 0x16 POINTER_SCAN
  Input:  {pid, target_addr, max_depth, max_offset}
  Output: {pointer_chains[]}
  用途: 找到从静态基址到动态对象的指针链
```

### 3.3 协议格式（不变）

```
请求: [CTTCP(5)] [CMD(1)] [PAYLOAD_SIZE(4)] [PAYLOAD...]
响应: [CTTCP(5)] [CMD(1)] [STATUS(1)] [PAYLOAD_SIZE(4)] [PAYLOAD...]
```

---

## 4. 技术约束

### 4.1 内核兼容性
- 目标: GKI 6.6 (Android 15)
- 编译: LLVM/Clang, `ARCH=arm64`
- API: 仅使用稳定导出符号（`sock_create_kern`, `access_process_vm`, `get_user_pages_remote` 等）

### 4.2 内存限制
- 单次读写上限: 1MB（`vmalloc` 分配）
- 扫描结果存储: 内核 `vmalloc` 动态数组，上限 1M 个结果
- 超出上限返回 `ERR_RESULT_OVERFLOW`

### 4.3 性能目标
- 单次 READ_MEM (4KB): < 1ms
- 单次 READ_MEM (1MB): < 10ms
- 全内存值扫描 (1GB 可读区域): < 5s
- 过滤扫描 (10K 结果集): < 100ms

---

## 5. 目录结构

```
src/
├── backend/
│   ├── kernel/           # 内核模块源码
│   │   ├── cttcp.h       # 协议定义
│   │   ├── cttcp_main.c  # 模块入口 + TCP server
│   │   ├── cttcp_ops.c   # 内存操作实现
│   │   ├── cttcp_ops.h
│   │   ├── cttcp_parser.c # 协议解析 + 命令分发
│   │   ├── cttcp_parser.h
│   │   └── Makefile
│   └── legacy_c_clients/ # 废弃的 C 客户端（仅参考）
│
├── frontend/
│   ├── cttcp.py          # Python CTTCP 客户端库
│   └── tools/            # 基于 cttcp.py 的工具
│       ├── scan_rva.py   # IL2CPP RVA 扫描器
│       ├── dump_il2cpp.py # 内存 dumper
│       └── diag.py       # 诊断工具
```

---

## 6. V2.0 实现路线图

```
Phase 1: 多线程 (1-2天) ✅ DONE
  → cttcp_main.c: kthread_create 4 个 worker
  → cttcp_parser.c: 增加 sk_rcvtimeo 防死锁与非阻塞 accept
  → 稳定承载并发连接

Phase 2: 真·调试器与断点操作 (2-3天)
  → cttcp_ops.c: 重构 hwbp_handler 加入 WAIT_QUEUE 挂起机制
  → 新增 registers 操作 (GET_REGS / SET_REG) 与执行流控制 (CONT / STEP)
  → cttcp.py: 提供 bp.wait(), bp.get_regs(), bp.cont() 等高级 API

Phase 3: 值扫描 (2-3天)
  → cttcp_ops.c: 新增 op_scan_init / op_scan_next / op_scan_results
  → 内核端直接遍历 VMA 树 + physical pages 快速扫描
  → cttcp.py: 新增 scan_init() / scan_next() / scan_results()

Phase 4: 前端 CE 工具与 UI (后续)
  → frontend/ce.py: 命令行交互式 CE
  → 支持: scan → filter → list → read → write → hwbp

Phase 5: ImGui Overlay (后续)
  → C++ ImGui + EGL 前端
  → 实时 ESP 绘制
```
