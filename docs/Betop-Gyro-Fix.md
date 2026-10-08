# 北通鲲鹏20 体感（gyro/IMU）修复记录

## 问题现象

北通鲲鹏20（BTP-KP20）在 NS 模式下伪装成官方 Switch Pro 控制器（VID/PID `057E:2009`）。
SDL3 能收到 200Hz 的 0x30 IMU HID 报告，但 `GetSensorData` / `GAMEPAD_SENSOR_UPDATE`
的值全部是 `inf/nan`，导致体感不可用。

## 根因

手柄对 Switch Pro 的 SPI 校准读取命令（subcommand `0x10`）**响应成功**，但返回的
校准数据区域全是 `0xFF`（未编程）。SDL 和 Eden 内置 joycon 驱动都把 `0xFF` 字节
解析为 `-1`（`0xFFFF`），导致缩放公式中 `coeff - offset = -1 - (-1) = 0`，分母为 0
→ 每次采样都变成 `inf/nan`。

- 相关上游 issue：https://github.com/libsdl-org/SDL/issues/16237
- 校验数据全 `0xFF` 的区域：IMU 校准 `0x6020`、摇杆校准 `0x8010` / `0x603D`

## 修复方案（方案 A：修 Eden 内置 joycon/procon 驱动，不改 SDL3）

让北通手柄走 Eden 内置的 joycon 驱动（通过 hidapi 直连、自行解析 IMU），
完全绕过 SDL3 的 sensor 管线。

### 改动文件

1. **`src/input_common/helpers/joycon_protocol/calibration.cpp`**
   - `ValidateValue(u16)`：增加 `value == 0xFFFF` 无效值检查，回退默认值
   - `ValidateValue(s16)`：增加 `value == -1`（即 `0xFFFF`，全 0xFF 读出）无效值检查，
     回退默认缩放。修复后 `GetAccelerometerValue` / `GetGyroValue` 不再出现分母为 0。

2. **`src/input_common/helpers/joycon_driver.cpp`**（`InitializeDevice`）
   - Pro 手柄读取 SPI `DEVICE_TYPE` 得到无效值（如 `0xFF`）时，回退到由 VID/PID
     判断出的 `handle_device_type`，避免 poller 因类型不匹配而完全无输入。

3. **`src/common/settings.h`**
   - `enable_procon_driver` 默认值 `false` → `true`，使北通（伪装成 `057E:2009`）
     默认被内置 procon 驱动接管。
   - 同步说明：开启后 SDL 会禁用 hidapi switch 驱动（`SDL_HINT_JOYSTICK_HIDAPI_SWITCH=0`），
     SDL 驱动也会在 `InitJoystick` 中放弃该设备，交给 joycon 驱动。

4. **用户配置 `%APPDATA%\eden\config\qt-config.ini`**
   - `enable_procon_driver=false` → `true`（否则旧的配置值会覆盖代码默认值）

## 体感数据流（修复后）

```
北通(NS模式, 057E:2009)
  → Joycons::ScanThread 枚举 VID 0x057e
  → 静态 GetDeviceType 匹配 PID 0x2009 → ControllerType::Pro
  → RegisterNewDevice → RequestDeviceAccess → InitializeDevice
  → GetImuCalibration 读 SPI 0x6020 全 0xFF
  → ValidateCalibration 经 ValidateValue 回退默认缩放（0x4000 / 0x3be7）
  → JoyconPoller::UpdateActiveProPadInput → GetMotionInput → 有效 accel/gyro
  → OnMotionUpdate → 上层模拟
```

## 编译命令

增量编译 GUI 主程序（`yuzu` 目标，输出 `build/bin/eden.exe`）：

```bash
# 1. 写一个构建脚本（PowerShell 转义 bash 变量易出错，统一走脚本）
#    build_gyro_fix.sh：
#!/usr/bin/env bash
set -e
export PATH=/mingw64/bin:/usr/bin:$PATH
cd /e/AI/mirror
cmake --build build --target yuzu -j8 > /e/AI/mirror/build_gyro_fix.log 2>&1

# 2. 执行
C:\msys64\usr\bin\bash.exe -lc "cd /e/AI/mirror && export PATH=/mingw64/bin:/usr/bin:\$PATH && bash build_gyro_fix.sh"

# 3. 查看编译结果
Get-Content E:\AI\mirror\build_gyro_fix.log -Tail 40
```

关键点：

- **必须用 `bash -lc` 走 MSYS2 执行**，不能用 PowerShell 直接跑 `cmake --build`，
  否则会找不到 `nproc`/`tail` 等 Unix 工具，且 `$(...)`、`$PATH` 会被 PowerShell 转义吞掉。
- 本次只改了 `input_common` 库与 `settings.h`，增量编译仅重编 `input_common` 并重链
  `yuzu`，约几分钟；全量编译约 40+ 分钟。
- 日志尾部看到 `[100%] Built target yuzu` 即成功。
- 确认产物更新时间：`Get-Item E:\AI\mirror\build\bin\eden.exe | Select LastWriteTime`

### 其他编译目标

| 目标名 | 输出 | 用途 |
|---|---|---|
| `yuzu` | `build/bin/eden.exe` | GUI 主程序（本次修复目标） |
| `yuzu-cmd` | `build/bin/eden-cli.exe` | 命令行前端 |
| `yuzu-room` | `build/bin/eden-room.exe` | 房间联机服务器 |
| `tests` | `build/bin/tests.exe` | 单元测试 |

单目标增量编译示例：`cmake --build build --target yuzu-cmd -j$(nproc)`

## 验证方式

编译 `yuzu` 目标（输出 `build/bin/eden.exe`）后：
1. 确保北通进入 NS 模式并连接
2. 启动 eden.exe，在「模拟设置 → 输入 → 高级」确认「Pro 控制器驱动」已勾选
   （代码默认已改为 true）
3. 在体感映射界面晃动手柄，观察加速度计/陀螺仪是否输出正常数值（静止时
   加速度 ~9.8 m/s²，陀螺仪 ~0 rad/s）

## 备注

- 本方案未修改 SDL3 源码，也未重新编译 SDL3.dll，运行时替换的是 Eden 二进制。
- 若日后 Eden 默认启用 `enable_procon_driver` 对官方 Pro 控制器有影响，可在
  GUI 中单独关闭该选项。
