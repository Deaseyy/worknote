## 什么是反射？？

　　在面向对象的思想中，把对象能够访问、查询、修改自身的状态或行为称为“反射”。



## python中的反射

　　Python思想：一切事物皆为对象。

　　在python中，可以通过字符串的的形式来操作对象的属性。这种行为称之为python中的反射



## python实现反射（自省）的手段：

　　通过4个内置函数来实现：hasattr(object,name)      getattr(object,name,default=None)   setattr(x,y,v)   delattr(x,y)

**注意：不仅仅是实例对象才能使用反射，类本身也是对象，所以类也可以用反射，即类也可以作为object参数参入到上述四个函数中去，文件再python中也被看作是对象，也可以使用，比如python模块对象**

### 用例场景：

根据用户输入的url的不同，调用不同的函数，实现不同的操作，也就是一个url路由器的功能

##### getattr()  字符串反射: 根据字符串去获取对应函数名

```python
import commons
def run():
    inp = input("请输入您想访问页面的url：  ").strip()
    if hasattr(commons,inp):
        func = getattr(commons,inp)
        func()
    else:
        print("404")
```



### 动态导入模块

**1. python提供了一个特殊的方法：__ import __ (字符串参数)。通过它，我们就可以实现类似的反射功能。**

**__ import __()方法会根据参数，动态的导入同名的模块。**

**2. `importlib`库中的 import_module()方法**

```python
def run():
    inp = input("请输入您想访问页面的url：  ").strip()
    modules, func = inp.split("/")
    # __import__默认只会导入最开头的圆点左边的目录，也就是“lib”
    # obj = __import__("lib." + modules)   所以这会有问题
    obj = __import__("lib." + modules, fromlist=True)  # 使用fromlist参数
    if hasattr(obj, func):
        func = getattr(obj, func)
        func()
    else:
        print("404")
        
# 还可使用 importlib库
importlib.import_module('lib.xxx')
```

