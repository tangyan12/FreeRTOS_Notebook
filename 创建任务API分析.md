以下是跟任务有关的列表

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726228079040-b9398cf2-6c38-424b-be4e-efca43e3c12a.png)

1.进入 xTaskCreate

2.为任务栈空间和任务控制块申请内存(任务创建的栈是以字为单位的)

3.进入 prvInitialiseNewTask，初始化任务控制块

4.初始化控制块的人物名字、优先级、状态/事件列表各种

5.进入pxPortInitialiseStack 初始化任务栈，主要添加一些关键寄存器信息，为一些 CPU 寄存器预留位置如下图

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726229901890-02dce014-67b2-4029-a817-572ea57080cc.png)

6.退出prvInitialiseNewTask

7.进入prvAddNewTaskToReadyList，将任务添加到就绪列表

8.如果是第一个创建的任务就初始化任务相关列表

如果不是第一个，且任务调度未开启，则更新 pxcurrentTCB 设定为最高优先级任务控制块

添加到就绪列表

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726235755693-6e43d7e1-90e9-4e02-aedd-209c10cca34b.png)

如果任务调度已经开启，则根据抢占机制，优先级高就切换任务







