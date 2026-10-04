# termux-xmrig 深度技术原理分析

> **项目地址**：https://github.com/TokiZeng/termux-xmrig
> **分析日期**：2026-10-04
> **文档版本**：v2.0 — 运行实现深度补充版

---

## 目录

- [一、项目概述](#一项目概述)
- [二、构建脚本实现原理](#二构建脚本实现原理)
- [三、Termux 编译环境深度解析](#三termux-编译环境深度解析)
- [四、XMRig 启动与初始化流程](#四xmrig-启动与初始化流程)
- [五、Stratum 协议通信实现](#五stratum-协议通信实现)
- [六、RandomX 算法执行原理](#六randomx-算法执行原理)
- [七、JIT 即时编译实现](#七jit-即时编译实现)
- [八、CPU 后端线程模型](#八cpu-后端线程模型)
- [九、虚拟内存管理](#九虚拟内存管理)
- [十、完整运行时数据流](#十完整运行时数据流)
- [十一、技术总结与思考](#十一技术总结与思考)

---

## 一、项目概述

### 1.1 项目本质

`termux-xmrig` 是一个**极简的自动化构建封装项目**，而非对 XMRig 源码的分叉或修改。它通过一个 Shell 脚本将 XMRig 在 Android Termux 环境中的编译和部署过程自动化，让普通用户也能在手机上运行 XMRig 挖矿程序。

整个仓库仅包含 3 个文件：

```
termux-xmrig/
├── README.md      # 双语使用说明（5,385 bytes）
├── build.sh       # 核心构建脚本（629 bytes）
└── start          # 启动脚本模板（241 bytes）
```

### 1.2 核心设计思想

- **零侵入**：不修改 XMRig 上游源码，不维护独立分支
- **极简封装**：仅通过一个 CMake 选项（`-DWITH_HWLOC=OFF`）适配 Termux 环境
- **一键体验**：将复杂的编译过程封装为一条命令
- **依托上游**：充分利用 XMRig 原生的 Android 平台支持和 ARM 优化

### 1.3 与 XMRig 官方的关系

| 维度 | XMRig 官方 | termux-xmrig |
|------|-----------|-------------|
| 项目性质 | 完整挖矿软件源码 | 自动化构建脚本封装 |
| 源码修改 | — | **无任何修改** |
| 补丁文件 | — | 无 |
| 自定义编译选项 | 用户自由配置 | 仅 `-DWITH_HWLOC=OFF` |
| Android 支持 | 上游原生支持 | 通过上游原生支持实现 |

---

## 二、构建脚本实现原理

### 2.1 build.sh 完整内容

```bash
#!/bin/bash
# Instructions:
# 1. First, go to https://github.com/termux/termux-app/actions/runs/7378253068
#    to download the Termux version suitable for your device.
# 2. Install and launch Termux.
# 3. Enter git clone https://github.com/TokiZeng/termux-xmrig
# 4. Enter chmod +x build.sh
# 5. Enter ./build.sh
# During the process, if prompted to download, press Y;
# for all other prompts, press N.

apt update && apt upgrade
pkg install automake clang git vim cmake
chmod +x start
git clone https://github.com/xmrig/xmrig
cd xmrig
mkdir build && cd build
cmake .. -DWITH_HWLOC=OFF
make -j$(nproc)
cd $HOME
cp termux-xmrig/start ~
rm termux-xmrig/start
```

### 2.2 逐行执行流程详解

#### 阶段一：环境准备

**第 1 步：`apt update && apt upgrade`**

更新 Termux 的 APT 包索引并升级所有已安装包。在 Termux 中，`apt` 是 `pkg` 命令的别名，底层使用 dpkg + apt 包管理体系，包源托管在 Termux 官方仓库（`packages.termux.org`）。

> **注意**：Termux 的包管理系统与 Debian/Ubuntu 类似，但使用的是 Bionic libc 而非 glibc，因此标准 Linux 的 `.deb` 包无法直接安装。

**第 2 步：`pkg install automake clang git vim cmake`**

安装 5 个关键编译工具：

| 工具 | 作用 | 为什么需要 |
|------|------|-----------|
| **clang** | C/C++ 编译器 | Termux 的默认编译器（GCC 不可用），基于 Android NDK 的 LLVM/Clang 工具链 |
| **cmake** | 跨平台构建系统 | XMRig 使用 CMake 管理构建配置 |
| **git** | 版本控制工具 | 克隆 XMRig 官方源码仓库 |
| **automake** | GNU 自动构建工具 | 部分依赖库可能需要 |
| **vim** | 文本编辑器 | 用户后续编辑配置文件 |

#### 阶段二：源码获取与构建配置

**第 3 步：`chmod +x start`**

为项目附带的 `start` 启动脚本添加可执行权限。

**第 4 步：`git clone https://github.com/xmrig/xmrig`**

直接从 XMRig 官方仓库克隆最新源码。这是关键证据：**termux-xmrig 没有维护自己的 XMRig 分支，所有 Android 兼容性都是 XMRig 上游原生实现的。**

**第 5 步：`cd xmrig && mkdir build && cd build`**

采用 out-of-source（源码外）构建方式，这是 CMake 项目的标准最佳实践。构建产物与源码分离，避免污染源码树。

**第 6 步：`cmake .. -DWITH_HWLOC=OFF`（核心步骤）**

这是整个脚本中**唯一的自定义编译选项**。

**为什么禁用 hwloc？**

hwloc（Portable Hardware Locality）是一个用于获取 CPU 拓扑信息（NUMA 节点、CPU 缓存层级、核心/线程绑定）的跨平台库。在桌面 Linux/Windows 上，XMRig 使用 hwloc 来实现：

- NUMA 感知的内存分配
- 最优线程绑定（将线程绑定到特定 CPU 核心）
- 精确的 CPU 拓扑检测

但在 Termux/Android 环境中：

1. hwloc 可能不是 Termux 预编译包，需要额外编译安装
2. 移动设备通常是单 SoC 单 NUMA 节点架构，NUMA 优化收益有限
3. 禁用后可减少一个外部依赖，降低编译失败概率

**禁用 hwloc 的具体影响：**

- XMRig 不会编译 `src/crypto/common/NUMAMemoryPool.cpp`
- 不会编译 `src/crypto/common/VirtualMemory_hwloc.cpp`
- 失去 NUMA 感知的内存池功能和精确的 CPU 亲和性绑定
- 核心挖矿功能不受影响（移动端单 NUMA 架构下性能损失可忽略）

#### 阶段三：编译与部署

**第 7 步：`make -j$(nproc)`**

使用所有可用 CPU 核心进行并行编译。`$(nproc)` 返回当前系统中可用的逻辑 CPU 核心数。在 Android 设备上，这通常等于设备的 CPU 核心数（如 8 核手机就是 `-j8`）。

> **编译时间**：根据设备性能，完整编译 XMRig 大约需要 10-30 分钟。期间手机会明显发热，建议插电并保持散热。

**第 8 步：`cd $HOME && cp termux-xmrig/start ~ && rm termux-xmrig/start`**

将 `start` 启动脚本复制到 Termux 用户主目录（`$HOME`，路径为 `/data/data/com.termux/files/home`），然后删除原位置的脚本。这样用户每次打开 Termux 后，可以直接在主目录运行 `./start` 启动挖矿。

### 2.3 构建流程图

```
用户执行 ./build.sh
    │
    ▼
┌─────────────────────┐
│ apt update & upgrade │  更新包索引和已安装包
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│ pkg install 依赖包   │  clang, cmake, git, automake, vim
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│ git clone xmrig     │  从官方仓库拉取源码
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│ cmake 配置          │  -DWITH_HWLOC=OFF
│                     │  自动检测 Android 平台
│                     │  自动检测 ARM 架构
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│ make -j$(nproc)     │  多线程并行编译
└─────────────────────┘
    │
    ▼
┌─────────────────────┐
│ 部署 start 脚本     │  复制到 $HOME
└─────────────────────┘
    │
    ▼
编译完成：xmrig/build/xmrig
```

---

## 三、Termux 编译环境深度解析

### 3.1 Termux 是什么

Termux 是一款运行在 Android 上的终端模拟器和 Linux 环境应用。它不需要 root 权限，通过将 Linux 工具链交叉编译为 Android 原生库，在 Android 系统上提供了一个接近完整的 Linux 命令行环境。

**核心特性：**
- 无需 root 权限
- 基于 Android NDK 编译所有软件包
- 使用 Bionic libc（而非 glibc）
- 支持 apt/dpkg 包管理
- 可运行 clang、python、git、cmake 等数百种工具

### 3.2 Termux 的文件系统布局

Termux 不遵循标准 Linux FHS（文件系统层次结构标准），所有文件都位于 Android 应用的私有数据目录中：

```
/data/data/com.termux/files/
├── usr/          # $PREFIX — Termux 的"根"文件系统
│   ├── bin/      # 可执行文件
│   ├── etc/      # 配置文件
│   ├── include/  # C/C++ 头文件
│   ├── lib/      # 共享库（.so 文件）
│   ├── share/    # 共享数据
│   ├── tmp/      # 临时目录
│   └── var/      # 可变数据
└── home/         # $HOME — 用户主目录
```

**关键环境变量：**
- `$PREFIX` = `/data/data/com.termux/files/usr`
- `$HOME` = `/data/data/com.termux/files/home`
- `TMPDIR` = `$PREFIX/tmp`

### 3.3 包管理系统工作原理

Termux 使用 **apt + dpkg** 作为底层包管理工具，架构与 Debian/Ubuntu 类似：

- `pkg` 是一个包装器脚本，提供更友好的命令快捷方式
- 官方包仓库位于 `https://packages.termux.org/apt/termux-main/`
- 包从 `termux-packages` 仓库中的构建脚本构建
- 由 Termux 开发团队维护、签名和发布

**重要限制：**
- 不支持标准 Debian/Ubuntu 的 `.deb` 包（ABI 不兼容）
- 单架构：同一时间只支持一种 CPU 架构
- 不支持包降级
- 不能以 root 使用 apt（避免破坏 SELinux 标签）

### 3.4 Clang 编译工具链

Termux 的 clang 编译器基于 **Android NDK** 构建。所有 Termux 软件包都使用 Android NDK 的交叉编译工具链编译，生成的二进制文件链接到 Android 的 Bionic libc。

**工具链组成：**

| 工具 | 来源 | 说明 |
|------|------|------|
| `clang` / `clang++` | NDK LLVM | C/C++ 编译器 |
| `ld.lld` | NDK LLVM | 链接器（LLVM lld） |
| `llvm-ar` | NDK LLVM | 静态库归档工具 |
| `llvm-objcopy` | NDK LLVM | 目标文件复制工具 |
| `llvm-strip` | NDK LLVM | 符号剥离工具 |

**目标三元组（以 aarch64 为例）：**
- 目标平台：`aarch64-linux-android`
- 编译器驱动：`aarch64-linux-android-clang`
- 在 Termux 中，`clang` 命令会自动配置为当前架构

### 3.5 默认编译与链接标志

Termux 环境下编译 C/C++ 程序时，默认会应用以下优化标志：

**CFLAGS（编译标志）：**
- `-Oz`：最小化体积优化（非调试模式）
- `-fstack-protector-strong`：栈保护
- `-isystem $PREFIX/include`：系统头文件路径

**LDFLAGS（链接标志）：**
- `-L$PREFIX/lib`：库搜索路径
- `-Wl,-rpath=$PREFIX/lib`：运行时库搜索路径（DT_RUNPATH）
- `-Wl,--enable-new-dtags`：启用新的动态标签
- `-Wl,--as-needed`：仅链接实际需要的库
- `-Wl,-z,relro,-z,now`：重定位只读 + 立即绑定（安全加固）

> **关于 rpath**：Android 7 及以上版本使用 ELF 头部的 `DT_RUNPATH` 属性来查找共享库，而不是依赖 `LD_LIBRARY_PATH` 环境变量。这就是为什么构建时需要设置 `-Wl,-rpath`。

### 3.6 Bionic libc 与 glibc 的差异

| 特性 | Bionic libc (Android/Termux) | glibc (标准 Linux) |
|------|---------------------------|-------------------|
| 动态链接器 | `/system/bin/linker64` (64位) | `/lib64/ld-linux-x86-64.so.2` |
| 目标 | 嵌入式/移动设备优化 | 通用服务器/桌面 |
| 体积 | 更小 | 更大 |
| POSIX 兼容性 | 部分兼容 | 完全兼容 |
| 线程库 | pthread（内置） | pthread（内置） |
| 数学库 | libm（内置） | libm（内置） |
| 扩展功能 | 较少 | 丰富 |

**实际影响：**
- glibc 链接的二进制文件**无法**在 Termux 中直接运行
- 即使静态链接，DNS 解析也可能失败（Android 没有 `/etc/resolv.conf`）
- 部分 POSIX API 在不同 Android API 级别下行为不同

### 3.7 XMRig 在 Termux 中的平台检测

当在 Termux 中执行 `cmake .. -DWITH_HWLOC=OFF` 时，CMake 会自动检测当前系统环境：

**平台检测（cmake/os.cmake）：**
```cmake
if (ANDROID OR CMAKE_SYSTEM_NAME MATCHES "Android")
    set(XMRIG_OS_ANDROID ON)
endif()
```

在 Termux 环境中，`CMAKE_SYSTEM_NAME` 为 `"Android"`，因此 `XMRIG_OS_ANDROID` 被自动设置为 `ON`。

**Android 专属链接库：**
```cmake
if (XMRIG_OS_ANDROID)
    set(EXTRA_LIBS pthread rt dl log)
endif()
```

链接的库：
- `pthread`：POSIX 线程库
- `rt`：实时信号和 POSIX 时钟
- `dl`：动态链接库运行时加载
- `log`：Android 日志系统（logcat）

**ARM 架构检测与优化（cmake/cpu.cmake）：**
```cmake
if (CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64|arm64|ARM64|armv8-a)$")
    set(ARM_TARGET 8)
endif()

# 编译标志
set(ARM8_CXX_FLAGS "-march=armv8-a+crypto")
```

`-march=armv8-a+crypto` 启用 ARMv8-A 架构指令集及 **ARM Crypto 扩展**（AES、SHA-1、SHA-256 硬件加速指令），对加密密集型的挖矿算法有性能提升作用。

---

## 四、XMRig 启动与初始化流程

### 4.1 入口点与主类

XMRig 的入口极其简洁，`src/xmrig.cpp` 中：

```cpp
int main(int argc, char **argv) {
    App app(argc, argv);
    return app.exec();
}
```

`App` 类是整个程序的顶层编排器，持有三个核心成员：

| 成员 | 类型 | 用途 |
|------|------|------|
| `m_controller` | `shared_ptr<Controller>` | 中央控制器，管理所有子系统 |
| `m_console` | `shared_ptr<Console>` | 控制台输入输出 |
| `m_signals` | `shared_ptr<Signals>` | 操作系统信号处理 |

### 4.2 App::exec() 启动序列

`App::exec()` 方法编排了完整的启动流程：

```
1. 配置校验
   └─ isReady() 检查命令行参数和配置文件是否有效

2. 后台模式（可选）
   └─ background() 守护进程化
      Unix: fork() + setsid()
      Windows: ShowWindow(SW_HIDE)

3. 信号注册
   └─ 创建 Signals 实例，监听 SIGTERM/SIGINT/SIGHUP

4. Controller 初始化（init 阶段）
   ├─ Base::init() 加载配置、设置 API
   ├─ VirtualMemory 系统初始化（内存池、Huge Pages）
   ├─ 创建 Network 实例
   └─ 注册 HwApi（API 端点）

5. 创建控制台
   └─ 非后台模式下创建 Console，处理键盘输入

6. 摘要输出
   └─ Summary::print() 打印 CPU/内存/线程/矿池信息

7. 开始挖矿（start 阶段）
   ├─ Base::start() 启动 API HTTP 服务
   ├─ 创建 Miner 实例（初始化所有后端：CPU/OpenCL/CUDA）
   └─ network()->connect() 建立矿池连接

8. 事件循环
   └─ 运行 libuv 事件循环，直至收到终止信号
```

### 4.3 Controller 的两阶段初始化

**Phase 1 — init()：**
- 加载并解析配置文件
- 初始化虚拟内存系统（根据配置分配 Huge Pages、内存池）
- 创建 Network 网络实例
- 注册 API 端点监听器

**Phase 2 — start()：**
- 启动 API HTTP 服务
- 创建 Miner 实例，初始化所有启用的后端（CPU/OpenCL/CUDA）
- 建立矿池连接，开始接收任务

### 4.4 关闭流程

当收到终止信号（Ctrl+C 或 SIGTERM）时：
1. `Base::stop()` 停止 API 服务
2. `m_miner->stop()` 停止所有挖矿 worker 线程
3. 关闭网络连接
4. `VirtualMemory::destroy()` 释放大页内存
5. 析构所有对象，正常退出

---

## 五、Stratum 协议通信实现

### 5.1 Stratum 类层次结构

XMRig 的 Stratum 实现围绕客户端类层次构建：

```
IClient（接口）
  └─ BaseClient（抽象基类）
       ├─ Client              — TCP + TLS，标准矿池挖矿
       ├─ DaemonClient        — ZMQ + HTTP RPC，solo 挖矿
       └─ SelfSelectClient    — 包装 Client，自选模式
```

**连接生命周期状态：**
```
UnconnectedState → HostLookupState (DNS解析)
  → ConnectingState (TCP握手)
  → ConnectedState (协议层login)
  → ClosingState / ReconnectingState
```

### 5.2 消息收发数据流

**入站消息处理流程：**
```
uv_read_cb → Client::onRead
  → LineReader（按 \n 分行缓冲）
    → Client::parse 解析 JSON-RPC
      ├─ 含 method → parseNotification（如 "job" 任务通知）
      └─ 含 result/error → 按 request ID 匹配 parseResponse
```

**出站消息（份额提交）流程：**
```
挖矿后端生成 JobResult
  → Client::submit(JobResult)
    → nonce、result、signature 转 hex
    → 构造 method 为 "submit" 的 JSON 对象
    → 序列化到 m_sendBuf
    → uv_write 发送到矿池
```

### 5.3 XMRig/Monero 变体的 JSON 消息格式

XMRig 在门罗币矿池上使用的是一种简化的 Stratum 变体（按换行分隔的 JSON-RPC），与比特币标准 Stratum 不同，使用 `login`/`job`/`submit` 方法名。

#### 登录消息（客户端 → 矿池）

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "method": "login",
  "params": {
    "login": "<钱包地址>",
    "pass": "<密码/矿工名>",
    "agent": "XMRig/6.26.0",
    "algo": ["rx/0", "cn/r", "cn/0"]
  }
}
```

#### 登录响应（矿池 → 客户端）

```json
{
  "id": 1,
  "jsonrpc": "2.0",
  "result": {
    "id": "<session_id>",
    "job": {
      "blob": "<区块头模板hex>",
      "job_id": "<任务ID>",
      "target": "<目标值hex>",
      "seed": "<RandomX种子哈希>",
      "algo": "rx/0"
    },
    "extensions": ["algo", "nicehash"]
  },
  "error": null
}
```

#### 任务分发通知（矿池 → 客户端）

```json
{
  "jsonrpc": "2.0",
  "method": "job",
  "params": {
    "blob": "<区块头模板hex>",
    "job_id": "<任务ID>",
    "target": "<目标值hex>",
    "seed": "<RandomX种子哈希>",
    "algo": "rx/0"
  }
}
```

#### 份额提交（客户端 → 矿池）

```json
{
  "id": 2,
  "jsonrpc": "2.0",
  "method": "submit",
  "params": {
    "id": "<session_id>",
    "job_id": "<任务ID>",
    "nonce": "<8位hex随机数>",
    "result": "<32位hex哈希结果>"
  }
}
```

#### 份额响应（矿池 → 客户端）

```json
{
  "id": 2,
  "jsonrpc": "2.0",
  "result": {
    "status": "OK"
  },
  "error": null
}
```

### 5.4 Job 数据结构

`Job` 类封装了一个挖矿任务的所有信息：

| 字段 | 类型 | 说明 |
|------|------|------|
| `m_blob` | `Buffer` | 区块头模板（blob） |
| `m_target` | `uint64_t` | 目标难度值（64 位） |
| `m_seed` | `uint64_t[4]` | RandomX 种子哈希 |
| `m_algorithm` | `Algorithm` | 算法类型（如 rx/0） |
| `m_nonceOffset` | `uint32_t` | nonce 在 blob 中的字节偏移 |
| `m_nonceSize` | `uint32_t` | nonce 的字节大小 |

> nonce 的偏移和大小因算法族而异：Cryptonight 通常在偏移 39 处有 4 字节 nonce；RandomX 使用类似的布局。

### 5.5 难度与目标的关系

份额难度与哈希目标值的精确关系：

```
target = 2^256 / (65536 × difficulty)
```

- 难度越高 → target 越小 → 找到有效 share 越难
- 矿工计算的哈希（按字节序解释为 256 位整数）若 ≤ target，则该 nonce 构成有效 share
- 矿池会对提交的 share 进行二次验证

### 5.6 传输层与安全

- **异步 I/O**：基于 libuv 的事件驱动模型
- **TLS 支持**：通过 OpenSSL 实现，支持证书指纹验证和 SNI
- **SOCKS5 代理**：支持通过 Tor 等代理挖矿
- **重连机制**：连接断开后自动指数退避重连

---

## 六、RandomX 算法执行原理

### 6.1 RandomX 设计目标

RandomX 是门罗币目前使用的工作量证明（PoW）算法，于 2019 年 11 月启用。其核心设计理念是**让通用 CPU 成为最高效的挖矿硬件**，从而抵抗专用 ASIC 矿机。

**关键设计策略：**
1. **随机程序执行**：每个哈希都生成不同的随机程序，验证需要实际运行代码
2. **大内存暂存区**：2MB Scratchpad，与现代 CPU L3 缓存匹配
3. **浮点运算**：大量浮点操作，充分利用 CPU 的 FPU
4. **随机内存访问**：频繁随机读写内存，ASIC 难以高效实现

### 6.2 核心数据结构

| 结构 | 大小 | 说明 |
|------|------|------|
| `randomx_cache` | ~256 MB | Argon2 派生内存 + 8 个 SuperscalarProgram + JIT 编译器 |
| `randomx_dataset` | ~2.03 GB | 大连续缓冲区，64 字节为一项，共约 3350 万项 |
| `randomx_vm` | ~2MB+ | 虚拟机：Scratchpad + 寄存器文件 + 程序 |

**虚拟机寄存器：**
- **8 个整数寄存器**（r0-r7）：64 位通用整数运算
- **12 个浮点寄存器**（f0-f3, e0-e3, a0-a3）：浮点运算
- **Scratchpad**：2MB 随机读写空间，分为 L1 (16KB) / L2 (256KB) / L3 (2MB) 层级

### 6.3 哈希计算完整流程

每个 RandomX 哈希的计算分为三个阶段：

#### 阶段一：Scratchpad 初始化

```
1. 将输入数据用 Blake2b 哈希，得到 64 字节的 tempHash
2. 用 tempHash 作为种子初始化 2MB 的 Scratchpad
   ├─ L1 (16KB)：直接由种子派生
   ├─ L2 (256KB)：基于 L1 扩展
   └─ L3 (2MB)：基于 L2 扩展
```

#### 阶段二：程序执行循环（核心）

执行 8 个随机程序（ProgramCount = 8），每个程序运行 2048 次迭代（ProgramIterations = 2048）：

```
对于每个程序 (共 8 个):
  │
  ├─ generateProgram(tempHash + counter)
  │   └─ 基于种子和计数器生成 256 条随机指令
  │
  ├─ resetRoundingMode() 重置浮点舍入模式
  │
  └─ 循环 2048 次:
      ├─ 随机读取 dataset/cache 中的数据
      ├─ 执行一条随机指令
      ├─ 读写 Scratchpad 内存
      └─ 更新寄存器状态

  ├─ 如果不是最后一个程序:
  │   └─ 哈希寄存器状态 → 新的 tempHash
  │       └─ 作为下一个程序的生成种子
  │
  └─ 进入下一个程序
```

#### 阶段三：Finalization（最终化）

```
1. 执行完 8 个程序后，寄存器中保存最终状态
2. 对寄存器状态进行哈希（Blake2b）
3. 输出 32 字节的最终哈希结果
```

### 6.4 指令集

每个程序由 256 条指令组成，从预定义的指令集中随机选择：

**整数指令：**
- `IADD_RS` / `ISUB_R`：加/减
- `IMUL_R` / `IMULH_R`：乘/高位乘
- `IXOR_R`：异或
- `IROR_R`：循环右移

**浮点指令：**
- `FADD_R` / `FSUB_R`：浮点加/减
- `FMUL_R` / `FDIV_M`：浮点乘/除
- `FSQRT_R`：浮点平方根

**内存指令：**
- `ISTORE`：存储到 Scratchpad

**控制流指令：**
- `CBRANCH`：条件分支

> 每种指令的出现频率由配置参数 `RANDOMX_FREQ_*` 控制，确保程序的多样性和平衡性。

### 6.5 数据集初始化原理

RandomX dataset 是一个约 2GB 的只读数据结构，用于在挖矿过程中随机读取。其初始化过程：

```
对于每个 64 字节的 dataset item:
  1. 用 item 编号初始化 8 个 64 位寄存器
  2. 执行 8 次循环:
     ├─ 根据当前寄存器状态计算 cache 地址
     ├─ 执行预生成的 SuperscalarProgram
     └─ 与 cache 数据进行 XOR 运算
  3. 将最终寄存器状态写入 dataset item
```

**关键特性：**
- Dataset 是**只读**的，一旦生成就不会改变（直到种子变化）
- 多个挖矿线程共享同一个 dataset
- 初始化可以多线程并行加速
- 种子大约每 3 天变化一次（Monero 配置）

### 6.6 VM 实现变体

根据三个功能标志（JIT / HARD_AES / FULL_MEM）的组合，XMRig 提供 8 种 VM 实现：

| VM 类型 | JIT | 硬件 AES | 完整内存 | 相对速度 |
|---------|-----|---------|---------|---------|
| `CompiledVmHardAes` | ✓ | ✓ | ✓ | 最快 (~3-4x) |
| `CompiledVmDefault` | ✓ | — | ✓ | ~2-3x |
| `InterpretedVmHardAes` | — | ✓ | ✓ | ~1.5x |
| `InterpretedVmDefault` | — | — | ✓ | 基线 (1x) |
| ... 轻量模式 ... | ... | ... | ~256MB cache | 更慢 |

> 在移动端 ARM CPU 上，JIT 编译（ARM64 架构）同样可用，可以显著提升挖矿性能。

### 6.7 Monero 默认配置参数

| 参数 | 值 | 说明 |
|------|-----|------|
| `ArgonMemory` | 262,144 KB | Cache 大小（256 MB） |
| `ArgonIterations` | 3 | Argon2 迭代次数 |
| `DatasetBaseSize` | 2 GB | Dataset 基础大小 |
| `ProgramSize` | 256 | 每个程序的指令数 |
| `ProgramIterations` | 2048 | 每个程序的迭代次数 |
| `ProgramCount` | 8 | 程序数量 |

---

## 七、JIT 即时编译实现

### 7.1 JIT 编译器架构

XMRig 的 RandomX JIT 编译器将随机生成的 RandomX 程序直接编译为本地机器码执行，相比解释执行可获得 2-4 倍的性能提升。

**JIT 编译器接口（`JitCompiler`）：**
- `prepare()` — 准备编译环境
- `generateProgram()` — 生成完整程序的机器码
- `generateProgramLight()` — 生成轻量模式程序
- `generateSuperscalarHash()` — 生成 Superscalar 哈希代码
- `generateDatasetInitCode()` — 生成数据集初始化代码
- `getProgramFunc()` — 获取编译后的程序函数指针

**多架构支持：**

| 架构 | 编译器类 | 静态汇编模板 |
|------|---------|-------------|
| x86-64 | `JitCompilerX86` | `jit_compiler_x86_static.S` |
| ARM64 (AArch64) | `JitCompilerA64` | `jit_compiler_a64_static.S` |
| RISC-V 64 | `JitCompilerRV64` | `jit_compiler_rv64_static.S` |
| 回退 | `JitCompilerFallback` | 抛出 "不支持" 错误 |

> 由 `CompiledVm` 类编排整个 JIT 编译和执行流程。

### 7.2 编译过程

```
RandomX Program (256 条指令)
    │
    ▼
┌───────────────────────┐
│ Prologue 生成         │  函数序言：保存寄存器、设置栈帧
└───────────────────────┘
    │
    ▼
┌───────────────────────┐
│ 指令翻译循环           │  逐条翻译 RandomX 指令
│                       │  • 寄存器映射到物理寄存器
│                       │  • 操作码处理派发
│                       │  • 发射原生机器码字节
└───────────────────────┘
    │
    ▼
┌───────────────────────┐
│ Epilogue 生成         │  函数尾声：恢复寄存器、返回
└───────────────────────┘
    │
    ▼
┌───────────────────────┐
│ 内存保护切换           │  W^X：从可写切换为可读可执行
└───────────────────────┘
    │
    ▼
执行 ProgramFunc(...)
```

### 7.3 寄存器映射（以 ARM64 为例）

JIT 编译器将 RandomX 虚拟机的逻辑寄存器映射到物理 CPU 寄存器：

**ARM64 寄存器映射：**
- RandomX 整数寄存器 r0-r7 → ARM64 通用寄存器 x0-x7
- Scratchpad 基地址 → x8 或其他可用寄存器
- Dataset 基地址 → x9
- 浮点寄存器 → 相应的 NEON/VFP 寄存器

**x86-64 寄存器映射：**
- r0-r7 → r8-r15
- f0-f3 → xmm0-xmm3
- e0-e3 → xmm4-xmm7
- a0-a3 → xmm8-xmm11
- Scratchpad 指针 → rsi
- Dataset 指针 → rdi

### 7.4 JIT 内存管理（W^X 保护）

现代操作系统要求内存页不能同时可写和可执行（W^X 安全策略）。JIT 编译器通过以下流程处理：

```
1. 分配：allocPagedMemory()
   └─ 分配可执行内存页（可选择 2MB Huge Pages 减少 TLB miss）

2. 写入：enableWriting()
   └─ 将内存页标记为可写（不可执行）
   └─ 逐条发射机器码字节到内存

3. 执行：enableExecution()
   └─ 将内存页标记为只读 + 可执行（RX）
   └─ 通过函数指针调用编译后的代码
```

> 这是 JIT 编译器的标准做法，确保符合操作系统的安全要求。

### 7.5 ARM64 JIT 的特殊性

在 ARM64（AArch64）架构上，JIT 编译有一些特殊的考量：

1. **指令缓存刷新**：ARM 架构有分离的指令缓存和数据缓存，自修改代码后需要手动刷新指令缓存（`__builtin___clear_cache`）
2. **NEON 指令**：浮点运算使用 NEON/VFP 指令集
3. **Crypto 扩展**：如果 CPU 支持 ARM Crypto 扩展，AES 相关操作使用硬件加速指令
4. **代码对齐**：ARM64 指令是 4 字节定长，需要注意对齐

### 7.6 性能优化技术

**JCC Erratum 缓解（x86 特定）：**
- 避免分支指令跨越 32 字节边界
- 缓解 Intel Skylake 系列 CPU 的性能损失

**Huge Pages JIT：**
- 使用 2MB 大页内存存储 JIT 代码
- 减少 TLB miss，提升指令获取效率
- 配置项：`"huge-pages-jit": true`

**代码补丁优化：**
- RandomX v2 调整：用 NOP 或直接移动覆盖旧版本的某些跳转
- 减少分支预测失败

---

## 八、CPU 后端线程模型

### 8.1 组件结构

```
Miner（顶层编排）
  └─ CpuBackend（CPU 后端实现 IBackend 接口）
       └─ CpuBackendPrivate（私有实现）
            ├─ Workers<CpuLaunchData>（Worker 管理）
            │    └─ Thread[]（工作线程数组）
            │         └─ CpuWorker<N>（挖矿 Worker）
            │              └─ VirtualMemory（虚拟机内存）
            └─ JobResults（份额提交队列）
```

### 8.2 后端初始化流程

```
1. Miner 创建 CpuBackend
   └─ 传入配置和回调接口

2. CpuBackend 构造
   └─ 创建 CpuBackendPrivate
      └─ 创建 Workers 实例
         └─ setBackend(this) 绑定后端

3. 任务到达时 setJob(job)
   └─ start() 启动 Worker
      └─ 为每个线程创建 Thread 对象
         └─ thread.start(onReady)
```

### 8.3 Worker 类详解

`CpuWorker<N>` 是一个模板类，`N` 表示 intensity（并发哈希数），即一个 Worker 同时处理的哈希数量。

**常见的特化版本：**
- `CpuWorker<1>`：单哈希模式
- `CpuWorker<2>`、`CpuWorker<4>`、`CpuWorker<8>`：多哈希模式

> 多哈希模式通过流水线和指令级并行提高效率，但需要更多的寄存器和缓存。

**Worker 构造时的内存分配：**
- RandomX：分配 Scratchpad + VM 状态
- CryptoNight：创建 CN 上下文
- GhostRider：创建 helper 线程

### 8.4 挖矿主循环

`CpuWorker::start()` 方法是挖矿的核心循环：

```
start()
  │
  ├─ consumeJob() → WorkerJob::add() 获取新任务
  │
  └─ 无限循环:
      │
      ├─ Nonce::next()
      │   └─ 原子操作预留 nonce 范围（默认 32768 个）
      │       └─ 防止多个线程重复工作
      │
      ├─ 计算哈希（根据算法不同调用不同函数）
      │   ├─ RandomX: randomx_calculate_hash + RxVm
      │   ├─ CryptoNight: cn_hash_fun 上下文哈希
      │   └─ GhostRider: ghostrider::hash_octa
      │
      ├─ 检查结果是否满足 target 难度
      │   └─ 如果满足 → JobResults::submit() 提交份额
      │
      └─ nextRound() → 继续下一 nonce 范围
```

### 8.5 Nonce 分配机制

`Nonce::next()` 使用原子操作在多个 Worker 线程间分配 nonce 范围：

- 每个 Worker 一次预留一批 nonce（默认 `kReserveCount = 32768`）
- 使用原子递增操作确保全局唯一性
- 减少线程间同步开销（不是每个 nonce 都同步，而是批量分配）

### 8.6 算力统计

`Workers::tick()` 周期性地收集每个 Worker 的算力数据：

```
每个 Worker:
  └─ tick(ticks) → hashrateData(hashCount, timestamp, rawHashes)
       └─ 中央 Hashrate 对象聚合:
            ├─ add(workerId, hashCount, timestamp)  单线程统计
            └─ add(totalHashCount, timestamp)        全局统计
```

- API 端点 `/1/hashes` 暴露算力统计数据
- 支持 10 秒、1 分钟、15 分钟等多时间窗口的平均算力

### 8.7 自检机制

启动前，每个 Worker 会执行 `selfTest()`，用硬编码的测试向量验证算法实现的正确性：

- CryptoNight 各变体测试
- RandomX 测试（N=1）
- GhostRider 测试（N=8）

> 如果自检失败，Worker 会报错并终止，防止提交错误的份额导致矿池拒绝。

### 8.8 移动端的线程策略

在 Android 手机上运行 XMRig 时，线程数选择需要权衡：

| 策略 | 线程数 | 特点 |
|------|--------|------|
| 保守 | 核心数 - 2 | 留 2 个核心给系统，手机基本可用 |
| 平衡 | 核心数 - 1 | 留 1 个核心，手机略有卡顿 |
| 满负载 | 全部核心 | 性能最高，但手机严重发热卡顿 |
| 低功耗 | 小核心数 | 仅用能效核心，发热少但算力低 |

> 示例中使用 `-t 3`，适合 8 核手机的保守/平衡策略。

---

## 九、虚拟内存管理

### 9.1 Huge Pages（大页内存）

Huge Pages 是 XMRig 最重要的性能优化之一。使用 2MB（或 1GB）的大内存页可以减少 TLB（Translation Lookaside Buffer）缓存未命中，从而提升内存访问性能。

**性能提升：**
- 典型提升：20-30%
- RandomX 可达：50%
- 1GB 大页额外提升：1-3%（仅 Linux）

**各平台实现：**

| 平台 | 名称 | 说明 |
|------|------|------|
| Windows | Large Pages | 需要 `SeLockMemoryPrivilege` 权限 |
| Linux | Huge Pages | 通过 `sysctl vm.nr_hugepages` 配置 |
| macOS | Super Pages | 系统自动管理 |
| Android/Termux | 不支持 | 受限于 Android 内核和权限 |

> **在 Termux 中**：由于 Android 系统的限制，普通应用无法配置 Huge Pages，因此 XMRig 在 Termux 中运行时无法享受这一性能优化。

### 9.2 内存分配策略

RandomX 需要两种主要的内存结构：

| 结构 | 大小 | 2MB Huge Pages | 1GB Pages | NUMA 感知 |
|------|------|---------------|-----------|-----------|
| Cache | 256 MB | 支持 | 否 | 是 |
| Dataset | ~2.03 GB | 支持 | 支持(Linux) | 是 |

`VirtualMemory` 类统一处理所有内存分配与保护：
- 尝试分配 Huge Pages
- 如果失败，回退到普通页
- 处理内存对齐要求
- 管理 JIT 代码的 W^X 保护

### 9.3 NUMA 与存储抽象

XMRig 通过 `IRxStorage` 接口抽象 dataset 存储：

- **`RxBasicStorage`**：单 dataset 共享全系统（单 NUMA 节点）
- **`RxNUMAStorage`**：每 NUMA 节点一个 dataset，最小化跨节点内存延迟

> 在移动端（Termux/Android），由于禁用了 hwloc 且通常是单 SoC 架构，使用的是 `RxBasicStorage` 模式。

### 9.4 Dataset 共享模型

Dataset 是一个**只读**结构，计算一次后被多个挖矿线程共享读取：

```
Dataset 初始化（种子变化时）:
  ├─ 计算 dataset 是 CPU 密集型任务
  ├─ 可以多线程并行加速
  └─ 完成后所有挖矿线程共享读取

运行时:
  ├─ 所有挖矿线程只读访问 dataset
  ├─ 不需要同步（只读）
  └─ 直到 seed 变化时整体重建
```

`RxQueue` 异步队列处理 dataset 初始化任务，避免阻塞挖矿线程。

### 9.5 内存池（Memory Pool）

配置项 `"memory-pool": true` 会预保留 Huge Pages 内存池：
- 防止算法切换时丢失 Huge Page 分配
- 在不同算法间切换时更快
- 避免反复申请/释放大页内存的开销

---

## 十、完整运行时数据流

### 10.1 端到端架构图

```
┌──────────────────────────────────────────────────────────────┐
│                     Android 设备                              │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ Termux 应用                                             │  │
│  │  ┌──────────────────────────────────────────────────┐  │  │
│  │  │ Bash Shell                                        │  │  │
│  │  │  ./start → xmrig -o pool:port -u wallet -t 3     │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  │                       │                                │  │
│  │  ┌────────────────────▼─────────────────────────────┐  │  │
│  │  │              XMRig 进程                           │  │  │
│  │  │                                                   │  │  │
│  │  │  ┌──────────┐   ┌──────────┐   ┌──────────────┐ │  │  │
│  │  │  │ Network  │──▶│  Miner   │──▶│ CPU Backend  │ │  │  │
│  │  │  │ (Stratum)│   │ (编排器) │   │ (多线程挖矿) │ │  │  │
│  │  │  └──────────┘   └──────────┘   └──────┬───────┘ │  │  │
│  │  │                                       │          │  │  │
│  │  │  ┌───────────────────────────┐        │          │  │  │
│  │  │  │  RandomX                  │◀───────┘          │  │  │
│  │  │  │  • Cache (~256MB)         │                   │  │  │
│  │  │  │  • Dataset (~2GB)         │                   │  │  │
│  │  │  │  • JIT Compiler (ARM64)   │                   │  │  │
│  │  │  │  • VMs (每个线程一个)      │                   │  │  │
│  │  │  └───────────────────────────┘                   │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  │                       │                                │  │
│  │  ┌────────────────────▼─────────────────────────────┐  │  │
│  │  │          ARM CPU 硬件 (多核 SoC)                 │  │  │
│  │  └──────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬──────────────────────────────┘
                                │ Stratum (TCP/TLS)
                                ▼
                    ┌──────────────────────┐
                    │      矿池服务器       │
                    │  (Herominers 等)     │
                    └──────────┬───────────┘
                               │ P2P 网络
                               ▼
                    ┌──────────────────────┐
                    │     区块链网络        │
                    │  (Monero / Zephyr)   │
                    └──────────────────────┘
```

### 10.2 完整运行时序

```
用户                XMRig                 矿池                 区块链
 │                   │                      │                    │
 │  ./start          │                      │                    │
 │──────────────────▶│                      │                    │
 │                   │                      │                    │
 │                   │ 1. App 初始化        │                    │
 │                   │    • 加载配置         │                    │
 │                   │    • 初始化内存       │                    │
 │                   │    • 创建 Miner       │                    │
 │                   │    • 启动 API         │                    │
 │                   │                      │                    │
 │                   │ 2. Stratum 连接+登录  │                    │
 │                   │─────────────────────▶│                    │
 │                   │                      │                    │
 │                   │ 3. 分发挖矿任务       │                    │
 │                   │◀─────────────────────│                    │
 │                   │    (job blob+target) │                    │
 │                   │                      │                    │
 │                   │ 4. Dataset 初始化     │                    │
 │                   │    (首次或seed变化)   │                    │
 │                   │                      │                    │
 │                   │ 5. 启动挖矿线程       │                    │
 │                   │    ├─ 分配 nonce 范围 │                    │
 │                   │    ├─ RandomX 哈希计算│                    │
 │                   │    ├─ JIT 编译执行    │                    │
 │                   │    └─ 检查 target     │                    │
 │                   │                      │                    │
 │                   │ 6. 提交有效份额       │                    │
 │                   │─────────────────────▶│                    │
 │                   │                      │                    │
 │                   │ 7. 验证+新任务        │                    │
 │                   │◀─────────────────────│                    │
 │                   │                      │                    │
 │                   │         ... 循环 ...  │                    │
 │                   │                      │                    │
 │                   │         （找到区块时）  │                    │
 │                   │                      │ 8. 提交区块        │
 │                   │                      │───────────────────▶│
 │                   │                      │                    │
 │  H 键查看哈希率     │                      │                    │
 │──────────────────▶│                      │                    │
 │                   │ 打印统计信息          │                    │
 │◀──────────────────│                      │                    │
 │                   │                      │                    │
 │  Ctrl+C 停止      │                      │                    │
 │──────────────────▶│                      │                    │
 │                   │ 优雅关闭              │                    │
```

### 10.3 启动脚本（start）详解

`start` 脚本是一个预配置的挖矿启动模板：

```bash
#!/bin/bash
./termux-xmrig/xmrig/build/xmrig \
  -o hk.zephyr.herominers.com:1123 \
  -u ZEPHs8RuJ66Tf43KBbbtnQNxjm48qN6S83Zko2hNv9uhMPHb3jchK9WRkvppjEtRQy5dr2UNBSggdNc1pNJYNYL1ipwqzYgMZZ5.op \
  -p x \
  -t 3
```

**参数详解：**

| 参数 | 示例值 | 说明 |
|------|--------|------|
| `-o` | `hk.zephyr.herominers.com:1123` | 矿池地址和端口 |
| `-u` | `ZEPHs8...YgMZZ5.op` | 用户名 = 钱包地址 + `.矿工名` |
| `-p` | `x` | 密码，大多数矿池用 `x` 占位 |
| `-t` | `3` | CPU 线程数 |

**两种启动方式：**

1. **start 脚本方式**：修改 `start` 文件后，每次打开 Termux 直接运行 `./start`（推荐）
2. **手动方式**：进入 `xmrig/build` 目录，手动输入完整命令（适合临时测试）

---

## 十一、技术总结与思考

### 11.1 项目的技术亮点

1. **极致精简的封装**：3 个文件、6KB 代码，却实现了从环境准备到编译部署的全自动化

2. **零侵入式设计**：不 fork 上游源码，不打补丁，始终使用最新的官方 XMRig

3. **巧妙的适配策略**：仅通过 `-DWITH_HWLOC=OFF` 一个编译选项就解决了 Termux 环境下的依赖问题

4. **充分利用上游支持**：依托 XMRig 原生的 Android 平台检测和 ARM 优化，不重复造轮子

5. **一键式用户体验**：将复杂的交叉编译过程封装成一条命令，大幅降低技术门槛

### 11.2 运行实现的核心技术栈

| 层级 | 技术 | 作用 |
|------|------|------|
| 硬件层 | ARMv8-A CPU + Crypto 扩展 | 执行挖矿计算 |
| 系统层 | Android + Bionic libc | 操作系统运行环境 |
| 环境层 | Termux + NDK Clang | 类 Linux 编译/运行环境 |
| 构建层 | CMake + Make | 跨平台构建系统 |
| 网络层 | libuv + Stratum 协议 | 异步 I/O + 矿池通信 |
| 算法层 | RandomX + JIT | PoW 哈希计算 |
| 线程层 | 多线程 + 原子 nonce 分配 | 并行挖矿 |

### 11.3 移动端挖矿的局限性

1. **算力有限**：移动 ARM CPU 的 RandomX 哈希率（~100-300 H/s 单线程）远低于桌面 CPU（~500-1000+ H/s 单线程）

2. **散热与降频**：手机长时间满载运行会严重发热，触发热节流（thermal throttling），实际性能持续下降

3. **无 Huge Pages**：Android 系统不支持配置 Huge Pages，损失约 20-50% 的性能

4. **后台限制**：Android 的电池优化机制可能限制或终止后台进程

5. **电池损耗**：持续高负载会加速电池老化，缩短设备寿命

6. **收益微薄**：考虑电费和硬件损耗，手机挖矿的经济收益几乎可以忽略不计

### 11.4 安全考量

- **钱包地址安全**：start 脚本中明文存储钱包地址，注意保护
- **矿池信任**：应选择信誉良好的矿池，避免恶意矿池
- **源码可信性**：直接从官方 xmrig/xmrig 仓库克隆，源码可信度较高
- **编译安全**：本地编译相比下载预编译二进制更安全，但仍需注意供应链安全

### 11.5 最终评价

从工程角度看，`termux-xmrig` 是一个**优秀的自动化封装范例**。它没有重复造轮子，而是用最少的代码将已有的强大工具（Termux + XMRig）巧妙地串联起来，为特定场景（Android 移动端）提供了开箱即用的解决方案。其设计思路充分体现了 Unix 哲学——做一件事，做好一件事。

从实用角度看，受限于移动设备的算力、散热和功耗，手机挖矿的实际收益非常有限。这个项目更多的价值在于**技术探索和学习意义**——它展示了如何在 Android 设备上编译和运行高性能原生计算程序，对于理解 Termux 环境、交叉编译、加密算法和分布式系统都有很好的参考价值。

---

## 参考来源

1. [GitHub: TokiZeng/termux-xmrig](https://github.com/TokiZeng/termux-xmrig) — 项目源码仓库
2. [XMRig Official Documentation](https://xmrig.com/docs/miner) — XMRig 官方文档
3. [DeepWiki: XMRig Source Analysis](https://deepwiki.com/xmrig/xmrig) — XMRig 源码深度分析
4. [Termux Wiki: Differences from Linux](https://wiki.termux.com/wiki/Differences_from_Linux) — Termux 与 Linux 的差异
5. [Termux Wiki: Package Management](https://wiki.termux.com/wiki/Package_Management) — Termux 包管理
6. [termux-packages build scripts](https://gitlab.com/termux-mirror/termux-packages) — Termux 包构建脚本
7. [Monero RandomX Specification](https://github.com/tevador/RandomX) — RandomX 算法规范
8. [Monero Guide: RandomX Mining](https://moneroguide.com/2026/03/25/how-to-mine-monero-in-2026-cpu-mining-with-randomx-xmrig-setup/) — RandomX 挖矿指南
9. [Stratum Protocol Reference](https://d-central.tech/data/stratum-protocol-reference/) — Stratum 协议参考
10. [Android NDK Guides](https://developer.android.google.cn/ndk/guides) — Android NDK 官方指南
