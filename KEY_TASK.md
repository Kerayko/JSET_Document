### 按键线程（key task）代码说明

本文详细说明 `app/task_drv_key.c` 中按键线程的设计、状态机、消息流与对外接口。

#### 1. 线程角色与职责
- 负责面板物理按键的扫描、去抖、长按/短按判定与重复按处理。
- 将解析后的键事件转换为 UI 线程可消费的消息，通过 `task_ui_send_key_api()` 进行上报。
- 提供从中断唤醒扫描的接口，缩短按键响应时延。

#### 2. 文件与核心对象
- 源文件：`app/task_drv_key.c`
- 核心结构：
  - `OS_TCB task_drv_key_machine`：按键状态机控制块。
  - `TASK_DRV_KEY_PARA task_drv_key_para`：运行参数（队列、计数器、状态位等）。
  - `QueueHandle_t task_drv_key_para.queue`：长度 1，用于事件/唤醒。

#### 3. 状态机设计
- 状态函数表：`task_drv_key_func[] = { init_idle/idle, init_scan_low/scan_low, init_scan_hight/scan_hight }`
- 状态流转：
  - `INIT_IDLE → IDLE`：初始化 GPIO 与参数，进入空闲等待。
  - `IDLE → SCAN_HIGHT`：
    - 有两种进入方式：
      1) 收到队列消息（来自 `task_drv_key_wakeup_api()` 或其他触发）；
      2) 轮询检测到按键被按下（`task_drv_key_read_port()` 返回非 0）。
    - 进入时清计数器 `cnt`、标志位 `key_msg_send` 等。
  - `SCAN_HIGHT`（按下稳定判定阶段）：
    - 按键值与上次相同且稳定计数自增；
    - `cnt == 2` 判定短按，若非 0 则通过 `task_ui_send_key_api()` 发送一次对应键位；
    - `cnt == 55` 进入连续按的节律节拍（保留位）；
    - `cnt == 300` 产生关机长按键（`TASK_KEY_POS_OFF_LONG`）。
    - 检测到电平变化则重置计数并继续扫描。
  - `INIT_SCAN_LOW → SCAN_LOW`：短按释放后的低电平稳定等待；`cnt` 递减到 0 切回 `IDLE`。

- 关键去抖/计数变量：
  - `p_para->cnt`：稳定计数/超时计数。
  - `p_para->key_pre`：上一个采样键值（整型位域）。
  - `p_para->key_msg_send`：短按消息只发一次的门闩。

#### 4. GPIO 采样与键位编码
- 初始化宏：`DRV_KEY_READ_INIT()`，中断/轮询前置。
- 采样函数：`task_drv_key_read_port()` 逐个读取各物理按键输入，低电平有效时置位到 `key_value`：
  - 依次读取：MODE、AUTO、OFF、DEF、REC、AC_MAX、PTC、AC（详见 GPIO 宏）。
  - 返回值为位图；例如 `1<<TASK_KEY_POS_OFF` 表示 OFF 键按下。

#### 5. 与 UI 线程的消息对接
- 短按事件在 `SCAN_HIGHT` 内达到阈值后，通过：
  - `task_ui_send_key_api(pui_task_tcb, key_bitmap)` 将完整键位位图写入 UI 线程的 `KEY` 队列。
- 长按 OFF 事件：在 `cnt == 300` 发送 `1 << TASK_KEY_POS_OFF_LONG`，用于进入 UI 的自检/错误处理分支。

#### 6. 队列与唤醒机制
- 创建：`task_drv_key_para.queue = xQueueCreate(1, sizeof(struct class_msg))`。
- 空闲态处理：`IDLE` 首先优先尝试从队列阻塞接收唤醒消息，其次轮询 GPIO 降低漏检。
- 中断唤醒：`task_drv_key_wakeup_api()` 在 ISR 中调用 `xQueueOverwriteFromISR()` 写入空消息以唤醒 `IDLE`，并在返回路径中执行一次快速扫描。

#### 7. 线程入口与启动
- 初始化：`task_drv_key_init()` 绑定参数、创建队列、调用 `OS_init()` 注册状态机与定时基准。
- 运行：`task_drv_key_task()` 内部 `for(;;) OS_run(&task_drv_key_machine);` 驱动状态机持续运行。

#### 8. 时序与参数建议
- `sm->timer_base` 为扫描周期的最小时间单位，影响去抖窗口与长按判定：
  - 短按阈值（`cnt==2`）约等于 `2 * timer_base`；
  - 长按阈值（`cnt==300`）约等于 `300 * timer_base`；
  - 连续按临界（`cnt==55`）可用于滚动加减节律。
- 如需优化响应：
  - 在按键 GPIO 中断中调用 `task_drv_key_wakeup_api()`；
  - 适当缩小 `timer_base`，或调整阈值常量以平衡抗抖与延迟。

#### 9. 常用扩展点
- 新增按键：
  - 在 `task_drv_key_read_port()` 中增加读取宏与位映射；
  - 在 UI 侧解析新位到对应信号；
  - 更新 `TASK_KEY_POS_*` 枚举位。
- 调整判定策略：
  - 修改 `SCAN_HIGHT` 中阈值（短按、连按、长按）以适配不同硬件与体验。

#### 10. 关键函数清单与引用
- `task_drv_key_init()`：初始化状态机与队列。
- `task_drv_key_task()`：线程主循环。
- `task_drv_key_func_idle()` / `task_drv_key_func_scan_hight()` / `task_drv_key_func_scan_low()`：三态核心逻辑。
- `task_drv_key_read_port()`：GPIO 采样与位图生成。
- `task_drv_key_wakeup_api()`：ISR 唤醒与快速响应。
- `task_ui_send_key_api()`：向 UI 线程投递键事件。

#### 11. 与其他线程的协作
- 与 `ui task`：通过 `KEY` 队列交互，UI 将键事件翻译为整车信号写入与显示刷新。
- 与 `sys ctrl task`：间接协作，键事件经 UI 产生信号后由系统控制线程下发至设备。

#### 12. 故障与调试建议
- 打开 `elog` 的 `log_v`/`log_i` 以观测状态流转、键值位图与事件发送。
- 若出现误触发：
  - 检查 `DRV_KEY_READ_*` 硬件电平与上拉/下拉配置；
  - 增大短按判定阈值或提高 `timer_base`；
  - 确认 `queue` 未被频繁覆盖导致丢事件（目前短消息仅用于唤醒，不携带键值）。
