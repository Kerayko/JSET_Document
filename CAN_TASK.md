### CAN 线程（can task）代码说明

本文详解 `app/usr_main.c` 中 `can_task`：协议栈初始化、主循环调用与数据流。

#### 1. 线程职责
- 组装并初始化 CAN 上层组件：`canif`、DBC 编解码、UDS/TP、NM、应用层；
- 在主循环中依次调度各模块的 `run`/`main` 入口，实现收发与业务处理。

#### 2. 初始化
- 构建 `CAN_PDUR_STRUCT_INIT canpdur[CAN_NUM]`：绑定 `canif_init/tx/tx_getstate`；
- DBC：`dbc_tx/rx` 指针与长度；
- UDS：`uds_tb_init` 填写服务表与长度；
- 依次调用：`canpdur_init`、`canuds_dtc_init`（条件编译）、`canuds_init`、`can_nm_init`、`canapp_task_init`；
- 设置软件版本信号等。

#### 3. 主循环
- 以 `vTaskDelay(1)` 为周期，顺序调用：
  - `can_nm_run()`（若启用）；
  - `canuds_tp_main_run()`（TP 层）；
  - `canuds_main_run()`；
  - `canapp_task_run()`（应用）；
  - `canpdur_run()`（PDU 层收发）。

#### 4. 协作
- 与系统控制/UI：通过信号层交互；
- 与底层驱动：`canif` 负责硬件收发接口抽象。

#### 5. 调优
- 合理设置循环延时与各模块的内部缓冲；
- 确认 DBC 表与信号取值范围一致；
- 在 `canif` 层确保发送邮箱与接收 FIFO 的状态被及时处理。
