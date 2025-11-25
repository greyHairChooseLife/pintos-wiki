## Gitbook

### Calling Convention

- x86-64(64비트) 아키텍처에서 함수 호출 시 사용하는 규칙


    - **인자 전달**  

        - 함수에 값을 넘길 때,  첫 번째부터 여섯 번째까지의 인자는 레지스터에 저장해서 전달

            - `%rdi`, `%rsi`, `%rdx`, `%rcx`, `%r8`, `%r9` 


    - **함수 호출 과정**  

        - 호출자(caller)는 자신의 다음 명령어 주소(복귀 주소)를 스택에 푸시(push)합니다.
        - 그리고 함수의 첫 번째 명령어로 점프(jump)합니다.
        - 이 두 동작을 `CALL` 명령어 하나로 처리합니다.


    - **함수 실행 및 반환값**  

        - 함수(callee)가 실행됩니다.
        - 반환값이 있으면 `%rax` 레지스터에 저장합니다.


    - **함수 반환**  

        - `RET` 명령어 사용
          : 함수가 끝나면, 스택에서 복귀 주소를 `pop %rip`하고 실행





### Program Startup Details


- 커널이 사용자 프로그램을 실행시켜줄 때에도 마찬가지로 Calling Convention을 따른다.



이 설명은 **프로그램을 실행할 때 인자(예: `/bin/ls -l foo bar`)를 스택에 어떻게 배치하는지**에 대한 과정입니다.

#### 단계별 설명

1. **명령어를 단어로 분리**  
   `/bin/ls`, `-l`, `foo`, `bar`로 나눔.

2. **각 단어(문자열)를 스택 상단에 저장**  
   문자열 자체를 스택에 올림.  
   (순서는 중요하지 않음, 포인터로 접근하기 때문)

3. **각 문자열의 주소와 널 포인터(종료 표시)를 스택에 저장**  
   - 문자열 주소들을 오른쪽에서 왼쪽 순서로(push) 저장.
   - 마지막에 널 포인터를 추가해 `argv[argc]`가 널 포인터가 되도록 함(C 표준 요구사항).
   - 이 주소 배열이 `argv`가 됨.
   - 이 순서로 하면 `argv[0]`이 가장 낮은 주소에 위치.

4. **스택 정렬**  
   - 스택 포인터를 8의 배수로 맞추면(word-aligned)  
     메모리 접근이 더 빠름.

5. **레지스터 설정**  
   - `%rsi`에 `argv`(주소 배열의 시작 주소)를 저장.
   - `%rdi`에 `argc`(인자 개수)를 저장.

6. **가짜 복귀 주소(push)**  
   - 함수가 실제로 반환하지 않더라도  
     스택 프레임 구조를 맞추기 위해 복귀 주소를 하나 넣어줌. 예를들면 exit handler라던가.




#### 스택 구조


- **스택은 아래로(작은 주소 방향으로) 성장**합니다.
- 각 주소에 어떤 데이터가 저장되어 있는지 보여줍니다.


| 주소         | 이름            | 데이터         | 타입         |
|--------------|-----------------|---------------|-------------|
| 0x4747fffc   | `argv[3][...]`    | 'bar\0'       | `char[4]`     |
| 0x4747fff8   | `argv[2][...]`    | 'foo\0'       | `char[4]`     |
| 0x4747fff5   | `argv[1][...]`    | '-l\0'        | `char[3]`     |
| 0x4747ffed   | `argv[0][...]`    | '/bin/ls\0'   | `char[8]`     |
| 0x4747ffe8   |  word-align      | 0             | `uint8_t[]`   |
| 0x4747ffe0   | `argv[4]`         | 0             | `char *`      |
| 0x4747ffd8   | `argv[3]`         | 0x4747fffc    | `char *`      |
| 0x4747ffd0   | `argv[2]`         | 0x4747fff8    | `char *`      |
| 0x4747ffc8   | `argv[1]`         | 0x4747fff5    | `char *`      |
| 0x4747ffc0   | `argv[0]`         | 0x4747ffed    | `char *`      |
| 0x4747ffb8   | return address  | 0             | `void (*) ()` |

**레지스터 상태:**
- `RDI = 4` (argc)
- `RSI = 0x4747ffc0` (argv 배열 시작 주소)



##### 주요 부분

1. **문자열 데이터**
   - `argv[0][...]` (`/bin/ls\0`)
   - `argv[1][...]` (`-l\0`)
   - `argv[2][...]` (`foo\0`)
   - `argv[3][...]` (`bar\0`)
   - 각각의 문자열이 스택에 저장되어 있습니다.

2. **word-align**
   - 스택 포인터를 8의 배수로 맞추기 위한 패딩(정렬용 0).

3. **argv 포인터 배열**
   - `argv[0]`~`argv[3]`: 각 문자열의 주소를 저장.
   - `argv[4]`: 널 포인터(0)로, C 표준에 따라 `argv[argc]`가 널이어야 함.

4. **return address**
   - 가짜 복귀 주소(여기서는 0).  
     유저 프로그램이 반환될 때 OS가 종료 처리를 할 수 있도록 함.

5. 레지스터 상태

    - **RDI**: 4  
      → 인자 개수(`argc`)
    - **RSI**: 0x4747ffc0  
      → `argv` 배열의 시작 주소



### Implement the argument passing.


1. **명령줄 파싱**

   - 입력 문자열을 공백 기준으로 나눕니다. (여러 공백은 하나로 처리)
   - 첫 번째 단어는 프로그램 이름, 나머지는 인자로 저장합니다.


2. **인자 크기 제한**

   - 전체 인자 문자열이 4KB(한 페이지)를 넘지 않도록 제한합니다.


3. **새 프로세스의 스택 준비**

   - 각 인자 문자열을 스택에 복사합니다.
   - 각 문자열의 주소를 `argv` 배열에 저장합니다.
   - 마지막에 널 포인터(0)를 추가해 `argv[argc]`가 널이 되도록 합니다.
   - 스택 포인터를 8의 배수로 정렬합니다.
   - `argc`와 `argv`를 해당 레지스터에 넣습니다.


4. **프로세스 시작**

   - 준비된 스택과 인자를 가지고 새 프로세스의 entry 함수로 제어를 넘깁니다.




#### 과정 이해


- **파싱/준비 단계**: 커널 메모리(커널 스택 등)에서 작업
- **실제 인자 저장**: 유저 스택(유저 프로세스의 vm 영역)에 복사
- **유저 프로그램 실행**: 유저 스택의 인자에 접근
 

- **step by step**

    1. **명령줄 파싱 및 인자 준비**  
       - 커널(커널 스레드)이 커널 메모리(커널 스택 등)에서  
         명령줄을 파싱하고 인자 배열을 만듭니다.

    2. **유저 스택 초기화**  
       - 새로 시작할 유저 프로세스의 **유저 스택 영역**(유저 가상 메모리)에  
         커널이 준비한 인자들을 복사해서 저장합니다.
       - 즉, 커널이 직접 유저 스택에 접근해서  
         인자 문자열과 포인터 배열을 올바른 위치에 복사합니다.

    3. **프로세스 실행**  
       - 유저 프로그램이 시작되면  
         `%rdi`, `%rsi` 레지스터를 통해  
         유저 스택에 저장된 인자 정보를 사용합니다.






## PDF자료 이해하기

- Process of Running user program in OS

    1. Create process and thread
    2. Setup virtual address of the program:
       - code, data, stack

    3. Load executable file
    4. Start executables: passing parameters




### 전체 흐름 파악


1. threads/init.c

    - booting
    - arguments parsing하여 action table에 따라 순차적 실행


    - File Path: threads/init.c, 243:243
      ```c
              process_wait(process_create_initd(task));
      ```

        - `process_create_initd()`이 완료 된후에 `process_wait()`이 실행된다.
            - 이때 `process_wait()`을 적절한 시점까지 종료되지 않도록 관리해야한다.



2. userprog/process.c

    - `process_create_initd()`: 유저프로그램을 실행하는 쓰레드를 생성한다.
        - 이떄 palloc을 통해 커널영역으로부터 페이지를 할당받고 file_name을 복사해 user program 쓰레드로 전달한다.
        - 왜냐면 부모 쓰레드인 커널 쓰레드에서 해당 변수에 값을 넣을 수 있고, 이러면 문제가 될 수 있기 때문이다.

        - `thread_create(file_name, PRI_DEFAULT, initd, fn_copy);`

            - palloc으로 TCB로 사용할 Page 하나 얻고, 거기에 쓰레드 초기화
            - `schedule()`
                - `process_activate(next);`
                    - `pml4_activate(next->pml4);` CR 레지스터(aka. Page Directory Base Reg)에 next->pml4 복사
                        - next->pml4가 NULL이 아니면(TCB에 있다면) 그것을 사용.
                        - 없다면 (커널 쓰레드 전용) base_pml4 사용
                    - `tss_update(next);` tss 업데이트
                        - interrupt/syscall 핸들러는 커널 가상주소공간을 사용해야한다.
                        - 그 방법은 현재 쓰레드가 유저 쓰레드인 경우 tss->rsp0을 사용하는 것인데, 그래서 이 값을 현재 쓰레드 구조체 + PGSIZE로 업데이트한다.
                - `thread_launch(next);`
                    - 기존 쓰레드의 컨텍스트 정보를 TCB에 백업
                    - 새로운 쓰레드의 컨텍스트 정보를 CPU 레지스터에 복사 및 entry(aux)실행
                        - 잡다한건 asm으로 직접하고, 마지막에 **iretq**로 rip, cs, eflags, rsp, ss를 원자적으로 복원 및 **jump(to rip)**


---

_iretq의 실행으로 여기부턴 별도의 쓰레드가 실행된다. 단, 여전히 커널 권한의 쓰레드이다._

---


  - `initd(fn_copy)` && `process_exec()`

      - `process_init()`: 
          > [!qt] 이건 왜 있는거지?

     - interrupt frame을 새로 하나 만들고, 유저 레벨로 기본 세팅한다.
        ```c
            struct intr_frame _if;
            _if.ds = _if.es = _if.ss = SEL_UDSEG;
            _if.cs = SEL_UCSEG;
            _if.eflags = FLAG_IF | FLAG_MBS;  // Must Be Set
        ```

     - `process_cleanup()`
         - 새로운 thread를 생성한 시점에서는 하는게 없다. 아직은 `curr_thread->pml4 == NULL`이기 때문
         - 이 시점에 실행되는 이 함수는 **VM**을 다루면 그때 의미가 있겠다.(ifdef로 하나 추가 하고있음)

           > 원론적으로는 아래 기능을 수행
           >
           > - cpu의 CR(aka. PDBR)을 base_pml4로 덮어씌운다.
           > - 기존 pml4에 관련된 메모리를 모두 free한다.
           >
           >   > [!nt] PML4
           >   > In x86-64 paging, the **PML4** (Page Map Level 4) is the top-level page directory.


     - `load()`
         - file_name을 가지고 executable을 CPU에 올린다. (rip, rsp 레지스터에 값 채우기!)

         - `t->pml4 = pml4_create();`
             - base_pml4로 memcpy 한 **page(from kernel pool) 생성** 및 TCB에 주소 저장

         - `process_activate(thread_current());`
             - `pml4_activate()`: CPU의 CR3(aka. PDBR)에 해당하는 thread의 pml4(4단계 중 최상위 페이지 테이블)를 복사한다.
             - `tss_update()`: thread의 kernel stack pointer 설정. interrupt, syscall 하면 커널 주소공간의 이곳 스택을 사용한다. 
               pss->rsp0 = (thread addr + PGSIZE)

         - `filesys_open()`

             - `dir_lookup(dir, name, &inode);` 디렉토리에 name이라는 파일 있는지 찾아보고 있으면 변수 inode에 찾아낸 파일의 inode정보를 복사
             - `file_open(inode);` 동적할당 받아서 아래 file 정보 복사하고 리턴
                 - **return**: File Path: filesys/file.c, 6:11
                   ```c
                   /* An open file. */
                   struct file {
                       struct inode* inode; /* File's inode. */
                       off_t pos;           /* Current position. */
                       bool deny_write;     /* Has file_deny_write() been called? */
                   };
                   ```
                   - With this `struct file` pointer, you can perform operations like **reading from** or **writing to** the file,
                     **seeking** to different positions, checking **file size**, or **closing** the file.


                     > [!nt] inode가 뭐냐?
                     >
                     > inode(아이노드)는 파일 시스템에서 파일이나 디렉터리에 대한 정보를 저장하는 데이터 구조체입니다.  
                     > 주로 다음과 같은 정보를 담고 있습니다:
                     > 
                     > - 파일의 크기
                     > - 파일의 소유자 및 권한
                     > - 파일이 저장된 디스크 블록 위치
                     > - 생성/수정/접근 시간 등
                     > 
                     > 즉, 파일의 "메타데이터"를 담고 있는 구조체라고 볼 수 있습니다.
                     > 
                     > 중복해서 inode를 열면 안 되는 이유는 다음과 같습니다:
                     > 
                     > - **자원 낭비**: 같은 파일에 대해 여러 개의 inode 구조체를 메모리에 만들면 불필요하게 메모리를 사용하게 됩니다.
                     > - **일관성 문제**: 파일에 대한 정보(예: 참조 횟수, 쓰기 금지 등)를 여러 구조체에서 따로 관리하면, 파일 상태가 꼬일 수 있습니다.
                     > - **관리 편의**: 하나의 inode 구조체만 관리하면, 파일을 닫거나 삭제할 때 처리하기가 훨씬 쉽습니다.
                     > 
                     > 그래서 이미 열린 inode가 있으면 그걸 재사용하고, 없을 때만 새로 만드는 방식으로 동작합니다.


                    > [!nt] ELF 파일의 전체 구조
                    >
                    >
                    > ```
                    > +---------------------+
                    > | ELF 헤더            |  // struct ELF64_hdr에 해당, 파일 맨 앞
                    > +---------------------+
                    > | 프로그램 헤더 테이블 |  // 실행에 필요한 정보(메모리 적재 등), 운영체제가 사용
                    > +---------------------+
                    > | 섹션 헤더 테이블     |  // 각 섹션(코드, 데이터, 심볼 등) 정보, 링커/디버거가 사용
                    > +---------------------+
                    > | 실제 섹션 데이터     |  // .text, .data, .bss, .symtab 등
                    > +---------------------+
                    > ```
                    > 
                    > - **ELF 헤더**: 파일의 맨 앞에 위치. 파일 전체 구조와 오프셋 정보를 담고 있음.
                    > - **프로그램 헤더 테이블**: 실행 시 운영체제가 참고. 메모리 맵핑, 적재 정보 등.
                    > - **섹션 헤더 테이블**: 파일 내 각 섹션의 위치, 크기, 타입 등 정보. 개발 도구에서 주로 사용.
                    > - **섹션 데이터**: 실제 코드(.text), 데이터(.data), 심볼 테이블(.symtab) 등.


                    > [!nt] ELF 파일의 헤더
                    > 
                    > ````c
                    > struct ELF64_hdr {
                    >     unsigned char e_ident[EI_NIDENT]; // ELF 파일 식별 정보(매직 넘버, 클래스, 엔디안 등)
                    >     uint16_t e_type;                  // 파일 타입(실행 파일, 오브젝트 파일, 공유 라이브러리 등)
                    >     uint16_t e_machine;               // 대상 아키텍처(x86_64, ARM 등)
                    >     uint32_t e_version;               // ELF 포맷 버전
                    >     uint64_t e_entry;                 // 프로그램 진입점(Entry Point) 주소
                    >     uint64_t e_phoff;                 // 프로그램 헤더 테이블의 파일 내 오프셋(시작 위치)
                    >     uint64_t e_shoff;                 // 섹션 헤더 테이블의 파일 내 오프셋(시작 위치)
                    >     uint32_t e_flags;                 // 아키텍처별 플래그(특수 옵션)
                    >     uint16_t e_ehsize;                // ELF 헤더 크기(바이트 단위)
                    >     uint16_t e_phentsize;             // 프로그램 헤더 엔트리 하나의 크기
                    >     uint16_t e_phnum;                 // 프로그램 헤더 엔트리 개수
                    >     uint16_t e_shentsize;             // 섹션 헤더 엔트리 하나의 크기
                    >     uint16_t e_shnum;                 // 섹션 헤더 엔트리 개수
                    >     uint16_t e_shstrndx;              // 섹션 이름 문자열 테이블의 인덱스
                    > };
                    > ````
                    > 
                    > - `e_phoff`는 **프로그램 헤더 테이블이 파일 내에서 어디서 시작하는지(오프셋)**를 알려줍니다.  
                    > - `e_shoff`는 **섹션 헤더 테이블이 파일 내에서 어디서 시작하는지(오프셋)**를 알려줍니다.
                    > 
                    > **프로그램 헤더 테이블**: 실행 시 운영체제가 필요한 정보를 담고 있습니다. 예를 들어, 메모리에 어떤 부분을 어떻게 적재할지 등 실행에 직접 필요한 정보입니다.
                    > 
                    > **섹션 헤더 테이블**: 파일 내 각 섹션(코드, 데이터, 심볼 등)에 대한 정보를 담고 있습니다. 링커나 디버거 등 개발 도구에서 주로 사용합니다.


                    > [!nt] `ELF64_PHDR`는 **프로그램 헤더 엔트리**
                    >
                    > ELF 헤더의 `e_phoff` 값이 프로그램 헤더 테이블의 시작 위치를 알려주고,  
                    > 그 위치부터 이 구조체 형태의 엔트리들이 `e_phnum` **개수만큼** 이어집니다.
                    > 
                    > ````c
                    > struct ELF64_PHDR {
                    >     uint32_t p_type;    // 세그먼트 타입(LOAD, DYNAMIC 등)
                    >     uint32_t p_flags;   // 세그먼트 플래그(R, W, X 등 접근 권한)
                    >     uint64_t p_offset;  // 파일 내에서 세그먼트 데이터의 시작 오프셋
                    >     uint64_t p_vaddr;   // 메모리에서 세그먼트가 적재될 가상 주소
                    >     uint64_t p_paddr;   // 물리 주소(일반적으로 사용되지 않음)
                    >     uint64_t p_filesz;  // 파일 내 세그먼트 크기(바이트)
                    >     uint64_t p_memsz;   // 메모리에서 세그먼트 크기(바이트)
                    >     uint64_t p_align;   // 세그먼트 정렬 기준(메모리/파일)
                    > };
                    > ````
                    > 
                    > **설명**
                    > - 프로그램 헤더 테이블의 각 엔트리는 실행 파일의 "세그먼트" 정보를 담고 있습니다.
                    > - 운영체제는 이 정보를 참고해서 ELF 파일을 메모리에 어떻게 적재할지 결정합니다.
                    > - 예를 들어, 코드(.text), 데이터(.data) 등 각 세그먼트의 위치와 크기, 권한을 명시합니다.
                    >
                    >
                    >
                    >  > [!qt] `p_vaddr` (가상 주소)는 어떻게 미리 결정되나요?
                    >  >   󱞪 
                    >  > 최종적으로 메모리 주소를 할당하는 것은 운영체제(OS)입니다.
                    >  > 하지만 `p_vaddr`은 실제 물리 주소가 아닌, **프로세스 고유의 가상 주소 공간**에서의 **시작 주소**를 의미합니다.
                    >  > 
                    >  > - **링커(Linker)의 역할**: 컴파일 과정의 마지막 단계인 링킹 시점에, 링커는 실행 파일이 메모리에 로드될 때의 이상적인 레이아웃을 결정합니다.
                    >  >   이때 각 세그먼트가 배치될 **상대적인 가상 주소**를 계산하여 `p_vaddr`에 기록합니다. 
                    >  >   예를 들어, 코드 세그먼트는 `0x400000`부터, 데이터 세그먼트는 `0x601000`부터 시작하도록 약속하는 것과 같습니다.
                    >  > 
                    >  > - **운영체제(OS 로더)의 역할**: 프로그램이 실행될 때, OS 로더는 이 `p_vaddr` 값을 참고합니다.
                    >  >    -   **고정 주소 방식**: 과거에는 로더가 `p_vaddr`에 명시된 주소에 프로그램을 그대로 로드하려고 시도했습니다.
                    >  >    -   **주소 공간 배치 난수화 (ASLR)**: 현대 운영체제는 보안을 위해 ASLR 기술을 사용합니다.
                    >  >        이 경우, 로더는 `p_vaddr`을 직접 사용하지 않고, 실행 시점마다 임의의 기준 주소(Base Address)를 정한 뒤,
                    >  >        `p_vaddr`에 명시된 오프셋만큼 떨어진 위치에 세그먼트를 로드합니다. 즉, `p_vaddr`은 세그먼트 간의 상대적인 간격을 유지하는 데 사용됩니다.
                    >  > 
                    >  > 결론적으로, `p_vaddr`은 링커가 정한 **권장 가상 주소**이며, OS는 이를 기준으로 실제 메모리 매핑을 수행합니다.
                    >
                    >
                    >  > [!qt] `p_filesz`와 `p_memsz`는 크기가 왜 다른가요?
                    >  >   󱞪 
                    >  > `filesz`와 `memsz`가 다른 주된 이유는 **.bss 세그먼트** 때문입니다.
                    >  > 
                    >  > - `filesz` (File Size): 실행 **파일 내에서** 해당 세그먼트가 차지하는 크기입니다.
                    >  > - `memsz` (Memory Size): 프로그램이 **메모리에 로드되었을 때** 해당 세그먼트가 차지해야 할 크기입니다.
                    >  > 
                    >  > **.bss (Block Started by Symbol) 세그먼트**는 초기화되지 않은 전역 변수나 정적(static) 변수들을 저장하는 공간입니다.
                    >  > 이 변수들은 프로그램 시작 시 0으로 초기화될 값들이므로, 굳이 파일에 0으로 채워진 공간을 저장할 필요가 없습니다.
                    >  > 
                    >  > 따라서,
                    >  > 1.  링커는 `.bss`에 해당하는 데이터 세그먼트의 `filesz`를 0 또는 매우 작은 값으로 설정합니다. (파일 공간 절약)
                    >  > 2.  대신 `memsz`에는 변수들이 실제로 차지할 메모리 크기를 기록합니다.
                    >  > 3.  OS 로더는 이 정보를 보고, `memsz` 크기만큼 메모리를 할당한 뒤 그 공간을 0으로 채워줍니다.
                    >  > 
                    >  > 이로 인해 `memsz`가 `filesz`보다 커지게 됩니다.



         - `file_read(file, &ehdr, sizeof ehdr)`

             - ELF 헤더만 읽어서 `ehdr`에 복사한다.
             - ELF 포멧이 맞는지 확인한다. 안맞으면 걍 goto로 종료시켜버림


         - 파일의 프로그램 헤더(ELF 헤더와는 다르다!) 정보를 읽고, `phdr`에 복사한다.
           그리고 프로그램 헤더를 검증한 뒤, 헤더의 메타데이터에 따라 **메모리에 파일을 로딩(`load_segment(...)`)**한다.


           ```c
               /* Read program headers. */
               file_ofs = ehdr.e_phoff;
               for (i = 0; i < ehdr.e_phnum; i++)
               {
                   struct Phdr phdr;

                   if (file_read(file, &phdr, sizeof phdr) != sizeof phdr) goto done;
                   file_ofs += sizeof phdr;

               //...
           ```

             - 중간중간 예외처리도 해준다.(비정상적으로 읽었다던지) 수틀리면 꺼버림.
             - 프로그램 헤더는 타입란게 있고, e_phnum 개수만큼 있다. 모든 헤더를 찾아서 읽어준다.
             - 한번 읽을때마다,
                 - `validate_segment(&phdr, file)`
                     - ELF 실행파일의 프로그램 헤더(`ELF64_PHDR`)에는 
                       **각 세그먼트가 메모리의 어느 가상 주소에 로드되어야 하는지**에 대한 정보가 이미 정의되다.
                     - 그 정보들을 통해 OS에서 허용하는 가상주소공간의 레이아웃인지 확인하는 과정

                 - 조건문 속에서 header segment마다 얼만큼 읽어야하는지 확인해서 이후 segment읽을 때 인자로 전달
                 - `load_segment(file, file_page, (void*)mem_page, read_bytes, zero_bytes, writable)`

                     - `uint8_t* kpage = palloc_get_page(PAL_USER);` 페이지 하나를 **user pool**에서 할당받는다.
                     - `file_read(file, kpage, page_read_bytes)` 계산된 page_read_bytes만큼 할당받은 page에 쓰기한다.
                     - `memset(kpage + page_read_bytes, 0, page_zero_bytes);` 이후 페이지의 남은 부분은 0으로 채운다.
                     - `install_page(upage, kpage, writable)` upage에 해당하는 user virtual address를 kpage에 해당하는 kernerl virtual address에 대응되는 physical memory address로 맵핑한다.
                       (`t->pml4`의 엔트리로 등록되는 방식.)

                     - **최종적으로 PT_LOAD 타입에 해당하는 프로그램 헤더들은 user virtual address의 각 세그먼트가 되어 로딩된다.
                       이때 각 세그먼트는 위 아래로 zero padding이 있을 수 있으며,
                       반드시 연속되지는 않고 바다에 뜬 섬처럼 띄엄띄엄 unmapped 영역을 사이에 두게된다. (이런 메모리 영역에 접근하면 abort시켜야한다.)**


                        > [!nt] 여기 이해하는거 중요하다.
                        >
                        >
                        > - File Path: userprog/process.c, 370:385
                        >   ```c
                        >               case PT_LOAD:
                        >                   if (validate_segment(&phdr, file))
                        >                   {
                        >                       bool writable = (phdr.p_flags & PF_W) != 0;
                        >                       uint64_t file_page = phdr.p_offset & ~PGMASK;
                        >                       uint64_t mem_page = phdr.p_vaddr & ~PGMASK;
                        >                       uint64_t page_offset = phdr.p_vaddr & PGMASK;
                        >                       uint32_t read_bytes, zero_bytes;
                        >                       if (phdr.p_filesz > 0)
                        >                       {
                        >                           /* Normal segment.
                        >                            * Read initial part from disk and zero the rest. */
                        >                           read_bytes = page_offset + phdr.p_filesz;
                        >                           zero_bytes =
                        >                               (ROUND_UP(page_offset + phdr.p_memsz, PGSIZE) -
                        >                                read_bytes);
                        >   ```
                        > 
                        >
                        >
                        > > [!nt]
                        > >
                        > >  `& ~PGMASK`: `p_offset`을 페이지 크기 단위로 내림(floor)하는 연산입니다. 즉, `p_offset`이 속한 **가상 페이지 격자의 시작 주소**를 계산합니다.
                        >
                        >
                        > 핵심은 ELF 세그먼트가 두 가지 다른 크기 값을 가진다는 점을 이해하는 것입니다.
                        > *   `p_filesz`: 실행 파일(디스크상)에서 세그먼트 데이터가 차지하는 크기입니다.
                        > *   `p_memsz`: 로드되었을 때 가상 메모리에서 차지해야 하는 총 크기입니다.
                        > 
                        > 종종 `p_memsz`가 `p_filesz`보다 큽니다. 이는 초기화되지 않은 전역 변수나 정적 변수를 포함하는 `.bss` 섹션 때문입니다.
                        > 이 변수들은 파일에서는 공간을 차지하지 않지만, 메모리에는 할당되어 0으로 초기화되어야 합니다.
                        > 
                        > 따라서 이 계산 로직은 여러 페이지에 걸쳐있을 수 있는 세그먼트의 전체 메모리 영역을 다룹니다.
                        > 
                        > 1.  **`page_offset`**: 세그먼트의 시작 가상 주소(`p_vaddr`)가 그 주소가 속한 페이지의 시작 지점으로부터 얼마나 떨어져 있는지를 나타내는 오프셋입니다.
                        >     예를 들어, 페이지 크기(`PGSIZE`)가 `0x1000`이고 `p_vaddr`이 `0x1004`라면, `page_offset`은 `4`가 됩니다.
                        > 
                        > 2.  **`read_bytes`**: `page_offset + phdr.p_filesz`
                        >     *   이 값은 파일로부터 읽어올 내용으로 채워질 가상 메모리 영역의 총 크기를 나타냅니다.
                        >     *   여기에는 파일에서 직접 읽어오는 `p_filesz` 바이트와, 세그먼트가 시작되기 전 첫 페이지의 빈 공간인 `page_offset` 바이트가 포함됩니다.
                        >         `load_segment` 함수는 이 `page_offset` 만큼의 앞부분을 0으로 채우고, 그 뒤부터 파일 내용을 채워 넣게 됩니다.
                        > 
                        > 3.  **`zero_bytes`**: `(ROUND_UP(page_offset + phdr.p_memsz, PGSIZE) - read_bytes)`
                        >     *   `page_offset + phdr.p_memsz`: 메모리 내 세그먼트의 총 크기(`p_memsz`)에 첫 페이지의 오프셋(`page_offset`)을 더한 값입니다.
                        >     *   `ROUND_UP(...)`: 이 총 크기를 다음 페이지 경계로 올림(round up)합니다. 이를 통해 이 세그먼트를 담기 위해 필요한 **전체 가상 페이지들의 총 크기**를 얻습니다.
                        >     *   `... - read_bytes`: 위에서 구한 전체 페이지 크기에서 파일 내용으로 채워질 부분(`read_bytes`)을 뺍니다. 그 결과,
                        >         세그먼트의 메모리 영역 끝부분에서 명시적으로 0으로 채워져야 하는 정확한 바이트 수가 남게 됩니다. 이 부분이 바로
                        >         `.bss` 영역과 마지막 페이지를 채우기 위한 패딩(padding)에 해당합니다.
                        > 
                        > 
                        > 
                        > - 예시
                        > 
                        >     다음과 같은 상황을 가정해 보겠습니다.
                        >     *   `PGSIZE` = 4096 바이트 (0x1000)
                        >     *   `p_vaddr` = 0x8048100 (`page_offset` = 0x100)
                        >     *   `p_filesz` = 5000 바이트
                        >     *   `p_memsz` = 6000 바이트
                        >     
                        >     계산:
                        >     *   `read_bytes` = 0x100 + 5000 = 5356
                        >     *   `zero_bytes`:
                        >         1.  `page_offset + p_memsz` = 0x100 + 6000 = 6252
                        >         2.  `ROUND_UP(6252, 4096)` = 8192 (2 페이지)
                        >         3.  `zero_bytes` = 8192 - 5356 = 2836




                 - `setup_stack(if_)`

                     - `kpage = palloc_get_page(PAL_USER | PAL_ZERO);` 유저풀에서 페이지 하나 할당받는다.
                     - `install_page(((uint8_t*)USER_STACK) - PGSIZE, kpage, true);` USER_STACK에서 페이지 하나만큼 아래의 주소(유저가상주소)를 할당받은 페이지에 맵핑한다.(pml4에 맵핑)
                     - `if_->rsp = USER_STACK;` trap_frame의 rsp를 지정해준다.


                 - `if_->rip = ehdr.e_entry;` trap_frame의 rip에 로드한 실행파일의 진입점을 복사한다.




                 - **여기가 이제 argument passing이 이루어지는 지점이다.** 

                 _상식적으로 뭘 해야겠냐? 진입점에 main함수가 실행되려면 인자를 전달해줘야지. 그것은 보통의 calling convention에 따르는거고.  그 컨벤션이 바로 깃북에 안내되어있는거고._





     - `do_iret(&_if);` 새로 만든 trap frame으로 do iret실행한다.

         - 레지스터를 전달받은 trap frame의 내용으로 싹 채운다.
         - iretq 명령으로 rip, cs, eflags, rsp, ss를 atomic하게 레지스터에 채우고, rip로 jump
           이 시점엔,

             - rip: 실행파일의 진입점 (`if_->rip = ehdr.e_entry;`)
             - rsp: USER_STACK (`if_->rsp = USER_STACK;`)




