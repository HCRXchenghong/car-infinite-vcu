# 10 CAN 报文矩阵（v0.2 · 信号级）

> 状态：主干总线全部报文已细化到信号级（v0.2），可作为 DBC 基线评审；达妙桥内按官方协议（ID 分配方案见 §6）；MSSD/嘉佰达待厂家手册。
> 原则：滚动计数器 + CRC8 防错发；超时即安全动作；标准帧 11bit；500K。

## 1. 总线总览

| 总线 | 速率 | 节点 | 收发器 | 状态 |
|---|---|---|---|---|
| 主干总线 | 500K（CAN-FD 硬件，协议先按 2.0 帧） | VCU、前桥、后桥、BCM、BMS | 隔离 CAN 收发器（自研板）/ 嘉佰达原口 | 本文档主体 |
| 前桥内转向 | 1Mbps | 前桥板、达妙 ×2 | TJA1443 + 共模电感/TVS | 达妙官方协议 |
| 后桥内转向 | 1Mbps | 后桥板、达妙 ×2 | 同上 | 达妙官方协议 |
| 前桥内驱动 | 待厂家定 | 前桥板、MSSD ×1 | 同上 | 类 CANopen，待手册 |
| 后桥内驱动 | 待厂家定 | 后桥板、MSSD ×1 | 同上 | 同上 |

## 2. 主干 ID 分配 + UDS 地址

| 段 | 范围 | 用途 |
|---|---|---|
| 系统层 | 0x000~0x0FF | 急停、模式、心跳 |
| VCU 指令 | 0x100~0x1FF | 目标指令、轮速广播、车身控制 |
| 前桥反馈 | 0x200~0x2FF | 状态/传感器/事件 |
| 后桥反馈 | 0x300~0x3FF | 同上 |
| BCM | 0x400~0x4FF | 车身状态 |
| BMS | 0x500~0x5FF | 嘉佰达协议按厂家实际，范围预留 |
| UDS 诊断 | 0x7E0~0x7EF | 见 §7 |
| 预留 | 0x700~0x7DF | 惯导/上位机扩展 |

## 3. 公共约定

- **RollingCounter**（4bit）+ **CRC8**：所有周期报文必带；CRC 算法 DBC 阶段定（候选 AUTOSAR CRC-8 / SAE J1850）
- **无效值**：int16 信号 0x7FFF = 无效；uint16 距离信号 0xFFFF = 无效
- **事件报文**：即时发送 + 100ms 重发 ×3，确保送达
- 字节序：Motorola（大端），DBC 阶段统一

## 4. 主干报文定义（信号级）

### 0x001 ESTOP_STATE（BCM→全体，100ms + 事件即发）
| 信号 | 长度 | 说明 |
|---|---|---|
| EstopActive | 1bit | 1=急停有效，全体执行安全动作 |
| EstopSource | 2bit | 0=无 1=物理按钮 2=远程指令 3=其他 |
| RollingCounter/CRC8 | 4+8bit | |

### 0x002 MODE_STATE（VCU→全体，100ms）
| 信号 | 长度 | 说明 |
|---|---|---|
| Mode | 2bit | 0=自动驾驶 1=远程接管 2=本地遥控 3=维护 |
| DriveEnable | 1bit | 行驶总使能 |
| SpeedLimitLevel | 2bit | 0=8 1=15 2=25 km/h 档位 |
| RC/CRC | | |

### 0x010 HEARTBEAT_NODE（各节点→，1s）
| NodeID | 8bit | 1=VCU 2=前桥 3=后桥 4=BCM 5=BMS |
| NodeFaultSummary | 8bit | 故障摘要位图 |

### 0x100 / 0x101 VCU_CMD_FRONT / VCU_CMD_REAR（VCU→桥，20ms）
| 信号 | 长度 | 因子/偏移 | 范围 |
|---|---|---|---|
| TargetAngleL/R | 16bit×2 | 0.01°，有符号 | ±90° |
| TargetSpeedL/R | 16bit×2 | 0.01 m/s，有符号（支持反转） | ±10 m/s |
| DriveEnable | 1bit | | |
| BrakeReq | 1bit | 制动请求（能量回收+抱闸预告） | |
| RC/CRC | | | |

### 0x200 / 0x300 FRONT/REAR_STATE（桥→VCU，20ms）
| ActualAngleL/R | 16bit×2 | 0.01°（达妙实测） |
| ActualSpeedL/R | 16bit×2 | 0.01 m/s（轮毂编码器实测） |
| FaultBitmap | 8bit | bit0 转向L失联 bit1 转向R失联 bit2 驱动失联 bit3 过温 bit4 过流 bit5 急停反射已触发 bit6-7 预留 |
| RC/CRC | | |

### 0x201 / 0x301 FRONT/REAR_SENSOR_US（桥→VCU，50ms）
| US1~US4 | uint16×4 | mm，0xFFFF=无效（超声波 ×4） |

### 0x203 / 0x303 FRONT/REAR_SENSOR_LAS（桥→VCU，50ms）
| LAS1/LAS2 | uint16×2 | mm，0xFFFF=无效（激光测距 ×2） |
| ValidBits | 6bit | 6 路传感器有效标志 |

### 0x202 / 0x302 FRONT/REAR_EVENT（桥→VCU，事件）
| EventType | 8bit | 1=近距障碍 2=悬崖/落差 3=执行器失联 4=传感器组故障 5=急停反射触发 |
| Severity | 2bit | 0=提示 1=减速 2=停车 |
| Distance | uint16 | 触发时距离 mm |
| SensorID | 4bit | 触发传感器编号 |

### 0x102 VCU_SPEED_SYN（VCU→惯导/上位机，20ms）—— 合成层
| VehicleSpeed | int16 | 0.01 m/s，车体纵向车速（运动学解算） |
| SpeedQuality | 2bit | 0=正常 1=降级 2=无效 |

### 0x103 / 0x104 VCU_WHEEL_RAW（VCU→上位机，20ms）—— 原始层
| 0x103：WheelSpeed FL/FR/RL/RR | int16×4 | 0.01 m/s（实测） |
| 0x104：WheelAngle FL/FR/RL/RR | int16×4 | 0.01°（实测） |

### 0x110 VCU_CMD_BODY（VCU→BCM，100ms）
| LidCmd | 2bit | 0=无 1=开盖 2=关盖 3=停止 |
| LockCmd | 2bit | 0=无 1=解锁 2=上锁 |
| LightBits | 16bit | 16 路灯光位图（对应灯光清单 11 §4.6） |
| FanCmd | 1bit | 静音风扇 |
| PowerSeqStep | 4bit | 远程上电时序步骤号 |

### 0x400 BCM_STATE（BCM→VCU，100ms）
| PowerSeqState | 4bit | 上电时序状态 |
| LidState | 3bit | 0=停止 1=开中 2=已开 3=关中 4=已关 5=故障 |
| LockState | 1bit | 锁反馈（无磁信号） |
| TempC / Humidity | int8 / uint8 | 控制板层温湿度（舱内温度随报） |
| FanState / EstopMirror | 1bit×2 | |

### 0x5xx BMS_*（BMS→VCU，按厂家）
- 嘉佰达协议到手后补：SOC/总压/总流/单节极值/温度/故障码

## 5. 超时与安全动作（报文级）

| 监控方 | 报文 | 超时 | 动作 |
|---|---|---|---|
| 桥端 | 0x100/0x101 | 200ms | 本桥斜坡停车 |
| 桥端 | 0x001 EstopActive | 即时 | 驱动切断 + 抱闸 |
| VCU | 0x200/0x300 | 200ms | 全车限速/停车 + 报故障 |
| VCU | 0x201/0x203/0x301/0x303 | 500ms | 感知降级：限速 |
| VCU | 0x400 | 1000ms | 报警，车身功能降级 |
| VCU | BMS 报文 | 1000ms | 降功率 + 报警 |
| 桥端 | 达妙状态帧 | 100~200ms | 切本桥驱动使能 |
| 桥端 | MSSD 状态 | 100~200ms | 切本桥驱动使能 |

## 6. 桥内总线协议

### 达妙转向（1Mbps）
- 帧格式按达妙官方协议（位置模式 + 力矩限幅），桥板做"0x100/0x101 → 达妙帧"网关转换
- **ID 分配方案**：前后桥为独立网段，段内 ID 可复用；建议每段内转向左 = CAN ID 0x01、转向右 = 0x02，Master ID 按达妙《CANID 与 MasterID 配置建议》配对；装机前用达妙上位机逐个写入
- 失联行为：配置为"保持当前位置"（已在 08 记录）

### MSSD 驱动（波特率待厂家）
- 类 CANopen：SDO 参数配置（建议节点 ID：每桥 1 台 = Node 1，请求 0x600+ID / 响应 0x580+ID）+ 电机状态主动上报（PDO，报文按厂家手册）
- 桥板做"0x100/0x101 → MSSD 帧"网关转换

## 7. 诊断（UDS，ISO 14229）

| 节点 | 物理请求地址 | 响应 |
|---|---|---|
| VCU | 0x7E0 | 0x7E8 |
| 前桥板 | 0x7E1 | 0x7E9 |
| 后桥板 | 0x7E2 | 0x7EA |
| BCM | 0x7E3 | 0x7EB |
| BMS | 0x7E4（按嘉佰达实际） | 0x7EC |
| 功能寻址 | 0x7DF | — |

常用服务：0x10 会话、0x11 复位、0x19 读故障码、0x22/0x2E 读写数据、0x27 安全访问、0x34~0x37 刷写（OTA 走 FOTA A/B 分区）、0x3E 测试在线。

## 8. 总线负载估算（500K）

| 类别 | 计算 | 负载 |
|---|---|---|
| 20ms 周期（7 帧） | 7×50Hz×130bit | ≈45.5 kbit/s |
| 50ms 周期（4 帧） | 4×20Hz×130bit | ≈10.4 kbit/s |
| 100ms 周期（4 帧） | 4×10Hz×130bit | ≈5.2 kbit/s |
| 心跳（4 节点） | 4×1Hz×130bit | ≈0.5 kbit/s |
| **合计**（+事件/诊断余量） | | **≈12~15%，健康**（目标 <30%） |

## 9. DBC 落库规范与下一步

- 文件：`chassis_vcu.dbc`（主干）、`chassis_axle_steering.dbc`（桥内转向模板）
- 命名：报文 `模块_方向_功能`，信号大驼峰；单位/因子/偏移/无效值与本文档一致
1. 评审 v0.2 → 出正式 DBC
2. 达妙：按 §6 方案实写 4 电机 ID，台架验证失联行为
3. MSSD：催协议手册，补桥内驱动报文
4. 嘉佰达：催 CAN 协议，替换 0x5xx 占位
5. CRC 算法定型（AUTOSAR CRC-8 候选）+ 台架故障注入测试（错帧/超时/总线断开）
