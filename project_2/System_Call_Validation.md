
## System Call Validation

system call을 한다고 해서 user thread가 kernel의 메모리에 접근할 수 있는건 아니다.
예를들어 write syscall을 호출할 때에도 , arguments로 전달하는 것들의 주소는 모두 유저 영역에 있어야 한다.

커널 입장에서는 이것을 믿고 기다리는게 아니라 잘못된 주소로 접근하는지 확인하고 대응해줘야한다.


- validation cases

1. 접근 주소가 NULL
2. 접근 주소가 USER_STACK 이상
3. 접근 주소가 unmapped
    - Not loaded segments
    - Under the stack pointer





## 어떻게 mapped/unmapped area를 구분할 수 있을까?

### 첫 시도

syscall_handler가 전달받는 tf argument는 TCB를 그대로 얻고있는가?

만약 그렇다면 주소 연산을 통해 TCB의 다른 위치에도 접근할 수 있고,
그렇다면 load_segments 할 때 loaded 영역을 TCB에 담아주면 될 것이다.


**근데 아니다.**

  - debugger에서 thread 구조체의 크기(352 bytes)만큼을 tf 구조체 주소에서 빼봐도 여전히 접근 가능하다.
    TCB였다면 접근 불가능한 주소라고 나왔을 것.

  - 이때 tf구조체는 syscall_entry가 호출되어 실행되면서부터 만들어진다. 즉, 커널 스택에 쌓아둔 것을
    rdi로 받고있을 뿐 TCB와 동떨어져있다.




### 둘째 시도

syscall_handler가 이용하는 Stack은 syscall을 호출한 쓰레드의 TCB의 Stack 영역이다. `mov %rsp, tss->rsp0`

즉, 예외 핸들러가 실행될 때 커널스택은 user thread가 관리되는 TCB의 스택 영역이다.

그럼 그 맨 밑에는 해당 user thread의 구조체가 있을 것이다.
그러면 거기에다 미리 할당 loaded seg address 영역을 저장해두면 되는 것이다!


**그리고!!!**
`thread_current()`로 바로 TCB 시작주소를 얻을 수 있다.

왜냐면 지금 커널모드로 실행됐다 뿐이지 커널 쓰레드가 실행되고있는건 아니기 때문이다.


**특이했던 것**
syscall_hander를 실행할 때 PGSIZE로 %rsp에 주소연산을 해보면 palloc으로 할당받은 페이지를
넘겨버리니까, 나는 당연히 page fault 또는 cannot access memory(debbuger)가 발생할 줄 알았다.

근데 아무렇지도 않게 접근했다.

왜냐면 page에 접근할 수 없는 것은 페이지 테이블에 맵핑 정보가 없거나 접근권한이 없기 때문인데, 
(pintos에서는)사실 모든 페이지 테이블에는 모든 주소로의 맵핑 정보가 있다. 페이지 테이블을 위해 
palloc으로 페이지를 할당받을 때 base_pml4를 복사하기 때문이다.

단지 유저 모드로 해당 페이지 테이블에 접근 할 때는 접근 권한이 없기 때문에 page fault가 발생한다.



> [!qt] 근데 이상하네. user thread 처음에 일단 base_pml4 깔아놓고 시작하는데?
>   󱞪 
>
> 그건 맞다. 근데 그래도 상관없다. 왜냐면 **어차피 PTE값을 통해 접근 권한을 확인한다.** 
>
> **이렇게 되는 이유는 해당 Page Table을 커널 모드로 전환되었을 때도 사용해야하기 때문이다.**


### 셋째 시도


페이지 테이블을 활용한다. 어차피 ELF executable에서 load segments 할 때마다 페이지테이블에 맵핑
정보를 심어놓는다. `install_page(void* upage, void* kpage, bool writable)`


그리고 매우 친절하게도 이 페이지 테이블을 통해 접근 가능한 주소인지 확인해주는 함수가 미리 만들어져있다.
심지어 유저 모드로 접근 가능한 메모리인지 확인까지해서...


그러나 나의 두 번째 아이디어도 틀린것이 아니다. 상황을 제대로 이해하고 있다는 반증이기도 하다!!
