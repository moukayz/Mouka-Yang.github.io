---
title: Assembly code snippets of Windows kernel
categories:
- Reverse Engineering
feature_image: "https://picsum.photos/2560/600?image=872"
---
<!-- more -->

## List\_Entry

List manipulation functions in Windows kernel are all inline function like

```c
FORCEINLINE
VOID
InitializeListHead( _Out_ PLIST_ENTRY ListHead ){
    ListHead->Flink = ListHead->Blink = ListHead;
    return;
}
```

So we need to recognize their assembly code snippets in other kernel functions

### InitializeListHead

C pseudocode

```c
ListHead->Flink = ListHead->Blink = ListHead;
```

Asm

```
; r/eax is the pointer to ListHead
; x86
mov [eax], eax
mov [eax + 4], eax

; x64
mov [rax], rax
mov [rax + 8], rax
```

### InsertHeadList

C pseudocode

```c
LIST_ENTRY Flink = ListHead->Flink;
ListHead->Flink = Entry;
Entry->Flink = Flink;
Entry->Blink = Blink;
Flink->Blink = Entry;
```

Asm

```auto
; eax is ListHead, ecx is Entry
; x86
mov edx, [eax]
mov [eax], ecx
mov [ecx], edx
mov [ecx+4], eax
mov [edx+4], ecx

; x64 is identical
```

Snippets in `nt!KiInsertQueueApc`

```auto
mov edi, dword ptr [eax]     ; Flink = ListHead->Flink
lea ecx, [edx+0Ch]           ; Entry
mov dword ptr [ecx], edi     ; Entry->Flink = Flink 
mov dword ptr [ecx+4], eax   ; Entry->Blink = ListHead
mov dword ptr [edi+4], ecx   ; Flink->Blink = Entry
mov dword ptr [eax], ecx     ; ListHead->Flink = Entry
```

### InsertTailList

C pseudocode

```c
LIST_ENTRY Blink = ListHead->Blink;
Entry->Flink = ListHead;
Entry->Blink = Blink;
ListHead->Blink = Entry;
Blink->Flink = Entry;
```

Asm

```auto
mov edx, [eax+4]; Blink = ListHead->B
mov [ecx], eax  ; Entry->Flink = ListHead
mov [ecx+4], edx; Entry->Blink = Blink
mov [eax+4], ecx; ListHead->Blink = Entry
mov [edx], ecx  ; Blink->Flink = Entry
```

Snippets in `nt!KeInsertQueueDpc`

```auto
mov ecx, dword ptr [ebx+4]
lea eax, [esi+4]
mov dword ptr [eax], ebx
mov dword ptr [eax+4], ecx
mov dword ptr [ecx], eax
mov dword ptr [ebx+4], eax
```

### RemoveHeadList

C pseudocode

```c
Entry = ListHead->Flink;
Remain = Entry->Flink;
ListHead->Flink = Remain;
Remain->Blink = ListHead;
```

Asm

```
mov ecx, [eax]   ; Entry = ListHead->Flink
mov edx, [ecx]   ; Remain = Entry->Flink
mov [eax], edx   ; ListHead->Flink = Remain
mov [edx+4], eax ; Remain->Blink = ListHead
```

Snippets in `nt!CcDeleteMbcb`

```
mov     edi, dword ptr [eax]    ; Entry = ListHead->Flink
cmp     edi, eax			   ; if List is empty then jump out
je 		...
mov     ecx, dword ptr [edi]    ; Remain = Entry->Flink
mov     eax, dword ptr [edi+4]
mov     dword ptr [eax], ecx    ; ListHead->Flink = Remain
mov     dword ptr [ecx+4], eax  ; Remain->Blink = ListHead
```

### RemoveTailList

C pseudocode

```c
Entry = ListHead->Blink;
Remain = Entry->Blink;
ListHead->Blink = Remain;
Remain->Flink = Entry;
```

Asm

```
mov ecx, [eax+4]	; Entry = ListHead->Blink
mov edx, [ecx+4]	; Remain = EntryHead->Blink
mov [edx], eax       ; Remain->Flink = ListHead
mov [eax+4], edx	; ListHead->Blink = Remain
```

Snippets in `nt!ObpPostOperationCallbacks`

```
mov     edi,dword ptr [ebp+0Ch]  ; ListHead
cmp     dword ptr [edi],edi      ; if List is empty then jump out
je      nt!ObpCallPostOperationCallbacks+0x67 (817c92f9)
mov     ebx,dword ptr [edi+4]    ; Entry = ListHead->Blink
mov     eax,dword ptr [ebx+4]    ; Remain = Entry->Blink
mov     dword ptr [edi+4],eax    ; ListHead->Blink = Remain
mov     dword ptr [eax],edi      ; Remain->Flink = ListHead
```

### RemoveEntryList

C pseudocode

```c
Pre = Entry->Flink;
Post = Entry->Blink;
Pre->Flink = Post;
Post->Blink = Pre;
```

Asm

```
mov ecx, [eax]   ; Post
mov edx, [eax+4] ; Pre
mov [edx], ecx   ; Pre->Flink = Post
mov [ecx+4], edx ; Post->Blink = Pre
```

Snippets in `nt!CmpPostApc`

```
lea     eax,[esi+8]
mov     edx,dword ptr [eax]
mov     ecx,dword ptr [eax+4]
mov     dword ptr [ecx],edx
mov     dword ptr [edx+4],ecx
```





