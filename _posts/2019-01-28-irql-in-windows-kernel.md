---
title: IRQL in Windows kernel
categories:
- Windows Internal
feature_image: "https://picsum.photos/2560/600?image=872"
---
<!-- more -->

Here’s a table of all IRQLs, as defined in the Windows NT header files (easily seen in the WDK.)  

| **IRQL**                        | **X86 IRQL Value** | **AMD64** **IRQL Value** | **IA64 IRQL Value** | **Description**                                              |
| ------------------------------- | ------------------ | ------------------------ | ------------------- | ------------------------------------------------------------ |
| PASSIVE_LEVEL                   | 0                  | 0                        | 0                   | User threads and most kernel-mode operations                 |
| APC_LEVEL                       | 1                  | 1                        | 1                   | Asynchronous procedure calls and page faults                 |
| DISPATCH_LEVEL                  | 2                  | 2                        | 2                   | Thread scheduler and deferred procedure calls (DPCs)         |
| CMC_LEVEL                       | N/A                | N/A                      | 3                   | Correctable machine-check level (IA64 platforms only)        |
| Device interrupt levels (DIRQL) | 3-26               | 3-11                     | 4-11                | Device interrupts                                            |
| PC_LEVEL                        | N/A                | N/A                      | 12                  | Performance counter (IA64 platforms only)                    |
| PROFILE_LEVEL                   | 27                 | 15                       | 15                  | Profiling timer for releases earlier than Windows 2000       |
| SYNCH_LEVEL                     | 27                 | 13                       | 13                  | Synchronization of code and instruction streams across processors |
| CLOCK_LEVEL                     | N/A                | 13                       | 13                  | Clock timer                                                  |
| CLOCK2_LEVEL                    | 28                 | N/A                      | N/A                 | Clock timer for x86 hardware                                 |
| IPI_LEVEL                       | 29                 | 14                       | 14                  | Interprocessor interrupt for enforcing cache consistency     |
| POWER_LEVEL                     | 30                 | 14                       | 15                  | Power failure                                                |
| HIGH_LEVEL                      | 31                 | 15                       | 15                  | Machine checks and catastrophic errors; profiling timer for Windows XP and later releases |

For driver writers, the only IRQLs that are usually interesting are 0 through 2 and DIRQL.  It’s worth mentioning, though, **that the NT kernel itself internally has spinlocks at DISPATCH_LEVEL and all the levels above that.** [Reference link](https://blogs.msdn.microsoft.com/doronh/2010/02/02/what-is-irql/)

## PASSIVE_LEVEL

**Interrupts Masked Off** — None.

**Driver Routines Called at** PASSIVE_LEVEL — [**DriverEntry**](https://msdn.microsoft.com/library/windows/hardware/ff544113), [*AddDevice*](https://msdn.microsoft.com/library/windows/hardware/ff540521), [*Reinitialize*](https://msdn.microsoft.com/library/windows/hardware/ff561022), [*Unload*](https://msdn.microsoft.com/library/windows/hardware/ff564886) routines, most dispatch routines, driver-created threads, worker-thread callbacks.

Any interrupt can occur at PASSIVE_LEVEL.  **User-mode code** executes at PASSIVE_LEVEL. 

## APC_LEVEL

**Interrupts Masked Off** — APC_LEVEL interrupts are masked off.

**Driver Routines Called at** APC_LEVEL — Some dispatch routines (see [Dispatch Routines and IRQLs](https://docs.microsoft.com/en-us/windows-hardware/drivers/kernel/dispatch-routines-and-irqls)).

APC interrupts, by the way, are sent by a processor, either to itself or to another processor.  No external device is involved. 

## DISPATCH_LEVEL

**Interrupts Masked Off** — DISPATCH_LEVEL and APC_LEVEL interrupts are masked off. Device, clock, and power failure interrupts can occur.

**Driver Routines Called at** DISPATCH_LEVEL — [*StartIo*](https://msdn.microsoft.com/library/windows/hardware/ff563858), [*AdapterControl*](https://msdn.microsoft.com/library/windows/hardware/ff540504), [*AdapterListControl*](https://msdn.microsoft.com/library/windows/hardware/ff540513), [*ControllerControl*](https://msdn.microsoft.com/library/windows/hardware/ff542049), [*IoTimer*](https://msdn.microsoft.com/library/windows/hardware/ff550381), [*Cancel*](https://msdn.microsoft.com/library/windows/hardware/ff540742) (while holding the cancel spin lock), [*DpcForIsr*](https://msdn.microsoft.com/library/windows/hardware/ff544079), [*CustomTimerDpc*](https://msdn.microsoft.com/library/windows/hardware/ff542983), [*CustomDpc*](https://msdn.microsoft.com/library/windows/hardware/ff542972) routines.

There is no process that decides which other processes should run.  Each processor “dispatches” itself by looking at runnable threads and deciding which one to run next.  

If your code raises IRQL to DISPATCH_LEVEL, you have **disabled the dispatcher on that processor**, and only on that processor.  This means that your thread will **not be preempted by another thread** and it will not be moved to another processor until you lower IRQL. 

**Page fault**

Since, as noted above, I/O completion depends on code running at APC_LEVEL, and since APC_LEVEL code won’t run while the processor is at DISPATCH_LEVEL, **page faults can’t be resolved at DISPATCH_LEVEL**.  So code that holds a DISPATCH_LEVEL lock (like a spinlock) **can’t reference memory which might be paged out.** 

**Dispatcher object**

you can’t wait on a dispatcher object at DISPATCH_LEVEL.  You’ve already **disabled the dispatcher**.  Your only choice at DISPATCH_LEVEL is a **spinlock**. 

## DIRQL

**Interrupts Masked Off** — All interrupts at IRQL<= DIRQL of driver's interrupt object. Device interrupts with a higher DIRQL value can occur, along with clock and power failure interrupts.

Driver Routines Called at DIRQL — [*InterruptService*](https://msdn.microsoft.com/library/windows/hardware/ff547958), [*SynchCritSection*](https://msdn.microsoft.com/library/windows/hardware/ff563928) routines.

**(The only difference between APC_LEVEL and PASSIVE_LEVEL is that a process executing at APC_LEVEL cannot get APC interrupts. But both IRQLs imply a thread context and both imply that the code can be paged out.)**

## ATTENTIONS

When calling driver support routines, **be aware of** the following.

- Calling [**KeRaiseIrql**](https://msdn.microsoft.com/library/windows/hardware/ff553079) with an input ***NewIrql*** value that is less than the current IRQL causes a fatal error. Calling [**KeLowerIrql**](https://msdn.microsoft.com/library/windows/hardware/ff552968) except to restore the original IRQL (that is, after a call to **KeRaiseIrql**) also causes a fatal error.

- While running at IRQL >= DISPATCH_LEVEL, calling [**KeWaitForSingleObject**](https://msdn.microsoft.com/library/windows/hardware/ff553350) or [**KeWaitForMultipleObjects**](https://msdn.microsoft.com/library/windows/hardware/ff553324) for kernel-defined dispatcher objects to wait for a nonzero interval causes a fatal error.

- The only driver routines that can safely wait for events, semaphores, mutexes, or timers to be set to the signaled state are those that run in a nonarbitrary thread context at IRQL PASSIVE_LEVEL, such as **driver-created threads**, the **DriverEntry** and ***Reinitialize*** routines, or **dispatch routines** for inherently synchronous I/O operations (such as most device I/O control requests).

- Even while running at IRQL PASSIVE_LEVEL, **pageable driver code** must not call [**KeSetEvent**](https://msdn.microsoft.com/library/windows/hardware/ff553253), [**KeReleaseSemaphore**](https://msdn.microsoft.com/library/windows/hardware/ff553143), or [**KeReleaseMutex**](https://msdn.microsoft.com/library/windows/hardware/ff553140) with the input *Wait* parameter set to **TRUE**. Such a call can cause a fatal page fault.

- Any routine that is running at greater than IRQL APC_LEVEL can neither allocate memory from paged pool nor access memory in paged pool safely. If a routine running at IRQL greater than APC_LEVEL causes a page fault, it is a fatal error.

- A driver must be running at IRQL DISPATCH_LEVEL when it calls [**KeAcquireSpinLockAtDpcLevel**](https://msdn.microsoft.com/library/windows/hardware/ff551921) and [**KeReleaseSpinLockFromDpcLevel**](https://msdn.microsoft.com/library/windows/hardware/ff553150).

  A driver can be running at IRQL <= DISPATCH_LEVEL when it calls **KeAcquireSpinLock** but it must release that spin lock by calling **KeReleaseSpinLock**. In other words, it is a programming error to release a spin lock acquired with **KeAcquireSpinLock** by calling **KeReleaseSpinLockFromDpcLevel**.

  A driver must not call **KeAcquireSpinLockAtDpcLevel**, **KeReleaseSpinLockFromDpcLevel**, **KeAcquireSpinLock**, or **KeReleaseSpinLock** while running at IRQL > DISPATCH_LEVEL.

- Calling a support routine that uses a spin lock, such as an **ExInterlockedXxx** routine, raises IRQL on the current processor either to DISPATCH_LEVEL or to DIRQL if the caller is not already running at a raised IRQL.

- Driver code that runs at IRQL > PASSIVE_LEVEL should execute as quickly as possible. The higher the IRQL at which a routine runs, the more important it is for good overall performance to tune that routine to execute as quickly as possible. For example, any driver that calls **KeRaiseIrql** should make the reciprocal call to **KeLowerIrql** as soon as it can.