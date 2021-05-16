

# 一，合理使用数据结构

## string

### 单值存储

set key value

get key

### 对象存储

对于一个数据库三个字段(id,name,balance)的表结构：

#### 方式1：

一条记录存储一个用户对象

```
set user:1 value(json格式数据)
```

优缺点：

- 耗内存，会存储整个对象的所有字段
- 取值时也必须取所有字段
- 可用于无需取单字段的整体缓存

#### 方式2：

一条记录存储一个用户对象

```
mset user:1:name zhangsan user:1:balance 1888
mget user:1:name user:1:balance
```

优缺点：

- 节约内存，存取方便，可以只存取需要的字段



## hash

### 对象存储

一条记录存储所有用户对象

- hmset  批量设置value

```
// 保存第一个user
hmset user 1:name zhangsan 1:balance 1888
hmget user 1:name 1:balance
// 保存第二个user
hmset user 2:name lisi 2:balance 1777
hmget user 2:name 2:balance
```

![image-20201020000139832](C:\Users\12395\AppData\Roaming\Typora\typora-user-images\image-20201020000139832.png)





# 二，实战使用场景

## string

#### 1.计数器

```
incr article:readcount:文章id
get articl:readcount:文章id
```

场景：

- 文章（页面）浏览次数

#### 2.分布式系统全局序列号

```
// 每次加1，高并发下会影响效率，redis每秒tps并发不到10W,
incr orderId  

// (多个web集群)每次每个服务器取1000个id到本服务器内存，慢慢使用，用完再取
incrby orderId 1000   
```

场景：

- 分库分表的主键id

#### 3.web集群session共享

redis实现session共享



## hash

hash结构不太适合在redis集群使用

#### 1.电商购物车

1.以用户id为key，  `cart:用户id`

2.商品id为field

3.该商品数量为value

```
hset cart:1001 10099 1   // 添加第1个商品10099
hset cart:1001 20088 1   // 添加第2个商品20088
hincrby cart:1001 10099 1  // 该商品数加1
hget cart:1001 10099  // 获取该商品数
hlen cart:1001  // 获取商品种类数量
hdel cart:1001 20088  // 删除该商品
hgetall cart:1001  // 获取所有种类商品及个数
```



## list

#### 1.微博消息和微信公众号消息推送

假如用户M关注了A和B两个大v，那么当大V发布消息后，会有后台任务去将消息推送给订阅他的粉丝

```
1.A发微博，消息id为10018
  lpush msg:用户M的id 10018
2.B发微博，消息id为10086
  lpush msg:用户M的id 10086
3.用户M查看最新微博消息
  lrange msg:用户M的id 0 5   // 查看前五条
```



## set

#### 1.抽奖小程序

key 为活动id：`active:活动id`

value 为用户id

```
1.点击参与抽奖加入集合
 sadd active:101 用户id
2.查看参与抽奖所有用户
 smembers active:101
3.抽取count名中奖者
 srandmember active:101 {count}  // 随机取几个，不删除
 spop active:101 {count}   // 随机取几个，并踢除
```

#### 2.微信微博点赞，收藏，标签

key为点赞的消息id： `like:消息id`

value 为用户id

```
1.点赞
 sadd like:{消息id} {用户id}
2.取消点赞
 srem like:{消息id} {用户id}
3.检查用户是否点过赞
 sismember like:{消息id} {用户id}
4.获取点赞的用户列表
 smembers like:{消息id}
5.获取点赞用户数
 scard like:{消息id}
```

#### 3.集合操作实现微博微信关注模型

sinter  取交集

sismember  判断集合中是否有指定元素

```
1.用户A关注的人：
 Aset->{B,C,D}
2.用户B关注的人：
 Bset->{A,C,D,E}
3.用户C关注的人：
 Cset->{A,B,D,E,F}
4.用户A和用户B共同关注的人：
 sinter A B
5.我关注的人也关注他(B)：
 sismember Cset B  // B是否在我关注的人的关注集合中
 sismember Dset B
6.我可能认识的人：
 sdiff Bset Aset   // (差集)我关注的人的关注集合中，我没有关注的人

```

#### 4.集合操作实现电商商品按商标筛选

商标为集合名

```
sadd brand:huawei P30
sadd brand:xiaomi 6X
sadd os:android p30 6X
sadd cpu:brand:intel P30 6x
sadd ram:8G P30 6X

sinter os:android cpu:brand:intel ram:8G --> {P30,6X}
```



## Zset

#### 1.Zset有序集合实现排行榜

场景：微博热搜榜排名

```
1.点击新闻,搜索加1
  zincrby hotNews:20190101 1 守护香港
2.展示当日排行前10
  zrevrange hotNews:20190101 0 10 withscores
3.七日搜索榜单计算
  zunionstore hotNews:20190101-20190107 7 
hotNews:20190101 hotNews:20190102 ... hotNews:20190107
4.展示七日排行前十
  zrevrange hotNews:20190101-20190107 0 10 withscores
```



