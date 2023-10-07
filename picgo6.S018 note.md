# 6.S018 note

## Lec 3 OS organization and system calls

- four topic: isolation, kernel/user mode, system call, xv6
- isolation

  - to prevent a crash from one application to whole operating system
  - if no operating system, application will interact with hardware
    - when CPU need to change to another application, current application may not give up CPU which may contain a infinite loop or some thing, and nothing can prevent it. 
    - one application may access memory of another application. there is no boundary.
  - Unix interface
    - abstract the hardware resources
    - example: Process
      - not a CPU, but like a CPU, can do computation.
      - application can use CPU as a process
      - switch between process, which mean real multiplex
    - exec: abstract of memory
    - file: abstract disk blocks
      - every block in one file
- kernel/user mode
    - OS should be defensive
    
      - application cannot crash OS
      - application cannot break isolation
      - and strong isolation between application and OS, which mean a wall
      - typical hardware support
        1. user/kernel mode
        2. virtual memory
    - kernel mode: privileged instructions
        - set up page table
        - interrupt
    - user mode: unprivileged instructions
    - 一个问题，如果一个application处于user mode， 他想要切换到kernel mode，在risc5里面使用ecall命令，需要修改cpu的状态位，这也需要优先权限。那么怎么在user mode里面执行一个privilege instruction呢
    - CPU provide virtual memory
    - page table map virtual address to physical memory which provide strong memory isolation
- system call
     - entry the kernel(only place): `ecall <n>` (n is a system number known by operating)
     - ex: `fork()  ->  ecall 2  -> syscall  -> fork()`
     - the function of system function will check whether this application can call this function 
     - kernel = trusted computing base
          - kernel must have no bugs
          - kernel must treat process as malicious
       - monolithic kernel design
         - disadvantage: small bugs may crash whole kernel
         - advantage: different part can cooperate easily with good performance
       - micro kernel design
           - disadvantage: poor performance due to much message
           - advantage: security
- xv6 code
  - treat **qemu** like real board, and it's just a infinite loop which executes instructions

## Lec 4 Page table

- plan

  - address spaces
  - paging hw(riscv)
  - code xv6
- address space(each process have its own whole space)

  - page table

    - `CPU -> va -> mmu -> pa -> mem`
    - page table also in memory and there is a register satp in CPU(the kernel store satp for each process)
- paging hw

    - multilevel <img src="c51c24e5a827fa625b12d3ebf5d7f32.jpg" alt="multilevel"  />
    - only when we need, then we allocate another page. this means we can shrink the number of PTE
    - flags
      - valid: this is a valid page
      - write: can write to it
      - execute: can execute the instructions in it
      - user: can access by user space
    - Translation look-aside buffer(TLB)
      - TLB is just a cache store the [va, pa]
      - and when change page table, we need to flush TLB
    - MMU is in processor, some cache in it, some is out of that
    - kernel virtual address
      - <img src="image-20220902161658188.png" alt="image-20220902161658188" style="zoom:200%;" />

## Lec 5 Riscv calling convention

1. c -> ASM/Processor
   - instruction set
2. Risc-v & x86
   1. reduced instruction set
   2. complex instruction set
3. Registers
4. stack + calling convention
5. struct layout in memory

## Lec 6 Traps

- supervisor:
  - R/W control regs: SATP, STVEC, SEPC, SSC
  - use PTEs w/r PTE_U  
- ecall
  1. change mode from user to supervisor
  2. store pc in sepc
  3. jump to stvec
- 如果在执行的过程中kernel的stack改变了呢 ok已经解决
- 对于进入kernel来说，究竟有什么变化呢，kernel在这个过程中扮演一个什么样的角色呢
- 

## Lec 7 Page Fault

- plan
  - complement vm
  - feature using page fault(lazy allocation and )
- information needed
  - the faulting va
  - the type of page fault
    - scause(R, W, I)
  - the va of instruction cause the fault
- allocation
  - sbrk() -> eager allocation
  - lazy allocation, sbrk() just increase p->sz n. when application use that memory, it cause page fault and allocate 1 page, zero the page, map the page, restart access
  - zero fill one demand
- Copy on write(COW)
  - fork and exec
  - child point to the same physical memory, and set to read only for all page. when child try to access, allocate another page
  - ref record the amount reference to a physical page
- Demand paging

## Lec 8 interrupt

- interrupt
  - HW want attention now from kernel
  - SW
    - save its work
    - process interrupt
    - resume its work
  - attention
    - asynchronous
    - concurrency
    - program devices
  
- where do interrupt come from
  - different devices below 0x80000000, DRAM on 0x80000000
  - PLIC(platform-level interrupt control)
    - 53 interrupt lines from devices
    - control external interrupt to four or more processores
    - when core is in interrupt or situation can't receive interrupt, PLIC can hold that interrupt until a processor can receive that.(this mean PLIC has registers to record that)
  
- driver manages device(high level of HW)
  - uart.c for example
  - it is a part of code where bottom is interrupt handler and the top is functions interrupt need
  - 为什么下面和上面需要解耦？
  - driver might be larger than kernel
  
- programming devices
  - memory mapped i/o
  - ld/st read/write register device
  
- case study: \$ls
  - \$: device puts '$' into uart, uart generate interrupt when the char been sent
  - keyboard connect to receive line and generate interrupt
  
- RISC-V support for interrupt
  - SIE: one bit for interrupt of E(external), S(software), T(timer)
  - SSTATUS: bit enable/disable interrupt
  - SIP: interrupt pending(indicate the kink of interrupt type)
  - SCAUSE:
  - STVEC: save program place
  
- 为什么sheduler调用intr_on可以防止死锁, 是因为可以被中断意味着在死循环的时候进行中断程序进行处理吗

- 为什么devsw会调用console的write函数

- Interrupt
  1. clear SIE bit
  2. sepc <- pc
  3. save currrent mode
  4. mode <= super
  5. pc <- stvec
  
- 对于write 来说，他调用sys call write，CPU进入中断，然后将字符逐个写入到UART的缓冲区，并将字符写入到传输的reg，然后呢？？？怎么样将这个reg上的东西传递到console呢。 有可能是因为这个时候设置了相关的位。然后在发出一个中断？？？？？？？

- 通俗说法，首先device发出一个中断请求，PLIC收到后寻找到一个CPU， CPU进行中断处理，然后向PLIC发出信号说我可以接受中断了，然后CPU开始uartintr的处理函数，开始从uart的reg中读取字符，在上述过程的同时，设备在不断进行字符的读入并放到缓存区中，等待uart（不知所云

- 在从键盘输入的时候，首先shell调用read 的syscall，调用consoleread进入睡眠状态，等待consoleintr的唤醒。当键盘进行输入的时候，进行中断，cpu在trap调用uartintr，进一步调用consoleintr，将输入的字符放入console的buffer中，当输入的字符为回车时，唤醒。问题：1与上一个相同，键盘输入和系统调用输入到console有什么不同？2.如何将实时的buf里面的东西显示给屏幕，这个过程？3.在这种并发过程中，如果只有一个cpu如何完成上述操作，是因为sleep导致可以进行中断吗？如果是这样的话，那么wakeup的时候应该怎么处理呢

- Interrupt and concurency

  1. device and CPU run in parallel(**producer/consumer parallelism**)

  2. interrupt stops the current program

     but in kernel, it might be bad to interrupt in some place

  3. top of driver + bottom driver may run in parallel using locks

     it can happens that different CPU interrupt at the same time 

- producer/consumer

  - a FIFO queue

## Lec 9 Locks

app wants to multiple cores

kermel must handle parallel system call

access shared data

- why locks -- avoid

- lock abstraction

  - a struct
  - acquire(&lock) one process can acquire lock
  - release(&lock) only current lock release then other can acquire
  - code between acquire and release is atomic(要么全都执行，要么都不执行)
  - lock serialize the operation

- when to lock

  - guide rule: 2 processses access a shared data structure(one is writer) -> lock data structure
  - lock free programming

- lcok perspective

  1. locks help avoid lost update
  2. locks make multi step operation atomic
  3. locks help mantain invariant

- deadlock

  - acquire acquire again

- locks vs. modularity

  - lock ordering -> global
  - m1 -> m2  m1 need to know m2 might have what lock
  - contrast with abstraction

- lock vs. performance

  - need to split update structure
  - best split is a challenge, and may need to rewrite code

- case

  - uart
    - uart_tx_lock protect data structure of uart buffer
    - tail end is  
    - 如果上半部分和下半部分同时使用锁，那为什么还可以并行呢

- wrong case

  ```c
  broken acquire(struct lock *l){
      while(l){
          if(l->locked == 0){//but is A and B run this at the same time, both will get lock
              l->locked = 1;
              return;
          }
      }
  }
  ```

  solusion: hardware test and set suport
  
  ​	 `amoswap addr, r1, r2`  depend implementation of memory system

​			if use `store`, it might not a atomic operation.

## Lec 10 threads

- thread -- one seriac execution
  - contain: pc, regs, stack,
  - interleave
    - multi-core
    - switch between threads
  - shared memory
    -   ***xv6 has kernel thread for each process and all of them share kernel memory!!!*** 
    -  xv6 user process don't share process
- thread challenge
  - switch(scheduler)
  - what need to be saved
  - compute-bound(stop long time thread)
- timer interrupts
  - kernel handler, yields
  - preemptive schedule
  - state
    - running
    - runnable
    - sleeping
- step of thread switch 
  - from userspace into corresponding kernel thread
  - schedule to another context
  - back to user space which is another program

## Lec 11 coordination(sleep and wake up)

- no other locks when swtch

  - when P1 acquire the lock and call swtch
  - P2 may want this lock to0, and that cause P2 spin all the way

- lost wakeups

  ```c
  broken_sleep(chan){
      p->state = SLEEPING;
      p->chan = chan;
      swtch();
  }
  
  wakeup(cahn){
      for each p in procs[]
          if(p->state == SLEEPING && p->chan == chan)
              p->state = RUNNING;
  }
  ```

  - if we put the `release` before `sleep`, it would be possible that `wakeup` happen before sleep, then it would be a dead lock 

- why almost every sleep need a loop tp check?

  - because there might multi processes being waked up, and only one can get out of loop and the rest have to sleep again.

## Lec 12 file system

- some point for file system

  - user-friendly name pathname

  - share files between user/process

  - persistence/durability

- file system is interesing

  - abstraction
  - **crash safely**
  - disk layout(organization of disk)
  - performance(buffer cache and concurrence)

- API example/ file system calss

  `fd = open("x/y", --);`(pathname which is human-readable)

  `write(fd, "abc", 3);`(offset -> 3)

  `link("x/y", "x/z")`(multiple names)

  `unlink("x/y");`

- file system structure

  - inode(fileinfo, independent of name)
  - only when these being set to zero that cat be delete
    - link count
    - open fd count
    - fd should contain a offset

- FS layers

  - names/fds
  - inode -> read and write
  - icache
  - logging
  - buf cache
  - disk

- storage devices

  - SSD and HDD
  - RAM <-> CPU <-> Disk(read and write between disk and CPU)

- Disk layout

  - block 0 contain boot or nothing
  - block 1 is super block which record information about this disk(how many blocks for log, inodes, bitmap, data)
  - meta data
    - block 2 - 32 log
    - block 32 - 45 inodes
    - block 46 bitmap
  - rest is data

- on-disk inode(64 bytes)

  - type

  - nlink

  - size

  - 12 direct block number(each block number is 4 bytes)(total 48 bytes)

  - 1 indirect block number pointing to a block which contain more block number(4 bytes)

    max file size: (12 + 256) * 1024 bytes

    to read 8000 byte of a inode(file), 1024 divide 8000, which get the 6th block number in inode, and get that block

- directory

  - dir == file with some structure
  - entry (inum(2 bytes), filename(14 bytes))
  - pathname lookup
    - find /y/x
    - root inode(1) read the inode
    - scan its block for entry and compare name to get inode

## Lec 13 Crash safety

- problem: crash can lead the on-disk fs to be incorrect
  - solution: log
- risk
  - fs operation are multi step disk operation
  - crash may leave fs invariant violated
  - crash again
- logging
  - atomic fscalls
  - fast recovery
  - high performance
- steps with log(block write in disk is atomic)
  - log write
  - commit op
  - install
  - clean log
- challenge problems
  - evicto

# Lec 14 logging system linux ext3

- xv6 log review
  - file system: inode
  - disk system
  - rules
    - write-ahead
    - freeing rule
