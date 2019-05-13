---
title: Effective C++ —— Part 1
categories:
- C/C++


---

<!-- more -->

## 2. Prefer consts , enums, inlines to #defines

- For simple constants, prefer **const** objects or **enum** to #defines

  ```c++
  class CostEstimate {
  private:
      static const double FudgeFactor; // declaration of static class
      ... 							// constant; goes in header file
  };
  const double // definition of static class
  CostEstimate::FudgeFactor = 1.35; // constant; goes in impl. file
  
  // Or use enums
  class GamePlayer {
  private:
      enum { NumTurns = 5 }; // “the enum hack” — makes
      					// NumTurns a symbolic name for 5
      int scores[NumTurns]; // fine
      ...
  };
  ```

  

- For function-like macros, prefer inline function to #defines

  ```c++
  template<typename T> // because we don’t
  inline void callWithMax(const T& a, const T& b) // know what T is, we
  { 												// pass by reference-to
      f(a > b ? a : b); // const — see Item 20
  }
  
  // Instead of 
  // #define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
  ```

  

## 3. Use const whenever possible

- Declare something **const** helps compilers **detect usage errors**. 

- **const** iterator

  ```c++
  const std::vector<int>::iterator iter = // iter acts like a T* const
  vec.begin();
  *iter = 10; // OK, changes what iter points to
  ++iter; // error! iter is const
  
  std::vector<int>::const_iterator cIter = // cIter acts like a const T*
  vec.begin();
  *cIter = 10; // error! *cIter is const
  ++cIter; // fine, changes cIter
  ```

  

- **const** member function -- work with **const** objects

  ```c++
  class TextBlock {
  public:
  ...
      const char& operator[](std::size_t position) const // operator[] for
      { return text[position]; } // const objects
      
      char& operator[](std::size_t position) // operator[] for
      { return text[position]; } // non-const objects
  private:
  	std::string text;
  };
  
  // Can be used like this
  TextBlock tb("Hello");
  std::cout << tb[0]; // calls non-const
  
  // TextBlock::operator[]
  const TextBlock ctb("World");
  std::cout << ctb[0]; // calls const TextBlock::operator[]
  ```

- Use **mutable** member variables in **const** member function

  ```c++
  class CTextBlock {
  public:
      ...
      std::size_t length() const;
  private:
      char *pText;
      mutable std::size_t textLength; // these data members may
      mutable bool lengthIsValid; // always be modified, even in
  }; 
  // const member functions
  std::size_t CTextBlock::length() const
  {
      if (!lengthIsValid) {
      textLength = std::strlen(pText); // now fine
      lengthIsValid = true; // also fine
  }
  return textLength;
  }
  ```

- When const and non-const member functions have essentially identical implementations, code duplication can be avoided by having the non-const version call the const version 

  ```c++
  // The const and non-const member function do exactly the same things except the // return value
  class TextBlock {
  public:
  ...
  const char& operator[](std::size_t position) const
  {
      ... // do bounds checking
      ... // log access data
      ... // verify data integrity
      return text[position];
  }
  char& operator[](std::size_t position)
  {
      ... // do bounds checking
      ... // log access data
      ... // verify data integrity
      return text[position];
  }
  private:
  	std::string text;
  };
  
  // having the non-const version call the const version 
  class TextBlock {
  public:
      ...
      const char& operator[](std::size_t position) const // same as before
      {
          ...
          ...
          ...
          return text[position];
      }
      char& operator[](std::size_t position) // now just calls const op[]
      {
          return
          const_cast<char&>( // cast away const on op[]’s return type;
              static_cast<const TextBlock&>(*this)[position] 
              // add const to *this’s type;
              // call const version of op[]
          );
      }
  ...
  };
  ```



## 4. Make sure that objects are initialized before being used

- Always initialize your objects before you use them

  - For non-member objects or built-in types, do this manually

    ```c++
    int x = 0; // manual initialization of an int
    const char * text = "A C-style string"; // manual initialization of a
    										// pointer (see also Item 3)
    double d; // “initialization” by reading from
    std::cin >> d; // an input stream
    ```

  - Make sure that all constructors initialize everything in the object

- Member variables are initialized before entering into the constructor body. Their initialization **take place when their default constructors are called prior to entering to the body of the class constructor**.

- prefer use of the member initialization list to assignment inside the body of the constructor. List data members in the initialization list in the same order they’re declared in the class. 


 ```c++
class PhoneNumber { ... };
class ABEntry { // ABEntry = “Address Book Entry”
    public:
    ABEntry(const std::string& name, const std::string& address,
            const std::list<PhoneNumber>& phones);
    private:
    std::string theName;
    std::string theAddress;
    std::list<PhoneNumber> thePhones;
    int numTimesConsulted;
};

/* 
  ABEntry::ABEntry(const std::string& name, const std::string& address,
  const std::list<PhoneNumber>& phones)
  {
      // They have been initialized.
      // these are all assignments, not initializations. 
      theName = name; 
      theAddress = address; 
      thePhones = phones;
      numTimesConsulted = 0;
  }
  */

// Instead, using member initialization list
ABEntry::ABEntry(const std::string& name, const std::string& address,
                 const std::list<PhoneNumber>& phones)
    // these are now all initializations
    : theName(name),
    theAddress(address), 
    thePhones(phones),
    numTimesConsulted(0)
{}

// Default constructor
ABEntry::ABEntry()
    : theName(), // call theName’s default ctor;
    theAddress(), // do the same for theAddress;
    thePhones(), // and for thePhones;
    numTimesConsulted(0) // but explicitly initialize
{}

 ```



## 5. Know what functions  C++ silently writes and calls

- If you define a empty class, the compiler will declare its own version of a copy constructor, a copy assginment constructor, a default constructor and a destructor. All these functions are both **public** and **inline**

  ```c++
  // If you write
  class empty{};
  
  // Actually it will be
  class Empty {
  public:
      Empty() { ... } // default constructor
      Empty(const Empty& rhs) { ... } // copy constructor
      ~Empty() { ... } // destructor — see below
      // for whether it’s virtual
      Empty& operator=(const Empty& rhs) { ... } // copy assignment operator
  };
  ```

- These functions are generated only if they are needed

  ```c++
  // The following code will cause each function to be generated
  Empty e1; // default constructor; destructor
  Empty e2(e1); // copy constructor
  e2 = e1; // copy assignment operator
  ```

  

  ## 6. Explicitly disallow the use of compiler generated functions you don't want

  - To disallow functionality automatically provided by compilers, **declare the corresponding member functions private and give no implementations**. Using a base class like **Uncopyable** is one way to do this. 

