# 과제 분석

## `#ifdef VM`

- File Path: userprog/exception.c, 115:158
  ```c
  static void page_fault(struct intr_frame* f)
  {
      bool not_present; /* True: not-present page, false: writing r/o page. */
      bool write;       /* True: access was write, false: access was read. */
      bool user;        /* True: access by user, false: access by kernel. */
      void* fault_addr; /* Fault address. */
  
      /* Obtain faulting address, the virtual address that was
         accessed to cause the fault.  It may point to code or to
         data.  It is not necessarily the address of the instruction
         that caused the fault (that's f->rip). */
  
      fault_addr = (void*)rcr2();
  
      /* Turn interrupts back on (they were only off so that we could
         be assured of reading CR2 before it changed). */
      intr_enable();
  
      /* Determine cause. */
      not_present = (f->error_code & PF_P) == 0;
      write = (f->error_code & PF_W) != 0;
      user = (f->error_code & PF_U) != 0;
  
      /* 유저 영역에서 패닉은 프로세스만 종료 */
      if (user)
          exit(-1);
  
  #ifdef VM
      /* For project 3 and later. */
      if (vm_try_handle_fault(f, fault_addr, user, write, not_present))
          return;
  #endif
  
      /* Count page faults. */
      page_fault_cnt++;
  
      struct thread* curr = thread_current();
      curr->exit_status = EXIT_KERNEL;
  
      /* If the fault is true fault, show info and exit. */
      printf("Page fault at %p: %s error %s page in %s context.\n", fault_addr,
             not_present ? "not present" : "rights violation", write ? "writing" : "reading", user ? "user" : "kernel");
      kill(f);
  }
  ```


- File Path: userprog/process.c, 100:119
  ```c
  static void initd(void* aux)
  {
      struct initd_aux* initd_aux = (struct initd_aux*)aux;
      char* f_name = initd_aux->file_name;
  
  #ifdef VM
      supplemental_page_table_init(&thread_current()->spt);
  #endif
  
      thread_current()->parent = initd_aux->parent;
      thread_current()->self_metadata = initd_aux->child;
  
      palloc_free_page(initd_aux);
  
      process_init();
  
      if (process_exec(f_name) < 0)
          PANIC("Fail to launch initd\n");
      NOT_REACHED();
  }
  ```


- File Path: userprog/process.c, 202:277
  ```c
  static void __do_fork(void* aux)
  {
      struct intr_frame if_;
      struct fork_aux* f_aux = (struct fork_aux*)aux;
      struct thread* parent = f_aux->parent;
      struct thread* current = thread_current();
      struct intr_frame* parent_if = f_aux->if_parent;
      bool succ = true;
  
      /* 1. Read the cpu context to local stack. */
      memcpy(&if_, parent_if, sizeof(struct intr_frame));
  
      /* Save parent/child linkage for wait(). */
      current->parent = parent;
      current->self_metadata = f_aux->ch;
  
      /* 2. Duplicate PT */
      current->pml4 = pml4_create();
      if (current->pml4 == NULL)
          goto error;
  
      process_activate(current);
  #ifdef VM
      supplemental_page_table_init(&current->spt);
      if (!supplemental_page_table_copy(&current->spt, &parent->spt))
          goto error;
  #else
      if (!pml4_for_each(parent->pml4, duplicate_pte, parent))
          goto error;
  #endif
      /* 파일 복사 */
      lock_acquire(&filesys_lock);
      if (parent->execute_file != NULL) {
          struct file* dup_execute_file = file_duplicate(parent->execute_file);
          if (dup_execute_file == NULL) {
              lock_release(&filesys_lock);
              goto error;
          }
          current->execute_file = dup_execute_file;
      }
  
      for (int i = MIN_FD; i <= MAX_FD; i++) { // fdte 전수 조사
          if (parent->fdte[i] != NULL) {
              struct file* f = file_duplicate(parent->fdte[i]);
              if (f == NULL) {
                  lock_release(&filesys_lock);
                  goto error;
              }
  
              current->fdte[i] = f;
          }
      }
      lock_release(&filesys_lock);
  
      /* 자식 상태 설정 */
      f_aux->ch->tid = current->tid;
      f_aux->ch->status = current->status;
      f_aux->ch->exit_status = current->exit_status;
  
      if_.R.rax = 0; /* 자식 프로세스 반환 값 */
  
      process_init();
  
      /* Finally, switch to the newly created process. */
      if (succ) {
          f_aux->success = true;
          sema_up(&f_aux->loaded);
          do_iret(&if_);
      }
  error:
      f_aux->success = false;
      current->exit_status = EXIT_KERNEL;
      f_aux->ch->exit_status = EXIT_KERNEL;
      sema_up(&f_aux->loaded);
      thread_exit();
  }
  ```



- File Path: userprog/process.c, 438:462
  ```c
  static void process_cleanup(void)
  {
      struct thread* curr = thread_current();
  
  #ifdef VM
      supplemental_page_table_kill(&curr->spt);
  #endif
  
      uint64_t* pml4;
      /* Destroy the current process's page directory and switch back
       * to the kernel-only page directory. */
      pml4 = curr->pml4;
      if (pml4 != NULL) {
          /* Correct ordering here is crucial.  We must set
           * cur->pagedir to NULL before switching page directories,
           * so that a timer interrupt can't switch back to the
           * process page directory.  We must activate the base page
           * directory before destroying the process's page
           * directory, or our active page directory will be one
           * that's been freed (and cleared). */
          curr->pml4 = NULL;
          pml4_activate(NULL);
          pml4_destroy(pml4);
      }
  }
  ```



- File Path: threads/init.c, 67:127
  ```c
  int main(void)
  {
      uint64_t mem_end;
      char** argv;
  
      /* Clear BSS and get machine's RAM size. */
      bss_init();
  
      /* Break command line into arguments and parse options. */
      argv = read_command_line();
      argv = parse_options(argv);
  
      /* Initialize ourselves as a thread so we can use locks,
         then enable console locking. */
      thread_init();
      console_init();
  
      /* Initialize memory system. */
      mem_end = palloc_init();
      malloc_init();
      paging_init(mem_end);
  
  #ifdef USERPROG
      tss_init();
      gdt_init();
  #endif
  
      /* Initialize interrupt handlers. */
      intr_init();
      timer_init();
      kbd_init();
      input_init();
  #ifdef USERPROG
      exception_init();
      syscall_init();
  #endif
      /* Start thread scheduler and enable interrupts. */
      thread_start();
      serial_init_queue();
      timer_calibrate();
  
  #ifdef FILESYS
      /* Initialize file system. */
      disk_init();
      filesys_init(format_filesys);
  #endif
  
  #ifdef VM
      vm_init();
  #endif
  
      printf("Boot complete.\n");
  
      /* Run actions specified on kernel command line. */
      run_actions(argv);
  
      /* Finish up. */
      if (power_off_when_done)
          power_off();
      thread_exit();
  }
  ```
  


- File Path: include/threads/thread.h, 109:143
  ```cpp
  struct thread {
      /* Owned by thread.c. */
  
      tid_t tid;                           /* Thread identifier. */
      enum thread_status status;           /* Thread state. */
      char name[16];                       /* Name (for debugging purposes). */
      int priority;                        /* Priority. */
      int base_priority;                   /* Space for saving base priority when receiving donation*/
      struct list donations;               /* Donations */
      struct list_elem donation_elem;      /* elem to put into donation list if donation recieved or given*/
      struct lock* waiting_lock;           /* Address of Lock the thread is waiting for*/
      enum thread_exit_status exit_status; /* to keep track of exit status of process*/
      /* Shared between thread.c and synch.c. */
      struct list_elem elem; /* List element. */
  
      int64_t wakeup_tick;
  
  #ifdef USERPROG
      /* Owned by userprog/process.c. */
      uint64_t* pml4;                     /* Page map level 4 */
      struct file* fdte[MAX_FD + 1];      /* file descriptor table */
      struct thread* parent;              /* 부모 프로세스 */
      struct child_thread* self_metadata; /* 부모가 가진 현재 프로세스 메타데이터 연결고리 */
      struct list children;               /* 자식 프로세스 목록 */
      struct file* execute_file;          /* 현재 프로세스 실행한 파일 */
  #endif
  #ifdef VM
      /* Table for whole virtual memory owned by thread. */
      struct supplemental_page_table spt;
  #endif
  
      /* Owned by thread.c. */
      struct intr_frame tf; /* Information for switching */
      unsigned magic;       /* Detects stack overflow. */
  };
  ```







  ## hm...


  jj

  sdf
