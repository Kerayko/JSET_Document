### 旋钮线程（knob task）代码说明

本文详细说明 `app/task_drv_knob.c` 中旋钮线程的设计、状态机、方法抽象、消息/时序与对外接口。

#### 1. 线程角色与职责
- 负责面板旋钮（风量/温度）编码器信号的采样、去抖、方向判定与事件上报。
- 将旋转方向映射为 UI 键事件（风量增减、温度增减），供 UI 线程与系统控制使用。
- 采用方法表注入读脚与发送函数，支持风量旋钮与温度旋钮的复用。

#### 2. 文件与核心对象
- 源文件：`app/task_drv_knob.c`
- 关键结构：
  - `task_drv_KNOB_func[]`：三态状态机函数表（init_idle/idle、init_scan_low/scan_low、init_scan_hight/scan_hight）。
  - `TASK_DRV_KNOB_PARA`：运行参数：`cnt`、`err_cnt`、`knob_pre_data`、`knob_low_data`、`pmethod`。
  - `struct task_drv_knob_method`：方法注入（见第 5 节）。
- 线程入口：
  - `task_drv_KNOB_init(OS_TCB *ptcb, TASK_DRV_KNOB_PARA *para)`：绑定参数与方法、初始化状态机（`TASK_DRV_KNOB_TIMER_BASE`）。
  - `task_drv_KNOB_task(OS_TCB *ptcb)`：驱动 `OS_run(ptcb)` 执行状态机。

#### 3. 状态机设计
- 状态流转：`IDLE ↔ SCAN_HIGHT ↔ SCAN_LOW`
  - `INIT_IDLE → IDLE`
    - 通过 `pmethod->read_init()` 做输入引脚准备。
    - 在 `IDLE` 中读取 `pmethod->read_port()` 判定是否进入高电平稳定阶段：
      - 01（A 高、B 低）或 10（A 低、B 高）→ `SCAN_HIGHT`。
  - `INIT_SCAN_HIGHT → SCAN_HIGHT`
    - 设置 `cnt = 2 / timer_base`，`err_cnt = 100 / timer_base`。
    - 在 `SCAN_HIGHT` 中若当前读值与 `knob_pre_data` 一致且 `cnt` 递减至 0，则进入 `SCAN_LOW`；否则重置 `cnt` 去抖。
  - `INIT_SCAN_LOW → SCAN_LOW`
    - 在 `SCAN_LOW` 中读取 `pmethod->read_port()`：
      - 若读值稳定为 11 或 00（两相同态）且与 `knob_low_data` 一致，则 `cnt` 递减；
      - 计时到期后根据前态 `knob_pre_data` 判定方向：
        - 若稳定态为 11，且前态为 01 → 右转；前态为 10 → 左转。
        - 若稳定态为 00，且前态为 01 → 左转；前态为 10 → 右转。
      - 判定后调用 `pmethod->knob_send(direction)` 发送事件，并回到 `IDLE`。
    - 发生异常图样则用 `err_cnt` 限制，超时回 `IDLE`。
- 关键变量：
  - `knob_pre_data`：进入 SCAN_HIGHT 时的两相编码值（01/10）。
  - `knob_low_data`：SCAN_LOW 中的稳定态记录（00/11）。
  - `cnt`：去抖计数；`err_cnt`：异常退出计数。

#### 4. 编码器读值与判向
- 旋钮读取由 `pmethod->read_port()` 提供二位位图：最低两位对应两路编码器相（A/B）。
- 判定策略：
  - 从 01 或 10 进入高稳态，随后在低稳态（00/11）时依据“前态 → 稳态”组合确定左右旋。
  - 使用去抖窗口（`2 / timer_base`）保证稳定采样，防毛刺。

#### 5. 方法抽象与多实例复用
- `struct task_drv_knob_method`：
  - `read_init`：引脚初始化（如 `DRV_KNOB_READ_INIT`）。
  - `read_port`：读取二位编码值（风量用 `task_drv_KNOB_fan_read_port`，温度用 `task_drv_KNOB_temp_read_port`）。
  - `knob_send`：方向结果回调，映射为 UI 键事件：
    - 风量：`task_drv_knob_fan_send(0/1)` → 分别发送 `Bloweing_Increase`/`Bloweing_Decrease`。
    - 温度：`task_drv_knob_temp_send(0/1)` → 分别发送 `TEMP_Increase`/`TEMP_Decrease`。
- 项目中已定义的两套方法：
  - `task_drv_knob_method_fana`（风量旋钮）。
  - `task_drv_knob_method_temp`（温度旋钮）。
- 通过 `usr_main.c` 中创建两个 `knob_task` 实例，分别传入不同 `pmethod` 实现双旋钮支持。

#### 6. 周期与去抖参数
- `timer_base = TASK_DRV_KNOB_TIMER_BASE` 影响去抖与错误窗口：
  - `cnt = 2 / timer_base`：高/低稳态确认时间。
  - `err_cnt = 100 / timer_base`：异常图样容忍时长，防止卡死。
- 可根据硬件旋钮质量与期望手感调整上述阈值。

#### 7. 与 UI 的对接
- 方向事件最终转化为 UI 键事件：
  - 风量：`task_ui_send_key_api(pui_task_tcb, 1 << TASK_KEY_POS_Bloweing_Increase/Decrease)`。
  - 温度：`task_ui_send_key_api(pui_task_tcb, 1 << TASK_KEY_POS_TEMP_Increase/Decrease)`。
- UI 线程在 `IDLE/RUN` 状态均会处理这类事件，进而更新信号、显示与系统控制。

#### 8. 常见扩展与修改点
- 适配新旋钮：实现新的 `read_port` 与 `knob_send`，并定义一个新的 `task_drv_knob_method`。
- 调整灵敏度：修改 `cnt` 与 `err_cnt` 计算或 `timer_base`；必要时增加状态图样验证以进一步抗抖。
- 方向反置：在 `knob_send` 实现内交换左右旋对应的 UI 键事件。

#### 9. 关键函数清单
- `task_drv_KNOB_init()`：绑定参数与方法，初始化状态机。
- `task_drv_KNOB_task()`：执行一次状态机步进（在外层循环中被周期调用）。
- `task_drv_KNOB_func_idle()` / `task_drv_KNOB_func_scan_hight()` / `task_drv_KNOB_func_scan_low()`：三态核心逻辑。
- `task_drv_KNOB_fan_read_port()` / `task_drv_KNOB_temp_read_port()`：读取风量/温度旋钮两路编码。
- `task_drv_knob_fan_send()` / `task_drv_knob_temp_send()`：向 UI 发送键事件。

#### 10. 调试建议
- 打开 `elog` 的 `log_v` 观察 `KNOB_value`、状态转移与发送方向。
- 若出现误判：
  - 检查 A/B 相电平与上拉配置；
  - 增加去抖窗口或提升 `timer_base`；
  - 在 `SCAN_LOW` 加强“前态-稳态”组合校验。
