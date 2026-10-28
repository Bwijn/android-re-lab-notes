# CTTCP 进程间通信协议 (Orange Driver Protocol)

本规范定义了前端分析工具（如使用 C++ ImGui 构建的外挂前端）与后端 Android 内核态驱动模块（Orange Driver 衍生的 TCP Sever `.ko`）之间的 **标准化二进制通信协议**。

## 1. 协议综述
*   **通信方式**：标准的本地回环 TCP Socket。内核驱动作为 Server 监听 `127.0.0.1:12345`。
*   **数据格式**：纯二进制 `struct`（大端/小端依据系统默认，由于全部在同一台设备的 ARM64 处理器上通信，统一采用**小端序 Little-Endian**）。
*   **对齐要求**：所有的网络数据包结构体（Request / Response）都必须使用 `#pragma pack(push, 1)` 进行单字节内存紧凑对齐，防止因编译器填充（Padding）导致的数据错位。

---

## 2. 基础数据包头部 (Header)

每一次完整的请求/响应都固定由 **[Header] + [Payload]** 组成。

### 请求头结构 (Frontend -> Kernel)
| 字段名称 | 大小 (Bytes) | 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| **`magic`** | 5 | `char[5]` | 固定魔法校验字符串，必须等于 `"CTTCP"`。 |
| **`cmd`** | 1 | `uint8_t` | 指令号 (Command ID)。详情见下表。 |
| **`payload_size`** | 4 | `uint32_t`| 紧随请求头之后的附加负载的字节数。无负载时为 0。 |

### 响应头结构 (Kernel -> Frontend)
| 字段名称 | 大小 (Bytes) | 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| **`magic`** | 5 | `char[5]` | 返回校验字符串 `"CTTCP"`。 |
| **`cmd`** | 1 | `uint8_t` | 原样返回请求方的指令号。 |
| **`status`** | 1 | `uint8_t` | 处理状态码。`0` 表示成功；`>0` 表示发生错误。 |
| **`payload_size`** | 4 | `uint32_t`| 紧随头部返回的数据长度。 |

#### 状态码 (Status Codes):
*   `0x00` (SUCCESS) - 指令成功执行。
*   `0x01` (ERR_GENERIC) - 常规内核故障。
*   `0x02` (ERR_NOT_FOUND) - 进程(PID)或模块(lib)未找到。
*   `0x03` (ERR_MEM_FAULT) - 物理内存 / 虚拟内存读取失败 (非法地址)。
*   `0x04` (ERR_INVALID_CMD) - 后端无法解析请求指令。

---

## 3. 指令集与载荷 (Command & Payload)

### 0x00: 回声测试 (PING)
用于 Frontend 和 Kernel 握手，保持 TCP 连接不被断开。
*   **请求 Payload**：无 (`payload_size = 0`)
*   **响应 Payload**：无 (`payload_size = 0`)，只要 `status == 0` 则代表后端存活。

### 0x01: 跨进程读内存 (READ_MEM)
前端向内核索要目标进程的内存数据。由于在内核态读取（`copy_from_user` 等机制），可以无视被调试进程自身的反调试保护（如 ptrace 屏蔽）。
*   **请求 Payload (`cttcp_req_payload_read_t`)**:
    *   `pid` (`uint32_t`): 目标进程的 ID。
    *   `address` (`uint64_t`): 要读取内存块的起始虚拟地址。
    *   `size` (`uint64_t`): 读取长度。
*   **响应 Payload**: 直接返回提取出的 **原始二进制 Byte 数组**（长度即为请求的 `size`）。

### 0x02: 跨进程写内存 (WRITE_MEM)
用于前端向目标进程内存区域注入/覆盖数据。
*   **请求 Payload (`cttcp_req_payload_write_t`)**:
    *   `pid` (`uint32_t`): 目标进程 ID。
    *   `address` (`uint64_t`): 要写入的起始地址。
    *   `size` (`uint64_t`): 将要写入的独立字节长度。
    *   **紧接着结构体之后发送：** `size` 长度的真实二进制写入数据 (`Raw Bytes`)。
*   **响应 Payload**: 无。以 `status` 字段结果判断写入是否成功。

### 0x03: 获取目标进程 ID (GET_PID)
*   **请求 Payload (`cttcp_req_payload_get_pid_t`)**:
    *   `process_name` (`char[64]`): 进程名称（如 `com.app.game`）。包含 C字符串的空终止符 `\0`。
*   **响应 Payload (`cttcp_resp_payload_get_pid_t`)**:
    *   `pid` (`uint32_t`): 获取到的实际 PID。

### 0x04: 获取动态链接库基址 (GET_BASE_ADDR)
用于外挂前端获取诸如 `libUE4.so`、`libunity.so` 或者 `libil2cpp.so` 的内存基址，从而定位偏移。
*   **请求 Payload (`cttcp_req_payload_get_base_t`)**:
    *   `pid` (`uint32_t`): 目标进程 ID。
    *   `module_name` (`char[64]`): 在 `/proc/$pid/maps` 中对应的 `.so` 的名称。
*   **响应 Payload (`cttcp_resp_payload_get_base_t`)**:
    *   `base_address` (`uint64_t`): 解析出的虚拟内存基址。
    *   `module_size` (`uint64_t`): 该共享库占用内存空间的总大小。

---

## 4. 示例代码
请直接在 C/C++ 全局引用 `src/cttcp.h`，利用其中的 `#pragma pack` 和枚举定义即可无差错开发。

### 0x05 / 0x06: [规划中] 隐蔽读写 (PTE_READ / PTE_WRITE)
*   **背景**: 常规的 `copy_from_user`/`copy_to_user` 仅仅是内核提供的数据拷贝接口。如果目标游戏（如带有强反作弊机制的 FPS）在 UserMode 层面（如通过额外的扫描线程）对其关键内存区域进行了完整的 CRC 校验或严格的权限监控 (`mprotect` 锁定只读)，常规的内核 `WRITE_MEM` 将导致目标进程崩溃或因 CRC 变更被封号。
*   **目标方案**: 
    1.  **Shadow PTE (影子页表)**: 在内核态拦截物理页映射，将原始的一份只读物理页复制一份可写的，修改这部分可写的 Shadow Page，并在访问时将 TLB 指向这部分修改后的页面。这样能够从物理内存级别完美绕过基于页表属性和虚拟内存区域的 CRC 校验。
    2.  **HWBP (硬件断点 / 异常拦截)**: 通过修改调试寄存器 (`DR0-DR3`)，拦截执行流。
*   **状态**: 定义中，目前暂未实现实际代码。

### 0x07: [已挂起] 触摸事件注入 (TOUCH_EVENT)
*   留作后续外设模拟（如压枪、自瞄准星拖动）的预留接口。
*   **状态**: 暂不实现（Stub）。

