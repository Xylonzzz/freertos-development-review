# RTOS 企业级嵌入式软件框架
## 1. 总体架构

把嵌入式系统想成一家公司：BSP 是大楼和水电，Driver 是设备操作员，Service
是公共部门，Task 是员工，Application 是业务规则。

``` text
+--------------------------------------------------+
| Application：业务状态机 / Safety Policy          |
+-------------------------+------------------------+
                          | Event / Queue
+-------------------------v------------------------+
| Services：Logger / Error / Watchdog / Config     |
|           Recovery / Diagnostics / Protocol       |
+-------------------------+------------------------+
                          |
+-------------------------v------------------------+
| Tasks：Sensor / Control / Comm / Monitor / Log   |
+-------------------------+------------------------+
                          |
+-------------------------v------------------------+
| Drivers：Sensor / UART / CAN / Flash / Motor     |
+-------------------------+------------------------+
                          |
+-------------------------v------------------------+
| BSP：GPIO / SPI / I2C / ADC / PWM / IWDG         |
+-------------------------+------------------------+
                          |
                       Hardware
```

推荐目录：

``` text
Project/
├── App/
│   ├── app_main.c
│   ├── app_config.h
│   ├── app_event.h
│   └── app_types.h
├── Service/
│   ├── state_machine/
│   ├── watchdog/
│   ├── logger/
│   ├── parameter/
│   ├── diagnostics/
│   └── error/
├── Tasks/
│   ├── sensor_task.c
│   ├── control_task.c
│   ├── communication_task.c
│   ├── monitor_task.c
│   └── log_task.c
├── Drivers/
├── BSP/
└── Middlewares/FreeRTOS/
```

业务层不要到处直接写 HAL：

``` c
/* 不推荐 */
HAL_GPIO_WritePin(GPIOA, GPIO_PIN_5, GPIO_PIN_SET);

/* 推荐 */
BSP_LED_Set(LED_STATUS, true);
```

这样换 PCB 时，主要修改 BSP，而不是把业务代码翻个底朝天。

## 2. Task 要“一个人干一类事”

不要做超级 Task：

``` c
void BigTask(void *argument)
{
    for (;;)
    {
        ReadSensor();
        ControlMotor();
        SendNetwork();
        SaveFlash();
        PrintLog();
        CheckKey();
    }
}
```

网络一卡，整个 Task 都卡。推荐拆分：

``` c
void SensorTask(void *argument)
{
    for (;;)
    {
        Sensor_Process();
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

void ControlTask(void *argument)
{
    for (;;)
    {
        Control_Process();
        vTaskDelay(pdMS_TO_TICKS(20));
    }
}

void CommTask(void *argument)
{
    for (;;)
    {
        Communication_Process();
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

更进一步：能等待消息的 Task 就阻塞等待，不要每 1ms 醒来问一次“有事吗”。

## 3. Queue：任务之间的快递柜

不要让很多 Task
随意读写同一堆全局变量。发送者把完整“包裹”放进队列，消费者取完整包裹。

``` c
typedef enum
{
    MSG_SENSOR_DATA,
    MSG_BUTTON_EVENT,
    MSG_COMM_RX,
    MSG_FAULT
} AppMsgId_t;

typedef struct
{
    AppMsgId_t id;
    union
    {
        struct {
            float temperature;
            float voltage;
        } sensor;

        struct {
            uint8_t key_id;
            uint8_t action;
        } key;

        uint32_t error_code;
    } data;
} AppMessage_t;
```

发送：

``` c
AppMessage_t msg = {0};
msg.id = MSG_SENSOR_DATA;
msg.data.sensor.temperature = 36.5f;

if (xQueueSend(g_app_queue, &msg, pdMS_TO_TICKS(10)) != pdPASS)
{
    Error_Report(ERR_QUEUE_FULL);
}
```

接收：

``` c
AppMessage_t msg;

for (;;)
{
    if (xQueueReceive(g_app_queue, &msg, portMAX_DELAY) == pdPASS)
    {
        switch (msg.id)
        {
            case MSG_SENSOR_DATA:
                HandleSensorData(&msg.data.sensor);
                break;
            case MSG_BUTTON_EVENT:
                HandleButton(&msg.data.key);
                break;
            case MSG_FAULT:
                HandleFault(msg.data.error_code);
                break;
            default:
                break;
        }
    }
}
```

## 4. 状态机：告诉系统“我现在是谁”

大量嵌套 `if`
很快会失控。状态机把“当前状态、允许事件、下一状态”明确写出来。

``` c
typedef enum
{
    SYS_STATE_INIT = 0,
    SYS_STATE_IDLE,
    SYS_STATE_RUNNING,
    SYS_STATE_ERROR,
    SYS_STATE_RECOVERY,
    SYS_STATE_MAX
} SystemState_t;

typedef enum
{
    EVENT_NONE = 0,
    EVENT_INIT_OK,
    EVENT_START,
    EVENT_STOP,
    EVENT_FAULT,
    EVENT_RESET
} SystemEvent_t;

typedef struct
{
    SystemState_t current;
    SystemState_t previous;
    TickType_t enter_tick;
} StateMachine_t;
```

状态图：

``` text
INIT --初始化成功--> IDLE --START--> RUNNING
 |                                  |
 FAULT                              FAULT
 |                                  |
 +---------------> ERROR <----------+
                      |
                    RESET
                      |
                      v
                  RECOVERY
                      |
                    成功
                      |
                      v
                    IDLE
```

初始化与切换：

``` c
void StateMachine_Init(StateMachine_t *sm)
{
    if (sm == NULL) return;

    sm->current = SYS_STATE_INIT;
    sm->previous = SYS_STATE_INIT;
    sm->enter_tick = xTaskGetTickCount();
}

static void ChangeState(StateMachine_t *sm, SystemState_t next)
{
    if (sm == NULL) return;

    if (next >= SYS_STATE_MAX)
        next = SYS_STATE_ERROR;

    sm->previous = sm->current;
    sm->current = next;
    sm->enter_tick = xTaskGetTickCount();
}
```

事件分发：

``` c
void StateMachine_Dispatch(StateMachine_t *sm, SystemEvent_t event)
{
    switch (sm->current)
    {
        case SYS_STATE_INIT:
            if (event == EVENT_INIT_OK) ChangeState(sm, SYS_STATE_IDLE);
            else if (event == EVENT_FAULT) ChangeState(sm, SYS_STATE_ERROR);
            break;

        case SYS_STATE_IDLE:
            if (event == EVENT_START) ChangeState(sm, SYS_STATE_RUNNING);
            break;

        case SYS_STATE_RUNNING:
            if (event == EVENT_STOP) ChangeState(sm, SYS_STATE_IDLE);
            else if (event == EVENT_FAULT) ChangeState(sm, SYS_STATE_ERROR);
            break;

        case SYS_STATE_ERROR:
            if (event == EVENT_RESET) ChangeState(sm, SYS_STATE_RECOVERY);
            break;

        case SYS_STATE_RECOVERY:
            if (event == EVENT_INIT_OK) ChangeState(sm, SYS_STATE_IDLE);
            break;

        default:
            ChangeState(sm, SYS_STATE_ERROR);
            break;
    }
}
```

`default -> ERROR` 是很便宜的防御：状态值异常时不要继续执行危险业务。

## 5. Enter / Execute / Exit 状态模式

状态不只是一个数字。进入、停留、离开时通常要做不同事情。

``` c
typedef struct
{
    void (*enter)(void);
    void (*execute)(void);
    void (*exit)(void);
} StateOps_t;
```

例如 ERROR：

``` c
static void Error_Enter(void)
{
    Motor_Stop();
    Heater_Off();
    Alarm_On();
    LOG_ERROR("enter ERROR");
}

static void Error_Execute(void)
{
    CheckRecoveryCondition();
}

static void Error_Exit(void)
{
    Alarm_Off();
}
```

状态表：

``` c
static const StateOps_t g_state_table[SYS_STATE_MAX] =
{
    [SYS_STATE_INIT]    = {Init_Enter, Init_Execute, Init_Exit},
    [SYS_STATE_IDLE]    = {Idle_Enter, Idle_Execute, Idle_Exit},
    [SYS_STATE_RUNNING] = {Run_Enter, Run_Execute, Run_Exit},
    [SYS_STATE_ERROR]   = {Error_Enter, Error_Execute, Error_Exit},
    [SYS_STATE_RECOVERY]= {Recovery_Enter, Recovery_Execute, Recovery_Exit},
};
```

## 6. 状态一定要考虑超时

连接服务器不能永远停在 CONNECTING。

``` c
static bool TimeExpired(TickType_t start, TickType_t timeout)
{
    return (TickType_t)(xTaskGetTickCount() - start) >= timeout;
}
```

使用：

``` c
case COMM_CONNECTING:
    if (Comm_IsConnected())
    {
        Comm_ChangeState(COMM_ONLINE);
    }
    else if (TimeExpired(state_enter_tick, pdMS_TO_TICKS(5000)))
    {
        Comm_ChangeState(COMM_RETRY_WAIT);
    }
    break;
```

使用 `now - start >= timeout` 的形式，也更容易正确处理 RTOS Tick 回绕。

## 7. 通信重试：指数退避

不要掉线以后每毫秒疯狂重连。

``` text
第1次：1s
第2次：2s
第3次：4s
第4次：8s
...
最大：30s
```

``` c
#define RETRY_MIN_MS   1000U
#define RETRY_MAX_MS  30000U

static uint32_t GetRetryDelay(uint32_t retry)
{
    uint32_t delay = RETRY_MIN_MS;

    while (retry-- > 0U)
    {
        if (delay >= RETRY_MAX_MS / 2U)
            return RETRY_MAX_MS;
        delay *= 2U;
    }

    return delay > RETRY_MAX_MS ? RETRY_MAX_MS : delay;
}
```

连接成功后把 `retry_count`
清零。大量联网设备还可加入少量随机抖动，避免同时重连。

## 8. Event Group：等待多个条件

例如系统必须等“传感器、通信、参数”都准备好。

``` c
#define EVT_SENSOR_READY  (1U << 0)
#define EVT_COMM_READY    (1U << 1)
#define EVT_PARAM_READY   (1U << 2)
#define EVT_ALL_READY (EVT_SENSOR_READY | EVT_COMM_READY | EVT_PARAM_READY)
```

等待：

``` c
EventBits_t bits = xEventGroupWaitBits(
    g_system_event,
    EVT_ALL_READY,
    pdFALSE,
    pdTRUE,
    pdMS_TO_TICKS(10000));

if ((bits & EVT_ALL_READY) == EVT_ALL_READY)
    StateMachine_Dispatch(&g_sm, EVENT_INIT_OK);
else
    Error_Report(ERR_INIT_TIMEOUT);
```

## 9. Mutex：共享设备只有一把钥匙

UART/I2C/SPI/Flash 可能被多个 Task 同时访问。

``` c
bool UART_SendSafe(const uint8_t *data, uint16_t len)
{
    if (data == NULL || len == 0U)
        return false;

    if (xSemaphoreTake(g_uart_mutex, pdMS_TO_TICKS(100)) != pdPASS)
        return false;

    HAL_StatusTypeDef ret =
        HAL_UART_Transmit(&huart1, (uint8_t *)data, len, 100);

    xSemaphoreGive(g_uart_mutex);
    return ret == HAL_OK;
}
```

拿锁以后，每条退出路径都必须释放锁。控制硬件时也不要随意
`portMAX_DELAY`，否则一个异常可能变成永久卡死。

## 10. 更好的共享方式：资源所有者

比“到处加锁”更好的办法往往是“不共享”。

``` text
其他 Task --> UART Tx Queue --> UART Task --> UART Hardware
```

只有 UART Task 操作 UART：

``` c
void UartTask(void *argument)
{
    UartMessage_t msg;

    for (;;)
    {
        xQueueReceive(g_uart_tx_queue, &msg, portMAX_DELAY);
        Driver_UART_Send(msg.data, msg.length);
    }
}
```

这样很多 UART 锁都可以消失。

## 11. 统一错误码

不要整个工程都是 `return -1`。

``` c
typedef enum
{
    ERR_OK = 0,

    ERR_SENSOR_BASE = 1000,
    ERR_SENSOR_TIMEOUT,
    ERR_SENSOR_CRC,
    ERR_SENSOR_DISCONNECTED,
    ERR_SENSOR_DATA_INVALID,

    ERR_COMM_BASE = 2000,
    ERR_COMM_TIMEOUT,
    ERR_COMM_DISCONNECTED,
    ERR_COMM_PROTOCOL,

    ERR_STORAGE_BASE = 3000,
    ERR_FLASH_WRITE,
    ERR_FLASH_CRC,

    ERR_SYSTEM_BASE = 9000,
    ERR_QUEUE_FULL,
    ERR_MUTEX_TIMEOUT,
    ERR_TASK_STALL,
    ERR_INIT_TIMEOUT
} ErrorCode_t;
```

错误集中投递：

``` c
void Error_Report(ErrorCode_t code)
{
    ErrorMessage_t msg = {
        .code = code,
        .tick = xTaskGetTickCount()
    };

    LOG_ERROR("error=%d", code);
    (void)xQueueSend(g_error_queue, &msg, 0);
}
```

## 12. 错误分级

``` c
typedef enum
{
    ERROR_LEVEL_INFO,
    ERROR_LEVEL_WARNING,
    ERROR_LEVEL_RECOVERABLE,
    ERROR_LEVEL_FATAL
} ErrorLevel_t;
```

处理：

``` c
switch (Error_GetLevel(msg.code))
{
    case ERROR_LEVEL_INFO:
        break;
    case ERROR_LEVEL_WARNING:
        LOG_WARN("warning=%d", msg.code);
        break;
    case ERROR_LEVEL_RECOVERABLE:
        System_RequestRecovery();
        break;
    case ERROR_LEVEL_FATAL:
        System_EnterSafeState();
        break;
}
```

临时通信超时和电机失控风险显然不能采用同一种处理策略。

## 13. Safe State：严重故障先保证硬件安全

``` c
void System_EnterSafeState(void)
{
    Motor_Stop();
    Heater_Off();
    Relay_Off();
    BSP_LED_Set(LED_ERROR, true);
    LOG_ERROR("enter safe state");
}
```

推荐流程：

``` text
发现严重错误
  ↓
关闭危险输出
  ↓
保存最小故障信息
  ↓
尝试恢复或停止喂 IWDG
  ↓
受控复位
```

## 14. 软件看门狗 + 硬件看门狗

如果 SensorTask 死了，但另一个 Task 还一直喂
IWDG，硬件狗发现不了业务已经坏掉。

``` c
typedef enum
{
    WD_TASK_SENSOR = 0,
    WD_TASK_CONTROL,
    WD_TASK_COMM,
    WD_TASK_LOG,
    WD_TASK_COUNT
} WatchdogTaskId_t;

typedef struct
{
    TickType_t last_feed_tick;
    TickType_t timeout_tick;
    bool enabled;
} SoftWatchdogNode_t;

static SoftWatchdogNode_t g_wd[WD_TASK_COUNT];
```

任务报到：

``` c
void SoftWatchdog_Feed(WatchdogTaskId_t id)
{
    if (id < WD_TASK_COUNT)
        g_wd[id].last_feed_tick = xTaskGetTickCount();
}
```

检查：

``` c
bool SoftWatchdog_CheckAll(void)
{
    TickType_t now = xTaskGetTickCount();

    for (uint32_t i = 0; i < WD_TASK_COUNT; ++i)
    {
        if (!g_wd[i].enabled)
            continue;

        if ((TickType_t)(now - g_wd[i].last_feed_tick) > g_wd[i].timeout_tick)
        {
            Error_Report(ERR_TASK_STALL);
            return false;
        }
    }
    return true;
}
```

MonitorTask：

``` c
void MonitorTask(void *argument)
{
    for (;;)
    {
        if (SoftWatchdog_CheckAll())
        {
            BSP_IWDG_Feed();
        }
        else
        {
            System_EnterSafeState();
            /* 不再喂硬件狗，最终由 IWDG 复位 */
        }

        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

保护链：

``` text
关键 Task -> 软件看门狗 -> MonitorTask -> IWDG -> MCU Reset
```

## 15. 传感器故障去抖

一次失败可能只是 EMI 干扰，不要一次失败就重启。

``` c
#define SENSOR_MAX_FAIL_COUNT 3U

static uint8_t fail_count;

void Sensor_Process(void)
{
    SensorData_t data;

    if (Sensor_Read(&data))
    {
        fail_count = 0;
        Sensor_Publish(&data);
    }
    else
    {
        ++fail_count;

        if (fail_count >= SENSOR_MAX_FAIL_COUNT)
        {
            Error_Report(ERR_SENSOR_DISCONNECTED);
            fail_count = 0;
        }
    }
}
```

逻辑就是：

``` text
失败1次 -> 观察
失败2次 -> 警惕
失败3次 -> 确认故障
```

## 16. 数据合法性检查

驱动返回 OK，不代表物理数据一定合理。

``` c
bool Temperature_IsValid(float value)
{
    return isfinite(value) &&
           value >= -40.0f &&
           value <= 125.0f;
}
```

还可以检查变化速度：

``` c
if (fabsf(current - previous) > MAX_TEMP_CHANGE_PER_SAMPLE)
{
    Error_Report(ERR_SENSOR_DATA_INVALID);
}
```

## 17. 简单滑动平均滤波

``` c
#define FILTER_SIZE 8

typedef struct
{
    float buffer[FILTER_SIZE];
    uint8_t index;
    uint8_t count;
    float sum;
} MovingAverage_t;

float MovingAverage_Update(MovingAverage_t *f, float value)
{
    f->sum -= f->buffer[f->index];
    f->buffer[f->index] = value;
    f->sum += value;

    f->index = (uint8_t)((f->index + 1U) % FILTER_SIZE);

    if (f->count < FILTER_SIZE)
        ++f->count;

    return f->sum / (float)f->count;
}
```

噪声偶尔蹦一下时，控制输出不会马上跟着发疯。

## 18. 控制阈值必须有回差

错误做法：30℃ 开，29.99℃ 关，温度在 30℃ 附近会疯狂开关。

``` c
#define FAN_ON_TEMP  30.0f
#define FAN_OFF_TEMP 28.0f

static bool fan_on;

void FanControl_Update(float temp)
{
    if (!fan_on && temp >= FAN_ON_TEMP)
    {
        Fan_On();
        fan_on = true;
    }
    else if (fan_on && temp <= FAN_OFF_TEMP)
    {
        Fan_Off();
        fan_on = false;
    }
}
```

像门一样：30℃ 推开，开了以后要降到 28℃ 才关。

## 19. 异步日志

实时 Task 不应该因为 `printf` 卡几十毫秒。

``` text
SensorTask ----\
ControlTask ----> Log Queue -> LogTask -> UART/Flash
CommTask ------/
```

``` c
#define LOG_TEXT_MAX 96

typedef struct
{
    LogLevel_t level;
    uint32_t tick;
    char text[LOG_TEXT_MAX];
} LogMessage_t;
```

投递时不要无限等：

``` c
if (xQueueSend(g_log_queue, &msg, 0) != pdPASS)
{
    ++g_log_drop_count;
}
```

日志队列满时，宁可统计低等级日志丢失，也不要反过来拖死关键控制任务。

## 20. 日志限流

错误每 1ms 打一次，日志自己就会变成事故。

``` c
static TickType_t last_log;

if ((TickType_t)(xTaskGetTickCount() - last_log) >= pdMS_TO_TICKS(1000))
{
    LOG_WARN("sensor still offline, total=%lu", error_count);
    last_log = xTaskGetTickCount();
}
```

## 21. Flash 参数：Magic + Version + CRC + Range

``` c
typedef struct
{
    uint32_t magic;
    uint16_t version;
    float fan_on_temp;
    float fan_off_temp;
    uint32_t report_period_ms;
    uint32_t crc32;
} DeviceConfig_t;

#define CONFIG_MAGIC   0x43464731UL
#define CONFIG_VERSION 1U
```

加载：

``` c
bool Config_Load(DeviceConfig_t *cfg)
{
    if (cfg == NULL)
        return false;

    if (!Flash_ReadConfig(cfg) ||
        cfg->magic != CONFIG_MAGIC ||
        cfg->version != CONFIG_VERSION ||
        !Config_CheckCRC(cfg) ||
        !Config_CheckRange(cfg))
    {
        Config_LoadDefault(cfg);
        return false;
    }

    return true;
}
```

CRC 正确也可能保存了业务上荒唐的数值，所以范围检查不能省。

``` c
bool Config_CheckRange(const DeviceConfig_t *cfg)
{
    if (cfg->fan_on_temp < 10.0f || cfg->fan_on_temp > 80.0f)
        return false;

    if (cfg->fan_off_temp >= cfg->fan_on_temp)
        return false;

    if (cfg->report_period_ms < 100U || cfg->report_period_ms > 60000U)
        return false;

    return true;
}
```

## 22. Flash A/B 双备份

写 Flash 中途掉电时，单份配置可能损坏。

``` text
Slot A: sequence=100 + config + CRC
Slot B: sequence=101 + config + CRC
```

启动：

``` text
A/B 都有效 -> 选 sequence 更新的
只有 A 有效 -> A
只有 B 有效 -> B
都无效 -> 默认配置
```

伪代码：

``` c
bool a_ok = ReadAndCheck(SLOT_A, &a);
bool b_ok = ReadAndCheck(SLOT_B, &b);

if (a_ok && b_ok)
    cfg = SequenceNewer(a.sequence, b.sequence) ? a.config : b.config;
else if (a_ok)
    cfg = a.config;
else if (b_ok)
    cfg = b.config;
else
    Config_LoadDefault(&cfg);
```

## 23. Stack / Heap / Queue 水位监控

Task 栈：

``` c
UBaseType_t remain = uxTaskGetStackHighWaterMark(g_sensor_task_handle);

if (remain < SENSOR_STACK_WARN_WORDS)
{
    LOG_WARN("SensorTask stack low=%u", (unsigned)remain);
}
```

Heap：

``` c
size_t free_heap = xPortGetFreeHeapSize();
size_t min_heap  = xPortGetMinimumEverFreeHeapSize();

if (min_heap < APP_HEAP_WARNING_BYTES)
{
    LOG_WARN("heap low free=%u min=%u",
             (unsigned)free_heap,
             (unsigned)min_heap);
}
```

Queue：

``` c
UBaseType_t waiting = uxQueueMessagesWaiting(g_app_queue);

if (waiting > APP_QUEUE_WARNING_LEVEL)
{
    LOG_WARN("app queue high=%u", (unsigned)waiting);
}
```

注意 `uxTaskGetStackHighWaterMark()` 的单位应结合具体 FreeRTOS 端口和
`StackType_t` 理解，不要直接想当然当作字节。

## 24. Stack Overflow / malloc Hook

建议开发阶段开启：

``` c
#define configCHECK_FOR_STACK_OVERFLOW 2
#define configUSE_MALLOC_FAILED_HOOK   1
```

``` c
void vApplicationStackOverflowHook(TaskHandle_t task, char *name)
{
    taskDISABLE_INTERRUPTS();
    BSP_EmergencyOutputsOff();

    for (;;)
    {
        /* 等待 IWDG 或执行经过设计的复位流程 */
    }
}

void vApplicationMallocFailedHook(void)
{
    taskDISABLE_INTERRUPTS();
    BSP_EmergencyOutputsOff();

    for (;;)
    {
    }
}
```

Hook 中不要再依赖一大堆“可能已经坏掉”的 RTOS 服务。

## 25. 静态创建 RTOS 对象

高可靠项目通常希望运行时内存更可控。

``` c
#define SENSOR_TASK_STACK_SIZE 512U

static StaticTask_t g_sensor_tcb;
static StackType_t g_sensor_stack[SENSOR_TASK_STACK_SIZE];

void CreateSensorTask(void)
{
    TaskHandle_t h = xTaskCreateStatic(
        SensorTask,
        "sensor",
        SENSOR_TASK_STACK_SIZE,
        NULL,
        tskIDLE_PRIORITY + 3,
        g_sensor_stack,
        &g_sensor_tcb);

    configASSERT(h != NULL);
}
```

也可以考虑 `xQueueCreateStatic()` 等静态 API。

## 26. ISR：只做“按门铃”

ISR 不要打印、写 Flash、跑复杂状态机。

``` c
void HAL_GPIO_EXTI_Callback(uint16_t pin)
{
    BaseType_t hpw = pdFALSE;
    uint8_t key = KEY_USER;

    xQueueSendFromISR(g_key_queue, &key, &hpw);
    portYIELD_FROM_ISR(hpw);
}
```

真正业务：

``` c
void KeyTask(void *argument)
{
    uint8_t key;

    for (;;)
    {
        if (xQueueReceive(g_key_queue, &key, portMAX_DELAY) == pdPASS)
            Key_Process(key);
    }
}
```

## 27. Task Notification：轻量敲门

DMA 完成这种“只需要告诉某一个 Task 一声”的事件，可以用 Notification。

ISR：

``` c
vTaskNotifyGiveFromISR(g_uart_task_handle, &hpw);
```

Task：

``` c
for (;;)
{
    ulTaskNotifyTake(pdTRUE, portMAX_DELAY);
    UART_HandleTxComplete();
}
```

## 28. 周期 Task 使用 vTaskDelayUntil

普通 `vTaskDelay(100ms)` 会把函数执行时间也加到周期里。

``` c
void ControlTask(void *argument)
{
    TickType_t last = xTaskGetTickCount();

    for (;;)
    {
        Control_Process();

        vTaskDelayUntil(
            &last,
            pdMS_TO_TICKS(20));
    }
}
```

更接近 0、20、40、60ms 的固定节拍。

## 29. 软件 Timer Callback 不干重活

``` c
static void ReportTimerCallback(TimerHandle_t timer)
{
    AppMessage_t msg = {.id = MSG_REPORT_TRIGGER};

    /* 只投递事件 */
    (void)xQueueSend(g_app_queue, &msg, 0);
}
```

不要在 Timer Service Task 里直接做耗时网络发送。

## 30. 通信协议解析也应该是状态机

协议：

``` text
SOF | LEN | CMD | DATA | CRC
```

解析状态：

``` c
typedef enum
{
    PARSER_WAIT_SOF,
    PARSER_WAIT_LEN,
    PARSER_WAIT_BODY,
    PARSER_WAIT_CRC
} ParserState_t;
```

伪代码：

``` text
WAIT_SOF:
    byte == 0xAA -> WAIT_LEN

WAIT_LEN:
    长度非法 -> RESET
    长度合法 -> WAIT_BODY

WAIT_BODY:
    收够数据 -> WAIT_CRC

WAIT_CRC:
    CRC 正确 -> 发布完整帧
    然后回 WAIT_SOF
```

千万不要相信外部长度：

``` c
if (len > PROTOCOL_MAX_PAYLOAD)
{
    Parser_Reset();
    Error_Report(ERR_COMM_PROTOCOL);
    return;
}
```

## 31. 所有外部输入都先检查

``` text
UART/CAN/TCP/Flash/传感器/上位机命令
        ↓
长度
        ↓
范围
        ↓
格式
        ↓
CRC
        ↓
当前状态是否允许
        ↓
真正执行
```

例如：

``` c
bool SetFanThreshold(float value)
{
    if (!isfinite(value))
        return false;

    if (value < 10.0f || value > 80.0f)
        return false;

    g_config.fan_on_temp = value;
    return true;
}
```

## 32. Recovery 状态机

``` c
typedef enum
{
    RECOVERY_IDLE,
    RECOVERY_STOP_OUTPUT,
    RECOVERY_RESET_DRIVER,
    RECOVERY_REINIT,
    RECOVERY_VERIFY,
    RECOVERY_DONE,
    RECOVERY_FAILED
} RecoveryState_t;
```

``` c
void Recovery_Run(void)
{
    switch (g_recovery_state)
    {
        case RECOVERY_STOP_OUTPUT:
            System_EnterSafeState();
            g_recovery_state = RECOVERY_RESET_DRIVER;
            break;

        case RECOVERY_RESET_DRIVER:
            Sensor_DeInit();
            g_recovery_state = RECOVERY_REINIT;
            break;

        case RECOVERY_REINIT:
            g_recovery_state =
                Sensor_Init() ? RECOVERY_VERIFY : RECOVERY_FAILED;
            break;

        case RECOVERY_VERIFY:
            g_recovery_state =
                Sensor_SelfTest() ? RECOVERY_DONE : RECOVERY_FAILED;
            break;

        default:
            break;
    }
}
```

恢复也必须有限制：

``` c
#define MAX_RECOVERY_COUNT 3U

if (recovery_count < MAX_RECOVERY_COUNT)
{
    ++recovery_count;
    Recovery_Start();
}
else
{
    System_EnterPermanentFault();
}
```

## 33. 优雅降级

网络坏了，不一定要把本地温控一起杀掉。

``` c
if (!Comm_IsOnline())
{
    System_SetMode(MODE_OFFLINE);
    LocalControl_Enable();
    CloudUpload_Disable();
    Comm_RequestReconnect();
}
```

思想是：坏一个非关键功能时，尽可能保留仍然安全可用的功能。

## 34. DMA Buffer 所有权

DMA 还在发送时，不可以马上改原 buffer。

``` text
Task 准备 Buffer A
   ↓
交给 DMA
   ↓
DMA 拥有 Buffer A
   ↓
完成中断
   ↓
Buffer A 归还池
```

高速采集可双缓冲：

``` text
DMA -> Buffer A    CPU -> Buffer B
完成后交换
DMA -> Buffer B    CPU -> Buffer A
```

伪代码：

``` c
void DMA_CompleteISR(void)
{
    BaseType_t hpw = pdFALSE;
    Buffer_t *finished = DMA_GetFinishedBuffer();

    xQueueSendFromISR(g_process_queue, &finished, &hpw);
    DMA_SwitchToNextBuffer();

    portYIELD_FROM_ISR(hpw);
}
```

## 35. 死锁与锁顺序

Task A：

``` text
拿 UART -> 等 FLASH
```

Task B：

``` text
拿 FLASH -> 等 UART
```

两个人可能永远互等。项目应规定统一顺序，例如：

``` text
CONFIG -> FLASH -> UART
```

所有模块只允许从左向右拿多个锁。

## 36. volatile 不是线程安全

``` c
volatile uint32_t counter;
counter++;
```

`volatile`
不自动提供原子性、Mutex、临界区或消息同步。它主要阻止编译器对特定访问做不恰当优化。跨
Task 的共享数据仍要用正确同步机制。

## 37. Demo 公共类型

``` c
typedef struct
{
    float temperature;
    float voltage;
    uint32_t tick;
    bool valid;
} SensorData_t;

typedef enum
{
    APP_MSG_SENSOR = 0,
    APP_MSG_BUTTON,
    APP_MSG_REPORT,
    APP_MSG_ERROR
} AppMessageId_t;

typedef struct
{
    AppMessageId_t id;

    union
    {
        SensorData_t sensor;

        struct {
            uint8_t id;
            uint8_t action;
        } button;

        uint32_t error_code;
    } payload;
} AppMessage_t;
```

## 38. Demo 配置

``` c
#define APP_SENSOR_PERIOD_MS   100U
#define APP_MONITOR_PERIOD_MS  500U
#define APP_SENSOR_MAX_FAIL      3U

#define APP_FAN_ON_TEMP         30.0f
#define APP_FAN_OFF_TEMP        28.0f

#define APP_QUEUE_LENGTH        16U
#define LOG_QUEUE_LENGTH        32U
```

把魔法数字集中起来。以后“采样 100ms 改 50ms”不需要全工程寻找数字 `100`。

## 39. Demo SensorTask

``` c
void SensorTask(void *argument)
{
    TickType_t last = xTaskGetTickCount();
    uint8_t fail_count = 0;

    for (;;)
    {
        SensorData_t data = {0};

        bool ok = Driver_Temperature_Read(&data.temperature);

        if (ok && Temperature_IsValid(data.temperature))
        {
            data.valid = true;
            data.tick = xTaskGetTickCount();
            fail_count = 0;

            AppMessage_t msg = {0};
            msg.id = APP_MSG_SENSOR;
            msg.payload.sensor = data;

            if (xQueueSend(g_app_queue, &msg, pdMS_TO_TICKS(10)) != pdPASS)
                Error_Report(ERR_QUEUE_FULL);
        }
        else
        {
            ++fail_count;

            if (fail_count >= APP_SENSOR_MAX_FAIL)
            {
                Error_Report(ERR_SENSOR_DISCONNECTED);
                fail_count = 0;
            }
        }

        SoftWatchdog_Feed(WD_TASK_SENSOR);
        vTaskDelayUntil(&last, pdMS_TO_TICKS(APP_SENSOR_PERIOD_MS));
    }
}
```

这个 Task
已经包含：固定周期、合法性检查、连续失败判定、队列返回值、错误上报和软件看门狗。

## 40. Demo ControlTask

``` c
static StateMachine_t g_sm;
static bool g_fan_on;

static void Control_UpdateFan(float temp)
{
    if (!g_fan_on && temp >= APP_FAN_ON_TEMP)
    {
        Driver_Fan_Set(true);
        g_fan_on = true;
        LOG_INFO("fan on");
    }
    else if (g_fan_on && temp <= APP_FAN_OFF_TEMP)
    {
        Driver_Fan_Set(false);
        g_fan_on = false;
        LOG_INFO("fan off");
    }
}

void ControlTask(void *argument)
{
    AppMessage_t msg;
    StateMachine_Init(&g_sm);

    for (;;)
    {
        if (xQueueReceive(g_app_queue, &msg, pdMS_TO_TICKS(100)) == pdPASS)
        {
            switch (msg.id)
            {
                case APP_MSG_SENSOR:
                    if (g_sm.current == SYS_STATE_RUNNING)
                        Control_UpdateFan(msg.payload.sensor.temperature);
                    break;

                case APP_MSG_BUTTON:
                    StateMachine_Dispatch(&g_sm, EVENT_START);
                    break;

                case APP_MSG_ERROR:
                    StateMachine_Dispatch(&g_sm, EVENT_FAULT);
                    break;

                default:
                    break;
            }
        }

        StateMachine_Run();
        SoftWatchdog_Feed(WD_TASK_CONTROL);
    }
}
```

## 41. Demo CommunicationTask

``` c
typedef enum
{
    COMM_OFFLINE = 0,
    COMM_CONNECTING,
    COMM_ONLINE,
    COMM_RETRY_WAIT
} CommState_t;

void CommunicationTask(void *argument)
{
    CommState_t state = COMM_OFFLINE;
    uint32_t retry_count = 0;
    TickType_t state_tick = xTaskGetTickCount();

    for (;;)
    {
        switch (state)
        {
            case COMM_OFFLINE:
                Driver_Comm_StartConnect();
                state = COMM_CONNECTING;
                state_tick = xTaskGetTickCount();
                break;

            case COMM_CONNECTING:
                if (Driver_Comm_IsOnline())
                {
                    retry_count = 0;
                    state = COMM_ONLINE;
                }
                else if (TimeExpired(state_tick, pdMS_TO_TICKS(5000)))
                {
                    ++retry_count;
                    state = COMM_RETRY_WAIT;
                    state_tick = xTaskGetTickCount();
                }
                break;

            case COMM_ONLINE:
                Comm_ProcessRx();
                Comm_ProcessTx();

                if (!Driver_Comm_IsOnline())
                {
                    state = COMM_RETRY_WAIT;
                    state_tick = xTaskGetTickCount();
                }
                break;

            case COMM_RETRY_WAIT:
                if (TimeExpired(state_tick,
                                pdMS_TO_TICKS(GetRetryDelay(retry_count))))
                {
                    state = COMM_OFFLINE;
                }
                break;

            default:
                state = COMM_OFFLINE;
                break;
        }

        SoftWatchdog_Feed(WD_TASK_COMM);
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

## 42. Demo MonitorTask

``` c
void MonitorTask(void *argument)
{
    TickType_t last = xTaskGetTickCount();

    for (;;)
    {
        bool healthy = SoftWatchdog_CheckAll();

        Monitor_CheckStack();
        Monitor_CheckHeap();
        Monitor_CheckQueues();
        RuntimeSelfTest();

        if (healthy)
        {
            BSP_IWDG_Feed();
        }
        else
        {
            System_EnterSafeState();
            /* 不再喂 IWDG */
        }

        vTaskDelayUntil(&last, pdMS_TO_TICKS(APP_MONITOR_PERIOD_MS));
    }
}
```

## 43. Demo RTOS 对象初始化

``` c
bool App_CreateRtosObjects(void)
{
    g_app_queue = xQueueCreate(APP_QUEUE_LENGTH, sizeof(AppMessage_t));
    g_log_queue = xQueueCreate(LOG_QUEUE_LENGTH, sizeof(LogMessage_t));
    g_error_queue = xQueueCreate(8, sizeof(ErrorMessage_t));

    g_uart_mutex = xSemaphoreCreateMutex();
    g_system_event = xEventGroupCreate();

    if (g_app_queue == NULL ||
        g_log_queue == NULL ||
        g_error_queue == NULL ||
        g_uart_mutex == NULL ||
        g_system_event == NULL)
    {
        return false;
    }

    return true;
}
```

初始化失败不能假装成功。

## 44. Demo 主初始化

``` c
void App_MainInit(void)
{
    /* 上电首先保证危险输出安全 */
    BSP_EmergencyOutputsOff();

    Logger_EarlyInit();

    if (!App_CreateRtosObjects())
        System_Fatal(ERR_RTOS_OBJECT_CREATE);

    /* 参数坏了可以退回默认值 */
    Parameter_Init();

    if (!Driver_InitAll())
        System_Fatal(ERR_DRIVER_INIT);

    SoftWatchdog_Init();

    if (!App_CreateTasks())
        System_Fatal(ERR_TASK_CREATE);

    LOG_INFO("application initialized");
}
```

推荐启动顺序：

``` text
Reset
 ↓
HAL/Clock
 ↓
危险输出安全值
 ↓
读取复位原因
 ↓
最小日志
 ↓
参数 + CRC
 ↓
Driver
 ↓
RTOS Object
 ↓
Task
 ↓
Self Test
 ↓
INIT 状态
 ↓
READY
```

## 45. 保存 Reset Reason

``` c
typedef enum
{
    RESET_UNKNOWN,
    RESET_POWER_ON,
    RESET_SOFTWARE,
    RESET_WATCHDOG,
    RESET_BROWNOUT
} ResetReason_t;
```

启动时：

``` c
ResetReason_t reason = BSP_GetResetReason();
LOG_INFO("boot reason=%d", reason);
```

现场突然重启时，至少能区分断电、软件复位和 Watchdog。

## 46. Crash Snapshot

``` c
typedef struct
{
    uint32_t tick;
    SystemState_t state;
    float temperature;
    uint32_t free_heap;
    uint32_t queue_depth;
    uint32_t error_code;
} FaultSnapshot_t;
```

``` c
void Fault_Capture(uint32_t error)
{
    FaultSnapshot_t s = {0};

    s.tick = xTaskGetTickCount();
    s.state = StateMachine_GetState();
    s.temperature = Sensor_GetTemperature();
    s.free_heap = xPortGetFreeHeapSize();
    s.queue_depth = uxQueueMessagesWaiting(g_app_queue);
    s.error_code = error;

    FaultStorage_Save(&s);
}
```

它就像小型“黑匣子”。

## 47. HardFault 原则

HardFault 后不要假设整个软件世界仍然正常。

``` text
关闭危险输出
 ↓
保存 PC/LR/CFSR/HFSR 等最小信息
 ↓
放到 Backup SRAM / 预留 RAM / 安全 Flash 区
 ↓
复位
```

伪代码：

``` c
void HardFault_Handler(void)
{
    BSP_EmergencyOutputsOff();
    Crash_SaveMinimalContext();
    NVIC_SystemReset();
}
```

真正的 Cortex-M 异常栈解析需要按 MCU、编译器和异常入口汇编代码实现。

## 48. 模块接口要窄

不要：

``` c
extern SensorContext_t g_sensor;
```

让所有文件都能乱改内部状态。推荐：

``` c
bool Sensor_GetLatest(SensorData_t *out);
bool Sensor_IsHealthy(void);
void Sensor_RequestReset(void);
```

而模块内部：

``` c
static SensorContext_t g_sensor;
```

`static` 是便宜又好用的围墙。

## 49. 统一返回值

``` c
typedef enum
{
    RET_OK = 0,
    RET_ERROR,
    RET_TIMEOUT,
    RET_BUSY,
    RET_INVALID_ARG,
    RET_NO_MEMORY,
    RET_NOT_READY
} Result_t;
```

Driver 把 HAL 返回值转换掉：

``` c
Result_t Driver_UART_Send(...)
{
    HAL_StatusTypeDef ret = HAL_UART_Transmit(...);

    switch (ret)
    {
        case HAL_OK:      return RET_OK;
        case HAL_TIMEOUT: return RET_TIMEOUT;
        case HAL_BUSY:    return RET_BUSY;
        default:          return RET_ERROR;
    }
}
```

上层不应该被 STM32 HAL 类型污染，这样以后换 MCU 更容易。

## 50. 防御式编程

公共 API：

``` c
bool Sensor_GetLatest(SensorData_t *out)
{
    if (out == NULL)
        return false;

    *out = g_sensor.latest;
    return true;
}
```

数组：

``` c
if (id >= MOTOR_COUNT)
    return false;

motor[id].speed = speed;
```

enum：

``` c
if (mode >= SYSTEM_MODE_MAX)
    return PROTOCOL_ERR_ARGUMENT;
```

整数长度：

``` c
if (len > UINT16_MAX - HEADER_SIZE)
    return false;

uint16_t total = len + HEADER_SIZE;
```

核心思想：不要把“调用者肯定不会传错”当作稳定性设计。

## 51. 状态机 + Queue 的推荐业务模型

``` text
ISR / Driver / Worker Task
          |
          | Event
          v
       App Queue
          |
          v
      AppTask
          |
          v
    State Machine
```

``` c
void AppTask(void *argument)
{
    AppEvent_t event;

    for (;;)
    {
        if (xQueueReceive(g_event_queue, &event, pdMS_TO_TICKS(100)) == pdPASS)
            AppStateMachine_Dispatch(&event);

        AppStateMachine_Run();
        SoftWatchdog_Feed(WD_TASK_APP);
    }
}
```

最大的好处之一：业务状态尽量只由一个 Task 修改，竞争条件少很多。

## 52. 状态转移表

规则很多时可以表驱动：

``` c
typedef struct
{
    SystemState_t from;
    AppEventId_t event;
    SystemState_t to;
} Transition_t;

static const Transition_t g_transitions[] =
{
    {SYS_STATE_INIT,    EVT_INIT_OK,      SYS_STATE_IDLE},
    {SYS_STATE_IDLE,    EVT_USER_START,   SYS_STATE_RUNNING},
    {SYS_STATE_RUNNING, EVT_USER_STOP,    SYS_STATE_IDLE},
    {SYS_STATE_RUNNING, EVT_SENSOR_FAULT, SYS_STATE_ERROR},
    {SYS_STATE_ERROR,   EVT_RECOVERY_OK,  SYS_STATE_IDLE},
};
```

无效事件：

``` c
if (!StateMachine_FindTransition(current, event.id, &next))
{
    LOG_WARN("invalid event=%d in state=%d", event.id, current);
    return;
}
```

ERROR 状态收到 START 时，不应该偷偷把设备启动。

## 53. 优先级设计

示例，不是固定答案：

``` text
SafetyTask  : 最高
ControlTask : 高
SensorTask  : 中高
CommTask    : 中
MonitorTask : 中低
LogTask     : 低
```

``` c
#define TASK_PRIO_SAFETY  (tskIDLE_PRIORITY + 5)
#define TASK_PRIO_CONTROL (tskIDLE_PRIORITY + 4)
#define TASK_PRIO_SENSOR  (tskIDLE_PRIORITY + 3)
#define TASK_PRIO_COMM    (tskIDLE_PRIORITY + 2)
#define TASK_PRIO_LOG     (tskIDLE_PRIORITY + 1)
```

真正设计要考虑
Deadline、执行时间、阻塞时间和共享资源，而不是“我觉得它重要所以给最高优先级”。

## 54. 优先级反转

``` text
Low 拿着 Mutex
High 等 Mutex
Medium 一直抢 Low 的 CPU
```

结果 High 被 Medium 间接拖住。FreeRTOS Mutex
有优先级继承机制，所以“保护互斥资源”时应使用正确的
Mutex，不要把所有同步都随便换成 Binary Semaphore。

## 55. 看门狗不要无脑喂

最差：

``` c
void TimerISR(void)
{
    HAL_IWDG_Refresh(&hiwdg);
}
```

业务全部死掉，只要 Timer ISR 还跑，设备就永远不复位。

正确思想：

``` text
Sensor OK? -----\
Control OK? -----\
Comm OK? ---------> Monitor -> 所有关键条件 OK -> Feed IWDG
Safety OK? ------/
```

Watchdog 超时时间还要大于关键路径的“合理最坏执行/阻塞时间 +
安全余量”，同时不能大得失去保护意义。

## 56. Self Test

``` c
bool System_SelfTest(void)
{
    bool ok = true;

    if (!Sensor_SelfTest())
    {
        LOG_ERROR("sensor self test failed");
        ok = false;
    }

    if (!Storage_SelfTest())
    {
        LOG_ERROR("storage self test failed");
        ok = false;
    }

    if (!Power_IsVoltageValid())
    {
        LOG_ERROR("voltage invalid");
        ok = false;
    }

    return ok;
}
```

不仅开机检查，MonitorTask 还可以做慢速 Runtime Self Test。

## 57. 故障注入测试

稳定性不是只测“所有东西都正常”。

应该故意测试：

``` text
拔传感器
断网
UART 喂乱码
CRC 错误
Queue 塞满
Flash 写失败
Mutex 拿不到
Task 不 Feed
状态永远不完成
```

测试钩子：

``` c
#ifdef FAULT_INJECTION
if (g_test_force_sensor_fail)
    return RET_TIMEOUT;
#endif
```

## 58. Watchdog 故障注入 Demo

``` c
#ifdef TEST_WATCHDOG
if (g_test_sensor_stall)
{
    for (;;)
    {
        /* 故意卡死并且不 Feed */
    }
}
#endif
```

预期链：

``` text
SensorTask 卡死
 ↓
Soft Watchdog 超时
 ↓
Monitor 发现
 ↓
Safe State
 ↓
停止喂 IWDG
 ↓
MCU Reset
 ↓
下次启动读取 RESET_WATCHDOG
```

只有真正测过这条链，才能知道 Watchdog 不是摆设。

## 59. Queue 满测试

``` c
for (int i = 0; i < 1000; ++i)
{
    AppMessage_t msg = {.id = APP_MSG_REPORT};

    if (xQueueSend(g_app_queue, &msg, 0) != pdPASS)
        ++g_diag.queue_full;
}
```

检查：

``` text
系统是否没有崩？
关键控制是否继续？
是否正确统计 drop？
日志会不会进一步把系统拖死？
Watchdog 是否仍正常？
```

## 60. 长时间压力测试

建议按产品重要程度做 24h、72h、7 天甚至更长 soak test。

持续观察：

``` text
Free Heap 是否一直下降？
Minimum Heap 是多少？
Task Stack 是否接近耗尽？
Queue 是否越来越满？
错误计数是否持续上涨？
CPU Load 是否异常？
是否出现 Watchdog Reset？
```

例如 Free Heap：

``` text
10000
9900
9800
9700
...
```

就应该怀疑泄漏。

## 61. 单元测试状态机

``` c
void Test_StateMachine(void)
{
    StateMachine_t sm;

    StateMachine_Init(&sm);
    ASSERT_EQ(sm.current, SYS_STATE_INIT);

    StateMachine_Dispatch(&sm, EVENT_INIT_OK);
    ASSERT_EQ(sm.current, SYS_STATE_IDLE);

    StateMachine_Dispatch(&sm, EVENT_START);
    ASSERT_EQ(sm.current, SYS_STATE_RUNNING);

    StateMachine_Dispatch(&sm, EVENT_FAULT);
    ASSERT_EQ(sm.current, SYS_STATE_ERROR);
}
```

还必须测非法组合，例如：

``` text
ERROR + START -> 不允许
INIT + STOP   -> 不允许
```

## 62. 企业代码注释写“为什么”

差：

``` c
i++; /* i 加 1 */
```

好：

``` c
/*
 * 连续 3 次失败才判定离线，
 * 避免一次 EMI 干扰导致系统误进入故障状态。
 */
if (++fail_count >= SENSOR_MAX_FAIL_COUNT)
{
    ...
}
```

注释最值钱的是解释：为什么这么设计、有什么坑、为什么不能使用看起来更简单的办法。

## 63. 命名里写单位

不要：

``` c
uint32_t timeout;
```

推荐：

``` c
uint32_t timeout_ms;
TickType_t timeout_ticks;
uint32_t frequency_hz;
uint32_t voltage_mv;
```

这能消灭很多“1000 到底是 ms 还是 tick”的错误。

## 64. 推荐任务表

| Task        | 触发/周期   | 职责               | 示例优先级 |
|-------------|-------------|--------------------|------------|
| SafetyTask  | 事件/高频   | 最终安全保护       | 最高       |
| ControlTask | 20ms/消息   | 状态机、控制       | 高         |
| SensorTask  | 100ms       | 采集、校验、滤波   | 中高       |
| CommTask    | 事件+短周期 | 协议、连接、重试   | 中         |
| ErrorTask   | 消息        | 集中错误策略       | 中         |
| MonitorTask | 500ms       | Watchdog、资源健康 | 中低       |
| LogTask     | 消息        | 异步日志           | 低         |

这是起点，不是所有项目的固定答案。

## 65. 推荐 Queue 表

| Queue      | Producer         | Consumer    | 内容       |
|------------|------------------|-------------|------------|
| AppQueue   | Sensor/Key/Error | App/Control | 业务事件   |
| LogQueue   | 所有模块         | LogTask     | 日志       |
| ErrorQueue | 所有模块         | ErrorTask   | 错误       |
| TxQueue    | App              | CommTask    | 待发送数据 |

## 66. 最终 main() Demo

``` c
int main(void)
{
    HAL_Init();
    SystemClock_Config();

    BSP_InitMinimal();

    /* 最先保证危险输出安全 */
    BSP_EmergencyOutputsOff();

    ResetReason_t reason = BSP_GetResetReason();

    Logger_EarlyInit();
    LOG_INFO("boot reason=%d", reason);

    App_MainInit();

    vTaskStartScheduler();

    /* 正常永远不应该返回 */
    System_Fatal(ERR_SCHEDULER_RETURNED);

    while (1)
    {
    }
}
```

## 67. 完整系统运行图

``` text
POWER ON
   |
   v
Safe Outputs
   |
   v
Load Config
   |
CRC + Version + Range
   |
   +------坏------> Default Config
   |
   v
Init Drivers
   |
   v
Self Test
   |
   +------失败----> RECOVERY ----失败----> ERROR/Safe State
   |                  |
   |                 成功
   v                  |
 IDLE <---------------+
   |
 START
   |
   v
RUNNING
   |
Runtime Fault
   |
   v
ERROR
   |
Safe State
   |
Recovery
```

## 68. “稳定性”到底是什么？

稳定不等于“永远没有 Bug”。更工程化的目标是：

``` text
预防 -> 检测 -> 隔离 -> 降级 -> 恢复 -> 记录
```

映射到代码：

``` text
参数检查        -> 预防
CRC/Watchdog    -> 检测
模块边界        -> 隔离
Offline Mode    -> 降级
Recovery FSM    -> 恢复
Logger/Snapshot -> 记录
```

每写一个重要模块都问六个问题：

``` text
1. 正常时怎么工作？
2. 失败时我怎么知道？
3. 卡死时谁知道？
4. 等太久怎么办？
5. 坏了能恢复吗？
6. 恢复不了时安全吗？
```

## 69. 最小落地顺序

第一次搭框架不用一天造完所有轮子：

``` text
1. 目录分层
2. Result_t
3. ErrorCode_t
4. Logger
5. Queue 消息总线
6. Application 状态机
7. Sensor/Control/Comm Task
8. Soft Watchdog + IWDG
9. Config CRC + 默认参数
10. Monitor：Stack/Heap/Queue
11. Recovery
12. Fault Injection
```

最核心骨架：

``` c
void AppTask(void *argument)
{
    AppEvent_t event;

    for (;;)
    {
        if (xQueueReceive(g_event_queue, &event, pdMS_TO_TICKS(100)) == pdPASS)
            AppStateMachine_Dispatch(&event);

        AppStateMachine_Run();
        SoftWatchdog_Feed(WD_TASK_APP);
    }
}
```

## 70. Code Review Checklist

``` text
[ ] 模块职责是否单一？
[ ] 公共 API 是否检查 NULL？
[ ] 外部输入是否检查长度？
[ ] enum/index 是否检查范围？
[ ] 是否存在无限等待？
[ ] RTOS API 返回值是否检查？
[ ] Queue 满时怎么办？
[ ] Mutex 拿不到怎么办？
[ ] 是否可能死锁？
[ ] ISR 是否足够短？
[ ] 状态机是否有 default 安全处理？
[ ] 状态是否有超时？
[ ] 硬件失败是否有重试？
[ ] 重试是否有限制/退避？
[ ] 是否支持降级？
[ ] 严重故障是否先 Safe State？
[ ] 关键 Task 是否有软件 Watchdog？
[ ] IWDG 是否在健康检查后才喂？
[ ] 参数是否有 CRC？
[ ] 参数是否有 Version？
[ ] 参数是否做范围检查？
[ ] Flash 写入掉电是否考虑？
[ ] Stack 是否监控？
[ ] Heap 是否监控？
[ ] Queue 水位是否监控？
[ ] 日志是否异步？
[ ] 日志是否限流？
[ ] 是否有错误计数？
[ ] 是否保存 Fault Snapshot？
[ ] 是否测试断网/拔传感器？
[ ] 是否测试 Queue 满？
[ ] 是否测试 Task 卡死？
[ ] 是否做长时间压力测试？
```

## 71. 继续升级的方向

框架稳定以后可以继续加入：

``` text
Bootloader
A/B Firmware
OTA
固件 Hash / 签名验证
Rollback
配置迁移
Crash Dump
CLI Shell
生产测试模式
低功耗状态机
CANopen / Modbus / MQTT
文件系统
Secure Boot
单元测试
静态分析
CI 自动构建
MISRA C / CERT C
代码覆盖率
HIL 测试
```

OTA 也可以继续使用状态机：

``` text
IDLE
 ↓
DOWNLOAD
 ↓
VERIFY_HASH
 ↓
VERIFY_SIGNATURE
 ↓
WRITE
 ↓
VERIFY_IMAGE
 ↓
MARK_PENDING
 ↓
REBOOT
 ↓
SELF_TEST
 ├── OK   -> CONFIRM
 └── FAIL -> ROLLBACK
```

## 72. 工程口诀

``` text
任务要分工，数据走消息。
共享要加锁，最好少共享。

等待有超时，错误有编号。
状态有边界，切换有规则。

输入先检查，输出保安全。
失败别慌张，先尝试恢复。

恢复有次数，严重就降级。
任务要心跳，最后硬狗兜底。

参数有版本，数据有 CRC。
日志别阻塞，故障留现场。

栈堆要监控，队列看水位。
正常路径要测，异常路径更要测。
```


