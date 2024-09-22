```c
		
		//下面这个节选于空闲任务，如果一次的xExpectedIdleTime太大，SYSTICK-LOAD-REG装不下
		就会循环执行，这点注意下
		
		//configUSE_TICKLESS_IDLE应该被设置成除1以外的其他数(为什么)
        #if ( configUSE_TICKLESS_IDLE != 0 )
            {
                TickType_t xExpectedIdleTime;

                //在空闲任务运行期间，重复挂起恢复调度器以此来达到低功耗是不可取的
				//这里的计算结果xExpectedIdleTime不一定有效
                xExpectedIdleTime = prvGetExpectedIdleTime();
				//要求空闲任务运行时间至少>=configEXPECTED_IDLE_TIME_BEFORE_SLEEP才能进入低功耗
                if( xExpectedIdleTime >= configEXPECTED_IDLE_TIME_BEFORE_SLEEP )
                {
                    vTaskSuspendAll();
                    {
						//任务调度器挂起，这次xExpectedIdleTime的值是有效的
                        configASSERT( xNextTaskUnblockTime >= xTickCount );
                        xExpectedIdleTime = prvGetExpectedIdleTime();

						//如果不想启用低功耗就定义这个宏，将xExpectedIdleTime设为0
                        configPRE_SUPPRESS_TICKS_AND_SLEEP_PROCESSING( xExpectedIdleTime );

                        if( xExpectedIdleTime >= configEXPECTED_IDLE_TIME_BEFORE_SLEEP )
                        {
                            portSUPPRESS_TICKS_AND_SLEEP( xExpectedIdleTime );
                        }
                    }
                    ( void ) xTaskResumeAll();
                }
            }
        #endif /* configUSE_TICKLESS_IDLE */
		
#ifndef portSUPPRESS_TICKS_AND_SLEEP
        extern void vPortSuppressTicksAndSleep( TickType_t xExpectedIdleTime );
        #define portSUPPRESS_TICKS_AND_SLEEP( xExpectedIdleTime )    vPortSuppressTicksAndSleep( xExpectedIdleTime )
    #endif
	
	
#if ( configUSE_TICKLESS_IDLE == 1 )

    __weak void vPortSuppressTicksAndSleep( TickType_t xExpectedIdleTime )
    {
        uint32_t ulReloadValue, ulCompleteTickPeriods, ulCompletedSysTickDecrements;
        TickType_t xModifiableIdleTime;
		//貌似受限于SYSTICKLOAD寄存器的24位限制，等待的时钟数只能为0x748
        if( xExpectedIdleTime > xMaximumPossibleSuppressedTicks )
        {
            xExpectedIdleTime = xMaximumPossibleSuppressedTicks;
        }

        //停止Tick中断
        portNVIC_SYSTICK_CTRL_REG &= ~portNVIC_SYSTICK_ENABLE_BIT;

        //SYSTICK-LOAD-REG重新装填的值
        ulReloadValue = portNVIC_SYSTICK_CURRENT_VALUE_REG + ( ulTimerCountsForOneTick * ( xExpectedIdleTime - 1UL ) );
		//减去一个很小的损耗
        if( ulReloadValue > ulStoppedTimerCompensation )
        {
            ulReloadValue -= ulStoppedTimerCompensation;
        }
		
        __dsb( portSY_FULL_READ_WRITE );
        __isb( portSY_FULL_READ_WRITE );

        //如果还有任务在等待恢复之类的，就放弃进入低功耗模式
        if( eTaskConfirmSleepModeStatus() == eAbortSleep )
        {
            /* Restart from whatever is left in the count register to complete
             * this tick period. */
            portNVIC_SYSTICK_LOAD_REG = portNVIC_SYSTICK_CURRENT_VALUE_REG;

            /* Restart SysTick. */
            portNVIC_SYSTICK_CTRL_REG |= portNVIC_SYSTICK_ENABLE_BIT;

            /* Reset the reload register to the value required for normal tick
             * periods. */
            portNVIC_SYSTICK_LOAD_REG = ulTimerCountsForOneTick - 1UL;

            /* Re-enable interrupts - see comments above __disable_irq() call
             * above. */
            __enable_irq();
        }
        else
        {
			//设置LOAD-REG
            portNVIC_SYSTICK_LOAD_REG = ulReloadValue;

            //将VAL-REG清空
            portNVIC_SYSTICK_CURRENT_VALUE_REG = 0UL;
			//重新启动SYSTICK
            portNVIC_SYSTICK_CTRL_REG |= portNVIC_SYSTICK_ENABLE_BIT;

            //下面这句我不太清楚，可以查看源代码
            xModifiableIdleTime = xExpectedIdleTime;
			//进入睡眠前要做的，自定义宏
            configPRE_SLEEP_PROCESSING( xModifiableIdleTime );

            if( xModifiableIdleTime > 0 )
            {
                __dsb( portSY_FULL_READ_WRITE );
                __wfi();
                __isb( portSY_FULL_READ_WRITE );
            }
			//退出睡眠前要做的，自定义宏
            configPOST_SLEEP_PROCESSING( xExpectedIdleTime );

            //重新开启中断
            __enable_irq();
            __dsb( portSY_FULL_READ_WRITE );
            __isb( portSY_FULL_READ_WRITE );

            //关闭中断
            __disable_irq();
            __dsb( portSY_FULL_READ_WRITE );
            __isb( portSY_FULL_READ_WRITE );

           //停止SYSTICK计数
            portNVIC_SYSTICK_CTRL_REG = ( portNVIC_SYSTICK_CLK_BIT | portNVIC_SYSTICK_INT_BIT );

			//判断计时标志
            //这下面的我就都不懂了，不过影响不大
            if( ( portNVIC_SYSTICK_CTRL_REG & portNVIC_SYSTICK_COUNT_FLAG_BIT ) != 0 )
            {
                uint32_t ulCalculatedLoadValue;

                ulCalculatedLoadValue = ( ulTimerCountsForOneTick - 1UL ) - ( ulReloadValue - portNVIC_SYSTICK_CURRENT_VALUE_REG );

             
                if( ( ulCalculatedLoadValue < ulStoppedTimerCompensation ) || ( ulCalculatedLoadValue > ulTimerCountsForOneTick ) )
                {
                    ulCalculatedLoadValue = ( ulTimerCountsForOneTick - 1UL );
                }

                portNVIC_SYSTICK_LOAD_REG = ulCalculatedLoadValue;

                ulCompleteTickPeriods = xExpectedIdleTime - 1UL;
            }
            else
            {
                //tick中断以外的其他中断唤醒了CPU
                ulCompletedSysTickDecrements = ( xExpectedIdleTime * ulTimerCountsForOneTick ) - portNVIC_SYSTICK_CURRENT_VALUE_REG;

                
                ulCompleteTickPeriods = ulCompletedSysTickDecrements / ulTimerCountsForOneTick;

               
                portNVIC_SYSTICK_LOAD_REG = ( ( ulCompleteTickPeriods + 1UL ) * ulTimerCountsForOneTick ) - ulCompletedSysTickDecrements;
            }

            portNVIC_SYSTICK_CURRENT_VALUE_REG = 0UL;
            portNVIC_SYSTICK_CTRL_REG |= portNVIC_SYSTICK_ENABLE_BIT;
            vTaskStepTick( ulCompleteTickPeriods );
            portNVIC_SYSTICK_LOAD_REG = ulTimerCountsForOneTick - 1UL;

            /* Exit with interrupts enabled. */
            __enable_irq();
        }
    }

#endif /* #if configUSE_TICKLESS_IDLE */
```

说实话感觉比较难分析，主体思路大概是 freertos 会计算空闲任务可以运行的时间，如果时间大于一个阈值就会进入低功耗模式，低功耗会通过 wfi 指令使 CPU 进入睡眠模式，在此之前会配置好 SYSTICK 中断以唤醒 CPU



光靠 freertos 的 tickless 低功耗无法达到很好的效果，一般需要自己来实现

configPRE_SLEEP_PROCESSING 和configPOST_SLEEP_PROCESSING 宏完成进入睡眠前和退出睡眠前执行的任务，并且可以通过 configPRE_SLEEP_PROCESSING(xModifiableIdleTime)这个宏将xModifiableIdleTime 设为 0，不使用 wfi 而使用自己的低功耗设计



参考了网上一篇文章，里面介绍了很好的自定义停机模式的低功耗设计

[FreeRTOS 低功耗之 tickless 模式 - Crystal_Guang - 博客园 (cnblogs.com)](https://www.cnblogs.com/yangguang-it/p/7232448.html)

```c
/*
*********************************************************************************************************
*    函 数 名: OS_PreSleepProcessing
*    功能说明: 下面的函数在文件FreeRTOSConfig.h文件里面进行了宏定义：
*              #define configPRE_SLEEP_PROCESSING(x)  OS_PreSleepProcessing(x)
*              #define configPOST_SLEEP_PROCESSING(x) OS_PostSleepProcessing(x)
*              在文件port.c里面函数vPortSuppressTicksAndSleep调用了上面这两个函数：
*              ---------------------------------------------------------------------
*                configPRE_SLEEP_PROCESSING( xModifiableIdleTime );
*                if( xModifiableIdleTime > 0 )
*                {
*                    __dsb( portSY_FULL_READ_WRITE );
*                    __wfi();
*                    __isb( portSY_FULL_READ_WRITE );
*                }
*                configPOST_SLEEP_PROCESSING( xExpectedIdleTime );
*             -----------------------------------------------------------------------
*             通过这两个函数可以实现在调用__WFI或者__WFE指令前后执行进一步的低功耗操作，主要有以下三种：
*             1. 降低系统主频。
*             2. 关闭外设时钟。
*             3. IO引脚要做处理，防止拉电流和灌电流增加功耗。
*                如果此IO口带上拉，请设置为高电平输出或者高阻态输入；
*                如果此IO口带下拉，请设置为低电平输出或者高阻态输入；
*             下面的函数未做关闭外设时钟的处理。
*    形    参: 无
*    返 回 值: 无
*********************************************************************************************************
*/
void OS_PreSleepProcessing(uint32_t vParameters)
{
    (void)vParameters;
    
    /* 用户可以考虑在此处加入关闭外设时钟来进一步降低功耗 */
    vParameters = 0;
    PWR_EnterSTOPMode(PWR_Regulator_LowPower, PWR_STOPEntry_WFE);
}

void OS_PostSleepProcessing(uint32_t vParameters)
{
    /* 如果前面关闭了外设时钟，需要在这里恢复 */
    /* 
      1、当一个中断或唤醒事件导致退出停止模式时，HSI RC振荡器被选为系统时钟。
      2、退出低功耗的停机模式后，需要重新配置使用HSE。        
    */
    RCC_HSEConfig(RCC_HSE_ON);
    while (RCC_GetFlagStatus(RCC_FLAG_HSERDY) == RESET){}

    RCC_PLLCmd(ENABLE);
    while (RCC_GetFlagStatus(RCC_FLAG_PLLRDY) == RESET){}
    RCC_SYSCLKConfig(RCC_SYSCLKSource_PLLCLK);
        
    while (RCC_GetSYSCLKSource() != 0x08){}
}
```

还有一点要提醒，这个貌似是停机模式下调试的支持，我目前没用到，先记录着

![](https://cdn.nlark.com/yuque/0/2024/png/40891866/1726735667370-25b29b18-c9bf-4e11-85a8-0bd3cdcfe0c3.png)

如何获得空闲任务可以运行的时间我没有去分析，有空再说

Note：SYSTICK 是 向下计数的

