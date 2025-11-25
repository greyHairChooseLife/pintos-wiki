## 기본적인 자료구조/알고리즘은 이미 구현되어있다. 

- `./lip/`


## 테스트 방식은 user_processes가 stdout에 출력하는 내용을 로깅하고 그것을 모범답안(`*.ck`)과 비교하는 것이다.


- Project.1은 그러했다...
- 다른것도 마찬가지인듯?



## 테스트를 실행한다는 것은 아래의 과정을 시뮬레이션 하는 방식이다.

1. 컴퓨터 켜기

     - bios가 OS sourcecode 찾기
     - 메모리에 load
     - main함수 실행

2. (테스트...)
3. 컴퓨터 종료


- 컴퓨터가 켜지는 과정 자체가 구현되어있으니, 한번 살펴보는것도 좋겠다.

    - assembly로 된 코드들도 있다.


## 쓰레드 구조체 생성 과정과 초기 모습


쓰레드는 당연히 커널이 생성해준다. 생성시의 순서는 대강 아래와 같다.

  1. palloc으로 페이지 하나 얻기
  2. 해당 페이지를 TCB로 삼고, 하단에는 thread 구조체로, 상단에는 stack으로 사용
  3. thread 구조체의 일부 멤버를 초기화
      - entry function, auxiliary
      - segments
      - name, tid, status, ...

  4. 프로세스 활성화

  5. scheduling (thread launch)
     - current thread(즉, 새로운 쓰레드를 생성해주는 커널 쓰레드)의 레지스터 정보를 자기 TCB에 백업해두고,
     - 새로운 쓰레드의 TCB로부터 CPU 레지스터를 꽉 채운다.
     - 그러고선 do_iret 함수 실행
       **do_iret이 뭔데?**





## 초기화된 thread의 TCB 모습

| Address         | Name       | Size (Byte) | Copied Value                               |
|-----------------|------------|-------------|--------------------------------------------|
| &thread+PSIZE   | null       | 8           | intended emptiness                         |
| &thread+PSIZE-8 | sp         |             |                                            |
| ...             | ...        | ...         | ...                                        |
| -               | magic      | 4           | MAGIC                                      |
| -               | ss         | 8           | kernel stack_segment                       |
| -               | **rsp**    | 8           | `&thread + PGSIZE(4KB) - (void *)`         |
| -               | eflags     | 8           | FLAG_IF                                    |
| -               | cs         | 8           | kernel code_segment                        |
| -               | **rip**    | 8           | `f(kernel_thread)`                         |
| -               | error_code | 8           |                                            |
| -               | vec-no     | 8           |                                            |
| -               | ds         | 8           | kernel stack_segment                       |
| -               | es         | 8           | kernel stack_segment                       |
| -               | **R**      | **120**     | registers(**rdi**=`f(entry)`, **rsi**=`auxiliary`) |
| -               | plm4       | 8           |                                            |
| ...             | ...        | ...         | ...                                        |
| &thread         | thread     |             | `struct thread` (tid, status, name, ...)   |




