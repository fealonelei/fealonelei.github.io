---
layout: post
title: 为 stm32 设计一个实用的日志输出系统
category: 开发
tags: stm32, Cortex-M3, Logger, 设计模式
keywords: stm32H7, Cortex-M3, Logger, 设计模式
description:
---

引言：

**不论用什么编程语言，调试过程中输出日志对提高 debug 效率都是很重要的一环。**
本文将针对嵌入式系统（基于 stm32H7）如何做一个高效的日志输出系统做出探讨。

- [基本的日志 IO 输出](#基本的日志-io-输出)
  - [嵌入式开发过程中日志输出](#嵌入式开发过程中日志输出)
    - [1. retarget printf 到串口；](#1-retarget-printf-到串口)
    - [2. 启用 microLib，使能 printf 功能；](#2-启用-microlib使能-printf-功能)
    - [3. 自定义 usart\_printf 函数，使日志输出到串口，（stm32H7 还可以通过 usb 虚拟串口功能定义 usb\_printf 达到相同(甚至更好)的效果）](#3-自定义-usart_printf-函数使日志输出到串口stm32h7-还可以通过-usb-虚拟串口功能定义-usb_printf-达到相同甚至更好的效果)
- [日志输出方式存在的问题](#日志输出方式存在的问题)
  - [理解半主机(semihosting)模式](#理解半主机semihosting模式)
  - [1. retarget printf 到串口的问题](#1-retarget-printf-到串口的问题)
  - [2. 启用 microLib的问题](#2-启用-microlib的问题)
  - [3. 自定义 usart\_printf/usb\_printf 函数，使日志输出到串口](#3-自定义-usart_printfusb_printf-函数使日志输出到串口)
- [设计日志输出 API](#设计日志输出-api)
  - [单一工程或主工程 Log API 设计](#单一工程或主工程-log-api-设计)
  - [Framework 或第三方组件的 Log API 设计](#framework-或第三方组件的-log-api-设计)
- [总结](#总结)


## 基本的日志 IO 输出
对于软件程序，日志 IO 输出分为两种：
- 开发调试过程中日志输出到控制台或其他可视化工具; 
- 最终产品中日志输出到存储或者网络；

### 嵌入式开发过程中日志输出
最常见的输出方式为通过串口把日志内容输出到串口助手；

#### 1. retarget printf 到串口；

```c

#if (__ARMCC_VERSION >= 6010050)            /* 使用AC6编译器时 */
__asm(".global __use_no_semihosting\n\t");  /* 声明不使用半主机模式 */
__asm(".global __ARM_use_no_argv \n\t");    /* AC6下需要声明main函数为无参数格式，否则部分例程可能出现半主机模式 */

#else
/* 使用AC5编译器时, 要在这里定义__FILE 和 不使用半主机模式 */
#pragma import(__use_no_semihosting)

struct __FILE
{
    int handle;
    /* Whatever you require here. If the only file you are using is */
    /* standard output using printf() for debugging, no file handling */
    /* is required. */
};

#endif

/* 不使用半主机模式，至少需要重定义_ttywrch\_sys_exit\_sys_command_string函数,以同时兼容AC6和AC5模式 */
int _ttywrch(int ch)
{
    ch = ch;
    return ch;
}

/* 定义_sys_exit()以避免使用半主机模式 */
void _sys_exit(int x)
{
    x = x;
}

char *_sys_command_string(char *cmd, int len)
{
    return NULL;
}

/* FILE 在 stdio.h里面定义. */
FILE __stdout;

/* 重定义fputc函数, printf函数最终会通过调用fputc输出字符串到串口 */
int fputc(int ch, FILE *f)
{
    while ((USART_UX->ISR & 0X40) == 0);    /* 等待上一个字符发送完成 */

    USART_UX->TDR = (uint8_t)ch;            /* 将要发送的字符 ch 写入到DR寄存器 */
    return ch;
}

```

#### 2. 启用 microLib，使能 printf 功能；
![keil_using_microLib](/assets/image/keil_using_microLib.png)

#### 3. 自定义 usart_printf 函数，使日志输出到串口，（stm32H7 还可以通过 usb 虚拟串口功能定义 usb_printf 达到相同(甚至更好)的效果）

```c

void usart_printf(USART_TypeDef* USARTx, const char *format, ...) {
    va_list list;
    va_start(list, format);
    int len = vsnprintf(0, 0, format, list);
    char *s; 
    s = (char *)malloc(len + 1);
    vsprintf(s, format, list);
    USART_puts(USARTx,s);
    free(s);
    va_end(list);
    return;
}

void USART_putc(USART_TypeDef* USARTx, char c){
    while(USART_GetFlagStatus(USARTx, USART_FLAG_TXE) == RESET);
    USART_SendData(USARTx,c);
}

void USART_puts(USART_TypeDef* USARTx, const char *s){
    int i;
    for(i=0;s[i]!='\0';i++) USART_putc(USARTx,s[i]);
}

```

## 日志输出方式存在的问题

### 理解半主机(semihosting)模式
半主机(semihosting)是一种机制，它使 ARM 目标上运行的代码能够与运行调试器的主机进行通信并使用主机上的输入/输出功能。
例如，可以在半主机模式让 C 库中的函数（如 printf() 和 scanf()）使用主机的屏幕和键盘，而不是在目标系统上的屏幕和键盘（因为目标板子上可能就没有键盘、屏幕这些外设）

![arm_semi_hosting](/assets/image/arm_semi_hosting.png)

stm32 默认启用半主机模式，代码中如果调用了 printf, 程序下载到板子后，程序是无法运行的。因此需要前述的几种方式处理 IO 输出。但是前述几种方式各有一些问题：

### 1. retarget printf 到串口的问题
**1.1 如下代码可以关闭半主机模式，关闭后无法编译**

```c
#if (__ARMCC_VERSION >= 6010050)            /* 使用AC6编译器时 */
__asm(".global __use_no_semihosting\n\t");  /* 声明不使用半主机模式 */
__asm(".global __ARM_use_no_argv \n\t");    /* AC6下需要声明main函数为无参数格式，否则部分例程可能出现半主机模式 */

#else
/* 使用AC5编译器时, 要在这里定义__FILE 和 不使用半主机模式 */
#pragma import(__use_no_semihosting)
```

关闭后将无法编译，报错 **Error: L6915E: Library reports error: __use_no_semihosting was requested, but _ttywrch was referenced** , 这就是为什么要 [1. retarget printf 到串口；](#1-retarget-printf-到串口) 重写 `_ttywrch` `_sys_exit` `_sys_command_string` 函数。

**1.2 可能遇到 Error: L6200E: Symbol __stdout Multiply Defined**

根据 [Arm 文档](https://developer.arm.com/documentation/ka003082/latest/)，使用 retarget 的方式可以让开发者使用 printf 等简单 IO 函数，但是当再使用 `fprintf, assert, fopen, fclose, `等复杂的 IO 函数时就会遇到上述错误。我们遇到此问题是在移植 zbar 到 stm32H7 时，zbar 有使用到 fprintf / snprintf 等函数。Arm 文档给出的解决方式是移除（复杂 IO 函数）的调用。显然，这种解决方案绝大多数情况下是不可接受的。

### 2. 启用 microLib的问题

根据 [Keil 文档](https://www.keil.com/arm/microlib.asp)，MicroLib和标准C库之间的主要区别是:

> 1、MicroLib是专为深度嵌入式应用程序而设计的。<br />
> 2、MicroLib经过优化，比使用ARM标准库使用更少的代码和数据内存。<br />
> 3、MicroLib被设计成在没有操作系统的情况下工作，但是这并不妨碍它与任何操作系统或RTOS一起使用，如Keil RTX <br />
> 4、MicroLib不包含文件I/O或宽字符支持。<br />
> 5、由于MicroLib已经优化到最小化代码大小，一些函数将比ARM编译工具中可用的标准C库例程执行得更慢。<br />

MDK 勾选 Use MicroLIB 以后就可以使用 printf ，sprintf 等函数了，但是 microLib 默认向 UART1 发送数据，如果想向其他串口发送数据需要做额外的工作。其次，microLib 确实优化了 hex 文件大小，但是对于整个工程来说效果有限。同时降低了运行速度，**这对于使用 stm32H7 系列芯片的产品来说，时间换空间并不划算。因为 stm32H7 内存资源丰富，并且可以通过外挂 SDRAM 扩展空间。** 如果以后有移植需求，使用 microLib 还可能带来额外的工作量。

### 3. 自定义 usart_printf/usb_printf 函数，使日志输出到串口

使用自定义 printf 函数，能够有效避免重写半主机模式和使用 microLib 存在的问题，并提供**最大的灵活性**。

但是，自定义输出的方式比较挑战人的开发习惯，有人更倾向于使用 printf 等函数，也可能工程已经大量存在使用 printf 函数；其次，如果使用 usb_printf usb 虚拟串口功能，移植到其他 mcu 平台时可能需要二次修改。

## 设计日志输出 API

### 单一工程或主工程 Log API 设计

我们根据 STMicroelectronics Demo 总结，常见的设计方式为

```c

// 定义日志级别
#define LOGQUIET 0
#define	LOGERR	  1
#define	LOGWARN  2
#define	LOGINFO  3
#define	LOGDBG   4

// 定义默认使用的日志级别
#ifndef LOGLEVEL
#define LOGLEVEL LOGINFO
#endif

// 根据所使用的日志级别，决定相应的 log_xxx 宏定义，这里是 ST 提供的 Demo ，
// 实际上我们可以根据需要，定义一个如 logging_format_printf 函数，输出时间、文件、函数、行号等信息
#if LOGLEVEL >= LOGDBG
#define log_dbg(fmt, ...)  printf("[%05ld.%03ld][DBG  ]" fmt, HAL_GetTick()/1000, HAL_GetTick() % 1000, ##__VA_ARGS__)
#else
#define log_dbg(fmt, ...)
#endif
#if LOGLEVEL >= LOGINFO
#define log_info(fmt, ...) printf("[%05ld.%03ld][INFO ]" fmt, HAL_GetTick()/1000, HAL_GetTick() % 1000, ##__VA_ARGS__)
#else
#define log_info(fmt, ...)
#endif
#if LOGLEVEL >= LOGWARN
#define log_warn(fmt, ...) printf("[%05ld.%03ld][WARN ]" fmt, HAL_GetTick()/1000, HAL_GetTick() % 1000, ##__VA_ARGS__)
#else
#define log_warn(fmt, ...)
#endif
#if LOGLEVEL >= LOGERR
#define log_err(fmt, ...)  printf("[%05ld.%03ld][ERR  ]" fmt, HAL_GetTick()/1000, HAL_GetTick() % 1000, ##__VA_ARGS__)
#else
#define log_err(fmt, ...)
#endif

```

通过以上设计，可以确保输出定义级别以上的日志的输出。并且如果自定义 `logging_format_printf` 函数，在补充必要的信息的同时，能够通过调用 `usart_printf` / `usb_printf` 灵活进行输出。

### Framework 或第三方组件的 Log API 设计
一个第三方组件可能用于多个工程，因此**需要保证组件和工程解耦**。通常需要
> 1, 提供一个函数定义，并定义函数变量，<br/>
> 2, 由外部提供实现，并调用组件内的注册方法使得函数变量指向外部的实现，<br/>
这样在调用该函数变量时，就实现了向外输出日志。

以 LVGL 的实现为例（LVGL 的实现非常典型，非常实用）:

```c
/**
 * 提供函数定义
 */
typedef void (*lv_log_print_g_cb_t)(const char * buf);

// 定义函数变量
static lv_log_print_g_cb_t custom_print_cb;

// 组件内注册函数函数，
void lv_log_register_print_cb(lv_log_print_g_cb_t print_cb)
{
    custom_print_cb = print_cb;
}

// 外部实现函数
static void lvgl_log_cb(const char * buf)
{
    // 接入外部日志输出函数
    SdkLog( "LVGL", buf );
}

//********************************//
void some_proper_function()
{
    /// 在适当位置调用 lv_log_register_print_cb，
    /// custom_print_cb 将指向外部实现函数 lvgl_log_cb 
    /// 从而实现向外部输出日志
    lv_log_register_print_cb(lvgl_log_cb);
}

```

[LVGL lv_log.h](https://github.com/lvgl/lvgl/blob/v8.3.10/src/misc/lv_log.h) 包含了 LVGL 日志相关的功能函数，`LV_LOG_INFO` `LV_LOG_WARN` `LV_LOG_ERROR` 均调用 

```c
/**
 * Add a log
 * @param level the level of log. (From `lv_log_level_t` enum)
 * @param file name of the file when the log added
 * @param line line number in the source code where the log added
 * @param func name of the function when the log added
 * @param format printf-like format string
 * @param ... parameters for `format`
 */
void _lv_log_add(lv_log_level_t level, const char * file, int line, const char * func, const char * format, ...)
{
    // 按照需求往 buf 格式化输出字符串
    ......
    .....

    // 最后调用 custom_print_cb 向外输出日志，实现干净的解耦
    custom_print_cb(buf);
}
```

实际上，LVGL 所使用的日志输出模式和 Java Spring 开发中经常提到的`依赖注入` 有点像。

LVGL 定义的函数变量相当于一个接口，外部注入实现。实现接口和实现的分离，实现解耦，方便复用，从而大幅度提高软件开发效率。

## 总结
通过实现高效的 IO 输出和合理的日志 API ，将极大地提高嵌入式开发调试效率，节省大量的时间。

