# 中山大学本科生实验报告
课程名称：Operation System
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
<br />
## Project1 Thread
### 1. 实验目的
* 通过修改pintos的线程休眠函数来保证pintos不会在一个线程休眠时忙等待。
* 通过修改pintos排队的方式来使得所有线程按优先级正确地被唤醒。

### 2. 实验过程
#### （一）Test 源文件分析
* **alarm single & alarm multiple：测试线程是否以正确的时间顺序被唤醒。**
```c
void
test_alarm_single (void) 
{
	test_sleep (5, 1);
}
void
test_alarm_multiple (void) 
{
	test_sleep (5, 7);
}
```
这两个测试均调用 `test_sleep()` 来测试，只是传的参数不一样。两个参数分别为 `thread_cnt` 和 `iterations` ，分别表示 **线程数量** 和 **休眠次数** 。    
在 `test_sleep()` 中，我们会创造 `thread_cnt` 个休眠 `iterations` 次的线程，第k个线程每次休眠(k+1)×10 ticks。
```c
  for (i = 0; i < thread_cnt; i++)
    {
      struct sleep_thread *t = threads + i;
      char name[16];
      
      t->test = &test;
      t->id = i;
      t->duration = (i + 1) * 10;
      t->iterations = 0;

      snprintf (name, sizeof name, "thread %d", i);
      thread_create (name, PRI_DEFAULT, sleeper, t);
    }
```
在线程执行完毕后，我们测试 **thread_cnt × iterations** （即唤醒时间）的不减性。
```c
  product = 0;
  for (op = output; op < test.output_pos; op++) 
    {
      struct sleep_thread *t;
      int new_prod;

      ASSERT (*op >= 0 && *op < thread_cnt);
      t = threads + *op;

      new_prod = ++t->iterations * t->duration;
      if (new_prod >= product)
        product = new_prod;
      else
        fail ("thread %d woke up out of order (%d > %d)!",
              t->id, product, new_prod);
    }
```
以及检测每个线程被唤醒的次数是否为 `thread_cnt` 。
```c
  for (i = 0; i < thread_cnt; i++)
    if (threads[i].iterations != iterations)
      fail ("thread %d woke up %d times instead of %d",
            i, threads[i].iterations, iterations);
```
期待的运行的结果：single中创建5个线程，每个线程唤醒一次，唤醒时间等于休眠时间。
```
(alarm-single) thread 0: duration=10, iteration=1, product=10
(alarm-single) thread 1: duration=20, iteration=1, product=20
(alarm-single) thread 2: duration=30, iteration=1, product=30
(alarm-single) thread 3: duration=40, iteration=1, product=40
(alarm-single) thread 4: duration=50, iteration=1, product=50
```
multiple中创建5个进程，每次休眠时间分别为10/20/30/40/50，每个线程唤醒7次，每次唤醒时间为休眠时间与唤醒次数的积。
```
(alarm-multiple) thread 0: duration=10, iteration=1, product=10
(alarm-multiple) thread 0: duration=10, iteration=2, product=20
(alarm-multiple) thread 1: duration=20, iteration=1, product=20
(alarm-multiple) thread 0: duration=10, iteration=3, product=30
(alarm-multiple) thread 2: duration=30, iteration=1, product=30
...
(alarm-multiple) thread 3: duration=40, iteration=7, product=280
(alarm-multiple) thread 4: duration=50, iteration=6, product=300
(alarm-multiple) thread 4: duration=50, iteration=7, product=350
```
* **alarm-simultaneous：测试应该同时被唤醒的线程是否能在同一个tick、按照创建顺序唤醒。**
```c
void
test_alarm_simultaneous (void) 
{
	test_sleep (3, 5);
}
```
使用 `test_sleep()` 测试，传入两个参数分别为 `thread_cnt` 和 `iterations` ，分别表示 **线程数量** 和 **休眠次数** 。    
在 `test_sleep()` 中，我们会创造 `thread_cnt` 个休眠 `iterations` 次的线程，执行 `sleeper()` 函数。在 ` sleeper()` 中，线程在 **开始时间+10×k tick** 的时候被第k次唤醒。
```c
  for (i = 1; i <= test->iterations; i++) 
    {
      int64_t sleep_until = test->start + i * 10;
      timer_sleep (sleep_until - timer_ticks ());
      *test->output_pos++ = timer_ticks () - test->start;
      thread_yield ();
    }
```
期待运行结果：创建了3个线程，每个线程均在10/20/30/40tick的时候被按照创建顺序唤醒。
```
(alarm-simultaneous) iteration 0, thread 0: woke up after 10 ticks
(alarm-simultaneous) iteration 0, thread 1: woke up 0 ticks later
(alarm-simultaneous) iteration 0, thread 2: woke up 0 ticks later
(alarm-simultaneous) iteration 1, thread 0: woke up 10 ticks later
...
(alarm-simultaneous) iteration 3, thread 2: woke up 0 ticks later
(alarm-simultaneous) iteration 4, thread 0: woke up 10 ticks later
(alarm-simultaneous) iteration 4, thread 1: woke up 0 ticks later
(alarm-simultaneous) iteration 4, thread 2: woke up 0 ticks later
```
* **alarm-priority：测试线程是否按照优先级被唤醒。**   
首先这里生成了一个初始值为0的信号量，和10个线程，他们的优先级是先增后减的。线程生成后，他们会执行 `alarm_priority_thread()` 函数。
```c
  sema_init (&wait_sema, 0);
  for (i = 0; i < 10; i++) 
    {
      int priority = PRI_DEFAULT - (i + 5) % 10 - 1;
      char name[16];
      snprintf (name, sizeof name, "priority %d", priority);
      thread_create (name, priority, alarm_priority_thread, NULL);
    }
```
然后我们的测试线程会将自己的优先级降到最低，并请求10个信号量。这里实现了测试线程的阻塞。
```c
  thread_set_priority (PRI_MIN);
  for (i = 0; i < 10; i++)
    sema_down (&wait_sema);
```
在 `alarm_priority_thread()` 中，我们让线程休眠到 `wake_time` 后唤醒。然后释放一个信号量。10个信号量都释放后，测试线程才能继续执行。
```c
  timer_sleep (wake_time - timer_ticks ());
  sema_up (&wait_sema);
```
在 `wake_time` 时，所有线程被唤醒，此时按照线程优先级顺序唤醒，优先级高的先被唤醒。故期待运行结果：
```
(alarm-priority) Thread priority 30 woke up.
(alarm-priority) Thread priority 29 woke up.
(alarm-priority) Thread priority 28 woke up.
(alarm-priority) Thread priority 27 woke up.
(alarm-priority) Thread priority 26 woke up.
(alarm-priority) Thread priority 25 woke up.
(alarm-priority) Thread priority 24 woke up.
(alarm-priority) Thread priority 23 woke up.
(alarm-priority) Thread priority 22 woke up.
(alarm-priority) Thread priority 21 woke up.
```
* **alarm-zero & alram-negative：测试休眠时间为0、为负时是否会运行时错误**
```c
void
test_alarm_zero (void) 
{
	timer_sleep (0);
	pass ();
}
void
test_alarm_negative (void) 
{
	timer_sleep (-100);
	pass ();
}
```
期待运行结果：
```
(alarm-zero) PASS
(alarm-negative) PASS
```

#### （二）实验思路与代码分析
* **通过修改pintos的线程休眠函数来保证pintos不会在一个线程休眠时忙等待。**  
在pintos中，我们可以主动让当前RUNNING状态的线程休眠，即放弃RUNNING状态。   
用于处理进程休眠的函数为 `timer_sleep()` 系统已经原生实现。其中，传入的参数 `ticks` 表示当前进程需要休眠的时间。进入函数时获得进入时间 `start` ，调用 `timer_elapsed(start)` 获得距离开始休眠已经过去的时间，当 `timer_elapsed(start) < ticks` 时，我们执行 `thread_yield()` 。
```c
void
timer_sleep (int64_t ticks)
{
	int64_t start = timer_ticks ();

	ASSERT (intr_get_level () == INTR_ON);
	while (timer_elapsed (start) < ticks)
		thread_yield ();
}
```
在 `thread_yield()` 中，我们首先获得当前进程，然后进行一个原子操作。原子操作中，我们将当前进程加入ready序列，并将其状态设为READY，然后执行 `schedule()` 调度ready序列中的进程。
```c
void
thread_yield (void)
{
	struct thread *cur = thread_current ();
	enum intr_level old_level;

	ASSERT (!intr_context ());

	old_level = intr_disable ();
	if (cur != idle_thread)
		list_push_back (&ready_list, &cur->elem);
	cur->status = THREAD_READY;
	schedule ();
	intr_set_level (old_level);
}
```
在这里，我猜 `schedule()` 是异步的，我们调用后会直接返回到 `timer_sleep()` 判断上一个进程是否继续休眠（如果我猜错了请忽略这句话）。所以在 `timer_sleep()` 中的while循环形成了一个忙等待，使得在每次运行完 `thread_yield()` 后，都需要检查一次条件 `timer_elapsed(start) < tick` 是否为真。但是正如我们所知道的，在ticks的时间内，这个条件将一直为假，故反复检查造成了CPU资源浪费。   
为了解决这个问题，我们可以直接阻塞需要休眠的线程，当它需要被唤醒的时候，再让他进入ready队列。   
首先，我们在线程中引入一个变量 `int64_t tick_blocked` 来表示这个线程需要继续休眠的时间，并在 `init_thread()` 中将它初始化为0，然后我们就可以通过判断 `tick_blocked == 0` 是否为真来决定是否将当前休眠的进程加入ready队列。于是我们写一个 `blocked_thread_check()` 函数来做这个判断，并同时完成 `tick_blocked` 的递减。 
```c
void blocked_thread_check(struct thread *t, void *aux UNUSED) {
	if (t->status == THREAD_BLOCKED && t->tick_blocked > 0) {
		t->tick_blocked --;
		if (t->tick_blocked == 0) {
			thread_unblock(t);
		}
	}
}
```
由于休眠时间是以tick为单位，所以我们在每次中断的时候做如上判断。pintos已经给我们写好了一个遍历线程的函数 `thread_foreach()` ，那么我们只需在每次中断的时候遍历线程并调用上面写好的判断函数即可。
```c
static void timer_interrupt(struct intr_frame *args UNUSED) {
    ticks ++;
    thread_tick ();
    thread_foreach(blocked_thread_check, NULL);
}
```
最后，我们在 `timer_sleep()` 中给当前进程的 `tick_blocked` 赋值，并阻塞需要休眠的进程即可。另外，由于 `tick_blocked` 不能为负，所以我们需要对传入的 `ticks`进行正数判断。
```c
void timer_sleep (int64_t ticks) {
	if (ticks <= 0) return;
	enum intr_level old_level = intr_disable ();
	struct thread *current_thread = thread_current ();
	current_thread->tick_blocked = ticks;
	thread_block ();
	intr_set_level (old_level);
}
```
* **通过修改pintos排队的方式来使得所有线程按优先级正确地被唤醒。**  
在pintos的原生实现中，是通过FIFO进行线程调度的。即，将线程加入ready序列时，直接放入序列尾部，调度线程时，直接取ready序列的头部。例如， `thread_yield()` 是这样实现的：
```c
void
thread_yield (void) 
{
    ...
    if (cur != idle_thread) 
        list_push_back (&ready_list, &cur->elem);
    ...
}
```
而 `next_thread_to_run()` 是这样实现的：
```c
static struct thread *
next_thread_to_run (void) 
{
	if (list_empty (&ready_list))
		return idle_thread;
	else
    	return list_entry (list_pop_front (&ready_list), struct thread, elem);
}
```
现在我们想要让线程按照优先级顺序出队列，有两个选择：插入线程时让线程按照优先级顺序排在正确位置、拿出线程时对线程进行排序。   
实际上两种方法需要使用到的函数pintos都已经给我们写好了。
```c
void list_insert_ordered (struct list *, struct list_elem *, list_less_func *, void *aux);
void list_sort (struct list *, list_less_func *, void *aux);
```
我们这里使用第一种方法来实现。   
首先我们写一个比较函数。这个函数需要加在 **thread.c** 中，不能加在 **list.c** 中，否则make的时候会 **error: dereferencing pointer to incomplete typr 'struct thread'**，原因是 **list.c** 并没有 `include "thread.h"`。
```c
bool cmp (const struct list_elem *a, const struct list_elem *b, void *aux) {
	return (list_entry(a, struct thread, elem)->priority) > (list_entry(b, struct thread, elem)->priority);
}
```
然后在插入线程进ready队列的时候调用 `list_insert_ordered (&ready_list, &t->elem, cmp, NULL)`。   
需要进行插入的函数有三个：创建线程的 `init_thread ()` 函数、将RUNNING线程放回READY的 `thread_yield ()` 函数、解除阻塞的 `thread_unblock()` 函数。我们分别修改他们。
```c
static void init_thread (struct thread *t, const char *name, int priority) {
	...
	list_insert_ordered (&all_list, &t->allelem, cmp, NULL);
}
void thread_yield (void) {
	...
	if (cur != idle_thread) 
		list_insert_ordered (&ready_list, &cur->elem, cmp, NULL);
	...
}
void thread_unblock (struct thread *t) {
	...
	list_insert_ordered (&ready_list, &t->elem, cmp, NULL);
	...
}
```
至此，我们保证了ready队列的优先级有序性，使得线程可以按照优先级唤醒。   
![](/home/airy/圖片/project1-3.png)
 
### 3. 实验结果
![](/home/airy/圖片/project1.png)   
&   
![](/home/airy/圖片/project1-2.png)   

### 4. 回答问题
* **之前为什么20/27？为什么那几个test在什么都没有修改的时候还过了？**   
pintos原生已经正确实现了一些功能，例如休眠函数 `timer_sleep()` 。这些功能是可以正常使用的，只是需要优化。

* **intr_disable()返回值是什么？为什么还要intr_set_level()函数？**   
返回值是当前的中断状态，我们需要的原子操作结束后需要恢复之前的状态。如果一直处于不能中断的状态，系统无法获得CPU时间，无法调度进程。 

* **什么情况下schedule调度的还是当前线程？**   
ready序列为空的时候。

* **为什么线程休眠要保证中断打开？**    
系统通过当前tick判断是否需要唤醒线程，如果中断关闭，则无法使tick递增，则休眠线程无法唤醒。

### 5. 实验感想
作为第一次实验，过程还是很艰难的，因为对系统源码十分陌生，并且源码里有大量底层的东西，阅读起来十分困难。TA的课件翻来覆去看了很多次，才大概明白了我们要写的东西的原理和过程。   

首先在分析test源码的时候，我并没有找到 **alarm-single.c** 和 **alarm-multiple.c** ，结果test是把这两个写在了同一个测试文件叫 **alarm-wait.c** 里面的，调用的也是同样的函数，只是传了不同的参数。    

然后看测试文件 **alarm-priority.c** 的时候，我虽然知道它想干什么，但是并不是很懂为什么要在这里引入信号量。查阅了资料发现他是为了阻塞主线程。首先创建了一个初始值为0的信号量，然后循环请求信号量，由于初始值为0，那么测试程序会阻塞在这里。然后在每个创建出来的线程调用的函数末尾释放一个信号量。当测试线程成功得到所有信号量时，便能解除测试线程的阻塞。   

在修改 `timer_sleep()` 函数时，我发现其实要理解这里为什么形成忙等待并不是一件容易的事。虽然作为一个循环判断，确实是一个很明显的忙等待的形式，但是具体为什么系统会反复检查、在什么时候会检查，TA并没有完全讲清楚。查阅了网上的资料发现大家也都讲的比较含糊，所以我自己就简单做了一个 `schedule()` 是异步调用的猜测，毕竟作为一个单线程的系统，多半是要靠异步来提高效率的。（如果我错了，当我没说。）   

在实现线程插入的时候， `cmp()` 的位置最开始没有放对，所以导致编译一直不过，后来发现是因为 **list.c** 并没有 `include "thread.h"`，所以并不能获得当前进程的优先级，所以 `cmp()` 函数并不能放在 **list.c** 里面。然后在修改 `list_push_back()` 的时候，我最初只修改了 `thread_unblock()`，后来发现在另外两个函数中也会有插入操作，而原生实现也是直接放进队尾，所以也需要修改。由此可见，细节还是很重要的。    

由于年轻的时候在截图代码的时候经常怀疑人生，所以这次的实验报告是使用Markdown来写的，实验过程中也复习了一下Markdown的很多语法。Markdown虽然没有LaTeX万能，但是还是很好用的。    

也顺便复习了一下git，使用git仓库来控制代码版本，虽然好像暂时还没派上用场。   