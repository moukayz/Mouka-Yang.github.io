---
title: 打印Python字典
categories:
- Python Tricks
feature_image: "https://picsum.photos/2560/600?image=872"
---
<!-- more -->

**标准字典打印方法有时候难以阅读（单行表达）**

```python
>>> my_mapping = {'a': 23, 'b': 42, 'c': 0xc0ffee}
>>> my_mapping
{'b': 42, 'c': 12648430. 'a': 23}
```

**"json" 模块可以以另一种方式打印字典**

```python
>>> import json

>>> json_dict = json.dumps(
    my_mapping,    # 将字典转化为 json 格式
    indent=4,    # 设置缩进为 4
    sort_keys=True)    # 以 key 排序
>>> print()
{
    "a": 23,
    "b": 42,
    "c": 12648430
}

# 注意该方法只适用于 key 为原子类型（str,int ...)的字典
>>> json.dumps({all: 'yup'})
TypeError: keys must be a string
```

