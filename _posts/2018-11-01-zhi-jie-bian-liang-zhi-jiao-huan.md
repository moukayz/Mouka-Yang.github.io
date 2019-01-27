---
title: 直接变量值交换
categories:
- Python tricks
feature_image: "https://picsum.photos/2560/600?image=872"
---
<!-- more -->

```python
a = 23
b = 42

# 普通方法
tmp = a
a = b
b = tmp

# Oops ！
a, b = b, a
```

