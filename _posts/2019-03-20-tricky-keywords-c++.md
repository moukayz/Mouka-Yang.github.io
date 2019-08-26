---
title: Tricky Keyword in Cpp
categories:
- Cpp

---

<!-- more -->

- **Internal linkage**: the name is not visible outside the file in which the variable is declared.
- **External linkage**: the name of the variable is visible from outside the file in which the variable is declared.     

# static

The **static** storage class means that the variable or object **is** **allocated when program starts and is deallocated when the program ends**. By default an object or variable that is defined in the global namespace has **static duration and external linkage**. The **static** keyword can be used in the following situations.

- Global static variable or function: 

  - the variable or function has **internal linkage**, 
  - the variable has **static duration** and the compiler **initializes it to 0** unless you specify another value.
  - the variables or objects are initialized **before entry to the program's main function**

- Local static variable (defined in a function): 

  - the variable has **static duration** 
  - retains its state between calls to that function.
  - the variable or object is initialized **the first time the flow of control reaches its definition** 

  ```c++
  void showstat( int curr ) {
     static int nStatic;    // Value of nStatic is retained
                            // between each function call
     nStatic += curr;
     cout << "nStatic is " << nStatic << endl;
  }
  
  int main() {
     for ( int i = 0; i < 5; i++ )
        showstat( i );
  }
  
  /* output
  nStatic is 0
  nStatic is 1
  nStatic is 3
  nStatic is 6
  nStatic is 10
  */
  ```

- Class static data member: 

  - one copy of the member is shared by all instances of the class.
  - a static data member must be defined at file scope.

  ```c++
  class CMyClass {
  public:
     static int m_i; // not defined yet
  };
  
  // static data member can only defined at file scope
  // instead of function or block
  int CMyClass::m_i = 0; 
  CMyClass myObject1;
  CMyClass myObject2;
  
  int main() {
     cout << myObject1.m_i << endl; // 0
     cout << myObject2.m_i << endl; // 0
  
     myObject1.m_i = 1;
     cout << myObject1.m_i << endl; // 1
     cout << myObject2.m_i << endl; // 1
  
     myObject2.m_i = 2;
     cout << myObject1.m_i << endl; // 2
     cout << myObject2.m_i << endl; // 2
  ```

- Class static function member:

  - the function is shared by all instances of the class
  - the function cannot access an instance member

# extern

Objects or variables declared as **extern** declare an object that is defined in **another translation unit (another source file) or in an enclosing scope as having external linkage**. It can be used in following conditions.

* **In a non-const global variable declaration**: **extern** specifies the variable or function is defined in another translation unit. ( The **extern** keyword must be applied in all files except the one where the variable is defined, because global non-const variables are external by default)

```c++
//fileA.cpp
int i = 42; // declaration and definition

//fileB.cpp
extern int i;  // declaration only. same as i in FileA

//fileC.cpp
extern int i;  // declaration only. same as i in FileA

//fileD.cpp
int i = 43; // LNK2005! 'i' already has a definition.
extern int i = 43; // same error (extern is ignored on definitions)
```

* **In a const global variable declaration:** it specifies that the variable has extern linkage. The **extern** keyword must be applied to all declarations in all files, including the one where the variable is defined ( global const variables have internal linkage by default)

```c++
//fileA.cpp
extern const int i = 42; // extern const definition

//fileB.cpp
extern const int i;  // declaration only. same as i in FileA
```

For example, the following code shows two **extern** declarations. 

```c++
// DefinedElsewhere is defined in another translation unit
extern int DefinedElsewhere;
int main() {
   int DefinedHere;
   {
      // refers to DefinedHere in the enclosing scope
      extern int DefinedHere;
   }
}
```

# thread_local

The **thread_local** specifier makes the variable is accessible only on the thread on which it is created. 

* The variable is **created when the thread is created, and destroyed when the thread terminates.** 
* Each thread has its own copy of the variable.
* On Windows, **thread_local** is functionally equivalent to **__declspec(thread)**

* **thread_local** can only be applied to data declarations and definitions (not functions)
* **thread_local** can only be applied to data items with **static** storage duration (including global data, local static data and static data member of classes). So at block scope **thread_local** is equivalent to **thread_local static**.
* Must specify **thread_local** for both the declaration and the definition of a thread local object , whether the declaration and the definition occur in the same file or separate files.

```c++
thread_local float f = 42.0; // Global namespace. Not implicitly static.

struct S // cannot be applied to type definition
{
    thread_local int i; // Illegal. The member must be static.
    thread_local static char buf[10]; // OK
};

void DoSomething()
{
    // Apply thread_local to a local variable.
    // Implicitly "thread_local static S my_struct".
    thread_local S my_struct;
}
```

