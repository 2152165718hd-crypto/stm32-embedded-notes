<DOCUMENT filename="EXTI.md">
# EXTI 外部中断/事件控制器详解 (STM32F4/F7/H7 系列 · HAL 库)

> **标签**：#STM32 #EXTI #中断 #事件 #HAL库 #嵌入式 #外设 #寄存器映射

> **关联笔记**：[[STM32-GPIO]]、[[STM32-NVIC]]、[[STM32-时钟系统]]、[[STM32-外设概述]]

## 1. 外设概述与应用场景 (Introduction & Use Cases)

### 功能简介

EXTI（External Interrupt/Event Controller）是 STM32 用于处理外部中断和事件的专用控制器。它监控来自 GPIO 引脚（或其他外设）的信号变化，产生中断请求（发送到 NVIC）或事件（用于唤醒、低功耗等）。

### 主要特性（STM32F4/F7/H7 系列）

- 共 **23 条 EXTI 线**（F4/F7/H7 系列）：
  - **EXTI0 ~ EXTI15**：对应引脚编号 0~15，可由任意端口的同编号引脚触发（通过 SYSCFG 配置）
  - **EXTI16 ~ EXTI22**：专用线（PVD 输出、RTC 报警、USB 唤醒、Ethernet 唤醒等，视型号而定）
- 支持 **中断** 和 **事件** 两种模式
  - 中断：发送到 NVIC，可进入中断服务函数
  - 事件：不进入中断，仅用于唤醒或触发其他外设（如触发 ADC、启动定时器等）
- 触发方式：上升沿、下降沿、双边沿
- 软件触发（SWIER）
- 挂起标志（Pending）需手动清除
- 与 GPIO 紧密协作，但配置需额外涉及 SYSCFG 外设

### 典型应用场景

- 按键检测（去抖动后触发中断）
- 外部传感器信号（如限位开关、光电门）
- 低功耗唤醒（事件模式，从 Stop/Standby 唤醒）
- 外部信号同步（如触发捕获）
- 配合 RTC、LPTIM 等实现定时唤醒

## 2. 硬件原理与内部框图 (Hardware Architecture)

### 内部框图解析

EXTI 的核心是一个多路选择 + 边沿检测结构：

1. **信号来源选择**
   - EXTI0~15：通过 SYSCFG_EXTICR 寄存器选择来自 PAx/PBx/PCx... 的引脚（x 为线号）
   - 每条线只能选择一个端口的引脚

2. **边沿检测电路**
   - 上升沿检测（RTSR）
   - 下降沿检测（FTSR）
   - 同步器：将输入信号与 AHB 时钟同步（避免亚稳态）

3. **掩码与输出**
   - IMR（中断掩码）：控制是否产生中断请求
   - EMR（事件掩码）：控制是否产生事件
   - 生成的请求发送到 NVIC 或事件总线

4. **挂起寄存器（PR）**
   - 记录已触发的线，需软件写 1 清除

数据流向：
- GPIO 引脚 → SYSCFG 多路选择 → 边沿检测 → 掩码 → 中断/事件 → NVIC 或其他外设

### 时钟树关联

- EXTI 本身不需独立时钟，但信号选择依赖 **SYSCFG** 外设
- SYSCFG 挂载在 **APB2** 总线
- 时钟使能宏：`__HAL_RCC_SYSCFG_CLK_ENABLE()`

## 3. 核心机制：API 与寄存器映射 (Core Mapping: API vs Registers)

本节完整覆盖所有 EXTI 和 SYSCFG 相关寄存器及 HAL 库主要 API 映射。

### 引脚来源选择（SYSCFG）

**关键寄存器**：
- **SYSCFG_EXTICR1 ~ EXTICR4**（偏移 0x08 ~ 0x14，32 位，R/W，重置值 0x00000000）
  - 每个寄存器控制 4 条线（EXTI0~3、EXTI4~7、EXTI8~11、EXTI12~15）
  - 每条线占用 4 位（EXTIx[3:0]）
    - 0000：PAx
    - 0001：PBx
    - 0010：PCx
    - ... （依型号而定，最高到 PHx）

**HAL API 映射**：
HAL 库在 `HAL_GPIO_Init()` 中若 Mode 包含 `GPIO_MODE_IT_RISING`/`FALLING`/`RISING_FALLING`，会自动调用内部函数设置 SYSCFG_EXTICR。

手动也可通过直接写寄存器实现。

### 中断/事件掩码与触发选择

**关键寄存器**（EXTI 基地址 0x4001 3C00）：

- **EXTI_IMR**（中断掩码寄存器，偏移 0x00，32 位，R/W，重置值 0x0F940000 或类似）
  - 位 y：0 = 屏蔽中断，1 = 使能中断（仅 EXTI0~22 有效）

- **EXTI_EMR**（事件掩码寄存器，偏移 0x04，32 位，R/W，重置值 0x00000000）
  - 同上，用于事件

- **EXTI_RTSR**（上升沿触发选择寄存器，偏移 0x08，32 位，R/W，重置值 0x00000000）
  - 位 y：1 = 使能上升沿触发

- **EXTI_FTSR**（下降沿触发选择寄存器，偏移 0x0C，32 位，R/W，重置值 0x00000000）
  - 位 y：1 = 使能下降沿触发

- **EXTI_SWIER**（软件中断/事件寄存器，偏移 0x10，32 位，R/W，重置值 0x00000000）
  - 写 1 软件触发对应线

- **EXTI_PR**（挂起寄存器，偏移 0x14，32 位，R/W（写 1 清除），重置值 0x00000000）
  - 位 y：1 = 该线已触发，需写 1 清除

**HAL API 映射**：

EXTI 配置主要通过 GPIO 的中断模式完成：

```c
GPIO_InitStruct.Mode = GPIO_MODE_IT_RISING;   // RTSR=1, IMR=1
GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;  // FTSR=1, IMR=1
GPIO_InitStruct.Mode = GPIO_MODE_IT_RISING_FALLING; // RTSR=1, FTSR=1, IMR=1

GPIO_InitStruct.Mode = GPIO_MODE_EVT_RISING;  // RTSR=1, EMR=1（事件模式）
```

中断服务函数：
```c
void HAL_GPIO_EXTI_IRQHandler(uint16_t GPIO_Pin)
{
    // 自动清除 PR 对应位
    __HAL_GPIO_EXTI_CLEAR_IT(GPIO_Pin);
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    // 用户回调，处理具体逻辑
}
```

NVIC 配置（必须）：
```c
HAL_NVIC_SetPriority(EXTI0_IRQn, 2, 0);
HAL_NVIC_EnableIRQ(EXTI0_IRQn);
```

注意：EXTI0~4 各有独立 IRQ，EXTI5~9 共用 EXTI9_5_IRQn，EXTI10~15 共用 EXTI15_10_IRQn。

## 4. 高级特性 (Advanced Features)

### 事件模式

- 不产生中断，仅生成事件脉冲
- 可用于触发其他外设（如 ADC、定时器）或从低功耗模式唤醒
- 配置 EMR 而非 IMR

### 软件触发

- 通过 SWIER 写 1 可模拟外部触发，用于测试或同步

### 半线与全线中断

- 某些线（如 EXTI16/PVD）专用于特定功能

## 5. 标准配置流程 (Configuration Workflow)

1. **开启 SY 开启 SYSCFG 时钟**
   ```c
   __HAL_RCC_SYSCFG_CLK_ENABLE();
   ```

2. 配置 GPIO 为中断模式（自动配置 EXTI 和 SYSCFG）
   ```c
   __HAL_RCC_GPIOx_CLK_ENABLE();

   GPIO_InitTypeDef GPIO_InitStruct = {0};
   GPIO_InitStruct.Pin  = GPIO_PIN_x;
   GPIO_InitStruct.Mode = GPIO_MODE_IT_FALLING;  // 或 RISING / RISING_FALLING
   GPIO_InitStruct.Pull = GPIO_PULLUP;           // 通常按键用上拉
   HAL_GPIO_Init(GPIOx, &GPIO_InitStruct);
   ```

3. **配置 NVIC**
   ```c
   HAL_NVIC_SetPriority(EXTIx_IRQn, priority, 0);
   HAL_NVIC_EnableIRQ(EXTIx_IRQn);
   ```

4. **实现回调函数**
   ```c
   void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
   {
       if (GPIO_Pin == GPIO_PIN_x)
       {
           // 处理中断逻辑
       }
   }
   ```

## 6. 常见避坑与调试指南 (Pitfalls & Troubleshooting)

### 常见问题

- **忘记开启 SYSCFG 时钟**：EXTICR 配置无效，引脚来源默认为 PAx
- **未配置 NVIC**：中断使能但不进入服务函数
- **中断线冲突**：多引脚同线号时，只有一个能触发（先配置的生效）
- **未清除挂起位**：中断只触发一次，或持续触发
- **去抖动缺失**：机械按键产生多次中断
- **低功耗模式下中断失效**：需检查是否允许 EXTI 唤醒

### 调试技巧

- 查看 EXTI_PR 寄存器确认是否触发
- 用示波器观察引脚信号与中断进入时间
- 在中断服务函数开头加 LED 翻转，快速判断是否进入
- 使用 `__HAL_GPIO_EXTI_GET_IT(GPIO_Pin)` 判断具体引脚

## 7. 示例代码 (Example Code)

### 示例 1：按键中断（PC13，下降沿，上拉）

```c
void Key_EXTI_Config(void)
{
    __HAL_RCC_SYSCFG_CLK_ENABLE();
    __HAL_RCC_GPIOC_CLK_ENABLE();

    GPIO_InitTypeDef GPIO_InitStruct = {0};
    GPIO_InitStruct.Pin   = GPIO_PIN_13;
    GPIO_InitStruct.Mode  = GPIO_MODE_IT_FALLING;
    GPIO_InitStruct.Pull  = GPIO_PULLUP;
    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

    HAL_NVIC_SetPriority(EXTI15_10_IRQn, 2, 0);
    HAL_NVIC_EnableIRQ(EXTI15_10_IRQn);
}

void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    if (GPIO_Pin == GPIO_PIN_13)
    {
        HAL_Delay(20);  // 简单软件去抖
        if (HAL_GPIO_ReadPin(GPIOC, GPIO_PIN_13) == GPIO_PIN_RESET)
        {
            // 按键按下处理
            HAL_GPIO_TogglePin(GPIOA, GPIO_PIN_5);  // LED 翻转
        }
    }
}

void EXTI15_10_IRQHandler(void)
{
    HAL_GPIO_EXTI_IRQHandler(GPIO_PIN_13);
}
```

> **最后更新**：2026-01-28  
> 本笔记适用于 STM32F4/F7/H7 系列，内容来源于官方参考手册 + 实战经验总结。
</DOCUMENT>