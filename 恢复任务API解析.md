```c
#if ( INCLUDE_vTaskSuspend == 1 )

    void vTaskResume( TaskHandle_t xTaskToResume )
    {
        TCB_t * const pxTCB = xTaskToResume;

        /* It does not make sense to resume the calling task. */
		//不可以设置NULL
        configASSERT( xTaskToResume );

        /* The parameter cannot be NULL as it is impossible to resume the
         * currently executing task. */
        if( ( pxTCB != pxCurrentTCB ) && ( pxTCB != NULL ) )
        {
            taskENTER_CRITICAL();
            {
				//看该任务是否处于挂起
                if( prvTaskIsTaskSuspended( pxTCB ) != pdFALSE )
                {

                    /* The ready list can be accessed even if the scheduler is
                     * suspended because this is inside a critical section. */
					 //将任务从挂起列表送到就绪列表
                    ( void ) uxListRemove( &( pxTCB->xStateListItem ) );
                    prvAddTaskToReadyList( pxTCB );

                    /* A higher priority task may have just been resumed. */
					//如果恢复的任务的优先级比当前任务高，任务切换
                    if( pxTCB->uxPriority >= pxCurrentTCB->uxPriority )
                    {
                        /* This yield may not cause the task just resumed to run,
                         * but will leave the lists in the correct state for the
                         * next yield. */
                        taskYIELD_IF_USING_PREEMPTION();
                    }
                }
            }
            taskEXIT_CRITICAL();
        }
    }

#endif /* INCLUDE_vTaskSuspend */
```

