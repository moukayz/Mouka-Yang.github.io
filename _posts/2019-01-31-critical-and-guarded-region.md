---
title: Critical and Guarded Region
categories:
- Windows Internal
---

# Critical Region

This region is relative to two kernel functions `KeEnterCriticalRegion` and `KeLeaveCriticalRegion` which must be used in pairs.

Reverse the `KeEnterCriticalRegion`:

```assembly
; void KeEnterCriticalRegion(void)
KeEnterCriticalRegion proc near
    mov     rax, gs:188h
    dec     word ptr [rax+1C4h] ; ETHREAD->KernelApcDisable (UINT2B)
    retn
KeEnterCriticalRegion endp
```

Obviously the function just decreases the `KernelApcDisable` field in current thread's `ETHREAD` structure, which convert 0 to FF if it's the first call to `KeEnterCriticalRegion`.

This will disable all user-mode APCs and kernel-mode regular APCs **except for kernel-mode special APCs** (which `ETHREAD->KernelApcDisable` controls)

Reverse the `KeLeaveCriticalRegion`:

```assembly
KeLeaveCriticalRegion proc near         ; CODE XREF: AdtpBuildAccessesString+3B4p
    mov     rcx, gs:188h
    add     word ptr [rcx+1C4h], 1 		; ETHREAD->KernelApcDisable
    jnz     short locret_14008CDDC
    lea     rax, [rcx+50h]  			; ETHREAD->ApcState
    cmp     [rax], rax      			; If Apc list is not empty
    jnz     short loc_14008CDDD
    locret_14008CDDC:                      
        retn
    loc_14008CDDD:                          
        cmp     word ptr [rcx+1C6h], 0 		; ETHREAD->SpecialApcDisable
        jnz     short locret_14008CDDC
        jmp     KiCheckForKernelApcDelivery
KeLeaveCriticalRegion endp
```

The function first add 1 to `ETHREAD-KernelApcDisable`, if there is no nested call of `KeEnterCriticalRegion` (`KernelApcDisable` equals 0), the function just returns.

If not the case, then the function checks if current thread's APC list is empty and just returns if so.

If APC list is not empty, the function checks if special APC is disabled (`ETHREAD->SpecialApcDisable`), if not, the function delivers these APCs by calling `KiCheckForKernelApcDelivery`

# Guarded Region

There are two functions `KeEnterGuardedRegion` and `KeLeaveGuardedRegion` which should also be used in pairs.

Reverse `KeEnterGuardedRegion`

```assembly
KeEnterGuardedRegion proc near
    mov     rax, gs:188h
    dec     word ptr [rax+1C6h]		;ETHREAD->SpecialApcDisable
    retn
KeEnterGuardedRegion endp
```

Like critical region, the function just decreases  the `SpecialApcDisable` field in current thread's `ETHREAD` structure, which will **disable all APCs** (including special ones)

Reverse `KeLeaveGuardedRegion`:

```assembly
KeLeaveGuardedRegion proc near          
    mov     rax, gs:188h
    add     word ptr [rax+1C6h], 1	;ETHREAD->SpecialApcDisable
    jnz     short locret_1400A21CC
    add     rax, 50h
    cmp     [rax], rax
    jnz     short loc_1400A21CD
    locret_1400A21CC:                       
    	retn
    loc_1400A21CD:                          
    	jmp     KiCheckForKernelApcDelivery
KeLeaveGuardedRegion endp
```

The function first add 1 to `ETHREAD-SpecialApcDisable`, **if there is nested call** (`SpecialApcDisable` != 0), the function just returns.

If not so and the APC list is not empty, the function delivers the APCs  by calling `KiCheckForKernelApcDelivery`

# Summary

A thread that is inside a ***critical region*** executes with user APCs and normal kernel APCs disabled.

A thread inside a ***guarded region*** runs with all APCs disabled.

