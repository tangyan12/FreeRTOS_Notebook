```c
#if ( INCLUDE_vTaskDelete == 1 )

    void vTaskDelete( TaskHandle_t xTaskToDelete )
    {
        TCB_t * pxTCB;
		//进入临界区
        taskENTER_CRITICAL();
        {
            //根据任务句柄获得任务控制块
            pxTCB = prvGetTCBFromHandle( xTaskToDelete );

            //将任务从就绪态/阻塞态列表移除
            if( uxListRemove( &( pxTCB->xStateListItem ) ) == ( UBaseType_t ) 0 )
            {
				
                taskRESET_READY_PRIORITY( pxTCB->uxPriority );
            }

            //任务事件列表删除
            if( listLIST_ITEM_CONTAINER( &( pxTCB->xEventListItem ) ) != NULL )
            {
                ( void ) uxListRemove( &( pxTCB->xEventListItem ) );
            }
			
            //用于调试
            uxTaskNumber++;
			
			//如果待删除任务是调用任务本身
            if( pxTCB == pxCurrentTCB )
            {
                //任务无法删除本身，将任务移到待删除任务列表交由空闲任务删除
                vListInsertEnd( &xTasksWaitingTermination, &( pxTCB->xStateListItem ) );

                //告知空闲任务有任务要删除
                ++uxDeletedTasksWaitingCleanUp;
            }
            else//带删除任务不是任务本身
            {
				//任务数量减少
                --uxCurrentNumberOfTasks;
				//待删除任务可能是阻塞态任务，所以需要更新下一个任务的阻塞时间
            
                prvResetNextTaskUnblockTime();
            }
        }
        taskEXIT_CRITICAL();

        //如果待删除任务是调用任务本身，则调用任务负责删除该任务
        if( pxTCB != pxCurrentTCB )
        {
            prvDeleteTCB( pxTCB );
        }

        //如果删除的任务就是现在运行的任务则需要进行任务切换
        if( xSchedulerRunning != pdFALSE )
        {
            if( pxTCB == pxCurrentTCB )
            {
				//任务调度不处于挂起
                configASSERT( uxSchedulerSuspended == 0 );
                portYIELD_WITHIN_API();
            }
        }
    }

#endif /* INCLUDE_vTaskDelete */

```

```c
#if ( INCLUDE_vTaskDelete == 1 )

    static void prvDeleteTCB( TCB_t * pxTCB )
    {
        #if ( ( configSUPPORT_DYNAMIC_ALLOCATION == 1 ) && ( configSUPPORT_STATIC_ALLOCATION == 0 ) && ( portUSING_MPU_WRAPPERS == 0 ) )
            {
                
                vPortFreeStack( pxTCB->pxStack );
                vPortFree( pxTCB );
            }   
    }

#endif /* INCLUDE_vTaskDelete */
```

```c
static void prvResetNextTaskUnblockTime( void )
{
    if( listLIST_IS_EMPTY( pxDelayedTaskList ) != pdFALSE )
    {
        如果阻塞列表为空，则设为最大值
        xNextTaskUnblockTime = portMAX_DELAY;
    }
    else
    {
        //获取阻塞列表首项的阻塞时间
        xNextTaskUnblockTime = listGET_ITEM_VALUE_OF_HEAD_ENTRY( pxDelayedTaskList );
    }
}
```



