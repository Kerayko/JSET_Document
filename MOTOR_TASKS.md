### 电机线程（motor tasks）代码说明

本文详细说明 `app/task_drv_ctrl_motor.c` 中电机控制线程的设计、状态机、消息流、硬件抽象、故障策略与项目中的实例化（模式/冷暖/内外循环/吹脚）。

#### 1. 线程角色与职责
- 为不同类型电机提供统一的控制框架：
  - 反馈型（Feedback）电机：以电压/位置为目标，闭环靠反馈 `get_motor_volt` 与“差值阈值”对齐停止。
  - 非反馈型（No-Feedback）电机：纯开/关逻辑，依靠超时锁定计时完成动作。
- 管理堵转/历史故障、开闭状态、目标值去抖与动作时序。

#### 2. 文件与核心对象
- 源文件：`app/task_drv_ctrl_motor.c`
- 核心结构：
  - `StateMachine_Task_withrtos_func task_drv_ctrl_motor_tb[] = { idle, run }`：两态状态机。
  - `struct class_task_drv_ctrl_motor`（作为 `OS_TCB->p_para`）：
    - `QueueHandle_t queue`：长度 1，使用 `xQueueOverwrite` 以“后到优先”。
    - `unsigned int ctrl_volt`/`ctrl_volt_setted`/`ctrl_volt_pos`：目标/已应用/反馈位置。
    - `unsigned int cur_setting_api`：API 去抖缓存，避免重复命令。
    - `unsigned char open_close_setting/open_close_cur/open_close_actioning/open_close_unkown`：无反馈型的开闭状态。
    - `unsigned int locked_timer`：动作/堵转锁定计时（单位 `1/timer_base`）。
    - 故障标志：`motor_cur_err`、`motor_history_err`、`motor_history_p`、`motor_history_n`。
    - `struct class_task_drv_ctrl_motor_method *pmethod`：硬件抽象方法集。
- 硬件抽象 `class_task_drv_ctrl_motor_method`：
  - `set_motor_stop/positive/negative`：停止/正转/反转。
  - `get_motor_volt(unsigned int *pvolt)`：读取当前反馈电压/位置。

#### 3. 状态机设计
- 初始化：`task_drv_ctrl_motor_init(OS_TCB *sm, method)`
  - 绑定方法、创建队列、清零历史状态，将 `ctrl_volt_setted/pos` 设为 `0xffffffff`（未知），进入 `IDLE`。

- `IDLE`（`task_drv_ctrl_motor_idle`）：
  - 行为：调用 `set_motor_stop()` 停止输出；清 `cur_setting_api` 为 `0xffffffff`，以允许后续同值再次下发。
  - 反馈型处理：
    - 接收 `TASK_DRV_CTRL_CMD_SET_VOLT`：解包 4 字节 `volt`；如与 `ctrl_volt_setted` 不同，设置 `locked_timer = TASK_DRV_CTRL_LOCKED_TIME / timer_base`，并转入 `RUN`。
    - 接收 `TASK_DRV_CTRL_CMD_IDLE`：保持空闲（用于上层收敛）。
  - 非反馈型处理：
    - 接收 `TASK_DRV_CTRL_CMD_OPEN_CLOSE`：当与当前状态不同或未知，设置 `open_close_setting` 与 `locked_timer = TASK_DRV_CTRL_OPEN_CLOSE_TIME / timer_base`，进入 `RUN`。
    - 接收 `TASK_DRV_CTRL_CMD_IDLE`：保持空闲。

- `RUN`（`task_drv_ctrl_motor_run`）：
  - 反馈型电机：
    - 周期读取反馈 `cur_volt` 并与 `ctrl_volt_setted` 比较：差值 < 50 即认为到位，`set_motor_stop()` 后回 `IDLE`；
    - 否则根据正/负方向调用 `set_motor_positive/negative()` 追踪；
    - 超时 `locked_timer == 0` 判定堵转：记录 `ctrl_volt_pos=cur_volt` 与正/反历史方向位，回 `IDLE`。
    - 运行中再次收到 `SET_VOLT`：若新目标与已设不同，刷新 `ctrl_volt_setted` 与 `locked_timer`。
  - 非反馈型电机：
    - 根据 `open_close_setting` 选择正/反转以达目标；
    - 计时归零后认为完成，置 `open_close_cur = open_close_setting`，回 `IDLE`；
    - 运行中收到新的 `OPEN_CLOSE` 且与当前设定不同，则刷新 `locked_timer` 并继续动作。

#### 4. 消息与对外 API
- 消息类型：
  - 反馈型：`TASK_DRV_CTRL_CMD_SET_VOLT`（`data[0..3]` 为小端 32 位）、`TASK_DRV_CTRL_CMD_IDLE`。
  - 非反馈型：`TASK_DRV_CTRL_CMD_OPEN_CLOSE`（`data[0]` 0/1）、`TASK_DRV_CTRL_CMD_IDLE`。
- API：
  - `task_drv_ctrl_motor_set_volt_api(OS_TCB *sm, unsigned int volt)`：
    - API 去抖：若 `cur_setting_api == volt` 则直接返回；否则更新并 `xQueueOverwrite()` 投递。
  - `task_drv_ctrl_motor_set_open_close_api(OS_TCB *sm, unsigned char onoff)`：同上。
  - `task_drv_ctrl_motor_set_idle_api(OS_TCB *sm)`：令线程空闲。
  - 状态/故障查询：
    - `task_drv_ctrl_motor_get_open_close_state_api()`：无反馈型当前开闭。
    - `task_drv_ctrl_motor_get_feedback_volt_api()`：反馈型最近到位位置。
    - `task_drv_ctrl_motor_get_err_flag_api()` / `clr_err_flag` / `get/ clr_history_err_flag`：故障标志访问。

#### 5. 项目中的线程实例
- 定义位置：`app/usr_main.c`
- 实例化：
  - 模式电机 `mode_task`（反馈型）：方法 `mode_method` 注入了 GPIO 控制与 `mode_motor_get_volt()`（ADC 采样通道 CH10）。
  - 冷暖电机 `temp_motor_task`（反馈型）：方法 `temp_motor_method`（ADC 通道 CH19）。
  - 内外循环电机 `inout_task`（反馈型/按实现为反馈型，电压读取 CH26）。
  - 吹脚电机 `foot_task`（反馈型，电压读取 CH2）。
- 线程入口：各任务分配 `OS_TCB` 与参数，设置 `sm->obj` 区分对象索引，调用 `task_drv_ctrl_motor_init(sm, &method)` 后 `task_drv_ctrl_motor(sm)` 进入状态机。

#### 6. 与系统控制/界面的协作
- `sys ctrl task` 周期计算目标：
  - 风向模式 → `sys_ctrl_mode_to_volt_tb[value]` → 调用 `task_drv_ctrl_motor_set_volt_api(pmotor_mode_sm, volt)`。
  - 冷暖开度（0-250） → 映射到 `TMEP_COLD_HEAT_MIN/MAX` 区间电压后下发。
  - 内外循环开度（0/250） → `INOUT_MOTOR_MIN/MAX` 电压；若为中间态，读取无反馈开闭状态做兜底。
- `ui task` 根据信号状态更新面板图标；电机线程不直接操作显示，仅执行动作。

#### 7. 关键代码片段
```c
// API：设置反馈型电机目标电压
unsigned char task_drv_ctrl_motor_set_volt_api(OS_TCB *sm, unsigned int volt){
    if (!sm) return false;
    struct class_task_drv_ctrl_motor *p = (struct class_task_drv_ctrl_motor *)sm->p_para;
    if (p->cur_setting_api == volt) return false; // 去抖
    p->cur_setting_api = volt;
    struct class_msg msg = {.cmd = TASK_DRV_CTRL_CMD_SET_VOLT};
    msg.data[0] = (volt >> 0) & 0xff;
    msg.data[1] = (volt >> 8) & 0xff;
    msg.data[2] = (volt >> 16) & 0xff;
    msg.data[3] = (volt >> 24) & 0xff;
    xQueueOverwrite(p->queue, &msg);
    return true;
}

// RUN：反馈型核心追踪与堵转处理（节选）
if (p->locked_timer) {
    p->locked_timer--;
} else {
    p->pmethod->get_motor_volt(&cur_volt);
    p->ctrl_volt_pos = cur_volt;
    p->motor_cur_err = true;
    p->motor_history_err = true;
    p->pmethod->set_motor_stop();
    if (cur_volt > p->ctrl_volt_setted) p->motor_history_p = true; else p->motor_history_n = true;
    return TASK_DRV_CTRL_STATE_IDLE;
}
// 到位判断与方向控制
p->pmethod->get_motor_volt(&cur_volt);
if (cur_volt > p->ctrl_volt_setted) {
    if (cur_volt - p->ctrl_volt_setted < 50) { p->ctrl_volt_pos = p->ctrl_volt_setted; p->pmethod->set_motor_stop(); return TASK_DRV_CTRL_STATE_IDLE; }
    else { p->pmethod->set_motor_positive(); }
} else {
    if (p->ctrl_volt_setted - cur_volt < 50) { p->ctrl_volt_pos = p->ctrl_volt_setted; p->pmethod->set_motor_stop(); return TASK_DRV_CTRL_STATE_IDLE; }
    else { p->pmethod->set_motor_negative(); }
}
```

#### 8. 参数与调优建议
- 差值阈值 `50` 可视硬件灵敏度与反馈噪声调整；
- `TASK_DRV_CTRL_LOCKED_TIME` 与 `TASK_DRV_CTRL_OPEN_CLOSE_TIME` 影响到位超时和堵转判定，应结合实际行程与速率配置；
- 建议将 API 去抖保留（`cur_setting_api`），并在上层逻辑层再做限流，降低频繁指令。

#### 9. 调试与故障定位
- 开启 `elog` 的 `log_v/log_e` 观察 `set volt`、`cur_volt`、`locked_timer` 与转向函数调用；
- 堵转出现时读取 `ctrl_volt_pos` 与历史方向位，判断卡滞位置；
- 注意 ADC 读取需在互斥/临界区内进行（源码已做 `vPortEnter/ExitCritical()` 的采样封装）。
