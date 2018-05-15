# 中山大学本科生实验报告
课程名称：Operating System  
任课教师：饶洋辉

<table>
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

## Lab3 优先级捐赠
### 1. 实验目的
* 理解锁与信号量对线程调度的影响。
* 在考虑线程锁的情况下，通过优先级捐赠，保证优先级高的线程先执行。
![](/home/airy/圖片/project3-1.png)

* 考虑优先级捐赠参与时的优先级修改。   
<table>
  <tr>
    <td> </td>
    <td>被捐赠</td>
    <td>未被捐赠</td>
  </tr>
  <tr>
    <td>新优先级 &lt;= 优先级</td>
    <td>ori_priority = new_priority</td>
    <td>priority = ori_priority = new_priority</td>
  </tr>
    <tr>
    <td>新优先级  &gt; 优先级</td>
    <td>priority = ori_priority = new_priority</td>
    <td>priority = ori_priority = new_priority</td>
  </tr>
</table>

### 2.实验过程
#### （一）Test 源文件分析
首先每个测试之前都有两个断言确定 __不是多级反馈调度队列模式__ 以及 __测试线程优先级为默认__ 。以下分析中默认这两个条件成立。
```c
  ASSERT (!thread_mlfqs);
  ASSERT (thread_get_priority () == PRI_DEFAULT);
```

* __priority-donate-one：情况3。多个线程同时请求一把锁的时候，持有锁的线程优先级将不断更新。__   
首先我们先初始化一个锁，并让主线程持有。   
然后创建一个线程 __acquire1__，优先级为 __PRI_DEFAULT + 1__，抢占执行 `acquire1_thread_func()`。
```c
  thread_create ("acquire1", PRI_DEFAULT + 1, acquire1_thread_func, &lock);
```
在 `acquire1_thread_func()` 中，线程 __acquire1__ 通过 `lock_acquire (lock)` 请求锁，然而锁被测试线程持有，所以被阻塞，并把优先级 __PRI_DEFAULT + 1__ 捐给测试线程。
回到测试线程，现在创建一个线程 __acquire2__。此时，由于测试线程优先级为 __PRI_DEFAULT__， __acquire2__ 为 __PRI_DEFAULT + 2__，所以抢占执行 `acquire2_thread_func()`。
```c
  thread_create ("acquire2", PRI_DEFAULT + 2, acquire2_thread_func, &lock);
```
在 `acquire2_thread_func()` 中，线程 __acquire2__ 请求锁，锁被测试线程持有，并把 __PRI_DEFAULT + 2__ 捐给测试线程。 
此时测试线程释放锁，优先级回到 __PRI_DEFAULT__ 。   
现在两个阻塞的进程可以拿到锁继续执行，由于优先级高于测试线程，所以测试线程被抢占，阻塞。当前阻塞的两个线程中，__acquire2__ 优先级较高，先给 __acquire2__ 锁。然后 __acquire2__ 通过 `lock_release (lock)` 又释放锁，并释放捐给 __测试线程__ 的优先级。
 __acquire2__ 执行完后，同样被阻塞的 __acquire1__ 得到了锁，此后又释放。  
现在又回到测试线程，结束测试。  
![](/home/airy/圖片/project3-4.png)  
期待输出如下：
```shell
  (priority-donate-one) begin
  (priority-donate-one) This thread should have priority 32.  Actual priority: 32.
  (priority-donate-one) This thread should have priority 33.  Actual priority: 33.
  (priority-donate-one) acquire2: got the lock
  (priority-donate-one) acquire2: done
  (priority-donate-one) acquire1: got the lock
  (priority-donate-one) acquire1: done
  (priority-donate-one) acquire2, acquire1 must already have finished, in that order.
  (priority-donate-one) This should be the last line before finishing this test.
  (priority-donate-one) end
```
现在， __acquire1__ 和 __acquire2__ 创建后立即抢占，但被阻塞。而 __测试线程__ 解锁以后只是把 __acquire1__ 放入了等待队列，然后继续执行 __测试线程__，于是 __acquire1__ 和 __acquire2__ 都没能继续执行。所以现在的输出为：
```shell
  (priority-donate-one) begin
  (priority-donate-one) This thread should have priority 32.  Actual priority: 31.
  (priority-donate-one) This thread should have priority 33.  Actual priority: 31.
  (priority-donate-one) acquire2, acquire1 must already have finished, in that order.
  (priority-donate-one) This should be the last line before finishing this test.
  (priority-donate-one) end
```

* __priority-donate-multiple：情况5。一个线程持有多把锁，任何一个锁解锁后都应该更新优先级。__   
首先初始化了两个锁a/b，并且 __测试线程__ 请求并拿到两个锁。   
此时创建线程 __A__ 与 __B__ *（为方便区分锁与线程，这里线程使用大写）*，优先级分别为 __PRI_DEFAULT + 1__、 __PRI_DEFAULT + 2__，并且他们会分别请求锁 a 与 b 。
```c
  thread_create ("a", PRI_DEFAULT + 1, a_thread_func, &a);
  thread_create ("b", PRI_DEFAULT + 2, b_thread_func, &b);
```
事实上，在创建的时候他们就会分别抢占测试线程，但是由于没有锁，所以两个都被阻塞，并把优先级捐赠给 __测试线程__ 。于是过程中，测试线程的优先级会先变为 __PRI_DEFAULT + 1__，然后变为 __PRI_DEFAULT + 2__。  
接下来 __测试线程__ 通过 `lock_release (&b)` 释放锁 b，线程 __B__ 拿到锁，并抢占执行。而此时由于 __测试线程__ 不再锁 __B__，所以优先级变为 __A__ 捐赠的 __PRI_DEFAULT + 1__ 。  
而后， __B__ 释放锁 b 并执行完毕，回到 __测试线程__ ，通过 `lock_release (&a)` 释放锁 a。此时 __A__ 拿到锁，并抢占执行，同时， __测试线程__ 优先级回到 __PRI_DEFAULT__ 。  
结束测试。   
![](/home/airy/圖片/project3-5.png)  
期待的输出：
```shell
  (priority-donate-multiple) begin
  (priority-donate-multiple) Main thread should have priority 32.  Actual priority: 32.
  (priority-donate-multiple) Main thread should have priority 33.  Actual priority: 33.
  (priority-donate-multiple) Thread b acquired lock b.
  (priority-donate-multiple) Thread b finished.
  (priority-donate-multiple) Thread b should have just finished.
  (priority-donate-multiple) Main thread should have priority 32.  Actual priority: 32.
  (priority-donate-multiple) Thread a acquired lock a.
  (priority-donate-multiple) Thread a finished.
  (priority-donate-multiple) Thread a should have just finished.
  (priority-donate-multiple) Main thread should have priority 31.  Actual priority: 31.
  (priority-doate-multiple) end
```
同上一个test，现在的 __A__ 和 __B__ 都没能继续执行：
```shell
  (priority-donate-multiple) begin
  (priority-donate-multiple) Main thread should have priority 32.  Actual priority: 31.
  (priority-donate-multiple) Main thread should have priority 33.  Actual priority: 31.
  (priority-donate-multiple) Thread b should have just finished.
  (priority-donate-multiple) Main thread should have priority 32.  Actual priority: 31.
  (priority-donate-multiple) Thread a should have just finished.
  (priority-donate-multiple) Main thread should have priority 31.  Actual priority: 31.
  (priority-donate-multiple) end
```

* __priority-donate-multiple2：情况5。一个线程持有多把锁，任何一个锁解锁后都应该更新优先级。__   
首先初始化了两个锁a/b，并且测试线程请求并拿到两个锁。   
首先我们创建优先级为 __PRI_DEFAULT + 3__ 的线程 __A__， __A__ 抢占执行 `a_thread_func()`。 
```c
thread_create ("a", PRI_DEFAULT + 3, a_thread_func, &a);
```
在`a_thread_func` 中，__A__ 请求锁 a，锁被测试线程持有，故捐献 __PRI_DEFAULT + 3__ 优先级给测试线程，然后被休眠。  
接下来创建优先级为 __PRI_DEFAULT + 1__ 的 c 线程，优先级低于测试线程，阻塞。
```c
thread_create ("c", PRI_DEFAULT + 1, c_thread_func, NULL);
```
接下来我们创建优先级为 __PRI_DEFAULT + 5__ 的线程 __B__， __B__ 抢占执行 `b_thread_func()`。
```c
thread_create ("b", PRI_DEFAULT + 5, b_thread_func, &b);
```
然而它和 __A__ 是一个结局，在`b_thread_func()` 中，__B__ 请求锁 b，锁被测试线程持有，故捐献 __PRI_DEFAULT + 5__ 优先级给测试线程，然后被休眠。  
接下来 __测试线程__ 通过 `lock_release (&a)` 释放锁 a，线程 __A__ 拿到锁，但是优先级不如 __测试线程__ 高，所以继续执行测试线程。   
然后 __测试线程__ 通过 `lock_release (&b)` 释放锁 b，线程 __B__ 拿到锁，并抢占执行。而此时由于 __测试线程__ 不再锁 __B__，所以优先级变回 __PRI_DEFAUL__ 。  
__B__ 释放锁 b 并执行完毕，开始执行刚刚被阻塞的 __A__ ，__A__ 释放锁 a。   
然后由于刚刚被阻塞的 __C__ 优先级高于 __测试线程__ ，所以先执行 __C__，之后回到 __测试线程__ 。  
测试结束。   
![](/home/airy/圖片/project3-6.png)   
期望输出如下：
```shell
  (priority-donate-multiple2) begin
  (priority-donate-multiple2) Main thread should have priority 34.  Actual priority: 34.
  (priority-donate-multiple2) Main thread should have priority 36.  Actual priority: 36.
  (priority-donate-multiple2) Main thread should have priority 36.  Actual priority: 36.
  (priority-donate-multiple2) Thread b acquired lock b.
  (priority-donate-multiple2) Thread b finished.
  (priority-donate-multiple2) Thread a acquired lock a.
  (priority-donate-multiple2) Thread a finished.
  (priority-donate-multiple2) Thread c finished.
  (priority-donate-multiple2) Threads b, a, c should have just finished, in that order.
  (priority-donate-multiple2) Main thread should have priority 31.  Actual priority: 31.
  (priority-donate-multiple2) end
```
现在由于没有完成优先级捐献，所以没有被阻塞的 __C__ 先执行完了，释放了锁后 __A__ 和 __B__ 也并没有被调度，然后就结束了：  
```shell
  (priority-donate-multiple2) begin
  (priority-donate-multiple2) Main thread should have priority 34.  Actual priority: 31.
  (priority-donate-multiple2) Thread c finished.
  (priority-donate-multiple2) Main thread should have priority 36.  Actual priority: 31.
  (priority-donate-multiple2) Main thread should have priority 36.  Actual priority: 31.
  (priority-donate-multiple2) Threads b, a, c should have just finished, in that order.
  (priority-donate-multiple2) Main thread should have priority 31.  Actual priority: 31.
  (priority-donate-multiple2) end
```

* __priority-donate-nest：情况4。测试递归捐赠，以及解锁后优先级及时更新。__   
首先初始化了两个锁a/b，并且测试线程请求并拿到锁 a 。    
创建线程 __medium__ ，优先级为 __PRI_DEFAULT + 1__ ，抢占执行 `medium_thread_func()`。
```c
  thread_create ("medium", PRI_DEFAULT + 1, medium_thread_func, &locks);
```
在 `medium_thread_func()` 中，__medium__ `lock_acquire (locks->b)`请求并拿到锁 b，并 `lock_acquire (locks->a)` 请求锁 a，a 被测试线程持有，所以被阻塞。__medium__ 此时将优先级 __PRI_DEFAULT + 1__ 捐给 __测试线程__ 。
回到 __测试线程__，创建线程 __high__ ，优先级为 __PRI_DEFAULT + 2__，抢占执行 `high_thread_fun()` 。
```c
  thread_create ("high", PRI_DEFAULT + 2, high_thread_func, &b);
```
在 `high_thread_fun()` 中，__high__ 请求锁 b，此时被 __medium__ 持有，故被阻塞。优先级 __PRI_DEFAULT + 2__ 被捐赠给 __medium__ 和 __测试线程__ 。  
现在回到 __测试线程__，执行 `thread_yield ()`，但是由于ready队列为空，所以继续执行 __测试线程__ 。  
现在通过 `lock_release (&a)` 释放锁 a ， __medium__ 拿到锁 a 。__测试线程__ 优先级回到 __PRI_DEFAULT__。   
__medium__ 拿到 a 后，释放 a ，进行一次线程调度。这个时候， __medium__ 优先级更高，所以会继续执行 __medium__ 。  
通过 `lock_release (locks->b)` 释放锁 b ，__high__ 拿到锁， __medium__ 的优先级此时会回到 __PRI_DEFAULT + 1__ ，然后被 __high__ 抢占。
__high__ 拿到锁，再通过 `lock_release (lock)` 释放，退出。  
回到 __medium__ 线程，现在又进行一次 `thread_yield ()`，__medium__ 优先级更高，继续执行 __medium__ ，然后退出。  
回到 __测试线程__ ，再次 `thread_yield ()`，没有别的线程了所以继续。之后结束测试。  
![](/home/airy/圖片/project3-7.png)  
期望的输出：
```shell
  (priority-donate-nest) begin
  (priority-donate-nest) Low thread should have priority 32.  Actual priority: 32.
  (priority-donate-nest) Low thread should have priority 33.  Actual priority: 33.
  (priority-donate-nest) Medium thread should have priority 33.  Actual priority: 33.
  (priority-donate-nest) Medium thread got the lock.
  (priority-donate-nest) High thread got the lock.
  (priority-donate-nest) High thread finished.
  (priority-donate-nest) High thread should have just finished.
  (priority-donate-nest) Middle thread finished.
  (priority-donate-nest) Medium thread should just have finished.
  (priority-donate-nest) Low thread should have priority 31.  Actual priority: 31.
  (priority-donate-nest) end
```
现在的输出，虽然输出顺序没问题，但是没有完成优先级捐赠：
```shell
  (priority-donate-nest) begin
  (priority-donate-nest) Low thread should have priority 32.  Actual priority: 31.
  (priority-donate-nest) Low thread should have priority 33.  Actual priority: 31.
  (priority-donate-nest) Medium thread should have priority 33.  Actual priority: 32.
  (priority-donate-nest) Medium thread got the lock.
  (priority-donate-nest) High thread got the lock.
  (priority-donate-nest) High thread finished.
  (priority-donate-nest) High thread should have just finished.
  (priority-donate-nest) Middle thread finished.
  (priority-donate-nest) Medium thread should just have finished.
  (priority-donate-nest) Low thread should have priority 31.  Actual priority: 31.
  (priority-donate-nest) end
```

* __priority-donate-sema：情况2。测试在有信号量参与的情况下，优先级捐赠的情况。__   
首先创建一个包含一个信号量和一个锁的结构体 `lock_and_sema ls`并初始化。
```c
  struct lock_and_sema ls;
  lock_init (&ls.lock);
  sema_init (&ls.sema, 0);
```
先创建一个线程 __low__，优先级为 __PRI_DEFAULT + 1__ ，抢占执行 `l_thread_func()`。
```c
  thread_create ("low", PRI_DEFAULT + 1, l_thread_func, &ls);
```
在 `l_thread_func()` 中，先请求并拿到了锁，然后进行一次semadown（P操作），被阻塞。
```c
  lock_acquire (&ls->lock);
  sema_down (&ls->sema);
```
回到 __测试线程__，创建线程 __medium__，优先级为 __PRI_DEFAULT + 3__，抢占执行 `m_thread_func()`。
```c
  thread_create ("med", PRI_DEFAULT + 3, m_thread_func, &ls);
```
在 `m_thread_func()` 中，进行一次semadown（P操作），被阻塞。   
回到 __测试线程__，创建线程 __high__，优先级为 __PRI_DEFAULT + 5__，抢占执行 `h_thread_func()`。
```c
  thread_create ("high", PRI_DEFAULT + 5, h_thread_func, &ls);
```
在 `h_thread_func()` 中，请求锁，锁被 __low__ 持有，贡献优先级给 __low__，并阻塞。   
现在 __测试线程__ 通过 `sema_up (&ls.sema)` 释放信号量，由于此时请求信号量的 __low__ 优先级更高，所以它拿到信号量后释放锁，优先级回到初始。   
优先级最高的 __high__ 此时可以拿到锁，然后释放一个信号量，然后结束。   
__medium__ 此时可以拿到信号量继续执行到结束。__medium__ 结束后，线程 __low__ 也结束。  
测试结束。
![](/home/airy/圖片/project3-8.png)
理论输出：
```shell
  (priority-donate-sema) begin
  (priority-donate-sema) Thread L acquired lock.
  (priority-donate-sema) Thread L downed semaphore.
  (priority-donate-sema) Thread H acquired lock.
  (priority-donate-sema) Thread H finished.
  (priority-donate-sema) Thread M finished.
  (priority-donate-sema) Thread L finished.
  (priority-donate-sema) Main thread finished.
  (priority-donate-sema) end
```
实际输出：
```shell
  (priority-donate-sema) begin
  (priority-donate-sema) Thread L acquired lock.
  (priority-donate-sema) Main thread finished.
  (priority-donate-sema) end
```
三个线程被阻塞后都没有继续再执行。

* __priority-donate-lower：测试置低优先级时，是否不会影响捐献的优先级。__  
首先 __测试线程__ 请求并拿到锁。然后创建了优先级为 __PRI_DEFAULT + 10__ 的 __acquire__ 线程，抢占执行。
```c
lock_acquire (&lock);
thread_create ("acquire", PRI_DEFAULT + 10, acquire_thread_func, &lock);
```
在 `acquire_thread_func()` 中，acquire 线程请求锁，被阻塞。  
__测试线程__ 通过 `thread_set_priority (PRI_DEFAULT - 10)` 降低优先级，但由于仍然被 __acquire__ 捐赠优先级，所以继续运行。  
而后 __测试线程__ 通过 `lock_release (&lock)` 释放锁，不再被捐赠，优先级回到 __PRI_DEFAULT - 10__ 。获得锁的 __acquire__ 抢占执行。  
线程 __acquire__ 结束后，回到 __测试线程__ ，测试结束。
![](/home/airy/圖片/project3-9.png)
理论输出：
```shell
  (priority-donate-lower) begin
  (priority-donate-lower) Main thread should have priority 41.  Actual priority: 41.
  (priority-donate-lower) Lowering base priority...
  (priority-donate-lower) Main thread should have priority 41.  Actual priority: 41.
  (priority-donate-lower) acquire: got the lock
  (priority-donate-lower) acquire: done
  (priority-donate-lower) acquire must already have finished.
  (priority-donate-lower) Main thread should have priority 21.  Actual priority: 21.
  (priority-donate-lower) end
```
实际输出：
```shell
  (priority-donate-lower) begin
  (priority-donate-lower) Main thread should have priority 41.  Actual priority: 31.
  (priority-donate-lower) Lowering base priority...
  (priority-donate-lower) Main thread should have priority 41.  Actual priority: 21.
  (priority-donate-lower) acquire must already have finished.
  (priority-donate-lower) Main thread should have priority 21.  Actual priority: 21.
  (priority-donate-lower) end
```
没有实现优先级捐赠，且acquire被阻塞后没有再唤醒继续执行。

* __priority-donate-chain：测试是否能正确完成优先级递归捐赠、优先级还原以及优先级抢占。__  
首先通过 `thread_set_priority (PRI_MIN)` 将 __测试线程__ 的优先级置为最低优先级。而后创建 __NESTING_DEPTH - 1__ （7）个锁。
```c
for (i = 0; i < NESTING_DEPTH - 1; i++)
lock_init (&locks[i]);
```
然后主线程通过 `lock_acquire (&locks[0])` 拿到 0 号锁。  
接下来会创建 7 组线程，每组有两个线程。  
<ul>
    <li>对于第 i 组的第一个线程： 
    <ul>
    <li>置优先级：PRI_MIN + i \* 3</li>
    <li>请求并拿到锁：i（第 7 组的第一个不拿锁）</li>
    <li>请求锁（被阻塞）：i - 1</li>
    <li>命名为：thread i</li>
    </ul>
    </li>
    <li>对于第 i 组的第二个线程：
    <ul>
    <li>置优先级：PRI_MIN + i * 3 - 1</li>
    <li>命名为：interloper i</li>
    </ul>
    </li>
</ul>
```c
for (i = 1; i < NESTING_DEPTH; i++)
{
  char name[16];
  int thread_priority;

  snprintf (name, sizeof name, "thread %d", i);
  thread_priority = PRI_MIN + i * 3;
  lock_pairs[i].first = i < NESTING_DEPTH - 1 ? locks + i: NULL;
  lock_pairs[i].second = locks + i - 1;

  thread_create (name, thread_priority, donor_thread_func, lock_pairs + i);

  snprintf (name, sizeof name, "interloper %d", i);
  thread_create (name, thread_priority - 1, interloper_thread_func, NULL);
}
```
显然，这里的前一组会阻塞后一组，而第 0 组会被 __测试线程__ 阻塞。那么现在 __测试线程__ 以及每组的第一个线程的优先级均会被捐赠为 __PRI_MIN + 21__，而第二个线程也会因为优先级较低不会立刻执行。  
此时，__测试线程__ 通过 `lock_release (&locks[0])` 释放 0 锁，优先级回到 __PRI_MIN__ 。  
现在thread 1获得锁0并执行，然后释放锁0和锁1，优先级回到 __PRI_MIN + 3__。   
——thread 2获得锁1并执行，然后释放锁1和锁2，优先级回到 __PRI_MIN + 6__。  
——……   
——thread 6获得锁5并执行，然后释放锁5和锁6，优先级回到 __PRI_MIN + 18__。  
——thread 7获得锁6并执行，然后释放锁6，运行完毕。   
至此，不再存在优先级为 __PRI_MIN + 21__ 的线程，开始依次执行优先级为 __PRI_MIN + 20__ 的interloper 7、优先级为 __PRI_MIN + 18__ 的thread 6、优先级为 __PRI_MIN + 17__ 的interloper 6、优先级为 __PRI_MIN + 15__ 的 thread5……最后回到 __测试线程__ 并结束测试。
![](/home/airy/圖片/project3-10.png)
理论输出：
```shell
  (priority-donate-chain) begin
  (priority-donate-chain) main got lock.
  (priority-donate-chain) main should have priority 3.  Actual priority: 3.
  (priority-donate-chain) main should have priority 6.  Actual priority: 6.
  ...
  (priority-donate-chain) main should have priority 21.  Actual priority: 21.
  (priority-donate-chain) thread 1 got lock
  (priority-donate-chain) thread 1 should have priority 21. Actual priority: 21
  (priority-donate-chain) thread 2 got lock
  (priority-donate-chain) thread 2 should have priority 21. Actual priority: 21
  ...
  (priority-donate-chain) thread 6 should have priority 21. Actual priority: 21
  (priority-donate-chain) thread 7 got lock
  (priority-donate-chain) thread 7 should have priority 21. Actual priority: 21
  (priority-donate-chain) thread 7 finishing with priority 21.
  (priority-donate-chain) interloper 7 finished.
  (priority-donate-chain) thread 6 finishing with priority 18.
  (priority-donate-chain) interloper 6 finished.
  ...
  (priority-donate-chain) thread 2 finishing with priority 6.
  (priority-donate-chain) interloper 2 finished.
  (priority-donate-chain) thread 1 finishing with priority 3.
  (priority-donate-chain) interloper 1 finished.
  (priority-donate-chain) main finishing with priority 0.
  (priority-donate-chain) end
```
实际输出：
```shell
  (priority-donate-chain) begin
  (priority-donate-chain) main got lock.
  (priority-donate-chain) main should have priority 3.  Actual priority: 0.
  (priority-donate-chain) interloper 1 finished.
  (priority-donate-chain) main should have priority 6.  Actual priority: 0.
  (priority-donate-chain) interloper 2 finished.
  ...
  (priority-donate-chain) main should have priority 21.  Actual priority: 0.
  (priority-donate-chain) interloper 7 finished.
  (priority-donate-chain) main finishing with priority 0.
  (priority-donate-chain) end
```
每组第一个线程被阻塞后没有再被调度，而第二个线程由于没有被测试线程阻塞，所以在创建的时候就执行了。   

* __priority-sema：测试信号量改变时的抢占。__   
首先初始化一个信号量，然后将 __测试线程__ 优先级置最低。   
现在创建十个线程，优先级排列不有序。并且，这十个线程都会通过 `sema_down (&sema)` 请求信号量，被阻塞。
```c
  for (i = 0; i < 10; i++) 
    {
      int priority = PRI_DEFAULT - (i + 3) % 10 - 1;
      char name[16];
      snprintf (name, sizeof name, "priority %d", priority);
      thread_create (name, priority, priority_sema_thread, NULL);
    }
```
然后 __测试线程__ 依次释放十个信号量。   
现在，十个线程应当按照优先级被依次唤醒。
```shell
(priority-sema) begin
(priority-sema) Thread priority 30 woke up.
(priority-sema) Back in main thread.
(priority-sema) Thread priority 29 woke up.
(priority-sema) Back in main thread.
...
(priority-sema) Thread priority 22 woke up.
(priority-sema) Back in main thread.
(priority-sema) Thread priority 21 woke up.
(priority-sema) Back in main thread.
(priority-sema) end
```
在我们没有设置信号量改变时的抢占时，这十个线程是都不会被唤醒的。

#### （二）实验过程
* __需要解决的问题：__
    1. 将信号量队列变为优先队列。
    2. 高优先级被低优先级线程锁住时进行优先级捐赠。
    3. 递归捐赠。
    4. 考虑某一线程的一把锁锁住了多个高优先级线程。
    	* 为锁添加优先级。
    5. 考虑一个线程持有多把锁。
		1. 优先级为所有锁优先级中最大的。
		2. 释放某把锁的时候取剩余的最大优先级。
    6. 释放锁（信号量）考虑优先级抢占。
    7. 考虑捐赠情况下的置优先级修改。
    	1. 判断是否被捐赠。
    	2. 比较新优先级、原始优先级、被捐赠优先级的大小。
* __需要添加的变量（注意要初始化）：__
	1. thread
		1. `int ori_priority` 原始优先级
		2. `struct list hold` 现在持有的锁
		3. `struct lock *wait` 现在需要的锁 
	2. lock
		1. `int priority` 锁的优先级（被锁住的线程中最大的优先级）
		2. `struct list_elem elem` 拿去排队的苦力
* __修改sema_up()和sema_down()：__   
由于这玩意儿看得我太难受了所以我们先解决这个问题。首先它现在是这样的：
```c
void sema_up (struct semaphore *sema)  {
    ...
	if (!list_empty (&sema->waiters)) 
		thread_unblock (list_entry (list_pop_front (&sema->waiters),
                                struct thread, elem));
	sema->value++;
    ...
}
void sema_down (struct semaphore *sema) {
	...
	while (sema->value == 0) {
      list_push_back (&sema->waiters, &thread_current ()->elem);
      thread_block ();
    }
	sema->value--;
	...
}
```
我们看到，每个 sema 专门有个成员 list 叫 __waiters__。我们每次 sema_down 的时候，如果没能成功请求到信号量，那么就把它放进这个 __waiters__ 里面，然后每次 sema_up 的时候，就取出 __waiters__ 里面的第一个。   
搜索一下，发现这个 __waiters__ 其实只会在这两个函数里使用到。也就是说，其实我们并不需要保证这个队列随时有序，只需要考虑取出时，取出的是优先级最大的线程即可。所以，取出前排一次序。  
另外，这里要处理一下抢占。
```c
void sema_up(struct semaphore *sema) {
	...
    if (!list_empty (&sema->waiters)) {
        list_sort(&sema->waiters, cmp, NULL);
        thread_unblock (list_entry (list_pop_front (&sema->waiters),
                                    struct thread, elem));
    }
	sema->value++;
	thread_yield ();
	...
}
```
* __修改lock_acquire()：__   
现在我们来捐赠了。顺便处理一下递归捐赠。   
首先，为了对得起“递归”二字，我们来写一个递归函数 `donate()` 。
```swift
procedure DONATE(lock)
	lock.priority ← current_priority
    lock.holder.priority ← current_priority
    recursively call DONATE(lock.holder.wait)
```
实现如下：
```c
void donate(struct lock* lock, int current_priority) {
    if (lock == NULL) return;
    lock->priority = current_priority;
    lock->holder->priority = current_priority;
    donate(lock->holder->wait, current_priority);
}
```
然后用 `donate(&lock->holder->wait, current_priority);` 调用这个函数就好啦！   
接下来处理拿到锁以后的事。锁：修改优先级和持有者。线程：修改等待的锁和持有锁队列。   
现在这个东西变成了这样（为了减少BUG的产生，我们做尽量少的工作）：  
```c
void lock_acquire (struct lock *lock) {
	...
    if (lock->holder) {
    	thread_current()->wait = lock;
    	donate(&lock, thread_current()->priority);
	}
	sema_down (&lock->semaphore);
    lock->holder = thread_current();
    lock->priority = thread_current()->priority;
    thread_current()->wait = NULL;
    list_push_back(&thread_current()->hold, &lock->elem);
}
```
* __修改lock_release()：__   
```swift
procedure LOCK_RELEASE(lock)
	lock.holder ← NULL
    lock.priority ← PRI_MIN
    remove lock from thread_current.hold
    if thread_current.hold is empty
    	priority ← ori_priority 
    else do
    	sort thread_current.hold
        priority ← max ( max_priority, ori_priority )
    Operation V
```
先设置一下锁：1. 持有者为空。2. 优先级回到最低。然后从 thread_current 的 hold 列表中把这个锁拿出来。   
然后现在来看看 thread_current 手上还有没有锁。1. 如果没有锁了，就直接恢复到原始优先级。2.手上有锁，取出锁队列里优先级最大的锁。如果这个锁优先级比原始优先级更大，那么继续 thread_current 继续被捐赠，否则回到原始优先级。  
最后还是要执行一下V操作。
```c
void lock_release (struct lock *lock) {
	...
	lock->holder = NULL;
    lock->priority = PRI_MIN;
    list_remove(&lock->elem);
    if (list_empty(&thread_current()->hold)) {
    	thread_current()->priority = thread_current()->ori_priority;
    } else {
    	list_sort(&thread_current()->hold, lock_cmp, NULL);
    	int max_priority = list_entry(list_front(&thread_current()->hold), struct lock, elem)->priority;
    	if (max_priority <= thread_current()->ori_priority) {
        	thread_current()->priority = thread_current()->ori_priority;
    	} else {
    		thread_current()->priority = max_priority;
    	}
    }
    sema_up (&lock->semaphore);
}
```
* __修改 thread_set_priority() __  
```swift
procedure THREAD_SET_PRIORITY(new_priority)
	if [isn't donated] or [new_priority > priority]
    	priority = ori_priority = new_priority
    else
    	ori_priority = new_priority
```
首先 ori_priority = new_priority 是一定没问题的。那么现在的问题在于需不需要更改priority。  
考虑没有被捐赠的情况，那么需要保证 priority == ori_priority，这个时候priority是不需要修改的。   
然后考虑被捐赠时。这个时候我们的 priority 应当取 max(priority, new_priority)，这样保证了优先级的捐赠不被影响，也保证了自己的priority的准确。   
实现如下：
```c
void thread_set_priority (int new_priority) {
	if (thread_current()->priority == thread_current()->ori_priority) {
		thread_current()->priority = thread_current()->ori_priority = new_priority;
	} else if (new_priority <= thread_current()->priority) {
		thread_current()->ori_priority = new_priority;
	} else {
		thread_current()->priority = thread_current()->ori_priority = new_priority;
	}

	if (!list_empty(&ready_list) && 
    	new_priority < list_entry(list_begin (&ready_list), struct thread, elem)->priority)
		thread_yield ();
}
```

### 3. 实验结果
![](/home/airy/圖片/project3-11.png)

### 4. 回答问题
__1. 就绪队列中的线程优先级是否可能发生改变？如果能，请描述在哪种情况下可能发生。__  
可能发生改变。当前线程如果被就绪队列中的某一（某些）线程锁住，那么就会把优先级捐献给它（们）。

__2. 如何解决一个线程占有多个锁的优先级捐赠问题？__  
在线程结构体中添加一个 list 记录现在持有的锁。   
请求锁的时候，将锁放入被锁线程的 list 中，并且直接将当前线程的优先级向前捐赠（当前线程，一定是，优先级最高的）。  
释放锁的时候判断一下锁队列里是否有剩余的可以捐赠优先级的锁，如果有，就继续被捐赠，没有就回到初始优先级。

__3. 在实现优先级捐赠后， thread_set_priority 要考虑多少种情况？分别怎么处理？__   
1. 未被捐赠时正常处理：priority = ori_priority = new_priority
2. 捐赠时保证优先级为最大值：priority = max(priority, new_priority); ori_priority = new_priority

### 5.实验感想
首先我最大的感想是：网上那些人写的博客真的不靠谱。

我可以暂且认为他们写的东西是没问题的，但是大部分东西都是冗余的。我觉得可能还是因为学习这门课的人，一般代码量都比较不足吧。说实话，虽然我们也算半个“科班出生”，但是学校能提供给我们的代码练习资源还是太少了。这里就不展开说了，毕竟我自己的代码能力也在肉眼可见地退步。  

来说说这次学习的内容。我觉得了解这些聪明的科学家折腾出来的调度算法，还是很有趣的。利用捐献来解决反转问题，实在是很精彩的做法。<del>但是我一直觉得“捐献”这两个字翻译得不太好</del>。很多 test 也是写的很优雅的，比如说 priority-donate-chain，虽然它分析起来比较复杂，但是确实是一个很姿势好的测试代码，看得出智慧所在。

其实写的中途还是遇到了很多问题，比如 markdown 渲染器不兼容什么的，还有遇到莫名其妙需要重新 make 的，以及画图画到手疼（还是伪代码好啊），以及看网上那些神奇的代码一脸懵逼。为了让报告尽量简洁，也改了很多次，去掉了很多辛辛苦苦打出来的废话。 

其实真的用来写代码的时间也就一两个小时，本身也没太多东西需要添加。但是为了写这几十行代码还是耗了不少时间。总之，写完了。该好好休息一下了。活着太累了。