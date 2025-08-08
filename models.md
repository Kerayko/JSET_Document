# 法雷奥空调控制器代码模块化文档

## 目录
1. [系统架构概述](#系统架构概述)
2. [主程序模块](#主程序模块)
3. [应用层模块](#应用层模块)
4. [空调控制模块](#空调控制模块)
5. [舒适性控制模块](#舒适性控制模块)
6. [通信模块](#通信模块)
7. [硬件抽象层模块](#硬件抽象层模块)
8. [诊断模块](#诊断模块)
9. [配置模块](#配置模块)

---

## 系统架构概述

### 整体架构图
```
┌─────────────────────────────────────────────────────────────┐
│                    主程序入口 (main.c)                      │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                    调度器 (SchM)                            │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                    应用层 (APP)                             │
│  ┌─────────────┬─────────────┬─────────────┬─────────────┐  │
│  │   AppMain   │   AppCalc   │   AppCom    │   AppDiag   │  │
│  └─────────────┴─────────────┴─────────────┴─────────────┘  │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                  舒适性控制 (Comfort)                       │
│  ┌─────────────┬─────────────┬─────────────┬─────────────┐  │
│  │     Cmf     │   CmfBlw    │   CmfMix    │   CmfMode   │  │
│  └─────────────┴─────────────┴─────────────┴─────────────┘  │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                  空调控制 (AcCtrl)                          │
│  ┌─────────────┬─────────────┬─────────────┬─────────────┐  │
│  │      Ac     │    AcEDC    │   AcWPTC    │   AcAPTC    │  │
│  └─────────────┴─────────────┴─────────────┴─────────────┘  │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                  通信层 (COM)                               │
│  ┌─────────────┬─────────────┬─────────────┬─────────────┐  │
│  │   COM_CAN   │   COM_LIN   │   CanIf     │   LinIf     │  │
│  └─────────────┴─────────────┴─────────────┴─────────────┘  │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                硬件抽象层 (ECUAL)                           │
│  ┌─────────────┬─────────────┬─────────────┬─────────────┐  │
│  │   Blower    │   Stepper   │   Sensor    │   Power     │  │
│  └─────────────┴─────────────┴─────────────┴─────────────┘  │
└─────────────────────┬───────────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────────┐
│                微控制器抽象层 (MCAL)                         │
│  ┌─────────────┬─────────────┬─────────────┬─────────────┐  │
│  │     ADC     │     DIO     │     GPT     │     SPI     │  │
│  └─────────────┴─────────────┴─────────────┴─────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 主程序模块

### 文件位置
- `Sources/main.c`

### 功能描述
主程序模块是系统的入口点，负责初始化和启动调度器。

### 核心代码

```c
/******************************************************************************/
/* 主程序入口 */
/******************************************************************************/
#include "SchM.h"
#include "StartInit.h"

void main(void)
{
   Start_vidInit();        // 系统初始化
   SchM_vidScheduler();    // 启动调度器
}
```

### 关键特性
- 简洁的入口点设计
- 分离初始化和调度逻辑
- 基于调度器的实时任务管理

---

## 应用层模块

### 模块概述
应用层是系统的核心逻辑层，负责处理业务逻辑、信号处理和状态管理。

### 1. AppMain 模块

#### 文件位置
- `Sources/APP/AppMain.c`
- `Sources/APP/AppMain.h`

#### 功能描述
应用层主控制模块，负责系统状态管理和主要业务流程控制。

#### 核心代码

```c
/******************************************************************************/
/* 应用层主控制模块 */
/******************************************************************************/

// 系统状态枚举
typedef enum
{
    App_APPWORKMODE_UNKNOWN = 0u,
    App_APPWORKMODE_OFF,
    App_APPWORKMODE_ON
} App_tenuAppWorkModeType;

// 主控制函数
extern void AppMain_vidInit(void)
{
    // 初始化系统状态
    App_enuAppWorkMode = App_APPWORKMODE_OFF;
    
    // 初始化各子模块
    AppCalc_vidInit();
    AppCom_vidInit();
    AppDiag_vidInit();
    
    // 初始化舒适性控制
    Cmf_vidInit();
    
    // 初始化空调控制
    Ac_vidInit();
}

// 主管理函数
extern void AppMain_vidManage(void)
{
    // 处理系统状态转换
    switch(App_enuAppWorkMode)
    {
        case App_APPWORKMODE_OFF:
            vidOffHandling(App_enuAppWorkModeLast);
            break;
        case App_APPWORKMODE_ON:
            vidOnHandling();
            break;
        default:
            break;
    }
}

// 快速管理函数 (高频调用)
extern void AppMain_vidFastManage(void)
{
    // 处理实时性要求高的任务
    vidOnHandlingFast();
    
    // 数据计算
    vidDataCalc();
}

// 慢速管理函数 (低频调用)
extern void AppMain_vidSlowManage(void)
{
    // 处理低频任务
    AppDiag_vidManage();
    AppDtc_vidManage();
}
```

### 2. AppCalc 模块

#### 文件位置
- `Sources/APP/AppCalc.c`
- `Sources/APP/AppCalc.h`

#### 功能描述
计算模块，负责温度、风速、功率等参数的计算和转换。

#### 核心代码

```c
/******************************************************************************/
/* 应用层计算模块 */
/******************************************************************************/

// 温度设定值获取宏
#define App_u16GetTsetFL() (((uint16)App_uniAcSetCur.strAcSet.bfTsetFL * 5u) + 160u)
#define App_u16GetTsetFR() (((uint16)App_uniAcSetCur.strAcSet.bfTsetFR * 5u) + 160u)
#define App_u16GetTsetRr() (((uint16)App_uniAcSetRearCur.strAcSetRear.bfRearTset * 5u) + 160u)

// 风速获取宏
#define App_u8GetBlwFr() (App_strAcOut.u8VolBlw_F)
#define App_u8GetBlwRr() (App_strAcOut.u8VolBlw_R)

// 模式获取宏
#define App_u8GetModeFr() (App_strAcOut.u8PosMode_F)
#define App_u8GetModeRr() (App_strAcOut.u8PosMode_R)

// 温度计算函数
extern sint16 AppCalc_s16CalcTemp(uint8 u8TempSet)
{
    sint16 s16Temp;
    
    // 温度设定值转换 (0-63 -> 16-32°C)
    s16Temp = (sint16)((uint16)u8TempSet * 5u) + 160u;
    
    return s16Temp;
}

// 风速计算函数
extern uint8 AppCalc_u8CalcBlowerVoltage(uint8 u8BlowerLevel)
{
    uint8 u8Voltage;
    
    // 风速等级转换为电压值
    if(u8BlowerLevel <= 10u)
    {
        u8Voltage = u8BlowerLevel * 12u;  // 0-10级 -> 0-120V
    }
    else
    {
        u8Voltage = 120u + (u8BlowerLevel - 10u) * 2u;  // 11-17级 -> 122-134V
    }
    
    return u8Voltage;
}
```

### 3. AppCom 模块

#### 文件位置
- `Sources/APP/AppCom.c`
- `Sources/APP/AppCom.h`

#### 功能描述
通信处理模块，负责CAN/LIN信号的接收、解析和发送。

#### 核心代码

```c
/******************************************************************************/
/* 应用层通信模块 */
/******************************************************************************/

// CAN信号接收函数
extern void AppCom_vidReceiveCanSignal(uint32 u32CanId, uint8 *pu8Data)
{
    switch(u32CanId)
    {
        case CAN_ID_AC_SETTINGS:
            // 解析空调设置信号
            AppCom_vidParseAcSettings(pu8Data);
            break;
            
        case CAN_ID_SENSOR_DATA:
            // 解析传感器数据
            AppCom_vidParseSensorData(pu8Data);
            break;
            
        case CAN_ID_HVAC_STATUS:
            // 解析HVAC状态
            AppCom_vidParseHvacStatus(pu8Data);
            break;
            
        default:
            break;
    }
}

// 空调设置信号解析
static void AppCom_vidParseAcSettings(uint8 *pu8Data)
{
    // 解析温度设定值
    App_uniAcSetCur.strAcSet.bfTsetFL = (pu8Data[0] >> 2) & 0x3F;
    App_uniAcSetCur.strAcSet.bfTsetFR = ((pu8Data[0] & 0x03) << 4) | ((pu8Data[1] >> 4) & 0x0F);
    
    // 解析AC开关状态
    App_uniAcSetCur.strAcSet.bfAc = (pu8Data[1] >> 3) & 0x01;
    
    // 解析HVAC模式
    App_uniAcSetCur.strAcSet.bfHvacMode = (pu8Data[1] >> 4) & 0x07;
    
    // 解析鼓风机档位
    App_uniAcSetCur.strAcSet.bfBlw = pu8Data[2] & 0x0F;
}

// CAN信号发送函数
extern void AppCom_vidSendCanSignal(uint32 u32CanId, uint8 *pu8Data, uint8 u8Length)
{
    // 通过CAN接口发送信号
    CanIf_vidTransmit(u32CanId, pu8Data, u8Length);
}
```

### 4. AppDiag 模块

#### 文件位置
- `Sources/APP/AppDiag.c`
- `Sources/APP/AppDiag.h`

#### 功能描述
诊断模块，负责系统诊断、故障检测和状态监控。

#### 核心代码

```c
/******************************************************************************/
/* 应用层诊断模块 */
/******************************************************************************/

// 诊断状态枚举
typedef enum
{
    AppDiag_STATUS_OK = 0u,
    AppDiag_STATUS_WARNING,
    AppDiag_STATUS_ERROR
} AppDiag_tenuStatusType;

// 诊断初始化
extern void AppDiag_vidInit(void)
{
    // 初始化诊断计数器
    AppDiag_u16ErrorCounter = 0u;
    AppDiag_u16WarningCounter = 0u;
    
    // 初始化诊断状态
    AppDiag_enuStatus = AppDiag_STATUS_OK;
}

// 诊断管理函数
extern void AppDiag_vidManage(void)
{
    // 检查传感器状态
    AppDiag_vidCheckSensors();
    
    // 检查执行器状态
    AppDiag_vidCheckActuators();
    
    // 检查通信状态
    AppDiag_vidCheckCommunication();
    
    // 更新诊断状态
    AppDiag_vidUpdateStatus();
}

// 传感器诊断检查
static void AppDiag_vidCheckSensors(void)
{
    // 检查温度传感器
    if(AppDiag_s16CheckTempSensor(Sensor_u8INDOOR_ID) == FALSE)
    {
        AppDiag_vidSetError(AppDiag_ERROR_INDOOR_SENSOR);
    }
    
    // 检查湿度传感器
    if(AppDiag_s16CheckHumiditySensor() == FALSE)
    {
        AppDiag_vidSetWarning(AppDiag_WARNING_HUMIDITY_SENSOR);
    }
}

// 设置诊断错误
static void AppDiag_vidSetError(AppDiag_tenuErrorType enuError)
{
    AppDiag_u16ErrorCounter++;
    AppDiag_enuStatus = AppDiag_STATUS_ERROR;
    
    // 记录错误到DTC
    AppDtc_vidSetDtc(enuError);
}
```

---

## 空调控制模块

### 模块概述
空调控制模块负责整个空调系统的控制逻辑，包括制冷、制热、除湿等功能。

### 1. Ac 主控制模块

#### 文件位置
- `Sources/AcCtrl/Ac.c`
- `Sources/AcCtrl/Ac.h`

#### 功能描述
空调系统主控制模块，协调各个子系统的运行。

#### 核心代码

```c
/******************************************************************************/
/* 空调控制主模块 */
/******************************************************************************/

// 空调系统状态定义
#define Ac_u8ACSYSSTAT_IDLE        0x00u
#define Ac_u8ACSYSSTAT_HEAT        0x01u
#define Ac_u8ACSYSSTAT_COOL        0x02u
#define Ac_u8ACSYSSTAT_HEATCOOL    0x03u

// 空调系统状态变量
static uint8 Ac_u8AcSysStat = Ac_u8ACSYSSTAT_IDLE;

// 空调初始化
extern void Ac_vidInit(void)
{
    // 初始化空调系统状态
    Ac_u8AcSysStat = Ac_u8ACSYSSTAT_IDLE;
    
    // 初始化子模块
    AcEDC_vidInit();
    AcWPTC_vidInit();
    AcAPTC_vidInit();
}

// 空调慢速管理
extern void Ac_vidSlowManage(void)
{
    // 根据舒适性计算结果确定空调系统状态
    Ac_vidDetermineSystemState();
    
    // 根据系统状态调用相应的控制逻辑
    switch(Ac_u8AcSysStat)
    {
        case Ac_u8ACSYSSTAT_IDLE:
            Ac_vidIdleControl();
            break;
            
        case Ac_u8ACSYSSTAT_HEAT:
            Ac_vidHeatControl();
            break;
            
        case Ac_u8ACSYSSTAT_COOL:
            Ac_vidCoolControl();
            break;
            
        case Ac_u8ACSYSSTAT_HEATCOOL:
            Ac_vidHeatCoolControl();
            break;
            
        default:
            break;
    }
}

// 确定系统状态
static void Ac_vidDetermineSystemState(void)
{
    sint16 s16TairTgt = Cmf_s16GetTairTgt();    // 目标空气温度
    sint16 s16Tair = App_s16GetTair();          // 当前空气温度
    sint16 s16TevaTgt = Cmf_s16GetTevaTgt();    // 目标蒸发器温度
    
    if(s16TairTgt > s16Tair)
    {
        // 需要加热
        if(s16TevaTgt < s16Tair)
        {
            // 同时需要制冷和制热
            Ac_u8AcSysStat = Ac_u8ACSYSSTAT_HEATCOOL;
        }
        else
        {
            // 仅需要制热
            Ac_u8AcSysStat = Ac_u8ACSYSSTAT_HEAT;
        }
    }
    else
    {
        // 需要制冷
        Ac_u8AcSysStat = Ac_u8ACSYSSTAT_COOL;
    }
}

// 制热控制
static void Ac_vidHeatControl(void)
{
    // 控制水暖PTC
    AcWPTC_vidControl();
    
    // 控制空气PTC
    AcAPTC_vidControl();
    
    // 控制鼓风机
    CmfBlw_vidControl();
}
```

### 2. AcEDC 模块

#### 文件位置
- `Sources/AcCtrl/AcEDC.c`
- `Sources/AcCtrl/AcEDC.h`

#### 功能描述
电子膨胀阀控制模块，负责制冷系统的流量控制。

#### 核心代码

```c
/******************************************************************************/
/* 电子膨胀阀控制模块 */
/******************************************************************************/

// EDC控制参数
typedef struct
{
    uint8 u8ValvePosition;     // 阀门开度 (0-100%)
    uint8 u8ControlMode;       // 控制模式
    sint16 s16SuperheatTgt;    // 目标过热度
    sint16 s16SuperheatAct;    // 实际过热度
} AcEDC_tstrControlType;

static AcEDC_tstrControlType AcEDC_strControl;

// EDC初始化
extern void AcEDC_vidInit(void)
{
    AcEDC_strControl.u8ValvePosition = 0u;
    AcEDC_strControl.u8ControlMode = AcEDC_MODE_CLOSED;
    AcEDC_strControl.s16SuperheatTgt = 50;  // 5.0°C
    AcEDC_strControl.s16SuperheatAct = 0;
}

// EDC控制函数
extern void AcEDC_vidControl(void)
{
    sint16 s16TevaTgt = Cmf_s16GetTevaTgt();    // 目标蒸发器温度
    sint16 s16TevaAct = Sensor_s16Read(Sensor_u8EVAPORATOR_ID); // 实际蒸发器温度
    sint16 s16TevaIn = Sensor_s16Read(Sensor_u8EVAPORATOR_IN_ID); // 蒸发器入口温度
    
    // 计算实际过热度
    AcEDC_strControl.s16SuperheatAct = s16TevaIn - s16TevaAct;
    
    // PID控制计算阀门开度
    AcEDC_strControl.u8ValvePosition = AcEDC_u8CalcValvePosition(
        s16TevaTgt, 
        s16TevaAct, 
        AcEDC_strControl.s16SuperheatTgt,
        AcEDC_strControl.s16SuperheatAct
    );
    
    // 输出阀门控制信号
    AcEDC_vidSetValvePosition(AcEDC_strControl.u8ValvePosition);
}

// 计算阀门开度
static uint8 AcEDC_u8CalcValvePosition(sint16 s16TevaTgt, sint16 s16TevaAct, 
                                       sint16 s16SuperheatTgt, sint16 s16SuperheatAct)
{
    sint16 s16Error = s16TevaTgt - s16TevaAct;
    sint16 s16SuperheatError = s16SuperheatTgt - s16SuperheatAct;
    uint8 u8Position;
    
    // 简单的比例控制
    u8Position = (uint8)(50u + (s16Error * 2u) + (s16SuperheatError * 3u));
    
    // 限制阀门开度范围
    if(u8Position > 100u)
    {
        u8Position = 100u;
    }
    else if(u8Position < 0u)
    {
        u8Position = 0u;
    }
    
    return u8Position;
}
```

### 3. AcWPTC 模块

#### 文件位置
- `Sources/AcCtrl/AcWPTC.c`
- `Sources/AcCtrl/AcWPTC.h`

#### 功能描述
水暖PTC控制模块，负责水循环加热系统的控制。

#### 核心代码

```c
/******************************************************************************/
/* 水暖PTC控制模块 */
/******************************************************************************/

// WPTC控制参数
typedef struct
{
    uint16 u16PowerLevel;      // 功率等级 (0-100%)
    uint8 u8ControlMode;       // 控制模式
    sint16 s16TwatTgt;         // 目标水温
    sint16 s16TwatAct;         // 实际水温
} AcWPTC_tstrControlType;

static AcWPTC_tstrControlType AcWPTC_strControl;

// WPTC初始化
extern void AcWPTC_vidInit(void)
{
    AcWPTC_strControl.u16PowerLevel = 0u;
    AcWPTC_strControl.u8ControlMode = AcWPTC_MODE_OFF;
    AcWPTC_strControl.s16TwatTgt = 800;  // 80.0°C
    AcWPTC_strControl.s16TwatAct = 0;
}

// WPTC控制函数
extern void AcWPTC_vidControl(void)
{
    sint16 s16TwatTgt = Cmf_s16GetTwatTgt();    // 目标水温
    sint16 s16TwatAct = App_s16GetTwat();        // 实际水温
    uint8 u8WPTCAllowed = App_u8GetWPTCAlotd(); // PTC允许功率
    
    AcWPTC_strControl.s16TwatTgt = s16TwatTgt;
    AcWPTC_strControl.s16TwatAct = s16TwatAct;
    
    // 计算所需功率
    AcWPTC_strControl.u16PowerLevel = AcWPTC_u16CalcPowerLevel(
        s16TwatTgt, 
        s16TwatAct, 
        u8WPTCAllowed
    );
    
    // 输出功率控制信号
    AcWPTC_vidSetPowerLevel(AcWPTC_strControl.u16PowerLevel);
}

// 计算功率等级
static uint16 AcWPTC_u16CalcPowerLevel(sint16 s16TwatTgt, sint16 s16TwatAct, uint8 u8Allowed)
{
    sint16 s16Error = s16TwatTgt - s16TwatAct;
    uint16 u16PowerLevel;
    
    if(s16Error > 0)
    {
        // 需要加热
        u16PowerLevel = (uint16)((s16Error * 10u) / 10u);  // 每度温差10%功率
        
        // 限制在允许功率范围内
        if(u16PowerLevel > (uint16)u8Allowed)
        {
            u16PowerLevel = (uint16)u8Allowed;
        }
    }
    else
    {
        // 不需要加热
        u16PowerLevel = 0u;
    }
    
    return u16PowerLevel;
}
```

---

## 舒适性控制模块

### 模块概述
舒适性控制模块负责计算和优化乘客的舒适性体验，包括温度、湿度、风速等参数的综合控制。

### 1. Cmf 主控制模块

#### 文件位置
- `Sources/Comfort/Cmf.c`
- `Sources/Comfort/Cmf.h`

#### 功能描述
舒适性控制主接口，协调各个舒适性子模块的运行。

#### 核心代码

```c
/******************************************************************************/
/* 舒适性控制主模块 */
/******************************************************************************/

// 舒适性控制状态
typedef struct
{
    sint16 s16TairTgt;         // 目标空气温度
    sint16 s16TevaTgt;         // 目标蒸发器温度
    sint16 s16TwatTgt;         // 目标水温
    uint8 u8BlowerLevel;       // 鼓风机档位
    uint8 u8ModePosition;      // 模式风门位置
    uint8 u8MixPosition;       // 混合风门位置
} Cmf_tstrControlType;

static Cmf_tstrControlType Cmf_strControl;

// 舒适性初始化
extern void Cmf_vidInit(void)
{
    // 初始化舒适性控制参数
    Cmf_strControl.s16TairTgt = 2200;  // 22.0°C
    Cmf_strControl.s16TevaTgt = 500;   // 5.0°C
    Cmf_strControl.s16TwatTgt = 800;   // 80.0°C
    Cmf_strControl.u8BlowerLevel = 5u;
    Cmf_strControl.u8ModePosition = 50u;
    Cmf_strControl.u8MixPosition = 50u;
    
    // 初始化子模块
    CmfBlw_vidInit();
    CmfMix_vidInit();
    CmfMode_vidInit();
    CmfRec_vidInit();
    CmfCal_vidInit();
}

// 舒适性管理函数
extern void Cmf_vidManage(void)
{
    // 计算舒适性参数
    CmfCal_vidCalculate();
    
    // 更新控制参数
    Cmf_strControl.s16TairTgt = CmfCal_s16GetTairTgt();
    Cmf_strControl.s16TevaTgt = CmfCal_s16GetTevaTgt();
    Cmf_strControl.s16TwatTgt = CmfCal_s16GetTwatTgt();
    
    // 控制各个子系统
    CmfBlw_vidControl();
    CmfMix_vidControl();
    CmfMode_vidControl();
    CmfRec_vidControl();
}

// 获取目标空气温度
extern sint16 Cmf_s16GetTairTgt(void)
{
    return Cmf_strControl.s16TairTgt;
}

// 获取目标蒸发器温度
extern sint16 Cmf_s16GetTevaTgt(void)
{
    return Cmf_strControl.s16TevaTgt;
}

// 获取目标水温
extern sint16 Cmf_s16GetTwatTgt(void)
{
    return Cmf_strControl.s16TwatTgt;
}
```

### 2. CmfBlw 鼓风机控制模块

#### 文件位置
- `Sources/Comfort/CmfBlw.c`
- `Sources/Comfort/CmfBlw.h`

#### 功能描述
鼓风机控制模块，负责前/后排鼓风机的转速控制和风量调节。

#### 核心代码

```c
/******************************************************************************/
/* 鼓风机控制模块 */
/******************************************************************************/

// 鼓风机控制参数
typedef struct
{
    uint8 u8FrontLevel;        // 前排鼓风机档位 (0-17)
    uint8 u8RearLevel;         // 后排鼓风机档位 (0-17)
    uint8 u8FrontVoltage;      // 前排鼓风机电压 (0-134V)
    uint8 u8RearVoltage;       // 后排鼓风机电压 (0-134V)
    uint8 u8ControlMode;       // 控制模式
} CmfBlw_tstrControlType;

static CmfBlw_tstrControlType CmfBlw_strControl;

// 鼓风机初始化
extern void CmfBlw_vidInit(void)
{
    CmfBlw_strControl.u8FrontLevel = 0u;
    CmfBlw_strControl.u8RearLevel = 0u;
    CmfBlw_strControl.u8FrontVoltage = 0u;
    CmfBlw_strControl.u8RearVoltage = 0u;
    CmfBlw_strControl.u8ControlMode = CmfBlw_MODE_OFF;
}

// 鼓风机控制函数
extern void CmfBlw_vidControl(void)
{
    uint8 u8FrontLevel = App_u8GetBlwFr();  // 获取前排鼓风机档位
    uint8 u8RearLevel = App_u8GetBlwRr();   // 获取后排鼓风机档位
    
    // 计算鼓风机电压
    CmfBlw_strControl.u8FrontVoltage = CmfBlw_u8CalcVoltage(u8FrontLevel);
    CmfBlw_strControl.u8RearVoltage = CmfBlw_u8CalcVoltage(u8RearLevel);
    
    // 输出控制信号
    Blower_vidSetFrontVoltage(CmfBlw_strControl.u8FrontVoltage);
    Blower_vidSetRearVoltage(CmfBlw_strControl.u8RearVoltage);
}

// 计算鼓风机电压
static uint8 CmfBlw_u8CalcVoltage(uint8 u8Level)
{
    uint8 u8Voltage;
    
    if(u8Level <= 10u)
    {
        // 0-10级：线性增加 (0-120V)
        u8Voltage = u8Level * 12u;
    }
    else
    {
        // 11-17级：继续增加 (122-134V)
        u8Voltage = 120u + (u8Level - 10u) * 2u;
    }
    
    return u8Voltage;
}
```

### 3. CmfMix 混合风门控制模块

#### 文件位置
- `Sources/Comfort/CmfMix.c`
- `Sources/Comfort/CmfMix.h`

#### 功能描述
混合风门控制模块，负责冷热空气的混合比例控制。

#### 核心代码

```c
/******************************************************************************/
/* 混合风门控制模块 */
/******************************************************************************/

// 混合风门控制参数
typedef struct
{
    uint8 u8FrontLeftPos;      // 前左混合风门位置 (0-100%)
    uint8 u8FrontRightPos;     // 前右混合风门位置 (0-100%)
    uint8 u8RearLeftPos;       // 后左混合风门位置 (0-100%)
    uint8 u8RearRightPos;      // 后右混合风门位置 (0-100%)
    sint16 s16TsetFL;          // 前左温度设定
    sint16 s16TsetFR;          // 前右温度设定
    sint16 s16TsetRL;          // 后左温度设定
    sint16 s16TsetRR;          // 后右温度设定
} CmfMix_tstrControlType;

static CmfMix_tstrControlType CmfMix_strControl;

// 混合风门初始化
extern void CmfMix_vidInit(void)
{
    CmfMix_strControl.u8FrontLeftPos = 50u;
    CmfMix_strControl.u8FrontRightPos = 50u;
    CmfMix_strControl.u8RearLeftPos = 50u;
    CmfMix_strControl.u8RearRightPos = 50u;
}

// 混合风门控制函数
extern void CmfMix_vidControl(void)
{
    // 获取温度设定值
    CmfMix_strControl.s16TsetFL = App_u16GetTsetFL();
    CmfMix_strControl.s16TsetFR = App_u16GetTsetFR();
    CmfMix_strControl.s16TsetRL = App_u16GetTsetRr();
    CmfMix_strControl.s16TsetRR = App_u16GetTsetRr();
    
    // 计算混合风门位置
    CmfMix_strControl.u8FrontLeftPos = CmfMix_u8CalcPosition(
        CmfMix_strControl.s16TsetFL, 
        CmfMix_s16GetTairFL()
    );
    
    CmfMix_strControl.u8FrontRightPos = CmfMix_u8CalcPosition(
        CmfMix_strControl.s16TsetFR, 
        CmfMix_s16GetTairFR()
    );
    
    // 输出控制信号
    Stepper_vidSetMixFL(CmfMix_strControl.u8FrontLeftPos);
    Stepper_vidSetMixFR(CmfMix_strControl.u8FrontRightPos);
}

// 计算混合风门位置
static uint8 CmfMix_u8CalcPosition(sint16 s16Tset, sint16 s16Tair)
{
    sint16 s16Error = s16Tset - s16Tair;
    uint8 u8Position;
    
    // 基于温度误差计算混合比例
    if(s16Error > 0)
    {
        // 需要加热，增加热风比例
        u8Position = (uint8)(50u + (s16Error * 2u));
    }
    else
    {
        // 需要制冷，增加冷风比例
        u8Position = (uint8)(50u + (s16Error * 2u));
    }
    
    // 限制位置范围
    if(u8Position > 100u)
    {
        u8Position = 100u;
    }
    else if(u8Position < 0u)
    {
        u8Position = 0u;
    }
    
    return u8Position;
}
```

### 4. CmfMode 模式风门控制模块

#### 文件位置
- `Sources/Comfort/CmfMode.c`
- `Sources/Comfort/CmfMode.h`

#### 功能描述
模式风门控制模块，负责出风模式的控制（面部、脚部、除霜等）。

#### 核心代码

```c
/******************************************************************************/
/* 模式风门控制模块 */
/******************************************************************************/

// 模式风门控制参数
typedef struct
{
    uint8 u8FrontMode;         // 前排模式风门位置
    uint8 u8RearMode;          // 后排模式风门位置
    uint8 u8ModeType;          // 模式类型
} CmfMode_tstrControlType;

static CmfMode_tstrControlType CmfMode_strControl;

// 模式类型定义
#define CmfMode_TYPE_FACE       0u    // 面部模式
#define CmfMode_TYPE_FOOT       1u    // 脚部模式
#define CmfMode_TYPE_DEFROST    2u    // 除霜模式
#define CmfMode_TYPE_MIX        3u    // 混合模式

// 模式风门初始化
extern void CmfMode_vidInit(void)
{
    CmfMode_strControl.u8FrontMode = 50u;
    CmfMode_strControl.u8RearMode = 50u;
    CmfMode_strControl.u8ModeType = CmfMode_TYPE_FACE;
}

// 模式风门控制函数
extern void CmfMode_vidControl(void)
{
    uint8 u8ModeFr = App_u8GetModeFr();  // 获取前排模式
    uint8 u8ModeRr = App_u8GetModeRr();  // 获取后排模式
    
    // 根据模式类型设置风门位置
    CmfMode_strControl.u8FrontMode = CmfMode_u8GetModePosition(u8ModeFr);
    CmfMode_strControl.u8RearMode = CmfMode_u8GetModePosition(u8ModeRr);
    
    // 输出控制信号
    Stepper_vidSetModeF(CmfMode_strControl.u8FrontMode);
    Stepper_vidSetModeR(CmfMode_strControl.u8RearMode);
}

// 获取模式位置
static uint8 CmfMode_u8GetModePosition(uint8 u8Mode)
{
    uint8 u8Position;
    
    switch(u8Mode)
    {
        case CmfMode_TYPE_FACE:
            u8Position = 0u;    // 面部位置
            break;
            
        case CmfMode_TYPE_FOOT:
            u8Position = 100u;  // 脚部位置
            break;
            
        case CmfMode_TYPE_DEFROST:
            u8Position = 0u;    // 除霜位置
            break;
            
        case CmfMode_TYPE_MIX:
            u8Position = 50u;   // 混合位置
            break;
            
        default:
            u8Position = 50u;
            break;
    }
    
    return u8Position;
}
```

---

## 通信模块

### 模块概述
通信模块负责CAN和LIN总线的通信处理，包括信号接收、解析、发送和诊断通信。

### 1. COM_CAN 模块

#### 文件位置
- `Sources/COM_CAN/`

#### 功能描述
CAN总线通信模块，包括CAN驱动、接口、诊断和传输协议。

#### 核心代码

```c
/******************************************************************************/
/* CAN通信模块 */
/******************************************************************************/

// CAN信号ID定义
#define CAN_ID_AC_SETTINGS      0x123u
#define CAN_ID_SENSOR_DATA      0x456u
#define CAN_ID_HVAC_STATUS      0x789u
#define CAN_ID_DIAGNOSTIC       0xABCu

// CAN接收处理函数
extern void Can_vidReceiveHandler(uint32 u32CanId, uint8 *pu8Data, uint8 u8Length)
{
    switch(u32CanId)
    {
        case CAN_ID_AC_SETTINGS:
            Can_vidProcessAcSettings(pu8Data, u8Length);
            break;
            
        case CAN_ID_SENSOR_DATA:
            Can_vidProcessSensorData(pu8Data, u8Length);
            break;
            
        case CAN_ID_HVAC_STATUS:
            Can_vidProcessHvacStatus(pu8Data, u8Length);
            break;
            
        case CAN_ID_DIAGNOSTIC:
            Can_vidProcessDiagnostic(pu8Data, u8Length);
            break;
            
        default:
            break;
    }
}

// 处理空调设置信号
static void Can_vidProcessAcSettings(uint8 *pu8Data, uint8 u8Length)
{
    if(u8Length >= 4u)
    {
        // 解析温度设定值
        uint8 u8TsetFL = (pu8Data[0] >> 2) & 0x3F;
        uint8 u8TsetFR = ((pu8Data[0] & 0x03) << 4) | ((pu8Data[1] >> 4) & 0x0F);
        
        // 解析AC开关状态
        uint8 u8AcSwitch = (pu8Data[1] >> 3) & 0x01;
        
        // 解析HVAC模式
        uint8 u8HvacMode = (pu8Data[1] >> 4) & 0x07;
        
        // 更新应用层数据
        App_uniAcSetCur.strAcSet.bfTsetFL = u8TsetFL;
        App_uniAcSetCur.strAcSet.bfTsetFR = u8TsetFR;
        App_uniAcSetCur.strAcSet.bfAc = u8AcSwitch;
        App_uniAcSetCur.strAcSet.bfHvacMode = u8HvacMode;
    }
}

// CAN发送函数
extern void Can_vidTransmit(uint32 u32CanId, uint8 *pu8Data, uint8 u8Length)
{
    // 通过CAN接口发送数据
    CanIf_vidTransmit(u32CanId, pu8Data, u8Length);
}
```

### 2. COM_LIN 模块

#### 文件位置
- `Sources/COM_LIN/`

#### 功能描述
LIN总线通信模块，用于与从设备通信。

#### 核心代码

```c
/******************************************************************************/
/* LIN通信模块 */
/******************************************************************************/

// LIN节点ID定义
#define LIN_NODE_MASTER         0u
#define LIN_NODE_SLAVE1         1u
#define LIN_NODE_SLAVE2         2u

// LIN消息处理函数
extern void Lin_vidMessageHandler(uint8 u8NodeId, uint8 *pu8Data, uint8 u8Length)
{
    switch(u8NodeId)
    {
        case LIN_NODE_SLAVE1:
            Lin_vidProcessSlave1Data(pu8Data, u8Length);
            break;
            
        case LIN_NODE_SLAVE2:
            Lin_vidProcessSlave2Data(pu8Data, u8Length);
            break;
            
        default:
            break;
    }
}

// 处理从设备1数据
static void Lin_vidProcessSlave1Data(uint8 *pu8Data, uint8 u8Length)
{
    if(u8Length >= 2u)
    {
        // 解析传感器数据
        uint8 u8SensorValue = pu8Data[0];
        uint8 u8SensorStatus = pu8Data[1];
        
        // 更新传感器状态
        Sensor_vidUpdateStatus(LIN_NODE_SLAVE1, u8SensorValue, u8SensorStatus);
    }
}
```

---

## 硬件抽象层模块

### 模块概述
硬件抽象层模块提供对硬件设备的统一接口，包括传感器、执行器、电源管理等。

### 1. Blower 鼓风机模块

#### 文件位置
- `Sources/ECUAL/Blower.c`
- `Sources/ECUAL/Blower.h`

#### 功能描述
鼓风机硬件控制模块，提供对前/后排鼓风机的控制接口。

#### 核心代码

```c
/******************************************************************************/
/* 鼓风机硬件控制模块 */
/******************************************************************************/

// 鼓风机状态
typedef struct
{
    uint8 u8FrontVoltage;      // 前排鼓风机电压
    uint8 u8RearVoltage;       // 后排鼓风机电压
    uint8 u8FrontStatus;       // 前排鼓风机状态
    uint8 u8RearStatus;        // 后排鼓风机状态
    uint16 u16FrontSpeed;      // 前排鼓风机转速
    uint16 u16RearSpeed;       // 后排鼓风机转速
} Blower_tstrStatusType;

static Blower_tstrStatusType Blower_strStatus;

// 鼓风机初始化
extern void Blower_vidInit(void)
{
    Blower_strStatus.u8FrontVoltage = 0u;
    Blower_strStatus.u8RearVoltage = 0u;
    Blower_strStatus.u8FrontStatus = Blower_STATUS_OFF;
    Blower_strStatus.u8RearStatus = Blower_STATUS_OFF;
    Blower_strStatus.u16FrontSpeed = 0u;
    Blower_strStatus.u16RearSpeed = 0u;
    
    // 初始化硬件
    Blower_vidInitHardware();
}

// 设置前排鼓风机电压
extern void Blower_vidSetFrontVoltage(uint8 u8Voltage)
{
    Blower_strStatus.u8FrontVoltage = u8Voltage;
    
    if(u8Voltage > 0u)
    {
        Blower_strStatus.u8FrontStatus = Blower_STATUS_ON;
        // 输出PWM信号控制鼓风机
        PWM_vidSetDutyCycle(PWM_CHANNEL_FRONT_BLOWER, u8Voltage);
    }
    else
    {
        Blower_strStatus.u8FrontStatus = Blower_STATUS_OFF;
        PWM_vidSetDutyCycle(PWM_CHANNEL_FRONT_BLOWER, 0u);
    }
}

// 设置后排鼓风机电压
extern void Blower_vidSetRearVoltage(uint8 u8Voltage)
{
    Blower_strStatus.u8RearVoltage = u8Voltage;
    
    if(u8Voltage > 0u)
    {
        Blower_strStatus.u8RearStatus = Blower_STATUS_ON;
        // 输出PWM信号控制鼓风机
        PWM_vidSetDutyCycle(PWM_CHANNEL_REAR_BLOWER, u8Voltage);
    }
    else
    {
        Blower_strStatus.u8RearStatus = Blower_STATUS_OFF;
        PWM_vidSetDutyCycle(PWM_CHANNEL_REAR_BLOWER, 0u);
    }
}

// 获取鼓风机状态
extern uint8 Blower_u8GetFrontStatus(void)
{
    return Blower_strStatus.u8FrontStatus;
}

extern uint8 Blower_u8GetRearStatus(void)
{
    return Blower_strStatus.u8RearStatus;
}
```

### 2. Stepper 步进电机模块

#### 文件位置
- `Sources/ECUAL/Stepper.c`
- `Sources/ECUAL/Stepper.h`

#### 功能描述
步进电机控制模块，用于控制各种风门的位置。

#### 核心代码

```c
/******************************************************************************/
/* 步进电机控制模块 */
/******************************************************************************/

// 步进电机状态
typedef struct
{
    uint8 u8MixFLPosition;     // 前左混合风门位置
    uint8 u8MixFRPosition;     // 前右混合风门位置
    uint8 u8ModeFPosition;     // 前排模式风门位置
    uint8 u8ModeRPosition;     // 后排模式风门位置
    uint8 u8RecPosition;       // 循环风门位置
    uint8 u8Status;            // 电机状态
} Stepper_tstrStatusType;

static Stepper_tstrStatusType Stepper_strStatus;

// 步进电机初始化
extern void Stepper_vidInit(void)
{
    Stepper_strStatus.u8MixFLPosition = 50u;
    Stepper_strStatus.u8MixFRPosition = 50u;
    Stepper_strStatus.u8ModeFPosition = 50u;
    Stepper_strStatus.u8ModeRPosition = 50u;
    Stepper_strStatus.u8RecPosition = 50u;
    Stepper_strStatus.u8Status = Stepper_STATUS_READY;
    
    // 初始化硬件
    Stepper_vidInitHardware();
}

// 设置前左混合风门位置
extern void Stepper_vidSetMixFL(uint8 u8Position)
{
    if(u8Position <= 100u)
    {
        Stepper_strStatus.u8MixFLPosition = u8Position;
        Stepper_vidMoveToPosition(STEPPER_MIX_FL, u8Position);
    }
}

// 设置前右混合风门位置
extern void Stepper_vidSetMixFR(uint8 u8Position)
{
    if(u8Position <= 100u)
    {
        Stepper_strStatus.u8MixFRPosition = u8Position;
        Stepper_vidMoveToPosition(STEPPER_MIX_FR, u8Position);
    }
}

// 设置前排模式风门位置
extern void Stepper_vidSetModeF(uint8 u8Position)
{
    if(u8Position <= 100u)
    {
        Stepper_strStatus.u8ModeFPosition = u8Position;
        Stepper_vidMoveToPosition(STEPPER_MODE_F, u8Position);
    }
}

// 移动到指定位置
static void Stepper_vidMoveToPosition(uint8 u8MotorId, uint8 u8TargetPosition)
{
    uint8 u8CurrentPosition;
    uint8 u8Direction;
    uint16 u16Steps;
    
    // 获取当前位置
    u8CurrentPosition = Stepper_u8GetCurrentPosition(u8MotorId);
    
    // 计算移动方向和步数
    if(u8TargetPosition > u8CurrentPosition)
    {
        u8Direction = Stepper_DIRECTION_FORWARD;
        u16Steps = (uint16)(u8TargetPosition - u8CurrentPosition) * 10u;
    }
    else
    {
        u8Direction = Stepper_DIRECTION_BACKWARD;
        u16Steps = (uint16)(u8CurrentPosition - u8TargetPosition) * 10u;
    }
    
    // 执行移动
    Stepper_vidMoveMotor(u8MotorId, u8Direction, u16Steps);
}
```

### 3. Sensor 传感器模块

#### 文件位置
- `Sources/ECUAL/Sensor.c`
- `Sources/ECUAL/Sensor.h`

#### 功能描述
传感器接口模块，提供对各种温度、湿度传感器的统一访问接口。

#### 核心代码

```c
/******************************************************************************/
/* 传感器接口模块 */
/******************************************************************************/

// 传感器ID定义
#define Sensor_u8INDOOR_ID      0u    // 室内温度传感器
#define Sensor_u8OUTDOOR_ID     1u    // 室外温度传感器
#define Sensor_u8EVAPORATOR_ID  2u    // 蒸发器温度传感器
#define Sensor_u8WATER_ID       3u    // 水温传感器
#define Sensor_u8HUMIDITY_ID    4u    // 湿度传感器

// 传感器状态
typedef struct
{
    sint16 s16IndoorTemp;      // 室内温度
    sint16 s16OutdoorTemp;     // 室外温度
    sint16 s16EvaporatorTemp;  // 蒸发器温度
    sint16 s16WaterTemp;       // 水温
    uint8 u8Humidity;          // 湿度
    uint8 u8SensorStatus;      // 传感器状态
} Sensor_tstrStatusType;

static Sensor_tstrStatusType Sensor_strStatus;

// 传感器初始化
extern void Sensor_vidInit(void)
{
    Sensor_strStatus.s16IndoorTemp = 0;
    Sensor_strStatus.s16OutdoorTemp = 0;
    Sensor_strStatus.s16EvaporatorTemp = 0;
    Sensor_strStatus.s16WaterTemp = 0;
    Sensor_strStatus.u8Humidity = 0u;
    Sensor_strStatus.u8SensorStatus = Sensor_STATUS_OK;
    
    // 初始化ADC
    ADC_vidInit();
}

// 读取传感器值
extern sint16 Sensor_s16Read(uint8 u8SensorId)
{
    sint16 s16Value = 0;
    uint16 u16AdcValue;
    
    switch(u8SensorId)
    {
        case Sensor_u8INDOOR_ID:
            u16AdcValue = ADC_u16Read(ADC_CHANNEL_INDOOR);
            s16Value = Sensor_s16ConvertTemp(u16AdcValue);
            Sensor_strStatus.s16IndoorTemp = s16Value;
            break;
            
        case Sensor_u8OUTDOOR_ID:
            u16AdcValue = ADC_u16Read(ADC_CHANNEL_OUTDOOR);
            s16Value = Sensor_s16ConvertTemp(u16AdcValue);
            Sensor_strStatus.s16OutdoorTemp = s16Value;
            break;
            
        case Sensor_u8EVAPORATOR_ID:
            u16AdcValue = ADC_u16Read(ADC_CHANNEL_EVAPORATOR);
            s16Value = Sensor_s16ConvertTemp(u16AdcValue);
            Sensor_strStatus.s16EvaporatorTemp = s16Value;
            break;
            
        case Sensor_u8WATER_ID:
            u16AdcValue = ADC_u16Read(ADC_CHANNEL_WATER);
            s16Value = Sensor_s16ConvertTemp(u16AdcValue);
            Sensor_strStatus.s16WaterTemp = s16Value;
            break;
            
        case Sensor_u8HUMIDITY_ID:
            u16AdcValue = ADC_u16Read(ADC_CHANNEL_HUMIDITY);
            s16Value = (sint16)Sensor_u8ConvertHumidity(u16AdcValue);
            Sensor_strStatus.u8Humidity = (uint8)s16Value;
            break;
            
        default:
            break;
    }
    
    return s16Value;
}

// 温度转换函数
static sint16 Sensor_s16ConvertTemp(uint16 u16AdcValue)
{
    sint16 s16Temp;
    
    // ADC值转换为温度 (假设10位ADC，参考电压5V)
    // 温度传感器特性：0°C对应1V，100°C对应4V
    s16Temp = (sint16)((u16AdcValue * 5000u) / 1024u - 1000u) / 30u;
    
    return s16Temp;
}

// 湿度转换函数
static uint8 Sensor_u8ConvertHumidity(uint16 u16AdcValue)
{
    uint8 u8Humidity;
    
    // ADC值转换为湿度百分比
    u8Humidity = (uint8)((u16AdcValue * 100u) / 1024u);
    
    return u8Humidity;
}
```

---

## 诊断模块

### 模块概述
诊断模块负责系统故障检测、故障码管理和诊断通信。

### 1. AppDtc 故障码管理模块

#### 文件位置
- `Sources/APP/AppDtc.c`
- `Sources/APP/AppDtc.h`

#### 功能描述
故障码管理模块，负责故障码的设置、清除和存储。

#### 核心代码

```c
/******************************************************************************/
/* 故障码管理模块 */
/******************************************************************************/

// 故障码定义
#define DTC_SENSOR_ERROR        0x1001u
#define DTC_ACTUATOR_ERROR      0x1002u
#define DTC_COMMUNICATION_ERROR 0x1003u
#define DTC_SYSTEM_ERROR        0x1004u

// 故障码状态
typedef struct
{
    uint16 u16DtcCode;         // 故障码
    uint8 u8Status;            // 故障状态
    uint16 u16OccurrenceCount; // 发生次数
    uint32 u32FirstOccurrence; // 首次发生时间
    uint32 u32LastOccurrence;  // 最后发生时间
} AppDtc_tstrDtcType;

static AppDtc_tstrDtcType AppDtc_astrDtcList[AppDtc_u8MAX_DTC_COUNT];

// 故障码初始化
extern void AppDtc_vidInit(void)
{
    uint8 u8Index;
    
    for(u8Index = 0u; u8Index < AppDtc_u8MAX_DTC_COUNT; u8Index++)
    {
        AppDtc_astrDtcList[u8Index].u16DtcCode = 0u;
        AppDtc_astrDtcList[u8Index].u8Status = AppDtc_STATUS_INACTIVE;
        AppDtc_astrDtcList[u8Index].u16OccurrenceCount = 0u;
        AppDtc_astrDtcList[u8Index].u32FirstOccurrence = 0u;
        AppDtc_astrDtcList[u8Index].u32LastOccurrence = 0u;
    }
}

// 设置故障码
extern void AppDtc_vidSetDtc(uint16 u16DtcCode)
{
    uint8 u8Index;
    uint8 u8Found = FALSE;
    
    // 查找是否已存在该故障码
    for(u8Index = 0u; u8Index < AppDtc_u8MAX_DTC_COUNT; u8Index++)
    {
        if(AppDtc_astrDtcList[u8Index].u16DtcCode == u16DtcCode)
        {
            u8Found = TRUE;
            break;
        }
    }
    
    if(u8Found == FALSE)
    {
        // 查找空闲位置
        for(u8Index = 0u; u8Index < AppDtc_u8MAX_DTC_COUNT; u8Index++)
        {
            if(AppDtc_astrDtcList[u8Index].u16DtcCode == 0u)
            {
                break;
            }
        }
    }
    
    if(u8Index < AppDtc_u8MAX_DTC_COUNT)
    {
        // 设置故障码
        AppDtc_astrDtcList[u8Index].u16DtcCode = u16DtcCode;
        AppDtc_astrDtcList[u8Index].u8Status = AppDtc_STATUS_ACTIVE;
        AppDtc_astrDtcList[u8Index].u16OccurrenceCount++;
        AppDtc_astrDtcList[u8Index].u32LastOccurrence = AppDtc_u32GetCurrentTime();
        
        if(AppDtc_astrDtcList[u8Index].u32FirstOccurrence == 0u)
        {
            AppDtc_astrDtcList[u8Index].u32FirstOccurrence = AppDtc_u32GetCurrentTime();
        }
        
        // 保存到EEPROM
        AppDtc_vidSaveToEeprom();
    }
}

// 清除故障码
extern void AppDtc_vidClearDtc(uint16 u16DtcCode)
{
    uint8 u8Index;
    
    for(u8Index = 0u; u8Index < AppDtc_u8MAX_DTC_COUNT; u8Index++)
    {
        if(AppDtc_astrDtcList[u8Index].u16DtcCode == u16DtcCode)
        {
            AppDtc_astrDtcList[u8Index].u8Status = AppDtc_STATUS_INACTIVE;
            break;
        }
    }
}

// 获取故障码状态
extern uint8 AppDtc_u8GetDtcStatus(uint16 u16DtcCode)
{
    uint8 u8Index;
    uint8 u8Status = AppDtc_STATUS_INACTIVE;
    
    for(u8Index = 0u; u8Index < AppDtc_u8MAX_DTC_COUNT; u8Index++)
    {
        if(AppDtc_astrDtcList[u8Index].u16DtcCode == u16DtcCode)
        {
            u8Status = AppDtc_astrDtcList[u8Index].u8Status;
            break;
        }
    }
    
    return u8Status;
}
```

---

## 配置模块

### 模块概述
配置模块负责系统参数的配置和管理，包括查找表、校准参数等。

### 1. Cmf_Cfg 舒适性配置模块

#### 文件位置
- `Sources/Comfort/Cmf_Cfg.c`
- `Sources/Comfort/Cmf_Cfg.h`

#### 功能描述
舒适性配置模块，提供舒适性控制相关的配置参数。

#### 核心代码

```c
/******************************************************************************/
/* 舒适性配置模块 */
/******************************************************************************/

// 舒适性配置参数
typedef struct
{
    sint16 s16TairDefault;     // 默认空气温度
    sint16 s16TevaDefault;     // 默认蒸发器温度
    sint16 s16TwatDefault;     // 默认水温
    uint8 u8BlowerDefault;     // 默认鼓风机档位
    uint8 u8ModeDefault;       // 默认模式
} CmfCfg_tstrConfigType;

static const CmfCfg_tstrConfigType CmfCfg_strDefaultConfig = {
    2200,   // 22.0°C
    500,    // 5.0°C
    800,    // 80.0°C
    5u,     // 5级
    50u     // 50%
};

// 获取默认空气温度
extern sint16 CmfCfg_s16GetTairDefault(void)
{
    return CmfCfg_strDefaultConfig.s16TairDefault;
}

// 获取默认蒸发器温度
extern sint16 CmfCfg_s16GetTevaDefault(void)
{
    return CmfCfg_strDefaultConfig.s16TevaDefault;
}

// 获取默认水温
extern sint16 CmfCfg_s16GetTwatDefault(void)
{
    return CmfCfg_strDefaultConfig.s16TwatDefault;
}

// 获取默认鼓风机档位
extern uint8 CmfCfg_u8GetBlowerDefault(void)
{
    return CmfCfg_strDefaultConfig.u8BlowerDefault;
}

// 获取默认模式
extern uint8 CmfCfg_u8GetModeDefault(void)
{
    return CmfCfg_strDefaultConfig.u8ModeDefault;
}
```

### 2. Cmf_Tab 查找表模块

#### 文件位置
- `Sources/Comfort/Cmf_Tab.c`
- `Sources/Comfort/Cmf_Tab.h`

#### 功能描述
查找表模块，提供各种控制参数的查找表数据。

#### 核心代码

```c
/******************************************************************************/
/* 查找表模块 */
/******************************************************************************/

// 温度设定查找表
static const uint8 CmfTab_au8TsetTable[64] = {
    16, 16, 17, 17, 18, 18, 19, 19,  // 0-7
    20, 20, 21, 21, 22, 22, 23, 23,  // 8-15
    24, 24, 25, 25, 26, 26, 27, 27,  // 16-23
    28, 28, 29, 29, 30, 30, 31, 31,  // 24-31
    32, 32, 32, 32, 32, 32, 32, 32,  // 32-39
    32, 32, 32, 32, 32, 32, 32, 32,  // 40-47
    32, 32, 32, 32, 32, 32, 32, 32,  // 48-55
    32, 32, 32, 32, 32, 32, 32, 32   // 56-63
};

// 鼓风机电压查找表
static const uint8 CmfTab_au8BlowerVoltageTable[18] = {
    0,   12,  24,  36,  48,  60,  72,  84,  96,  108,  // 0-9级
    120, 122, 124, 126, 128, 130, 132, 134              // 10-17级
};

// 混合风门位置查找表
static const uint8 CmfTab_au8MixPositionTable[101] = {
    0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10,  // 0-10%
    11, 12, 13, 14, 15, 16, 17, 18, 19, 20,  // 11-20%
    // ... (省略中间数据)
    90, 91, 92, 93, 94, 95, 96, 97, 98, 99, 100  // 91-100%
};

// 获取温度设定值
extern uint8 CmfTab_u8GetTset(uint8 u8Index)
{
    if(u8Index < 64u)
    {
        return CmfTab_au8TsetTable[u8Index];
    }
    return 22u;  // 默认22°C
}

// 获取鼓风机电压
extern uint8 CmfTab_u8GetBlowerVoltage(uint8 u8Level)
{
    if(u8Level < 18u)
    {
        return CmfTab_au8BlowerVoltageTable[u8Level];
    }
    return 0u;
}

// 获取混合风门位置
extern uint8 CmfTab_u8GetMixPosition(uint8 u8Percentage)
{
    if(u8Percentage <= 100u)
    {
        return CmfTab_au8MixPositionTable[u8Percentage];
    }
    return 50u;  // 默认50%
}
```

---

## 总结

### 模块化设计特点

1. **分层架构**: 清晰的软件分层，便于维护和扩展
2. **模块独立**: 各模块功能独立，接口清晰
3. **配置化**: 大量参数通过配置文件管理
4. **诊断完善**: 完整的故障诊断和记录功能
5. **实时性**: 基于调度器的实时任务管理
6. **可扩展**: 模块化设计便于功能扩展

### 关键技术

1. **CAN/LIN通信**: 支持多种总线协议
2. **PWM控制**: 精确的鼓风机和PTC控制
3. **步进电机控制**: 精确的风门位置控制
4. **PID控制**: 温度、压力等参数的闭环控制
5. **故障诊断**: 完整的故障检测和记录
6. **参数校准**: 支持传感器校准和参数调整

这个法雷奥空调控制器代码体现了现代汽车电子系统的设计理念，具有高度的模块化、可维护性和可靠性。