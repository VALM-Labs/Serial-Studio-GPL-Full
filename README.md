# Serial-Studio GPL Full-Features Patches

This repository holds a single Git patch that unlocks the driver panels,
Console Export and MQTT support in a GPLv3 build of
[Serial-Studio](https://github.com/Serial-Studio/Serial-Studio) v3.2.7.

The patch produces a GPL binary versioned **3.2.7.1** to distinguish it from
upstream Pro/GPL releases.

## What's unlocked

- UART / Network / BluetoothLE (already in upstream GPL)
- **Audio** input driver
- **CAN Bus** driver (Qt SerialBus, needs CAN hardware to enumerate devices)
- **Modbus** driver (TCP + RTU)
- **USB** raw transfer driver (uses libusb)
- **HID** device driver (uses hidapi)
- **Process** driver (stdin/stdout & named pipes)
- **MQTT** publisher/subscriber (uses Qt6::Mqtt)
- **Console Export** to file — synchronously writes per-device `.txt` logs to
  `WorkspaceManager.path("Console")/<projectTitle>/yyyy_MMM_dd HH_mm_ss[_deviceN].txt`

See `PATCHES.md` (added by the patch) for the full inventory, build
prerequisites, and the list of features that stay Pro-only.

## How to apply

```bash
# Clone upstream Serial-Studio at the v3.2.7 tag
git clone --branch v3.2.7 --depth 1 https://github.com/Serial-Studio/Serial-Studio.git
cd Serial-Studio

# Fetch the patch and apply
curl -L -O https://raw.githubusercontent.com/VALM-Labs/Serial-Studio-GPL-Full/main/0001-gpl-full-unlock.patch
git apply --check 0001-gpl-full-unlock.patch && git am 0001-gpl-full-unlock.patch

# Read PATCHES.md for the full build prerequisites (Qt 6.9.2 + extra modules,
# hidapi, libusb, Qt MQTT from source) and then configure as usual:
cmake -S . -B build -G Ninja ^
  -DCMAKE_BUILD_TYPE=Release ^
  -DCMAKE_PREFIX_PATH=D:/Qt/6.9.2/msvc2022_64 ^
  -DBUILD_GPL3=ON ^
  -DPRODUCTION_OPTIMIZATION=OFF ^
  -DCPACK_WIX_ROOT=D:/BuildTools/wix3
cmake --build build --config Release --target package
# Output: build/Serial-Studio-GPL3-v3.2.7.1.msi
```

## License

Modifications are GPL-3.0-only (see `LICENSES/GPL-3.0-only.txt` in upstream).

## Files

- `0001-gpl-full-unlock.patch` — the patch (`git am`-compatible)
- `PATCHES.md` — full change inventory and build guide (also added by the patch)
