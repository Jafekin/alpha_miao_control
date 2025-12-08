# AlphaMeow Control Firmware

> 2024 嵌入式创新大赛·海思赛道「智慧象棋」机械臂控制端。运行在 BearPi-HM Nano (Hi3861V100) 上，负责棋子吸取、落子以及与上位机的 UDP 指令交互。

## 项目简介 / Overview
- 通过 PCA9685 扩展板驱动 5 自由度机械臂、蠕动泵和电磁阀，实现棋子抓取与放置。
- 预置 10×10 棋盘网格的 `action[101][5]` 角度表，可在运行中通过 UDP 或实体按键进行标定。
- 集成 Wi-Fi STA 连接、UDP 指令通道与 SSD1306 OLED 状态面板，便于调试与在线监控。

## 功能亮点 / Feature Highlights
- **一体化动作管线**：`MechanicalArmDown → PumpSuckUp → MechanicalArmUp` 等模块化流程，确保动作安全顺序。
- **在线调参**：ADC 按键 + OLED 显示即可逐个伺服微调；UDP 协议支持批量写入并立即回放。
- **网络联动**：`WifiConnect` 自动扫描并接入指定热点，UDP 端口 `888` 提供上位机控制、状态回执与异常日志。
- **可拓展 GN 组件**：`src/applications/sample/wifi-iot/app/chessrobot` 以 GN `static_library` 形式集成，便于在 OpenHarmony LiteOS-M 工程内复用。

## 硬件与依赖 / Hardware Stack
| 模块 | 说明 |
| --- | --- |
| BearPi-HM Nano (Hi3861V100) | OpenHarmony LiteOS-M 开发板，提供 Wi-Fi、GPIO、I²C、ADC |
| PCA9685 PWM 扩展板 | 驱动 6× 舵机 + 蠕动泵/电磁阀，I²C0, SDA GPIO13, SCL GPIO14 |
| 5-DOF 机械臂 + 吸盘 | 对应 `ACTUATOR_CHANNEL_1~5`，动作表以角度 (0°–180°) 定义 |
| 蠕动泵 + 电磁阀 | `PUMP_CHANNEL/PUMP_SuckUp`、`SOLENOID_VALVE_CHANNEL` 控制吸附 |
| SSD1306 OLED | I²C 接口，实时显示通道号与当前 PWM |
| 3 段式 ADC 按键 | 0.0–2.0 V 不同电平切换通道/增减角度 |

## 目录结构 / Repo Layout
```text
src/applications/sample/wifi-iot/app/
├─ BUILD.gn                # 选择需要编译的应用特性
└─ chessrobot/
   ├─ BUILD.gn            # 定义 control 静态库
   ├─ main.c              # 任务入口、动作序列、UDP 服务
   ├─ wifi_connect.c      # STA 扫描、连接与 DHCP
   ├─ pca9685.{c,h}       # PWM 扩展驱动
   ├─ oled_ssd1306.{c,h}  # OLED 驱动
   ├─ ssd1306_fonts.{c,h} # 字模数据
   └─ debug.c             # 选配的调试辅助（默认未编译）
```

## 构建与烧录 / Build & Flash
1. **准备环境**：安装 OpenHarmony Hi3861 LiteOS-M 工具链与 `hb` 构建脚本，确保 `OHOS_ARM_TOOLCHAIN` 已配置。
2. **选择产品**：在工程根目录执行 `hb set`，Target 选择 `wifiiot` / `bearpi_hm_nano` 对应的产品配置。
3. **启用特性**：确认 `src/applications/sample/wifi-iot/app/BUILD.gn` 中 `features` 包含 `"chessrobot:control"`。
4. **构建**：运行 `hb build -T //src/applications/sample/wifi-iot/app`，生成 `out/.../wifi_iot_app.bin`。
5. **烧录**：使用 HiBurn 或串口烧录工具，将固件写入 Hi3861，复位后即可启动。

## 运行配置 / Runtime Setup
### 1. Wi-Fi & 网络
- 在 `main.c` 中修改以下宏以匹配实际热点：
  ```c
  #define CONFIG_WIFI_SSID "<YourSSID>"
  #define CONFIG_WIFI_PWD  "<YourPassword>"
  #define NATIVE_IP_ADDRESS "<Board IP>"
  #define DEVICE_IP_ADDRESS "<Host IP>"
  #define HOST_PORT 888
  #define DEVICE_PORT 45841
  ```
- `WifiConnect` 会扫描并连接 `ssid`，随后在 `HOST_PORT` 上创建 UDP Socket。上位机需与开发板处于同一网段。

### 2. 动作表 `action[101][5]`
- 采用行优先 `(x-1)*10 + y` 索引，覆盖 10×10 棋盘（含原点/缓存位）。
- 每行 5 个角度值，依次对应 `ACTUATOR_CHANNEL_1~5`。
- 可通过 UDP `@` 指令或在源码中直接编辑后重新编译。

### 3. OLED + 调参按键
- `AdjustTask` 线程会在 OLED 上显示当前伺服编号与角度。
- ADC 电压区间映射：`>2.0V` 空闲、`0.85–2.0V` 递增、`0.5–0.85V` 递减、`<0.5V` 轮换通道。
- 所有微调实时写回 `pwm[]` 并调用 `PCA9685_Angle`，便于现场标定。

### 4. UDP 指令协议
| 指令 | 载荷格式 | 行为 |
| --- | --- | --- |
| `@x1,x2,x3,x4,x5` | 5×整数角度 | 写入当前位置 (`adjust_x/adjust_y`) 的舵机角度并执行抓取→放置流程，用于单点标定 |
| `#x,y` | 两个坐标 | 更新 `adjust_x/adjust_y` 并执行一次抓取循环，确认该网格动作 |
| `mL,M` | 线路、模式 | 依次遍历同一行 (1–9)，批量执行抓取，用于展示或批处理 |
| `%x1,y1,x2,y2` | 起点与终点 | 完整搬运流程（抓取→移动→放置→复位）；当终点为 `(1,0)` 时会设置 `drop=true` 以触发弃子逻辑 |

## 常见问题 / Troubleshooting
- **无法连接热点**：确认 2.4 GHz 网络、密码正确，并在串口日志中检查 `WiFiInit failed` 具体错误码。
- **UDP 无响应**：确保上位机向 `HOST_PORT` 发送，并允许板端 IP 通过 PC 防火墙；可在串口查看 `Ack!` 是否回写。
- **舵机抖动/越界**：检查 `action` 表是否超出 `0–180`，必要时调用 `ResetPwm()` 后重新下发。
- **OLED 无显示**：验证 I²C 连接及供电，确认 `IoTI2cInit(HI_I2C_IDX_0, 400000)` 返回值为 0。

## 许可证 / License
本仓库遵循根目录 `LICENSE` 中的条款（Apache License 2.0）。如需在其他项目中复用代码，请保留原始版权声明。
# 2024嵌入式竞赛海思赛道-智慧象棋-大傻猫启动队-控制部分

该部分主打一个能用。

[1] `main`分支：成品代码

[2] `fixed`分支：调参用代码