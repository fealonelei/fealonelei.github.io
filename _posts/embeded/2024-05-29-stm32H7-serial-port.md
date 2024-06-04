---
layout: post
title: stm32H7 UART HAL 代码分析
category: 开发
tags: stm32H7
keywords: stm32H7, 串口, SerialPort, DMA
description:
---

引言 **串口，最常用的调试工具**
stm32H7 UART/USART 功能丰富，本文只对 stm32H7 串口的使用及 HAL 代码进行浅浅的分析。

## stm32H7 串口基本功能使用

#### 1. 配置 UART (通过 CubeIDE 生成省时省力)：

> 1. 配置串口参数；
  - 串口参数：波特率、数据位、停止位、校验位；
```c {.line-numbers}
    UART_HandleTypeDef huart3;
    HAL_UART_Init(&huart3);

```
> 2. 配置串口 Msp；

```c {.line-numbers}
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
```C
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

```c {.line-numbers}
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
```C
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
