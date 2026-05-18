# GPL Full-Features Build Patches

This fork unlocks features that the upstream Serial-Studio source disables in
GPL builds (Audio / CAN Bus / Modbus / USB / HID / Process / MQTT drivers and
console export to file). All changes are limited to:

- Removing or relocating `#ifdef BUILD_COMMERCIAL` guards around code that is
  already in the GPL source tree.
- Switching QML `Loader.active: Cpp_CommercialBuild` to `active: true` so the
  driver config panels actually instantiate.
- Implementing the GPL branch of `Console::Export` (the upstream branch was a
  Pro-only stub that just popped a "buy a license" dialog).
- Wrapping the few license-tier checks inside `MQTT::Client` with
  `#ifdef BUILD_COMMERCIAL` so they no longer gate the GPL build.

Nothing was reverse-engineered. The actual driver implementations (Audio.cpp
1675 lines, CANBus.cpp 840 lines, Modbus.cpp 1425 lines, MQTT/Client.cpp 943
lines, etc.) ship in the upstream GPL source — they were just excluded from
the build target on the GPL side. MQTT and CAN Bus require Qt modules that
are LGPL/Commercial dual-licensed and not packaged in the standard Qt online
installer; the build process compiles `qtmqtt` from source.

## Build prerequisites (Windows / MSVC)

- Visual Studio 2022 Build Tools — C++ workload (MSVC 14.44+)
- Qt 6.9.2 — `msvc2022_64` arch, modules: `qtbase qtdeclarative qtsvg qttools
  qtcharts qtserialport qtconnectivity qtwebchannel qtwebsockets qtpositioning
  qtimageformats qtmultimedia qtshadertools qt5compat qtgraphs qtquick3d
  qtwebview qtserialbus qtwebengine qtpdf`
- Qt 6.9.2 **Mqtt** module — compile from `https://github.com/qt/qtmqtt.git`
  branch `v6.9.2` and install into the same Qt prefix
- CMake 4.x (3.30+ required for `cmake_policy(CMP0167)` in vendored mdflib)
- WiX Toolset v3.14 — only needed when packaging the `.msi`
- hidapi v0.14.0 prebuilt Windows binary
  (https://github.com/libusb/hidapi/releases/tag/hidapi-0.14.0) extracted to
  `D:/BuildTools/extlibs/hidapi/{include, x64}`
- libusb v1.0.27 prebuilt Windows binary
  (https://github.com/libusb/libusb/releases/tag/v1.0.27) extracted to
  `D:/BuildTools/extlibs/libusb/{include, VS2022/MS64/dll}`

`EXTLIBS_ROOT` (default `D:/BuildTools/extlibs`) can be overridden at configure
time if hidapi/libusb live elsewhere.

## Configure & build

```
cmake -S . -B build -G Ninja ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_PREFIX_PATH=D:/Qt/6.9.2/msvc2022_64 ^
  -DBUILD_GPL3=ON ^
  -DPRODUCTION_OPTIMIZATION=OFF ^
  -DCPACK_WIX_ROOT=D:/BuildTools/wix3
cmake --build build --config Release
cmake --build build --config Release --target package   :: produces .msi
```

Output: `build/Serial-Studio-GPL3-v3.2.7.msi`, MSI installs as
"Serial Studio GPLv3" (does not collide with upstream "Serial Studio Pro").

## Patch inventory

### Build system

| File | Change |
|------|--------|
| `app/CMakeLists.txt` | Add `SerialBus` + `Mqtt` to GPL `QT_MODULES` / `QT_LIBS`. Driver sources (`Audio.cpp`, `CANBus.cpp`, `Modbus.cpp`, `HID.cpp`, `USB.cpp`, `Process.cpp`, `MQTT/Client.cpp`, `ThirdParty/miniaudio.cpp`) and their headers + driver setup QML files moved into the unconditional `SOURCES`/`HEADERS`/`QML_SOURCES` lists. On GPL+WIN32 link hidapi + libusb prebuilt `.lib`, define `WIN32_LEAN_AND_MEAN`/`NOMINMAX`/`_WINSOCKAPI_` to keep winsock2 winning over winsock.h, copy the two DLLs next to the exe via `POST_BUILD`, and `install(FILES ...)` them into MSI `bin/`. |

### C++ source

| File | Change |
|------|--------|
| `app/src/SerialStudio.h` | `BusType` enum: drop the `#ifdef BUILD_COMMERCIAL` around `Audio`/`ModBus`/`CanBus`/`RawUsb`/`HidDevice`/`Process`. |
| `app/src/IO/Drivers/USB.h`, `HID.h`, `Process.h` | Remove the outer `#ifdef BUILD_COMMERCIAL ... #endif` that wrapped the entire header. USB.h additionally adds `#include <winsock2.h>` before `<libusb.h>` (Windows). |
| `app/src/IO/Drivers/USB.cpp`, `HID.cpp`, `Process.cpp` | Same outer `#ifdef` removed. USB.cpp pulls `winsock2.h` first on `_WIN32`. |
| `app/src/IO/ConnectionManager.h`, `ConnectionManager.cpp` | Drop `#ifdef` around driver `#include`s, accessor declarations, `unique_ptr` members, switch cases in `activeUiDriver` / `uiDriverForBusType` / `createDriver`, the destructor's disconnect loop, the `availableBuses()` list, `setupExternalConnections()` driver wires, and the connection-manager-side `connect` calls. MQTT include and `MQTT::Client::instance().hotpathTxFrame(...)` calls are now unconditional; Licensing includes/calls stay behind `#ifdef BUILD_COMMERCIAL`. |
| `app/src/Misc/ModuleManager.cpp` | Driver singletons and `MQTT::Client::instance()` moved out of the `#ifdef BUILD_COMMERCIAL` block. `Cpp_IO_Audio` / `Cpp_IO_CANBus` / `Cpp_IO_Modbus` / `Cpp_IO_USB` / `Cpp_IO_HID` / `Cpp_IO_Process` / `Cpp_MQTT_Client` context properties registered for GPL. DBCImporter / Licensing / Trial / LemonSqueezy stay Pro-only. |
| `app/src/MQTT/Client.cpp` | Licensing includes and four license-tier checks (`!token.isValid() || !SS_LICENSE_GUARD()` ...) wrapped in `#ifdef BUILD_COMMERCIAL`. `hotpathTxFrame()` falls back to an unrestricted publish path on GPL builds. |
| `app/src/Console/Export.h`, `Export.cpp` | Add a GPL branch that implements console-to-file export synchronously: per-device `QFile` + `QTextStream` map, lazy file creation on first data using the upstream timestamped naming (`yyyy_MMM_dd HH_mm_ss[_deviceN].txt` under `WorkspaceManager.path("Console")/<projectTitle>/`), `closeFile()` flushes and closes everything, `setExportEnabled(true)` no longer pops the "Pro feature" dialog. The Pro branch is unchanged. |

### QML

| File | Change |
|------|--------|
| `app/qml/MainWindow/Panes/SetupPanes/Hardware.qml` | Six driver `Loader { active: Cpp_CommercialBuild ... }` blocks (Audio / Modbus / CANBus / USB / HID / Process) changed to `active: true`. |
| `app/qml/MainWindow/Panes/SetupPanes/Drivers/CANBus.qml` | Drop the standalone `DBCPreviewDialog { id: dbcPreviewDialog }` instance — it referenced a Pro-only QML type that is not registered in the GPL build, causing the whole file to fail to load and the CAN Bus panel to render blank. The dialog is never referenced elsewhere in the file. |
| `app/qml/MainWindow/Panes/Toolbar.qml` | MQTT button `Loader { active: Cpp_CommercialBuild }` → `active: true` so the toolbar entry appears. |
| `app/qml/Dialogs/MQTTConfiguration.qml` | Top "MQTT is a Pro Feature" `Widgets.ProNotice` set to `visible: false` / `activationFlag: false`. The rest of the dialog is unchanged. |
| `app/qml/main.qml` | `proVersion` returns `true` on GPL builds (was `false`) so the MQTT dialog's `enabled: app.proVersion` controls go live. `showMqttConfiguration()` no longer guards on `Cpp_CommercialBuild`. |

### External dependencies (not stored in the repo)

| Tool | How obtained |
|------|--------------|
| Qt MQTT 6.9.2 | `git clone --depth 1 --branch v6.9.2 https://github.com/qt/qtmqtt.git`; `cmake -S . -B build -G Ninja -DCMAKE_PREFIX_PATH=D:/Qt/6.9.2/msvc2022_64 -DCMAKE_INSTALL_PREFIX=D:/Qt/6.9.2/msvc2022_64`; build + install. |
| hidapi 0.14.0 | Release zip `hidapi-win.zip` extracted to `D:/BuildTools/extlibs/hidapi/`. |
| libusb 1.0.27 | Release 7z `libusb-1.0.27.7z` extracted to `D:/BuildTools/extlibs/libusb/`. |

## What stays Pro-only

These features are not unlocked because their actual implementation is **not**
in the public GPL source — only Pro stubs / dialogs are. Adding them would
mean writing the implementation from scratch.

- MDF4 export (`Console/Export.cpp` Pro branch writes the file format; GPL has
  a stub. Reader / writer code lives in `lib/mdflib`, which is open source, so
  a GPL implementation is possible but non-trivial.)
- License management UI, Trial timer, Lemon Squeezy activation
- Plot 3D widget, ImageView dashboard widget, in-line image export
- DBC importer (CAN database file parsing)
- AI-Assistant integration

## What does not work yet

- The CAN Bus driver was never tested against real hardware in this fork; on a
  machine without a CAN adapter the panel shows "No CAN Drivers Found" /
  empty interface list — that is the expected behavior, not a regression.
- The JSON API Handler classes for these drivers (e.g.
  `API/Handlers/AudioHandler.cpp`) are still wrapped in
  `#ifdef BUILD_COMMERCIAL`. The drivers work from the GUI, but the JSON-RPC
  API server does not expose `audio.*` / `modbus.*` / `canbus.*` / `usb.*` /
  `hid.*` / `process.*` / `mqtt.*` commands on GPL builds. Same with the gRPC
  surface.

## License

Modifications are GPL-3.0-only (per the inherited upstream license on the
files touched). See `LICENSES/GPL-3.0-only.txt`.
