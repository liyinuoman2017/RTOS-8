# RTOS-8
**嵌入式实时操作系统8——等待表**

## 1.等待表用途

在多任务系统中经常会有部分任务在运行到某个节点时需要延时等待一段时间，等待表的作用就是帮助操作系统内核管理需要延时等待的任务。
当运行的任务需要延时等待，此时操作系统内核会将该任务从就绪表中移动到等待表中；当完成延时等待时间后，操作系统内核会将该任务从等待表中移动到就绪表中，状态图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/de9ba54d9be3475b94669ae70c0f538d.png)
例如现在需要连续读取一个温湿度传感头的数据，但是该温度传感器在启动一次测量到输出一个稳定数值需要等待50ms。这种情况我们有两种策略：

1、启动测量后死等50ms，然后读取测量值。
2、启动测量后，运行其他任务，50ms后再来读取温湿度测量值。
这两种策略的运行状态如下图：

![在这里插入图片描述](https://img-blog.csdnimg.cn/f58d6dc7688842cb8df1b9bab73223b7.png)

很显然需要延时等待时去执行其他任务的这种策略的CPU利用率更高，操作系统内核就时利用等待表实现这种策略。

## 2.等待策略

要完成等待很多种策略，常见的策略有以下两种：
**1、倒计时法
2、闹钟法**

相信大家都看过火箭发射的场面，那清晰洪亮的倒计时声仿佛就在耳边回响：
10...9...8...7...6...5...4...3...2...1...0
火箭发射的就是使用的倒计时法，这种策略只用关注剩余的时间。

![在这里插入图片描述](https://img-blog.csdnimg.cn/b3724c638edc405d9ba6893039e0b604.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

大部分人早上起床的时候肯定是被设定的闹钟叫醒，这种策略使用起来十分方便，只用关注设定的时间即可。

> 倒计时法的算法逻辑是：每一次判断剩余时间是否为0，如果剩余时间不为0，如果剩余时间为0，则完成延时等待。
> 闹钟法的算法逻辑是：比较当前时间是否小于设定时间，小于设定时间则继续等待，不小于设定时间则完成延时等待。

**比较这两种策略**

> 倒计时需要完成一次判断操作和一次减法操作，闹钟法只需要完成一次判断操作（当前时间由系统生成）。由于闹钟法执行步骤较少，通常情况下实时操作系统的等待表使用闹钟法机制。
> 闹钟法存在一个时间归零问题，假设当前时间是23点，如果需要等待2小时，闹钟时间就变成了1点。需要注意这里的比较逻辑和计算逻辑。

## 3.等待表实现


使用静态数组的方式可以构建一个就绪表，代码实现如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/d5e2740cdbf94c0f9b496068ddd8a24b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_15,color_FFFFFF,t_70,g_se,x_16)

其中tcb_item_t为 TCB项 ，list_item_t列表项，delay_table为等待表。
delay_table中的每个列表包含10个TCB项，itme[0]表示任务0，itme[9]表示任务9。
每个TCB项中包含一个标志位和TCB指针，TCB指针指向任务的TCB数据结构。数据结构图如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/8a085dca522d459d8404ed86cbe88457.png)

使用**静态数组**方式的优点是：结构简单，使用方便。但是使用静态数组的缺点非常明显：
1、每个一优先级容纳的任务数量是固定的，一旦需要增加某个优先级任务数量，整个列表大小将增加。
2、在同一个优先级任务中间插入一个任务，需要移动多个TCB项。
3、存在多个优先级未用的情况，导致内存浪费严重。
**<font color=red>综合上述问题，因此使用静态数组的方式是不明智的选择。**

构建等待表可以使用**双向链表**的方式，代码实现如下：
  
![在这里插入图片描述](https://img-blog.csdnimg.cn/efbfb16cc33442189ded1331627e855f.png)
  
list_item_t列表项，delay_table为等待表。
每个list_item_t列表中包含一个TCB指针，下一个列表项指针和上一个列表项指针。数据结构图如下：
  
![在这里插入图片描述](https://img-blog.csdnimg.cn/4f2b19ec351b4116a97e6efa12b5c88c.png)
  
使用**双向链表**的方式有以下优点：
1、每一个链表可以连接任意数量的链表项，长度不受限制。
2、链表的每一项，都是有用项，不存在内存浪费
3、在链表中间插入一项，操作效率较高。
**因此使用双向链表构建等待表是很好的选择。**


## 4.等待表优化

**1、排序链表法**
前文提出了一种利用链表的方式构建的等待表，根据时间构建链表，时间小的放在链表头部，时间大的放在链表尾部。每次更新时间只有检查第一个对象即可，因为第一个对象是时间最小的（与当前时间间隔最小）。
  
![在这里插入图片描述](https://img-blog.csdnimg.cn/797fa2e59d864a4b8b0b9bd851d3c59d.png)
  
这种方式操作简单，但是有一个问题：当任务很多时，插入和移除一个新任务的时间开销会非常大。比如现在有100个任务，假设现在新插入一个任务，该新插入的任务延时时间最大，因此该任务需要执行100次比较之和100次读取下一个任务的操作才能完成插入操作。

**2、 时间取模分表**
先用一个实例解释时间取模分表法：
假设现在有一个靠近火车站的旅店，客户进旅店休息并申请一个叫醒业务，旅店到时间后提供服务员上面叫醒服务，假设叫醒业务是以分为单位。由于来的客户的时间是乱序的，不同的客户等待的时间也是乱序的，这里如果制作一个普通的叫醒表，每次更新一分钟，服务员要核对以下所有客户的定时时间，这样费时也容易出错。
因此我们用时间取模分表法制作一个特殊的表：
  
![在这里插入图片描述](https://img-blog.csdnimg.cn/5e5a2646c68241189aecdc51e3896208.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_17,color_FFFFFF,t_70,g_se,x_16)
  
时间是当前时间的分时间的个位，如12点37分对应的时间为7 ，8点15分对应的时间为5。客户信息中包含闹钟时间和房号，客户信息中按照时间由近及远的方式排列。
加入客户后的叫醒表如下：
  
![在这里插入图片描述](https://img-blog.csdnimg.cn/453c8ca3193047f3924868f33380d4ee.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_20,color_FFFFFF,t_70,g_se,x_16)

> 这时候服务员只用检查当前时间的分时间的个位，然后再判断对应表项的第一个客户的时间是否到了即可，整个过程只用判断2次既可。
> 例如：当前时间为8点05分，服务员只有判断表中时间为5的那列，然后检查第一个，结果是没有客户需要唤醒。
> 例如：当前时间为9点02分，服务员只有判断表中时间为2的那列，然后检查第一个，结果是没有客户需要唤醒。
> 例如：当前时间为11点19分，服务员只有判断表中时间为9的那列，然后检查第一个，结果是有客户需要唤醒。

**这种策略执行效率较高，并且插入和移除一个新任务的时间开销比较小。**


## 5.FreeRTOS源码分析

FreeRTOS的等待表定义如下：
  
![在这里插入图片描述](https://img-blog.csdnimg.cn/f1f64c6441264c2e843405db9d336777.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAbGl5aW51bzIwMTc=,size_17,color_FFFFFF,t_70,g_se,x_16)
  
FreeRTOS的等待表使用的是排序链表法。

> 等待表的3种操作方法：
>  1、插入等待表。
>   2、移除等待表。
>    3、查询等待表。

**插入等待表**

```c
/* 将任务插入等待表，插入位置和延时时候有关 */ 	
vListInsert( pxDelayedTaskList, &( pxCurrentTCB->xStateListItem ) );
```

将任务插入等待表，插入位置和延时时候有关，时间小的放在链表头部，时间大的放在链表尾部。
	

**移除等待表**
将任务从等待表头部移除，每次更新时间只检查等待表的第一个对象，因为第一个对象是时间最小的（与当前时间间隔最小）。

```c
/* 将等待表的表头任务移除 */ 	
( void ) uxListRemove( &( pxTCB->xStateListItem ));
```


**查询等待表**

```c
pxTCB = listGET_OWNER_OF_HEAD_ENTRY( pxDelayedTaskList ); 				/* 获取等待表的表头 */	
xItemValue = listGET_LIST_ITEM_VALUE( &( pxTCB->xStateListItem ) );		/* 获取表头任务的延时时间 */	

if( xConstTickCount < xItemValue )  /* 比较延时时间 */
{
	xNextTaskUnblockTime = xItemValue;
	break; 
}
```
从等待表头部获取任务，每次更新时间只检查等待表的第一个对象，因为第一个对象是时间最小的（与当前时间间隔最小），判断该任务等待时间是否完成。


## 6.FreeRTOS等待表更新维护

前文说明了等待表的3种操作方法：插入等待表，移除等待表，查询等待表。操作系统在哪些地方完成这些操作，下面来一一列举一下：

**vTaskDelay**


vTaskDelay的作用是当前任务需要延时等待一定时间,该系统函数的调用流程如下：

```c
vTaskDelay ->  
prvAddCurrentTaskToDelayedList->
vListInsert( pxDelayedTaskList, &( pxCurrentTCB->xStateListItem ) )
```

等待表变化：将当前任务从就绪表中移除，然后将当前任务插入等待表中，对应的优先级位清0。（清0操作会判断该优先级下的就绪任务总数量）



**vTaskDelayUntil**
vTaskDelayUntil的作用是当前任务需要延时等待一定时间,该系统函数的调用流程如下：

```c
vTaskDelayUntil->  
prvAddCurrentTaskToDelayedList->
vListInsert( pxDelayedTaskList, &( pxCurrentTCB->xStateListItem ) )
```

等待表变化：将当前任务从就绪表中移除，然后将当前任务插入等待表中，对应的优先级位清0。（清0操作会判断该优先级下的就绪任务总数量）


**XPortSysTickHandler**
XPortSysTickHandler的作用是定时更新系统时间,并判断任务是否完成等待时间，该系统函数的调用流程如下：

```c
XPortSysTickHandler->  
xTaskIncrementTick->  
listGET_OWNER_OF_HEAD_ENTRY
listGET_LIST_ITEM_VALUE
( void ) uxListRemove( &( pxTCB->xStateListItem ) );
```

等待表变化：将当前任务从等待表中移除，然后将当前任务插入就绪表中，对应的优先级位清1。


## 7.源码

```c
	PRIVILEGED_DATA static List_t * volatile pxDelayedTaskList; /* 等待表 */

	void vTaskDelay( const TickType_t xTicksToDelay )
	{
	BaseType_t xAlreadyYielded = pdFALSE;

		/* A delay time of zero just forces a reschedule. */
		if( xTicksToDelay > ( TickType_t ) 0U )
		{
			configASSERT( uxSchedulerSuspended == 0 );
			vTaskSuspendAll();
			{
				traceTASK_DELAY();

				/* A task that is removed from the event list while the
				scheduler is suspended will not get placed in the ready
				list or removed from the blocked list until the scheduler
				is resumed.

				This task cannot be in an event list as it is the currently
				executing task. */
				prvAddCurrentTaskToDelayedList( xTicksToDelay, pdFALSE );
			}
			xAlreadyYielded = xTaskResumeAll();
		}
		else
		{
			mtCOVERAGE_TEST_MARKER();
		}

		/* Force a reschedule if xTaskResumeAll has not already done so, we may
		have put ourselves to sleep. */
		if( xAlreadyYielded == pdFALSE )
		{
			portYIELD_WITHIN_API();
		}
		else
		{
			mtCOVERAGE_TEST_MARKER();
		}
	}




static void prvAddCurrentTaskToDelayedList( TickType_t xTicksToWait, const BaseType_t xCanBlockIndefinitely )
{
TickType_t xTimeToWake;
const TickType_t xConstTickCount = xTickCount;

	#if( INCLUDE_xTaskAbortDelay == 1 )
	{
		/* About to enter a delayed list, so ensure the ucDelayAborted flag is
		reset to pdFALSE so it can be detected as having been set to pdTRUE
		when the task leaves the Blocked state. */
		pxCurrentTCB->ucDelayAborted = pdFALSE;
	}
	#endif

	/* Remove the task from the ready list before adding it to the blocked list
	as the same list item is used for both lists. */
	if( uxListRemove( &( pxCurrentTCB->xStateListItem ) ) == ( UBaseType_t ) 0 )
	{
		/* The current task must be in a ready list, so there is no need to
		check, and the port reset macro can be called directly. */
		portRESET_READY_PRIORITY( pxCurrentTCB->uxPriority, uxTopReadyPriority ); /*lint !e931 pxCurrentTCB cannot change as it is the calling task.  pxCurrentTCB->uxPriority and uxTopReadyPriority cannot change as called with scheduler suspended or in a critical section. */
	}
	else
	{
		mtCOVERAGE_TEST_MARKER();
	}

	#if ( INCLUDE_vTaskSuspend == 1 )
	{
		if( ( xTicksToWait == portMAX_DELAY ) && ( xCanBlockIndefinitely != pdFALSE ) )
		{
			/* Add the task to the suspended task list instead of a delayed task
			list to ensure it is not woken by a timing event.  It will block
			indefinitely. */
			vListInsertEnd( &xSuspendedTaskList, &( pxCurrentTCB->xStateListItem ) );
		}
		else
		{
			/* Calculate the time at which the task should be woken if the event
			does not occur.  This may overflow but this doesn't matter, the
			kernel will manage it correctly. */
			xTimeToWake = xConstTickCount + xTicksToWait;

			/* The list item will be inserted in wake time order. */
			listSET_LIST_ITEM_VALUE( &( pxCurrentTCB->xStateListItem ), xTimeToWake );

			if( xTimeToWake < xConstTickCount )
			{
				/* Wake time has overflowed.  Place this item in the overflow
				list. */
				vListInsert( pxOverflowDelayedTaskList, &( pxCurrentTCB->xStateListItem ) );
			}
			else
			{
				/* The wake time has not overflowed, so the current block list
				is used. */
				vListInsert( pxDelayedTaskList, &( pxCurrentTCB->xStateListItem ) );

				/* If the task entering the blocked state was placed at the
				head of the list of blocked tasks then xNextTaskUnblockTime
				needs to be updated too. */
				if( xTimeToWake < xNextTaskUnblockTime )
				{
					xNextTaskUnblockTime = xTimeToWake;
				}
				else
				{
					mtCOVERAGE_TEST_MARKER();
				}
			}
		}
	}
	#else /* INCLUDE_vTaskSuspend */
	{
		/* Calculate the time at which the task should be woken if the event
		does not occur.  This may overflow but this doesn't matter, the kernel
		will manage it correctly. */
		xTimeToWake = xConstTickCount + xTicksToWait;

		/* The list item will be inserted in wake time order. */
		listSET_LIST_ITEM_VALUE( &( pxCurrentTCB->xStateListItem ), xTimeToWake );

		if( xTimeToWake < xConstTickCount )
		{
			/* Wake time has overflowed.  Place this item in the overflow list. */
			vListInsert( pxOverflowDelayedTaskList, &( pxCurrentTCB->xStateListItem ) );
		}
		else
		{
			/* The wake time has not overflowed, so the current block list is used. */
			vListInsert( pxDelayedTaskList, &( pxCurrentTCB->xStateListItem ) );

			/* If the task entering the blocked state was placed at the head of the
			list of blocked tasks then xNextTaskUnblockTime needs to be updated
			too. */
			if( xTimeToWake < xNextTaskUnblockTime )
			{
				xNextTaskUnblockTime = xTimeToWake;
			}
			else
			{
				mtCOVERAGE_TEST_MARKER();
			}
		}

		/* Avoid compiler warning when INCLUDE_vTaskSuspend is not 1. */
		( void ) xCanBlockIndefinitely;
	}
	#endif /* INCLUDE_vTaskSuspend */
}

void xPortSysTickHandler( void )
{
	/* The SysTick runs at the lowest interrupt priority, so when this interrupt
	executes all interrupts must be unmasked.  There is therefore no need to
	save and then restore the interrupt mask value as its value is already
	known - therefore the slightly faster vPortRaiseBASEPRI() function is used
	in place of portSET_INTERRUPT_MASK_FROM_ISR(). */
	vPortRaiseBASEPRI();
	{
		/* Increment the RTOS tick. */
		if( xTaskIncrementTick() != pdFALSE )
		{
			/* A context switch is required.  Context switching is performed in
			the PendSV interrupt.  Pend the PendSV interrupt. */
			portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;
		}
	}
	vPortClearBASEPRIFromISR();
}


BaseType_t xTaskIncrementTick( void )
{
TCB_t * pxTCB;
TickType_t xItemValue;
BaseType_t xSwitchRequired = pdFALSE;

	/* Called by the portable layer each time a tick interrupt occurs.
	Increments the tick then checks to see if the new tick value will cause any
	tasks to be unblocked. */
	traceTASK_INCREMENT_TICK( xTickCount );
	if( uxSchedulerSuspended == ( UBaseType_t ) pdFALSE )
	{
		/* Minor optimisation.  The tick count cannot change in this
		block. */
		const TickType_t xConstTickCount = xTickCount + ( TickType_t ) 1;

		/* Increment the RTOS tick, switching the delayed and overflowed
		delayed lists if it wraps to 0. */
		xTickCount = xConstTickCount;

		if( xConstTickCount == ( TickType_t ) 0U ) /*lint !e774 'if' does not always evaluate to false as it is looking for an overflow. */
		{
			taskSWITCH_DELAYED_LISTS();
		}
		else
		{
			mtCOVERAGE_TEST_MARKER();
		}

		/* See if this tick has made a timeout expire.  Tasks are stored in
		the	queue in the order of their wake time - meaning once one task
		has been found whose block time has not expired there is no need to
		look any further down the list. */
		if( xConstTickCount >= xNextTaskUnblockTime )
		{
			for( ;; )
			{
				if( listLIST_IS_EMPTY( pxDelayedTaskList ) != pdFALSE )
				{
					/* The delayed list is empty.  Set xNextTaskUnblockTime
					to the maximum possible value so it is extremely
					unlikely that the
					if( xTickCount >= xNextTaskUnblockTime ) test will pass
					next time through. */
					xNextTaskUnblockTime = portMAX_DELAY; /*lint !e961 MISRA exception as the casts are only redundant for some ports. */
					break;
				}
				else
				{
					/* The delayed list is not empty, get the value of the
					item at the head of the delayed list.  This is the time
					at which the task at the head of the delayed list must
					be removed from the Blocked state. */
					pxTCB = listGET_OWNER_OF_HEAD_ENTRY( pxDelayedTaskList ); /*lint !e9079 void * is used as this macro is used with timers and co-routines too.  Alignment is known to be fine as the type of the pointer stored and retrieved is the same. */
					xItemValue = listGET_LIST_ITEM_VALUE( &( pxTCB->xStateListItem ) );

					if( xConstTickCount < xItemValue )
					{
						/* It is not time to unblock this item yet, but the
						item value is the time at which the task at the head
						of the blocked list must be removed from the Blocked
						state -	so record the item value in
						xNextTaskUnblockTime. */
						xNextTaskUnblockTime = xItemValue;
						break; /*lint !e9011 Code structure here is deedmed easier to understand with multiple breaks. */
					}
					else
					{
						mtCOVERAGE_TEST_MARKER();
					}

					/* It is time to remove the item from the Blocked state. */
					( void ) uxListRemove( &( pxTCB->xStateListItem ) );

					/* Is the task waiting on an event also?  If so remove
					it from the event list. */
					if( listLIST_ITEM_CONTAINER( &( pxTCB->xEventListItem ) ) != NULL )
					{
						( void ) uxListRemove( &( pxTCB->xEventListItem ) );
					}
					else
					{
						mtCOVERAGE_TEST_MARKER();
					}

					/* Place the unblocked task into the appropriate ready
					list. */
					prvAddTaskToReadyList( pxTCB );

					/* A task being unblocked cannot cause an immediate
					context switch if preemption is turned off. */
					#if (  configUSE_PREEMPTION == 1 )
					{
						/* Preemption is on, but a context switch should
						only be performed if the unblocked task has a
						priority that is equal to or higher than the
						currently executing task. */
						if( pxTCB->uxPriority >= pxCurrentTCB->uxPriority )
						{
							xSwitchRequired = pdTRUE;
						}
						else
						{
							mtCOVERAGE_TEST_MARKER();
						}
					}
					#endif /* configUSE_PREEMPTION */
				}
			}
		}

		/* Tasks of equal priority to the currently running task will share
		processing time (time slice) if preemption is on, and the application
		writer has not explicitly turned time slicing off. */
		#if ( ( configUSE_PREEMPTION == 1 ) && ( configUSE_TIME_SLICING == 1 ) )
		{
			if( listCURRENT_LIST_LENGTH( &( pxReadyTasksLists[ pxCurrentTCB->uxPriority ] ) ) > ( UBaseType_t ) 1 )
			{
				xSwitchRequired = pdTRUE;
			}
			else
			{
				mtCOVERAGE_TEST_MARKER();
			}
		}
		#endif /* ( ( configUSE_PREEMPTION == 1 ) && ( configUSE_TIME_SLICING == 1 ) ) */

		#if ( configUSE_TICK_HOOK == 1 )
		{
			/* Guard against the tick hook being called when the pended tick
			count is being unwound (when the scheduler is being unlocked). */
			if( uxPendedTicks == ( UBaseType_t ) 0U )
			{
				vApplicationTickHook();
			}
			else
			{
				mtCOVERAGE_TEST_MARKER();
			}
		}
		#endif /* configUSE_TICK_HOOK */
	}
	else
	{
		++uxPendedTicks;

		/* The tick hook gets called at regular intervals, even if the
		scheduler is locked. */
		#if ( configUSE_TICK_HOOK == 1 )
		{
			vApplicationTickHook();
		}
		#endif
	}

	#if ( configUSE_PREEMPTION == 1 )
	{
		if( xYieldPending != pdFALSE )
		{
			xSwitchRequired = pdTRUE;
		}
		else
		{
			mtCOVERAGE_TEST_MARKER();
		}
	}
	#endif /* configUSE_PREEMPTION */

	return xSwitchRequired;
}
```

> 未完待续…
> 
> 实时操作系统系列将持续更新
> 
> 创作不易希望朋友们点赞，转发，评论，关注。
> 
> 您的点赞，转发，评论，关注将是我持续更新的动力
> 
> 作者：李巍
> 
> Github：liyinuoman2017
