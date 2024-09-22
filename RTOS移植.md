[FreeRTOS官网](https://www.freertos.org)

从上面可以下载到 RTOS 源码，分为最新版本和 LTS (Long Time Support)版本



将 RTOS 移植到 stm32



进入固件包/FreeRTOS/Source，该目录下就是我们需要的全部文件

可以直接将它拷贝到工程的组件文件夹下(新建一个 FreeRTOS)

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726142736898-9c4b2b4d-e7eb-43a1-bd50-8591c029a3fc.png)

这是这个文件夹的详细情况

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726142808448-1dd2862b-2d6c-4e9a-a32a-b022130b55ef.png)

在 Keil 下将所需.c 文件引入，并添加了 include 头文件路径，portable 内也有包含路径，主要是针对不同编译器的移植文件 portmacro.h，针对 AC5 编译链，portable 所需文件只有

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726143012829-691fb415-01b4-4582-90fa-83812592e07d.png)

如果是 AC6 则删除 RVDS 保留 GCC 文件夹

**还有一个 freertosconfig.h 文件**，这个在固件包的 demo 里有找到 stm32 对应工程复制一份即可

至此，内核文件已经全部处理完毕，接下来让 rtos 跟原来工程兼容一下就可以开始运行了



中断的处理

```c
#if !defined(USE_FREERTOS)
/**
  * @brief  This function handles PendSVC exception.
  * @param  None
  * @retval None
  */
void PendSV_Handler(void)
{
}
#endif	
#if !defined(USE_FREERTOS)
/**
  * @brief  This function handles SVCall exception.
  * @param  None
  * @retval None
  */
void SVC_Handler(void)
{
}

#endif
#ifdef USE_FREERTOS
	extern void xPortSysTickHandler(void);
#endif
/**
  * @brief  This function handles SysTick Handler.
  * @param  None
  * @retval None
  */
void SysTick_Handler(void)
{
  HAL_IncTick();
	#ifdef USE_FREERTOS
	if(xTaskGetSchedulerState() != taskSCHEDULER_NOT_STARTED){
		xPortSysTickHandler();
	}
 	#endif
}
```

SVC 和 PenSV 中断 freertos 都有定义，所以需要注释，SYSTick 中断是为了延时函数才这样改，这样子就可以在任务调度开始之前也可以使用 delay_us 和 delay_ms 延时了



除了上面那个，其他主要还是跟 config 配置文件有关，我现在了解的也不多，只知道一个中断位数

```c
#ifdef __NVIC_PRIO_BITS
    #define configPRIO_BITS __NVIC_PRIO_BITS
#else
    #define configPRIO_BITS 4
#endif

```

这个是由正点原子编写的条件编译，这个 __NVIC_PRIO_BITS 可以在 stm32103xE.h 中找到，但是最后编译却是执行下面那句，可能那个头文件因为什么条件没开启那个宏

[FreeRTOSConfig.h](https://www.yuque.com/attachments/yuque/0/2024/txt/40891866/1726144054253-6edc6def-6b59-42a2-8c4a-f0172aff1772.txt)











