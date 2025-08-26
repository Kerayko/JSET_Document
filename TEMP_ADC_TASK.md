### 温度 ADC 线程（temp adc task）代码说明

本文详解 `app/task_temp_adc.c` 及 `usr_main.c` 中的传感器读取函数：状态机、函数表、信号写入与时序。

#### 1. 线程职责
- 轮询室内、蒸发、吹面、吹脚温度及 ACC 电压等通道，执行采样、计算与信号写入；
- 维护故障位（超/欠范围）与必要的换算（热敏电阻曲线/分压）。

#### 2. 对象与方法
- `struct task_temp_adc_para`：
  - `QueueHandle_t queue`（长度 1）；`unsigned cnt` 轮询索引；
  - `struct task_temp_adc_method *pmethod`：`temp_func_len`、`type_temp_func *pfunc`。
- 线程入口：`task_temp_adc(ptcb, &temp_adc_method)`。
  - `temp_func[]` 在 `usr_main.c` 中定义并填入：`indoor_temp`、`EVAP_temp`、`FACE_temp`、`foot_temp`、`acc_volt`。

#### 3. 状态机
- `temp_adc_tb[] = { IDLE, RUN }`，初始化后直接进入 RUN。
- `RUN`：
  - 每个周期（`TASK_ADC_TEMP_TIMER_BASE`）检查队列（一般为空），随后执行 `pfunc[cnt](&volt)`，自增 `cnt`；
  - 到达 `temp_func_len` 后 `cnt=0` 循环。

#### 4. 传感器实现要点（见 `usr_main.c`）
- 采样采用 `vPortEnter/ExitCritical()` 包裹，防止被打断；
- 电压换算：`volt = 5000 * adc / 0xfff`（mV）；
- 室内/蒸发/吹面/吹脚温度：电阻换算 → 曲线换算函数 `sys_ctrl_get_*_tempture()` → 限幅/偏移后写入信号；
- ACC：分压比例换算至实际电压，更新系统电压模式（低/正常/高/停发）。

#### 5. 输出
- `sig_wr_*` 系列信号：各温度值、ACC 电压相关状态；
- 相关错误位：`SET/CLR_SYS_ERR_CODE_DATA0(...)`。

#### 6. 调优
- `TASK_ADC_TEMP_TIMER_BASE` 控制采样频率；
- 热敏曲线与分压比常量需与硬件一致；
- 如需抗抖，可在各 `*_temp()` 内增加多次平均。
