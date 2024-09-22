```c
#if ( INCLUDE_vTaskSuspend == 1 )

    void vTaskSuspend( TaskHandle_t xTaskToSuspend )
    {
        TCB_t * pxTCB;

        taskENTER_CRITICAL();
        {
            pxTCB = prvGetTCBFromHandle( xTaskToSuspend );

            if( uxListRemove( &( pxTCB->xStateListItem ) ) == ( UBaseType_t ) 0 )
            {
                taskRESET_READY_PRIORITY( pxTCB->uxPriority );
            }
            //这与删除任务不同，挂起任务会将任务添加到挂起列表
            vListInsertEnd( &xSuspendedTaskList, &( pxTCB->xStateListItem ) );

        }
        taskEXIT_CRITICAL();

        if( xSchedulerRunning != pdFALSE )
        {
            taskENTER_CRITICAL();
            {
                prvResetNextTaskUnblockTime();
            }
            taskEXIT_CRITICAL();
        }
        //如果待挂起任务就是当前任务
        if( pxTCB == pxCurrentTCB )
        {
            if( xSchedulerRunning != pdFALSE )
            {
                /* The current task has just been suspended. */
                configASSERT( uxSchedulerSuspended == 0 );
                //进行任务切换
                portYIELD_WITHIN_API();
            }
            else
            {
                //如果任务调度未开启
                if( listCURRENT_LIST_LENGTH( &xSuspendedTaskList ) == uxCurrentNumberOfTasks ) /*lint !e931 Right has no side effect, just volatile. */
                {
                    //没有其他就绪态任务了，pxcurrentTCB设为空
                    pxCurrentTCB = NULL;
                }
                else
                {
                    vTaskSwitchContext();
                }
            }
        }
    }

#endif /* INCLUDE_vTaskSuspend */
```

```c

void vTaskSwitchContext( void )
{
    if( uxSchedulerSuspended != ( UBaseType_t ) pdFALSE )
    {
        //任务调度器挂起，不允许上下文切换，直接返回
        xYieldPending = pdTRUE;
    }
    else
    {
        xYieldPending = pdFALSE;

        /* Select a new task to run using either the generic C or port
         * optimised asm code. */
        taskSELECT_HIGHEST_PRIORITY_TASK(); /*lint !e9079 void * is used as this macro is used with timers and co-routines too.  Alignment is known to be fine as the type of the pointer stored and retrieved is the same. */
        traceTASK_SWITCHED_IN();

        
    }
}
```

```c
 #define taskSELECT_HIGHEST_PRIORITY_TASK()                                                  \
    {                                                                                           \
        UBaseType_t uxTopPriority;                                                              \
                                                                                                \
        /* Find the highest priority list that contains ready tasks. */                         \
        portGET_HIGHEST_PRIORITY( uxTopPriority, uxTopReadyPriority );                          \
        configASSERT( listCURRENT_LIST_LENGTH( &( pxReadyTasksLists[ uxTopPriority ] ) ) > 0 ); \
        listGET_OWNER_OF_NEXT_ENTRY( pxCurrentTCB, &( pxReadyTasksLists[ uxTopPriority ] ) );   \
    } /* taskSELECT_HIGHEST_PRIORITY_TASK() */
```

