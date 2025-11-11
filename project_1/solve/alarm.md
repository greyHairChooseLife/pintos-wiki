


## alarm clock 문제


### 문제 이해

이 문제는 busy-waiting 방식을 개선하는 것이다.
(CPU 안쓰니까)정해진 시간만큼 자러 가겠다는데, 아직 약속시간도 안된 쓰레드를 자꾸만 불러들여서 정해진 시간만큼 지났나 안지났나 확인을 한다.

그래서 waiting인데도 아주 바쁜 것이다.



### 어려운 점

이 문제를 증명하는 별도의 테스트케이스는 없는것 같다. 그니까, busy-waiting인지 그부분이 개선된건지 알 방법이 제공되어있지 않은것같다.

그래서 아래처럼 출력했따.

```gitcommit
28529fc etc: revealing busy waiting <sangyeon kim>
```

- File Path: ntos/7team/.git/28529fc9fba/devices/timer.c, 85:97
  ```c
  void timer_sleep(int64_t ticks) {
      int64_t start = timer_ticks();
  
      ASSERT(intr_get_level() == INTR_ON);
      int count = 0;
  
      while (timer_elapsed(start) < ticks)
      {
          printf("%s: %dth call: I'm not ready!\n", thread_current()->name,
                 ++count);
          thread_yield();
      }
  }
  ```

- 결과
  ```log
  (alarm-single) --------------------------------------- START
  (alarm-single) Creating 5 threads to sleep 1 times each.
  (alarm-single) Thread 0 sleeps 10 ticks each time,
  (alarm-single) thread 1 sleeps 20 ticks each time, and so on.
  (alarm-single) If successful, product of iteration count and
  (alarm-single) sleep duration will appear in nondescending order.
  main: 1th call: I'm not ready!
  thread 0: 1th call: I'm not ready!
  thread 1: 1th call: I'm not ready!
  thread 2: 1th call: I'm not ready!
  thread 3: 1th call: I'm not ready!
  thread 4: 1th call: I'm not ready!
  main: 2th call: I'm not ready!
  thread 0: 2th call: I'm not ready!
  thread 1: 2th call: I'm not ready!
  thread 2: 2th call: I'm not ready!
  thread 3: 2th call: I'm not ready!
  thread 4: 2th call: I'm not ready!
  main: 3th call: I'm not ready!
  thread 0: 3th call: I'm not ready!
  thread 1: 3th call: I'm not ready!
  thread 2: 3th call: I'm not ready!
  thread 3: 3th call: I'm not ready!
  thread 4: 3th call: I'm not ready!
  main: 4th call: I'm not ready!
  thread 0: 4th call: I'm not ready!
  ```


### 해결

  - as is: 자러 간다고 하면 곧바로 또 줄세우기
  - to be: 자러 간다고 하면 잠자는 리스트에 넣기
    - timer handler가 돌 때마다 잠자는 리스트에서 깨울애들 확인하고 깨우기

