### 项目线程一览（FreeRTOS）

以下内容基于 `app/usr_main.c` 内的 `task_init[]` 初始化表与各入口函数实现整理。

#### 说明
- **优先级**以 `configMAX_PRIORITIES - N` 形式表示，数值越大优先级越高。
- **栈大小**为 `xTaskCreate` 传入的 `stack_size`（以 FreeRTOS 字为单位，通常等于 4 字节/字）。

#### 线程列表

- **test**
  - 入口函数: `test_task`
  - 优先级: `configMAX_PRIORITIES - 6`
  - 栈大小: 256
  - 职责: 测试用任务，包含延时与示例调用。

- **spm mgr**
  - 入口函数: `task_spm_mgr`
  - 优先级: `configMAX_PRIORITIES - 6`
  - 栈大小: 256
  - 职责: 定时喂狗 `IWDT_Clr()` 等系统管理。

- **can task**
  - 入口函数: `can_task`
  - 优先级: `configMAX_PRIORITIES - 4`
  - 栈大小: 1024
  - 职责: CAN 驱动/PDU/UDS/NM 等初始化与循环处理（`canpdur_run`、`canuds_*`、`canapp_task_run`）。

- **key task**
  - 入口函数: `key_task`
  - 优先级: `configMAX_PRIORITIES - 1`
  - 栈大小: 256
  - 职责: 按键扫描与处理（`task_drv_key_*`）。

- **knob fan task**
  - 入口函数: `knob_task`（参数为 `task_drv_knob_method_fana`）
  - 优先级: `configMAX_PRIORITIES - 3`
  - 栈大小: 256
  - 职责: 旋钮（风量）处理。

- **knob temp task**
  - 入口函数: `knob_task`（参数为 `task_drv_knob_method_temp`）
  - 优先级: `configMAX_PRIORITIES - 3`
  - 栈大小: 256
  - 职责: 旋钮（温度设定）处理。

- **fan task**
  - 入口函数: `fan_task`（参数 `fan_a`）
  - 优先级: `configMAX_PRIORITIES - 6`
  - 栈大小: 256
  - 职责: 风机 PWM 控制与启动/停止。

- **foot motor task**
  - 入口函数: `foot_task`（参数 `foot_method`）
  - 优先级: `configMAX_PRIORITIES - 6`
  - 栈大小: 256
  - 职责: 吹脚电机控制与电压采样。

- **mode task**
  - 入口函数: `mode_task`（参数 `mode_method`）
  - 优先级: `configMAX_PRIORITIES - 6`
  - 栈大小: 256
  - 职责: 模式电机（风向/模式）控制与电压采样。

- **temp task**
  - 入口函数: `temp_motor_task`（参数 `temp_motor_method`）
  - 优先级: `configMAX_PRIORITIES - 6`
  - 栈大小: 256
  - 职责: 冷暖电机控制与电压采样。

- **inout task**
  - 入口函数: `inout_task`（参数 `inout_method`）
  - 优先级: `configMAX_PRIORITIES - 6`
  - 栈大小: 256
  - 职责: 内外循环电机控制与电压采样。

- **temp adc task**
  - 入口函数: `temp_adc_task`（参数 `temp_adc_method`）
  - 优先级: `configMAX_PRIORITIES - 6`
  - 栈大小: 256
  - 职责: 采集室内/蒸发器/吹面/吹脚温度及 ACC 电压，并写入信号。

- **fault check task**
  - 入口函数: `fault_check_task`（参数 `task_fault_check_method`）
  - 优先级: `configMAX_PRIORITIES - 6`
  - 栈大小: 256
  - 职责: 故障自检表驱动与处理（传感器、电机等）。

- **lcd drv task**
  - 入口函数: `lcd_drv_task`（参数 `lcd_drv_method`）
  - 优先级: `configMAX_PRIORITIES - 1`
  - 栈大小: 256
  - 职责: HT1621 LCD 驱动、背光与显存写入。

- **sys ctrl task**
  - 入口函数: `sys_ctrl_task`（参数 `sys_ctrl_method`）
  - 优先级: `configMAX_PRIORITIES - 5`
  - 栈大小: 256
  - 职责: 系统控制逻辑（运行状态、ACC 模式与收发使能）。

- **task_control_lcd_panle task**
  - 入口函数: `control_lcd_panel_task`（参数 `lcd_panel_method`）
  - 优先级: `configMAX_PRIORITIES - 2`
  - 栈大小: 256
  - 职责: LCD 面板控件状态管理（图标、数字、重绘等）。

- **ui task**
  - 入口函数: `ui_task`
  - 优先级: `configMAX_PRIORITIES - 3`
  - 栈大小: 512
  - 职责: UI 逻辑与交互。

#### 分类视图

- **设备驱动类**: `fan task`、`mode task`、`temp task`、`inout task`、`foot motor task`、`lcd drv task`
- **输入采集类**: `key task`、`knob fan task`、`knob temp task`、`temp adc task`
- **通信协议类**: `can task`
- **系统控制类**: `sys ctrl task`、`spm mgr`
- **界面与显示类**: `task_control_lcd_panle task`、`ui task`
- **诊断与测试类**: `fault check task`、`test`

#### 任务创建位置
所有任务在 `usr_main_init()` 中经由 `xTaskCreate()` 遍历 `task_init[]` 统一创建。
