<a id="zh"></a>

# Piantor Wireless TPS43 — ZMK 固件配置

**中文** · [English](#en)

支持 **SuperMini nRF52840 (`nice_nano//zmk`)**、**ZMK Studio 在线改键（已免解锁）**、**Azoteq TPS43 触摸板**（[beekeeb/zmk_driver_azoteq](https://github.com/beekeeb/zmk_driver_azoteq)）、**KeyBeacon 状态监控**（[ykiewang/keybeacon](https://github.com/ykiewang/keybeacon) / [zmk-keybeacon](https://github.com/ykiewang/zmk-keybeacon)）以及 **42 键 ⇄ 36 键物理布局无损切换**。

---

## 编译产物版本说明（`build.yaml`）

推送到 GitHub 后，GitHub Actions 会自动编译出 **3 套完整分体固件** + `settings_reset`：

| 版本 | 主手 (Central，插线/蓝牙直连) | 从机 (Peripheral) | 触摸板位置 |
| :--- | :--- | :--- | :--- |
| **1. 默认版本（右手主手+右手 TPS43）** | `piantor_tps43_right_central` | `piantor_tps43_left_peripheral` | **右手**（直连主手，最低延迟） |
| **2. 左手主手版（左手主手+左手 TPS43）** | `piantor_tps43_left_central` | `piantor_tps43_right_peripheral` | **左手**（直连主手，最低延迟） |
| **3. 纯无线无触摸板版（左手主手）** | `piantor_notp_left_central` | `piantor_notp_right_peripheral` | 无（不启用 I2C 与 TPS43） |
| **重置配对固件** | `settings_reset` | `settings_reset` | 切换版本/主从前先刷一次 |

> **重要**：三个版本的**主手 (Central)** 固件均已开启 **ZMK Studio（默认免解锁）** (`CONFIG_ZMK_STUDIO=y`、`CONFIG_ZMK_STUDIO_LOCKING=n` + `studio-rpc-usb-uart`) 和 **KeyBeacon KBP 1.2** (`CONFIG_ZMK_KEYBEACON=y`)，且共用同一份 [config/piantor_wireless.keymap](config/piantor_wireless.keymap)。

---

## KeyBeacon 桌面状态显示（KBP 1.2）

- 已通过 [config/west.yml](config/west.yml) 引入 `ykiewang/zmk-keybeacon`（固定版本 `v1.2.0`），并在主手通用配置 [piantor_role_central.conf](boards/shields/piantor_role_central/piantor_role_central.conf) 中启用：
  - `CONFIG_ZMK_KEYBEACON=y`：上报当前激活层名（来自 keymap 的 `display-name`，即 `QWERTY` / `NUM_NAV` / `SYM_FN` / `ADJUST`）与修饰键状态
  - `CONFIG_ZMK_KEYBEACON_CONNECTIVITY=y`：上报主机连接状态、当前 BLE Profile 槽位、分体在线状态及当前输出端点（USB / BLE）
  - `CONFIG_ZMK_KEYBEACON_BATTERY=y`：配合 [piantor_wireless.conf](config/piantor_wireless.conf) 中的 `CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y` 上报双半电量
  - `CONFIG_ZMK_KEYBEACON_HOST_LINK=y`：基于每连接只读特征区分“当前电脑正在输入”与“当前在其他已配对电脑上使用”
- 从机 (Peripheral) 与 `settings_reset` 目标不引用 `piantor_role_central`，且模块 CMake 保护条件要求 `CONFIG_ZMK_SPLIT_ROLE_CENTRAL`，因此不会编入从机固件。
- 桌面端下载（macOS / Windows）：前往 [ykiewang/keybeacon Releases](https://github.com/ykiewang/keybeacon/releases) 下载对应桌面端应用，蓝牙连接键盘后即可自动发现并显示实时状态。
- 说明：若刷写的是**版本 1（右手主手）**，KeyBeacon 协议的双半电量槽位第 1 字节对应主手（右手）、第 2 字节对应从手（左手）。

---

## 42 键 ⇄ 36 键 Layout 切换方法

1. **在 ZMK Studio 中实时切换**：
   - 将主手通过 USB 或蓝牙直接连接 [ZMK Studio](https://zmk.studio/)。
   - 在 ZMK Studio 右上角布局选择菜单即可在 **`Piantor 42-Key (6-Col)`** 与 **`Piantor 36-Key (5-Col)`** 之间切换。
   - 已配置 `zmk,physical-layout-position-map`（[piantor_wireless-layouts.dtsi](boards/shields/piantor_wireless/piantor_wireless-layouts.dtsi)），切换至 36 键时会自动忽略外侧第 6 列，其余 36 颗按键映射保持 100% 精准对应。
2. **编译时直接默认 36 键**：
   - 打开 [config/piantor_wireless.keymap](config/piantor_wireless.keymap)，取消注释顶部的 `chosen { zmk,physical-layout = &physical_layout_36; };` 后重新编译即可。

---

## 硬件引脚对照（与 PCB 完全一致）

- **矩阵**：`col2row`，左/右统一接脚：
  - `row0–row3`：`P0.31 / P0.29 / P0.02 / P1.15`（`&pro_micro 21, 20, 19, 18`）
  - `col0–col5`：`P1.00 / P0.11 / P0.09 / P0.10 / P1.11 / P1.13`（`&pro_micro 6, 7, 10, 16, 14, 15`，其中 `col0` 为可折断第 6 列）
- **TPS43 触摸板 (`J_TPS`)**：
  - `SDA`：`P0.17`（`&pro_micro 2`） / `SCL`：`P0.20`（`&pro_micro 3`）
  - `RDY`：`P1.04`（`&pro_micro 8`，高电平有效中断） / `NRST`：`P1.06`（`&pro_micro 9`，驱动拉低 10ms 复位、拉高 3.3V 工作）

---

## 刷写步骤

1. 若之前刷过不同主从角色的固件，先对左右手各双击 RST 刷入一次 `settings_reset.uf2`。
2. 根据你的实际装机选择对应的一对 `.uf2`（例如默认右手接 TPS43：右手刷 `piantor_tps43_right_central.uf2`，左手刷 `piantor_tps43_left_peripheral.uf2`）。
3. 若后续需要调整触摸板方向（如旋转 90°/180° 或上下反转），编辑 [piantor_tps43_central.overlay](boards/shields/piantor_tps43_central/piantor_tps43_central.overlay) 取消注释 `switch-xy` / `invert-x` / `invert-y` / `invert-scroll-y` 即可。

---

<a id="en"></a>

# Piantor Wireless TPS43 — ZMK Firmware Config

[中文](#zh) · **English**

ZMK firmware configuration for the wireless split **Piantor** with **SuperMini nRF52840 (`nice_nano//zmk`)**, **ZMK Studio** live keymap editing (unlocked by default), **Azoteq TPS43** trackpad ([beekeeb/zmk_driver_azoteq](https://github.com/beekeeb/zmk_driver_azoteq)), **KeyBeacon** status display ([ykiewang/keybeacon](https://github.com/ykiewang/keybeacon) / [zmk-keybeacon](https://github.com/ykiewang/zmk-keybeacon)), and **lossless 42-key ⇄ 36-key physical layout switching**.

---

## Build Targets (`build.yaml`)

GitHub Actions automatically builds **3 complete split firmware pairs** plus `settings_reset`:

| Variant | Central (USB / Host BLE) | Peripheral | Trackpad Side |
| :--- | :--- | :--- | :--- |
| **1. Default (Right Central + Right TPS43)** | `piantor_tps43_right_central` | `piantor_tps43_left_peripheral` | **Right** (directly on Central, lowest latency) |
| **2. Left Central + Left TPS43** | `piantor_tps43_left_central` | `piantor_tps43_right_peripheral` | **Left** (directly on Central, lowest latency) |
| **3. Wireless No-Trackpad (Left Central)** | `piantor_notp_left_central` | `piantor_notp_right_peripheral` | None (I2C and TPS43 disabled) |
| **Settings Reset** | `settings_reset` | `settings_reset` | Flash before switching roles/variants |

> **Note**: All three **Central** targets enable **ZMK Studio (pre-unlocked)** (`CONFIG_ZMK_STUDIO=y`, `CONFIG_ZMK_STUDIO_LOCKING=n` + `studio-rpc-usb-uart`) and **KeyBeacon KBP 1.2** (`CONFIG_ZMK_KEYBEACON=y`), and share [config/piantor_wireless.keymap](config/piantor_wireless.keymap).

---

## KeyBeacon Desktop Status Monitor (KBP 1.2)

- Pulls in `ykiewang/zmk-keybeacon` (pinned to `v1.2.0`) via [config/west.yml](config/west.yml) and enables the full KBP 1.2 stack on Central targets in [piantor_role_central.conf](boards/shields/piantor_role_central/piantor_role_central.conf):
  - `CONFIG_ZMK_KEYBEACON=y`: reports the active layer name (from keymap `display-name`: `QWERTY` / `NUM_NAV` / `SYM_FN` / `ADJUST`) and held modifiers
  - `CONFIG_ZMK_KEYBEACON_CONNECTIVITY=y`: reports host connection state, active BLE profile slot, split link status, and active output endpoint (`USB` / `BLE`)
  - `CONFIG_ZMK_KEYBEACON_BATTERY=y`: reports dual-half battery levels together with `CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y` in [piantor_wireless.conf](config/piantor_wireless.conf)
  - `CONFIG_ZMK_KEYBEACON_HOST_LINK=y`: exposes per-connection read-only host slot context ("in use on this computer" vs "active on another paired host")
- Peripheral and `settings_reset` targets do not include `piantor_role_central` and are also excluded by `zmk-keybeacon`'s `CONFIG_ZMK_SPLIT_ROLE_CENTRAL` CMake guard.
- Desktop app (macOS / Windows): download from [ykiewang/keybeacon Releases](https://github.com/ykiewang/keybeacon/releases) and connect the keyboard over Bluetooth.
- Note: for **Variant 1 (Right Central)**, KeyBeacon battery byte 0 represents Central (Right half) and byte 1 represents Peripheral (Left half).

---

## 42-Key ⇄ 36-Key Layout Switching

1. **Switch live in ZMK Studio**:
   - Connect the Central half directly over USB or BLE to [ZMK Studio](https://zmk.studio/).
   - Use the layout dropdown in the top-right corner of ZMK Studio to switch between **`Piantor 42-Key (6-Col)`** and **`Piantor 36-Key (5-Col)`**.
   - Mapped via `zmk,physical-layout-position-map` in [piantor_wireless-layouts.dtsi](boards/shields/piantor_wireless/piantor_wireless-layouts.dtsi), so switching to 36 keys drops the break-off 6th outer column while preserving the remaining 36 bindings 1:1.
2. **Default to 36 keys at compile time**:
   - Uncomment `chosen { zmk,physical-layout = &physical_layout_36; };` at the top of [config/piantor_wireless.keymap](config/piantor_wireless.keymap) and rebuild.

---

## Hardware Pinout (Matches PCB)

- **Matrix**: `col2row`, identical pin assignment on Left and Right halves:
  - `row0–row3`: `P0.31 / P0.29 / P0.02 / P1.15` (`&pro_micro 21, 20, 19, 18`)
  - `col0–col5`: `P1.00 / P0.11 / P0.09 / P0.10 / P1.11 / P1.13` (`&pro_micro 6, 7, 10, 16, 14, 15`, where `col0` is the snap-off 6th column)
- **Azoteq TPS43 (`J_TPS`)**:
  - `SDA`: `P0.17` (`&pro_micro 2`) / `SCL`: `P0.20` (`&pro_micro 3`)
  - `RDY`: `P1.04` (`&pro_micro 8`, active-high interrupt) / `NRST`: `P1.06` (`&pro_micro 9`, driven low 10 ms to reset, high 3.3 V to run)

---

## Flashing

1. If switching from firmware with different Central/Peripheral roles, double-press RST on both halves and flash `settings_reset.uf2` first.
2. Flash the matching `.uf2` pair for your build (e.g., for Default Right TPS43: flash `piantor_tps43_right_central.uf2` to the Right half and `piantor_tps43_left_peripheral.uf2` to the Left half).
3. To adjust trackpad orientation (90°/180° rotation or inverted axes/scroll), uncomment `switch-xy` / `invert-x` / `invert-y` / `invert-scroll-y` in [piantor_tps43_central.overlay](boards/shields/piantor_tps43_central/piantor_tps43_central.overlay).
