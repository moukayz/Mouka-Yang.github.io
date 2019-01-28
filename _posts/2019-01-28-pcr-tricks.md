---
title: Tricks of PCR
categories:
- Reverse Engineering
---
<!-- more -->

PCR (Processor Control Region) is a Windows kernel mode data structure that contains information about the current processor. It can be accessed via the **FS** segment register on x86 versions, or the **GS** segment register on x64 versions respectively.

The PCR contains a substructure called Processor Control Block (KPRCB), which contains information such as CPU step and **a pointer to the thread object of the current thread**.

We can get much information about current thread and process through the PCR directly instead of calling documented APIs. See [Wikipedia link](https://en.wikipedia.org/wiki/Win32_Thread_Information_Block) .

**(Following code are all based on x64 platform, the code in x86 are similar except the offset of corresponding fields) **

## Get IDT

The base address of IDT (Interrupt Dispatch Table) is stored in `PCR->IdtBase`, as below

```
1: kd> dt nt!_kpcr idtbase
   +0x038 IdtBase : Ptr64 _KIDTENTRY64
```

We have

```c
PVOID GetIdtBase(){
    /* ASM
	mov rax,qword ptr gs:[38h]
	*/
    return __readgsqword(0x38);	// KPCR->IdtBase
}
```

## Get Current Thread

We can get current thread object (type `_ETHREAD`) through the `PRCB->CurrentThread`, the relative fields of PCR and PRCB are as below:

```
1: kd> dt nt!_kpcr prcb.currentthread
   +0x180 Prcb               : 
      +0x008 CurrentThread      : Ptr64 _KTHREAD
```

So we have

```c
PETHREAD GetCurrentThread(){
    /* ASM
	mov rax,qword ptr gs:[188h]
	*/
     return __readgsqword(0x188);	// KPCR->KPRCB->CurrentThread
}

// 188 = Offset(PRCB) + Offset(CurrentThread) = 180h + 8h
```

## Get Current TEB (Thread Environment Block)

The **teb** object of current thread is stored in `PCR->_NT_TIB->Self`, as below

```
1: kd> dt nt!_kpcr NtTib.Self
   +0x000 NtTib      : 
      +0x030 Self       : Ptr64 _NT_TIB
```

We have

```c
PVOID GetCurrentTeb(){
#if defined(_M_IX86)
    return __readfsdword(0x18);
#elif defined (_M_AMD64)
    /* ASM
	mov rax,qword ptr gs:[30h]
	*/
    return __readgsqword(0x30);	// PCR->NT_TIB->Self
}

// 30 = Offset(NT_TIB) + Offset(Self) = 0 + 30h
```

## Get Current PEB (Process Environment Block)

We can retrieve the corresponding **PEB** through `TEB->ProcessEnvironmentBlock` as below

```
1: kd> dt nt!_teb processenvironmentblock
   +0x060 ProcessEnvironmentBlock : Ptr64 _PEB
```

We have

```c
PVOID GetCurrentPeb(){
    /* ASM
	mov rax,qword ptr gs:[30h]
	mov rax,qword ptr [rax+60h]
	*/
    return ((PTEB)GetCurrentTeb())->ProcessEnvironmentBlock;
}
```

## Get Current Process

We can retrieve **Process** object through `_ETHREAD->Tcb->ApcState->Process`

```
1: kd> dt nt!_ethread tcb.apcstate.process
   +0x000 Tcb                  : 
      +0x050 ApcState             : 
         +0x020 Process              : Ptr64 _KPROCESS
```

We have

```c
PEPROCESS GetCurrentProcess(){
    /* ASM
	mov rax,qword ptr gs:[188h]
	mov rax,qword ptr [rax+70h]
	*/
    return GetCurrentThread()->Tcb->ApcState->Process;
}
```

## Get Current Process/Thread Id

We can retrieve corresponding **Pid** and **Tid** through `_ETHREAD->Cid` as below

```
1: kd> dt nt!_ethread cid.uniqueprocess
   +0x3b8 Cid               : 
      +0x000 UniqueProcess     : Ptr64 Void

1: kd> dt nt!_ethread cid.uniquethread
   +0x3b8 Cid              : 
      +0x008 UniqueThread     : Ptr64 Void
```

We have

```c
ULONG_PTR GetCurrentProcessId(){
    /* ASM
	mov rax,qword ptr gs:[188h]
	mov rax,qword ptr [rax+3B8h]
	*/
    return GetCurrentThread()->Cid->UniqueProcess;
}

ULONG_PTR GetCurrentThreadId(){
    /* ASM
	mov rax,qword ptr gs:[188h]
	mov rax,qword ptr [rax+3C0h]
	*/
    return GetCurrentThread()->Cid->UniqueThread;
}
```

