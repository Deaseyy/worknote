一般我们对多线程中的全局变量都会加锁处理，这种变量是共享变量，每个线程都可以读写变量，为了保持同步我们会做加锁处理。

**但是有些变量初始化以后，我们只想让他们在每个线程中一直存在，相当于一个线程内的共享变量，线程之间又是隔离的。** python threading模块中就提供了这么一个类，叫做local。local是一个小写字母开头的类，用于管理 thread-local（线程局部的）数据。**对于同一个local对象，线程无法访问其他线程设置的属性；线程设置的属性不会被其他线程设置的同名属性替换。**

可以把local看成是一个“线程-属性字典”的字典，local封装了从自身使用线程作为 key检索对应的属性字典、再使用属性名作为key检索属性值的细节。

local对象类似如：

```python
{
	'线程id1'：{'a':1, b:2},
	'线程id1'：{'a':1, b:2},
	'线程id1'：{'a':1, b:2},
}
# 各线程之间的属性互不干扰
```



正式开始之前，先对比一下局部变量和全局变量。

### 1.局部变量

```python
import threading
import time
 
def worker():
    x = 0
    for i in range(100):
        time.sleep(0.0001)
        x += 1
    print(threading.current_thread(),x)
 
for i in range(3):
    threading.Thread(target=worker).start()
```

结果分析：

```
运行结果：
<Thread(Thread-2, started 23764)> 100
<Thread(Thread-3, started 24316)> 100
<Thread(Thread-1, started 19564)> 100
```

x是线程内部的局部变量，线程之间互不干扰，计算结果都保持一致。



### 2.全局变量：

```python
import threading
import time

x = 0

def worker():
    global x
    x = 0
    for i in range(100):
        time.sleep(0.0001)
        x += 1
    print(threading.current_thread(), x)

for i in range(3):
    threading.Thread(target=worker).start()


```

**结果1：**

```
<Thread(Thread-1, started 18456)> 297
<Thread(Thread-2, started 21316)> 299
<Thread(Thread-3, started 6536)> 300
```

主线程中x是全局变量(进程级别)时，就变成了公共资源(即同一个对象)，每个子线程互相干扰，都会去操作x。

**结果2：**

```
<Thread(Thread-2, started 21096)> 290
<Thread(Thread-3, started 18396)> 293
<Thread(Thread-1, started 11576)> 296
```

从这个结果看出：它最终的结果并非是300，所以它是非线程安全的，一个线程可能覆盖另一个线程的计算结果。

例如： 三个线程同时取到了x的值为10，都还没来得及加1和赋值操作，然后线程1执行: x=10+1，线程2执行：x=10+1, 线程3执行: x=10+1，经过三个线程的修改，最终结果变成了11，但正常来说x应该要被加3, 即x=13。

所以才有了上面的结果，而且并发的线程越多，这种情况就越容易出现，可以增加循环次数来看看。

这种情况我们一般都会采取给变量加锁。



### 3.local对象

**Python提供了 threading.local 类，实例化后得到一个对象，但是不同的线程使用这个对象存储的数据其它线程不可见(本质上就是不同的线程使用这个对象时为其创建一个独立的字典)。**

```python
import threading
import time

# class G():
#     pass
# g = G()  # 全局的普通对象g
g = threading.local()  # 全局的local对象g

def worker():
    g.x = 0
    for i in range(100):
        time.sleep(0.0001)
        g.x += 1
    print(threading.current_thread(), g.x)

for i in range(3):
    threading.Thread(target=worker).start()
```

**结果分析：**

```
# 每个线程的最终运行结果都是一致的，即100
<Thread(Thread-1, started 22220)> 100
<Thread(Thread-2, started 20820)> 100
<Thread(Thread-3, started 5696)> 100
```

若使用全局的普通对象g，将会和上面使用全局变量x的效果一致（python中一切皆对象）。

每个子线程都使用这个全局的local对象g，但每个子线程存储在这个g对象的属性g.x是该线程独有的，其他子线程(包括主线程)都访问不到。



**简单验证如下：**

```python
y = 10
g = threading.local()
g.x = 5
def work():
    print(y)
    print(g.x)  # AttributeError: '_thread._local' object has no attribute 'x'

threading.Thread(target=work).start()
```

由此可见，子线程中无法访问主线程存储在g对象上的x属性，子线程本身又没有设置x属性，所以报错。

所以local对象的属性是每个线程独有的，其他线程无法访问。



