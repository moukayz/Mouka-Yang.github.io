---
title: 计算代码执行时间
categories:
- Python Tricks
feature_image: "https://picsum.photos/2560/600?image=872"
---
<!-- more -->

`timeit 模块可用来测量`**`少量代码`**`的执行时间` **``**

```python
>>> import timeit
>>> timeit.timeit('"-".join(str(n) for n in range(100))',
                  number=10000)
0.3412662749997253

>>> timeit.timeit('"-".join([str(n) for n in range(100)])',
                  number=10000)
0.2996307989997149

>>> timeit.timeit('"-".join(map(str, range(100)))',
                  number=10000)
0.24581470699922647
```

