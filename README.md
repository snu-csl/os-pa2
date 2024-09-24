# 4190.307 Operating Systems (Fall 2024)
# Project #2: System Calls
### Due: 11:59 PM, October 6 (Sunday)

## Introduction

System calls are interfaces that allow user applications to request various services from the operating system kernel. This project aims to explore and understand how system calls are implemented in the `xv6` operating system.

## Background

### RISC-V Trap Architecture

In RISC-V, a ___trap___ is a general term that encompasses both ___exceptions___ and ___interrupts___ (see Chap. 4 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2024-2/book-riscv-rev4.pdf)). Exceptions are typically generated  by the CPU itself in response to events such as illegal instructions, memory access faults, system call invocations via the `ecall` instruction, etc. Interrupts, by contrast, are triggered by external signals from devices or timers that indicate they require immediate attention. 

RISC-V supports multiple privilege levels (machine, supervisor, and user modes), each with its own set of capabilities and restrictions. Traps can be configured to be handled at different privilege levels depending on their type and the processor's current operating mode. When a trap occurs, the RISC-V hart automatically fills a register (`mcause` or `scause` depending on the privilege level) with a value that indicates the reason for the trap, as shown in the following table. 

| Interrupt | Exception code | Description                     |
|----------:|---------------:|:--------------------------------|                 
| 0         | 0              | Instruction address misaligned  |
| 0         | 1              | Instruction access fault        |
| 0         | 2              | Illegal instruction             |
| 0         | 3              | Breakpoint                      |
| 0         | 4              | Load address misaligned         |
| 0         | 5              | Load access fault               |
| 0         | 6              | Store/AMO address misaligned    |
| 0         | 7              | Store/AMO access fault          |
| 0         | 8              | Environment call from U-mode    |
| 0         | 9              | Environment call from S-mode    |
| 0         | 10             | _Reserved_                      |
| 0         | 11             | Environment call from M-mode (`mcause` only)    |
| 0         | 12             | Instruction page fault          |
| 0         | 13             | Load page fault                 |
| 0         | 14             | _Reserved_                      |
| 0         | 15             | Store/AMO page fault            |
| 0         | >= 16          | _Reserved_                      |
| 1         | 0              | _Reserved_                      |
| 1         | 1              | Supervisor software interrupt   |
| 1         | 2              | _Reserved_                      |
| 1         | 3              | Machine software interrupt (`mcause` only)     |
| 1         | 4              | _Reserved_                      |
| 1         | 5              | Supervisor timer interrupt      |
| 1         | 6              | _Reserved_                      |
| 1         | 7              | Machine timer interrupt (`mcause` only)        |
| 1         | 8              | _Reserved_                      |
| 1         | 9              | Supervisor external interrupt   |
| 1         | 10             | _Reserved_                      |
| 1         | 11             | Machine external interrupt (`mcause` only)     |
| 1         | >= 12          | _Reserved_                      |

Besides the `mcause` (or `scause`) register, various additional registers are used to handle traps; when a trap is taken into M-mode (or S-mode), `mepc` (or `sepc`) register is written with the virtual address of the instruction that was interrupted or that encountered the exception. The `mtvec` (or `stvec`) register holds the start address of the trap handler in M-mode (or S-mode). Also, the `mstatus` (or `sstatus`) register keeps track of important information such as M-mode or S-mode interrupt-enable bits (`MIE` or `SIE` bit), the value of the interrupt-enable bit active prior to the trap (`MPIE` or `SPIE` bit), and the previous privilege mode (`MPP` or `SPP` bits). 

To increase performance, certain exceptions and interrupts can be handled at a lower privilege level. For example, setting a bit in `medeleg` or `mideleg` register will delegate the corresponding trap or interrupt, when occurring in S-mode or U-mode, to the S-mode trap handler. By default, `xv6` delegates all interrupts and exceptions to S-mode.

### Handling system calls in `xv6`

Each system call in `xv6` is assigned a unique number, as you can see in `kernel/syscall.h`. This number is used to identify which system call is being requested by a user program. When a user program makes a system call, it places the system call number in a designated register, `a7`, and the arguments for the system call in other registers from `a0` to `a6`. 

The user program then executes a special `ecall` instruction, which triggers a trap from U-mode to S-mode, transferring the control to the trap handler `uservec() @ kernel/trampoline.S`. After saving user register contexts and switching into the kernel address space, the RISC-V hart jumps into `usertrap() @ kernel/trap.c`. Note that both system calls and interrupts originating from U-mode are directed to `usertrap()`. If the `sret` (return-from-S-mode) instruction is executed at the end of the system call handler, the control returns to U-mode. 
Additionally, to ensure the program resumes execution at the instruction following the `ecall` instruction, the value of `sepc` should be explicitly incremented by 4 before executing the `sret` instruction. 

The same mechanism can be used for the kernel running in S-mode to request services from M-mode. Specifically, when the kernel executes the `ecall` instruction in S-mode, control is transferred to the M-mode trap handler (unless it is delegated to S-mode). If the hart executes `mret` (return-from-M-mode) instruction in M-mode, control is returned to the location specified by the `mepc` register.

### Physical Memory Protection (PMP)

Physical Memory Protection (PMP) in RISC-V is a hardware feature that provides fine-grained control over access to memory regions. It allows the system to define a set of rules governing which memory regions can be accessed by different privilege levels (such as U-mode or S-mode). RISC-V supports up to 64 PMP entries, with each PMP entry defined by an 8-bit configuration register (e.g., `pmp0cfg`) and a corresponding 64-bit address register (e.g., `pmpaddr0`). 

The PMP address registers are named `pmpaddr0` through `pmpaddr63`. Each PMP address register encodes bits 55-2 of a 56-bit physical address. The PMP configuration registers, ranging from `pmp0cfg` to `pmp63cfg`, are densely packed into 64-bit registers, `pmpcfg0` to `pmpcfg14`, to minimize context-switch time. For example, `pmpcfg0` contains eight configuration registers from `pmp0cfg` to `pmp7cfg`. Each bit in the PMP configuration register specifies whether the corresponding memory region has permission for read (`R`), write (`W`), or instruction execution (`X`), as illustrated in the following figure. The `A` field encodes the address-matching mode of the associated PMP address register. In this project assignment, we will consider only the case of `A` = `0b01`, which indicates that the associated PMP address register defines the _top of the address range (TOR)_, with the preceding PMP address range forming the bottom of the range. 
The `L` bit indicates that the PMP entry is locked, i.e., writes to the configuration register and associated address registers are ignored. You can assume that the `L` bit is set to 0 (unlocked). Note that the PMP address registers and configuration registers are accessible only in M-mode. 

PMP configuration register format:
```
  7   6   5   4   3   2   1   0
+---+---+---+---+---+---+---+---+
| L | 0 | 0 |   A   | X | W | R |
+---+---+---+---+---+---+---+---+
```

In `xv6`, the PMP registers are initialized in the `start()` function at `./kernel/start.c` as follows. This gives the read, write, and execute permissions to the entire area of physical memory. (In the skeleton code, the corresponding initialization is done in `setpmp()` @ `./kernel/pmp.c`.)

```C
void
start()
{
  ...

  // configure Physical Memory Protection to give supervisor mode
  // access to all of physical memory.
  w_pmpaddr0(0x3fffffffffffffull);
  w_pmpcfg0(0xf);

  ...
}
```

## Problem specification

### 1. Implement the `nenter()` system call (30 points)

First, you need to implement the `nenter()` system call. The system call number of `nenter()` is already assigned to 22 in the `./kernel/syscall.h` file. 

__SYNOPSYS__
```
    int nenter();
```

__DESCRIPTION__

The `nenter()` system call returns the total count of `[ENTER]` key presses from the console input device (i.e., keyboard) since the system booted.

__RETURN VALUE__

* `nenter()` returns the cumulative count of `[ENTER]` key presses from the keyboard. This count is initialized to zero at kernel startup and is expected to increase monotonically thereafter.

### 2. Implement the `getpmpaddr()` system call (50 points)

You are required to implement the `getpmpaddr()` system call. The system call number of `getpmpaddr()` is already assigned to 23 in the `./kernel/syscall.h` file.

__SYNOPSYS__
```
    void *getpmpaddr(int n);
```

__DESCRIPTION__

The `getpmpaddr()` system call returns the 64-bit physical address stored in the PMP address register. The value `n` denotes the index of the PMP address register (e.g., 0 for `pmpaddr0`, 1 for `pmpaddr1`, etc.). You can assume that `n` is an integer value between 0 and 3. The contents of the PMP address registers can be read using the following RISC-V assembly instruction.
```
  # Reading pmpaddr0 register
  csrr a0, pmpaddr0
```
The PMP address registers contain only bits 55-2 of the physical address, so after reading the value using the `csrr` instruction, you need to shift the result left by 2 bits to obtain the full address. The remaining bits 63-56 are set to zero. Since PMP address registers are only accessible in machine mode, another (nested) system call from supervisor mode to machine mode is required to retrieve their values. 

__RETURN VALUE__

* `getpmpaddr()` returns a 64-bit physical address, where bits 55-2 are obtained from the `n`-th PMP address register (e.g., `pmpaddr0`, `pmpaddr1`, etc.), and the remaining bits are set to zero.
* If `n` is less than 0 or greater than 3, `getpmpaddr()` returns `(void *) -1`. 

### 3. Implement the `getpmpcfg()` system call (20 points)

Finally, you need to implement the `getpmpcfg()` system call. The system call number of `getpmpcfg()` is already assigned to 24 in the `./kernel/syscall.h` file.

__SYNOPSYS__
```
    int getpmpcfg(int n);
```

__DESCRIPTION__

The `getpmpcfg()` system call returns an integer, where the lower 8 bits represent the content of the specified PMP configuration register. The value `n` denotes the index of the PMP configuration register (e.g., 0 for `pmp0cfg`, 1 for `pmp1cfg`, etc.). You can assume that `n` is an integer value between 0 and 3. Since individual 8-bit PMP configuration registers, such as `pmp0cfg` and `pmp1cfg`, cannot be read directly, you must read the entire 64-bit `pmpcfg0` register,  which contains `pmp0cfg` through `pmp7cfg`, and then extract the corresponding 8-bit entry. The 64-bit PMP configuration registers can be accessed using the following RISC-V assembly instruction.

```
  # Reading pmpcfg0 register
  csrr a0, pmpcfg0
```

Similar to PMP address registers, PMP configuration registers are only accessible in machine mode. Therefore, you must make another (nested) system call from supervisor mode to machine mode to retrieve their values. 

__RETURN VALUE__

* `getpmpcfg()` returns a 32-bit integer, where lower 8 bits (bits 7-0) are obtained from the `n`-th PMP configuration register (e.g., `pmp0cfg`, `pmp1cfg`, etc.), and the remaining bits are set to zero. 
* If `n` is less than 0 or greater than 3, `getpmpcfg()` returns `-1`. 

## Restrictions

* For this project assignment, you should use the `qemu` version 8.2.0 or higher. To determine the `qemu` version, use the command: `$ qemu-system-riscv64 --version`
* You only need to change the files in the `./kernel` directory. Any other changes will be ignored during grading.
* Do not change the `./kernel/pmp.c` file. This file will be overwritten during the automatic grading process.

## Tips

* Read Chap. 4.1 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2024-2/book-riscv-rev4.pdf) to understand RISC-V's privileged modes (supervisor mode and machine mode) and trap handling mechanism.
* Read Chap. 4.2 ~ 4.5 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2024-2/book-riscv-rev4.pdf) to see how traps (system calls and interrupts) are handled in xv6.
* Read Chap. 5 of the [xv6 book](http://csl.snu.ac.kr/courses/4190.307/2024-2/book-riscv-rev4.pdf) to learn about hardware interrupts.
* More detailed information on physical memory protection (PMP) can be found in Chap. 3.7 of the [RISC-V Privileged Architecture manual](HTTP://csl.snu.ac.kr/courses/4190.307/2024-2/priv-isa-asciidoc_20240411.pdf).

* For your reference, the following roughly shows the amount of changes you need to make for this project assignment. Each `+` symbol indicates 1~10 lines of code that should be added, deleted, or altered.
   ```
   kernel/console.c   |  +
   kernel/kernelvec.S |  ++++
   kernel/riscv.h     |  +
   kernel/start.c     |  +
   kernel/syscall.c   |  +
   kernel/sysproc.c   |  ++++++
   kernel/trap.c      |  +
   ```
  
## Skeleton code

The skeleton code for this project assignment (PA2) is available as a branch named `pa2`. Therefore, you should work on the `pa2` branch as follows:

```
$ git clone https://github.com/snu-csl/xv6-riscv-snu
$ git checkout pa2
```

After downloading, you must first set your `STUDENTID` in the `Makefile` again.

The `pa2` branch has a user-level utility program called `nenter` whose source code is available in the `user/nenter.c` file. The `nenter` program simply calls the `nenter()` system call and prints its result. If you successfully implement the `nenter()` system call, the output should look like this:

```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M  -smp 3 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ nenter
nenter: 1
$ nenter
nenter: 2
$ nenter
nenter: 3
$ QEMU: Terminated
```

There is also a user-level program called `pmptest` in the `user` directory. Its source code can be found in the `user/pmptest.c` file. This program displays the values of PMP address registers from `pmpaddr0` to `pmpaddr3`, along with their corresponding permissions, after reading them using the `getpmpaddr()` and `getpmpcfg()` system calls. The following shows what you would get under the `xv6`'s default PMP setting.
```
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M  -smp 3 -nographic -global virtio-mmio.force-legacy=false -drive file=fs.img,if=none,format=raw,id=x0 -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0

xv6 kernel is booting

hart 2 starting
hart 1 starting
init: starting sh
$ pmptest
pmpaddr[0] = 0x00FFFFFFFFFFFFFC (RWX)
pmpaddr[1] = 0x0000000000000000 (---)
pmpaddr[2] = 0x0000000000000000 (---)
pmpaddr[3] = 0x0000000000000000 (---)
$ QEMU: Terminated
```

## Hand in instructions

* First, make sure you are on the `pa2` branch in your `xv6-riscv-snu` directory. And then perform the `make submit` command to generate a compressed tar file named `xv6-{PANUM}-{STUDENTID}.tar.gz` in the `../xv6-riscv-snu` directory. Upload this file to the submission server. You don't need to upload any documents for this project assignment.

* The total number of submissions for this project assignment will be limited to 30. Only the version marked as `FINAL` will be considered for the project score. Please remember to designate the version you wish to submit using the `FINAL` button. 
  
* Note that the submission server is only accessible inside the SNU campus network. If you want off-campus access (from home, cafe, etc.), you can add your IP address by submitting a Google Form whose URL is available in the eTL. Now, adding your new IP address is automated by a script that periodically checks the Google Form at minutes 0, 20, and 40 during the hours between 09:00 and 00:40 the following day, and at minute 0 every hour between 01:00 and 09:00.
     + If you cannot reach the server a minute after the update time, check your IP address, as you might have sent the wrong IP address.
     + If you still cannot access the server after a while, it is likely due to an error in the automated process. The TAs will check if the script is running properly, but since that is a ___manual___ process, please do not expect it to be completed immediately.

## Logistics

* You will work on this project alone.
* Only the upload submitted before the deadline will receive the full credit. 25% of the credit will be deducted for every single day delayed.
* __You can use up to _3 slip days_ during this semester__. If your submission is delayed by one day and you decide to use one slip day, there will be no penalty. In this case, you should explicitly declare the number of slip days you want to use on the QnA board of the submission server before the next project assignment is announced. Once slip days have been used, they cannot be canceled later, so saving them for later projects is highly recommended!
* Any attempt to copy others' work will result in a heavy penalty (for both the copier and the originator). Don't take a risk.

Have fun!

[Jin-Soo Kim](mailto:jinsoo.kim_AT_snu.ac.kr)  
[Systems Software and Architecture Laboratory](http://csl.snu.ac.kr)  
[Dept. of Computer Science and Engineering](http://cse.snu.ac.kr)  
[Seoul National University](http://www.snu.ac.kr)
