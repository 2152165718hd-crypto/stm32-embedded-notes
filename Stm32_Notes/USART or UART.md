<DOCUMENT filename="USART.md">
# USART/UART 通用同步异步收发器详解 (STM32F4/F7/H7 系列 · HAL 库)

> **标签**：#STM32 #USART #UART #串口 #HAL库 #嵌入式 #外设 #通信

> **关联笔记**：[[STM32-GPIO]]、[[STM32-NVIC]]、[[STM32-DMA]]、[[STM32-RCC]]、[[STM32-时钟系统]]

## 1. 外设概述与应用场景 (Introduction & Use Cases)

### 功能简介

USART（Universal Synchronous/Asynchronous Receiver/Transmitter）是 STM32 最常用的串行通信外设，支持全双工异步 UART、同步模式、IrDA、LIN、智能卡等多种协议。最常用的是异步 UART 模式，用于与 PC、模块（如 GPS、Bluetooth、ESP8266）通信。

### 主要特性（STM32F4/F7/H7 系列）

- 多个 USART/UART 实例（F4 通常 6 个 USART + 2 个 UART，H7 更多）
- 支持异步、单线半双工、同步、多处理器、IrDA、LIN 等模式
- 波特率最高可达 10+ Mbps（视时钟和过采样而定）
- 数据位：7/8/9 位
- 停止位：0.5/1/1.5/2 位
- 奇偶校验：无/偶/奇
- 硬件流控制（CTS/RTS）
- 支持中断、DMA 传输
- 接收超时、噪声检测、帧错误等高级功能
- 低功耗模式支持（唤醒功能）

### 典型应用场景

- 与 PC 通信（调试打印、AT 指令）
- 与外设模块通信（WiFi、GSM、传感器）
- 调试输出（printf 重定向）
- 多节点通信（LIN 总线）
- 同步模式下连接 SPI 设备

## 2. 硬件原理与内部框图 (Hardware Architecture)

### 内部框图解析

USART 主要由以下部分组成：

1. **时钟来源**
   - 异步模式：APB 时钟（USART1/6 在 APB2，其他在 APB1）
   - 波特率发生器：16 倍或 8 倍过采样

2. **发送器**
   - TX 移位寄存器、启动位/数据位/校验位/停止位生成

3. **接收器**
   - RX 采样（16x 或 8x 过采样）、噪声过滤、错误检测（帧错误、过载、奇偶）

4. **中断/状态控制**
   - 多达 10+ 个中断源（TXE、TC、RXNE、IDLE、ORE 等）

数据流向：
- 发送：CPU/DMA → TDR → 发送移位寄存器 → TX 引脚
- 接收：RX 引脚 → 接收移位寄存器 → RDR → CPU/DMA

### 时钟树关联

- USART1/6/ 等挂载 APB2，其他挂 APB1
- 时钟使能宏：
  - `__HAL_RCC_USART1_CLK_ENABLE()`
  - `__HAL_RCC_USART2_CLK_ENABLE()` 等

### 电气特性

- 波特率计算（16 倍过采样）：
  ```
  BaudRate = f_APB / (8 × (2 - OVER8) × USARTDIV)
  ```
- 引脚：TX/RX（复用）、CK（同步时钟）、CTS/RTS（流控）
- 大部分引脚 5V-tolerant

## 3. 核心机制：API 与寄存器映射 (Core Mapping: API vs Registers)

### 关键寄存器（以 USART1 为例）

- **USART_CR1**（控制寄存器 1）：使能 USART（UE）、TX/RX 使能、字长、奇偶校验、过采样等
- **USART_CR2**（控制寄存器 2）：停止位、LIN 模式等
- **USART_CR3**（控制寄存器 3）：DMA 使能、流控、半双工等
- **USART_BRR**（波特率寄存器）：设置分频系数
- **USART_ISR**（中断和状态寄存器，只读）：TXE、TC、RXNE、ORE、IDLE 等标志
- **USART_ICR**（中断标志清除寄存器）：写 1 清标志
- **USART_TDR**（发送数据寄存器）：写数据触发发送
- **USART_RDR**（接收数据寄存器）：读接收数据

**HAL API 映射**：

```c
UART_HandleTypeDef huart;

huart.Instance = USART1;
huart.Init.BaudRate     = 115200;
huart.Init.WordLength   = UART_WORDLENGTH_8B;
huart.Init.StopBits     = UART_STOPBITS_1;
huart.Init.Parity       = UART_PARITY_NONE;
huart.Init.Mode         = UART_MODE_TX_RX;
huart.Init.HwFlowCtl    = UART_HWCONTROL_NONE;
huart.Init.OverSampling = UART_OVERSAMPLING_16;

HAL_UART_Init(&huart);
```

轮询模式：
```c
HAL_UART_Transmit(&huart, (uint8_t*)"Hello\r\n", 7, HAL_MAX_DELAY);
HAL_UART_Receive(&huart, buf, 10, HAL_MAX_DELAY);
```

中断模式：
```c
HAL_UART_Receive_IT(&huart, buf, 10);
// 回调
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    // 处理接收完成
}
```

DMA 模式：
```c
HAL_UART_Receive_DMA(&huart, buf, 10);
```

## 4. 高级特性 (Advanced Features)

### 中断与 DMA

- 支持多种中断（TX 空、发送完成、接收非空、空闲线等）
- DMA 支持发送和接收，降低 CPU 负载
- IDLE 线中断：检测一帧结束（常用变长协议）

### 硬件流控制

- CTS/RTS 自动流控

### 低功耗模式

- 支持从 Stop 模式唤醒（Wakeup from Stop）

### printf 重定向

```c
int fputc(int ch, FILE *f)
{
    HAL_UART_Transmit(&huart1, (uint8_t *)&ch, 1, HAL_MAX_DELAY);
    return ch;
}
```

## 5. 标准配置流程 (Configuration Workflow)

1. **开启时钟 + 配置 GPIO**
   ```c
   __HAL_RCC_USART1_CLK_ENABLE();
   __HAL_RCC_GPIOA_CLK_ENABLE();

   GPIO_InitTypeDef GPIO_InitStruct = {0};
   GPIO_InitStruct.Pin       = GPIO_PIN_9 | GPIO_PIN_10;  // PA9 TX, PA10 RX
   GPIO_InitStruct.Mode      = GPIO_MODE_AF_PP;
   GPIO_InitStruct.Pull      = GPIO_NOPULL;
   GPIO_InitStruct.Speed     = GPIO_SPEED_FREQ_VERY_HIGH;
   GPIO_InitStruct.Alternate = GPIO_AF7_USART1;
   HAL_GPIO_Init(GPIOA, &GPIO_InitStruct);
   ```

2. **配置 USART**
   ```c
   UART_HandleTypeDef huart1 = {0};
   huart1.Instance          = USART1;
   huart1.Init.BaudRate     = 115200;
   huart1.Init.WordLength   = UART_WORDLENGTH_8B;
   huart1.Init.StopBits     = UART_STOPBITS_1;
   huart1.Init.Parity       = UART_PARITY_NONE;
   huart1.Init.Mode         = UART_MODE_TX_RX;
   huart1.Init.HwFlowCtl    = UART_HWCONTROL_NONE;
   huart1.Init.OverSampling = UART_OVERSAMPLING_16;
   HAL_UART_Init(&huart1);
   ```

3. **配置 NVIC（中断模式）**
   ```c
   HAL_NVIC_SetPriority(USART1_IRQn, 2, 0);
   HAL_NVIC_EnableIRQ(USART1_IRQn);
   ```

## 6. 常见避坑与调试指南 (Pitfalls & Troubleshooting)

### 常见问题

- **波特率不匹配**：计算错误或时钟源不对
- **GPIO 复用配置错误**：查 Datasheet 确认 AF 编号
- **中断模式未清除标志**：HAL 会自动处理，但自定义中断需注意
- **接收溢出（ORE）**：未及时读取 RDR
- **DMA 配置冲突**：多个外设共用 DMA 通道
- **半双工模式引脚冲突**：单线模式需特殊配置

### 调试技巧

- 用示波器/逻辑分析仪观察 TX/RX 波形
- 查看 ISR 寄存器判断错误类型（ORE/FE/NE）
- printf 调试时确保波特率一致
- CubeMX 生成代码作为参考
- 使用 HAL_UART_ErrorCallback() 捕获错误

## 7. 示例代码 (Example Code)

### 示例 1：轮询模式收发

```c
uint8_t tx_buf[] = "Hello STM32\r\n";
uint8_t rx_buf[10];

HAL_UART_Transmit(&huart1, tx_buf, sizeof(tx_buf)-1, HAL_MAX_DELAY);
HAL_UART_Receive(&huart1, rx_buf, 10, HAL_MAX_DELAY);
```

### 示例 2：中断模式回环

```c
uint8_t rx_byte;

void USART1_Config_IT(void)
{
    HAL_UART_Receive_IT(&huart1, &rx_byte, 1);
}

void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        HAL_UART_Transmit(huart, &rx_byte, 1, HAL_MAX_DELAY);  // 回传
        HAL_UART_Receive_IT(huart, &rx_byte, 1);              // 重新使能
    }
}
```

> **最后更新**：2026-01-28  
> 本笔记适用于 STM32F4/F7/H7 系列，内容来源于官方参考手册 + 实战经验总结。
</DOCUMENT>