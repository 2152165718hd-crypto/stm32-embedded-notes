<DOCUMENT filename="IWDG.md">
# IWDG 独立看门狗详解 (STM32F4/F7/H7 系列 · HAL 库)

> **标签**：#STM32 #IWDG #看门狗 #HAL库 #嵌入式 #外设 #可靠性

> **关联笔记**：[[STM32-时钟系统]]、[[STM32-RCC]]、[[STM32-低功耗模式]]、[[STM32-WWDG]]

## 1. 外设概述与应用场景 (Introduction & Use Cases)

### 功能简介

IWDG（Independent Watchdog，独立看门狗）是一个独立的硬件定时器，用于监控系统运行状态。当软件运行异常（如死循环、程序跑飞）导致在规定时间内未“喂狗”（刷新看门狗），IWDG 将产生复位信号，重启整个芯片。

### 主要特性（STM32F4/F7/H7 系列）

- 时钟来源：独立的 **LSI**（低速内部 RC，约 32 kHz，精度较差 ±50%）
- 12 位递减计数器
- 超时周期可配置：从 ~1 ms 到 ~32 s（取决于分频器和重载值）
- **一旦使能，无法通过软件关闭**（只能通过硬件复位或上电关闭）
- 支持窗口模式（部分系列支持早期警告中断）
- 支持在 Stop/Standby 低功耗模式下继续运行（可配置）
- 硬件看门狗与软件看门狗相比更可靠（独立时钟）

### 典型应用场景

- 防止程序死循环或跑飞导致系统失控
- 工业控制、医疗设备、汽车电子等高可靠性场景
- 关键任务系统中作为最后一道安全防线
- 与 WWDG（窗口看门狗）配合使用，实现更精细监控

## 2. 硬件原理与内部框图 (Hardware Architecture)

### 内部框图解析

IWDG 结构相对简单：

1. **时钟来源**
   - 专用 LSI RC 振荡器（~32 kHz）
   - 通过 RCC 独立使能

2. **预分频器**
   - 4~256 分频（2^n，n=2~8）

3. **12 位递减计数器**
   - 重载值（RLR）可配置 0~0xFFF
   - 计数到 0 时产生复位

4. **关键寄存器保护**
   - 写保护：需先写访问键 0x5555 才能修改 PR 和 RLR
   - 刷新：写 0xAAAA 到 KR

数据流向：
- LSI → 预分频器 → 12 位计数器 → 计数到 0 → 系统复位

### 时钟树关联

- LSI 时钟由 RCC 独立控制
- 使能宏：`__HAL_RCC_LSI_ENABLE()`
- IWDG 本身无需 AHB/APB 时钟

### 电气特性

- 超时时间计算公式：
  ```
  Timeout (s) = (Prescaler / LSI_freq) × (Reload + 1)
  ```
  - Prescaler = 4 × 2^PR[2:0]
  - LSI_freq ≈ 32 kHz（实际范围 17~47 kHz）
- 实际超时时间需考虑 LSI 偏差，建议留 50% 裕量

## 3. 核心机制：API 与寄存器映射 (Core Mapping: API vs Registers)

### 关键寄存器（基地址 0x4000 3000）

- **IWDG_KR**（键寄存器，偏移 0x00，32 位，只写）
  - 0xCCCC：启动看门狗（使能计数）
  - 0xAAAA：刷新计数器（喂狗）
  - 0x5555：解除写保护（允许修改 PR/RLR）
  - 其他值无效

- **IWDG_PR**（预分频器寄存器，偏移 0x04，32 位，R/W，写保护）
  - 位 2:0：分频系数
    - 000：4
    - 001：8
    - ...
    - 111：256

- **IWDG_RLR**（重载寄存器，偏移 0x08，32 位，R/W，写保护）
  - 位 11:0：重载值（0~0xFFF），计数从该值开始递减

- **IWDG_SR**（状态寄存器，偏移 0x0C，32 位，只读）
  - 位 0：PVU（预分频器更新中）
  - 位 1：RVU（重载值更新中）
  - 位 2：WVU（窗口更新中，H7 系列特有）

**HAL API 映射**：

HAL 库提供结构体配置方式：

```c
IWDG_HandleTypeDef hiwdg;

hiwdg.Instance = IWDG;
hiwdg.Init.Prescaler = IWDG_PRESCALER_256;  // 分频 256
hiwdg.Init.Reload    = 4095;                // 重载值 0xFFF
hiwdg.Init.Window    = IWDG_WINDOW_DISABLE; // F4 无窗口模式

HAL_IWDG_Init(&hiwdg);  // 内部写 0xCCCC 启动
```

刷新（喂狗）：
```c
HAL_IWDG_Refresh(&hiwdg);  // 写 0xAAAA
```

## 4. 高级特性 (Advanced Features)

### 窗口模式（部分系列支持，如 H7）

- 可设置窗口值，只有在计数器大于窗口值时才能刷新
- 防止过早或过晚喂狗

### 低功耗模式支持

- 可配置在 Stop/Standby 下继续运行
- HAL 通过选项字节或寄存器控制

### 硬件启动（选项字节）

- 可通过选项字节设置上电自动启动 IWDG（更安全）

## 5. 标准配置流程 (Configuration Workflow)

1. **开启 LSI 时钟**（可选，HAL 会自动处理）
   ```c
   __HAL_RCC_LSI_ENABLE();
   while (__HAL_RCC_GET_FLAG(RCC_FLAG_LSIRDY) == RESET) {}
   ```

2. **配置并启动 IWDG**
   ```c
   IWDG_HandleTypeDef hiwdg = {0};
   hiwdg.Instance = IWDG;
   hiwdg.Init.Prescaler = IWDG_PRESCALER_64;
   hiwdg.Init.Reload    = 500;   // 约 1s 超时（粗算）
   HAL_IWDG_Init(&hiwdg);
   ```

3. **正常运行中定期喂狗**
   ```c
   HAL_IWDG_Refresh(&hiwdg);
   ```

注意：HAL_IWDG_Init() 会等待 PVU/RVU 清零，确保配置生效。

## 6. 常见避坑与调试指南 (Pitfalls & Troubleshooting)

### 常见问题

- **LSI 频率偏差导致超时时间不准**：实际测试超时时间，留足裕量
- **调试时频繁复位**：使用硬件看门狗时，调试器需“Disable Watchdog”或在喂狗循环中单步
- **忘记喂狗**：主循环或任务中必须定期刷新
- **窗口模式误用**：过早刷新导致复位
- **低功耗模式下意外复位**：检查是否允许 IWDG 运行

### 调试技巧

- 计算超时时间时考虑 LSI 最慢情况（17 kHz）
- 使用 STM32CubeMonitor 查看复位原因（IWDG_RESET 标志）
- 临时注释喂狗代码观察是否复位
- 在 Keil/CubeIDE 中设置“Run to main”并快速喂狗

## 7. 示例代码 (Example Code)

### 示例：基本 IWDG 配置（约 4s 超时）

```c
void IWDG_Config(void)
{
    IWDG_HandleTypeDef hiwdg = {0};

    hiwdg.Instance = IWDG;
    hiwdg.Init.Prescaler = IWDG_PRESCALER_256;  // 分频 256
    hiwdg.Init.Reload    = 500;                 // (256/32000)*501 ≈ 4s

    if (HAL_IWDG_Init(&hiwdg) != HAL_OK)
    {
        Error_Handler();
    }
}

// 在主循环或RTOS任务中
while (1)
{
    HAL_IWDG_Refresh(&hiwdg);
    // 其他代码，确保刷新间隔 < 超时时间
    HAL_Delay(1000);
}
```

> **最后更新**：2026-01-28  
> 本笔记适用于 STM32F4/F7/H7 系列，内容来源于官方参考手册 + 实战经验总结。
</DOCUMENT>