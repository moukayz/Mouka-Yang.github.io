---
title: Finding the most common element in an iterable
categories:
- Python Tricks
---
<!-- more -->

`collections.Counter` lets you find the most common elements in an iterable:

```python
>>> import collections
>>> c = collections.Counter('helloworld')

>>> c
Counter({'l': 3, 'o': 2, 'e': 1, 'd': 1, 'h': 1, 'r': 1, 'w': 1})

>>> c.most_common(3)
[('l', 3), ('o', 2), ('e', 1)]
```