## 다양한 프로그램 실행법


```bash
/workspace/userprog/build# pintos-mkdisk fs.dsk 10   # file system 만들기
/workspace/userprog/build# pintos -- -f -q ls        # root dir 살펴보기
/workspace/userprog/build# cat > my.txt              # 텍스트 파일 생성
hello, world!

/workspace/userprog/build# pintos -p my.txt -- -q ls # root dir에 파일 복사 및 커널로 ls명령
/workspace/userprog/build# pintos -- -q cat my.txt   # cat 찍어보기
/workspace/userprog/build# pintos -- -q rm  my.txt   # 파일 삭제
```



### pintos OS에 사용할 파일시스템 생성 및 주입

- 기본 file system이 `fs.dsk`로 잡혀있다. 만들어두면 아래처럼 생략하고 사용 가능.

```bash
/workspace/userprog/build# pintos-mkdisk fs.dsk 10 # file system 만들기
/workspace/userprog/build# pintos -- -f -q         # 생성한 디스크를 포멧  (--fs-disk fs.dsk 생략)
/workspace/userprog/build# pintos --    -q ls      # 루트 디렉토리 살펴보기(--fs-disk fs.dsk 생략)
```


- File Path: utils/pintos, 232:232
  ```python
      parser.add_argument("--fs-disk", default="fs.dsk", help="Set FS disk file or size")
  ```


### pintos OS에 전달할 수 있는 옵션들


- File Path: utils/pintos, 203:272
  ```python
  if __name__ == "__main__":
      import argparse
  
      parser = argparse.ArgumentParser(
          description="a utility for running Pintos in a simulator"
      )
      parser.add_argument(
          "-v",
          "--no-vga",
          action="store_true",
          default=True,
          help="No VGA display or keyboard",
      )
      parser.add_argument(
          "-k",
          "--kill-on-failure",
          action="store_true",
          help="Kill Pintos a few seconds after a kernel or user"
          "panic, test failure, or triple fault (deprecated)",
      )
      parser.add_argument(
          "-T",
          "--timeout",
          type=int,
          default=0,
          help="Kill Pintos after N seconds CPU time",
      )
  
      parser.add_argument("-m", "--memory", type=int, default=256, help="memory capacity")
      parser.add_argument("--fs-disk", default="fs.dsk", help="Set FS disk file or size")
      parser.add_argument(
          "--swap-disk", default="swap.dsk", help="Set SWAP disk file or size"
      )
      parser.add_argument(
          "-p",
          "--put-file",
          dest="HOSTFNS",
          nargs=1,
          action="append",
          default=[],
          help='Copy HOSTFN into VM, splited by ":".'
          " (e.g. tests/userprog/args-none:args-none",
      )
      parser.add_argument(
          "-g",
          "--get-file",
          dest="GUESTFNS",
          nargs=1,
          action="append",
          default=[],
          help="Copy GUESTFN out of VM, by default under same name",
      )
      parser.add_argument(
          "--mnts",
          dest="MNTS",
          nargs=1,
          action="append",
          default=[],
          help="Additional mounting disks",
      )
      parser.add_argument(
          "--gdb", action="store_true", default=False, help="Debug with gdb"
      )
      parser.add_argument(
          "-t",
          "--threads-tests",
          action="store_true",
          default=False,
          help="Run proj1 test cases with USERPROG flag",
      )
  ```



### (pintos말고) kernel로 할 수 있는 일; run actions


- 아래처럼 다양한 파일시스템 명령을 사용할 수 있다.

  ```bash
  # /home/sy/jg/04.pintos/7team/filesys/fsutil.c
  fsutil_ls    # ls 
  fsutil_cat   # cat filename
  fsutil_rm    # rm  filename
  fsutil_put   # put filename
  fsutil_get   # get filename

  # 이 밖에도 run_task 함수를 "run" 인자로 실행가능 
  ```


- 이것은 pintos 실행파일이 아닌, pintos 실행파일을 통해 시뮬레이션한 kernel이 사용할 수 있는 것이다. 
  따라서 `--` 이후로 **옵션이 아닌** 명령으로 전달해주면 사용할 수 있다.

  ```bash
  /workspace/userprog/build# pintos -- -f -q ls
  /workspace/userprog/build# pintos -- -f -q cat my_file
  ```


### (pintos말고) kernel에 전달할 수 있는 옵션들

```c
/* ARGV[]의 옵션들을 파싱하고
   옵션이 아닌 첫 번째 인자를 반환합니다. */
static char** parse_options(char** argv) {
    for (; *argv != NULL && **argv == '-'; argv++)
    {
        char* save_ptr;
        char* name = strtok_r(*argv, "=", &save_ptr);
        char* value = strtok_r(NULL, "", &save_ptr);

        if (!strcmp(name, "-h"))
            usage();
        else if (!strcmp(name, "-q"))
            power_off_when_done = true;
#ifdef FILESYS
        else if (!strcmp(name, "-f"))
            format_filesys = true;
#endif
        else if (!strcmp(name, "-rs"))
            random_init(atoi(value));
        else if (!strcmp(name, "-mlfqs"))
            thread_mlfqs = true;
#ifdef USERPROG
        else if (!strcmp(name, "-ul"))
            user_page_limit = atoi(value);
        else if (!strcmp(name, "-threads-tests"))
            thread_tests = true;
#endif
        else
            PANIC("unknown option `%s' (use -h for help)", name);
    }

    return argv;
}
```
