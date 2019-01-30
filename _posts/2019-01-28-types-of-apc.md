---
title: Types of APC
categories:
- Windows Internal
feature_image: "https://picsum.photos/2560/600?image=872"
---
<!-- more -->

An asynchronous procedure call (APC) is a function that executes asynchronously. APCs are similar to deferred procedure calls (DPCs), but unlike DPCs, APCs execute within the context of a particular thread. Drivers (other than file systems and file-system filter drivers) do not use APCs directly, but other parts of the operating system do, so you need to be aware of how APCs work.

The Windows operating system uses three kinds of APCs:

- ***User APCs*** run strictly in user mode and only when the current thread is in an alertable wait state. The operating system uses user APCs to implement mechanisms such as overlapped I/O and the `QueueUserApc` Win32 routine.
- ***Normal kernel APCs*** run in kernel mode at IRQL = PASSIVE_LEVEL. A normal kernel APC preempts all user-mode code, including user APCs. Normal kernel APCs are generally used by file systems and file-system filter drivers.
- ***Special kernel APCs*** run in kernel mode at IRQL = APC_LEVEL. A special kernel APC preempts user-mode code and kernel-mode code that executes at IRQL = PASSIVE_LEVEL, including both user APCs and normal kernel APCs. The operating system uses special kernel APCs to handle operations such as I/O request completion.

## Disable APCs

The system provides three mechanisms to disable APCs for the current thread:

- **Critical regions.** When a thread is inside a critical region, its user APCs and normal kernel APCs are not executed. Special kernel APCs are still executed. For more information about these APC types, see [Types of APCs](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/types-of-apcs).
- **Guarded regions.** When a thread is inside a guarded region, none of its APCs are executed.
- **Raising the current IRQL to APC_LEVEL or higher.** A thread that is executing at IRQL >= APC_LEVEL executes with all APCs disabled.