外部中断优先级设置寄存器 IP[0]-IP[67]，只有高四位可用，数值越小优先级越高

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726214129402-92994f5e-cd8c-4d96-b58e-efd1493d36f4.png)

系统中断优先级设置寄存器，一共有三个寄存器 SHPR1-3，SHPR3 可以设置 PendSV 和 SYSTick 中断的优先级

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726214134785-a48f3883-5a48-4ec7-8686-40ec8d80c2b5.png)

下图为寄存器位定义，FAULTMASK它跟 PRIMASK 一样，只有 0 位有效，且也是关闭中断的作用，这两个寄存器不同的在于是否会关闭某个系统中断(目前不了解)

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726214175201-cf6f7640-7432-4df1-86a7-f6455df338b6.png)

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726214167350-6c487f1b-48ec-41dd-915d-f2bfee9aa684.png)

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726214255656-7ecb3156-a760-4b70-b83e-b570a9848c4f.png)

支持使用以下方式设置特定寄存器寄存器的值，关于中断的寄存器 PRIMASK/BASEPRI/FAULTMASK 都可以通过这种方式设置，定义在 cmsis_gcc.c 中(我是在 PIO 平台上运行，用的是 gcc 编译链，在 MDK 上运行不是这个文件)

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726214147846-6c370a18-b067-4fcd-be64-27237a5d74a7.png)

freertos 中定义了 SHPR3 寄存器，在任务调度开始函数中设置了 PendSV 和 SYSTick 的优先级

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726216212312-8d8c93da-0a39-46ff-ac30-15015ab8ad19.png)

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726216206047-2683ed8a-36db-4e75-af4b-0a013e36ff5d.png)

这是 FreeRTOSconfig.h 文件中关于中断的宏定义

```c
/* 中断嵌套行为配置 */
#ifdef __NVIC_PRIO_BITS
    #define configPRIO_BITS __NVIC_PRIO_BITS
#else
    #define configPRIO_BITS 4
#endif

#define configLIBRARY_LOWEST_INTERRUPT_PRIORITY         15                  /* 中断最低优先级 */
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY    5                   /* FreeRTOS可管理的最高中断优先级 */
#define configKERNEL_INTERRUPT_PRIORITY                 ( configLIBRARY_LOWEST_INTERRUPT_PRIORITY << (8 - configPRIO_BITS) )
#define configMAX_SYSCALL_INTERRUPT_PRIORITY            ( configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY << (8 - configPRIO_BITS) )
#define configMAX_API_CALL_INTERRUPT_PRIORITY           configMAX_SYSCALL_INTERRUPT_PRIORITY
```

configPRIO_BITS 中断优先级所用到的位数，STM32F1 为 4

configKERNEL_INTERRUPT_PRIORITY  这个是 PendSV 和 SYSTick 中断的优先级，设成了最低



**中断宏**

```c
  /*关闭/使能中断*/
  taskENABLE_INTERRUPTS();
  taskDISABLE_INTERRUPTS();
  /*进入/退出临界区*/
  taskENTER_CRITICAL();
  taskEXIT_CRITICAL();
  /*在中断中进入/退出临界区*/
  uint32_t ulreturn;
  ulreturn = taskENTER_CRITICAL_FROM_ISR();
  taskEXIT_CRITICAL_FROM_ISR(ulreturn);
```

下面是 freertos 关闭/使能中断(通过写 BASEPRI 寄存器关闭/使能 freertos 可管理的中断优先级范围)

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726216356332-db4f11fc-7a0c-49c9-970c-2e8f0e9545c3.png)

RaiseBASEPRI 就是将 freertosconfig.h 中的 configMAX_SYSCALL_INTERRUPT_PRIORITY 的值写到 BASEPRI 寄存器，关闭数值比它大，即优先级比它低的中断

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726216693257-97230433-a112-41b2-aad8-613ce94829a5.png)

SetBASEPRI 就是将 0 写给 BASEPRI 寄存器，写 0 在寄存器位定义中并无效果，也意味着中断不受限制



临界区：有一些代码如时序代码希望在执行的时候不要被打断，比如来个某个中断，轮到别的任务运行，那么就可以在时序代码的起始和结束设置为临界区，临界区其实也就是上面提到的开启关闭中断，只不过加了一些处理



进入临界区，先关闭中断，临界变量加 1，如果是第一次进入临界区需要判断调用该函数是否位于中断内，因为该函数不应该在中断内调用

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726216700297-0b9bf6cf-52d4-4287-a128-637d9e88f739.png)

退出临界区

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726216706203-a49c944c-6a8c-42dc-b8e6-6c783b19d7cd.png)



这是从中断中进入临界区调用的函数，它会把原先 BASEPRI 中的值记录下来并返回，然后写configMAX_SYSCALL_INTERRUPT_PRIORITY 到 BASEPRI 寄存器，关闭受 freertos 管理的中断，在中断中退出临界区需要把之前进入临界区返回的值当作参数传入

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726216806538-0ceb7edf-54f1-45d3-ae04-41c75ed1415e.png)

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726216813703-ec63d9f0-bcf8-4707-a16c-7d372e5c07e8.png)

