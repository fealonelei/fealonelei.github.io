---
layout: post
title: stm32H7 UART 使用、HAL 代码分析和实际遇到的问题
category: 嵌入式
tags: stm32H7
keywords: stm32H7, 串口, SerialPort, DMA
description:
---

## 前言 

**串口，最常用的调试工具**

stm32H7 UART/USART 功能丰富，本文只对 stm32H7 串口的使用及 HAL 代码进行浅浅的分析。
- [前言](#前言)
- [stm32H7 串口基本功能使用](#stm32h7-串口基本功能使用)
    - [1. 配置 UART (通过 CubeIDE 生成省时省力)：](#1-配置-uart-通过-cubeide-生成省时省力)
    - [2. 收发收据：](#2-收发收据)
- [stm32 串口 DMA 使用](#stm32-串口-dma-使用)
    - [2. 收发收据：](#2-收发收据-1)
- [HAL 库 UART 代码实现浅析](#hal-库-uart-代码实现浅析)
    - [串口初始化：](#串口初始化)
    - [串口接收](#串口接收)
    - [串口中断接收：](#串口中断接收)
- [stm32H7 串口使用中遇到的问题：](#stm32h7-串口使用中遇到的问题)
    - [1. 串口如何接收不定长数据](#1-串口如何接收不定长数据)
    - [2. 当串口开启 DMA 接收后，只能接收一次数据，然后无法接收到新的数据](#2-当串口开启-dma-接收后只能接收一次数据然后无法接收到新的数据)
    - [3. STM32串口中断卡死（主程序无法运行，一直陷在 USART1\_IRQHandler 串口中断函数）](#3-stm32串口中断卡死主程序无法运行一直陷在-usart1_irqhandler-串口中断函数)

## stm32H7 串口基本功能使用

#### 1. 配置 UART (通过 CubeIDE 生成省时省力)：

> 1. 配置串口参数；
  - 串口参数：波特率、数据位、停止位、校验位；

```c
    UART_HandleTypeDef huart3;
    HAL_UART_Init(&huart3);

```

> 2. 配置串口 Msp；

```c
void HAL_UART_MspInit(UART_HandleTypeDef* huart)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  RCC_PeriphCLKInitTypeDef PeriphClkInitStruct = {0};
  if(huart->Instance==USART3)
  {
  /** Initializes the peripherals clock
  */
    PeriphClkInitStruct.PeriphClockSelection = RCC_PERIPHCLK_USART3;
    PeriphClkInitStruct.Usart234578ClockSelection = RCC_USART234578CLKSOURCE_D2PCLK1;
    if (HAL_RCCEx_PeriphCLKConfig(&PeriphClkInitStruct) != HAL_OK)
    {
      Error_Handler();
    }

    /* Peripheral clock enable */
    __HAL_RCC_USART3_CLK_ENABLE();
    __HAL_RCC_GPIOB_CLK_ENABLE();

    
    /**USART3 GPIO Configuration
    PB10     ------> USART3_TX
    PB11     ------> USART3_RX
    */
    ....

    /* USART3 interrupt Init */
    HAL_NVIC_SetPriority(USART3_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(USART3_IRQn);
  }

}
```

> 3. 开启接收中断；

``` C

    // 开启接收中断
    HAL_UART_Receive_IT(&huart3, (uint8_t *)uart_rx_buf, 1);

```

#### 2. 收发收据：
> 
> - ~~通过轮询方式收发数据；~~

> - 通过中断方式收发数据；

```c
    // 中断发送
    HAL_UART_Transmit_IT(&huart3, (uint8_t *)uart_tx_buf, 1);

    // 接收中断入口函数
    void USART3_IRQHandler(void)
    {
        // 两种处理方式
        // 1，通过公有处理逻辑，在回调函数里处理
        HAL_UART_IRQHandler(&huart3);
        // 2，通过 __HAL_UART_GET_FLAG 获取中断标志位，在回调函数里处理
        ...
    }

```

## stm32 串口 DMA 使用

> 0. 配置 DMA（如果需要）

```c
DMA_HandleTypeDef hdma_usart3_tx;
DMA_HandleTypeDef hdma_usart3_rx;
static void MX_DMA_Init(void)
{
  /* DMA controller clock enable */
  __HAL_RCC_DMA1_CLK_ENABLE();

  /* DMA interrupt init */
  /* DMA1_Stream0_IRQn interrupt configuration */
  HAL_NVIC_SetPriority(DMA1_Stream0_IRQn, 0, 0);
  HAL_NVIC_EnableIRQ(DMA1_Stream0_IRQn);
  /* DMA1_Stream1_IRQn interrupt configuration */
  HAL_NVIC_SetPriority(DMA1_Stream1_IRQn, 0, 0);
  HAL_NVIC_EnableIRQ(DMA1_Stream1_IRQn);
}
```

> 2. 配置串口 Msp；
<details> <summary>点击以展开完整的代码块</summary> 

```c
void HAL_UART_MspInit(UART_HandleTypeDef* huart)
{
  GPIO_InitTypeDef GPIO_InitStruct = {0};
  RCC_PeriphCLKInitTypeDef PeriphClkInitStruct = {0};
  if(huart->Instance==USART3)
  {
    ....
    ....

/************* 如果需要启动 DMA 功能 ↓↓ **********************/
    
    /* USART3 DMA Init */
    /* USART3_TX Init */
    hdma_usart3_tx.Instance = DMA1_Stream0;
    hdma_usart3_tx.Init.Request = DMA_REQUEST_USART3_TX;
    hdma_usart3_tx.Init.Direction = DMA_MEMORY_TO_PERIPH;
    hdma_usart3_tx.Init.PeriphInc = DMA_PINC_DISABLE;
    hdma_usart3_tx.Init.MemInc = DMA_MINC_ENABLE;
    hdma_usart3_tx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
    hdma_usart3_tx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
    hdma_usart3_tx.Init.Mode = DMA_NORMAL;
    hdma_usart3_tx.Init.Priority = DMA_PRIORITY_LOW;
    hdma_usart3_tx.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
    if (HAL_DMA_Init(&hdma_usart3_tx) != HAL_OK)
    {
      Error_Handler();
    }

    __HAL_LINKDMA(huart,hdmatx,hdma_usart3_tx);

    /* USART3_RX Init */
    hdma_usart3_rx.Instance = DMA1_Stream1;
    hdma_usart3_rx.Init.Request = DMA_REQUEST_USART3_RX;
    hdma_usart3_rx.Init.Direction = DMA_PERIPH_TO_MEMORY;
    hdma_usart3_rx.Init.PeriphInc = DMA_PINC_DISABLE;
    hdma_usart3_rx.Init.MemInc = DMA_MINC_ENABLE;
    hdma_usart3_rx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
    hdma_usart3_rx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
    hdma_usart3_rx.Init.Mode = DMA_NORMAL;
    hdma_usart3_rx.Init.Priority = DMA_PRIORITY_LOW;
    hdma_usart3_rx.Init.FIFOMode = DMA_FIFOMODE_DISABLE;
    if (HAL_DMA_Init(&hdma_usart3_rx) != HAL_OK)
    {
      Error_Handler();
    }

    __HAL_LINKDMA(huart,hdmarx,hdma_usart3_rx);

/************* 如果需要启动 DMA 功能 ↑↑ **********************/

    /* USART3 interrupt Init */
    HAL_NVIC_SetPriority(USART3_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(USART3_IRQn);
  }

}
```

</details>

#### 2. 收发收据：
> 
> - ~~通过轮询方式收发数据；~~

> - 通过 DMA 方式收发数据；

```c
    // DMA 发送
    hal_uart_dma_transmit(&huart3, (uint8_t *)uart_tx_buf, 1, 1000);

    // DMA 接收
    void DMA1_Stream2_IRQHandler(void)
    {
      HAL_DMA_IRQHandler(&hdma_usart3_rx);
    }
    // 接收中断入口函数
    void USART3_IRQHandler(void)
    {
        // 与 '通过中断方式收发数据；' 部分逻辑相同
    }
```

## HAL 库 UART 代码实现浅析

#### 串口初始化：
串口初始化时调用 `HAL_StatusTypeDef HAL_UART_Init(UART_HandleTypeDef *huart)`主要完成：
1. 参数检查；
2. 检查 USE_HAL_UART_REGISTER_CALLBACKS, 
    - 如果 USE_HAL_UART_REGISTER_CALLBACKS 为 1，则注册默认回调函数，并检查 MspInitCallback() 是否为空，是，注册默认回调函数，不是，调用相应函数；
    - 如果 use_HAL_UART_REGISTER_CALLBACKS 为 0，则直接使用默认 Msp 初始化函数 void HAL_UART_MspInit(UART_HandleTypeDef *huart) **(这就是为什么我们定义了HAL_UART_MspInit 却不需要显式调用的原因)**
3. 配置串口基础参数，（如需要）配置高阶参数；
4. 清理 USART_CR2 LINEN and CLKEN bits，清理 USART_CR3 SCEN, HDSEL and IREN  bits
5. 检查串口 Idle state 并返回最终初始化结果；

#### 串口接收
启用串口接收，有如下几个函数
- HAL_StatusTypeDef UART_Start_Receive_IT(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size)
- HAL_StatusTypeDef UART_Start_Receive_DMA(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size)

定义在 uart_ex 文件中的 
- HAL_StatusTypeDef HAL_UARTEx_ReceiveToIdle_IT(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size)
- HAL_StatusTypeDef HAL_UARTEx_ReceiveToIdle_DMA(UART_HandleTypeDef *huart, uint8_t *pData, uint16_t Size)

UART 这几个接收函数，控制 UART 接收模式 ReceptionType 

__IT 函数相对简单，写入 CR3 CR1 寄存器，启用接收中断，并根据初始化时的参数 WordLength 确定 RxISR 是 UART_RxISR_16BIT 或 UART_RxISR_8BIT. 

__DMA 函数相对复杂，首先绑定 UART 相关函数到 DMA 通道的回调函数，然后开启 DMA 接收中断（串口接收寄存器和接收数组地址绑定到 DMA 通道）

#### 串口中断接收：
在中断函数中调用 void HAL_UART_IRQHandler(UART_HandleTypeDef *huart);

HAL_UART_IRQHandler 检查 `errorflags`, 
- 如果没有错误，并且 UART 处在 Receiver mode，则直接进入 RxISR 函数；
- 如果有错误，则依次处理 parity error, frame error, noise error, overrun error, timeout；如果 UART 处在 Receiver mode, 调用 RxISR 函数；
    - 如果是 Receiver 超时、溢出(Overrun) 或者 DMA 相关的任何错误等 Blocking 错误，则 abort 发送；将 UART 状态设置为准备就绪，以便能够重新启动该过程；启用了 DMA，则禁用 Rx DMA 中断，并调用 DMA (默认或自定义) abort 回调函数或者 UART error 回调函数，没有启用 DMA，则回调 UART error (默认或自定义)回调函数；
- 如果启用接收为 uart_ex 的 ReceiveToIdle_IT 或者 ReceiveToIdle_DMA，还要检查 IDLE flag 状态，并触发相应的回调函数

## stm32H7 串口使用中遇到的问题：
#### 1. 串口如何接收不定长数据
- 串口接收不定长数据，可以使用 `HAL_UARTEx_ReceiveToIdle_IT` 或者 `HAL_UARTEx_ReceiveToIdle_DMA` 函数，该函数会一直接收到串口接收缓冲区为空，或者接收到 IDLE 状态。
- 开启 DMA 功能，在中断接收里处理
  
```c
void USART1_IRQHandler(void)
{
    if (__HAL_UART_GET_FLAG(&g_uart1_handle, UART_FLAG_IDLE) != RESET){           //获取接收IDLE标志位是否被置位 
        __HAL_UART_CLEAR_IDLEFLAG(&g_uart1_handle);
        g_uart1_handle.Instance->ISR;
        g_uart1_handle.Instance->RDR;
        HAL_UART_DMAStop(&g_uart1_handle);            //停止DMA传输，防止干扰
        uart1_rx_len = USART_RX_BUFFER_SIZE - __HAL_DMA_GET_COUNTER(&hdma_usart1_rx);  //获取接收到的数据长度
        LogInfo("recv: %s, recv_len: %d\r\n", g_rx_buffer, uart1_rx_len);
    }
}

```

  ***IDLE的中断标志位, 必须通过软件序列清零, 并且清理顺序不能颠倒***

```
    g_uart1_handle.Instance->ISR;
    g_uart1_handle.Instance->RDR;

```

#### 2. 当串口开启 DMA 接收后，只能接收一次数据，然后无法接收到新的数据

不管通过 st 文档或者在线搜索，通过 HAL 开启串口 DMA 接收是比较简单的。但是我们使用 USART1_IRQHandler 中断接收处理数据时，只能接收一次数据，后续无法再获取数据。

出现这一问题的原因是：

  对于运行在具有可缓存内存区域的微控制器 (MCU) 上的应用程序，缓存一致性问题不可避免，这些应用程序使用直接内存访问 (DMA) 进行数据传输操作。这是因为 CPU 从缓存执行读/写操作，而 DMA 在外设和物理内存之间传输数据。解决缓存一致性的方法之一是创建一个一致性或非可缓存的内存区域，并将数据变量置于争用状态。当数据保持一致时，CPU 始终从主内存 (SRAM) 访问数据。

![DMA_cache_issue](/assets/image/DMA_cache_issue.png)

简单理解就是，启用了 stm32H7 Cache 功能，stm32H7 Cache 功能和 DMA （Direct Memory Access）直接访问之间存在矛盾。<br />
解决办法就是，让 DMA 能读到正确的地址和该地址上的内容。

  - 禁用 Cache 功能 (这样做将降低总体性能，不推荐)
  
  ```
      SCB_DisableDCache();
  ```

  - 魔改 linker script file 
  - （**推荐**）调用 Arm `core_cm7.h` 提供的 API `__STATIC_FORCEINLINE void SCB_InvalidateDCache_by_Addr (void *addr, int32_t dsize)` 【实际上 [STM32CubeH7 Package](https://github.com/STMicroelectronics/STM32CubeH7) 提供的 Demo 也是采用这种方式解决缓存一致性问题】
  
  ```
      SCB_InvalidateDCache_by_Addr((uint32_t *)g_rx_buffer, USART_RX_BUFFER_SIZE);
  ```
    
#### 3. STM32串口中断卡死（主程序无法运行，一直陷在 USART1_IRQHandler 串口中断函数）

stm32H7 UART 配置正确，和外设连接都正确，但是上电后，主程序无法运行，通过 Keil 开启 debug 模式，发现程序一直进入 USART1_IRQHandler 串口中断函数，并且不会进入 HardFault_Handler ，说明程序并未出现硬件错误，而是“正常运行并一直进入串口中断函数”。

首先排除 `UART_FLAG_RXNE` `UART_IT_RXNE` 的问题，通过清理这两个标志位不能恢复正常。<br />
在 `startup_stm32h750xx.s` 扩大 Stack_Size 数值也不能解决问题。【实际上，预期从外设获取的数据量并不大，定义的 USART_RX_BUFFER_SIZE 完全够用】<br />
检查 `"our_project.map"`没有函数冲突，没有其他函数占中断函数的地址。`startup_stm32h750xx.s` 没有 USART1_IRQHandler 相关的修改。

通过查看 RM 手册 ![stm32H7_UART_RXNE](/assets/image/stm32H7_UART_RXNE.png) 得知，打开RXNEIE，默认会同时打开RXNE和ORE中断 ！<br />
而工程中使用的外设比较特殊，它在上电后就会默认一直发送数据。而上电后串口还没有初始化，当串口完成初始化后，它会因为接收寄存器中存在数据，从而产生**溢出中断**，因此程序会一直陷在 USART1_IRQHandler ，无法跳出中断。

如果通过 `void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)` 调试，会发现 ErrorCode 为 8，它对应的就是 

```
#define  HAL_UART_ERROR_ORE              ((uint32_t)0x00000008U)    /*!< Overrun error           */
```

解决方案为：
1. 禁用串口接收溢出中断，即 `__HAL_UART_DISABLE_IT(&g_uart1_handle, UART_IT_ORE)` 或者在初始化配置时加入
   
   ```
    g_uart1_handle.AdvancedInit.AdvFeatureInit = UART_ADVFEATURE_RXOVERRUNDISABLE_INIT;
    g_uart1_handle.AdvancedInit.OverrunDisable = UART_ADVFEATURE_OVERRUN_DISABLE;
   ```

2. UART 接收管脚 Msp Init 时配置为上拉
   
   ```
   gpio_init_struct.Pull = GPIO_PULLUP;
   ```

3. 【根据实际情况】配置外设上电时不发送数据（可以省电，可以解决卡死问题）；














