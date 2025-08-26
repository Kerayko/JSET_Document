### 空调面板控制逻辑（基于源码的逆向分析）

本文从 `ui task`、`sys ctrl task`、`temp adc task`、`fault check task`、LCD 相关任务与 `usr_main.c` 的初始化与线程编排出发，梳理空调面板的人机交互到执行端的整体控制逻辑与信号流。

#### 1. 总览：从输入到执行、从执行到显示
- 输入端：
  - 物理按键（`key task`）：MODE/AUTO/OFF/DEF/REC/AC_MAX/PTC/AC 以及长按 OFF。
  - 旋钮（`knob task`）：风量旋钮（增/减）、温度设定旋钮（增/减）。
- 中间层：
  - UI 线程：将输入转为 DBC 信号；根据 DBC 信号驱动面板显示与 LED；控制系统进入 RUN/IDLE/ERR。
  - 系统控制线程：依据信号与电压模式计算并下发设备控制（风机与多路电机）。
  - 传感采集线程：提供室内/蒸发/吹面/吹脚温度与 ACC 电压，并更新错误位与运行模式。
  - 故障检查线程：轮询检查电机与传感器等，设置错误码。
- 执行端：
  - 风机 PWM 控制（百分比）；
  - 模式电机（风向档位）；
  - 冷暖电机（混风门开度）；
  - 内外循环电机（进/外循环门）；
  - 吹脚电机（按模式联动）。
- 输出显示：
  - LCD 面板：图标（风扇、除霜、人物/吹风方向、雪花、AUTO、温度点）、三位数码、条形段（风量），通过面板线程对象映射后由 LCD 驱动线程刷新；
  - LED 指示：AUTO、AC、PTC、REC、DEF 等对应灯的开关。

#### 2. 主要状态与模式
- 系统高层模式（UI）：`IDLE`、`RUN`、`ERR(自检)`。
  - `IDLE`：显示清空/关闭、LED 关闭，系统控制收到 `CMD_OFF`，设备 Idle；
  - `RUN`：根据信号刷新显示与设备；
  - `ERR`：显示错误码序列，OFF 键退出。
- 系统控制模式（SYS CTRL）：`IDLE`、`RUN`、`ERR`，并受 ACC 电压模式影响：
  - `SYS_VOLT_LOW_STOP`/`LOW`/`NORMAL`/`HIGH`/`HIGH_STOP`。
  - 非 NORMAL 时，风机与电机 Idle，收发可能受限。

#### 3. 信号与映射关系（核心逻辑）
- 电压管理（`usr_main.c: acc_volt()` → `sys_ctrl.runtine.acc_volt_mode`）
  - ACC 电压通过 ADC 分压换算；分 5 档；低/高档位会禁止 CAN 应用收发与设备动作；具有 1V 左右滞回。

- 风量（Level → PWM 百分比）
  - UI 在 `RUN` 中读取 `sig_rd_v(&value)` 并显示：
    - 面板条形段与风扇图标；
  - SYS CTRL 在 `RUN` 且 NORMAL 下：
    - `task_drv_crtl_fan_set_percent_api(pfan, sys_ctrl_level_to_per_tb[value])`。

- 模式（Face / Face+Foot / Foot / Foot+Defrost / Defrost）
  - UI：根据 `Mode_Ctrl` 设置三图标（前吹、吹脚、吹人）与 DEF LED；
  - SYS CTRL：将 `Mode_Ctrl` 映射为模式电机目标电压 `sys_ctrl_mode_to_volt_tb[mode]`（除霜有专用档位）；
  - 吹脚电机联动：在含 Foot 的模式下使能（`FOOT_MOTOR_ENABLE`），否则 `DISABLE`。

- 冷暖设定温度
  - UI：从 `AC_Regulation_Temperature` 读取并转三位数显示（带小数点规则与负号处理），同时显示温度图标；
  - SYS CTRL：从 `Cooling_and_heating_air_door_motor_opening(0-250)` 计算目标电压：`TMEP_COLD_HEAT_MIN..MAX` 线性映射并下发至冷暖电机；
  - 传感与算法：具体温控策略不在当前文件内，线程侧只做设定开度到电机位置的映射。

- AC/PTC/循环
  - UI：根据 `AC_Mode_Ctrl`/`Heat_Mode_Ctrl` 设置雪花图标与 LED；
  - 循环开度：0/250 映射为内/外循环两端；中间值根据无反馈开闭状态兜底（保持上一端）。

- 按键与旋钮
  - KEY：各物理键转为 DBC 的 `*_Sw_VALUE_PRESS` 信号；
  - 旋钮：
    - 风量旋钮 → `Bloweing_Increase/Decrease`；
    - 温度旋钮 → `SetTemp_Increase/Decrease`。
  - 长按 OFF → UI 进入 `ERR`。

#### 4. 故障与自检
- 故障检查线程周期执行检查表：电机堵转/传感器异常/电压异常等：
  - 设置系统错误码与状态位；
- UI 在 `ERR` 模式：
  - 关闭所有图标/条形段；
  - 依序显示错误代码（两位数字，第一位固定 `E`），循环若干遍；
  - OFF 退出。

#### 5. 显示通路
- 面板线程维护对象值：
  - 图标布尔、三位七段数、条形段；
- UI 在 RUN 内根据信号更新对象值后调用 `task_control_lcd_panel_update_api()`；
- 面板线程 `DISPLAY` 将对象值映射为 RAM 位并 `pre_draw()`；
- 驱动线程周期将 RAM 批量写入 HT1621；
- IDLE 模式下两个线程均会进入空闲（清屏/关背光）。

#### 6. 执行通路
- SYS CTRL 在 NORMAL 下每周期：
  - 下发风机百分比；
  - 下发模式电机目标电压；
  - 下发冷暖电机目标电压；
  - 下发内外循环电机目标电压（或开闭）；
- 电机线程负责：
  - 反馈型：根据 `get_motor_volt()` 追踪到位（阈值±50mV），堵转超时退出并记录历史方向；
  - 非反馈型：基于定时开合到位。

#### 7. 电压保护策略（ACC）
- `acc_volt()` 根据电压阈值与滞回切换 5 档模式；
- `LOW_STOP/HIGH_STOP`：禁止 CAN 应用收发与设备动作；
- `LOW/HIGH`：受限运行；
- `NORMAL`：全功能运行。

#### 8. 线程协同顺序（典型一次交互）
1) 用户旋钮加风量 → `knob task` 发送 KEY 位图 → `ui task` 写入 `Bloweing_Increase` 信号。
2) `ui task` 触发 `sys ctrl` ON，进入 `RUN`；
3) `sys ctrl task` 读取风量级别 → 映射到百分比 → 调用 `fan task` API；
4) `fan task` 设置 PWM 输出；
5) `ui task` 更新面板条形段与风扇图标 → `lcd panel` 映射 → `lcd drv` 刷新；
6) `temp adc task` 周期更新温度/电压，影响显示与保护；
7) 若异常 → `fault check task` 设置错误码 → `ui task` 可进入 `ERR` 展示。

#### 9. 可配置与扩展点
- 表驱动映射：`sys_ctrl_level_to_per_tb`、`sys_ctrl_mode_to_volt_tb`、`TMEP_COLD_HEAT_MIN/MAX`、`INOUT_MOTOR_MIN/MAX`；
- UI 显示规则：数码位取整/小数点/负号显示逻辑可在 `task_ui.c` 中微调；
- 电机阈值/锁定时间：`±50mV` 与 `TASK_DRV_CTRL_LOCKED_TIME/OPEN_CLOSE_TIME`；
- ACC 阈值：`usr_main.c` 中各电压值与滞回。

#### 10. 结论
该空调面板控制采用“输入→信号→控制→执行→显示”的分层架构：
- 输入统一转 DBC 信号；
- 控制逻辑集中在 `sys ctrl task`，将用户意图映射到设备目标；
- 执行由风机/电机线程完成，具备反馈判定与故障保护；
- 显示由 UI/面板/驱动三层协作，确保界面与状态同步；
- ACC 电压与故障体系提供全局保护与降级策略。
