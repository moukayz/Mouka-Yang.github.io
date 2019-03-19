---
title: Breakpoints in Windows Debugger
categories:
- Windows Internal

---

<!-- more -->

### Hardware Interrupt Breakpoint (INT 3 )

This is the most common and simple breakpoint to implement in a debugger.

Usually, when the user set a breakpoint on one instruction, the debugger replaced the lowest byte with `0xCC`, which will trigger a software exception interrupt ( also known as **INT 3** interrupt) when the instruction executes, then the system will send  `EXCEPTION_BREAKPOINT` exception to the debugger to handle it (like pausing the program, viewing program states). After that, the debugger will replace the `0xCC` byte with the original byte of that instruction and execute it again.

To make the INT3 breakpoint set on an instruction keep working (0xCC is removed after the breakpoint hit to let the instruction execute), when a breakpoint breaking on, the debugger should enable the **Trace Flag** in **EFLAGS** register so the CPU will enter SINGLE-STEP mode. When the original instruction bytes are resumed and the instruction is executed, the CPU will trigger a interrupt and Windows will send a `EXCEPTION_SINGLE_STEP` to the debugger, at that time the debugger can **AGAIN** replace the original bytes of that instruction with 0xCC to set a breakpoint again.

**note**: the number of INT3 breakpoints is unlimited, but they **only work on code.**

### Hardware Debug Register (Hardware Breakpoint)

This breakpoint takes the advantages of the debug registers supplied by the instruction set. In x86 they are mainly **DR0, DR1, DR2, DR3, DR6 and DR7** ( see [detailed description about x86 debug register](https://en.wikipedia.org/wiki/X86_debug_register) ). 

The first four register (**Debug Address Register** ) are used to hold the address which the hardware breakpoint will break on (which means there are four hardware breakpoints at most in x86). 

The DR7 (**Debug Control Register**)  is used to control the enablement of these registers, and set breakpoint conditions. The flags and fields in this register control the following things:

- **L0-L3 (Local breakpoint enable) flags (bits 0, 2, 4, 6)**
  - Enable (when set) the breakpoint condition for the associated breakpoint for **current task**
  - When a breakpoint  condition is detected and its associated **Ln**(0-3) flag is set, a debug exception is generated
  - The processor automatically **clear these flags on every task switch** to avoid unwanted breakpoint conditions in new task
- **G0-G3 (Global breakpoint enable) flags (bits 1, 3, 5, 7)**
  - Enable (when set) the breakpoint condition for the associated breakpoint for **all tasks**
  - When a breakpoint  condition is detected and its associated **Gn**(0-3) flag is set, a debug exception is generated
  - The processor **does not clear these flags on a task switch** , allowing a breakpoint to be enabled on all tasks
- **LE and GE (Local and Global exact breakpoint enable) flags (bits 8, 9)**
- **GD (General detect enable) flag (bit 13)**
  - Enable debug register protection
  - When set, a debug exception will be generated before **ANY** MOV instruction that accesses a debug register
- **R/W0-3 (Read Write ) fields (bits 16-17, 20-21, 24-25, 28-29)**
  - Specify the breakpoint condition for the corresponding breakpoint 
    - 00 - break on instruction execution only
    - 01 - break on data writes only
    - 10 - break on IO writes or reads
    - 11 - break on data reads or writes but not instruction fetches
- **LEN0-3 (Length) fields (bits 18-19, 22-23, 26-27, 30-31)**
  - Specify the size of the memory location at the address specified in the corresponding debug address register (DR0-DR3)
    - 00 - 1-byte length
    - 01 - 2-byte length
    - 10 - undefined ( or 8-byte length on particular CPU family and x64)
    - 11 - 4-byte length
  - Instruction breakpoint address must have a length specification of 1 byte (00). Code breakpoints for other operand sizes are undefined.

**On x64 architecture, DR0-DR7 are 64 bits. The upper 32 bit of DR6 and DR7 must be set to zeros**

### Software breakpoint (Memory breakpoint)

This breakpoint is implemented at the **PAGE** level instead of the address level like other breakpoints. When a software breakpoint is set, the memory address of the breakpoint will have its page permissions changed to that of a [guard page](http://msdn.microsoft.com/en-us/library/windows/desktop/aa366549%28v=vs.85%29.aspx) using `VirtualProtectEx` API. When any memory location in that page is accessed, Windows will send a `EXCEPTION_GUARDED_PAGE` to the debugger.

Because the guard page protection will be removed from the page after the exception raised, once the exception is handled and the execution continues, any access to that page will not trigger such a exception. In order to make the software breakpoint keep working, just like INT3 breakpoint, the debugger can change the CPU to single-step mode and resume the guard page protection again after the instruction that accesses the breakpoint address is executed.