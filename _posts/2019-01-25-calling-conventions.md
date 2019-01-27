---
title: Windows calling conventions
categories:
- Reverse Engineering
feature_image: "https://picsum.photos/2560/600?image=872"
---
<!-- more -->

## x86

There are three primary calling conventions whose specifier are `stdcall`, `cdecl` and `fastcall`

### Parameters

| Keyword | Stack cleanup | Parameter passing |
| :--- | :--- | :--- |
| \_\_cdecl | caller | pushed on the stack \( right to left\) |
| \_\_stdcall | callee | pushed on the stack \(right to left\) |
| \_\_fastcall | callee | first two passed by ECX,EDX \( left to right\); Others passed on the stack \(right to left\) |

\( [see MSDN Specification](https://docs.microsoft.com/en-us/cpp/cpp/calling-conventions?view=vs-2017) \)

**Naked**: Functions declared with the **naked**attribute are emitted without **prolog** or **epilog** code \( [Naked Function calls](https://docs.microsoft.com/en-us/cpp/cpp/naked-function-calls?view=vs-2017) \)

You can specify **naked** function with above calling conventions like:

```c
// processor: x86
__declspec(naked) int __fastcall  power(int i, int j) {
   // calculates i^j, assumes that j >= 0

   // prolog
   __asm {
      push ebp
      mov ebp, esp
      sub esp, __LOCAL_SIZE
     // store ECX and EDX into stack locations allocated for i and j
     mov i, ecx
     mov j, edx
   }

   {
      int k = 1;   // return value
      while (j-- > 0)
         k *= i;
      __asm {
         mov eax, k
      };
   }

   // epilog
   __asm {
      mov esp, ebp
      pop ebp
      ret
   }
}
```

## x64

Actually there is only the `fastcall` in x64 platform

### Parameters

| Floating point | First 4 parameters - XMM0 through XMM3. Others passed on stack. |
| :--- | :--- |
| Integer | First 4 parameters - RCX, RDX, R8, R9. Others passed on stack. |
| Aggregates \(8, 16, 32, or 64 bits\) and \_\_m64 | First 4 parameters - RCX, RDX, R8, R9. Others passed on stack. |
| Aggregates \(other\) | By pointer. First 4 parameters passed as pointers in RCX, RDX, R8, and R9 |
| \_\_m128 | By pointer. First 4 parameters passed as pointers in RCX, RDX, R8, and R9 |

**passing examples**:

```c
func1(int a, int b, int c, int d, int e);
// a in RCX, b in RDX, c in R8, d in R9, e pushed on stack

func2(float a, double b, float c, double d, float e);
// a in XMM0, b in XMM1, c in XMM2, d in XMM3, e pushed on stack

func3(int a, double b, int c, float d);
// a in RCX, b in XMM1, c in R8, d in XMM3

func4(__m64 a, _m128 b, struct c, float d);
// a in RCX, ptr to b in RDX, ptr to c in R8, d in XMM3
```



