
## Alarm Clock

### 목표 확인

- **objective:**

  Reimplement timer_sleep(), defined in devices/timer.c.

---


- Although a working implementation is provided,
- it busy waits,

    it spins in a loop checking the current time and calling `thread_yield()` until enough time has gone by.

- **Reimplement it to avoid busy waiting.**



`void timer_sleep (int64_t ticks);

Suspends execution of the calling thread until time has advanced by at least x
timer ticks. Unless the system is otherwise idle, the thread need not wake up
after exactly x ticks. Just put it on the ready queue after they have waited
for the right amount of time.


timer_sleep() is useful for threads that operate in real-time, e.g. for
blinking the cursor once per second. The argument to timer_sleep() is expressed
in timer ticks, not in milliseconds or any another unit. There are TIMER_FREQ
timer ticks per second, where TIMER_FREQ is a macro defined in devices/timer.h.
The default value is 100. We don't recommend changing this value, because any
change is likely to cause many of the tests to fail.


Separate functions timer_msleep(), timer_usleep(), and timer_nsleep() do exist
for sleeping a specific number of milliseconds, microseconds, or nanoseconds,
respectively, but these will call timer_sleep() automatically when necessary.
You do not need to modify them. The alarm clock implementation is not needed
for later projects, although it could be useful for project 4.


### 결과물 Log

리팩토링 해도 딱히 변화는 없다. 테스트 케이스가 너무 짧아서 그런가..?

#### 단순히 sleep list에 삽입(정렬 x)


- File Path: my_test/output, 29:31
  ```
  Execution of 'alarm-single' complete.
  Timer: 272 ticks
  Thread: 250 idle ticks, 22 kernel ticks, 0 user ticks
  ```

- File Path: my_test/output, 59:61
  ```
  Execution of 'alarm-multiple' complete.
  Timer: 578 ticks
  Thread: 550 idle ticks, 28 kernel ticks, 0 user ticks
  ```

- File Path: my_test/output, 37:39
  ```
  Execution of 'alarm-simultaneous' complete.
  Timer: 274 ticks
  Thread: 250 idle ticks, 24 kernel ticks, 0 user ticks
  ```


##### code

- File Path: threads/thread.c, 232:242
  ```c
  void thread_block(void) {
      struct thread* curr = thread_current();
  
      ASSERT(!intr_context());
      ASSERT(intr_get_level() == INTR_OFF);
  
      curr->status = THREAD_BLOCKED;
      if (curr != idle_thread) list_push_front(&sleep_list, &curr->elem);
  
      schedule();
  }
  ```


#### 넣을 때 정렬해서 넣기(빨리 깨야하는 애들부터)



- File Path: my_test/output, 29:31
  ```
  Execution of 'alarm-single' complete.
  Timer: 273 ticks
  Thread: 250 idle ticks, 23 kernel ticks, 0 user ticks
  ```

- File Path: my_test/output, 59:61
  ```
  Execution of 'alarm-multiple' complete.
  Timer: 578 ticks
  Thread: 550 idle ticks, 29 kernel ticks, 0 user ticks
  ```

- File Path: my_test/output, 37:39
  ```
  Execution of 'alarm-simultaneous' complete.
  Timer: 274 ticks
  Thread: 250 idle ticks, 24 kernel ticks, 0 user ticks
  ```

##### code

- File Path: threads/thread.c, 235:258
  ```c
  void thread_block(void) {
      struct thread* curr = thread_current();
  
      ASSERT(!intr_context());
      ASSERT(intr_get_level() == INTR_OFF);
  
      curr->status = THREAD_BLOCKED;
      if (curr != idle_thread)
          list_insert_ordered(&sleep_list, &curr->elem, soonner_first, NULL);
  
      schedule();
  }
  
  // 빨랑 깨워야하는 애들부터 정렬
  bool soonner_first(const struct list_elem* a,
                     const struct list_elem* b,
                     void* _) {
      int64_t wakeupA = list_entry(a, struct thread, elem)->wakeup_time;
      int64_t wakeupB = list_entry(b, struct thread, elem)->wakeup_time;
  
      if (wakeupA - wakeupB <= 0) return true;
  
      return false;
  }
  ```



#### timer interrupt handler에서 sleep list 순회시 early return


- File Path: my_test/output, 29:31
  ```
  Execution of 'alarm-single' complete.
  Timer: 273 ticks
  Thread: 250 idle ticks, 23 kernel ticks, 0 user ticks
  ```

- File Path: my_test/output, 59:61
  ```
  Execution of 'alarm-multiple' complete.
  Timer: 578 ticks
  Thread: 550 idle ticks, 28 kernel ticks, 0 user ticks
  ```

- File Path: my_test/output, 37:39
  ```
  Execution of 'alarm-simultaneous' complete.
  Timer: 274 ticks
  Thread: 250 idle ticks, 25 kernel ticks, 0 user ticks
  ```


  ##### code

- File Path: threads/thread.c, 155:172
  ```c
  void awake(int64_t total_elapsed) {
      if (list_empty(&sleep_list)) return;  // early return
  
      struct thread* front =
          list_entry(list_front(&sleep_list), struct thread, elem);
  
      size_t len = list_size(&sleep_list);
  
      for (size_t i = 0; i < len; i++)
      {
          if (front->wakeup_time - total_elapsed > 0) return;
  
          struct list_elem* next = list_remove(&front->elem);
          thread_unblock(front);
  
          front = list_entry(next, struct thread, elem);
      }
  }
  ```


## Priority Scheduling



- [x] 1
- 상황: 새로운 쓰레드가 ready_list에 추가 된 때, _그것이 만약 현재 running 쓰레드보다 우선순위가 높다면_
    - ~create_thread~  ==> 결국은 unblock 실행하는 방식
    - `unblock()` 직후
    - ~`thread_yield()`~  즉시 CPU 양보하는 작업을 이놈이 하기때문에 예외
- 조치: 새로운 쓰레드에게 즉시 CPU 양보
    - thread_yield()

> [!qt] awake함수도 sleep_list ==> ready_list로 추가하는 작업을 하는데?
>   󱞪 
>
> 그런데 awake는 timer_interrupt시에 실행된다. 그럼 `ASSERT(!intr_context());` 조건 때문에 thread_yield가
> 에러를 이르킨다. **결국엔 ready_list에 추가되는 모든 경우는 안된다는 말인가?**
>
> 새로운 쓰레드가 새롭게 생성된 쓰레드를 뜻하는 것인가?



- [x] 2
- 조치:
    - 언제든 자신의 우선순위를 조정(Up/Down) 할 수 있어야하는데,
        - _다른 쓰레드의 우선순위는 건드리면 안된다._
    - 낮출 경우, 최고우선순위가 아니면 (사용하던) CPU를 즉시 양보해야한다.
    - 여기에 구현: `void thread_set_priority (int new_priority);`


> - ~낮추고 그냥 yield~
>     - 그냥 넣으면 알아서 우선순위 비교해준다. 더 높으면 다시 되찾을 것
>     - (넣을까 말까) 비교를 한번 덜한다는 장점
>     - 근데 굳이 넣었다 빼야해?
>
> - 낮추고 비교하고 yield
>   - front랑 비교해서 넣어야 하면 넣어
>   - 안넣어도 되면 그냥 하던거 해(이게 장점)



- [x] 3
- 상황: 쓰레드가 lock, semaphore, cond_var를 기다릴 때,
- 조치: Higher_P가 먼저 자원을 얻게된다.
    - ~lock의 경우 ready_list에서 알아서 관리된다.~
      - lock도 binary semaphore를 사용하여 같은 struct를 쓴다.
    - waiter 멤버(list)에 넣을때부터 정렬한다.




- [ ] 4
- 상황: Low_P 쓰레드가 lock한 자원을 High_P 쓰레드가 필요로 할 때,
- 조치: 
    - High_P 쓰레드가 Low_P 쓰레드에게 우선순위를 빌려준다.
    - Low_P 쓰레드가 해당 자원을 unlock하면 즉시 High_P에게 우선순위를 돌려준다.
    - 만약 여러개의 Higer_P 쓰레드가 존재하면 Highest로 받는다.
    - 만약 계단식으로 자원을 lock하고 있으면, Highest를 받는다.
        - H는, M을 기다리고, M은 L을 기다린다면, (M, L)모두 H의 우선순위를 받는다.
        - 최대 중첩에는 어느정도 제한이 있어야 한다.
            - 최대 제한이 걸리면 어떻게 대응할지...?

    - _locks에 대해서만 구현(sema, cond 제외)_


    - `int thread_get_priority (void);`
      - Returns the current thread's priority. 
      - _In the presence of priority donation, returns the higher (donated) priority._
      - priority donated인지 상태를 저장할 필요..?
      - 누구한테 줬는지 기억할 필요..?






> [!rf]
>
> - The initial thread priority is passed as an argument to `thread_create()`
>     - default to `PRI_DEFAULT`
>
> 
> - ```c
>   struct lock {
>       struct thread* holder;      /* Thread holding lock (for debugging). */
>       struct semaphore semaphore; /* Binary semaphore controlling access. */
>   };
>   
>   struct semaphore {
>       unsigned value;      /* Current value. */
>       struct list waiters; /* List of waiting threads. */
>   };
>   
>   struct condition {
>       struct list waiters; /* List of waiting threads. */
>   };
>   ```
>   
> - ```c
>   struct list {
>       struct list_elem head; /* List head. */
>       struct list_elem tail; /* List tail. */
>   };
>   
>   struct list_elem {
>       struct list_elem* prev; /* Previous list element. */
>       struct list_elem* next; /* Next list element. */
>   };
>   ```



## Advanced Scheduler


hm...
