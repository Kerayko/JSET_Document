### LCD 面板线程（lcd panel task）代码说明

本文详解 `app/task_control_lcd_panle.c` 中的面板线程：对象模型、位映射、状态机、更新策略与对外接口。该线程将“面板对象状态”转换为底层 LCD 显示 RAM 位。

#### 1. 线程角色与职责
- 维护面板对象（图标、数字、条形段）到 LCD RAM 位的映射关系。
- 响应外部设置请求，更新内部状态并驱动重绘。
- 提供进入空闲/刷新控制，与 `lcd drv task` 配合完成最终显示。

#### 2. 文件与核心对象
- 源文件：`app/task_control_lcd_panle.c`
- 参数结构：`struct task_control_lcd_panel_para`（作为 `OS_TCB->p_para`）
  - `QueueHandle_t queue`：单队列（长度 1）。
  - `struct task_control_lcd_panel_method *pmethod`：方法表，定义对象到位的映射与基础操作：
    - `flag_obj_bool[]`：若干布尔图标（如风扇、除霜、人物等）的位置映射（addr/offset）。
    - `flag_num[]`：多位七段数字的位映射（每位 7 个段地址/偏移）。
    - `flag_cold_seting[]`：条形段（8 段）的位映射数组。
    - `pwr_ram(addr, offset, enable)`：设置单个位；`pre_draw()`：批量重绘；`pidle()`：空闲处理。

#### 3. 状态机
- 表：`task_control_lcd_panel_tb[] = { IDLE, DISPLAY }`
- 初始化：`task_control_lcd_panel_init(OS_TCB *tcb, method)` 创建队列、保存 `method`、注册 `OS_init_for_rtos(..., TASK_CONTROL_LCD_TIMER_BASE)`。

- `IDLE`（`task_control_lcd_panel_idle`）
  - 行为：将所有对象对应位清零（图标位/七段数码/条形段），`pre_draw()` 并调用 `pidle()`；
  - 事件：收到 `TASK_CONTROL_LCD_CMD_SET_OBJ` 返回 `DISPLAY`。

- `DISPLAY`（`task_control_lcd_panel_display`）
  - 行为：
    - 遍历所有对象：
      - 图标：按 `flag_obj_bool[i].value` 调用 `pwr_ram(addr, offset, value)`；
      - 数字：将 `flag_num[k].value` 译码为七段段码，逐段 `pwr_ram(...)`；
      - 条形段：前 `value` 个段置位，其余清零；
    - 完成后 `pre_draw()` 提交帧；
  - 事件：`TASK_CONTROL_LCD_CMD_IDLE` 返回空闲。

#### 4. 对外接口
- 设置对象：`task_control_lcd_panel_set_obj_api(OS_TCB *tcb, obj, value)`
  - `obj < NUM0`：布尔图标（true/false）。
  - `NUM0 <= obj < COLD_SETTING`：七段数字索引（0-9、字母/符号索引）。
  - `COLD_SETTING <= obj < MAX`：条形段（0-8）。
  - 该函数仅更新 `method` 中缓存，不立即绘制；需调用 `task_control_lcd_panel_update_api()` 才会刷新。
- 进入空闲：`task_control_lcd_panel_idle_api(OS_TCB *tcb)` 发送 `IDLE` 命令。
- 触发刷新：`task_control_lcd_panel_update_api(OS_TCB *tcb)` 发送 `SET_OBJ` 命令，驱动 `DISPLAY` 执行映射与 `pre_draw()`。

#### 5. 与驱动线程协作
- 面板线程通过 `pmethod->pwr_ram()`/`pre_draw()` 实际写入到 `lcd drv task` 的显存缓冲（由方法层承接）。
- 典型流程：
  - UI/控制层调用 `set_obj_api()` 批量更新对象值；
  - 调用 `update_api()` 触发一次显示任务，将缓存映射到 RAM，并由 `lcd drv task` 刷新至 LCD 控制器。

#### 6. 数字段码表
- 文件内定义 `drv_lcd_tb` 将数字/字母映射成 7 段段码：`0-9`、`E/O/F`、`a-z` 等，特殊值如 `DRV_LCD_TB_INDEX_HENG_GANG/DRV_LCD_TB_INDEX_NONE` 用于减号/空白。

#### 7. 关键代码（节选）
```c
unsigned char task_control_lcd_panel_display(OS_TCB *sm){
    struct task_control_lcd_panel_para *p = (struct task_control_lcd_panel_para *)sm->p_para;
    for(unsigned obj=0; obj<TASK_CONTROL_LCD_OBJ_INDEX_MAX; obj++){
        if (obj < TASK_CONTROL_LCD_OBJ_INDEX_NUM0){
            p->pmethod->pwr_ram(p->pmethod->flag_obj_bool[obj].addr, p->pmethod->flag_obj_bool[obj].offset, p->pmethod->flag_obj_bool[obj].value);
        } else if (obj < TASK_CONTROL_LCD_OBJ_INDEX_COLD_SETTING){
            uint8_t seg = drv_lcd_tb[p->pmethod->flag_num[obj - TASK_CONTROL_LCD_OBJ_INDEX_NUM0].value];
            for (uint8_t i=0;i<7;i++){
                bool on = (seg & (1u<<i)) != 0;
                p->pmethod->pwr_ram(p->pmethod->flag_num[obj - TASK_CONTROL_LCD_OBJ_INDEX_NUM0].addr_des[i].addr,
                                    p->pmethod->flag_num[obj - TASK_CONTROL_LCD_OBJ_INDEX_NUM0].addr_des[i].offset,
                                    on);
            }
        } else if (obj < TASK_CONTROL_LCD_OBJ_INDEX_MAX){
            for (uint8_t i=0;i<8;i++){
                bool on = p->pmethod->flag_cold_seting[obj - TASK_CONTROL_LCD_OBJ_INDEX_COLD_SETTING].value > i;
                p->pmethod->pwr_ram(p->pmethod->flag_cold_seting[obj - TASK_CONTROL_LCD_OBJ_INDEX_COLD_SETTING].addr_des[i].addr,
                                    p->pmethod->flag_cold_seting[obj - TASK_CONTROL_LCD_OBJ_INDEX_COLD_SETTING].addr_des[i].offset,
                                    on);
            }
        } else break;
    }
    p->pmethod->pre_draw();
    if (xQueueReceive(p->queue, &msg, 100)) { if (msg.cmd == TASK_CONTROL_LCD_CMD_IDLE) return TASK_CONTROL_LCD_STATE_IDLE; }
    return TASK_CONTROL_LCD_STATE_DISPLAY;
}
```

#### 8. 调优建议
- 频繁更新多个对象时，建议批量 `set_obj_api()` 后一次性 `update_api()`，减少多次 `pre_draw()`；
- 面板线程只做“映射”，不要在此加入业务逻辑；
- 当不需要显示时，通过 `idle_api()` 降功耗，与 `lcd drv task` 的 `IDLE` 配合效果更佳。
