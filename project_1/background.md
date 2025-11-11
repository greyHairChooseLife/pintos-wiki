

> [!nt] OS 관점에서 생각하기



## Background


It is minimally functional thread system.

Your job is to extend the functionality of this system to gain a better understanding of
synchronization problems.

You will be working in
- the threads directory (primarily)
- the devices directory 

Better have basic understanding about the **synchronization**.


### knowledge

- 실행중인 processes는 linked list로 관리된다. e.g., wait_list

    - 필요에 따라 sleep을 통해 유휴자원(CPU)을 반환하기도 한다.
    - 이것을 잘 활용하기 위해 sleep_list를 활용할 수 있다.

- 이놈들한테 CPU를 잘 분배 해줘야한다.

- user process에게 제어권을 넘기면 자체적으로 되찾을 수 없다.

    - 그래서 timer(H.W)가 일정 시간마다 CPU에게 interrupt를 발생시키도록 하고, 그때마다 CPU 자원을 재분배한다.
    - 이때 재분배하는 함수(`timer_handler`)를 잘 만드는 것이 이번 과제의 관건

    - `timer_handler`가 하는 일
        
        - 다른 프로세스로 제어권 넘기기

        - wait_list와 sleep_list를 언제 어떻게 관리할 것인가?

            - 정렬이 필요한가?
            - 정렬이 필요하면 어느 타이밍에 정렬하는것이 좋을까?


        - priority에 따라 우선순위를 메기는데, 이걸 잘 만져야 한다.

            - process들은 다른 resources에 접근하기도 한다.
            - 이때 race 등 문제를 막기 위해 lock을 걸기도 하는데, 이 부분과 전반적인 스케쥴링을 잘 버무려야한다.

                - 예를들어, 어떤 resource에 lock이 걸린채로 제어권을 넘겨줬는데, 이후 다른 더 높은 우선순위의 프로세스가 접근한다면 어떻게 하는가?


        - 매우 중요한 타이밍엔 interrupt 자체를 disable시켰다가 이후 재개하기도 한다.

            - 이때 콜스택이 쌓여가며 복잡성을 추가한다. 
            - 잘하자...




## Build & Run & Test

### Build & Makefile


> [!lg] Building Pintos
>
>
>   - **First, cd into the threads directory.**
>   - **Then, issue the make command.**
>
> _This will create a build directory under threads, populate it with a
> Makefile and a few subdirectories, and then build the kernel inside._
> 
> Following the build, the following are the interesting files in the build
> directory:
> 
>
> - **Makefile:** A copy of pintos/src/Makefile.build. It describes how to build the
>   kernel.
> 
> - **kernel.o:** Object file for the entire kernel. This is the result of
>   linking object files compiled from each individual kernel source file into a
>   single object file. It contains debug information, so you can run GDB (see
>   GDB) or backtrace (Backtraces) on it.
> 
> - **kernel.bin:** Memory image of the kernel, that is, the exact bytes loaded
>   into memory to run the Pintos kernel. This is just kernel.o with debug
>   information stripped out, which saves a lot of space, which in turn keeps
>   the kernel from bumping up against a 512 kB size limit imposed by the kernel
>   loader's design.
> 
> - **loader.bin:** Memory image for the kernel loader, a small chunk of code
>   written in assembly language that reads the kernel from disk into memory and
>   starts it up. It is exactly 512 bytes long, a size fixed by the PC BIOS.
>   Subdirectories of build contain object files (.o) and dependency files (.d),
>   both produced by the compiler. The dependency files tell make which source
>   files need to be recompiled when other source or header files are changed.


#### Makefile 이해하기 1단계


> [!nt]
>
> 아래처럼 진행하면 소스코드의 내용을 가져와서 ./thread/build를 생성하고
> ./Makefile.build를 복사해 ./thread/build/Makfile로 만들어준다.
> 
> 최종적으로 이 Makefile을 이용해 all, check, grade따위의 타겟을 실행하게 된다.
>
> 따라서 `./tests/Make.tests`, `./Makefile.build`를 추가적으로 이해해야 한다.




- File Path: threads/Makefile
  ```make
  include ../Makefile.kernel
  ```

- File Path: Makefile.kernel
  ```make
  # -*- makefile -*-
  
  all:
  
  include Make.vars
  
  DIRS = $(sort $(addprefix build/,$(KERNEL_SUBDIRS) $(TEST_SUBDIRS) lib/user))
  
  all grade check: $(DIRS) build/Makefile
    cd build && $(MAKE) $@
  $(DIRS):
    mkdir -p $@
  build/Makefile: ../Makefile.build
    cp $< $@
  
  build/%: $(DIRS) build/Makefile
    cd build && $(MAKE) $*
  
  clean:
    rm -rf build
  ```



**전체 흐름**

1. `make all` 실행
2. 필요한 디렉토리들 생성 (`mkdir -p $@`)
3. `../Makefile.build`를 `build/Makefile`로 복사 (`cp $< $@`)
4. **(필요시)** `build` 디렉토리에서 복사된 Makefile로 하위 타겟 빌드 (`cd build && $(MAKE) $*`)


> [!nt] 자동 변수
>
> - `$@` : 현재 타겟 이름
> - `$<` : 첫 번째 prerequisite
> - `$*` : stem (패턴 규칙에서 `%`에 매칭된 부분)


##### 상세보기


**작업 순서**

1. **`all` 타겟 실행**
   - 타겟: `all grade check: $(DIRS) build/Makefile`
   - 의존성: `$(DIRS)`(여러 디렉토리), `build/Makefile`

2. **`$(DIRS)` 타겟들**
   - 각 디렉토리가 없으면 `mkdir -p $@`로 생성
   - 자동 변수:  
     - `$@` : 현재 타겟(예: `build/kernel`)
     - `$<` : 없음 (prerequisite 없음)

3. **`build/Makefile` 타겟**
   - prerequisite: `../Makefile.build`
   - 명령: `cp $< $@`
   - 자동 변수:  
     - `$@` : `build/Makefile`
     - `$<` : `../Makefile.build`



**(필요시) `build/%` 패턴 규칙**
   - 예: `make build/kernel` 실행 시
   - prerequisite: `$(DIRS) build/Makefile`
   - 명령: `cd build && $(MAKE) $*`
   - 자동 변수:  
     - `$@` : 예를 들어 `build/kernel`
     - `$*` : stem, 즉 `kernel`



#### Makefile 이해하기 2단계


> [!td]
>
> - [ ] `./Makefile.build`
> - [ ] `./tests/Make.tests`


##### `./Makefile.build`


- File Path: Makefile.build
  ```make
  # -*- makefile -*-
  
  SRCDIR = ../..
  
  all: os.dsk
  
  include ../../Make.config
  include ../Make.vars
  include ../../tests/Make.tests
  
  # Compiler and assembler options.
  os.dsk: CPPFLAGS += -I$(SRCDIR)/lib/kernel
  
  # Core kernel.
  include ../../threads/targets.mk
  # User process code.
  include ../../userprog/targets.mk
  # Virtual memory code.
  include ../../vm/targets.mk
  # Filesystem code.
  include ../../filesys/targets.mk
  # Library code shared between kernel and user programs.
  include ../../lib/targets.mk
  # Kernel-specific library code.
  include ../../lib/kernel/targets.mk
  # Device driver code.
  include ../../devices/targets.mk
  
  SOURCES = $(foreach dir,$(KERNEL_SUBDIRS),$($(dir)_SRC))
  OBJECTS = $(patsubst %.c,%.o,$(patsubst %.S,%.o,$(SOURCES)))
  DEPENDS = $(patsubst %.o,%.d,$(OBJECTS))
  
  threads/kernel.lds.s: CPPFLAGS += -P
  threads/kernel.lds.s: threads/kernel.lds.S
  
  kernel.o: threads/kernel.lds.s $(OBJECTS)
  	$(LD) $(LDFLAGS) -T $< -o $@ $(OBJECTS)
  
  kernel.bin: kernel.o
  	$(OBJCOPY) -O binary -R .note -R .comment -S $< $@.tmp
  	dd if=$@.tmp of=$@ bs=4096 conv=sync
  	rm $@.tmp
  
  threads/loader.o: threads/loader.S kernel.bin
  	$(CC) -c $< -o $@ $(ASFLAGS) $(CPPFLAGS) $(DEFINES) -DKERNEL_LOAD_PAGES=`perl -e 'print +(-s "kernel.bin") / 4096;'`
  
  loader.bin: threads/loader.o
  	$(LD) $(LDFLAGS) -N -e start -Ttext 0x7c00 --oformat binary -o $@ $<
  
  os.dsk: loader.bin kernel.bin
  	cat $^ > $@
  
  clean::
  	rm -f $(OBJECTS) $(DEPENDS)
  	rm -f threads/loader.o threads/kernel.lds.s threads/loader.d
  	rm -f kernel.o kernel.lds.s
  	rm -f kernel.bin loader.bin os.dsk
  	rm -f bochsout.txt bochsrc.txt
  	rm -f results grade
  
  Makefile: $(SRCDIR)/Makefile.build
  	cp $< $@
  
  -include $(DEPENDS)
  ```



##### `./tests/Make.tests`


- File Path: tests/Make.tests
  ```make
  # -*- makefile -*-
  
  include $(patsubst %,$(SRCDIR)/%/Make.tests,$(TEST_SUBDIRS))
  
  PROGS = $(foreach subdir,$(TEST_SUBDIRS),$($(subdir)_PROGS))
  TESTS = $(foreach subdir,$(TEST_SUBDIRS),$($(subdir)_TESTS))
  EXTRA_GRADES = $(foreach subdir,$(TEST_SUBDIRS),$($(subdir)_EXTRA_GRADES))
  
  OUTPUTS = $(addsuffix .output,$(TESTS) $(EXTRA_GRADES))
  ERRORS = $(addsuffix .errors,$(TESTS) $(EXTRA_GRADES))
  RESULTS = $(addsuffix .result,$(TESTS) $(EXTRA_GRADES))
  
  ifdef PROGS
  include ../../Makefile.userprog
  endif
  
  TIMEOUT = 60
  MEMORY = 20
  SWAP_DISK = 4
  
  clean::
  	rm -f $(OUTPUTS) $(ERRORS) $(RESULTS) 
  
  grade:: results
  	$(SRCDIR)/tests/make-grade $(SRCDIR) $< $(GRADING_FILE) | tee $@
  
  check:: results
  	@cat $<
  	@COUNT="`egrep '^(pass|FAIL) ' $< | wc -l | sed 's/[ 	]//g;'`"; \
  	FAILURES="`egrep '^FAIL ' $< | wc -l | sed 's/[ 	]//g;'`"; \
  	if [ $$FAILURES = 0 ]; then					  \
  		echo "All $$COUNT tests passed.";			  \
  	else								  \
  		echo "$$FAILURES of $$COUNT tests failed.";		  \
  		exit 1;							  \
  	fi
  
  results: $(RESULTS)
  	@for d in $(TESTS) $(EXTRA_GRADES); do			\
  		if echo PASS | cmp -s $$d.result -; then	\
  			echo "pass $$d";			\
  		else						\
  			echo "FAIL $$d";			\
  		fi;						\
  	done > $@
  
  outputs:: $(OUTPUTS)
  
  $(foreach prog,$(PROGS),$(eval $(prog).output: $(prog)))
  $(foreach test,$(TESTS),$(eval $(test).output: $($(test)_PUTFILES)))
  $(foreach test,$(TESTS),$(eval $(test).output: TEST = $(test)))
  
  # Prevent an environment variable VERBOSE from surprising us.
  VERBOSE =
  
  TESTCMD = pintos -v -k -T $(TIMEOUT) -m $(MEMORY)
  TESTCMD += $(SIMULATOR)
  TESTCMD += $(PINTOSOPTS)
  ifeq ($(filter userprog, $(KERNEL_SUBDIRS)), userprog)
  TESTCMD += --fs-disk=$(FSDISK)
  TESTCMD += $(foreach file,$(PUTFILES),-p $(file):$(notdir $(file)))
  endif
  ifeq ($(filter vm, $(KERNEL_SUBDIRS)), vm)
  TESTCMD += --swap-disk=$(SWAP_DISK)
  endif
  TESTCMD += -- -q 
  TESTCMD += $(KERNELFLAGS)
  ifeq ($(filter userprog, $(KERNEL_SUBDIRS)), userprog)
  TESTCMD += -f
  endif
  TESTCMD += $(if $($(TEST)_ARGS),run '$(*F) $($(TEST)_ARGS)',run $(*F))
  TESTCMD += < /dev/null
  TESTCMD += 2> $(TEST).errors $(if $(VERBOSE),|tee,>) $(TEST).output
  %.output: os.dsk
  	$(TESTCMD)
  
  %.result: %.ck %.output
  	perl -I$(SRCDIR) $< $* $@
  ```


### Run


- **Pintos 실행 프로그램:**  

    - `pintos` 명령어로 Pintos 커널을 시뮬레이터(QEMU)에서 실행할 수 있음.


- **기본 실행 방법:**  

    - `pintos argument...`  
    - 각 argument는 Pintos 커널에 전달됨.


- **테스트 실행 예시:**  

    - `cd build`  
    - `pintos run alarm-multiple`  

        - `run`: 테스트 실행 명령  
        - `alarm-multiple`: 실행할 테스트 이름


- **출력 로그 저장:**  

    - `pintos -- run alarm-multiple > logfile`  

        - `>`로 출력 결과를 파일에 저장


- **옵션 사용:**  

    - 옵션은 커널 명령어 앞에 작성, 옵션과 명령어 사이에 `--`로 구분  
    - 예시: `pintos -v -- run alarm-multiple`  

        - `-v`: VGA 화면 끄기  
        - `-s`: 시리얼 입력/출력 끄기


- **옵션/명령어 목록 확인:**  

    - `pintos` : 옵션 목록 출력  
    - `pintos -h` : 커널 명령어 목록 출력


### Run Test



**Running Tests**

  - Each project has several tests (names start with `tests`).
  - To run all tests:  
    `make check` (from the build directory)
  - This builds, runs all tests, and prints pass/fail messages and a summary.


**Running Individual Tests**

  - To run a single test:  
    `make tests/threads/alarm-multiple.result`
  - Test output goes to `.output`, verdict goes to `.result`.
  - To re-run a test, delete its `.output` file or run `make clean`.


**Test Feedback**

  - By default, feedback is shown only after each test finishes.
  - For live progress:  
    `make check VERBOSE=1`


**Test Files**

  - All tests are in `pintos/src/tests`.
