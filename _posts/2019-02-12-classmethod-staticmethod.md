---
title: classmethod vs staticmethod vs native method
categories:
- Python Tricks
---
<!-- more -->

```python
class MyClass:
    """
        Instance methods need a class instance and
        can access the instance through `self`.
    """
    def method(self):
        return "instance method called", self

    """
        Class methods don't need a class instance.
        They can't access the instance (self) but
        they have access to the class itself via `cls`.
    """
    @classmethod
    def classmethod(cls):
        return "class method called", cls

    """
        Static methods don't have access to `cls` or `self`.
        They work like regular functions but belong to
        the class's namespace.
    """
    @staticmethod
    def staticmethod(cls):
        return "static method called"
```
All method types can be called in a class instance
```python
>>> obj = MyClass()
>>> obj.method()
('instance method called', <MyClass instance at 0x1019381b8>)
>>> obj.classmethod()
('class method called', <class MyClass at 0x101a2f4c8>)
>>> obj.staticmethod()
'static method called'
```
Calling instance method failed if we only have the class object:
```python
>>> MyClass.classmethod()
('class method called', <class MyClass at 0x101a2f4c8>)
>>> MyClass.staticmethod()
'static method called'
>>> MyClass.method()
TypeError: 
    "unbound method method() must be called with MyClass "
    "instance as first argument (got nothing instead)"
```
