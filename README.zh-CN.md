# Serial Studio GPL 完整功能版

[English](README.md) | **中文**

把上游 [Serial-Studio](https://github.com/Serial-Studio/Serial-Studio) v3.2.7 GPL 编译开关后面挡住的功能全部解锁，得到一个**和 Pro 版界面一样、不带 14 天试用提醒**的 GPLv3 二进制。

> ⚠️ **法律声明**：本仓库的修改都基于上游已公开的 GPL 源代码，**没有逆向工程任何商业组件**，没有破解 license。它的工作原理是把那些"功能代码本来就在 GPL 仓库里、只是被 `#ifdef BUILD_COMMERCIAL` 编译挡住"的部分重新打开。详见下文「实现原理」。

---

## 这个 fork 解锁了什么

| 分类 | 功能 | 说明 |
|---|---|---|
| **IO 驱动** | UART/COM | 上游 GPL 本来就有 |
| | 网络套接字（TCP/UDP）| 上游 GPL 本来就有 |
| | 蓝牙 LE | 上游 GPL 本来就有 |
| | **音频输入** | ✅ 解锁，用 miniaudio 库 |
| | **CAN Bus** | ✅ 解锁，Qt6::SerialBus |
| | **Modbus** | ✅ 解锁，Qt6::SerialBus |
| | **USB** raw transfer | ✅ 解锁，libusb |
| | **HID 设备** | ✅ 解锁，hidapi |
| | **Process**（外部进程 stdin/out） | ✅ 解锁 |
| **数据协议** | **MQTT** 订阅/发布 | ✅ 解锁，Qt6::Mqtt（开源 LGPL 模块） |
| | Binary Direct 模式 | 上游 GPL 本来就有，文档强调下 |
| **可视化 widget** | UART/Plot/MultiPlot/Gauge/Bar/Compass/Terminal/FFT/Accelerometer/Gyroscope/GPS 等 | 上游 GPL 都有 |
| | **3D Plot**（Qt Quick3D 散点轨迹） | ✅ 解锁 |
| | **Image View**（实时图像帧）| ✅ 解锁（zip 导出仍 Pro-only） |
| | **XY Plot**（相图，dataset 选 X 轴源） | ✅ 解锁 |
| **数据导入** | **DBC 文件导入**（CAN 数据库）| ✅ 解锁 |
| **数据导出** | CSV 文件 | 上游 GPL 本来就有 |
| | **控制台导出到 .txt** | ✅ 我们写了 GPL 实现（上游是 Pro stub） |

## 仍然是 Pro-only 的功能

这些功能的**实际代码不在 GPL 源码里**，只有提示弹窗或 stub。要做就得从零写：

- **MDF4 文件导出**（写 .mf4 测量数据格式，依赖 `lib/mdflib`）
- **ImageView 的 zip 会话录制**（依赖 QuaZip 商业链接）
- **License 管理 UI / Trial 倒计时 / Lemon Squeezy 激活**
- **AI 助手**（只在 upstream master 分支，v3.2.7 没有）
- **Waterfall 频谱图 / Painter 自定义 Canvas2D widget**（同上）
- **JSON-RPC API server 中的解锁驱动命令**（API 注册逻辑还在 `#ifdef`，GUI 能用 driver 但外部 API 调不到对应命令）

---

## 下载

去 [Releases](https://github.com/VALM-Labs/Serial-Studio-GPL-Full/releases) 拿。**推荐 v3.2.7.5**（最新，含 Plot 3D 完整修复）。

每个 Release 三件套：
- `Serial-Studio-GPL3-v3.2.7.X.msi` — Windows 安装包（约 154MB）
- `Serial-Studio-v3.2.7.X-src.zip` — 已 patch 的完整源码（约 44MB）
- `0001-gpl-full-unlock.patch` — `git am` 格式的单 patch 文件（约 68KB）

## 安装

双击 MSI 一路下一步。装到 `C:\Program Files\Serial Studio GPLv3\`，独立目录，**和原版 Pro 试用版互不干扰**，可共存。

---

## 版本历史

### v3.2.7.5（推荐）

修 Plot 3D 数据 pipeline——之前 widget 容器有了但 3D 内容不渲染。

- `Dashboard.cpp`: 解锁 `configurePlot3DSeries()` / `m_plotData3D` 写循环 / `registerXAxisIfNeeded()`
- `DashboardWidget.cpp`: 解锁 `new Widgets::Plot3D()` 实例化的 switch case
- `DSP.h`: 解锁 `LineSeries3D` typedef
- 实测 EM Wave Simulator 示例 3D 散点完整渲染

### v3.2.7.3

补加 Plot 3D / Image View / DBC Import / XY Plot / Binary Mode 文档。

- `SerialStudio.h`: `BusType` enum 暴露 6 个 Pro 值；`DashboardWidget` enum 暴露 `DashboardImageView`
- `ModuleManager.cpp`: `qmlRegisterType<Widgets::Plot3D>` / `Cpp_JSON_DBCImporter` / `Cpp_IO_Audio`...`Cpp_IO_Process` / `Cpp_MQTT_Client` context property 暴露给 GPL
- `app/CMakeLists.txt`: Plot3D / ImageView / ImageProvider / DBCImporter / MQTT/Client 加到 GPL `SOURCES`
- `Plot.cpp`: 删 `xAxisId` 的 license 检查
- `ImageView.h/.cpp / ImageProvider.h/.cpp`: 去外层 `#ifdef BUILD_COMMERCIAL`，stub 化 ImageExport 调用

### v3.2.7.1

第一版解锁：driver / MQTT / Console Export。

- IO 驱动 `Audio/CANBus/Modbus/HID/USB/Process` 的 `.cpp/.h` 加到 GPL SOURCES，去外层 `#ifdef`
- USB.cpp 加 `winsock2.h` include 顺序修正
- ConnectionManager 全面去 `#ifdef`（保留 MQTT/Licensing 那块）
- Hardware.qml 6 个 driver `Loader active: Cpp_CommercialBuild` → `active: true`
- CANBus.qml 删未注册的 `DBCPreviewDialog` 引用
- MQTT::Client::cpp 把 license 检查包进 `#ifdef BUILD_COMMERCIAL`，GPL 走 unrestricted 路径
- main.qml `proVersion` 在 GPL 也返回 true（MQTT 控件 enabled）
- Toolbar.qml MQTT 按钮 Loader 解锁
- `Console::Export` 写一份 GPL 实现：同步、每设备一个 `.txt`、命名规则跟 Pro 一致
- `app/CMakeLists.txt`: 加 `Qt6::SerialBus` + `Qt6::Mqtt` 到 GPL link、加 hidapi + libusb 预编译二进制链接、加 `WIN32_LEAN_AND_MEAN/NOMINMAX/_WINSOCKAPI_` 解决 winsock 冲突、`install(FILES)` 把 hidapi/libusb DLL 打进 MSI

---

## 编译（Windows / MSVC）

### 依赖

| 工具 | 版本 | 备注 |
|---|---|---|
| Visual Studio 2022 Build Tools | C++ workload (MSVC 14.44+) | `winget install Microsoft.VisualStudio.2022.BuildTools` |
| CMake | 3.30+ | `winget install Kitware.CMake` |
| Qt | 6.9.2 `msvc2022_64` | 用 `aqtinstall` 装，需要 `qtbase qtdeclarative qtsvg qttools qtcharts qtserialport qtconnectivity qtwebchannel qtwebsockets qtpositioning qtimageformats qtmultimedia qtshadertools qt5compat qtgraphs qtquick3d qtwebview qtserialbus qtwebengine qtpdf` 这堆模块 |
| Qt MQTT | 6.9.2 | 不在标准 Qt installer，从 `https://github.com/qt/qtmqtt.git` v6.9.2 自己 cmake 编译 install 到 Qt 前缀 |
| WiX Toolset | v3.14 | 打 MSI 用 |
| hidapi | v0.14.0 | [Windows 预编译](https://github.com/libusb/hidapi/releases/tag/hidapi-0.14.0) zip 解压 |
| libusb | v1.0.27 | [Windows 预编译](https://github.com/libusb/libusb/releases/tag/v1.0.27) 7z 解压 |

NJU 镜像 `https://mirror.nju.edu.cn/qt/` 给国内装 Qt 比较快（不过它用 SHA1 校验，要在 `aqt_settings.ini` 加 `hash_algorithm: sha1`）。

### Qt MQTT 自编（不在标准 installer）

```bash
git clone --depth 1 --branch v6.9.2 https://github.com/qt/qtmqtt.git
cd qtmqtt
cmake -S . -B build -G Ninja ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_PREFIX_PATH=D:/Qt/6.9.2/msvc2022_64 ^
  -DCMAKE_INSTALL_PREFIX=D:/Qt/6.9.2/msvc2022_64
cmake --build build --config Release
cmake --install build --config Release
```

### 主项目配置 & 编译

```bash
cmake -S . -B build -G Ninja ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_PREFIX_PATH=D:/Qt/6.9.2/msvc2022_64 ^
  -DBUILD_GPL3=ON ^
  -DPRODUCTION_OPTIMIZATION=OFF ^
  -DCPACK_WIX_ROOT=D:/BuildTools/wix3 ^
  -DEXTLIBS_ROOT=D:/BuildTools/extlibs
cmake --build build --config Release
cmake --build build --config Release --target package
```

输出：`build/Serial-Studio-GPL3-v3.2.7.5.msi`

### 直接打 patch

或者更省事——`git am` 应用 patch 文件：

```bash
git clone --branch v3.2.7 --depth 1 https://github.com/Serial-Studio/Serial-Studio.git
cd Serial-Studio
curl -L -O https://raw.githubusercontent.com/VALM-Labs/Serial-Studio-GPL-Full/main/0001-gpl-full-unlock.patch
git apply 0001-gpl-full-unlock.patch
# 然后按上面的 cmake 流程编译
```

---

## 实现原理（为啥这事是合法的）

上游 Serial-Studio 是双协议（GPLv3 / Commercial）。源码中那些"Pro feature"——比如 Audio.cpp（1675 行音频采集实现）、CANBus.cpp（840 行）、MQTT/Client.cpp（943 行）、Plot3D.cpp（1559 行）——**全部完整存在于 GPL 仓库**，只是用 `#ifdef BUILD_COMMERCIAL` 编译开关挡住了，GPL build 时被预处理器跳过。

这套设计来自上游作者的商业策略：让你选择「编译商业版（写真实 license key 验证过的）」或「编译 GPL 版（功能阉割）」。但代码本身没有"加密"或"保护"——它是公开的 GPL 文本。

我们做的是：
1. 把 `#ifdef` 区分翻过来——driver/widget 这种没有 license 强依赖的代码，GPL 也编译它们
2. 几个 license 检查代码段（`if (!token.isValid() || !SS_LICENSE_GUARD()) return;`）改成 GPL 路径 unrestricted 通过
3. 把 build 系统的 link 链路补全（Qt6::Mqtt、Qt6::SerialBus、hidapi、libusb）
4. 给 Console Export 写了一份 GPL 实现（这块上游 GPL 真没有，但 30 行代码就能搞定）

GPL 协议本来就允许这种修改——衍生作品继续 GPL 即可。我们没有：
- ❌ 逆向编译 Pro 二进制
- ❌ 解密 license key 校验
- ❌ 用闭源 Pro 仓库的代码

---

## 文件清单

| 文件 | 内容 |
|---|---|
| `README.md` | 英文版（GitHub 默认显示） |
| `README.zh-CN.md` | 本文档 |
| `PATCHES.md` | 详细的修改清单（哪一行改了什么），英文 |
| `0001-gpl-full-unlock.patch` | 单个 `git am` 兼容 patch 文件，68KB |

---

## License

修改部分继承 **GPL-3.0-only**（被改动的所有文件本身都是 GPL/Commercial 双协议，我们选 GPL 那一支）。

不接受 Pull Request 关于添加 MDF4 export / 商业 license UI 等 Pro-only 功能——那是上游作者的商业产品。

---

## 致谢

- [Alex Spataru](https://github.com/alex-spataru) — Serial Studio 原作者，这个软件确实做得很棒
- [Qt](https://www.qt.io/) — UI 框架 + Mqtt/SerialBus/Quick3D 等模块
- [hidapi](https://github.com/libusb/hidapi) / [libusb](https://github.com/libusb/libusb) — USB/HID 库
- [miniaudio](https://miniaud.io/) — 音频跨平台单文件库（vendored in `src/ThirdParty/`）

如果你长期用得开心，**请考虑给上游买个 Pro license 支持作者**。GPL 版本给你折腾、改造、做爱好项目用，商业场景下还是付费版责任清楚。
