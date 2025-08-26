### LCD 驱动线程（lcd drv task）代码说明

本文详解 `app/task_drv_ctrl_lcd.c` 中的 LCD 驱动线程：状态机、消息、显存缓冲与对外接口。该线程对接 HT1621 控制器（或等价），提供字节/位写入与重绘能力。

#### 1. 线程角色与职责
- 初始化 LCD 控制器（上电、打开/关闭、背光控制）。
- 维护一份显示 RAM 缓冲区 `ram[]`，周期性将其批量写入 LCD。
- 提供异步接口写单字节、写位与触发重绘；支持进入空闲以关闭显示及背光。

#### 2. 文件与核心对象
- 源文件：`app/task_drv_ctrl_lcd.c`
- 方法注入：`struct task_drv_ctrl_lcd_method`（由 `usr_main.c` 填充）
  - `init`、`lcd_on_off(bool)`、`lcd_backlight(bool)`、`wr_ram(addr, byte)`、`wr_ram_len(start, buf, len)`。
- 线程参数：`struct task_drv_ctrl_lcd_para`（作为 `OS_TCB->p_para`）
  - `QueueHandle_t queue` 单队列（长度 1）。
  - `uint8_t ram[TASK_DRV_CTRL_LCD_DISPLAY_RAM_SIZE]` 本地显存。
  - `check_sun` 用于变化检测（可选）。

#### 3. 状态机
- 表：`task_drv_ctrl_lcd_tb[] = { IDLE, SCAN }`
- 初始化：`task_drv_ctrl_lcd_init(OS_TCB *tcb, method)`
  - 创建队列，调用 `method->init()` 完成底层初始化，`OS_init_for_rtos(..., TASK_DRV_CTRL_LCD_TIMER_BASE)`。

- `IDLE`（`task_drv_ctrl_lcd_idle`）
  - 行为：清空 LCD 显示（逐字节写 0）、关背光与 LCD 主开关；
  - 事件：
    - `TASK_DRV_CTRL_LCD_CMD_WR_DISPLAY_RAM`：更新内存 `ram[idx]=data`，转 `SCAN`；
    - `TASK_DRV_CTRL_LCD_CMD_REDRAW`：转 `SCAN`。

- `SCAN`（`task_drv_ctrl_lcd_display`）
  - 行为：开背光与 LCD；周期 `vTaskDelay(TASK_DRV_CTRL_LCD_TIMER_BASE)` 将 `ram[]` 批量写入：`wr_ram_len(0, ram, SIZE)`；
  - 事件：
    - `TASK_DRV_CTRL_LCD_CMD_IDLE`：转 `IDLE`；
    - `TASK_DRV_CTRL_LCD_CMD_WR_DISPLAY_RAM`：更新 `ram[idx]`。

#### 4. 消息与对外接口
- 消息类型：
  - `TASK_DRV_CTRL_LCD_CMD_WR_DISPLAY_RAM`：`data[0]=addr, data[1]=byte`
  - `TASK_DRV_CTRL_LCD_CMD_REDRAW`
  - `TASK_DRV_CTRL_LCD_CMD_IDLE`
- 对外 API：
  - `task_drv_ctrl_lcd_wr_ram_byte_api(OS_TCB *tcb, addr, data)`：发送写字节消息；
  - `task_drv_ctrl_lcd_wr_ram_byte_no_notfiacate_api(tcb, addr, data)`：直接写内存不发消息；
  - `task_drv_ctrl_lcd_wr_ram_bit_no_notfiacate_api(tcb, addr, offset, enable)`：按位修改；
  - `task_drv_ctrl_lcd_redraw_api(tcb)`：触发重绘；
  - `task_drv_ctrl_lcd_idle_api(tcb)`：请求进入空闲。

#### 5. 与面板线程协作
- `lcd panel task` 负责“控件→显存位”的映射与管理，驱动线程仅负责把 `ram[]` 刷到 LCD。
- 常规流程：面板线程修改 `ram[]`（通过 no_notify API 或队列），随后调用 `task_drv_ctrl_lcd_redraw_api()` 触发 SCAN。

#### 6. 关键代码（节选）
```c
unsigned char task_drv_ctrl_lcd_idle(OS_TCB *psm){
    struct task_drv_ctrl_lcd_para *p = psm->p_para;
    for (int i = 0; i < TASK_DRV_CTRL_LCD_DISPLAY_RAM_SIZE; i++) p->pmethod->wr_ram(i, 0);
    p->pmethod->lcd_backlight(false);
    p->pmethod->lcd_on_off(false);
    for(;;){
        if (xQueueReceive(p->queue, &msg, portMAX_DELAY)){
            if (msg.cmd == TASK_DRV_CTRL_LCD_CMD_WR_DISPLAY_RAM) { p->ram[msg.data[0]] = msg.data[1]; return TASK_DRV_CTRL_LCD_SCAN; }
            else if (msg.cmd == TASK_DRV_CTRL_LCD_CMD_REDRAW) { return TASK_DRV_CTRL_LCD_SCAN; }
        }
    }
}

unsigned char task_drv_ctrl_lcd_display(OS_TCB *psm){
    struct task_drv_ctrl_lcd_para *p = psm->p_para;
    p->pmethod->lcd_backlight(true);
    p->pmethod->lcd_on_off(true);
    for(;;){
        if (xQueueReceive(p->queue, &msg, 0)){
            if (msg.cmd == TASK_DRV_CTRL_LCD_CMD_IDLE) return TASK_DRV_CTRL_LCD_IDLE;
            else if (msg.cmd == TASK_DRV_CTRL_LCD_CMD_WR_DISPLAY_RAM){ if (msg.data[0] < TASK_DRV_CTRL_LCD_DISPLAY_RAM_SIZE) p->ram[msg.data[0]] = msg.data[1]; }
        }
        vTaskDelay(TASK_DRV_CTRL_LCD_TIMER_BASE);
        p->pmethod->wr_ram_len(0, p->ram, TASK_DRV_CTRL_LCD_DISPLAY_RAM_SIZE);
    }
}
```

#### 7. 调优建议
- 若 `ram[]` 变化稀疏，可考虑差分刷新，减少总线写入；
- `wr_ram_len` 批量写能显著降低刷新时间，优先使用；
- 空闲态关闭背光/LCD 可降功耗；
- 队列深度为 1，在高频写入场景建议采用 no_notify 写内存并配合 `REDRAW`。
