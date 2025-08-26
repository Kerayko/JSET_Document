### UI 线程（ui task）代码说明

本文详解 `app/task_ui.c`：状态机、消息队列集合、数据流、与显示/控制的协作。

#### 1. 线程职责
- 汇聚按键与旋钮事件，转化为 DBC 信号写入；
- 根据当前信号状态驱动 LCD 面板对象与 LED 指示；
- 管理运行/空闲/自检错误三种模式间的切换。

#### 2. 对象与消息
- `struct task_ui_para`（`OS_TCB->p_para`）：
  - `QueueSetHandle_t xQueueSet`：队列集合；
  - `QueueHandle_t queue[TASK_UI_CMD_TYPE_MAX]`：KEY/CMD/UPDATE_UI 三类队列，各深度 1；
- 主要 API：
  - `task_ui_send_key_api(ptcb, key_bitmap)`：覆盖写 KEY 队列；
  - `task_ui_send_cmd_api(ptcb, cmd)`：覆盖写 CMD（如 ON/OFF）；
  - `task_ui_send_update_ui_api(ptcb)`：覆盖写 UPDATE_UI（状态刷新触发）。

#### 3. 状态机
- `ui_tb[] = { IDLE, RUN, ERR }`，`OS_init_for_rtos(ptcb, ui_tb, obj, TASK_UI_TIMER_BASE)`。
- `IDLE`：
  - 关闭所有 LED，面板进入空闲；写入 `selfchek=OFF`；向系统控制发送 `CMD_OFF`；
  - 等待队列集合事件：KEY 转写 DBC 按键信号；CMD/UPDATE_UI 可转入 RUN；长按 OFF 键转入 ERR。
- `RUN`：
  - 周期读取 DBC 信号（AUTO、温度、风量、模式、AC/PTC、循环等），据此：
    - 设置面板对象（数码、图标、条形段）并 `task_control_lcd_panel_update_api()`；
    - 控制 LED 状态；
  - KEY/CMD/UPDATE_UI 事件引起状态切换或继续刷新；
- `ERR`（自检/错误显示）：
  - `selfchek=ON`；循环拼装错误码序列，轮流在数码位显示，按 OFF 退出回 IDLE。

#### 4. 数据流与协作
- 输入：`key task`/`knob task` → KEY 队列；
- 输出：
  - DBC 信号写入：`sig_wr_*`；
  - 面板显示：`task_control_lcd_panel_*`；
  - 系统控制：`task_sys_ctrl_send_cmd_api()`。

#### 5. 关键代码（节选）
```c
// 队列集合等待与分发
xActivatedMember = xQueueSelectFromSet(p_para->xQueueSet, portMAX_DELAY);
if (xActivatedMember == p_para->queue[TASK_UI_CMD_TYPE_KEY]) {
    xQueueReceive(xActivatedMember, &msg, 0);
    uint32_t key = (msg.data[3]<<24)|(msg.data[2]<<16)|(msg.data[1]<<8)|msg.data[0];
    if (BIT_GET(key, TASK_KEY_POS_OFF_LONG)) return TASK_UI_STATE_ERR;
    // 其余键转对应 sig_wr_*
} else if (xActivatedMember == p_para->queue[TASK_UI_CMD_TYPE_CMD]) {
    // ON/OFF 驱动 RUN/IDLE
} else if (xActivatedMember == p_para->queue[TASK_UI_CMD_TYPE_UPDATE_UI]) {
    // 根据 ON_OFF_Ctrl 状态选择 RUN/IDLE
}
```

#### 6. 调优与建议
- KEY 与旋钮事件优先覆盖写，保证最新输入；
- UI 每次循环后聚合调用一次 `task_control_lcd_panel_update_api()`；
- ERR 模式下忽略 UPDATE_UI，避免干扰错误显示节奏。
