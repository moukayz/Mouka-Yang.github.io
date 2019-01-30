---
title: get()方法访问字典
categories:
- Python Tricks
feature_image: "https://picsum.photos/2560/600?image=872"
---
<!-- more -->

**get(keyword, default_arg)**

```python
name_for_userid = {
    382: "Alice",
    590: "Bob",
    951: "Dilbert",
}

def greeting(userid):
    return "Hi %s!" % name_for_userid.get(userid, "there")

>>> greeting(382)    # 访问 key 382 ，返回 value
"Hi Alice!"

>>> greeting(333333) # 访问 key 333333，不存在，使用默认value
"Hi there!"
```

