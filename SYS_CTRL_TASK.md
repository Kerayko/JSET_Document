### 系统控制线程（sys ctrl task）代码说明

本文详解 `app/task_sys_control.c`：状态机、队列集合、与风机/电机/显示的协作逻辑。

#### 1. 线程职责
- 依据 DBC/信号状态与 ACC 电压模式，统一下发设备控制：风机百分比、模式电机目标、冷暖电机电压、内外循环电机电压等；
- 管理 RUN/IDLE/ERR 工作模式；当电压异常时保护停机。

#### 2. 对象与消息
- `struct class_task_sys_control_para`：
  - `QueueSetHandle_t xQueueSet`；`QueueHandle_t queue[TASK_SYS_CTRL_CMD_TYPE_MAX]`（CMD 等）；
  - `struct task_sys_control_runtine runtine`：运行态（ACC 电压模式等）。
- 主要 API：`task_sys_ctrl_send_cmd_api(ptcb, CMD_ON/OFF/CHECK)`。

#### 3. 状态机
- `sys_ctrl_tb[] = { IDLE, RUN, ERR }`。
- `IDLE`：
  - 关闭所有 LED（由 UI 完成）、令风机/各电机 Idle；等待 CMD：`ON` → RUN。
- `RUN`：
  - 若 `runtine.acc_volt_mode == SYS_VLOT_NORMAL`：
    - 读取风量级别 `v`，调用 `task_drv_crtl_fan_set_percent_api(pfan_task_sm, sys_ctrl_level_to_per_tb[v])`；
    - 模式：读取 `Mode_Ctrl`，按除霜特殊值映射到 `sys_ctrl_mode_to_volt_tb[]` 并 `set_volt_api(pmotor_mode_sm, volt)`；
    - 冷暖：开度 0-250 → 线性映射至 `TMEP_COLD_HEAT_MIN/MAX`，下发至 `pmotor_cold_heat_sm`；
    - 内外循环：开度 0/250 → `INOUT_MOTOR_MIN/MAX`，中间态根据电机开闭状态兜底选择；
  - 否则：令风机与各电机 Idle；
  - CMD：`OFF` → IDLE；`CHECK` → ERR。
- `ERR`：
  - 执行电机自检序列（多段位移），支持 CMD 返回 RUN/IDLE。

#### 4. 协作
- 输入：UI 写入的 DBC 信号；
- 输出：fan/motor 线程 API 调用。

#### 5. 调优
- 适配不同车型映射表：`sys_ctrl_level_to_per_tb`、`sys_ctrl_mode_to_volt_tb`、`INOUT/TEMP` 区间；
- 电压异常阈值与滞回在 `usr_main.c` 的 `acc_volt()` 中维护，可按需调整。
