

**REFERENCE**

```c
/* States in a thread's life cycle. */
enum thread_status {
    THREAD_RUNNING, /* Running thread. */
    THREAD_READY,   /* Not running but ready to run. */
    THREAD_BLOCKED, /* Waiting for an event to trigger. */
    THREAD_DYING    /* About to be destroyed. */
};
```



---


**FLOW**



1. `void thread_init(void)`

    - Initializes the threading system
    - Also initializes the run queue and the tid lock.



2. `void thread_start(void)`

    - Starts preemptive thread scheduling by enabling interrupts.
    - Also _creates the idle thread_ using,

        - `thread_create("idle", PRI_MIN, idle, &idle_started)`

            - `palloc_get_page(PAL_ZERO)`: get page address with zero setup. This is used as thread pointer.
            - `init_thread(t, name, priority)`
                - t 구조체 멤버에 thread status, rsp, name, priority 값 세팅
                - 특히, `t->status = THREAD_BLOCKED;`
            - t.tid = `allocate_tid()`; 이놈은 고유한 tid를 생성해서 할당해준다.(next_tid static변수 선언하고 1씩 증가)
            - `thread_unblock(t)`;
                - `list_push_back(&ready_list, &t->elem);`
                - `t->status = THREAD_READY;`


            - what is `idle()`?

                - It prevents the ready_list not to be empty so scheduler is safe.
                - `thread_block()`

                    - assign `thread_current()->status = THREAD_BLOCKED`
                    - run `schedule()`

                        - `thread_launch(next)`

                            * We first restore the whole execution context into the _intr_frame_
                            * And then _switching to the next thread_ by calling do_iret.
     


---


- `schedule()` sets `thread_ticks = 0` and `void thread_tick(void)` checks like;

    - Destroy its struct thread, if the thread we switched from is dying, 

    - `intr_yield_on_return()`  which is;

      ```c
          if (++thread_ticks >= TIME_SLICE) intr_yield_on_return();
      ```

      During processing of an external interrupt, directs the
      interrupt handler to _yield to a new process_ just before
      returning from the interrupt.  May not be called at any other time.




- `thread_create(const char* name)`

  ```c
  tid_t thread_create(const char* name,
                      int priority,
                      thread_func* function,
                      void* aux) {
  ```

    - `palloc_get_page(PAL_ZERO)`: get page address with zero setup. This is used as thread pointer.

    - `init_thread(t, name, priority)`

        - t 구조체 멤버에 thread status, rsp, name, priority 값 세팅
        - 특히, `t->status = THREAD_BLOCKED;`

    - t.tid = `allocate_tid()`; 이놈은 고유한 tid를 생성해서 할당해준다.(next_tid static변수 선언하고 1씩 증가)

    - `thread_unblock(t)`;

        - `list_push_back(&ready_list, &t->elem);`
        - `t->status = THREAD_READY;`




- `void thread_yield(void)`

    - 현재 쓰레드를 ready_list 맨 뒤에 넣어버리기
    - `schedule()`



