### 风机线程（fan task）代码说明

本文详细说明 `app/task_drv_crtl_fan.c` 中风机控制线程的设计、状态机、消息流、硬件抽象与对外接口。

#### 1. 线程角色与职责
- 负责鼓风机 PWM 占空比控制：接收风量百分比命令（0-100），设置并维持硬件 PWM 输出。
- 支持在空闲/运行两态间切换，允许随时刷新目标占空比或进入空闲停机。
- 提供对上层（如 `sys ctrl task`）简单可靠的百分比接口，避免重复设置与无效写入。

#### 2. 文件与核心对象
- 源文件：`app/task_drv_crtl_fan.c`
- 核心结构：
  - `StateMachine_Task_withrtos_func task_drv_crtl_fan_tb[] = { idle, run }`：两态状态机表。
  - `struct class_task_drv_ctrl_fan`（作为 `OS_TCB->p_para`）：
    - `QueueHandle_t queue`：消息队列，长度 `TASK_DRV_CTRL_FAN_QENUE_LEN`。
    - `unsigned char crtl_percent`/`crtl_percent_setted`：目标与已应用的百分比。
    - `unsigned char crtl_percent_api_setted`：API 层去抖缓存，避免重复命令。
    - `struct class_task_drv_ctrl_fan_method *pmethod`：硬件抽象方法集。

#### 3. 硬件抽象（HAL Method）
- 由 `usr_main.c` 注入 `class_task_drv_ctrl_fan_method fan_a`：
  - `set_pwm_percent(unsigned char percent)`：设置目标占空比。
  - `set_pwm_stop()`：立刻停止 PWM 输出/关闭驱动管脚。
- 线程不直接依赖底层寄存器，便于移植与单元测试。

#### 4. 状态机设计
- `task_drv_crtl_fan_init(OS_TCB *sm, class_task_drv_ctrl_fan_method *pmethod)`：
  - 绑定 `pmethod`、创建队列、初始化 `crtl_percent_api_setted = 0xff`，注册状态机并进入空闲态。

- `IDLE`（`task_drv_crtl_fan_idle`）：
  - 行为：调用 `pmethod->set_pwm_stop()` 停止输出，将 `crtl_percent_api_setted` 复位为 0xff。
  - 事件处理：阻塞等待 `queue` 消息；当收到 `TASK_DRV_CTRL_CMD_SET_PERCENT` 且 0-100 合法时，设置 `crtl_percent`，切换至 `RUN`。

- `RUN`（`task_drv_crtl_fan_run`）：
  - 进入动作：尝试 `pmethod->set_pwm_percent(crtl_percent)`，失败则退回 `IDLE`。
  - 循环逻辑：
    - 将 `crtl_percent_setted = crtl_percent` 作为当前应用值；
    - 阻塞等待消息：
      - `SET_PERCENT`：当新命令与已应用值不同则再次调用 `set_pwm_percent`，失败回 `IDLE`；
      - `IDLE`：主动切回空闲，停止输出。

#### 5. 消息与 API 接口
- 队列消息类型：
  - `TASK_DRV_CTRL_CMD_SET_PERCENT` + `data[0]=percent`
  - `TASK_DRV_CTRL_CMD_IDLE`
- 对外 API：
  - `task_drv_crtl_fan_set_percent_api(OS_TCB *sm, unsigned char percent)`：
    - 参数校验与 API 级去抖：当 `crtl_percent_api_setted == percent` 时直接返回 true，避免重复覆盖队列；否则更新缓存并 `xQueueOverwrite()` 投递消息。
  - `task_drv_crtl_fan_set_idle_api(OS_TCB *sm)`：投递 `IDLE` 命令，令线程停机。

#### 6. 与系统控制/界面的协作
- `sys ctrl task` 在正常电压下周期读取风量级别 `v` 并调用 `task_drv_crtl_fan_set_percent_api(pfan_task_sm, sys_ctrl_level_to_per_tb[v])`，实现风量闭环设定。
- `ui task` 通过旋钮/按键事件更新 `v`，由 `sys ctrl` 转化为百分比；风机线程专注执行占空比层面的动作。

#### 7. 线程入口与生命周期
- `usr_main.c` 中的 `fan_task(void *pvParameters)`：
  - 分配 `OS_TCB` 与参数对象，设置 `sm->p_para`；
  - 调用 `task_drv_crtl_fan_init(sm, pvParameters)` 完成初始化；
  - 调用 `task_drv_crtl_fan(sm)` 启动状态机（内部 `OS_run_for_rtos`）。

#### 8. 关键代码片段
```c
// 初始化与运行
typedef struct {
    QueueHandle_t queue;
    unsigned char crtl_percent;
    unsigned char crtl_percent_setted;
    unsigned char crtl_percent_api_setted; // API 去抖
    const struct class_task_drv_ctrl_fan_method *pmethod;
} class_task_drv_ctrl_fan;

unsigned char task_drv_crtl_fan_init(void *pvParameters, struct class_task_drv_ctrl_fan_method *pmethod) {
    OS_TCB *sm = (OS_TCB *)pvParameters;
    struct class_task_drv_ctrl_fan *p_para = (struct class_task_drv_ctrl_fan *)sm->p_para;
    p_para->pmethod = pmethod;
    p_para->queue = xQueueCreate(TASK_DRV_CTRL_FAN_QENUE_LEN, sizeof(struct class_msg));
    p_para->crtl_percent_api_setted = 0xff;
    OS_init_for_rtos(sm, (void *)task_drv_crtl_fan_tb, sm->obj, TASK_DRV_CTRL_FAN_TIMER_BASE);
    return p_para->queue != NULL;
}

// 设置百分比 API（去抖 + 覆盖写）
unsigned char task_drv_crtl_fan_set_percent_api(OS_TCB *sm, unsigned char percent){
    if (!sm) return false;
    struct class_task_drv_ctrl_fan *p = (struct class_task_drv_ctrl_fan *)sm->p_para;
    if (p->crtl_percent_api_setted == percent) return true;
    p->crtl_percent_api_setted = percent;
    struct class_msg msg = { .cmd = TASK_DRV_CTRL_CMD_SET_PERCENT, .data = {percent} };
    xQueueOverwrite(p->queue, &msg);
    return true;
}
```

#### 9. 参数与调优建议
- 百分比范围建议限制为 0-100；异常输入在 `IDLE/RUN` 侧直接丢弃。
- 若硬件设置代价较高，可在 `RUN` 中加入“死区判定”（如与 `crtl_percent_setted` 差异小于阈值则忽略），降低频繁更新。
- 如需最小时延，可考虑在 `RUN` 中采用非阻塞接收与更短的轮询周期，权衡 CPU 占用。

#### 10. 调试与故障处理
- 打开 `elog` 检查进入/退出状态与目标值变化。
- 若 `set_pwm_percent` 返回失败：
  - 立即回 `IDLE`，避免维持未知输出；
  - 检查底层 `pmethod` 的 GPIO/PWM 初始化与权限；
  - 核实传入百分比是否越界或硬件通道是否被占用。
