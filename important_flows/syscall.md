# syscall flow



**요약**

_- 모드가 바뀌고, 스택 영역이 바뀌고, rip가 바뀌지 running thread가 바뀌진 않는다._


- (booting 시) `%MSR` 셋업
    - `%MSR_STAR`: `%CS`, `%SS` 변경값
        - `syscall` 호출시 `%CS`, `%SS`의 CPL(Curernt Privilege Level)을 kernel mode로 설정
        - `sysret`  호출시 `%CS`, `%SS`의 CPL(Curernt Privilege Level)을 user mode로 설정
    - `%MSR_LSTAR`
        - `syscall` 호출시 진입점
    - `%MSR_SYSCALL_MASK`
        - `syscall` 호출시 masking 할 인터럽트 등
        - pintos에선 h/w interrupt


- user level에서 library의 syscall 호출
    - `%rax`에 syscall vector number 복사
    - `%rdi, %rsi, %rdx, %r10, %r8, %r9`에 차례로 전달 인자 복사
    - `syscall`
        - `%MSR_START`에 따라 `%CS`, `%SS` CPL 변경
        - `%MSR_SYSCALL_MASK`에 따라 마스킹
        - `%MSR_LSTAR`의 주소를 `%rip`에 복사(jump)
    - syscall 호출 반환시, %rax의 값을 반환
        - syscall_handler가 `%rax`에 반환값 복사함


- syscall 진입점인 kernel 코드 실행(syscall_entry)
    - 커널 스택 영역 확보
        - `tss->rsp0`의 주소값으로 `%rsp` 업데이트
    - CPU의 현재 register 값들로 스택에 trap frame 구조체 생성 및 `%rdi`에 복사(syscall_handler가 인자로 받게 됨)
    - `syscall` instruction 호출시 masking 한 요소 재활성화
    - syscall_handler 실행
        - 반환값 `%rax`에 저장
    - 완료시 스택에서 기존 user level 컨텍스트를 복원(CPU register에다가)
    - `sysretq`
        - `%rip, %cs, %efags, %rsp, %ss` 복구
        - `%MSR_START`에 따라 `%CS`, `%SS` CPL 변경(user level로 복구)




## 관련 코드


### File Path: userprog/syscall.c, 31:49

- **OS 부팅시 system call 관련 setup 과정**

  ```c
  // %MSR은 syscall 관련 레지스터인데, 역할별로 여러개가 있다.
  //  이 함수에서는 3개의 %MSR을 세팅해준다.
  void syscall_init(void) {
      // syscall, sysret instructions을 각각 호출할 때
      // %cs, %ss의 CPL(Current Privilege Level)을 어떻게 설정할지 설정
      write_msr(MSR_STAR,
                ((uint64_t)SEL_UCSEG - 0x10) << 48 | ((uint64_t)SEL_KCSEG) << 32);
  
      // syscall instruction 호출 진입점 설정
      write_msr(MSR_LSTAR, (uint64_t)syscall_entry);
  
      /* The interrupt service rountine should not serve any interrupts
       * until the syscall_entry swaps the userland stack to the kernel
       * mode stack. Therefore, we masked the FLAG_FL. */
      // syscall instruction 호출 시 마스킹할 요소들
      //  e.g Hardware Interrupts
      write_msr(MSR_SYSCALL_MASK,
                FLAG_IF | FLAG_TF | FLAG_DF | FLAG_IOPL | FLAG_AC | FLAG_NT);
  }
  ```




### File Path: lib/user/syscall.c, 5:34

- **user 영역에 존재하는 system call library**
- **우리가 include해서 사용하는 standard library에 해당한다.**

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
          "mov %1, %%rax\n"  // syscall vector number
          "mov %2, %%rdi\n"  // 인자 1
          "mov %3, %%rsi\n"  // 인자 2
          "mov %4, %%rdx\n"  // 인자 3
          "mov %5, %%r10\n"  // 인자 4
          "mov %6, %%r8\n"   // 인자 5
          "mov %7, %%r9\n"   // 인자 6
          "syscall\n"
          : "=a"(ret)  // syscall의 리턴값을 ret 변수에 복사
          : "g"(num), "g"(a1), "g"(a2), "g"(a3), "g"(a4), "g"(a5), "g"(a6)
          : "cc", "memory");
      return ret;
  }
  ```




### File Path: include/threads/interrupt.h, 19:64

- **interrupt frame 자료구조**

  ```cpp
  struct gp_registers {
      uint64_t r15;
      uint64_t r14;
      uint64_t r13;
      uint64_t r12;
      uint64_t r11;
      uint64_t r10;
      uint64_t r9;
      uint64_t r8;
      uint64_t rsi;
      uint64_t rdi;
      uint64_t rbp;
      uint64_t rdx;
      uint64_t rcx;
      uint64_t rbx;
      uint64_t rax;
  } __attribute__((packed));
  
  struct intr_frame {
      /* Pushed by intr_entry in intr-stubs.S.
         These are the interrupted task's saved registers. */
      struct gp_registers R;
      uint16_t es;
      uint16_t __pad1;
      uint32_t __pad2;
      uint16_t ds;
      uint16_t __pad3;
      uint32_t __pad4;
      /* Pushed by intrNN_stub in intr-stubs.S. */
      uint64_t vec_no; /* Interrupt vector number. */
                       /* Sometimes pushed by the CPU,
                          otherwise for consistency pushed as 0 by intrNN_stub.
                          The CPU puts it just under `eip', but we move it here. */
      uint64_t error_code;
      /* Pushed by the CPU.
         These are the interrupted task's saved registers. */
      uintptr_t rip;
      uint16_t cs;
      uint16_t __pad5;
      uint32_t __pad6;
      uint64_t eflags;
      uintptr_t rsp;
      uint16_t ss;
      uint16_t __pad7;
      uint32_t __pad8;
  } __attribute__((packed));
  ```



### File Path: userprog/syscall-entry.S

- **`syscall` instruction 호출시 진입점**

  ```asm
  #include "threads/loader.h"
  
  .text
  .globl syscall_entry
  .type syscall_entry, @function
  syscall_entry:
  	movq %rbx, temp1(%rip)
  	movq %r12, temp2(%rip)     /* callee saved registers */
  	movq %rsp, %rbx            /* Store userland rsp    */
  	movabs $tss, %r12
  	movq (%r12), %r12
  	movq 4(%r12), %rsp         /* Read ring0 rsp from the tss */
  	/* Now we are in the kernel stack */
  	push $(SEL_UDSEG)      /* if->ss */
  	push %rbx              /* if->rsp */
  	push %r11              /* if->eflags */
  	push $(SEL_UCSEG)      /* if->cs */
  	push %rcx              /* if->rip */
  	subq $16, %rsp         /* skip error_code, vec_no */
  	push $(SEL_UDSEG)      /* if->ds */
  	push $(SEL_UDSEG)      /* if->es */
  	push %rax              // syscall vector number
  	movq temp1(%rip), %rbx
  	push %rbx
  	pushq $0
  	push %rdx              // syscall 인자 3
  	push %rbp
  	push %rdi              // syscall 인자 1
  	push %rsi              // syscall 인자 2
  	push %r8               // syscall 인자 5
  	push %r9               // syscall 인자 6
  	push %r10              // syscall 인자 4
  	pushq $0 /* skip r11 */
  	movq temp2(%rip), %r12
  	push %r12
  	push %r13
  	push %r14
  	push %r15
  	movq %rsp, %rdi        // syscall_handler가 사용할 interrupt_frame 인자를 셋업(h/w interrupt가 다시 살아남)
  
  check_intr:
  	btsq $9, %r11          /* Check whether we recover the interrupt */
  	jnb no_sti
  	sti                    /* restore interrupt */
  no_sti:
  	movabs $syscall_handler, %r12   // syscall 핸들러 실행
  	call *%r12
  	popq %r15   // syscall 핸들러 실행이 완료되면 stack에 넣어둔 registers 값을 복구한다.
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
  	sysretq  // %MSR에 따라 %CS, %SS의 CPL을 user mode로 복구 및 %rip, %rflags 복구
  
  .section .data
  .globl temp1
  temp1:
  .quad	0
  .globl temp2
  temp2:
  .quad	0
  ```



### File Path: userprog/syscall.c, 76:238

- **syscall_entry에서 호출하는 handler**

  ```c
  void syscall_handler(struct intr_frame* f UNUSED) {
      struct thread* curr = thread_current();
  
      switch (f->R.rax)
      {
          case SYS_HALT: {
              power_off();
          }
  
          case SYS_EXIT: {
              // TODO:: 자원회수 해야함
              // - 각종 대기열에서 제거
              // - 세마포어
              // - (페이지 테이블의 맵핑된 영역 및 페이지 디렉토리 자체는 이미
              // 구현됨)
  
              curr->exitStatus = f->R.rdi;
  
              // WARN::
              // 현재는 thread_exit하면 바로 TCB까지 회수된다.
              // 일단 좀비 상태로 기다려야하고, 부모가 wait를 호출해서
              // exitStatus를 받아줄 때까지 대기한다.
              // - 세마포어 등 활용 필요
              thread_exit();
          }
  
          case SYS_CREATE: {
              char* fileName = (char*)f->R.rdi;
              unsigned initialSize = f->R.rsi;
  
              if (!is_valid_address(fileName))
              {
                  f->R.rax = false;
                  break;
              }
  
              f->R.rax = filesys_create(fileName, initialSize);
              break;
          }
  
          case SYS_REMOVE: {
              char* fileName = (char*)f->R.rdi;
  
              if (!is_valid_address(fileName))
              {
                  f->R.rax = false;
                  break;
              }
  
              f->R.rax = filesys_remove(fileName);
              break;
          }
  
          case SYS_WRITE: {
              int fd = (int)f->R.rdi;
              void* buffer = (void*)f->R.rsi;
              unsigned size = (unsigned)f->R.rdx;
  
              if (!is_valid_address_buffer(buffer, size))
              {
                  f->R.rax = -1;  // gitbook에 자료가 없다. 표준을 따른다.
                  break;
              }
  
              switch (fd)
              {
                  // 표준 입력(키보드)에 쓰기 할 수는 없다.
                  case 0: {
                      f->R.rax = -1;
                      break;
                  }
  
                  // 터미널에 한글자씩 출력
                  case 1: {
                      putbuf(buffer, size);
                      f->R.rax = size;
                      break;
                  }
  
                  default:
                      f->R.rax =
                          (unsigned)file_write(curr->fdTable[fd], buffer, size);
              }
  
              break;
          }
  
          case SYS_OPEN: {
              int fd = findFD(curr->fdTable);
  
              f->R.rax = fd;
  
              if (fd == -1) break;
  
              char* fileName = (char*)f->R.rdi;
              struct file* opened = filesys_open(fileName);
  
              if (opened)
                  curr->fdTable[fd] = opened;
              else
                  f->R.rax = -1;  // fail
  
              break;
          }
  
          case SYS_READ: {
              int fd = (int)f->R.rdi;
              char* buffer = (char*)f->R.rsi;
              unsigned size = (unsigned)f->R.rdx;
              unsigned read;  // 실제로 읽은 길이
  
              if (!is_valid_address_buffer(buffer, size))
              {
                  f->R.rax = -1;  // gitbook에 자료가 없다. 표준을 따른다.
                  break;
              }
  
              switch (fd)
              {
                  // 표준 입력(키보드 읽기)
                  case 0: {
                      for (read = 0; read < size; read++)
                      {
                          buffer[read] = (char)input_getc();
  
                          // 개행문자를 만나면 탈출
                          if (buffer[read] == '\n')
                          {
                              f->R.rax = read + 1;
                              break;  // for 반복만 빠져나간다.
                          }
                      }
  
                      // 끝까지 읽은 경우인지 아닌지 확인 필요
                      if (size == read) f->R.rax = size;
                      break;
                  }
  
                  // 표준 출력(터미널)을 읽을수는 없다.
                  case 1: {
                      f->R.rax = -1;
                      break;
                  }
  
                  // 일반적인 파일 읽기
                  default:
                      f->R.rax =
                          (unsigned)file_read(curr->fdTable[fd], buffer, size);
              }
  
              break;
          }
  
          default: thread_exit();
      }
      // SYS_FORK,     /* Clone current process. */
      // SYS_EXEC,     /* Switch current process. */
      // SYS_WAIT,     /* Wait for a child process to die. */
      // SYS_FILESIZE, /* Obtain a file's size. */
      // SYS_SEEK,     /* Change position in a file. */
      // SYS_TELL,     /* Report current position in a file. */
      // SYS_CLOSE,    /* Close a file. */
  }
  ```
