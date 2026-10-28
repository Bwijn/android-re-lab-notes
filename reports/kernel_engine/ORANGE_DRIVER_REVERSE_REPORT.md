# 橘子驱动 (Orange Driver) 逆向分析报告 V2

## 0. 核心发现 — 通信机制

> [!IMPORTANT]
> **橘子驱动不使用 ioctl，也不注册 `/dev/` 设备节点。** 它的 kernel driver 在内核空间**创建了一个 TCP Server Socket**，监听 `127.0.0.1` 的某个端口。前端 (ImGui 绘制进程) 通过普通的 `connect()` 连接到这个内核 TCP 服务，实现持续双向通信。

**关键证据链：**

| 证据 | 来源 | 说明 |
|------|------|------|
| `Connected to server at %s:%d` | 前端 binary (V3.22.sh) | 前端连接到 server (driver) 的日志 |
| `Server closed connection.` | 前端 binary | 连接断开时的日志 |
| `CTTCP` | 两端binary 均存在 | 推测为 "ConnecT TCP"缩写 — 自定义协议标识 |
| `127.0.0.1` | 两端binary | 通信局限在 localhost |
| 无 `misc_register` / `register_chrdev` / `cdev_add` | pure_binary strings 搜索 | 没有任何 ioctl 设备注册痕迹 |
| `内核链接完毕` / `内核已链接完毕` | 用户截图 | 前端显示 "内核链接完毕"，即建立 TCP 连接成功 |

---

## 1. 总体架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Android 设备                           │
│                                                             │
│  ┌──────────────────────┐      ┌──────────────────────────┐ │
│  │   Frontend (HPX_UI)  │      │   Backend (pure_binary)  │ │
│  │   辅助文件夹/V3.22   │      │   内核文件夹/V4.12       │ │
│  │                      │      │                          │ │
│  │ • Dear ImGui 1.90.6  │      │ • libcurl 8.7.1          │ │
│  │ • EGL + OpenGL ES    │      │ • OpenSSL 1.1.1w         │ │
│  │ • SurfaceFlinger     │      │ • SQLite (本地存储)      │ │
│  │   TrustedOverlay     │      │ • 版本检查 + 验证        │ │
│  │ • /dev/uinput        │      │ • 解密 .ko → insmod      │ │
│  │   (内核自瞄触摸注入) │      │                          │ │
│  └──────────┬───────────┘      └───────────┬──────────────┘ │
│             │ connect()                    │ finit_module   │
│             │ TCP 127.0.0.1:PORT           │ / init_module  │
│             ▼                              ▼                │
│  ┌─────────────────────────────────────────────────────────┐│
│  │         OrangeKernel Module V5.0 (.ko)                  ││
│  │                                                         ││
│  │  ┌─────────────────┐  ┌───────────────┐  ┌───────────┐ ││
│  │  │ Kernel TCP Srv  │  │ Memory R/W    │  │ Kernel    │ ││
│  │  │ sock_create()   │  │ Engine        │  │ Aimbot    │ ││
│  │  │ kernel_bind()   │  │               │  │ (input    │ ││
│  │  │ kernel_listen() │◄─┤ page_table    │  │  event)   │ ││
│  │  │ kernel_accept() │  │ walk / direct │  │           │ ││
│  │  │                 │  │ phys read     │  │           │ ││
│  │  │ kthread 持续    │  │               │  │           │ ││
│  │  │ 接收/发送数据   │  │ 任意PID/进程  │  │           │ ││
│  │  └─────────────────┘  └───────────────┘  └───────────┘ ││
│  │                    Kernel Space (Ring 0)                 ││
│  └─────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                          ▲
                          │ 云端下载 .ko
                          │ (gitee.com/xiyueYA/xydriver)
                          ▼
                 ┌─────────────────┐
                 │  Cloud Server   │
                 │  19+ 预编译 .ko │
                 │  按内核版本适配 │
                 └─────────────────┘
```

---

## 2. 为什么用 Kernel TCP Socket？（回答你的问题）

### 2.1 ioctl 的困境

传统内核驱动通过 `misc_register()` 或 `register_chrdev()` 创建 `/dev/xxx` 设备节点，然后用户态通过 `ioctl()` 交互。但这有**严重问题**：

1. **需要注册设备** → 在 `/dev/` 下留下痕迹，反作弊轻松扫描
2. **需要 CFI 兼容的函数指针** → `file_operations` 结构的 `.unlocked_ioctl` 必须通过 CFI 校验
3. **GKI 2.0 下无法导出 `misc_register` 等符号** → 除非你能解析 `kallsyms`
4. **单次 ioctl = 一次请求/响应** → 不适合持续的数据流（如每帧读取游戏内存）

### 2.2 Kernel TCP Socket 的优势

内核提供了完整的 Socket API 供内核模块使用：

```c
// 在内核中创建 TCP 服务 — 伪代码逆向推测
static struct socket *server_sock;

int orange_server_init(void) {
    struct sockaddr_in addr;
    
    // 1. 创建 socket
    sock_create_kern(&init_net, AF_INET, SOCK_STREAM, IPPROTO_TCP, &server_sock);
    
    // 2. 绑定 127.0.0.1:PORT
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_LOOPBACK);
    addr.sin_port = htons(ORANGE_PORT);
    kernel_bind(server_sock, (struct sockaddr *)&addr, sizeof(addr));
    
    // 3. 监听
    kernel_listen(server_sock, 1);
    
    // 4. 在 kthread 中 accept 客户端
    kthread_run(orange_accept_thread, NULL, "kworker/orange");
    return 0;
}
```

**优势分析：**

| 特性 | ioctl 方案 | Kernel TCP 方案 |
|------|-----------|-----------------|
| 设备节点痕迹 | `/dev/xxx` 可被扫描 | **无** — 不注册任何设备 |
| CFI 兼容性 | 需要合法 `file_operations` | **不需要** — `sock_create` 等是稳定 API |
| 数据流类型 | 请求-响应（同步） | **双向流** — 持续收发 |
| 反检测 | 容易被发现 | 极难发现 — 内核TCP连接不会出现在 `/proc/net/tcp` 中（除非专门查） |
| Turn on/off | 需要额外状态管理 | **断开连接即关闭** — 重连即开启 |
| 多前端支持 | 每个需要独立 fd | **天然支持** — accept 多个客户端 |

### 2.3 通信协议推测

基于字符串 `CTTCP` 和前端功能，推测使用自定义二进制协议：

```
┌──────────┬──────────┬──────────────────────┐
│ Magic    │ CMD Type │ Payload              │
│ "CTTCP"  │ uint8    │ variable length      │
│ 5 bytes  │ 1 byte   │                      │
└──────────┴──────────┴──────────────────────┘

CMD Types (推测):
  0x01 = READ_MEMORY   { pid, addr, size }  → { data[] }
  0x02 = WRITE_MEMORY  { pid, addr, data }  → { status }
  0x03 = GET_PROCESS   { name_hash }        → { pid, base_addr }
  0x04 = KERNEL_TOUCH  { x, y, action }     → { status }
  0x05 = HEARTBEAT     { }                  → { }
```

**前端每帧流程：**
1. 发送 `READ_MEMORY` → 读取游戏进程的 World/玩家列表/坐标
2. 内核 driver 通过页表遍历直接读取目标进程物理内存
3. 通过 TCP 回传数据
4. ImGui 绘制 ESP/骨骼/雷达
5. 如需自瞄 → 发送 `KERNEL_TOUCH` → 内核 driver 通过 input subsystem 注入触摸事件

---

## 3. Backend (内核文件夹) 详解

### 3.1 加载流程

```
V4.12.sh (shell脚本)
    │
    ▼ sed 导出 → /data/data/HPX
pure_binary (HPX)
    │
    ├─ 1. 读取 /proc/version 获取内核版本
    ├─ 2. uname 获取设备信息
    ├─ 3. 连接 gitee.com/xiyueYA/xydriver/raw/main/{version}.ko
    │     (OrangeCurl — 自定义 libcurl wrapper)
    ├─ 4. 下载加密的 .ko 文件到内存
    ├─ 5. OpenSSL 解密 → 还原真实 ELF .ko
    ├─ 6. finit_module() / init_module() 系统调用加载
    ├─ 7. 监控游戏进程 PID（pidof com.tencent.tmgp.pubgmhd）
    └─ 8. 打印 "内核已链接完毕"
```

### 3.2 云端适配库

`pure_binary` 中硬编码了 **19+ 个 gitee 下载链接**，覆盖几乎所有主流内核版本：

| URL 后缀 | 对应内核版本/设备 |
|-----------|-----------------|
| `414180.ko` | 4.14.180 |
| `4141861.ko` ~ `4141864.ko` | 4.14.186 变体 (Note8Pro/K30Ultra/Note11Pro/K40游戏增强) |
| `4191571.ko` / `4191572.ko` | 4.19.157 (K40/K10/K40S) |
| `51.ko` / `515.ko` / `54.ko` | 5.x 系列 |
| `61.ko` / `612.ko` / `66.ko` | 6.x 系列 |
| `49186.ko` / `41981.ko` | 4.x 其他变体 |

每个 `.ko` 的 CRC (`__versions`) 精确匹配目标设备的 `Module.symvers`，直接通过内核校验。

### 3.3 加密方案

`orange_66.ko` 文件头：
```
Offset  Hex                                      ASCII
0x0000  4f 72 61 6e 67 65 4b 65 72 6e 65 6c 20   OrangeKernel 
0x000D  4d 6f 64 75 6c 65 20 56 35 2e 30         Module V5.0
0x0020  43 6f 70 79 52 69 67 68 74 20 32 30 32 35 CopyRight 2025
0x0030  41 75 74 68 6f 72 3a 20 58 69 59 75 65    Author: XiYue
0x0040  54 65 6c 65 67 72 61 6d 20 3a 20 ...      Telegram : https://t.me/ORANGEFRE
0x0059  84 92 f3 9d 05 18 9a 9a ...               [加密数据开始]
```

- 前 ~0x59 字节为明文 header（版本、作者、Telegram）
- 之后为加密的 ELF 内容
- `HPX` 读取到内存后解密，**不落盘** → 内存中直接 `init_module()` 加载

---

## 4. Frontend (辅助文件夹) 详解

### 4.1 打包方式

`(橘子公益和平)V3.22.sh` 与 backend 相同结构：
```bash
# Shell 脚本头部
sed "1,/^# END OF THE SCRIPT/d" "$0" > /data/data/HPX
chmod u+x /data/data/HPX
nohup /data/data/HPX > /data/data/out 2>&1 &
# END OF THE SCRIPT ---------->
[ELF 二进制数据]
```

### 4.2 绘制方案

| 组件 | 技术 |
|------|------|
| UI 框架 | **Dear ImGui 1.90.6** |
| 渲染后端 | **EGL + OpenGL ES** (`imgui_impl_opengl3`) |
| 窗口管理 | `SurfaceComposerClient` → `CreateSurface` → `setTrustedOverlay(true)` |
| 窗口属性 | `SetLayer` (置顶) + 透明 |
| 原生窗口 | `ANativeWindow_acquire` / `setBuffersGeometry` |

> **`setTrustedOverlay`** 是关键：这是 Android SurfaceFlinger 的特权 API，标记 Surface 为"可信覆盖层"，使其不会被游戏的反录屏检测捕获，也不会出现在截图中。需要 ROOT 权限。

### 4.3 自瞄与触摸

```
/data/data/NoKernelTouch    ← 存在此文件则禁用内核自瞄
/data/data/NoScreenRecording ← 存在此文件则禁用录屏保护

内核自瞄: 
  → Driver 通过内核 input subsystem 直接注入触摸事件
  → 不经过用户态 → 反作弊无法通过 /proc 追踪

普通触摸自瞄:
  → 前端通过 /dev/uinput 创建虚拟触摸设备
  → 注入 input_event → 可能被检测到触摸设备痕迹
```

---

## 5. 回答三个核心问题

### Q1: 如何在 kernel 层安插内存读取器并 turn on/off？

**答：** Kernel module 启动后创建 kthread 运行 TCP Server，等待前端 connect。
- **Turn ON** = 前端 `connect(127.0.0.1, PORT)` → 内核 `kernel_accept()` → 开始服务
- **Turn OFF** = 前端断开 TCP 连接 → 内核检测到 socket 关闭 → 停止读写
- **内存读取**：driver 直接在 Ring 0 通过页表遍历 (`find_vma` / `__get_user_pages` 或直接 walk page table) 读取任意进程的虚拟内存，完全绕过 `process_vm_readv` 等用户态 API

### Q2: 没有 CFI 怎么注册 ioctl？

**答：** 它**根本不用 ioctl**。不注册任何字符设备、不创建 `/dev/` 节点、不需要 `file_operations` 结构体。
- `sock_create_kern()` / `kernel_bind()` / `kernel_listen()` 等是内核稳定导出的 API
- 不涉及 CFI 保护的函数指针间接调用
- 这也是为什么它不需要 `raw_kcall` 那样的 ASM stub — **正规的 .ko 模块可以直接调用这些内核函数**

### Q3: 如何做到通用、不限进程/游戏？

**答：** 
- Kernel TCP Server 接收前端发来的 `{pid, address, size}` 指令
- Driver 不绑定特定进程 — 任何前端告诉它读谁它就读谁
- 前端是可替换的：只要遵循相同的 TCP 协议 (`CTTCP` magic + 命令格式)，任何游戏的前端都能对接同一个 driver
- 这就是为什么 "向 orange 作者申请对接" = 获取 TCP 协议文档 + 通信 SDK

---

## 6. 反检测策略总结

| 层面 | 策略 |
|------|------|
| 模块加载 | 内存解密 + `init_module()` 直接注入，不落盘 |
| 通信通道 | Kernel TCP Socket — 无 `/dev/` 痕迹、无 ioctl 痕迹 |
| 绘制层 | `setTrustedOverlay` — 不可截屏、不可录屏 |
| 触摸注入 | 内核 input subsystem — 无用户态 `/dev/uinput` 痕迹 |
| 进程伪装 | 随机 6 字符进程名（`tr -dc 'a-z' < /dev/urandom | head -c 6`） |
| 内存读取 | Ring 0 页表遍历 — 不触发 `ptrace` / `process_vm_readv` 检测 |
| 云端 | 加密 .ko 文件、HTTPS 下载、版本检查 |

---

## 7. 文件清单

| 文件 | 大小 | 角色 |
|------|------|------|
| `pure_binary` | 5.4MB | Backend 用户态 Loader (ARM64 ELF, libcurl+OpenSSL+SQLite) |
| `orange_66.ko` | 366KB | 加密的内核模块 (V5.0, 6.6 内核版本) |
| `extracted_driver.ko` | 5.0MB | 解密/提取后的内核模块（含重定位表） |
| `V4.12.sh` | 5.4MB | Backend 自解压脚本 (shell + 尾部嵌入 pure_binary) |
| `V3.22.sh` | 12MB | Frontend 自解压脚本 (shell + 尾部嵌入 ImGui 绘制程序) |

> **作者**: XiYue (曦月)  
> **Telegram**: [@orangefre](https://t.me/ORANGEFRE)  
> **云端仓库**: `gitee.com/xiyueYA/xydriver`
