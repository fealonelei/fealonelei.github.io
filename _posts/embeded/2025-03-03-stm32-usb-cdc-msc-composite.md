---
layout: post
title: stm32 USB 实现 CDC-MSC 复合设备
category: 嵌入式
tags: stm32, 设计模式, Wireshark 抓包
keywords: CDC MSC Composite, USB composite, Wireshark, 设计模式, USB抓包
description:
---

## 前言

stm32 提供了功能完整的 USB 模块，并且 usb host 和 device library 都很完整，如果要使用 USB 单一功能，如 CDC/MSC/HID 等，通过 stm32CubeMX 可以轻松配置
<!-- ![stm32-cubemx-usb-device](/assets/image/stm32-cubemx-usb-device-category.png) -->
<picture>
<img src="../../assets/image/stm32-cubemx-usb-device-category.png" alt="stm32-cubemx-usb-device-category" width="560" height="320"></picture>
<br/>但是，对于 CDC-MSC 这种复合设备，stm32CubeMX 并没有提供直接生成配置的选项。而且 stm32 并没有给出参考示例。【如果有 CDC+HID / Audio+CDC 等方向的 USB 符合设备的需求，可以参考 [STM32CubeH7 Package dev/usb/composite分支](https://github.com/STMicroelectronics/STM32CubeH7/tree/dev/usb/composite/Projects/STM32H743I-EVAL/Applications/USB_Device)（太难找了，只有 st community 上有 ST employee 回复问题时提了一下，正常人谁找得到）】


## 如何实现 CDC-MSC 复合设备？
通过搜索，可以参考的资料有如下：
* 多年前 ST 官方 USB 培训资料，详细讲解了如何实现 CDC+MSC 复合设备
* 各路大神基于 ST 官方资料或者自己深厚的基础，自主实现 CDC+MSC Composite Interface
* 兼容 stm32 的厂商提供了 CDC+MSC 的实现 
* （还发现了一位俄语开发者的实现，并且他还参考了国内开发者的实现，挺有趣的）

总结这些资料，CDC-MSC Composite Interface 的实现方式大体如下:
> 1、提供正确的设备描述符、配置描述符、接口描述符；<br/>
> 2、修改端点描述符的值；<br/>
> 3、usbd_conf.c 代码也有部分修改；<br/>
> 4、将设备注册到 USB 运行代码中；

但是可参考资料都或多或少存在一些问题：<br/>资料可能较老，实现方式基于旧版的 st usb library. <br/>有的大神的实现魔改了 usb libaray CORE 文件夹下的内容，有的实现相当于做了一个自定义设备，收到 usb host 协议帧转发给对应的逻辑处理函数（这是真有实力，对 USB 协议规范有深入的研究）。<br/>兼容厂商的 usb library 或多或少还是要和 st 的有差异（不然可能有一些商业法律上的问题？？）。

因此，本文提出一种基于当前 ST USB Device Library 的 CDC-MSC Composite 实现方式，实现足够**简洁、无侵入、好理解**。

### 1. 建立工程
使用 STM32CubeMX 建立两个基于 stm32L476  独立的工程，一个是 CDC 工程，一个是 MSC 工程。<br/>然后以一个工程为母版，将 CDC Class 目录和 MSC Class 目录放在 Middlewares\ST\STM32_USB_Device_Library\Class 目录下。
将 CDC 工程 usbd_cdc_if.c/h 和 MSC 工程 usbd_storage.c/h 放在 Application 的目录下，将任一工程下的 usbd_conf.c/h usbd_desc.c/h 放在 Application 同一目录下。<br/>
**将 stm32L4 Firmware Package Middlewares/ST/STM32_USB_Device_Library/Class/CompositeBuilder 下的内容放在 Middlewares\ST\STM32_USB_Device_Library\Class 目录下。** 

### 2. 修改设备描述符、配置描述符、接口描述符
#### 2.1 设备描述符
在Github Repo提供完整的设备描述符，这里只写出关键的修改；
```c
/** USB standard device descriptor. */
__ALIGN_BEGIN uint8_t USBD_FS_DeviceDesc[USB_LEN_DEV_DESC] __ALIGN_END =
{
………………
  0xEF,                       /*bDeviceClass*/
  0x02,                       /*bDeviceSubClass*/
  0x01,                       /*bDeviceProtocol*/
………………
};
```

#### 2.2 配置描述符
too long, 在 Github Repo 中提供；**这里需要注意的是，CDC MSC 描述符的顺序要和初始化时 CDC MSC 注册顺序一致。**

#### 2.3 端点描述
在 usbd_conf.h 添加如下宏定义：
```c
/* Activate the IAD option */
#define USBD_COMPOSITE_USE_IAD                             1U

/* Activate the composite builder */
#define USE_USBD_COMPOSITE

#define USBD_CMPST_MAX_CONFDESC_SZ        106
#define USB_CMPSIT_CONFIG_DESC_SIZ        106          

/* Activate CustomHID and CDC classes in composite builder */
#define USBD_CMPSIT_ACTIVATE_MSC                           1U
#define USBD_CMPSIT_ACTIVATE_CDC                           1U

/* The definition of endpoint numbers must respect the order of classes instantiation  */
#define CDC_IN_EP                             0x81U  /* EP1 for CDC data IN */
#define CDC_OUT_EP                            0x01U  /* EP1 for CDC data OUT */
#define CDC_CMD_EP                            0x82U  /* EP2 for CDC commands */

#define MSC_EPIN_ADDR                0x83U
#define MSC_EPOUT_ADDR               0x03U
```

#### 2.4 usbd_conf.c USBD_StatusTypeDef USBD_LL_Init(USBD_HandleTypeDef *pdev) 函数需要修改
**这里给出的是错误的修改，正确的在Github Repo**

```c
USBD_StatusTypeDef USBD_LL_Init(USBD_HandleTypeDef *pdev)
{

…………
…………
  hpcd_USB_OTG_FS.Init.dev_endpoints = 6;

…………
…………

  HAL_PCDEx_SetRxFiFo(&hpcd_USB_OTG_FS, 0x80);
  HAL_PCDEx_SetTxFiFo(&hpcd_USB_OTG_FS, 0, 0x40);
  HAL_PCDEx_SetTxFiFo(&hpcd_USB_OTG_FS, 1, 0x40);
  HAL_PCDEx_SetTxFiFo(&hpcd_USB_OTG_FS, 2, 0x40);
  HAL_PCDEx_SetTxFiFo(&hpcd_USB_OTG_FS, 3, 0x40);
  }
  return USBD_OK;
}
```

#### 2.4 初始化 USB 
```c
uint8_t MSC_EpAdress[2] = {MSC_EPIN_ADDR, MSC_EPOUT_ADDR};    /* MSC Endpoint Adress */
uint8_t CDC_EpAdd_Inst1[3] = {CDC_IN_EP, CDC_OUT_EP, CDC_CMD_EP}; /* CDC Endpoint Adress First Instance */
void MX_USB_DEVICE_Init(void)
{
  /* USER CODE BEGIN USB_DEVICE_Init_PreTreatment */
  /* USER CODE END USB_DEVICE_Init_PreTreatment */

  /* Init Device Library, add supported class and start the library. */
  /* Init Device Library */
  USBD_Init(&hUsbDeviceFS, &FS_Desc, DEVICE_FS);
  
  /* Store CDC instance Class ID */
  CDC_InstID = hUsbDeviceFS.classId;
  
  /* Register CDC class First Instance */
  USBD_RegisterClassComposite(&hUsbDeviceFS, USBD_CDC_CLASS, CLASS_TYPE_CDC, CDC_EpAdd_Inst1);
  /* Add CDC Interface Class Instance */
  hUsbDeviceFS.classId--;
  USBD_CDC_RegisterInterface(&hUsbDeviceFS, &USBD_Interface_fops_FS);
  hUsbDeviceFS.classId++;
  
  /* Register MSC class First Instance */
  USBD_RegisterClassComposite(&hUsbDeviceFS, USBD_MSC_CLASS, CLASS_TYPE_MSC, MSC_EpAdress);
  /* Add MSC Interface Class Instance */
  hUsbDeviceFS.classId--;
  USBD_MSC_RegisterStorage(&hUsbDeviceFS, &USBD_DISK_fops);
  hUsbDeviceFS.classId++;
  
  /* Start Device Process */
  USBD_Start(&hUsbDeviceFS);

  /* USER CODE BEGIN USB_DEVICE_Init_PostTreatment */

  /* USER CODE END USB_DEVICE_Init_PostTreatment */
}
```
这里需要注意两点：

1, 正确注册设备端点数量和地址，CDC 需要 3 个端点，MSC 需要 2 个端点，HID 只有 1 个端点。一开始我参考 ST 提供的 CDC+HID 实现，只给 MSC 注册一个端点。结果上电就 crash <br/>

2, **为什么在 USBD_CDC_RegisterInterface() 和 USBD_MSC_RegisterStorage() 上下都有  hUsbDeviceFS.classId--; hUsbDeviceFS.classId++; ？**
这是因为 USBD_RegisterClassComposite() 函数设计上的缺陷。
```c
USBD_StatusTypeDef USBD_RegisterClassComposite(USBD_HandleTypeDef *pdev, USBD_ClassTypeDef *pclass,
                                               USBD_CompositeClassTypeDef classtype, uint8_t *EpAddr)
{

      /* Link the class to the USB Device handle */
      pdev->pClass[pdev->classId] = pclass;
      ret = USBD_OK;

      pdev->tclasslist[pdev->classId].EpAdd = EpAddr;

      /* Call the composite class builder */
      (void)USBD_CMPSIT_AddClass(pdev, pclass, classtype, 0);

      /* Increment the ClassId for the next occurrence */
      pdev->classId ++;
      pdev->NumClasses ++;
    }
```
它在调用 USBD_CMPSIT_AddClass() 函数之后 pdev->classId++ ，而 USBD_CDC_RegisterInterface() 和 USBD_MSC_RegisterStorage() 都是根据 classId 绑定数据的。
假设没有 hUsbDeviceFS.classId--; hUsbDeviceFS.classId++; 那么实际的逻辑会变成 <br/>

| CDC                                                                  | MSC                                                                 |
| --- | --- |
| pdev->pClass[0] = USBD_CDC_CLASS;                                    | pdev->pClass[1] = USBD_MSC_CLASS;                                   |
| pdev->tclasslist[0].EpAdd = CDC_EpAdd_Inst1;                         | pdev->tclasslist[1].EpAdd = MSC_EpAdress;                           |
| (void)USBD_CMPSIT_AddClass(pdev, USBD_CDC_CLASS, CLASS_TYPE_CDC, 0); | (void)USBD_CMPSIT_AddClass(pdev,USBD_MSC_CLASS, CLASS_TYPE_MSC, 0); |
| pdev->pUserData[**1**] = fops;                                       | pdev->pUserData[**2**] = fops;                                      |

<br/>显然 pUserData Index 和其他参数无法对应，这会导致程序运行之后 crash. 
所以，需要 hUsbDeviceFS.classId--; hUsbDeviceFS.classId++; 保证 pUserData Index 和其他参数对应。
这里也可以使用 stm32H7 Package 提供的实现:
```c
/* Add CDC Interface Class First Instance */
if (USBD_CMPSIT_SetClassID(&hUsbDeviceFS, CLASS_TYPE_CDC, 0) != 0xFF)
{
  USBD_CDC_RegisterInterface(&hUsbDeviceFS, &USBD_Interface_fops_FS);
}
if (USBD_CMPSIT_SetClassID(&hUsbDeviceFS, CLASS_TYPE_MSC, 0) != 0xFF)
{
  USBD_MSC_RegisterStorage(&hUsbDeviceFS, &USBD_DISK_fops);
}

```
<br/>
理论上讲，如果上述步骤都提供正确实现，那么编译运行后，USB 连接到 Windows PC 上，应该可以在设备管理器看到两个设备：MSC 和 CDC. <br/> 
如果 MSC storage 接口提供得当，那么应该可以看到 U 盘设备<br/>

然而，如 2.4 所介绍，USBD_StatusTypeDef USBD_LL_Init() 函数的修改出现了错误，导致 MSC 接口异常，连接 Windows PC 后，看到的如下：
<!-- ![usb-cdc-msc-wrong](/assets/image/usb-cdc-msc-wrong.png) <br> -->
<img src="/assets/image/usb-cdc-msc-wrong.png" alt="usb-cdc-msc-wrong" width="324" height="460">

那么，如何排查问题呢？

## 如何使用 Wireshark 排查 USB MSC 通信存在的问题
### 1. 代码分析，问题出现在哪一层？
ST USB library 接入 MSC 设备时的结构如图
<!-- ![usb-msc-st-structure](/assets/image/usbd_storage_if_for_MSC.png) -->
<img src="/assets/image/usbd_storage_if_for_MSC.png" alt="usbd_storage_if_for_MSC" width="404" height="166">

根据“2.4初始化 USB” `void MX_USB_DEVICE_Init(void)`，MSC 初始化分4步：
1. 初始化USBD内核
   `USBD_Init(&hUsbDeviceFS, &FS_Desc, DEVICE_FS);` 将 descriptor 注册给 USBD 内核；
2. 给USBD内核注册USBD_MSC类
   `USBD_RegisterClassComposite(&hUsbDeviceFS, USBD_MSC_CLASS, CLASS_TYPE_MSC, MSC_EpAdress);`初始化 MSC 类的各个字段；
3. 给USBD_MSC类注册USBD_MSC_If接口
   `USBD_MSC_RegisterStorage(&hUsbDeviceFS, &USBD_DISK_fops);`注册存储设备通信接口
4. 正式启动USBD

分析，在设备管理器看到的结果，USB 内核运行是成功的，并且正确识别了 MSC 设备，但是“USB 大容量存储设备”图标上有一个⚠，**推测应该是 SCSI 通信中存在问题，导致Windows无法获取 MSC 存储设备信息**，这里不禁倒吸一口凉气，SCSI 协议内容居多，该如何处理

### 2. Keil Debug 先确定大体范围
```c
USBD_StorageTypeDef USBD_DISK_fops = {
  STORAGE_Init,
  STORAGE_GetCapacity,
  STORAGE_IsReady,
  STORAGE_IsWriteProtected,
  STORAGE_Read,
  STORAGE_Write,
  STORAGE_GetMaxLun,
  STORAGE_Inquirydata, 
};
```
注册的USBD_MSC_If接口数量不多，可以依次打端点进行调试，结果每个接口都会调用。最重要的是**CDC+MSC复合设备 USBD_MSC_If 接口的调用情况和 MSC 设备 USBD_MSC_If 接口的调用情况一致**，显然通过 Keil 调试无法排查问题，而且 Keil 打断点，会影响 Windows 和设备之间的通信，带来更多不确定性。

### 3. Wireshark 抓包分析 
必须说一句：Wireshark 大法好。作为最强大的网络封包分析软件，它不仅可以分析 TCP/IP 协议，还可以分析 USB 协议，蓝牙协议等。
启动 USBPap2 , 设置过滤条件，`usb.bInterfaceClass == 0x08` 只分析 MSC 相关的协议包，并对比 MSC 在单独设备和 MSC+CDC 复合设备中通信的异同点。
结果令人惊喜，真是**一次抓包，问题锁定** ！！

对于 Windows USB 发出的 SCSI Inquiry 命令，MSC 单独设备回复：
<!-- ![wireshark-usb-right](/assets/image/wireshark-usb-right.png) <br/> -->
<img src="/assets/image/wireshark-usb-right.png" alt="wireshark-usb-right" width="512" height="306">

而 MSC+CDC 复合设备的回复：
<!-- ![wireshark-usb-wrong](/assets/image/wireshark-usb-wrong.png) <br/> -->
<img src="../../assets/image/wireshark-usb-wrong.png" alt="wireshark-usb-wrong" width="512" height="306">

显然问题出在这里，复合设备回复的长度信息错误。

这里对 Wireshark 的使用显得轻描淡写，但是这种趁手的工具真是节省了大量的时间，**Wireshark 是一个名副其实的“十倍效率”的工具**。使用 Keil 只能确定大体范围，如果想通过 Keil debug 锁定问题，怕是时间要以天计算。而 Wireshark 在 10 分钟内就可以基本精准定位问题。

### 4. 问题已经定位，如何解决？
`USBD_StatusTypeDef USBD_LL_Transmit(USBD_HandleTypeDef *pdev, uint8_t ep_addr, uint8_t *pbuf, uint32_t size)` 是 ST USB Library  回复 SCSI Inquiry 命令最后一个函数,截至到这个函数，每一个入参在 MSC 单独设备和 MSC+CDC 复合设备中都完全一样，在这个函数里将调用 HAL 库里的 `HAL_PCD_EP_Transmit(pdev->pData, ep_addr, pbuf, size);`, 进入总线层控制程序。
经过深挖，问题定位在`HAL_StatusTypeDef USB_EPStartXfer(USB_OTG_GlobalTypeDef *USBx, USB_OTG_EPTypeDef *ep)`函数，这是回复 SCSI Inquiry 命令的最后一个函数，该函数主要是设置寄存器，之后物理层就会把数据发出去。
错误就在写入数据的位置不对，而错误的来源则是前面提到的 `2.4 usbd_conf.c USBD_StatusTypeDef USBD_LL_Init(USBD_HandleTypeDef *pdev) 函数需要修改`
正确的应该是 
```C
  HAL_PCDEx_SetRxFiFo(&hpcd_USB_OTG_FS, 0x80);
  HAL_PCDEx_SetTxFiFo(&hpcd_USB_OTG_FS, 0, 0x40);
  HAL_PCDEx_SetTxFiFo(&hpcd_USB_OTG_FS, 1, 0x40);
  // HAL_PCDEx_SetTxFiFo(&hpcd_USB_OTG_FS, 2, 0x40);
  HAL_PCDEx_SetTxFiFo(&hpcd_USB_OTG_FS, 3, 0x40);
```
### 5. 修改，编译下载验证
修改后，编译下载，验证通过，CDC 能正常模拟串口通信，MSC 存储设备正常，问题解决。<br/>
<!-- ![usb-cdc-msc-good](/assets/image/usb-cdc-msc-good.png) <br/> -->
<img src="/assets/image/usb-cdc-msc-good.png" alt="usb-cdc-msc-good" width="360" height="408">
<br/>
<!-- ![this-compute-demostration](/assets/image/this-compute-demostration.png) <br/> -->
<img src="../../assets/image/this-compute-demostration.png" alt="this-compute-demostration" width="384" height="288">









































