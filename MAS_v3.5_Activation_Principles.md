# Microsoft Activation Scripts (MAS) v3.5 系统激活原理总结

> 日期：2026-09-20
>
> 项目地址：[https://github.com/massgravel/Microsoft-Activation-Scripts](https://github.com/massgravel/Microsoft-Activation-Scripts)
>
> 官方文档：[https://massgrave.dev](https://massgrave.dev)

---

## 一、MAS 概述

**Microsoft Activation Scripts (MAS)** 是一个开源的 Windows 和 Office 激活工具，使用批处理脚本（PowerShell / CMD）实现，代码完全开源，可供社区审计。MAS v3.5 提供了以下五种激活方法：

| 方法 | 适用产品 | 激活时效 | 是否需要网络 |
|------|---------|---------|-------------|
| **HWID** | Windows 10/11 | 永久 | 首次需要 |
| **Ohook** | Microsoft Office | 永久 | 不需要 |
| **KMS38** | Windows 10/11/Server | 至 2038 年 | 不需要 |
| **Online KMS** | Windows / Office | 180 天（自动续期） | 需要 |
| **TSforge** | Windows / Office（几乎所有版本） | 永久 | 不需要 |

---

## 二、各激活方法技术原理

### 1. HWID（Hardware ID）激活

#### 核心原理

HWID 激活利用了微软在 Windows 10 免费升级期间建立的**数字许可证（Digital License）** 机制，将激活状态与设备的硬件指纹绑定。

#### 技术流程

```
硬件信息采集 → 生成硬件哈希（Hardware Hash）→ 构造 GenuineTicket.xml
→ 提交至微软激活服务器 → 服务器端记录数字许可证 → 激活完成
```

**关键技术细节：**

- **硬件哈希（Hardware Hash）**：基于约 8 个硬件组件的标识符（CPU、内存、主板、磁盘等）组合生成的唯一指纹值。
- **GenuineTicket.xml**：一个经数字签名的 XML 格式激活票据，包含设备的硬件配置信息。存储路径为：
  ```
  %ProgramData%\Microsoft\Windows\ClipSVC\GenuineTicket\
  ```
- **ClipSVC（Client License Platform Service）**：Windows 客户端许可证平台服务，负责处理数字许可证的验证和管理。

#### 激活特性

- **永久有效**：激活信息存储在微软服务器端，与硬件 ID 绑定
- **重装免激活**：只要硬件配置未发生重大变化（尤其是主板），重装系统后自动激活
- **硬件变更影响**：更换主板等关键硬件可能导致激活失效

---

### 2. Ohook 激活

#### 核心原理

Ohook 通过 **DLL 劫持（DLL Hijacking / DLL Proxying）** 技术，拦截 Office 的许可证验证调用，使 Office 认为自身已被合法激活。

#### 技术流程

```
放置自定义 sppc.dll 至 Office 安装目录
→ Office 启动时优先加载本地 sppc.dll（DLL 搜索顺序）
→ 拦截许可证验证相关 API 调用 → 返回"已激活"状态
→ 其余非激活相关调用 → 透传至系统原始 sppc.dll
```

**关键技术细节：**

- **sppc.dll（Software Protection Platform Client）**：Windows 软件保护平台的客户端 DLL，负责软件许可证验证。
- **DLL 搜索顺序利用**：Windows 在加载 DLL 时优先搜索应用程序所在目录，Ohook 利用此机制在 Office 安装目录放置自定义 `sppc.dll`，使 Office 优先加载它而非 `System32` 下的系统版本。
- **代理模式（Proxy）**：自定义 DLL 并非完全替代原始 DLL，而是将非激活相关的合法调用透明转发给系统原始 `sppc.dll`，仅拦截和修改激活验证相关的调用。

#### 激活特性

- **不修改系统文件**：仅在 Office 安装目录放置额外 DLL，不修改 `System32` 下的文件
- **Office 更新兼容**：可在 Office 更新后继续生效
- **适用范围**：支持 Microsoft 365、Office LTSC 等版本

---

### 3. KMS38 激活

#### 核心原理

KMS38 利用 Windows 的 **KMS（Key Management Service）** 企业激活协议，在本地生成一个有效期至 **2038 年 1 月 19 日**的 KMS 激活票据。

#### 技术流程

```
安装 GVLK（通用批量许可证密钥）
→ 在本地构造 KMS 激活票据（ClipSVC ticket）
→ 将票据的到期时间设定为 2038 年
→ 票据写入本地存储 → 激活完成
```

**关键技术细节：**

- **GVLK（Generic Volume License Key）**：微软公开发布的通用批量许可证密钥，用于标识 Windows 版本以便 KMS 激活，本身不包含激活权限。
- **2038 年到期**：日期上限与 Unix 时间戳的 32 位整数溢出（Y2K38 问题）相关，`2^31 - 1` 秒对应的 UTC 时间为 2038 年 1 月 19 日 03:14:07。
- **本地票据**：与传统 KMS 不同，KMS38 不需要外部 KMS 服务器，所有票据在本地生成并存储。

#### 激活特性

- **长期有效**：持续至 2038 年，接近永久
- **无需网络**：一次性本地激活，不需要定期联网续期
- **离线运作**：整个激活过程可在断网环境下完成

---

### 4. Online KMS 激活

#### 核心原理

Online KMS 模拟了企业环境中的 **KMS 激活流程**，通过连接互联网上的第三方 KMS 模拟服务器获取 180 天的激活授权，并通过计划任务自动续期。

#### 技术流程

```
安装 GVLK → 配置 KMS 服务器地址（指向第三方 KMS 模拟服务器）
→ 发送激活请求 → 服务器返回 180 天有效期的激活响应
→ 创建计划任务（每 7 天自动续期）→ 激活完成
```

**关键技术细节：**

- **KMS 协议**：微软为企业批量授权设计的密钥管理服务协议，允许组织在内部运行 KMS 服务器来集中管理和激活客户端设备。
- **GVLK**：与 KMS38 相同，使用微软公开的通用批量许可证密钥。
- **180 天激活周期**：KMS 协议规定客户端激活有效期为 180 天，客户端默认每 7 天尝试续期一次。
- **自动续期**：MAS 会创建系统计划任务，定期向 KMS 服务器发起续期请求，重置 180 天计时器。

#### 激活特性

- **需要网络连接**：依赖外部 KMS 服务器进行续期
- **非永久**：如果 KMS 服务器不可达超过 180 天，激活将过期
- **广泛兼容**：可同时激活 Windows 和 Office

---

### 5. TSforge 激活

#### 核心原理

TSforge 是 MAS 2.8 版本（2025 年 2 月）引入的新方法。它**直接修改 SPP（Software Protection Platform）的受信任存储（Trusted Store）数据**，在底层改写许可证验证数据，使系统认为产品已合法激活。

#### 技术流程

```
分析目标产品的 SPP 数据结构
→ 定位 Trusted Store 存储文件
→ 构造合法激活状态所需的数据
→ 直接写入 SPP 数据存储
→ SPP 服务重新加载数据 → 识别为已激活
```

**关键技术细节：**

- **SPP（Software Protection Platform）**：Windows 的软件保护平台，是微软所有产品许可证管理的核心组件，负责存储和验证激活状态。
- **Trusted Store（受信任存储）**：SPP 内部用于持久化激活信息的安全数据库，存储了产品密钥、激活状态、许可证令牌等关键数据。
- **关键存储文件**：
  - `data.dat`：SPP 物理存储文件，包含激活相关数据（产品密钥、激活状态、受信任存储条目）
  - `tokens.dat`：包含许可证令牌信息，如许可证证书和策略
- **直接写入**：TSforge 绕过了 SPP 的常规 API 接口，直接在存储层面操作数据，本质上是在系统底层"伪造"合法的激活记录。

#### 激活特性

- **覆盖范围最广**：可激活几乎所有微软产品，包括各版本 Windows（从 Windows Vista 到 Windows 11）和 Office
- **永久激活**：修改后的数据持久存储
- **无需网络**：完全本地操作
- **技术深度最高**：需要对 SPP 内部数据结构有深入了解

---

## 三、各方法技术对比

| 特性 | HWID | Ohook | KMS38 | Online KMS | TSforge |
|------|------|-------|-------|-----------|---------|
| **技术层面** | 服务器端数字许可证 | DLL 劫持/代理 | 本地 KMS 票据 | 远程 KMS 协议 | SPP 数据存储修改 |
| **适用产品** | Windows 10/11 | Office | Windows 10/11/Server | Windows + Office | 几乎所有版本 |
| **持续时间** | 永久 | 永久 | 至 2038 年 | 180 天（需续期） | 永久 |
| **网络需求** | 首次需要 | 不需要 | 不需要 | 持续需要 | 不需要 |
| **系统文件修改** | 否 | 是（Office 目录） | 否 | 否 | 是（SPP 存储文件） |
| **重装保持** | 是（硬件不变） | 否 | 否 | 否 | 否 |
| **复杂度** | 中 | 中 | 中 | 低 | 高 |

---

## 四、涉及的核心 Windows 组件

### 4.1 SPP（Software Protection Platform）
Windows 软件保护平台，是所有微软产品许可证管理的统一框架。包含以下子组件：
- **sppsvc.exe**：SPP 服务进程
- **sppc.dll**：SPP 客户端 DLL，提供激活验证 API
- **Trusted Store**：受信任数据存储（`data.dat`、`tokens.dat`）

### 4.2 ClipSVC（Client License Platform Service）
客户端许可证平台服务，Windows 10 及更高版本中负责处理数字许可证和现代激活方式（如 HWID、KMS38）。

### 4.3 GVLK（Generic Volume License Key）
微软为企业批量授权公开发布的通用密钥，用于标识产品版本。KMS38 和 Online KMS 方法均依赖此密钥来启动 KMS 激活流程。

---

## 五、参考信息

- 项目仓库：[massgravel/Microsoft-Activation-Scripts](https://github.com/massgravel/Microsoft-Activation-Scripts)
- 官方文档：[massgrave.dev](https://massgrave.dev)
- TSforge 技术文档：[massgrave.dev/tsforge](https://massgrave.dev/tsforge)
- TSforge 发布博客：[massgrave.dev/blog/tsforge](https://massgrave.dev/blog/tsforge)

---

> **声明**：本文档仅用于技术学习和研究目的，旨在理解 Windows/Office 激活机制的技术原理。使用相关工具应遵守所在地区的法律法规和微软的软件许可协议。
