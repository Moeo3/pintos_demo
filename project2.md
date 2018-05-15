# 中山大学本科生实验报告
课程名称：Operating System
任课教师：饶洋辉

<table class="table table-bordered table-striped table-condensed">
  <tr>
    <td>年级+班级</td>
    <td>16信息与计算科学</td>
    <td>专业（方向）</td>
    <td>信息与计算科学</td>
  </tr>
  <tr>
    <td>学号</td>
    <td>16339051</td>
    <td>姓名</td>
    <td>张倩如</td>
  </tr>
</table>

## Project2 优先级抢占制度
### 1. 实验目的
* 实现优先级的抢占调度，了解抢占制度的优点
* 理解并发，以及信号量与互斥锁

### 2.实验过程
#### （一）Test 源文件分析
* __priority-fifo：测试CPU运行线程优先级变化时，ready队列中高于变化后优先级的线程是否能抢占CPU。__   
在源码中，我们先把测试线程的优先级设为 __PRI_DEFAULT + 2__ :
```c
thread_set_priority (PRI_DEFAULT + 2);
```
然后创建 __THREAD_CNT__ 个优先级为 __PRI_DEFAULT + 1__ 的线程，他们执行 `simple_thread_func()` 函数。
```c
thread_create (name, PRI_DEFAULT + 1, simple_thread_func, d);
```
在 `simple_thread_func()` 中，将运行线程写入结果，并执行 `thread_yield ()`，将当前线程加入ready序列，并调度ready序列中的线程。
```c
for (i = 0; i < ITER_CNT; i++) 
    {
      lock_acquire (data->lock);
      *(*data->op)++ = data->id;
      lock_release (data->lock);
      thread_yield ();
    }
}
```
然后，我们将测试线程优先级改为 __PRI_DEFAULT__ ，低于刚才创建的线程，则测试线程被抢占，执行新线程。
```
thread_set_priority (PRI_DEFAULT);
```
现在是无法运行出结果的：
```shell
run: TIMEOUT after 60 seconds of host CPU time
```
期望的输出，迭代16次，16个线程依次输出：
```
(priority-fifo) begin
(priority-fifo) 16 threads will iterate 16 times in the same order each time.
(priority-fifo) If the order varies then there is a bug.
(priority-fifo) iteration: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
(priority-fifo) iteration: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
(priority-fifo) iteration: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
...
(priority-fifo) iteration: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
(priority-fifo) iteration: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
(priority-fifo) end
```

* __priority-preempt：测试创造高优先级的线程时，这个线程是否可以抢占当前线程。__  
首先，我们的测试线程优先级为默认优先级 __PRI_DEFAULT__ 。现在我们创造一个优先级为 __PRI_DEFAULT + 1__ 的线程 `high-priority`，并执行 `simple_thread_func()`。
```c
thread_create ("high-priority", PRI_DEFAULT + 1, simple_thread_func, NULL);
```
在 `simple_thread_func()` 中，这个线程会迭代5次，即放入ready队列5次，由于优先级高于测试线程，所以每次都会调度这个线程。每一次迭代，都会输出 __Thread high-priority iteration k__ ，k为迭代次数。迭代完成后，会输出 __Thread high-priority done!__ 。  
```c
  for (i = 0; i < 5; i++) 
    {
      msg ("Thread %s iteration %d", thread_name (), i);
      thread_yield ();
    }
  msg ("Thread %s done!", thread_name ());
```
现在的输出，创建出的线程并没能抢占CPU：
```
(priority-preempt) begin
(priority-preempt) The high-priority thread should have already completed.
(priority-preempt) end
```
期待的输出，新线程抢占CPU并迭代5次：
```
  (priority-preempt) begin
  (priority-preempt) Thread high-priority iteration 0
  (priority-preempt) Thread high-priority iteration 1
  (priority-preempt) Thread high-priority iteration 2
  (priority-preempt) Thread high-priority iteration 3
  (priority-preempt) Thread high-priority iteration 4
  (priority-preempt) Thread high-priority done!
  (priority-preempt) The high-priority thread should have already completed.
  (priority-preempt) end
```

* __priority-change：测试创建线程和修改优先级时是否能成功抢占。__  
首先，我们的测试线程优先级为默认优先级 __PRI_DEFAULT__ 。然后创建出一个优先级为 __PRI_DEFAULT + 1__ 的线程`thread 2`，并运行 `changing_thread()` 函数。
```c
thread_create ("thread 2", PRI_DEFAULT + 1, changing_thread, NULL);
```
在 `changing_thread()` 中，会将`thread 2`的优先级改为 __PRI_DEFAULT - 1__，此时CPU将被测试线程抢占，输出 __Thread 2 should have just lowered its priority.__。
然后又将测试线程的优先级改为 __PRI_DEFAULT - 2__，此时CPU将被thread 2抢占，输出 __Thread 2 exiting.__。
```c
thread_set_priority (PRI_DEFAULT - 2);
```
然后thread 2运行完毕，调度回主线程（输出 __Thread 2 should have just exited.__ ）。  
现在的输出， `thread 2` 并没有成功抢占：
```
  (priority-change) begin
  (priority-change) Creating a high-priority thread 2.
  (priority-change) Thread 2 should have just lowered its priority.
  (priority-change) Thread 2 should have just exited.
  (priority-change) end
```
期待的输出，`thread 2` 先抢占然后降低优先级，又被测试线程抢占，然后测试线程降低优先级，`thread 2` 抢占，然后运行完 `thread 2`，回到测试线程：
```
  (priority-change) begin
  (priority-change) Creating a high-priority thread 2.
  (priority-change) Thread 2 now lowering priority.
  (priority-change) Thread 2 should have just lowered its priority.
  (priority-change) Thread 2 exiting.
  (priority-change) Thread 2 should have just exited.
  (priority-change) end
```
#### （二） 实验思路与代码分析
* __重新设计 thread_set_priority() 函数__   
伪代码：
```
THREAD_SET_PRIOERTY(new_priority)
begin
	thread_current.priority ← new_priority
    if mew_priority < list_max
    	then do THREAD_YIELD()
    end if
end
```
——   
在原生的 `thread_set_priority()` 中，只改变了当前线程的优先级。
```c
void
thread_set_priority (int new_priority) 
{
	thread_current ()->priority = new_priority;
}
```
而我们需要在我们修改当前线程的优先级的时候，需要加入一个判断。如果新的优先级低于了ready队列最高优先级的线程的优先级，那么要让当前线程退出CPU，并调度ready队列里的线程，即调用 `thread_yield()` 。
```c
void thread_set_priority (int new_priority) {
	thread_current()->priority = new_priority;
	struct list_elem *max= list_max(&ready_list, cmp, NULL);
	if (new_priority < list_entry(max, struct thread, elem)->priority) {
		thread_yield();
	}
}
```
*是否还有别的方法完成？*  
其实，由于上一次实验我们已经修改了pintos的排队机制，所以ready队列中优先级最高的线程一定排在队首，所以我们直接取出队首来判断优先级就可以了。
```c
void thread_set_priority (int new_priority) {
	thread_current()->priority = new_priority;
	if (!list_empty(&ready_list) && new_priority < list_entry(list_begin (&ready_list), struct thread, elem)->priority)
    thread_yield ();
}
```
或者，我们在这里不做判断，直接调用 `thread_yield()` 将当前进程放入ready队列，然后进行调度，也不是不可以。
```c
void thread_set_priority (int new_priority) {
	thread_current()->priority = new_priority;
	thread_yield();
}
```

* __在 thread_create() 函数内检测是否需要抢占CPU__   
伪代码：
```
THREAD_CREATE(name_, priority_, func, aux)
begin
	(something initialize)
    ...
    THREAD_UNBLOCK (t)
    if priority_ > thread_current.priority
    	then do THREAD_YIELD()
    end if
    return tid
end
```
——  
在原先的thread_create()中，创建后直接将线程加入ready序列，没有判断是否需要抢占。
```c
tid_t
thread_create (const char *name, int priority,
               thread_func *function, void *aux) 
{
	...
	/* Add to run queue. */
	thread_unblock (t);
    
	return tid;
}
```
我们需要在创建最后加入判断。如果创建的这个进程优先级高于正在执行的进程，那么我们就让当前线程退出CPU，并调度这个进程。在这里，我们也可以直接调用 `thread_yield()` 来完成。所以我们在return前加上：
```c
  if (priority > thread_current()->priority) {
  	thread_yield();
  }
```
至此，我们实现了优先级的抢占制度。

### 3. 实验结果
![](/home/airy/圖片/project2-1.png)   
单独跑每个test：   
* priority-change  
![](/home/airy/圖片/project2-4.png)
* priority-fifo  
![](/home/airy/圖片/project2-5.png)  
* priority-preempt   
![](/home/airy/圖片/project2-6.png)   

### 4. 回答问题
* __如果不考虑修改 thread_create() 函数的情况，test 能通过吗？如果不能，会出现什么结果（请截图），并解释为什么会出现这个结果。__  
① 首先 __priority-fifo__ 这个测试，它是能过的。因为它根本没调用 `thread_create()` 。   
② 对于 __priority-preempt__ ，如果我们不修改 `thread_create()` ，它的输出和没改代码前是一样的。  
创建出的 `high-priority` 虽然被放在了ready队列里，但是需要等待测试线程主动放弃CPU才能被调用，而测试线程结束时系统会直接退出，所以 `high-priority` 并没有机会占用CPU，自然也不会有相关输出。  
![](/home/airy/圖片/project2-7.png)   
③ 对于 __priority-change__ ，也是不能过的。   
我们创建了优先级为 __PRI_DEFAULT + 1__ 的 `thread 2` 以后，只会把它放入ready队列，而不会让它抢占，所以会继续执行测试线程，直到测试线程将自己的优先级改为 __PRI_DEFAULT - 2__ 时，才会调度创建出的线程。然后创建出的线程将自己优先级置为 __PRI_DEFAULT - 1__ ，此时再次调度， `thread 2` 优先级高，所以继续执行。 `thread 2` 退出后，回到测试线程，再结束测试。也就是说，运行 `thread 2` 的时间会推后。   
![](/home/airy/圖片/project2-8.png)   

* __用自己的话阐述 Pintos 中的 semaphore 和 lock 的区别与联系。__   
首先这两个东西，是用来 __并发控制__ 的。优先级抢占会产生 __并发__，__并发__ 会出现的主要问题是资源的竞争，这就需要线程制约，制约关系有两种，__间接制约__ 和 __直接制约__，分别对应 __互斥__ 与 __同步__。   
__互斥__ 是指线程临界区不可以同时占用一个资源，即线程间互相排斥，但并不限制线程对其的访问顺序。而 __同步__ 是需要各个线程临界区按照某种次序来分别访问资源，线程间互相依赖。  
具体来说，__lock__ 可以实现 __互斥__，而 __semaphore__ 可以实现 __互斥__ 与 __同步__。  
——     
先说 __semaphore__，也就是 __信号量__ 。  
信号量限制了资源被有限个线程占用。有多少信号量就可以再容纳多少个线程，如果没有剩余信号量了，那么请求信号量的线程就需要等待，也就是被阻塞了。   
在具体的实现中，我们需要一个等待序列，这个序列里面放被阻塞的线程。  
我们对于信号量有两种操作（都是原子操作）：   
__P操作__：sema_down，减少信号量，如果信号量为0，则加入等待序列。  
__V操作__：sema_up，增加信号量，如果信号量为正，则唤醒等待队列中第一个进程。  
<del>这两个操作的名字是哪里来的呢？因为伟大的信号量发明者Dijkstra（就是最短路算法的那个）是个荷兰人。P是指荷兰语中的“proberen”（测试），V指荷兰语中的“verhogen”（增加）。</del>     
通过这两个操作，我们就可以人为来协调各个线程，让它们有限制性地使用公共资源。   
初始化信号量的时候，我们一般初始化为0。  
需要 __同步__ 时，P操作相当于确定当前线程依赖的线程是否已经访问资源完毕，而V操作相当于发送信号表示当前进程已经完成了某项任务，依赖于当前进程的进程可以开始运行了。   
需要 __互斥__ 时，P操作相当于当前线程确认需要的资源没有被占用，而V操作表示当前线程已经使用资源完毕，可以给其他线程使用了。   
——   
再说 __lock__，也就是 __锁__，在pintos中，锁通过 __binary semaphore（二值信号量）__ 完成，是一种 __mutex（互斥锁）__。也就是说，__lock__ 可以任务是 __semaphore__ 的一种特殊运用。  
首先初始化锁的时候，会把信号量设为1。当信号量为1的时候，表示锁可以使用，信号量为0的时候，表示锁被占用。   
具体来说，如果有线程需要请求锁，那么就执行一次P操作，当线程释放锁的时候，执行一次V操作。   
比较特殊的是，pintos中释放锁的必须是请求锁的线程，也就是说，__P操作和V操作会成对出现__。这保证了绝对的 __互斥__，假如某一线程占用一个资源，一定会等到它放弃占用这个资源，才会有其他线程来占用。  
——   
综上，__semaphore__ 与 __lock__ 的主要区别在于 __①功能 ②初值 ③对称性__ 。联系在于，__lock__ 是基于 __semaphore__ 实现的，两者都是sleep-waiting类型。

* __考虑优先级抢占调度后，重新分析 alarm-priority 测试。__  
首先，我们设置了一个唤醒时间，所有线程都会在 `wake_time` 唤醒。
```c
wake_time = timer_ticks () + 5 * TIMER_FREQ;
```
然后初始化一个信号量。
```c
sema_init (&wait_sema, 0);
```
然后创建10个线程，每个线程的优先级为 __PRI_DEFAULT - (i + 5) % 10 - 1__，并让它们执行`alarm_priority_thread()`。
```c
for (i = 0; i < 10; i++) 
    {
      int priority = PRI_DEFAULT - (i + 5) % 10 - 1;
      char name[16];
      snprintf (name, sizeof name, "priority %d", priority);
      thread_create (name, priority, alarm_priority_thread, NULL);
    }
```
然后测试线程将自己的优先级改为 __PRI_MIN__，这时候子线程就会立即抢占测试线程。
```c
  thread_set_priority (PRI_MIN);
```
在 `alarm_priority_thread()` 中，我们首先用忙等待保证时间已经不是 `start_time` 了。
```c
  int64_t start_time = timer_ticks ();
  while (timer_elapsed (start_time) == 0)
    continue;
```
然后让它休眠到 `wake_time`，放入ready队列。
```c
timer_sleep (wake_time - timer_ticks ());
```
`wake_time` 的时候，所有创建的10个线程都会被唤醒，但是由于ready队列中的排队制度，所以优先级高的线程会先被唤醒。  
线程被唤醒后发送消息说自己醒了，并且释放一个信号量。
```c
  msg ("Thread %s woke up.", thread_name ());
  sema_up (&wait_sema);
```
10个线程结束后，回到测试线程执行。测试线程将刚才释放出来的10个信号量请求掉。
```c
  for (i = 0; i < 10; i++)
    sema_down (&wait_sema);
```
在之前，我们阻塞测试线程是使用的信号量，因为请求不到信号量，所以会被放进ready队列。而现在，设置完主线程的优先级，子线程就会直接抢占测试线程，测试线程在请求信号量之前就会进入ready队列。

### 5. 实验感想
这次实验TA并不像上次直接给出了代码，所以要自己去研究应该怎么调用需要使用函数。我在调用 `list_max()` 的时候，又成功地CE了，原因是第一个参数需要传入的是ready_list的地址，而不是它本身。  

在尝试使用队首元素来比较优先级的时候，我最初找到了 `list_head()` 这个函数，然后发现这个函数并不能直接用，因为head是一个空元素。然后我发现可以使用 `list_begin()` 或者 `list_front()` 来获取第一个元素，但是会产生一个panic，原因是队列只有head的时候这个两个方法是无法使用的，那么我们在使用 的时候还得判断一下队列是否为空。  

当然这次实验其实很容易，而大部分时间我都用来研究信号量与锁了。过程中我仿佛又体验了一次当年被JavaScript支配的日子。并发虽然是一个高效的做法，但是会增加资源控制难度。最初人们使用自旋锁和互斥锁来处理资源竞争，后来发现并不能很好地解决同步问题，会发生抢锁问题，所以聪明的dijkstra发明了信号量。pintos中的锁是基于信号量的互斥锁，相当于一个特殊的接口，它的优点在于更利于操控。有了这些，pintos才能实现并发，允许高优先级线程来抢占CPU。   