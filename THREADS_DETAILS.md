### 线程功能实现详解（FreeRTOS）

本文件深入说明各线程的状态机、消息机制、周期行为与关键对外接口。依据源码：`app/usr_main.c`、`app/task_*.c`、`app/Control_LCD.c` 等。

#### 约定
- “状态机”基于 `OS_init_for_rtos`/`OS_run_for_rtos` 的两态或多态实现。
- “消息”包括 `QueueHandle_t` 队列和 `QueueSet` 队列集合；大多线程在空闲态阻塞等待消息或定期轮询。
- “周期”表示通过 `timer_base` 或 `vTaskDelay` 控制的循环节拍。

### key task（按键线程）
- 文件：`app/task_drv_key.c`
- 状态机：`IDLE` → `SCAN_HIGHT` → `SCAN_LOW`
  - `task_drv_key_func_init_idle` 初始化 GPIO；`IDLE` 中等待 `queue` 或按键电平触发进入 `SCAN_HIGHT`。
  - `SCAN_HIGHT` 去抖和时序计数：短按在 `cnt==2` 发送键值至 `ui`，长按在 `cnt==300` 发送关机长按键；计数阈值还用于连按处理。
  - `SCAN_LOW` 计时回落，结束后回 `IDLE`。
- 消息：`task_drv_key_para.queue`（长度1，class_msg），支持中断中 `task_drv_key_wakeup_api` 唤醒。
- 周期：在 `SCAN_HIGHT` 中 `vTaskDelay(sm->timer_base)`；`IDLE` 阻塞式 `xQueueReceive` 或轮询 GPIO。
- 关键接口：
  - `task_drv_key_task()` 线程主循环驱动状态机。
  - `task_drv_key_wakeup_api()` ISR 上下文唤醒扫描。
  - `task_ui_send_key_api()` 将解析后的键值投递给 `ui task`。

### knob task（旋钮线程，风量/温度）
- 文件：`app/task_drv_knob.c`
- 状态机：`IDLE` → `SCAN_HIGHT` → `SCAN_LOW`
  - `IDLE` 读取 `pmethod->read_port()` 检测到 01/10 相变化进入 `SCAN_HIGHT`。
  - `SCAN_HIGHT` 稳定计数后进入 `SCAN_LOW`，随后在 `SCAN_LOW` 判断前一状态，确定左右旋并调用 `pmethod->knob_send()`。
- 参数化方法：
  - `task_drv_knob_method_fana`（风量）与 `task_drv_knob_method_temp`（温度）分别封装读脚与发送函数。
  - 发送侧转为 UI 键事件：风量对应 Bloweing Increase/Decrease，温度对应 Temp Increase/Decrease。
- 周期：通过 `sm->timer_base` 控制去抖计数窗口与错误计数。

### fan task（鼓风机控制）
- 文件：`app/task_drv_crtl_fan.c`
- 状态机：`IDLE` ↔ `RUN`
  - `IDLE`：停止 PWM，阻塞等 `SET_PERCENT` 消息。
  - `RUN`：设置并维持 PWM 占空比，收到新的 `SET_PERCENT` 动态刷新；收到 `IDLE` 切回空闲。
- 消息：队列长度 `TASK_DRV_CTRL_FAN_QENUE_LEN`，使用 `xQueueOverwrite` 去抖设置。
- 接口：
  - `task_drv_crtl_fan_set_percent_api(sm, percent)` 设置 0-100 风量。
  - `task_drv_crtl_fan_set_idle_api(sm)` 停止。
- 硬件抽象：`struct class_task_drv_ctrl_fan_method` 提供 `set_pwm_percent`/`set_pwm_stop`，由 `usr_main.c` 注入具体实现。

### motor tasks（模式/冷暖/内外循环/吹脚电机）
- 文件：`app/task_drv_ctrl_motor.c`
- 状态机：`IDLE` ↔ `RUN`
  - 反馈型电机（`TASK_DRV_CTRL_MOTOR_TYPE_FEEDBACK`）：
    - `IDLE` 等 `SET_VOLT`（目标电压/位置）进入 `RUN`，RUN 中周期读取反馈电压 `get_motor_volt`，根据目标差值正/反/停；含“堵转锁定计时”与历史故障记录，达到阈值即判堵转回空闲。
  - 非反馈型电机（`NOFEEDBACK`）：
    - 通过 `OPEN_CLOSE` 开度命令＋定时锁定计数实现动作后回空闲，并维护开闭状态。
- 消息：每实例队列 1 深度；`xQueueOverwrite`。
- 接口：
  - `task_drv_ctrl_motor_set_volt_api(sm, volt)` 设位置/电压。
  - `task_drv_ctrl_motor_set_open_close_api(sm, onoff)` 非反馈开闭。
  - `task_drv_ctrl_motor_set_idle_api(sm)`、状态/故障查询与清除接口若干。
- 硬件抽象：`class_task_drv_ctrl_motor_method` 提供 `set_motor_*` 与 `get_motor_volt`。

### lcd drv task（HT1621 驱动）
- 文件：`app/task_drv_ctrl_lcd.c`
- 状态机：`IDLE` ↔ `SCAN`
  - `IDLE`：清显存、关背光与 LCD，等待写 RAM 或重绘消息。
  - `SCAN`：周期性将内存 `ram[]` 批量写入 LCD，处理 `IDLE`/`WR_DISPLAY_RAM` 消息。
- 消息：单队列（长度1），命令包括 `WR_DISPLAY_RAM`、`REDRAW`、`IDLE`。
- 接口：字节写入、位写入（带/不带通知）、重绘、进入空闲等 API 多个。

### lcd panel task（面板控件渲染）
- 文件：`app/task_control_lcd_panle.c`
- 状态机：`IDLE` ↔ `DISPLAY`
  - `IDLE`：关闭所有图标/段码，重绘并进入空闲，等待 `SET_OBJ`。
  - `DISPLAY`：将方法结构体 `flag_obj_bool`、`flag_num`、`flag_cold_seting` 中的值映射到若干 RAM 位，周期重绘；收到 `IDLE` 返回。
- 数据结构：
  - `flag_obj_bool` 图标映射，`flag_num` 三位七段数码映射，`flag_cold_seting` 多段条形映射。
- 接口：
  - `task_control_lcd_panel_set_obj_api()` 设置对象值，`task_control_lcd_panel_update_api()` 触发刷新，`task_control_lcd_panel_idle_api()` 进入空闲。

### ui task（界面与交互）
- 文件：`app/task_ui.c`
- 状态机：`IDLE` ↔ `RUN` ↔ `ERR`
  - 使用 `QueueSet` 聚合三类队列：`KEY`、`CMD`、`UPDATE_UI`。按键事件转换为 DBC 信号写入；`RUN` 中根据信号驱动面板控件与 LED；`OFF`/状态变更在 `CMD` 或 `UPDATE_UI` 转态。
  - `ERR` 自检模式：序列展示错误码，长按 `OFF` 返回。
- 上下游：
  - 输入：`key task`、`knob task` 产出的键事件；`sys ctrl` 的运行状态变化。
  - 输出：写入多种 DBC 信号；调用 `task_control_lcd_panel_*` 更新显示；LED 控制。

### sys ctrl task（系统控制）
- 文件：`app/task_sys_control.c`
- 状态机：`IDLE` ↔ `RUN` ↔ `ERR`
  - `RUN`：依据 DBC/信号状态与电压状态，统一下发设备控制：
    - 风量映射到鼓风机百分比。
    - 模式映射到模式电机电压；结合除雾等状态特殊处理。
    - 冷暖电机电压按开度映射。
    - 内外循环电机电压按开度或状态设定。
  - ACC 电压异常时进入保护，统一空闲化设备。
  - `ERR`：执行电机自检序列，支持指令返回。
- 消息：`QueueSet` 聚合 `CMD`；提供 `task_sys_ctrl_send_cmd_api()`。
- 运行时：`task_sys_control_runtine` 保存电压模式等运行态。

### temp adc task（温度/电压采集）
- 文件：`app/task_temp_adc.c`（配合 `usr_main.c` 中多路采样函数）
- 状态机：`IDLE` → `RUN`
  - `RUN`：循环按方法表 `pfunc[]` 轮询调用采样函数：室内、蒸发、吹面、吹脚、ACC 电压；各函数内部完成信号写入与告警位维护。
- 周期：由 `TASK_ADC_TEMP_TIMER_BASE` 控制窗口，空队列轮询。

### fault check task（故障检查）
- 文件：`app/task_fault_check.c`
- 状态机：`IDLE` → `CHECK`
  - `CHECK`：轮询方法表 `p_check_tb`，执行 `checkfunc` 并调用 `set_after_check_func(ret)`，覆盖电机/传感器等多项检测。
- 周期：`sm->timer_base` 周期检查。

### can task（通信栈）
- 文件：`app/usr_main.c` 中的 `can_task`
- 内容：初始化 `canpdur`、`uds`、`nm`、`app`，主循环依次执行 `can_nm_run`、`canuds_tp_main_run`、`canuds_main_run`、`canapp_task_run`、`canpdur_run`，周期 `vTaskDelay(1)`。

### spm mgr（简单系统管理）
- 文件：`app/usr_main.c` 中的 `task_spm_mgr`
- 内容：每 1000 tick 喂狗 `IWDT_Clr()`。

### 线程交互与数据流概览
- 输入侧线程：`key task`、`knob task`、`temp adc task`。
- 控制与逻辑：`sys ctrl task` 基于信号统一驱动 `fan/motor` 设备；`ui task` 基于信号驱动显示与 LED。
- 显示侧线程：`lcd drv task`（底层显存刷新）、`lcd panel task`（对象到显存映射）。
- 通信侧：`can task` 负责协议栈与应用交互。
- 维护侧：`fault check task` 周期性检测与错误标志维护，`spm mgr` 喂狗。
