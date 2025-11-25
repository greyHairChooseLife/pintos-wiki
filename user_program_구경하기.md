
## 유저프로그램 톺아보기

- memory map
    ```bash
    userprog$ objdump -p args-single | bat -l asm
    ```

- user program 어셈블리에서 Label만 모아보기
    ```bash
    userprog$ objdump -d args-single |gr -- :$ | sort |bat -l asm
    ```

- user program 어셈블리에서 Label만 모아보기
    ```bash
    userprog$ objdump -d args-single |gr -- :$ | sort |bat -l asm
    ```
