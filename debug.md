



### 일반 프로그램 vs 커널 개발


- **일반 프로그램**  
  - 보통 실행 파일(ex: a.out, main.exe)에 디버깅 정보가 포함되어 있습니다.
  - GDB로 실행 파일을 직접 실행하며 디버깅합니다.

- **커널 개발(Pintos 등)**  
  - 커널은 운영체제의 핵심이기 때문에, 일반 프로그램처럼 바로 실행할 수 없습니다.
  - 커널을 실행하려면 "커널 이미지(kernel.bin)"를 메모리에 올려서 부팅해야 합니다.
  - 디버깅 정보는 커널 이미지에 포함시키면 크기가 너무 커져서, 커널 로더가 로드하지 못할 수 있습니다.
  - 그래서 **실행용(kernel.bin)**과 **디버깅용(kernel.o)** 파일을 따로 만듭니다.



### 디버깅 방법

- 커널을 실행할 때는 **kernel.bin**을 메모리에 올려서 부팅합니다.
- GDB로 디버깅할 때는 **kernel.o**를 GDB에 로드해서, 소스 코드와 심볼 정보를 활용합니다.
- GDB가 kernel.bin의 실행 상태와 kernel.o의 디버깅 정보를 연결해서, 한 줄씩 실행하거나 변수 값을 볼 수 있게 해줍니다.



1. **gdbserver로 커널 실행**  
   서버에서:
   ```
   gdbserver :1234 ./kernel.bin
   ```

2. **로컬에서 GDB 실행**  
   로컬에서:
   ```
   gdb ./kernel.bin
   ```

3. **원격 연결**  
   GDB 프롬프트에서:
   ```
   (gdb) target extended-remote localhost:1234
   ```

4. **디버깅 정보 연결**  
   GDB 프롬프트에서:
   ```
   (gdb) symbol-file ./kernel.o
   ```
   이 명령은 현재 디버깅 중인 프로세스에 kernel.o의 심볼(디버깅 정보)을 연결합니다.



### 디버깅 방법 2



- 서버에서 프로그램 실행

```bash
pintos --gdb -v -k -T 48000 -m 20   -- -q   run priority-condvar
```


- 로컬에서 아래처럼 실행

  ```bash
  gdb
  ```
    
  ```gdb
    rmt
    directory threads/build
    file threads/build/kernel.o
    info functions -n
  ```




