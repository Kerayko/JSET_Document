### 故障检查线程（fault check task）代码说明

本文详解 `app/task_fault_check.c`：检查表驱动、状态机与设置回调。

#### 1. 线程职责
- 周期性执行一组故障检查函数（电机、传感器、电压等），并在检查后调用配套处理函数设置错误码/恢复动作。

#### 2. 对象与方法
- `struct fault_check_para`：
  - `QueueHandle_t queue`（长度 1）；
  - `struct fault_check_methed *pmethod`：包含 `check_tb_len` 与 `fault_check_func` 表（`checkfunc`、`set_after_check_func`）。
- 线程入口：`task_fault_check(psm, &task_fault_check_method)`，方法表在 `usr_main.c` 中组装。

#### 3. 状态机
- `fault_check_tb[] = { IDLE, CHECK }`；初始化后进入 `CHECK`。
- `CHECK`：
  - 以 `sm->timer_base` 为周期，遍历 `p_check_tb`：
    - 若 `checkfunc` 非空，取返回值 `ret`；
    - 若存在 `set_after_check_func`，以 `ret` 为参数执行设置。

#### 4. 常见检查项（见 `usr_main.c`）
- 模式/冷暖/循环/吹脚电机故障；
- 蒸发/室内温度传感器异常；
- ACC 电压异常等（与系统控制/ADC 线程配合）。

#### 5. 调优
- 合理设置 `TASK_FAULT_CHECK_TIMER_BASE`，避免过于频繁；
- 将耗时检查放在方法表末尾，或内部减少阻塞；
- `set_after_check_func` 应尽量无阻塞，仅设置状态与错误位。
