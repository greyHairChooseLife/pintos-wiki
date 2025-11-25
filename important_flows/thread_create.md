# thread creation flow



## 관련 코드

- File Path: threads/thread.c, 650:704
  ```c
  /* 스레드를 전환(Switching)합니다. */
  static void thread_launch(struct thread* th) {
      uint64_t tf_cur = (uint64_t)&running_thread()->tf;
      uint64_t tf = (uint64_t)&th->tf;
      ASSERT(intr_get_level() == INTR_OFF);
  
      /* 주된 전환 로직(switching logic). */
      __asm __volatile(
          /* 사용될 레지스터 저장. */
          "push %%rax\n"
          "push %%rbx\n"
          "push %%rcx\n"
          /* 입력을 한 번에 가져오기 */
          "movq %0, %%rax\n"
          "movq %1, %%rcx\n"
          "movq %%r15, 0(%%rax)\n"
          "movq %%r14, 8(%%rax)\n"
          "movq %%r13, 16(%%rax)\n"
          "movq %%r12, 24(%%rax)\n"
          "movq %%r11, 32(%%rax)\n"
          "movq %%r10, 40(%%rax)\n"
          "movq %%r9, 48(%%rax)\n"
          "movq %%r8, 56(%%rax)\n"
          "movq %%rsi, 64(%%rax)\n"
          "movq %%rdi, 72(%%rax)\n"
          "movq %%rbp, 80(%%rax)\n"
          "movq %%rdx, 88(%%rax)\n"
          "pop %%rbx\n"  // 저장된 rcx
          "movq %%rbx, 96(%%rax)\n"
          "pop %%rbx\n"  // 저장된 rbx
          "movq %%rbx, 104(%%rax)\n"
          "pop %%rbx\n"  // 저장된 rax
          "movq %%rbx, 112(%%rax)\n"
          "addq $120, %%rax\n"
          "movw %%es, (%%rax)\n"
          "movw %%ds, 8(%%rax)\n"
          "addq $32, %%rax\n"
          "call __next\n"  // 현재 rip 읽기.
          "__next:\n"
          "pop %%rbx\n"
          "addq $(out_iret -  __next), %%rbx\n"
          "movq %%rbx, 0(%%rax)\n"  // rip
          "movw %%cs, 8(%%rax)\n"   // cs
          "pushfq\n"
          "popq %%rbx\n"
          "mov %%rbx, 16(%%rax)\n"  // eflags
          "mov %%rsp, 24(%%rax)\n"  // rsp
          "movw %%ss, 32(%%rax)\n"
          "mov %%rcx, %%rdi\n"
          "call do_iret\n"
          "out_iret:\n"
          :
          : "g"(tf_cur), "g"(tf)
          : "memory");
  }
  ```



- File Path: threads/thread.c, 621:648
  ```c
  /* iretq를 사용하여 스레드를 실행(launch)합니다 */
  void do_iret(struct intr_frame* tf) {
      __asm __volatile(
          "movq %0, %%rsp\n"
          "movq 0(%%rsp),%%r15\n"
          "movq 8(%%rsp),%%r14\n"
          "movq 16(%%rsp),%%r13\n"
          "movq 24(%%rsp),%%r12\n"
          "movq 32(%%rsp),%%r11\n"
          "movq 40(%%rsp),%%r10\n"
          "movq 48(%%rsp),%%r9\n"
          "movq 56(%%rsp),%%r8\n"
          "movq 64(%%rsp),%%rsi\n"
          "movq 72(%%rsp),%%rdi\n"
          "movq 80(%%rsp),%%rbp\n"
          "movq 88(%%rsp),%%rdx\n"
          "movq 96(%%rsp),%%rcx\n"
          "movq 104(%%rsp),%%rbx\n"
          "movq 112(%%rsp),%%rax\n"
          "addq $120,%%rsp\n"
          "movw 8(%%rsp),%%ds\n"
          "movw (%%rsp),%%es\n"
          "addq $32, %%rsp\n"
          "iretq"
          :
          : "g"((uint64_t)tf)
          : "memory");
  }
  ```


## 이해하기 위한 LLM과의 대화

### Pintos 스레드 컨텍스트 스위칭 심층 분석 노트

이 문서는 Pintos 운영체제에서 스레드가 생성되고, 컨텍스트 스위칭이 일어나는 과정의 내부 동작을 심층적으로 분석하고 정리합니다.

---

#### 1. 스레드 초기화와 스택 설정

새로운 스레드가 생성될 때, 가장 먼저 하는 일 중 하나는 스택 포인터(`rsp`)를 초기화하는 것입니다.

```c
// in init_thread()
t->tf.rsp = (uint64_t)t + PGSIZE - sizeof(void*);
```

이 코드는 여러 중요한 의미를 담고 있습니다.

*   **스택의 위치**: 스레드 제어 블록(`struct thread`, `t`)은 페이지(`PGSIZE`, 보통 4KB)의 맨 아래 주소에 할당됩니다. 스택은 이 페이지의 맨 위에서부터 아래로 자라납니다.
*   **스택의 시작점**: `(uint64_t)t + PGSIZE`는 할당된 페이지의 최상단 경계 바로 다음 주소를 가리킵니다. (예: `0x1000`에서 시작했다면 `0x2000`)
*   **`sizeof(void*)`를 빼는 이유**: 이것이 가장 혼란스러웠던 부분입니다.
    1.  **안전한 첫 Push**: 만약 `rsp`를 페이지의 최상단 주소(`0x1FFF`)로 설정하면, 첫 `push` 동작 시 스택 포인터가 페이지 경계를 넘어갈 수 있습니다. `0x1FF8`에서 시작하면 첫 `push`가 안전하게 페이지 내부에 데이터를 저장합니다.
    2.  **스택 정렬 (Stack Alignment)**: x86-64 ABI(호출 규약)는 함수 호출 시 스택 포인터가 **16바이트 단위로 정렬**될 것을 요구합니다. `sizeof(void*)` (8바이트)를 빼는 것은 이러한 정렬 규칙을 맞추기 위한 관례적인 단계 중 하나입니다.

아래는 스레드 페이지의 메모리 구조입니다.

```text
(High Address)
0x...2000  <-- 페이지 경계 (접근 불가)
0x...1FF8  <-- 초기 RSP (Stack Pointer) 위치
           |
           | 스택(Stack)이 이 방향으로 자라남 (주소 감소)
           V
           .
           .
           .
0x...1000  <-- 스레드 구조체(TCB) 시작 위치 (t)
(Low Address)
```

#### 2. Trap Frame (`tf`)과 `kernel_thread`

*   `tf`는 **Trap Frame**의 약자입니다. 인터럽트나 컨텍스트 스위칭이 발생했을 때, CPU의 모든 레지스터 상태(rip, rsp, rflags 등)를 저장하기 위한 구조체입니다.
*   새 스레드의 `rip`(다음에 실행할 명령어 포인터)에 실제 함수가 아닌 `kernel_thread`를 넣는 이유는 다음과 같습니다.
    1.  **표준화된 실행 환경**: `kernel_thread`는 스레드 함수에 인자를 전달하고, 함수 실행이 끝나면 `thread_exit()`를 호출하여 스레드를 안전하게 종료시키는 **래퍼(Wrapper) 함수** 역할을 합니다.
    2.  **자원 정리**: 만약 스레드 함수를 직접 실행하면, 함수가 `return` 했을 때 스레드 자원을 누가 어떻게 정리할지 보장할 수 없습니다. `kernel_thread`가 이 문제를 해결해 줍니다.

#### 3. 컨텍스트 스위칭의 핵심: `thread_launch` 어셈블리

`thread_launch`는 현재 실행 중인 스레드의 상태를 저장하고, 다음 스레드의 상태를 복원하는 실제 컨텍스트 스위칭을 수행합니다. 이 과정은 C언어로는 불가능하여 어셈블리로 작성되었습니다.

##### 3.1. `rip` 값 얻기: `call __next` 트릭

가장 이해하기 어려웠던 부분입니다. `rip` 레지스터는 직접 읽을 수 없기 때문에 아래와 같은 트릭을 사용합니다.

```asm
call __next
__next:
pop %%rbx
```

1.  **`call __next`**: 이 명령어는 두 가지 일을 합니다.
    *   **다음 명령어의 주소**를 스택에 push합니다. 여기서 "다음 명령어"는 바로 `__next:` 레이블이 가리키는 `pop %%rbx`입니다. 따라서 **`__next:`의 주소가 스택에 저장됩니다.**
    *   `__next:` 레이블로 점프합니다.
2.  **`pop %%rbx`**: `__next:`로 점프한 직후, 스택에서 방금 저장한 값(즉, `__next:`의 주소)을 꺼내 `rbx` 레지스터에 넣습니다.

이 트릭을 통해, 우리는 현재 코드의 실행 위치를 `rbx` 레지스터에 성공적으로 가져올 수 있습니다.

##### 3.2. 복귀 주소(`out_iret`) 계산 및 저장

```asm
addq $(out_iret -  __next), %%rbx
movq %%rbx, 0(%%rax)  // rip
```

1.  `rbx`에 저장된 `__next:` 주소에 `out_iret`과 `__next` 사이의 거리(오프셋)를 더합니다.
2.  결과적으로 `rbx`는 `out_iret` 레이블의 주소를 갖게 됩니다.
3.  이 주소를 현재 스레드의 trap frame(`tf_cur`)의 `rip` 필드에 저장합니다.

**결론**: 현재 스레드는 나중에 다시 스케줄링될 때, `out_iret:` 레이블부터 실행을 재개하게 됩니다.

##### 3.3. `eflags` 저장하기

`rip`와 마찬가지로 `eflags` 레지스터도 직접 읽을 수 없으므로 스택을 이용한 트릭을 사용합니다.

```asm
pushfq        ; 현재 eflags를 스택에 push
popq %%rbx    ; 스택에서 eflags 값을 rbx로 pop
mov %%rbx, 16(%%rax) ; rbx 값을 trap frame의 eflags 필드에 저장
```

##### 3.4. 컨텍스트 전환 실행

```asm
mov %%rcx, %%rdi
call do_iret
out_iret:
```

1.  **`mov %%rcx, %%rdi`**: `do_iret` 함수를 호출하기 위해, 첫 번째 인자인 **새로운 스레드의 trap frame 주소(`th->tf`)**를 `rdi` 레지스터에 준비시킵니다.
2.  **`call do_iret`**: `do_iret` 함수는 새로운 trap frame의 내용으로 모든 레지스터를 복원하고, 마지막에 `iretq` 명령어를 실행합니다.
3.  **`iretq`**: 이 특수 명령어는 스택에 저장된 `rip`, `cs`, `eflags`, `rsp`, `ss`를 CPU 레지스터로 한 번에 복원하고, **새로운 스레드의 `rip`에서 실행을 재개**합니다. 이 순간, CPU는 완전히 새로운 스레드의 컨텍스트로 전환됩니다.
4.  **`out_iret:`**: 현재 스레드가 나중에 복귀할 지점입니다.

#### 4. 세그먼트 레지스터 (`cs`, `ss`)의 진짜 역할

*   **`cs` (Code Segment)**: 현재 코드가 **커널 모드(Ring 0)**인지 **사용자 모드(Ring 3)**인지를 결정합니다. 이는 특권 명령 실행 가능 여부를 결정하는 핵심입니다.
*   **`ss` (Stack Segment)**: `rsp`가 스택의 절대 주소를 가지고 있더라도, `ss`는 해당 스택이 **어떤 권한 수준(DPL)을 가졌는지**를 명시합니다.
*   **왜 둘 다 필요한가?**: CPU는 **현재 코드의 권한 수준(CPL, `cs`로 결정)과 스택의 권한 수준(DPL, `ss`로 결정)이 반드시 일치해야 한다**는 규칙을 하드웨어적으로 강제합니다. 커널 코드가 사용자 스택에 접근하는 것을 막기 위함입니다. 따라서 커널 모드로 전환될 때는 `cs`와 `ss` 모두 커널용 세그먼트로 변경되어야 합니다.

#### 5. GCC 인라인 어셈블리 문법

```c
: "g"((uint64_t)tf)
: "memory");
```

*   **`(uint64_t)tf`**: C 포인터 타입(`tf`)을 어셈블리가 이해할 수 있는 순수한 64비트 주소값(정수)으로 변환합니다.
*   **`"g"`**: "General Operand" 제약 조건. 컴파일러에게 값을 레지스터, 메모리, 상수 중 가장 효율적인 방법으로 전달하라고 지시합니다.
*   **`"memory"`**: "Clobber" 리스트. 이 어셈블리 코드가 메모리를 예측 불가능하게 변경함을 컴파일러에게 알립니다. 이는 컴파일러의 과도한 최적화(코드 재배치 등)를 막는 **메모리 배리어** 역할을 하여 코드의 안정성을 보장합니다.
