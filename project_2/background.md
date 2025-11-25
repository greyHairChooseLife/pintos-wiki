## Notice

- Main objectives are enabling programs to **interact** with the OS **via system calls**.
- No skeleton code. All designs up to myself.
- I can freely modify any source codes.
- Already Implemented:
    - loading
    - running
- **Yet no** Implemented:
    - I/O
    - interactivity
- Most In Use
    - `userprog/`
    - etc



## Background

- This project deals with user programs, which means there is restrictions accessing parts of the system.
    - This is difference between implementation a kernel area.

- Under user process,

    - Each process has **one thread**. No multithreading supported.

- Never locates code in the block that enclosed by `#ifdef VM`.

- _Strongly recommend to read synchronization and virtual addrees._



## Source Files


```bash
ls /home/sy/jg/04.pintos/7team/userprog/
```

Makefile
targets.mk


- `process.c`

    - Loads ELF binaries and starts processes.

    > [!nt] What is ELF?
    > Executable and Linkable Format, 리눅스에서 사용되는 실행 파일 포맷


- `syscall.c`

    - system call handlers


- `syscall-entry.S`
    
    - Small assembly codes that bootstrap the system call handler. You don't need to understand this code.


- `exception.c`

    _When a user process performs a privileged or prohibited operation,
    it traps into the kernel as an exception or fault_

    - This handle these **exceptions**. 


- `gdt.c`

    - The Global Descriptor Table (GDT) is a table that describes the segments in use.
    - _No need_ to modify for anything.
    

- `tss.c`

    The Task-State Segment (TSS) was used for x86 architectural task switching.
    However, in x86-64, task switching is deprecated. Nontheless, TSS is _still
    there to find the stack pointer during the ring switching._

    This means _when a user process enters an **interrupt handler**, the hardware
    consults to the tss to **look up the kernel's stack pointer**._

    - _No need_ to modify for anything.




## Using the File System


- **To deal with system call, it should interface with the file system a lot.**

- _No need_ to modify the file system.

- Simple file system is provided in the `filesys`.

    - **Check the interfaces** in the `filesys.h` and `file.h` to understand **how to use it**.
    - `filesys_remove()` are implemented; 

        > 파일이 삭제되어도, 그 파일을 이미 열고 있는 스레드(프로세스)는 계속 파일에
        > 접근할 수 있습니다.  즉, 파일의 데이터(블록)는 즉시 삭제되지 않고, 마지막으로
        > 파일을 닫을 때까지 남아 있습니다.
        >
        > 파일을 열고 있던 모든 스레드가 파일을 닫으면, 그때서야 파일의 데이터가 실제로
        > 디스크에서 제거(블록 해제)됩니다.

    - Understand **limitations**.

        > [!nt] Limitations
        > 
        > - **No internal synchronization**  
        >   파일 시스템 코드에 동시 접근(여러 프로세스가 동시에 접근) 시 충돌이 발생할 수
        >   있습니다. 동기화(예: 락)를 사용하지 않으므로, 한 번에 하나의 프로세스만 파일
        >   시스템 코드를 실행해야 안전합니다.
        > 
        > - **File size is fixed at creation time**  
        >   파일을 만들 때 크기가 고정되며, 이후에 크기를 변경할 수 없습니다. 루트
        >   디렉터리도 파일로 표현되기 때문에, 생성할 수 있는 파일의 개수도 제한됩니다.
        > 
        > - **File data is allocated as a single extent**  
        >   파일의 데이터가 디스크에서 연속된 공간(섹터)에 저장됩니다. 파일 시스템을 오래
        >   사용하면 외부 단편화(쓸 수 있는 공간이 여기저기 흩어짐) 문제가 심각해질 수
        >   있습니다.
        > 
        > - **No subdirectories**  
        >   하위 디렉터리(폴더)를 만들 수 없습니다. 모든 파일이 루트 디렉터리에만
        >   존재합니다.
        > 
        > - **File names are limited to 14 characters**  
        >   파일 이름은 최대 14자까지 가능합니다.
        > 
        > - **A system crash mid-operation may corrupt the disk**  
        >   파일 시스템 작업 중 시스템이 다운되면 디스크가 손상될 수 있으며, 자동으로
        >   복구할 방법이 없습니다. 복구 도구도 제공되지 않습니다.



### Running Test


_Unlike Project.1,_

- _Need to put test programs_(which runs in the user space) _into the Pintos virtual machine._
- `make check` will handle this

- However, **to run individual test cases;**

  1. create simulated disk with a file system partition.
     - `pintos-mkdisk` program will do it.

  2. 


- **결국은 pintos program이 사용할 디스크를 생성(pintos-mkdisk)하고,
  그 생성한 것을 pintos실행시 전달해준다.**

- 추가적인 명령을 전달해서 해당 파일시스템을 format도 해야한다. 완료되면 알아서 프로그램이 종료된다.

  - file system을 format하는것은 무엇인가?
    :해당 파일시스템의 데이터가 어떤 file format으로 관리될 것인지 지정한다. 예를들면 ELF.
   
- 이 가상의 파일시스템에 파일을 넣고 뺼 수 있다.

- 테스트케이스 등 user program을 이렇게 넣어서 실행시키는 것이 이번 프로젝트의 구동 방식이다.




## How User Programs Work


- **파일 시스템 디스크 관리**  

  - 디버깅 중에 파일 시스템 디스크(`filesys.dsk`)가 망가질 수 있습니다.

  - 이를 대비해, 깨끗한(초기 상태의) 참조용 파일 시스템 디스크를 만들어 두고,
    필요할 때마다 복사해서 사용하는 것이 좋습니다.



## Virtual Memory Layout


- Virtual memory in Pintos is divided into two regions:

    - User virtual memory
        - address 0 up to KERN_BASE(`0x8004000000`)
        - defined in `include/threads/vaddr.h`
        
    - kernel virtual memory
        - rest of the virtual address space.



- _**User virtual** memory is per-process._

    - When the kernel switches from one process to another (aka **context switching**),

        - it also switches user virtual address spaces
          by changing the **processor's page directory base register** 

        - this is implemented `pml4_activate() in thread/mmu.c`.

        - `struct thread` has member containing the pointer for this.



- _**Kernel virtual** memory is global._

    - 커널이 사용하는 가상 주소 공간은 모든 프로세스와 커널 스레드에서 항상 동일하게 매핑
      _즉, 어떤 프로세스나 스레드가 실행 중이든 커널의 가상 주소는 항상 같은 물리 주소를 가리킴_

    - Pintos에서는 커널 가상 메모리가 **KERN_BASE**라는 특정 가상 주소부터 시작

        - _이 주소부터 물리 메모리와 1:1로 매핑_
        - ex)  
          - 가상 주소 `KERN_BASE` → 물리 주소 `0`
          - 가상 주소 `KERN_BASE + 0x1234` → 물리 주소 `0x1234`

    - 이 매핑은 시스템의 실제 물리 메모리 크기만큼 이어짐



## Typical Memory Layout


- The code segment in Pintos starts at user virtual address 0x400000,
  approximately 128 MB from the bottom of the address space. 

- This value was typical value in ubuntu and _has no deep significance_.

- To view the layout of a particular executable, run objdump with the -p option.

### example

- File Path: hi.c
  ```c
  #include <stdio.h>
  
  int main() {
      printf("hellow");
      return 0;
  }
  ```

- File Path: , 2:64
  ```
  hi.out:     file format elf64-x86-64
  
  Program Header:
      PHDR off    0x0000000000000040 vaddr 0x0000000000000040 paddr 0x0000000000000040 align 2**3
           filesz 0x0000000000000310 memsz 0x0000000000000310 flags r--
    INTERP off    0x00000000000003b4 vaddr 0x00000000000003b4 paddr 0x00000000000003b4 align 2**0
           filesz 0x000000000000001c memsz 0x000000000000001c flags r--
      LOAD off    0x0000000000000000 vaddr 0x0000000000000000 paddr 0x0000000000000000 align 2**12
           filesz 0x0000000000000640 memsz 0x0000000000000640 flags r--
      LOAD off    0x0000000000001000 vaddr 0x0000000000001000 paddr 0x0000000000001000 align 2**12
           filesz 0x0000000000000165 memsz 0x0000000000000165 flags r-x
      LOAD off    0x0000000000002000 vaddr 0x0000000000002000 paddr 0x0000000000002000 align 2**12
           filesz 0x00000000000000cc memsz 0x00000000000000cc flags r--
      LOAD off    0x0000000000002dd0 vaddr 0x0000000000003dd0 paddr 0x0000000000003dd0 align 2**12
           filesz 0x0000000000000248 memsz 0x0000000000000250 flags rw-
   DYNAMIC off    0x0000000000002de0 vaddr 0x0000000000003de0 paddr 0x0000000000003de0 align 2**3
           filesz 0x00000000000001e0 memsz 0x00000000000001e0 flags rw-
      NOTE off    0x0000000000000350 vaddr 0x0000000000000350 paddr 0x0000000000000350 align 2**3
           filesz 0x0000000000000040 memsz 0x0000000000000040 flags r--
      NOTE off    0x0000000000000390 vaddr 0x0000000000000390 paddr 0x0000000000000390 align 2**2
           filesz 0x0000000000000024 memsz 0x0000000000000024 flags r--
      NOTE off    0x00000000000020ac vaddr 0x00000000000020ac paddr 0x00000000000020ac align 2**2
           filesz 0x0000000000000020 memsz 0x0000000000000020 flags r--
  0x6474e553 off    0x0000000000000350 vaddr 0x0000000000000350 paddr 0x0000000000000350 align 2**3
           filesz 0x0000000000000040 memsz 0x0000000000000040 flags r--
  EH_FRAME off    0x000000000000200c vaddr 0x000000000000200c paddr 0x000000000000200c align 2**2
           filesz 0x0000000000000024 memsz 0x0000000000000024 flags r--
     STACK off    0x0000000000000000 vaddr 0x0000000000000000 paddr 0x0000000000000000 align 2**4
           filesz 0x0000000000000000 memsz 0x0000000000000000 flags rw-
     RELRO off    0x0000000000002dd0 vaddr 0x0000000000003dd0 paddr 0x0000000000003dd0 align 2**0
           filesz 0x0000000000000230 memsz 0x0000000000000230 flags r--
  
  Dynamic Section:
    NEEDED               libc.so.6
    INIT                 0x0000000000001000
    FINI                 0x0000000000001158
    INIT_ARRAY           0x0000000000003dd0
    INIT_ARRAYSZ         0x0000000000000008
    FINI_ARRAY           0x0000000000003dd8
    FINI_ARRAYSZ         0x0000000000000008
    GNU_HASH             0x00000000000003d0
    STRTAB               0x0000000000000498
    SYMTAB               0x00000000000003f0
    STRSZ                0x000000000000008f
    SYMENT               0x0000000000000018
    DEBUG                0x0000000000000000
    PLTGOT               0x0000000000003fe8
    PLTRELSZ             0x0000000000000018
    PLTREL               0x0000000000000007
    JMPREL               0x0000000000000628
    RELA                 0x0000000000000568
    RELASZ               0x00000000000000c0
    RELAENT              0x0000000000000018
    FLAGS_1              0x0000000008000000
    VERNEED              0x0000000000000538
    VERNEEDNUM           0x0000000000000001
    VERSYM               0x0000000000000528
    RELACOUNT            0x0000000000000003
  
  Version References:
    required from libc.so.6:
      0x09691a75 0x00 03 GLIBC_2.2.5
      0x069691b4 0x00 02 GLIBC_2.34
  ```




## Accessing User Memory


**커널이 시스템 콜을 처리할 때 사용자 프로그램이 제공한 포인터를 어떻게 안전하게 다뤄야 하는지**


### 주요 내용

- **사용자 포인터의 위험성**  

    사용자 프로그램이 시스템 콜로 커널에 포인터를 넘길 때 잘못된 포인터를 넘길 수 있습니다.

    - 널 포인터(null pointer)
    - 매핑되지 않은 가상 메모리(unmapped virtual memory)
    - 커널 영역(KERN_BASE 이상)의 주소  
    - 등등


- **커널의 책임**  

    이런 잘못된 포인터를 사용하면 커널이나 다른 프로세스가 손상될 수 있으므로,  
    반드시 해당 프로세스를 종료하고 자원을 해제해야 합니다.


### 두 가지 처리 방법

1. **포인터 유효성 검사 후 접근**  

     - 포인터가 올바른지 먼저 확인한 뒤, 메모리에 접근합니다.
     - 관련 함수: `thread/mmu.c`, `include/threads/vaddr.h`
     - 구현이 간단합니다.


2. **KERN_BASE 이하만 확인 후 접근**  

     - 포인터가 KERN_BASE 아래인지 확인만 하고 바로 접근합니다.
     - 만약 잘못된 포인터라면 CPU가 "페이지 폴트(page fault)"를 발생시킵니다.
     - 이 경우, `userprog/exception.c`의 `page_fault()`에서 예외를 처리합니다.
     - MMU의 하드웨어 기능을 활용하므로 더 빠르고, 실제 운영체제(리눅스 등)에서 많이 사용됩니다.


### 리소스 누수 방지


- 시스템 콜 처리 중 락을 잡거나 메모리를 할당한 뒤,
  잘못된 포인터를 만나면 반드시 락을 해제하거나 메모리를 반환해야 합니다.

- 첫 번째 방법(유효성 검사 후 접근)은 처리하기 쉽지만,
  두 번째 방법(페이지 폴트 처리)은 예외 상황에서 리소스 해제가 더 어렵습니다.




### 함수 설명

#### `get_user`

- **역할**: 사용자 가상 주소(`uaddr`)에서 바이트를 읽어옴.
- **동작**:
  - 어셈블리 코드로 직접 메모리 접근 시도.
  - 만약 접근이 성공하면 해당 바이트 값을 반환.
  - 접근 중 페이지 폴트가 발생하면 `-1`을 반환.
- **전제**: `uaddr`가 KERN_BASE 아래임을 이미 확인했다고 가정.

#### `put_user`

- **역할**: 사용자 가상 주소(`udst`)에 바이트(`byte`)를 씀.
- **동작**:
  - 어셈블리 코드로 직접 메모리 쓰기 시도.
  - 성공하면 `true` 반환.
  - 페이지 폴트가 발생하면 `false` 반환.
- **전제**: `udst`가 KERN_BASE 아래임을 이미 확인했다고 가정.

### 페이지 폴트 처리

- 이 함수들이 제대로 동작하려면,  
  **커널에서 페이지 폴트가 발생했을 때**  
  - `rax` 레지스터를 `-1`로 설정하고,
  - 폴트가 발생한 명령어 위치(`%rip`)로 복귀하도록  
  `page_fault()`를 수정해야 합니다.
