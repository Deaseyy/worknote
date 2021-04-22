相关命令函数实现的位置：redis 包下的client 

## 获取原生redis连接

```python
# 用django_redis包，（需在django的settings设置）
from django_redis import get_redis_connection

conn = get_redis_connection()

# 用redis包
from redis import StrictRedis

```

