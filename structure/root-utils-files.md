
```bash
utils/
├── backtrace
├── pintos                      # run pintos using QEMU.
└── pintos-mkdisk
```


- `./utils/pintos`

    - The script launches Pintos (an educational OS) in **QEMU** using `qemu-system-x86_64`.
    - By default, it enables `-nographic` mode (line 118),
      which disables VGA graphics and redirects output _to the serial console for a text-based simulation._

    > [!nt] 
    >
    > QEMU (Quick Emulator) is an open-source machine emulator and virtualizer that allows you to run
    > operating systems and programs designed for one hardware architecture on another.
    >
    > It supports, 
    >
    >   - **full system emulation (running entire OSes in a virtual machine)**
    >   - **and user-mode emulation (running individual programs).**
    >
    > It's commonly used in software development, debugging, and testing for platforms like ARM, x86, or
    > RISC-V on a host system.
