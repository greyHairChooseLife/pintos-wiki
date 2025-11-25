
## 필수 이해 instructions

_processor의 ISA에 등록된 instruction을 뜻한다._

_과거 x86-32 시절에는 int 0x80을 사용했지만,
x86-64부터는 프로세서 설계자들이 syscall이라는 기능을 하드웨어 레벨에서 지원한다._



- `syscall`: _atomic execution_

    - `%MSR_STAR`에 따라 `%CS, %SS`의 CPL(Current Privilege Level)을 조정
        - user mode(ring 3)  ==>  kernel mode(ring 0)
    - `%MSR_SYSCALL_MASK`에 저장된 플래그에 따라 h/w interrupts 등 masking
    - `%MSR_LSTAR`에 들어있는 진입점 함수를 호출




- `sysret`: _atomic execution_

    - `%MSR_STAR`에 따라 `%CS, %SS`의 CPL(Current Privilege Level)을 조정
        - user mode(ring 3)  <==  kernel mode(ring 0)
    - `%RCX`의 값을 `%RIP`에 복사
    - `%R11`의 값을 `%RFLAGS`에 복사




## 흐름 요약 및 코드

### `syscall_init()`

- `%MSR_LSTAR` <- `CS`, `SS` 세그먼트 설정값(CPL; Current Privileged Level) 복사
    - syscall 호출: CS, SS를 kernel mode로 변경하는데 사용
    - sysret 호출 : CS, SS를 user mode로 변경하는데 사용

- `%MSR_STAR` <- `&syscall_entry` 복사
    - syscall 호출을 실행하면 `%rip`에 이 값을 복사하고 `jump`

- `%MSR_SYSCALL_MASK` <- h/w interrupt masking 정보 복사
    - _syscall 호출이 실행 될 때마다 h/w interrupt block_

#### code

- File Path: userprog/syscall.c, 14:37
  ```c
  /* System call.
   *
   * Previously system call services was handled by the interrupt handler
   * (e.g. int 0x80 in linux). However, in x86-64, the manufacturer supplies
   * efficient path for requesting the system call, the `syscall` instruction.
   *
   * The syscall instruction works by reading the values from the the Model
   * Specific Register (MSR). For the details, see the manual. */
  
  #define MSR_STAR 0xc0000081         /* Segment selector msr */
  #define MSR_LSTAR 0xc0000082        /* Long mode SYSCALL target */
  #define MSR_SYSCALL_MASK 0xc0000084 /* Mask for the eflags */
  
  void syscall_init(void) {
      write_msr(MSR_STAR,
                ((uint64_t)SEL_UCSEG - 0x10) << 48 | ((uint64_t)SEL_KCSEG) << 32);
      write_msr(MSR_LSTAR, (uint64_t)syscall_entry);
  
      /* The interrupt service rountine should not serve any interrupts
       * until the syscall_entry swaps the userland stack to the kernel
       * mode stack. Therefore, we masked the FLAG_FL. */
      write_msr(MSR_SYSCALL_MASK,
                FLAG_IF | FLAG_TF | FLAG_DF | FLAG_IOPL | FLAG_AC | FLAG_NT);
  }
  ```



### _(user process에서)_ `syscall()`
  

- user virtual memory의 공유 영역에 존재하는 표준 라이브러리를 호출하는 개념

- `#include` 해서 사용하는거


#### code

- File Path: lib/user/syscall.c, 5:43

  **key point: 이것이 시스템콜을 호출**
    - syscall vector number부터 시작해서 argv를 필요한 만큼 받는다.
    - 받은 것을 `%rax, %rdi, ...` 등 syscall 호출규약에 따라 레지스터에 저장한다.
    - syscall vector number는 `%rax`에 담는다.
    - syscall instruction을 호출한다.
    - syscall 핸들러가 종료되면 반환값이 %rax에 담겨있을텐데, 그것을 ret변수에 담는다. `"=a"(ret)`
    - 값을 리턴한다.(유저 프로그램이 사용)

  ```c
  __attribute__((always_inline)) static __inline int64_t syscall(uint64_t num_,
                                                                 uint64_t a1_,
                                                                 uint64_t a2_,
                                                                 uint64_t a3_,
                                                                 uint64_t a4_,
                                                                 uint64_t a5_,
                                                                 uint64_t a6_) {
      int64_t ret;
      register uint64_t* num asm("rax") = (uint64_t*)num_;
      register uint64_t* a1 asm("rdi") = (uint64_t*)a1_;
      register uint64_t* a2 asm("rsi") = (uint64_t*)a2_;
      register uint64_t* a3 asm("rdx") = (uint64_t*)a3_;
      register uint64_t* a4 asm("r10") = (uint64_t*)a4_;
      register uint64_t* a5 asm("r8") = (uint64_t*)a5_;
      register uint64_t* a6 asm("r9") = (uint64_t*)a6_;
  
      __asm __volatile(
          "mov %1, %%rax\n"
          "mov %2, %%rdi\n"
          "mov %3, %%rsi\n"
          "mov %4, %%rdx\n"
          "mov %5, %%r10\n"
          "mov %6, %%r8\n"
          "mov %7, %%r9\n"
          "syscall\n"
          : "=a"(ret)
          : "g"(num), "g"(a1), "g"(a2), "g"(a3), "g"(a4), "g"(a5), "g"(a6)
          : "cc", "memory");
      return ret;
  }
  
  /* Invokes syscall NUMBER, passing no arguments, and returns the
     return value as an `int'. */
  #define syscall0(NUMBER) (syscall(((uint64_t)NUMBER), 0, 0, 0, 0, 0, 0))
  
  /* Invokes syscall NUMBER, passing argument ARG0, and returns the
     return value as an `int'. */
  #define syscall1(NUMBER, ARG0) \
      (syscall(((uint64_t)NUMBER), ((uint64_t)ARG0), 0, 0, 0, 0, 0))
  ```



### `syscall_entry()`

- user stack을 kernel stack으로 전환; `mov tss->rsp0, %rsp`

- 기존 (cpu registers)컨텍스트를 stack에 push
    - interrupt_frame 생성 (향후 이것은 syscall_handler에 전달)
    - 그것을 `%rdi`에 복사해둠

- _중단된 h/w interrupt 재활성화_

- syscall_handler 호출
    - 스택에 쌓아둔 interrupt_frame이 `%rdi`에 들어음
    - 반환값이 있다면 `%rax`에 담아주는데, 이 interrupt_frame에 넣는 방식

- handler 호출이 반환되면 _사용자 모드로 복귀_
    - 레지스터 값 복구 `pop & sysret` 


#### code

- File Path: userprog/syscall-entry.S, 6:12

  **key point: user stack을 kernel stack으로 전환**

  ```asm
  syscall_entry:
  	movq %rbx, temp1(%rip)
  	movq %r12, temp2(%rip)     /* callee saved registers */
  	movq %rsp, %rbx            /* Store userland rsp    */
  	movabs $tss, %r12
  	movq (%r12), %r12
  	movq 4(%r12), %rsp         /* Read ring0 rsp from the tss */
  ```


- File Path: userprog/syscall-entry.S, 13:39
  
  **key point: 기존 (cpu registers)컨텍스트를 stack에 push**

  ```asm
  	/* Now we are in the kernel stack */
  	push $(SEL_UDSEG)      /* if->ss */
  	push %rbx              /* if->rsp */
  	push %r11              /* if->eflags */
  	push $(SEL_UCSEG)      /* if->cs */
  	push %rcx              /* if->rip */
  	subq $16, %rsp         /* skip error_code, vec_no */
  	push $(SEL_UDSEG)      /* if->ds */
  	push $(SEL_UDSEG)      /* if->es */
  	push %rax
  	movq temp1(%rip), %rbx
  	push %rbx
  	pushq $0
  	push %rdx
  	push %rbp
  	push %rdi
  	push %rsi
  	push %r8
  	push %r9
  	push %r10
  	pushq $0 /* skip r11 */
  	movq temp2(%rip), %r12
  	push %r12
  	push %r13
  	push %r14
  	push %r15
  	movq %rsp, %rdi
  ```


- File Path: userprog/syscall-entry.S, 41:44

  **key point: 중단된 h/w interrupt 재활성화**

  ```asm
  check_intr:
  	btsq $9, %r11          /* Check whether we recover the interrupt */
  	jnb no_sti
  	sti                    /* restore interrupt */
  ```



- File Path: userprog/syscall-entry.S, 45:68

  **key point: syscall_handler를 호출하고, 완료시 사용자 모드로 복구**

  ```asm
  no_sti:
  	movabs $syscall_handler, %r12
  	call *%r12
  	popq %r15
  	popq %r14
  	popq %r13
  	popq %r12
  	popq %r11
  	popq %r10
  	popq %r9
  	popq %r8
  	popq %rsi
  	popq %rdi
  	popq %rbp
  	popq %rdx
  	popq %rcx
  	popq %rbx
  	popq %rax
  	addq $32, %rsp
  	popq %rcx              /* if->rip */
  	addq $8, %rsp
  	popq %r11              /* if->eflags */
  	popq %rsp              /* if->rsp */
  	sysretq
  ```



### `syscall_handler( interrupt_frame )`

- argvs 검증
    - 유효한 주소값 등

- %rax에 있는 syscall vector number를 확인해서 handler 호출





## Syscall List

### halt

걍 종료

### exit


- [ ] TCB에 전달받은 status 상태 저장
- [ ] 자원정리
    - [ ] 메모리(페이지 디렉토리)
    - [ ] 파일 디스크립터
    - [ ] 세마포어, 뮤텍스 등 lock




## QnA

###  이 어셈블리 코드에 대해서 설명해봐. 특히 아래 부분에 대해서 말이야.

> - File Path: lib/user/syscall.c, 30:33
>   ```c
>           : "=a"(ret)
>           : "g"(num), "g"(a1), "g"(a2), "g"(a3), "g"(a4), "g"(a5), "g"(a6)
>           : "cc", "memory");
>       return ret;
>   ```

---

해당 코드는 C 코드 내에서 인라인 어셈블리(inline assembly)를 사용하여 시스템 콜을 호출하는 부분의 제약 조건(constraints)을 정의합니다. 이 부분은 컴파일러에게 C 변수와 레지스터를 어떻게 연결하고, 어셈블리 코드가 어떤 상태를 변경하는지 알려주는 역할을 합니다.

각 줄의 의미는 다음과 같습니다.

*   `: "=a"(ret)` (출력 피연산자)
    *   `=`는 이 피연산자가 쓰기 전용(출력)임을 의미합니다.
    *   `"a"`는 `rax` 레지스터를 가리키는 제약 조건입니다.
    *   `(ret)`은 C 변수 `ret`를 의미합니다.
    *   종합하면, 어셈블리 코드 실행 후 `rax` 레지스터에 담긴 값을 C 변수 `ret`에 저장하라는 의미입니다. 시스템 콜의 반환 값은 관례적으로 `rax` 레지스터를 통해 전달됩니다.

*   `: "g"(num), "g"(a1), ...` (입력 피연산자)
    *   이 줄은 어셈블리 코드에 입력으로 전달될 C 변수들을 지정합니다.
    *   `"g"`는 변수를 범용 레지스터(general-purpose register)나 메모리 중 컴파일러가 가장 효율적이라고 판단하는 곳에 위치시키라는 제약 조건입니다.
    *   어셈블리 코드 본문의 `%1`, `%2`, ... `%7`은 각각 C 변수 `num`, `a1`, ... `a6`에 해당합니다. (`%0`은 출력 피연산자인 `ret`에 해당)

*   `: "cc", "memory"` (변경되는 레지스터 목록, Clobber List)
    *   이 부분은 인라인 어셈블리 코드가 출력 피연산자 외에 어떤 레지스터나 상태를 변경하는지 컴파일러에게 알려줍니다.
    *   `"cc"`: `syscall` 명령어는 플래그 레지스터(EFLAGS)의 내용을 변경할 수 있으므로, 이를 명시하여 컴파일러가 해당 레지스터 값을 신뢰하지 않도록 합니다.
    *   `"memory"`: 어셈블리 코드가 명시되지 않은 메모리 영역을 읽거나 쓸 수 있음을 알립니다. 이는 컴파일러가 이 어셈블리 코드 전후로 메모리 접근 순서를 임의로 변경하는 최적화를 수행하지 못하도록 막는 메모리 장벽(memory barrier) 역할을 합니다.



### halt syscall을 구현하려고 해.


> 근데 관련 설명에 아래처럼 나와. 어차피 시스템을 종료하는데 deadlock situation 정보를 잃든 말든 무슨 상관이 있어?
> 
> Terminates Pintos by calling power_off() (declared in src/include/threads/init.h). This should be seldom used, because you lose some information about possible deadlock situations, etc.
> 

---

아주 좋은 지적입니다. 시스템이 어차피 종료될 텐데, 데드락 정보를 잃는 것이 왜 문제가 되는지 궁금해하시는 것은 당연합니다.

결론부터 말씀드리면, 그 정보는 **실행 중인 다른 프로세스를 위한 것이 아니라, 운영체제를 개발하고 디버깅하는 개발자를 위한 것**입니다.

`power_off()`를 호출하여 시스템을 즉시 종료하는 것은 컴퓨터의 전원 플러그를 뽑아버리는 것과 같습니다. 이 경우 다음과 같은 문제가 발생합니다.

1.  **디버깅 정보 소실**
    *   만약 시스템이 비정상적인 상태(예: 여러 스레드가 서로 락을 잡고 기다리는 **데드락** 상태)에 빠져있을 때 `halt`가 호출되면, 시스템은 그냥 꺼져버립니다.
    *   개발자는 왜 데드락이 발생했는지, 어떤 스레드가 어떤 자원을 점유하고 있었는지 분석할 기회를 완전히 잃게 됩니다.
    *   정상적인 종료 절차나 커널 패닉(kernel panic)은 이런 문제 상황에 대한 정보(메모리 덤프, 레지스터 상태 등)를 화면이나 로그 파일에 남겨주어 원인 분석을 돕습니다.

2.  **데이터 손실**
    *   파일 시스템 버퍼에 아직 디스크로 기록되지 않은 데이터가 있다면, 그 데이터는 영구적으로 손실됩니다.
    *   이는 파일 손상으로 이어질 수 있습니다. 정상적인 종료 절차는 모든 버퍼를 디스크에 동기화(sync)하는 과정을 포함합니다.

따라서 설명에 나온 "seldom used (거의 사용되지 않아야 한다)"라는 경고는 다음과 같은 의미입니다.

> "이 기능은 시스템을 강제로 꺼버리는 최후의 수단이다. 개발 및 테스트 과정에서는 시스템의 문제를 분석하는 데 필요한 중요한 단서들을 모두 날려버리기 때문에, 정말 필요한 경우가 아니면 사용하지 않는 것이 좋다."




