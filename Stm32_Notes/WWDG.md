<DOCUMENT filename="WWDG.md">
# WWDG 窗口看门狗详解 (STM32F4/F7/H7 系列 · HAL 库)

> **标签**：#STM32 #WWDG #看门狗 #HAL库 #嵌入式 #外设 #可靠性

> **关联笔记**：[[STM32-IWDG]]、[[STM32-RCC]]、[[STM32-时钟系统]]、[[STM32-NVIC]]、[[STM32-低功耗模式]]

## 1. 外设概述与应用场景 (Introduction & Use Cases)

### 功能简介

WWDG（Window Watchdog，窗口看门狗）是一个基于 APB1 时钟的硬件定时器，用于监控程序运行是否在预期的时间窗口内“喂狗”。与 IWDG 不同，WWDG 要求刷新操作必须在计数器进入指定“窗口”后进行，过早或过晚刷新都会导致复位，提供更严格的运行监控。

### 主要特性（STM32F4/F7/H7 系列）

- 时钟来源：**PCLK1**（APB1 时钟，通常几十 MHz）
- 7 位递减计数器（取值范围 0x40 ~ 0x7F，重置后从配置值开始）
- 可配置预分频器（1/2/4/8）
- **窗口机制**：仅在计数器值 < 窗口值（W[6:0]）且 > 0x3F 时允许刷新
- 支持**早期唤醒中断（EWI）**：计数器到达 0x40 时产生中断，可用于及时喂狗
- 可通过软件禁用（清除 WDGA 位）
- 超时时间较短：通常几 ms 到几十 ms
- 在 Stop/Standby 模式下行为可配置（通常停止）

### 典型应用场景

- 需要更精细监控的场景（如检测程序是否卡在某段代码过久或过早返回）
- 与 IWDG 配合使用：WWDG 监控主循环周期，IWDG 作为最终保险
- 实时性要求高的嵌入式系统
- 防止程序异常跳出主循环但仍能偶尔喂狗的情况

## 2. 硬件原理与内部框图 (Hardware Architecture)

### 内部框图解析

WWDG 结构：

1. **时钟来源**
   - 来自 APB1（PCLK1），经预分频器分频

2. **预分频器**
   - WDGTB[1:0]：分频系数 1、2、4、8

3. **7 位递减计数器**
   - 每 (4096 × 分频) 个 PCLK1 周期递减一次
   - 当计数器从 0x40 递减到 0x3F 时产生复位（除非及时刷新）

4. **窗口比较器**
   - 只有当计数器值 < 窗口值（W[6:0]）时才允许刷新
   - 过早刷新（计数器 ≥ 窗口值）也会触发复位

5. **早期唤醒中断**
   - 计数器 == 0x40 时置位 EWIF，产生中断

数据流向：
- PCLK1 → 预分频 → 7 位计数器 → 窗口比较 + 超时检测 → 系统复位 / EWI 中断

### 时钟树关联

- WWDG 挂载在 **APB1** 总线
- 时钟使能宏：`__HAL_RCC_WWDG_CLK_ENABLE()`

### 电气特性

- 超时时间计算公式（近似）：
  ```
  T_wwdg (ms) = (4096 × 2^WDGTB × (T[6:0] - 0x40 + 1)) / PCLK1(MHz)
  ```
  - T[6:0]：初始计数器值（0x40 ~ 0x7F，典型 0x7F 获取最长超时）
  - 实际以 PCLK1 频率为准，建议实测

## 3. 核心机制：API 与寄存器映射 (Core Mapping: API vs Registers)

### 关键寄存器（基地址 0x4000 2C00）

- **WWDG_CR**（控制寄存器，偏移 0x00，32 位，R/W）
  - 位 7：WDGA（使能位，1=使能）
  - 位 6:0：T[6:0]（计数器值，写 1 刷新并加载该值）

- **WWDG_CFR**（配置寄存器，偏移 0x04，32 位，R/W）
  - 位 9：EWI（早期唤醒中断使能）
  - 位 8:6：WDGTB[2:0]（预分频器：000=1，001=2，010=4，011=8）
  - 位 6:0：W[6:0]（窗口值，典型 0x7F ~ 0x50，须 > 0x3F）

- **WWDG_SR**（状态寄存器，偏移 0x08，32 位，R/W（写 1 清零））
  - 位 0：EWIF（早期唤醒中断标志）

**HAL API 映射**：

```c
WWDG_HandleTypeDef hwwdg;

hwwdg.Instance = WWDG;
hwwdg.Init.Prescaler = WWDG_PRESCALER_8;     // 分频 8
hwwdg.Init.Window    = 0x50;                 // 窗口值，必须 > 0x3F
hwwdg.Init.Counter   = 0x7F;                 // 初始计数器值，必须 > Window
hwwdg.Init.EWIMode   = WWDG_EWI_ENABLE;      // 使能 EWI

HAL_WWDG_Init(&hwwdg);  // 设置 CFR、CR，启动计数
```

刷新：
```c
HAL_WWDG_Refresh(&hwwdg);  // 写 CR = 0x80 | Counter（实际 HAL 会处理）
```

EWI 中断：
```c
void HAL_WWDG_EarlyWakeupCallback(WWDG_HandleTypeDef* hwwdg)
{
    HAL_WWDG_Refresh(hwwdg);  // 在 EWI 中及时喂狗
}
```

## 4. 高级特性 (Advanced Features)

### 早期唤醒中断（EWI）

- 必须在使能 WWDG 前配置并使能 NVIC
- 推荐：在 EWI 回调中立即刷新，避免超时

### 软件禁用

- 清除 CR.WDGA 位即可停止（但需在计数器允许刷新时）

## 5. 标准配置流程 (Configuration Workflow)

1. **开启 WWDG 时钟**
   ```c
   __HAL_RCC_WWDG_CLK_ENABLE();
   ```

2. **配置并启动 WWDG**
   ```c
   WWDG_HandleTypeDef hwwdg = {0};
   hwwdg.Instance = WWDG;
   hwwdg.Init.Prescaler = WWDG_PRESCALER_4;
   hwwdg.Init.Window    = 0x60;   // 窗口值
   hwwdg.Init.Counter   = 0x7F;   // 初始值 > Window
   hwwdg.Init.EWIMode   = WWDG_EWI_ENABLE;

   HAL_WWDG_Init(&hwwdg);
   ```

3. **使能 EWI 中断（推荐）**
   ```c
   HAL_NVIC_SetPriority(WWDG_IRQn, 2, 0);
   HAL_NVIC_EnableIRQ(WWDG_IRQn);
   ```

4. **在 EWI 回调中喂狗**
   ```c
   void HAL_WWDG_EarlyWakeupCallback(WWDG_HandleTypeDef* hwwdg)
   {
       HAL_WWDG_Refresh(hwwdg);
   }
   ```

## 6. 常见避坑与调试指南 (Pitfalls & Troubleshooting)

### 常见问题

- **窗口值设置不当**：Window ≥ Counter 或 ≤ 0x3F 导致立即复位
- **过早刷新**：主循环太快，在窗口外刷新 → 复位
- **未使用 EWI**：主循环稍慢就超时复位
- **调试时频繁复位**：调试器暂停时计数器继续 → 需在 EWI 中加断点或临时禁用
- **PCLK1 频率变化**：进入低功耗模式可能影响超时时间

### 调试技巧

- 先禁用 EWI，设置较大窗口测试
- 在 EWI 回调加 LED 翻转观察是否进入
- 使用 CubeMonitor 查看复位原因（WWDG_RESET）
- 计算超时时考虑 PCLK1 实际频率

## 7. 示例代码 (Example Code)

### 示例：基本 WWDG 配置（约 20ms 超时，PCLK1=42MHz）

```c
void WWDG_Config(void)
{
    __HAL_RCC_WWDG_CLK_ENABLE();

    WWDG_HandleTypeDef hwwdg = {0};
    hwwdg.Instance = WWDG;
    hwwdg.Init.Prescaler = WWDG_PRESCALER_8;
    hwwdg.Init.Window    = 0x50;
    hwwdg.Init.Counter   = 0x7F;
    hwwdg.Init.EWIMode   = WWDG_EWI_ENABLE;

    if (HAL_WWDG_Init(&hwwdg) != HAL_OK)
    {
        Error_Handler();
    }

    HAL_NVIC_SetPriority(WWDG_IRQn, 2, 0);
    HAL_NVIC_EnableIRQ(WWDG_IRQn);
}

void HAL_WWDG_EarlyWakeupCallback(WWDG_HandleTypeDef* hwwdg)
{
    HAL_WWDG_Refresh(hwwdg);
    // 可加其他处理，如 LED 指示正常运行
}

void WWDG_IRQHandler(void)
{
    HAL_WWDG_IRQHandler(&hwwdg);
}
```

> **最后更新**：2026-01-28  
> 本笔记适用于 STM32F4/F7/H7 系列，内容来源于官方参考手册 + 实战经验总结。
</DOCUMENT>