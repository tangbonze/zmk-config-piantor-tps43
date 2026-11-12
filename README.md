# Piantor Wireless TPS43 — ZMK 固件配置

支持 **SuperMini nRF52840 (`nice_nano_v2`)**、**ZMK Studio 在线改键**、**Azoteq TPS43 触摸板**（[beekeeb/zmk_driver_azoteq](https://github.com/beekeeb/zmk_driver_azoteq)）、**KeyBeacon 状态监控**（[ykiewang/keybeacon](https://github.com/ykiewang/keybeacon) / [zmk-keybeacon](https://github.com/ykiewang/zmk-keybeacon)）以及 **42 键 ⇄ 36 键物理布局无损切换**。

---

## 编译产物版本说明（`build.yaml`）

推送到 GitHub 后，GitHub Actions 会自动编译出 **3 套完整分体固件** + `settings_reset`：

| 版本 | 主手 (Central，插线/蓝牙直连) | 从机 (Peripheral) | 触摸板位置 |
| :--- | :--- | :--- | :--- |
| **1. 默认版本（右手主手+右手 TPS43）** | `piantor_tps43_right_central` | `piantor_tps43_left_peripheral` | **右手**（直连主手，最低延迟） |
| **2. 左手主手版（左手主手+左手 TPS43）** | `piantor_tps43_left_central` | `piantor_tps43_right_peripheral` | **左手**（直连主手，最低延迟） |
| **3. 纯无线无触摸板版（左手主手）** | `piantor_notp_left_central` | `piantor_notp_right_peripheral` | 无（不启用 I2C 与 TPS43） |
| **重置配对固件** | `settings_reset` | `settings_reset` | 切换版本/主从前先刷一次 |

> **重要**：三个版本的**主手 (Central)** 固件均已开启 **ZMK Studio** (`CONFIG_ZMK_STUDIO=y` + `studio-rpc-usb-uart`) 和 **KeyBeacon KBP 1.2** (`CONFIG_ZMK_KEYBEACON=y`)，且共用同一份 [config/piantor_wireless.keymap](config/piantor_wireless.keymap)。

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
   - 将主手通过 USB（或已解锁后的蓝牙）连接 [ZMK Studio](https://zmk.studio/)。
   - 同时按住 **`LOWER + RAISE`** 进入 `ADJUST` 层，按下左侧第 5 列中行键（`G` 位置）触发 **`&studio_unlock`** 解锁。
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
