# STM32 EXTI 实战笔记（F103 + StdPeriph）

> **标签**：#STM32 #EXTI #中断 #F103 #StdPeriph #嵌入式  
> **关联笔记**：[[STM32-NVIC-中断优先级]]、[[STM32-GPIO-基础]]、[[STM32-时钟系统-RCC]]  
> **适用范围**：STM32F103（Cortex-M3）+ 标准外设库（StdPeriph）

---

## 1. EXTI 概念与整体框架

### 1.1 EXTI 是什么，用来干什么

EXTI（External Interrupt/Event Controller，外部中断/事件控制器）负责把“外部引脚的电平变化”转换为：

- **中断（Interrupt）**：触发 CPU 进入 ISR（中断服务函数），适合“需要立刻响应”的事情  
- **事件（Event）**：不进中断，仅产生事件脉冲（可用于唤醒、触发某些机制），使用场景相对少

常见用途（尽量具体）：

- 按键按下触发中断：翻转 LED、改变模式、唤醒系统
- 传感器 INT 引脚：比如震动、光电门、触摸、加速度计中断脚
- 外部模块就绪信号：如通信模块“数据就绪”引脚

> 核心价值：从“主循环轮询按键”升级为“边沿触发 + 事件驱动”，减少 CPU 空转，提高响应速度与功耗表现。

### 1.2 EXTI 与 GPIO、NVIC 的关系（谁负责什么）

把链路拆开看最清楚：

1. **GPIO**：提供引脚实际电平（输入路径采样），电平变化发生在引脚上  
2. **AFIO（F1 系列）**：负责把“某个端口的某个引脚”连接到“EXTI 某条线”  
3. **EXTI**：负责检测该线的上升/下降沿，置位挂起标志（PR），并向 NVIC 发中断请求  
4. **NVIC**：负责中断通道使能、优先级仲裁、最终让 CPU 跳转到对应 ISR（如 `EXTI0_IRQHandler`）

一句话记忆：

> GPIO 提供信号，AFIO 选路，EXTI 判边沿并置标志，NVIC 决定 CPU 是否响应。

### 1.3 “线（Line）”的概念：EXTI0~EXTI15 对应 GPIOx0~GPIOx15

EXTI 有 0~15 共 16 条“线”，每条线的编号对应引脚号：

- **EXTI0**：只能接到 `PA0 / PB0 / PC0 / ...` 其中一个（取决于 AFIO 映射）
- **EXTI1**：只能接到 `PA1 / PB1 / PC1 / ...` 其中一个
- ...
- **EXTI15**：只能接到 `PA15 / PB15 / PC15 / ...` 其中一个

重要限制（非常常见的坑）：

- **同一条 EXTI 线同一时刻只能选择一个端口来源**  
  例如 EXTI0 你选了 PA0，就不能同时再让 PB0 也触发 EXTI0（除非你改映射，但那是“切换来源”不是“并联”）。

---

## 2. 相关模块与寄存器概览（F1：AFIO + EXTI + NVIC）

### 2.1 AFIO：EXTI 引脚映射的“选路器”

STM32F1 没有 SYSCFG，使用 **AFIO** 完成 EXTI 的端口选择。

- 关键寄存器：`AFIO_EXTICR1 ~ AFIO_EXTICR4`
- 每个 EXTICR 管 4 条线：
  - EXTICR1：EXTI0~EXTI3
  - EXTICR2：EXTI4~EXTI7
  - EXTICR3：EXTI8~EXTI11
  - EXTICR4：EXTI12~EXTI15
- 每条线用 4bit 选择端口：PA/PB/PC/PD/PE...

StdPeriph 中常用 API：

- `GPIO_EXTILineConfig(GPIO_PortSourceGPIOA, GPIO_PinSource0);`

> 忘了 AFIO 映射：最典型表现就是“换到 PB0 后怎么都不进中断”。

### 2.2 EXTI 核心寄存器（掌握这 6 个就够用）

| 寄存器 | 作用 | 常用关注点 |
|---|---|---|
| `EXTI_IMR` | Interrupt mask：中断屏蔽/使能 | 1=不屏蔽（允许中断），0=屏蔽 |
| `EXTI_EMR` | Event mask：事件屏蔽/使能 | 一般按中断用可不配 |
| `EXTI_RTSR` | Rising trigger：上升沿触发 | 位=1 表示开启上升沿触发 |
| `EXTI_FTSR` | Falling trigger：下降沿触发 | 位=1 表示开启下降沿触发 |
| `EXTI_SWIER` | Software interrupt：软件触发 | 写 1 可软件触发对应线 |
| `EXTI_PR` | Pending：挂起标志 | **写 1 清零**（W1C），ISR 必清 |

> 经验：EXTI 的“是否会一直进中断”基本都跟 `PR` 清除有关。

### 2.3 NVIC 通道：EXTI0/1/… 与共享向量

F103 常见 EXTI 中断向量（按线号分组）：

- `EXTI0_IRQn`：EXTI0 独立向量
- `EXTI1_IRQn`：EXTI1 独立向量
- `EXTI2_IRQn`：EXTI2 独立向量
- `EXTI3_IRQn`：EXTI3 独立向量
- `EXTI4_IRQn`：EXTI4 独立向量
- `EXTI9_5_IRQn`：EXTI5~EXTI9 **共享**
- `EXTI15_10_IRQn`：EXTI10~EXTI15 **共享**

共享向量意味着：ISR 里必须判断到底是哪一条线触发（检查 `EXTI_PR`）。

---

## 3. 典型配置流程（F103 + StdPeriph）

### 3.1 记忆用的伪流程（建议抄到脑子里）

1. 开时钟（GPIOx + AFIO）  
2. 配 GPIO 为输入（上拉/下拉/浮空取决于外部电路）  
3. AFIO 映射：GPIOx.y → EXTIy  
4. 配 EXTI：边沿 + 使能 IMR（中断）  
5. 配 NVIC：选择 `EXTI?` 或共享向量，设优先级并使能  
6. 写 ISR：判断 PR → 清 PR → 做轻量工作（重活丢到主循环）

> 建议：每次“不进中断”就按这 6 步逐条对照排查。

### 3.2 对应到 StdPeriph 的函数清单（常用）

- 时钟：
  - `RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_AFIO, ENABLE);`
- GPIO：
  - `GPIO_Init(GPIOA, &GPIO_InitStructure);`
- AFIO 映射：
  - `GPIO_EXTILineConfig(GPIO_PortSourceGPIOA, GPIO_PinSource0);`
- EXTI：
  - `EXTI_Init(&EXTI_InitStructure);`
  - `EXTI_ClearITPendingBit(EXTI_Line0);`（清 PR）
- NVIC：
  - `NVIC_Init(&NVIC_InitStructure);`
  - `NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);`（可选）

---

## 4. 示例代码：按键触发 EXTI0 中断点灯（PA0）

### 4.1 硬件假设与设计选择

- 按键接在 `PA0`  
- 按下时把引脚拉到 GND（常见接法），松开为高电平  
- 因此：
  - GPIO 配置为 **上拉输入**
  - 中断触发选择 **下降沿**（High → Low 表示按下）

LED 假设接在 `PC13`（仅示例，按你板子修改）。

### 4.2 初始化代码（GPIO + EXTI + NVIC）

```c
#include "stm32f10x.h"

static void LED_PC13_Config(void)
{
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOC, ENABLE);

    GPIO_InitTypeDef gpio = {0};
    gpio.GPIO_Pin   = GPIO_Pin_13;
    gpio.GPIO_Speed = GPIO_Speed_2MHz;
    gpio.GPIO_Mode  = GPIO_Mode_Out_PP;
    GPIO_Init(GPIOC, &gpio);

    GPIO_SetBits(GPIOC, GPIO_Pin_13); // 先灭（很多板子 PC13 低亮高灭，按实际调整）
}

static void KEY_PA0_EXTI0_Config(void)
{
    /* 1) 开时钟：GPIOA + AFIO（EXTI 映射依赖 AFIO） */
    RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA | RCC_APB2Periph_AFIO, ENABLE);

    /* 2) 配 GPIO：PA0 上拉输入（按下为低） */
    GPIO_InitTypeDef gpio = {0};
    gpio.GPIO_Pin   = GPIO_Pin_0;
    gpio.GPIO_Speed = GPIO_Speed_2MHz;
    gpio.GPIO_Mode  = GPIO_Mode_IPU; // 内部上拉
    GPIO_Init(GPIOA, &gpio);

    /* 3) AFIO 映射：把 PA0 连接到 EXTI0 */
    GPIO_EXTILineConfig(GPIO_PortSourceGPIOA, GPIO_PinSource0);

    /* 4) 配 EXTI：EXTI0，下降沿触发，中断模式，使能 */
    EXTI_InitTypeDef exti = {0};
    exti.EXTI_Line    = EXTI_Line0;
    exti.EXTI_Mode    = EXTI_Mode_Interrupt;
    exti.EXTI_Trigger = EXTI_Trigger_Falling; // 按下触发
    exti.EXTI_LineCmd = ENABLE;
    EXTI_Init(&exti);

    /* 可选：先清一次挂起，避免上电抖动导致直接进中断 */
    EXTI_ClearITPendingBit(EXTI_Line0);

    /* 5) 配 NVIC：使能 EXTI0 中断通道 + 优先级 */
    NVIC_PriorityGroupConfig(NVIC_PriorityGroup_2);

    NVIC_InitTypeDef nvic = {0};
    nvic.NVIC_IRQChannel = EXTI0_IRQn;
    nvic.NVIC_IRQChannelPreemptionPriority = 1;
    nvic.NVIC_IRQChannelSubPriority        = 1;
    nvic.NVIC_IRQChannelCmd = ENABLE;
    NVIC_Init(&nvic);
}
```

### 4.3 ISR（中断服务函数）

要点：

- 先判断是哪条线（即使是独立向量也建议养成习惯）
- **必须清 PR**：`EXTI_ClearITPendingBit()` 内部就是 PR 写 1 清零
- ISR 里尽量只做“置标志/轻量动作”，不要长延时

```c
void EXTI0_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line0) != RESET)
    {
        /* 清挂起标志：必须做，否则可能重复进入中断 */
        EXTI_ClearITPendingBit(EXTI_Line0);

        /* 示例动作：翻转 LED */
        if (GPIO_ReadOutputDataBit(GPIOC, GPIO_Pin_13))
            GPIO_ResetBits(GPIOC, GPIO_Pin_13);
        else
            GPIO_SetBits(GPIOC, GPIO_Pin_13);
    }
}
```

### 4.4 main 初始化顺序示例

```c
int main(void)
{
    LED_PC13_Config();
    KEY_PA0_EXTI0_Config();

    while (1)
    {
        /* 主循环做别的事；按键触发由 EXTI 异步响应 */
    }
}
```

> 提示：如果你在 Keil/IDE 里发现 ISR 根本没进，优先检查启动文件里是否有 `EXTI0_IRQHandler` 的弱符号被覆盖/是否名字写错。

---

## 5. 进阶与细节（从“能用”到“好用”）

### 5.1 边沿触发怎么选

- **上升沿（RTSR）**：信号从 0→1  
  典型：外部模块“数据就绪”拉高、光电门输出高脉冲
- **下降沿（FTSR）**：信号从 1→0  
  典型：上拉按键按下（高→低）
- **双边沿（RTSR+FTSR）**：两边都触发  
  典型：需要捕捉“按下/松开”两个动作，或编码器（但编码器一般更建议定时器编码器模式）

> 注意：双边沿 + 按键抖动 = 中断风暴，更需要消抖策略。

### 5.2 按键消抖：为什么不建议在 EXTI ISR 里 `delay`

常见错误做法：

- 在 `EXTI0_IRQHandler` 里 `DelayMs(10)`，然后再读电平确认

问题：

- ISR 里延时会阻塞其他中断响应（影响 SysTick、串口、定时器等）
- 抖动期间可能已经产生多次挂起，延时并不能从根本上“只触发一次”
- 最终表现为：系统卡顿、丢串口数据、定时不准

更好的思路（推荐之一）：

1. ISR 里只做两件事：清 PR + 记录时间/置标志  
2. 主循环或定时器里做消抖判定（例如 10~20ms 后再次确认电平）

示例（思想级伪代码）：

```c
volatile uint8_t key_event = 0;
volatile uint32_t key_tick = 0;

void EXTI0_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line0) != RESET)
    {
        EXTI_ClearITPendingBit(EXTI_Line0);
        key_event = 1;
        key_tick = SysTick_GetTick(); // 需要你有一个毫秒 tick
    }
}

/* main loop */
if (key_event && (SysTick_GetTick() - key_tick) > 20)
{
    key_event = 0;
    if (GPIO_ReadInputDataBit(GPIOA, GPIO_Pin_0) == Bit_RESET)
    {
        // 确认仍处于按下状态，认为一次有效按键
    }
}
```

> 如果你没有现成 tick：用 TIM 做 1ms/10ms 周期中断也可以。

### 5.3 共享向量（EXTI9_5、EXTI15_10）如何判断触发源

当多个线共享同一个 ISR，必须逐条检查挂起位：

```c
void EXTI9_5_IRQHandler(void)
{
    if (EXTI_GetITStatus(EXTI_Line5) != RESET)
    {
        EXTI_ClearITPendingBit(EXTI_Line5);
        // handle line5
    }

    if (EXTI_GetITStatus(EXTI_Line6) != RESET)
    {
        EXTI_ClearITPendingBit(EXTI_Line6);
        // handle line6
    }

    // ... line7/8/9
}
```

本质就是检查 `EXTI_PR` 哪些 bit 被置位。

### 5.4 PR 清除规则：写 1 清零（W1C）

EXTI 的挂起寄存器 `PR` 不是“写 0 清零”，而是：

- **写 1：清除对应 bit**
- 写 0：无效

StdPeriph 的 `EXTI_ClearITPendingBit(EXTI_LineX)` 已经封装了正确操作。

> 典型坑：忘记清 PR → ISR 退出后又立即再次进入，表现像“死循环中断”。

### 5.5 与低功耗/唤醒的简单联系（点到为止）

- EXTI 的“事件/中断”都可用于从 sleep/stop 等低功耗模式唤醒（具体取决于系列与低功耗模式）
- 实战上常见做法：按键 EXTI 作为唤醒源，唤醒后再在主循环完成业务处理

---

## 6. 常见问题与排查思路（FAQ）

### 6.1 为什么配置好了 EXTI，就是不进中断？

按从“最容易漏”到“相对底层”排查：

1. **时钟开了吗？**  
   - GPIO 端口时钟：`RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOx, ENABLE)`  
   - AFIO 时钟：`RCC_APB2PeriphClockCmd(RCC_APB2Periph_AFIO, ENABLE)`（EXTI 映射需要）
2. **GPIO 模式对吗？**  
   - 输入：`GPIO_Mode_IPU / IPD / IN_FLOATING`  
   - 外部电路是否需要上拉/下拉匹配（否则浮空乱跳）
3. **AFIO 映射写对了吗？（最常见）**  
   - `GPIO_EXTILineConfig(GPIO_PortSourceGPIOA, GPIO_PinSource0)`  
   - PinSource 要与线号一致（EXTI0 就是 PinSource0）
4. **EXTI 配置对吗？**  
   - `EXTI_Line` 是否对应  
   - `EXTI_Mode` 是否为 Interrupt  
   - `EXTI_Trigger` 上/下沿是否匹配实际按键电平变化  
   - `EXTI_LineCmd = ENABLE`  
5. **NVIC 通道开了吗？向量对吗？**  
   - EXTI0 用 `EXTI0_IRQn`，EXTI6 用 `EXTI9_5_IRQn`  
   - ISR 函数名必须严格：`EXTI0_IRQHandler` / `EXTI9_5_IRQHandler`
6. **PR 是否已经挂起但未清？**  
   - 上电抖动可能置位 PR；初始化后建议清一次  
   - ISR 必须清 `EXTI_ClearITPendingBit()`

> 调试技巧：在断点/Peripheral View 里看 `EXTI->PR`、`EXTI->IMR`、`AFIO->EXTICR[]` 三个寄存器，通常一眼能定位问题。

### 6.2 为什么按键按一次，触发多次中断？

常见原因：

- **机械按键抖动**：按下/松开都可能产生多次边沿  
- **触发边沿选错**：例如你选了双边沿，按下和松开都会触发  
- **输入悬空**：上拉/下拉没处理，导致噪声触发

解决建议：

- 硬件：RC 滤波、施密特触发器、按键专用芯片（成本允许时最省心）
- 软件：ISR 置标志 + 定时消抖（推荐），或忽略一定时间窗内的重复触发（节流）
- 配置：确认只开需要的边沿（通常按下只要下降沿）

### 6.3 为啥 EXTI0 用 PA0 时正常，改成 PB0 就不行了？

原因几乎都是：**EXTI0 仍然映射到 PA0，没有切到 PB0**。

要改为 PB0，需要：

- 开 GPIOB 时钟 + AFIO 时钟
- GPIOB0 配输入
- `GPIO_EXTILineConfig(GPIO_PortSourceGPIOB, GPIO_PinSource0);`

并保持 `EXTI_Line0` 和 `EXTI0_IRQn` 不变（线号不变，只是“端口来源”变了）。

### 6.4 ISR 里做了很多事后系统变卡/串口丢数据？

典型原因：

- ISR 内做了长循环、打印、延时、复杂计算，导致高优先级占用过久
- 其他中断（串口接收、SysTick、定时器）得不到及时响应

建议：

- ISR 里只做：清标志 + 快速采样 + 置位事件标志/写队列
- 真正的业务处理放到主循环或 RTOS 任务
- 必要时调整 NVIC 优先级（但不要用优先级掩盖“ISR 太重”的结构问题）

---

## 7. 小结（复习速记）

- EXTI 本质是“边沿检测 + 挂起标志 + 触发 NVIC”
- F103 配 EXTI 的关键链路：`GPIO 输入` → `AFIO EXTICR 映射` → `EXTI IMR/RTSR/FTSR` → `NVIC 通道` → `ISR 清 PR`
- 同一条 EXTI 线（0~15）只能选择一个端口来源（PAx/PBx/PCx… 选其一）
- `EXTI_PR` 必须在 ISR 清除（写 1 清零），共享向量要逐条检查 PR
- 按键建议：下降沿 + 上拉输入 + ISR 置标志 + 主循环/定时器消抖，避免 ISR 延时