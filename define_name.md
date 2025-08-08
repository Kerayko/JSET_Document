# 法雷奥空调控制器变量命名规范

## 目录
1. [命名规范概述](#命名规范概述)
2. [数据类型定义](#数据类型定义)
3. [应用层变量](#应用层变量)
4. [空调控制变量](#空调控制变量)
5. [舒适性控制变量](#舒适性控制变量)
6. [通信变量](#通信变量)
7. [硬件抽象层变量](#硬件抽象层变量)
8. [诊断变量](#诊断变量)
9. [配置变量](#配置变量)

---

## 命名规范概述

### 命名规则
- **前缀**: 模块缩写 (如 `App_`, `Ac_`, `Cmf_`)
- **类型**: 数据类型缩写 (如 `u8`, `s16`, `u32`)
- **功能**: 功能描述 (如 `Tset`, `Blw`, `Mode`)
- **后缀**: 位置或状态 (如 `FL`, `FR`, `Rr`, `Status`)

### 数据类型缩写
| 缩写 | 数据类型 | 说明 |
|------|----------|------|
| u8   | uint8    | 8位无符号整数 |
| s8   | sint8    | 8位有符号整数 |
| u16  | uint16   | 16位无符号整数 |
| s16  | sint16   | 16位有符号整数 |
| u32  | uint32   | 32位无符号整数 |
| s32  | sint32   | 32位有符号整数 |
| bf   | bitfield | 位域 |

---

## 数据类型定义

### 基础数据类型
| 类型名 | 定义 | 范围 | 说明 |
|--------|------|------|------|
| uint8 | unsigned char | 0-255 | 8位无符号整数 |
| sint8 | signed char | -128-127 | 8位有符号整数 |
| uint16 | unsigned short | 0-65535 | 16位无符号整数 |
| sint16 | signed short | -32768-32767 | 16位有符号整数 |
| uint32 | unsigned long | 0-4294967295 | 32位无符号整数 |
| sint32 | signed long | -2147483648-2147483647 | 32位有符号整数 |

### 温度数据类型
| 变量名 | 数据类型 | 单位 | 范围 | 说明 |
|--------|----------|------|------|------|
| s16Temp | sint16 | 0.1°C | -400-800 | 温度值 (实际值×10) |
| s16Tair | sint16 | 0.1°C | -400-800 | 空气温度 |
| s16Tout | sint16 | 0.1°C | -400-800 | 室外温度 |
| s16Tinc | sint16 | 0.1°C | -400-800 | 室内温度 |
| s16Teva | sint16 | 0.1°C | -400-800 | 蒸发器温度 |
| s16Twat | sint16 | 0.1°C | -400-800 | 水温 |

---

## 应用层变量

### 空调设置变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| App_uniAcSetCur | union | - | 当前空调设置 |
| App_uniAcSetRearCur | union | - | 当前后排空调设置 |
| App_strAcOut | struct | - | 空调输出状态 |
| bfTsetFL | uint8:6 | 0-63 | 前左温度设定 (16-32°C) |
| bfTsetFR | uint8:6 | 0-63 | 前右温度设定 (16-32°C) |
| bfRearTset | uint8:6 | 0-63 | 后排温度设定 (16-32°C) |
| bfAc | uint8:1 | 0-1 | AC开关状态 |
| bfHvacMode | uint8:3 | 0-7 | HVAC模式 |
| bfBlw | uint8:4 | 0-15 | 鼓风机档位 |
| bfRec | uint8:2 | 0-3 | 循环模式 |

### 系统状态变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| App_enuAppWorkMode | enum | 0-2 | 应用工作模式 |
| App_enuAppWorkModeLast | enum | 0-2 | 上次应用工作模式 |
| App_u16Vspd | uint16 | 0-65535 | 车速 |
| App_u16SunL | uint16 | 0-65535 | 左阳光强度 |
| App_u16SunR | uint16 | 0-65535 | 右阳光强度 |
| App_s16Tinc | sint16 | -400-800 | 室内温度 |
| App_s16ToutRV | sint16 | -400-800 | 右前室外温度 |
| App_s16ToutRF | sint16 | -400-800 | 右后室外温度 |
| App_s16TwatE | sint16 | -400-800 | 发动机水温 |

### 输出控制变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| u8PosRec | uint8 | 0-100 | 循环风门位置 (%) |
| u8PosMode_F | uint8 | 0-100 | 前排模式风门位置 (%) |
| u8PosMode_R | uint8 | 0-100 | 后排模式风门位置 (%) |
| u8PosMix_FL | uint8 | 0-100 | 前左混合风门位置 (%) |
| u8PosMix_FR | uint8 | 0-100 | 前右混合风门位置 (%) |
| u8VolBlw_F | uint8 | 0-134 | 前排鼓风机电压 (V) |
| u8VolBlw_R | uint8 | 0-134 | 后排鼓风机电压 (V) |
| u8APTC | uint8 | 0-100 | 空气PTC功率 (%) |
| u8WPTC | uint8 | 0-100 | 水暖PTC功率 (%) |
| u8PumpA | uint8 | 0-100 | 水泵A功率 (%) |
| u83WValveA | uint8 | 0-100 | 三通阀A位置 (%) |
| u8OnOffValveA | uint8 | 0-100 | 开关阀A状态 (%) |
| u16CprSpd | uint16 | 0-65535 | 压缩机转速 (rpm) |

---

## 空调控制变量

### 空调系统状态变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| Ac_u8AcSysStat | uint8 | 0-3 | 空调系统状态 |
| Ac_u16WPtcPwr | uint16 | 0-65535 | 水暖PTC功率 |
| Ac_u16APtcPwr | uint16 | 0-65535 | 空气PTC功率 |
| Ac_u8ValvePosition | uint8 | 0-100 | 电子膨胀阀位置 (%) |
| Ac_u8ControlMode | uint8 | 0-255 | 控制模式 |

### EDC控制变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| AcEDC_strControl | struct | - | EDC控制结构 |
| u8ValvePosition | uint8 | 0-100 | 阀门开度 (%) |
| s16SuperheatTgt | sint16 | -400-800 | 目标过热度 |
| s16SuperheatAct | sint16 | -400-800 | 实际过热度 |

### WPTC控制变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| AcWPTC_strControl | struct | - | WPTC控制结构 |
| u16PowerLevel | uint16 | 0-100 | 功率等级 (%) |
| s16TwatTgt | sint16 | -400-800 | 目标水温 |
| s16TwatAct | sint16 | -400-800 | 实际水温 |

### APTC控制变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| AcAPTC_strControl | struct | - | APTC控制结构 |
| u16PowerLevel | uint16 | 0-100 | 功率等级 (%) |
| s16TairTgt | sint16 | -400-800 | 目标空气温度 |
| s16TairAct | sint16 | -400-800 | 实际空气温度 |

---

## 舒适性控制变量

### 舒适性主控制变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| Cmf_strControl | struct | - | 舒适性控制结构 |
| s16TairTgt | sint16 | -400-800 | 目标空气温度 |
| s16TevaTgt | sint16 | -400-800 | 目标蒸发器温度 |
| s16TwatTgt | sint16 | -400-800 | 目标水温 |
| u8BlowerLevel | uint8 | 0-17 | 鼓风机档位 |
| u8ModePosition | uint8 | 0-100 | 模式风门位置 (%) |
| u8MixPosition | uint8 | 0-100 | 混合风门位置 (%) |

### 鼓风机控制变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| CmfBlw_strControl | struct | - | 鼓风机控制结构 |
| u8FrontLevel | uint8 | 0-17 | 前排鼓风机档位 |
| u8RearLevel | uint8 | 0-17 | 后排鼓风机档位 |
| u8FrontVoltage | uint8 | 0-134 | 前排鼓风机电压 (V) |
| u8RearVoltage | uint8 | 0-134 | 后排鼓风机电压 (V) |
| u8ControlMode | uint8 | 0-255 | 控制模式 |

### 混合风门控制变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| CmfMix_strControl | struct | - | 混合风门控制结构 |
| u8FrontLeftPos | uint8 | 0-100 | 前左混合风门位置 (%) |
| u8FrontRightPos | uint8 | 0-100 | 前右混合风门位置 (%) |
| u8RearLeftPos | uint8 | 0-100 | 后左混合风门位置 (%) |
| u8RearRightPos | uint8 | 0-100 | 后右混合风门位置 (%) |
| s16TsetFL | sint16 | -400-800 | 前左温度设定 |
| s16TsetFR | sint16 | -400-800 | 前右温度设定 |
| s16TsetRL | sint16 | -400-800 | 后左温度设定 |
| s16TsetRR | sint16 | -400-800 | 后右温度设定 |

### 模式风门控制变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| CmfMode_strControl | struct | - | 模式风门控制结构 |
| u8FrontMode | uint8 | 0-100 | 前排模式风门位置 (%) |
| u8RearMode | uint8 | 0-100 | 后排模式风门位置 (%) |
| u8ModeType | uint8 | 0-3 | 模式类型 |

---

## 通信变量

### CAN通信变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| u32CanId | uint32 | 0-0x1FFFFFFF | CAN消息ID |
| pu8Data | uint8* | - | CAN数据指针 |
| u8Length | uint8 | 0-8 | CAN数据长度 |
| u8TsetFL | uint8 | 0-63 | 前左温度设定 |
| u8TsetFR | uint8 | 0-63 | 前右温度设定 |
| u8AcSwitch | uint8 | 0-1 | AC开关状态 |
| u8HvacMode | uint8 | 0-7 | HVAC模式 |

### LIN通信变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| u8NodeId | uint8 | 0-63 | LIN节点ID |
| u8SensorValue | uint8 | 0-255 | 传感器值 |
| u8SensorStatus | uint8 | 0-255 | 传感器状态 |

---

## 硬件抽象层变量

### 鼓风机硬件变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| Blower_strStatus | struct | - | 鼓风机状态结构 |
| u8FrontVoltage | uint8 | 0-134 | 前排鼓风机电压 (V) |
| u8RearVoltage | uint8 | 0-134 | 后排鼓风机电压 (V) |
| u8FrontStatus | uint8 | 0-255 | 前排鼓风机状态 |
| u8RearStatus | uint8 | 0-255 | 后排鼓风机状态 |
| u16FrontSpeed | uint16 | 0-65535 | 前排鼓风机转速 (rpm) |
| u16RearSpeed | uint16 | 0-65535 | 后排鼓风机转速 (rpm) |

### 步进电机变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| Stepper_strStatus | struct | - | 步进电机状态结构 |
| u8MixFLPosition | uint8 | 0-100 | 前左混合风门位置 (%) |
| u8MixFRPosition | uint8 | 0-100 | 前右混合风门位置 (%) |
| u8ModeFPosition | uint8 | 0-100 | 前排模式风门位置 (%) |
| u8ModeRPosition | uint8 | 0-100 | 后排模式风门位置 (%) |
| u8RecPosition | uint8 | 0-100 | 循环风门位置 (%) |
| u8Status | uint8 | 0-255 | 电机状态 |

### 传感器变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| Sensor_strStatus | struct | - | 传感器状态结构 |
| s16IndoorTemp | sint16 | -400-800 | 室内温度 |
| s16OutdoorTemp | sint16 | -400-800 | 室外温度 |
| s16EvaporatorTemp | sint16 | -400-800 | 蒸发器温度 |
| s16WaterTemp | sint16 | -400-800 | 水温 |
| u8Humidity | uint8 | 0-100 | 湿度 (%) |
| u8SensorStatus | uint8 | 0-255 | 传感器状态 |

---

## 诊断变量

### 故障码变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| AppDtc_astrDtcList | struct[] | - | 故障码列表 |
| u16DtcCode | uint16 | 0-65535 | 故障码 |
| u8Status | uint8 | 0-255 | 故障状态 |
| u16OccurrenceCount | uint16 | 0-65535 | 发生次数 |
| u32FirstOccurrence | uint32 | 0-4294967295 | 首次发生时间 |
| u32LastOccurrence | uint32 | 0-4294967295 | 最后发生时间 |

### 诊断状态变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| AppDiag_enuStatus | enum | 0-2 | 诊断状态 |
| AppDiag_u16ErrorCounter | uint16 | 0-65535 | 错误计数器 |
| AppDiag_u16WarningCounter | uint16 | 0-65535 | 警告计数器 |

---

## 配置变量

### 舒适性配置变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| CmfCfg_strDefaultConfig | struct | - | 默认配置结构 |
| s16TairDefault | sint16 | -400-800 | 默认空气温度 |
| s16TevaDefault | sint16 | -400-800 | 默认蒸发器温度 |
| s16TwatDefault | sint16 | -400-800 | 默认水温 |
| u8BlowerDefault | uint8 | 0-17 | 默认鼓风机档位 |
| u8ModeDefault | uint8 | 0-100 | 默认模式 |

### 查找表变量
| 变量名 | 数据类型 | 范围 | 说明 |
|--------|----------|------|------|
| CmfTab_au8TsetTable | uint8[] | 0-63 | 温度设定查找表 |
| CmfTab_au8BlowerVoltageTable | uint8[] | 0-17 | 鼓风机电压查找表 |
| CmfTab_au8MixPositionTable | uint8[] | 0-100 | 混合风门位置查找表 |

---

## 枚举类型定义

### HVAC模式枚举
| 枚举值 | 数值 | 说明 |
|--------|------|------|
| App_HVAC_OFF | 0 | 关闭 |
| App_HVAC_DEFROST | 1 | 除霜 |
| App_HVAC_MANUAL | 2 | 手动 |
| App_HVAC_AUTO | 3 | 自动 |

### 后排HVAC模式枚举
| 枚举值 | 数值 | 说明 |
|--------|------|------|
| App_RHVAC_OFF | 0 | 关闭 |
| App_RHVAC_MANUAL | 1 | 手动 |
| App_RHVAC_AUTO | 2 | 自动 |

### 应用工作模式枚举
| 枚举值 | 数值 | 说明 |
|--------|------|------|
| App_APPWORKMODE_UNKNOWN | 0 | 未知 |
| App_APPWORKMODE_OFF | 1 | 关闭 |
| App_APPWORKMODE_ON | 2 | 开启 |

### 空调系统状态枚举
| 枚举值 | 数值 | 说明 |
|--------|------|------|
| Ac_u8ACSYSSTAT_IDLE | 0 | 空闲 |
| Ac_u8ACSYSSTAT_HEAT | 1 | 加热 |
| Ac_u8ACSYSSTAT_COOL | 2 | 制冷 |
| Ac_u8ACSYSSTAT_HEATCOOL | 3 | 加热制冷 |

---

## 常量定义

### 系统常量
| 常量名 | 数值 | 说明 |
|--------|------|------|
| App_u8HMIINPUT_BUFLEN | 14u | HMI输入缓冲区长度 |
| App_u8ACSET_BUFLEN | 4u | 空调设置缓冲区长度 |
| App_u8ACSETRR_BUFLEN | 2u | 后排空调设置缓冲区长度 |
| App_u8NUM_ACSTATUS | 39u | AC状态数量 |
| AppDtc_u8MAX_DTC_COUNT | 20u | 最大故障码数量 |

### 温度相关常量
| 常量名 | 数值 | 说明 |
|--------|------|------|
| TEMP_OFFSET | 160u | 温度偏移量 |
| TEMP_SCALE | 5u | 温度缩放因子 |
| TEMP_MIN | 160u | 最小温度 (16.0°C) |
| TEMP_MAX | 320u | 最大温度 (32.0°C) |

### 电压相关常量
| 常量名 | 数值 | 说明 |
|--------|------|------|
| VOLTAGE_MAX_FRONT | 134u | 前排鼓风机最大电压 (V) |
| VOLTAGE_MAX_REAR | 134u | 后排鼓风机最大电压 (V) |
| VOLTAGE_LINEAR_RANGE | 120u | 线性电压范围 (V) |

### 位置相关常量
| 常量名 | 数值 | 说明 |
|--------|------|------|
| POSITION_MIN | 0u | 最小位置 (%) |
| POSITION_MAX | 100u | 最大位置 (%) |
| POSITION_DEFAULT | 50u | 默认位置 (%) |

### 功率相关常量
| 常量名 | 数值 | 说明 |
|--------|------|------|
| POWER_MIN | 0u | 最小功率 (%) |
| POWER_MAX | 100u | 最大功率 (%) |
| POWER_DEFAULT | 0u | 默认功率 (%) |

---

## 位域定义

### 空调设置位域
| 位域名 | 位数 | 位置 | 说明 |
|--------|------|------|------|
| bfTsetFL | 6 | 0-5 | 前左温度设定 |
| bfTsetFR | 6 | 6-11 | 前右温度设定 |
| bfAc | 1 | 12 | AC开关 |
| bfHvacMode | 3 | 13-15 | HVAC模式 |
| bfBlw | 4 | 16-19 | 鼓风机档位 |
| bfRec | 2 | 20-21 | 循环模式 |

### 后排空调设置位域
| 位域名 | 位数 | 位置 | 说明 |
|--------|------|------|------|
| bfRearTset | 6 | 0-5 | 后排温度设定 |
| bfRearHvacMode | 2 | 6-7 | 后排HVAC模式 |
| bfRearMode | 2 | 8-9 | 后排模式 |
| bfRearBlw | 4 | 10-13 | 后排鼓风机档位 |

### HMI输入位域
| 位域名 | 位数 | 位置 | 说明 |
|--------|------|------|------|
| bfCCP_FLTempSwitchReq_Cur | 6 | 0-5 | 前左温度开关请求(当前) |
| bfCCP_OFFSwitchReq | 2 | 6-7 | 关闭开关请求 |
| bfCCP_FRTempSwitchReq_Cur | 6 | 8-13 | 前右温度开关请求(当前) |
| bfCCP_ACSwitchReq | 2 | 14-15 | AC开关请求 |
| bfCCP_RearTempSwitchReq_Cur | 6 | 16-21 | 后排温度开关请求(当前) |
| bfCCP_RearAutoACSwitchReq | 2 | 22-23 | 后排自动AC开关请求 |

---

## 函数命名规范

### 初始化函数
| 函数名 | 模块 | 说明 |
|--------|------|------|
| AppMain_vidInit | APP | 应用层主初始化 |
| AppCalc_vidInit | APP | 计算模块初始化 |
| AppCom_vidInit | APP | 通信模块初始化 |
| AppDiag_vidInit | APP | 诊断模块初始化 |
| Ac_vidInit | AcCtrl | 空调控制初始化 |
| Cmf_vidInit | Comfort | 舒适性控制初始化 |
| Blower_vidInit | ECUAL | 鼓风机初始化 |
| Stepper_vidInit | ECUAL | 步进电机初始化 |
| Sensor_vidInit | ECUAL | 传感器初始化 |

### 管理函数
| 函数名 | 模块 | 说明 |
|--------|------|------|
| AppMain_vidManage | APP | 应用层主管理 |
| AppMain_vidFastManage | APP | 应用层快速管理 |
| AppMain_vidSlowManage | APP | 应用层慢速管理 |
| Ac_vidSlowManage | AcCtrl | 空调慢速管理 |
| Ac_vidFastManage | AcCtrl | 空调快速管理 |
| Cmf_vidManage | Comfort | 舒适性管理 |

### 控制函数
| 函数名 | 模块 | 说明 |
|--------|------|------|
| Ac_vidHeatControl | AcCtrl | 制热控制 |
| Ac_vidCoolControl | AcCtrl | 制冷控制 |
| CmfBlw_vidControl | Comfort | 鼓风机控制 |
| CmfMix_vidControl | Comfort | 混合风门控制 |
| CmfMode_vidControl | Comfort | 模式风门控制 |
| Stepper_vidSetMixFL | ECUAL | 设置前左混合风门 |
| Blower_vidSetFrontVoltage | ECUAL | 设置前排鼓风机电压 |

### 获取函数
| 函数名 | 模块 | 说明 |
|--------|------|------|
| App_u16GetTsetFL | APP | 获取前左温度设定 |
| App_u16GetTsetFR | APP | 获取前右温度设定 |
| App_u8GetBlwFr | APP | 获取前排鼓风机档位 |
| App_u8GetModeFr | APP | 获取前排模式 |
| Cmf_s16GetTairTgt | Comfort | 获取目标空气温度 |
| Cmf_s16GetTevaTgt | Comfort | 获取目标蒸发器温度 |
| Sensor_s16Read | ECUAL | 读取传感器值 |

### 设置函数
| 函数名 | 模块 | 说明 |
|--------|------|------|
| App_vidSetAcSettings | APP | 设置空调参数 |
| Ac_vidSetSystemState | AcCtrl | 设置系统状态 |
| Cmf_vidSetTargetTemp | Comfort | 设置目标温度 |
| Blower_vidSetFrontVoltage | ECUAL | 设置前排鼓风机电压 |
| Stepper_vidSetMixFL | ECUAL | 设置前左混合风门 |

---

## 宏定义规范

### 温度相关宏
| 宏名 | 定义 | 说明 |
|------|------|------|
| App_u16GetTsetFL() | (((uint16)App_uniAcSetCur.strAcSet.bfTsetFL * 5u) + 160u) | 获取前左温度设定 |
| App_u16GetTsetFR() | (((uint16)App_uniAcSetCur.strAcSet.bfTsetFR * 5u) + 160u) | 获取前右温度设定 |
| App_u16GetTsetRr() | (((uint16)App_uniAcSetRearCur.strAcSetRear.bfRearTset * 5u) + 160u) | 获取后排温度设定 |

### 状态获取宏
| 宏名 | 定义 | 说明 |
|------|------|------|
| App_u8GetModeFr() | (App_strAcOut.u8PosMode_F) | 获取前排模式 |
| App_u8GetModeRr() | (App_strAcOut.u8PosMode_R) | 获取后排模式 |
| App_u8GetBlwFr() | (App_strAcOut.u8VolBlw_F) | 获取前排鼓风机档位 |
| App_u8GetBlwRr() | (App_strAcOut.u8VolBlw_R) | 获取后排鼓风机档位 |

### 系统状态宏
| 宏名 | 定义 | 说明 |
|------|------|------|
| App_enuGetFHVACStatus() | (App_uniAcSetCur.strAcSet.bfHvacMode) | 获取前排HVAC状态 |
| App_enuGetRHVACStatus() | (App_uniAcSetRearCur.strAcSetRear.bfRearHvacMode) | 获取后排HVAC状态 |
| App_u8GetAcKeyStat() | (App_uniAcSetCur.strAcSet.bfAc) | 获取AC按键状态 |

### 传感器数据宏
| 宏名 | 定义 | 说明 |
|------|------|------|
| App_s16GetTwat() | (Sensor_s16Read(Sensor_u8WATER_ID)) | 获取水温 |
| App_s16GetTair() | (App_s16TptcRr) | 获取空气温度 |
| App_s16GetTinc() | (App_s16Tinc) | 获取室内温度 |
| App_s16GetToutRV() | (App_s16ToutRV) | 获取右前室外温度 |

---

## 文件命名规范

### 源文件命名
| 文件类型 | 命名规则 | 示例 |
|----------|----------|------|
| 头文件 | 模块名.h | AppMain.h, Ac.h, Cmf.h |
| 源文件 | 模块名.c | AppMain.c, Ac.c, Cmf.c |
| 配置头文件 | 模块名_Cfg.h | App_Cfg.h, Cmf_Cfg.h |
| 配置源文件 | 模块名_Cfg.c | App_Cfg.c, Cmf_Cfg.c |

### 目录结构命名
| 目录名 | 说明 | 包含文件 |
|--------|------|----------|
| APP/ | 应用层模块 | AppMain.c/h, AppCalc.c/h |
| AcCtrl/ | 空调控制模块 | Ac.c/h, AcEDC.c/h |
| Comfort/ | 舒适性控制模块 | Cmf.c/h, CmfBlw.c/h |
| COM_CAN/ | CAN通信模块 | Can.c/h, CanIf.c/h |
| COM_LIN/ | LIN通信模块 | Lin.c/h, LinIf.c/h |
| ECUAL/ | 硬件抽象层 | Blower.c/h, Stepper.c/h |
| MCAL/ | 微控制器抽象层 | ADC.c/h, DIO.c/h |
| Services/ | 服务层 | CRC8.c/h, Eeprom.c/h |

---

## 注释规范

### 文件头注释
```c
/******************************************************************************/
/* PROJECT  :  M01                                                            */
/******************************************************************************/
/* !Layer           : App                                                     */
/* !Component       : AppMain                                                 */
/* !Description     : App main function                                       */
/* !Target          : uPD70F3375                                              */
/* !Vendor          : sb (aaa sb)                                            */
/* Coding language  : C                                                       */
/* COPYRIGHT 2017 aaa                                                         */
/* All Rights Reserved                                                        */
/******************************************************************************/
```

### 函数注释
```c
/******************************************************************************/
/* 函数名: AppMain_vidInit                                                    */
/* 功能: 应用层主初始化                                                       */
/* 参数: 无                                                                   */
/* 返回值: 无                                                                 */
/* 说明: 初始化应用层各模块                                                   */
/******************************************************************************/
```

### 变量注释
```c
/* 空调系统状态 */
static uint8 Ac_u8AcSysStat = Ac_u8ACSYSSTAT_IDLE;

/* 前排鼓风机电压 (0-134V) */
extern uint8 App_strAcOut.u8VolBlw_F;

/* 前左温度设定 (16-32°C) */
extern uint8 App_uniAcSetCur.strAcSet.bfTsetFL;
```

---

## 总结

### 命名规范特点

1. **模块化前缀**: 每个模块都有明确的前缀标识
2. **类型标识**: 数据类型在变量名中明确体现
3. **功能描述**: 变量名清晰表达其功能用途
4. **位置标识**: 使用FL/FR/Rr等标识位置
5. **状态标识**: 使用Status/Stat等标识状态

### 数据类型规范

1. **统一类型**: 使用标准化的数据类型定义
2. **位域优化**: 合理使用位域节省内存
3. **范围限制**: 明确的数据范围定义
4. **单位标识**: 在注释中明确单位信息

### 函数命名规范

1. **动作前缀**: vid(动作), u8(获取), s16(计算)
2. **模块标识**: 函数名包含模块前缀
3. **功能描述**: 函数名清晰表达功能
4. **参数类型**: 参数名体现数据类型

这个变量命名规范文档为法雷奥空调控制器代码提供了统一的命名标准，确保代码的可读性、可维护性和一致性。