# XFD（xfd-model-lab / 小fa模型锁）平台逻辑解析

> 本文档是对 XFD 这一 VTube Studio 模型锁平台（品牌曾用名 `模型启动器` / `小fa朵`，当前启动器名 `小fa模型锁.exe`，资源目录 `xfd-model-lab`）的**整体技术架构与保护逻辑**的解析。只描述 XFD 自身，不涉及任何第三方工具。内容基于对其发布二进制、`app.asar` 解包内容、运行时行为、系统痕迹、以及安装目录结构的静态与动态观察。

---

## 1. 定位与形态

XFD 是一款面向 VTube Studio（下称 VTS）的**模型分发与授权运行时**。作者以"模型锁"的产品形态售卖：模型作者用它封装模型文件，终端使用者用它按授权条件解锁并加载到 VTS 中。

- **技术形态**：Electron 桌面应用（Chromium + Node.js 运行时）。
- **典型安装目录**：`<某盘>:\<品牌目录>\xfd-model-lab\`（中文用户常见为 `C:\模型启动器\xfd-model-lab\`），或 `%LOCALAPPDATA%\Programs\xfd-model-lab\`。
- **典型启动器名**：`小fa模型锁.exe`（早期为 `xfd-model-lab.exe`，个别版本为 `模型启动器.exe`）。
- **典型运行态**：启动器（Electron 主进程）+ 若干 Chromium 子进程（`GPU Process`、`Renderer`、`Utility` 等）+ 一个常驻 Dokan 客户端 + 一个（或多个）Windows 后台服务。

---

## 2. 应用架构

### 2.1 磁盘布局

```
<install>\
├── 小fa模型锁.exe                        启动器（Electron 主 PE）
├── *.dll / chrome_*.pak / v8_context*.bin / icudtl.dat ...   Chromium 运行时
├── ffmpeg.dll / d3dcompiler_47.dll      多媒体 / D3D 支撑
├── LICENSES.chromium.html
└── resources\
    ├── app.asar                         业务逻辑打包（~180 MB）
    ├── app.asar.unpacked\               electron-builder 标记 asarUnpack 的文件
    │   └── resources\
    │       └── ex\
    │           ├── DokanSetup.exe       Dokan 驱动静默安装器
    │           ├── model-mfs.exe        Dokan 客户端（~3.6 MB）
    │           ├── ApiHookCheck.dll     自研反注入检测原生插件
    │           └── ahc.dll              ApiHookCheck 的同源 DLL
    └── electron.asar (或同类)            Electron 框架内置资源
```

### 2.2 进程模型

Electron 标准两层结构：

- **Main process**：`小fa模型锁.exe` 的主线程执行 Node.js 代码，持有 IPC 枢纽、文件系统、`child_process`、本地设置、网络、电源事件、菜单/窗口管理。
- **Preload**：通过 `contextBridge` 暴露受控 API 给 Renderer。
- **Renderer**：Vue 前端，驻留在 Chromium 子进程里，负责 UI 与用户交互。
- **Utility / GPU / Networking 子进程**：Chromium 默认进程拆分。

IPC 通道（由 `app.asar/out/main` 中的 `ipcMain.handle` 注册）观察到的关键 channel：


| Channel 名                       | 作用（推测）                      |
| ------------------------------- | --------------------------- |
| `ipc:md5_check_success`         | 将文件 MD5 校验通过的结果下发到 Renderer |
| `sendMd5CheckSuccessToRenderer` | 同上，内部发送函数名                  |
| `modelDetected`                 | 通知 Renderer 某模型被识别/启用       |


### 2.3 运行时依赖（`package.json` 核心项）


| 包                               | 作用                                  |
| ------------------------------- | ----------------------------------- |
| `electron` + `electron-updater` | 运行时 + 自升级                           |
| `electron-store`                | 本地持久化（被 DPAPI safeStorage 包装）       |
| `node-machine-id`               | HWID 的 MachineGuid 来源               |
| `node-wmi`                      | 进程 / 系统信息枚举                         |
| `get-installed-apps`            | 枚举已安装应用（被用于黑白名单查询）                  |
| `md5-file`                      | 计算任意文件 MD5                          |
| `chokidar`                      | 文件系统 Watcher（安装目录 / 资源目录自我监控）       |
| `axios`                         | 与授权/上报服务器的 HTTPS 通信                 |
| `koffi`                         | FFI（`ApiHookCheck.dll` 通过 koffi 载入） |


### 2.4 字节码打包

`app.asar/out/main` 下业务代码以 **V8 bytecode cache (`index.jsc`)** 形态分发，而非普通 `.js`。`node-machine-id`、部分 `node_modules` 仍保留为 `.js` 明文，因为 V8 bytecode 与 Node 版本强耦合，作者选择只把自有业务代码编成 bytecode 以加大静态分析成本。

> **副作用**：字节码化只是提高了逆向门槛，并未改变字符串常量的**可 grep 性**——所有字符串常量、函数名、IPC 通道名仍然以明文形式存在于 bytecode blob 中。

---

## 3. 模型保护层（Dokan 虚拟文件系统 + PID 白名单）

### 3.1 组件

1. **Dokan 用户态文件系统框架**（第三方开源项目）—— XFD 通过 `DokanSetup.exe` 在首次运行时静默安装。
2. `**model-mfs.exe`**（Model Mount File System）—— XFD 自研的 Dokan 客户端，负责：
  - 调用 Dokan 用户态 API 挂载一块虚拟卷；
  - 在该卷的 I/O 回调里根据请求来源 **PID** 做白名单过滤；
  - 把受保护的模型资源放在这块卷内，通过白名单 PID 提供给 VTS 读取；
  - 作为常驻后台进程运行，通常独立于启动器主进程，由一个专用 Windows 服务启动（见 §6）。

### 3.2 工作流

```
首次运行 XFD
  → silentInstallDokan2()       安装 Dokan2 驱动（若未安装）
  → initDokan2Check()            检查驱动就绪状态、版本、服务
  → model-mfs.exe 启动
  → Dokan 挂载虚拟卷（盘符或目录挂载点）
  → 模型资源仅在该卷中可见
  → Dokan I/O 回调按 PID 过滤读写请求
```

### 3.3 PID 白名单策略

`model-mfs.exe` 维护一份"允许读该卷"的 PID 集合。典型填充过程：

1. `**findVtsExePath**` 定位 VTS 可执行文件路径；
2. 启动器在 VTS 启动后通过进程事件或轮询拿到 VTS 的 PID；
3. 把该 PID 追加进 `model-mfs.exe` 的白名单；
4. Dokan 回调里 `GetProcessId(requestorHandle)` 与白名单比对，不在就返回 `STATUS_ACCESS_DENIED`。

**对外宣传**：`只有 VTube Studio 能读这块盘，因此模型受保护`。

### 3.4 结构性弱点（威胁模型分析）

`PID` 在 Windows 上是**进程表里的一个临时数字**，不是身份凭证，也不与代码来源绑定。这带来两类根本性局限：

1. **同源进程内读盘**：任何能在 VTS 进程空间内执行代码的逻辑（插件接口被滥用、利用 VTS 的 IPC 暴面、在 VTS 进程中注入等）都以"VTS 的 PID"发起文件 I/O。从 Dokan 用户态 Filter 的视角看不出来源是否为"官方逻辑"，直接放行。
2. **PID 复用与绑定问题**：进程退出后 PID 可能被系统复用。若白名单策略未与进程对象（`HANDLE`）、映像签名、会话/Job 生命周期做严格绑定，就可能出现跨进程误放行或状态错乱。

> 这些是 PID 白名单这一模型本身的**结构性**问题，而非 Dokan 框架的问题。Dokan 是成熟、合法、合理的用户态文件系统框架；被过度宣传的是"仅凭 PID 白名单就视作安全"这一**使用方式**。

---

## 4. 反滥用检测层

XFD 客户端侧持续监控宿主系统，试图在用户使用"绕过工具"时主动干预。观察到的主要手段：

### 4.1 MD5 黑名单

在 `index.jsc` 中可提取到以下字符串 / 函数：

```
verifyFileMd5            --- 对指定文件计算 MD5
md5File                  --- node md5-file 模块的调用入口
sendMd5CheckSuccessToRenderer
ipc:md5_check_success
```

工作流：

```
[周期循环]
  tasklist /FI "IMAGENAME eq <name>"   枚举候选进程
  → 对候选的 ExecutablePath 计算 MD5
  → 与本地（或服务端下发的）黑名单比对
  → 命中 → 触发应对动作
```

若 `wmic` 存在则优先 `queryByWmic`（`Win32_Process` WMI 查询），`isWmicAvailable()` 为 false 时回退 `tasklist`。

### 4.2 原生反注入检测

`ApiHookCheck.dll` / `ahc.dll` 是自研原生插件，通过 `koffi` 在主进程里 LoadLibrary。功能推测：

- 扫描当前进程关键系统 API 的入口是否被覆写（`NtQueryInformationProcess` / `Create*` / `LdrLoadDll` 等）；
- 枚举已加载模块，识别已知 Hook 引擎（Detours / MinHook 样式）；
- 结果上报给 Main process 的 JS 逻辑，作为"当前环境可信度"因子。

### 4.3 进程主动终止

`taskkill /F /PID` 调用在业务逻辑中直接出现，说明客户端会在检测命中时**主动 kill 目标进程**（或反过来自我终止以避免信息被进一步读出）。

### 4.4 文件系统 Watcher

通过 `chokidar` 监视：

- 自身安装目录（`resources\`）下的写入事件；
- `%APPDATA%\xfd-model-lab\*.json` 的异常改动；
- 可能还监视 VTS 的 `StreamingAssets` 目录。

监视命中后的行为推测为：标记异常 + 重新验证 + 上报服务端。

### 4.5 服务端状态码

`FORBIDDEN` 是在客户端常量池中观察到的服务端响应标记。推测在授权验证、心跳、上报接口中用作"封禁/强制下线"语义。

---

## 5. 身份与授权层

### 5.1 多源 HWID 合成

XFD 不依赖单一指纹。`%APPDATA%\xfd-model-lab\config.json` 中观察到以三段 base64（`djEw` = `v10` 前缀为 Electron `safeStorage` DPAPI 封装）存储的三类本地缓存：


| 键名               | 内容（推测）                    | 来源                                                      |
| ---------------- | ------------------------- | ------------------------------------------------------- |
| `..._driveInfo`  | 系统盘序列号 / 型号               | `node-wmi` 查询 `Win32_DiskDrive` / `Win32_PhysicalMedia` |
| `..._dokan2Info` | Dokan2 驱动版本 / 安装路径 / 服务状态 | `initDokan2Check` 等自有查询                                 |
| `..._vtsInfo`    | VTS 安装路径 / 版本 / 关键文件哈希    | `findVtsExePath` 结果再加工                                  |


键名前缀为一个 19 位十进制整数（例如 `2034281490028179457`），是本地生成的复合设备哈希（疑似 `SHA-xxx(driveInfo + machineGuid + cpuId)` 截取）。

**机器 GUID** 来自 `node-machine-id`：

```js
// node-machine-id 内部
reg query HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Cryptography /v MachineGuid
```

然后在上层做 SHA-256，得到 `machineId`。

### 5.2 授权令牌

`%APPDATA%\xfd-model-lab\app-security.json`（~1 KB，DPAPI 加密的二进制 blob）推测存储：

- 登录会话 token；
- 授权态缓存（含过期时间）；
- 可能还包含上次心跳的服务器 nonce。

清空该文件会迫使客户端下次启动时重新走登录流程。

### 5.3 服务端绑定

登录 / 注册时，客户端将上述多源 HWID 一次性上报，服务端记录：

```
account_id <-> (machineId, driveInfo_hash, dokan2Info_hash, vtsInfo_hash)
```

后续每次启动做一致性校验：

- `machineId` 与历史绑定不一致 → 触发再验证或直接 `FORBIDDEN`；
- `driveInfo` 异常（硬盘变更）但 `machineId` 一致 → 允许但标记；
- 多项同时变更 → 视同换机，按策略处理。

> 关键点：即便客户端本地的 `machineId` 被伪造，`driveInfo`（硬盘序列号）仍由 WMI 直接读取真实硬件。服务端可按"硬盘序列号重合"识别同一物理机不同伪造身份。

---

## 6. 后台服务与常驻进程

### 6.1 Windows 服务

观察到至少一项 Windows 服务相关命名（由 XFD 在安装/首次运行时注册）：

- `VtsSecureService`（主服务名）
- `VtsSecure`（可能的简写/别名）

作用推测：

- 以 `LocalSystem` 权限常驻，负责 Dokan 卷的生命周期管理；
- 监控 `model-mfs.exe` 存活，掉线后自动重启；
- 承载一个 `VTSLogMonitor` 子逻辑，观察 VTS 的日志 / 调试端口；
- 在用户未启动 GUI 的情况下维持保护状态。

### 6.2 常驻用户态进程

- `model-mfs.exe`：由服务或启动器拉起，挂载 Dokan 卷并处理 I/O。
- `小fa模型锁.exe`（Electron 主）及其 Chromium 子进程：GUI 在前台时存在，关闭 GUI 通常也会退出（除非作者设置了托盘常驻）。

### 6.3 自启动

Electron 层通过 `electron-updater` 支持自动更新；服务层通过 Windows Service Manager 的 `Auto-Start` 标志保证开机常驻。

---

## 7. 服务端交互

### 7.1 通信方式

- HTTPS（`axios`），推测使用 JSON payload。
- 客户端不主动暴露 HTTP 端口（不做 P2P），纯 C/S。

### 7.2 推测的端点类别


| 端点类别      | 触发时机                                      | 载荷要素                                       |
| --------- | ----------------------------------------- | ------------------------------------------ |
| 注册 / 登录   | 首次启动、token 过期                             | account_id, password/code, 多源 HWID         |
| 心跳 / 状态   | 周期性（秒—分钟级）                                | machineId, process env fingerprint, app 版本 |
| MD5 黑名单下发 | 登录后、周期刷新                                  | 服务端 → 客户端                                  |
| 检测命中上报    | `taskkill` / `ApiHookCheck` / chokidar 事件 | 命中文件/进程特征、时间戳、HWID                         |
| 模型授权验证    | 打开受保护模型                                   | modelId, HWID, 时间戳                         |
| 封禁 / 强制下线 | 服务端决策                                     | `FORBIDDEN` 状态码                            |


### 7.3 响应语义

- `FORBIDDEN` — 全局禁止；客户端接收后立即清空会话并退出。
- 其他常规 HTTP 2xx/4xx/5xx — 正常 RESTful 语义。

---

## 8. 分发与更新

### 8.1 打包与安装

- 作者侧用 **electron-builder** 打包；NSIS 安装器生成。
- 中文用户常见安装位置：`C:\模型启动器\xfd-model-lab\` 或 `D:\模型启动器\xfd-model-lab\`。
- 非标准安装也常见 `%LOCALAPPDATA%\Programs\xfd-model-lab\`。

### 8.2 自动更新

- `electron-updater` 走作者的更新服务器。
- 更新缓存：`%LOCALAPPDATA%\xfd-model-lab-updater\`。
- 客户端缓存：`%LOCALAPPDATA%\xfd-model-lab\`（Electron `userData`，含 `Local Storage` / `IndexedDB` / `Session Storage` / `Service Worker` / `Cache` 等标准子目录）。
- 配置与加密数据：`%APPDATA%\xfd-model-lab\`（含 `config.json` / `app-security.json`）。

> `%LOCALAPPDATA%\xfd-model-lab` 与 `%APPDATA%\xfd-model-lab` 是两个不同目录——一个是 Chromium 的 `userData`，另一个是 Roaming 下的业务配置，容易混淆。

---

## 9. 威胁模型与结构性弱点（整理）

将 §3 / §4 / §5 的内容合并到一个威胁模型视角：

### 9.1 对"外部工具扫描"能扛什么


| 对手能力                         | XFD 的应对                 | 有效性                           |
| ---------------------------- | ----------------------- | ----------------------------- |
| 从磁盘读 `app.asar`              | 静态明文 + V8 bytecode 部分模糊 | 仅延缓                           |
| 对受保护模型做"别的进程读文件"             | Dokan PID 白名单           | 有效（对来源为不同 PID 的外部扫描）          |
| 枚举安装目录下的 `app.asar.unpacked` | 直接文件系统读，无 Dokan         | **无任何保护**                     |
| 黑名单命中时自我保护                   | MD5 + taskkill          | 对具名工具**有效**，对"同一能力的新包装"**无效** |


### 9.2 对"同源进程内读盘"能扛什么


| 威胁                             | 应对                          | 有效性 |
| ------------------------------ | --------------------------- | --- |
| 在 VTS 进程内枚举并拷贝 StreamingAssets | `ApiHookCheck` 对 VTS 进程无可见性 | 无   |
| 插件滥用读取 Live2D 模型               | 同上                          | 无   |


**核心结论**：XFD 的保护链条的"底气"在 §3（Dokan + PID），但 PID 白名单无法区分"同一进程内的不同代码段"。如果攻击面已经在 VTS 进程里，保护失效。

### 9.3 对"HWID 绕过"能扛什么


| 绕过手段                                                  | 应对                 | 有效性            |
| ----------------------------------------------------- | ------------------ | -------------- |
| 改写 `HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid` | `driveInfo` 冗余校验   | 部分缓解           |
| Patch `node-machine-id` 返回假值                          | 同上，且未对客户端完整性做服务端重验 | 仅换号有效，同机被认出    |
| 换机/换盘                                                 | 服务端会识别为新设备         | 按换机处理（可能引入新成本） |


### 9.4 总体评价

- XFD 的检测层**功能完整**：MD5 黑名单 + 原生 Hook 检测 + FS Watcher + 服务端联动，覆盖了静态 + 动态两个面。
- XFD 的保护层**模型有硬伤**：PID 白名单不具备区分"同一进程内合法与非法调用"的能力，Dokan 框架本身再可靠也无法弥补。
- XFD 的身份层**冗余得当**：多源 HWID + 服务端绑定 + DPAPI 本地加密，单点伪造不足以让服务端认可为"新设备"。

---

## 10. 附录：观察到的关键字符串索引

从 `app.asar/out/main/index.jsc` 中可通过 `strings` 抽取的代表性常量（未穷举）：

```
sendMd5CheckSuccessToRenderer
verifyFileMd5
md5File
ipc:md5_check_success
modelDetected
tasklist /FI "IMAGENAME eq ...
taskkill /F /PID
isWmicAvailable
queryByWmic
VtsSecureService
VTSLogMonitor
findVtsExePath
silentInstallDokan2
isDokan2Installed
initDokan2Check
globalInjection
FORBIDDEN
```

以及在 `%APPDATA%\xfd-model-lab\config.json` 中可观察到的键模式：

```
<19位十进制>_driveInfo
<19位十进制>_dokan2Info
<19位十进制>_vtsInfo
```

---

*本文档基于对 XFD 公开发布版本的静态与动态观察整理，用于架构理解、威胁建模与防御/合规讨论的参考。*