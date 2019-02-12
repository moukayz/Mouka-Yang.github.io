---
title: Lambda function
categories:
- Python Tricks
---
<!-- more -->

The lambda keyword in Python provides a shortcut for declaring small and anonymous functions:
```python
>>> add = lambda x, y: x + y
>>> add(5, 3)
8
```
You could declare the same add()function with the def keyword:
```python
>>> def add(x, y):
...     return x + y
>>> add(5, 3)
8
```
you can also use:
```python
>>> (lambda x, y: x + y)(5, 3)
8
```
• Lambda functions are single-expression functions that are not necessarily bound to a name (they can be anonymous).

• Lambda functions can't use regular Python statements and always include an implicit `return` statement.