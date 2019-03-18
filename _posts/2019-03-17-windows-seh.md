---
title: Glance at Winodws Standard Exception Handler(SEH) and Vectored Exception Handler(VEH)
categories:
- Windows Internal
---
<!-- more -->

 *This post simply describes the mechanism of the Windows exception handling, for more detailed information see:*

http://bytepointer.com/resources/gordon_win32_exception_handling.htm

http://bytepointer.com/resources/pietrek_crash_course_depths_of_win32_seh.htm

http://bytepointer.com/resources/pietrek_vectored_exception_handling.htm

## SEH

In Visual C++ compiler, the SEH is implemented by the keywords `__try`, `__except` and `__finally`.

In general,

 `__try` block is the protected area, which exceptions may occur.

`__except`block is the exception handler which programmers should customize to handle exceptions.

`__finally`block is used to clean dirty things (release resources, etc.) and the codes in the block are guaranteed to be executed whether or not there is a exception.

### Steps for exception handling

1. When a exception occurs, Windows check if the program is being debugged, 

   if so, Windows will **notify the debugger** of the exception by suspending the program and sending ` EXCEPTION_DEBUG_EVENT`  (0x1) to the debugger.

2. If the program is not being debugged or if the exception is not dealt with by the debugger,

   the system sends the exception to your per-thread exception handler.  A per-thread handler is installed at run-time(installed by `__except` or `__finally`). and **is pointed to by the first dword in the Thread Information Block** (whose address is at FS:[0] in x86 and GS:[0] in x64)

3. The per-thread exception handler can try to deal with the exception, or leave it for handlers further up the chain.

   (FS:[0] or GS:[0] points to the latest installed exception handler and it has a pointer which points to the previous installed handler, in simple terms, FS:[0] or GS:[0] is the head of the **Exception Handler List**) , 

4. Eventually if none of the per-thread handlers deal with the exception, if the program is being debugged the system will **again suspend the program and notify the debugger.**

5. If the program is not being debugged or if the exception is still not dealt with by the debugger, the system will call your final handler if one is installed. This will be a final handler installed at run-time by the application using the API `SetUnhandledExceptionFilter`

6. If your final handler does not deal with the exception after it returns, the system final handler will be called. Optionally it will show the system's closure message box. Depending on the registry settings, this box may give the user a chance to attach a debugger to the program. If no debugger can be attached or if the debugger is powerless to assist, the program is doomed and the system will call `ExitProcess` to terminate the program.

7. Before finally terminating the program, though, the system will cause a "final unwind" of the stack for the thread in which the exception occurred.

![img](http://bytepointer.com/resources/pietrek_crash_course_depths_of_win32_seh_files/pietrek4.jpg)



## VEH

In a nutshell, vectored exception handling is similar to regular SEH, with three key differences:

- Handlers aren't tied to a specific function nor are they tied to a stack frame.
- The compiler doesn't have keywords (such as try or catch) to add a new handler to the list of handlers.
- Vectored exception handlers are explicitly added by your code, rather than as a byproduct of try/catch statements.

Programs should use `AddVectoredExceptionHandler` API (declared as below) to add his own handlers into the linked list of registered handlers. (**You can install as many vectored handlers as you want**)

If `FirstHandler` is not zero, then your handler  function `VectoredHandler` is placed in the head of the linked list (**Get called first**), or in the tail if zero.

```c++
WINBASEAPI PVOID WINAPI AddVectoredExceptionHandler(
    ULONG FirstHandler,
    PVECTORED_EXCEPTION_HANDLER VectoredHandler );
```

### Tips for VEH

- the vectored exception handler list is processed before the normal SEH list. 

- when a program is being debugged, the debugger still sees the first chance exception **before the target process does.**

- The vectored handler list is not tied to any thread, and **is global to the process.**

- The function is expected to return either `EXCEPTION_CONTINUE_SEARCH` or `EXCEPTION_CONTINUE_EXECUTION`. ( not `EXCEPTION_EXECUTE_HANDLER`)

  - In `EXCEPTION_CONTINUE_EXECUTION`, the program resume its execution (vectored handlers appear latter in the list won't be called 
  - In `EXCEPTION_CONTINUE_SEARCH`, the system moves on to the next vectored handlers

  

