### 背景

我们知道django的queryset在每次for循环时是缓存的，也就是第一次for时会发出sql请求，以后的for循环会使用已请求的缓存。

但django 在queryset里面再次get(filter)会再次发起sql查询。这会导致如果for循环queryset做进一步筛选的话，有多少次就会发多少次sql查询

### 自定义对queryset计算之后的结果作缓存

```python
def cache_filter(queryset, **option):
    """queryset 缓存filter"""
    return [query for query in queryset if all(getattr(query, k) == v for k, v in option.iteritems())]


def cache_get(queryset, **option):
    """queryset 缓存get"""
    result = cache_filter(queryset, **option)
    if len(result) != 1:
        raise ValueError("get should return one record, but now:%s" % len(result))
    return result[0]
```

通过for循环去做，时间复杂度O(n)，尤其是外层还有个for循环，我们可以使用字典去优化一下

```python
def fast_cache_filter(queryset, *keys):
    """使用字典加速cache filter"""
    cache = defaultdict(list)

    for query in queryset:
        v = [getattr(query, key) for key in keys]
        cache[tuple(v)].append(query)

    def query_filter(**option):
        assert len(keys)==len(option), "filter keys and option must same"
        return cache.get([option[key] for key in keys], [])

    return query_filter


def fast_cache_get(queryset, *keys):
    """使用字典加速cache get"""
    query_filter = fast_cache_filter(queryset, *keys)

    def query_get(**option):
        result = query_filter(**option)
        if len(result) != 1:
            raise ValueError("get should return one record, but now:%s" % len(result))
        return result[0]

    return query_get
```



使用第二种方案侵入性要比第一种高，必须先告诉fast_cache_get函数你的queryset和get的key,fast_cache_get会返回一个函数，这个函数使用就类似了cache_get的用法，典型的闭包。

经过比较，fast版本稍微快10%-20%，但侵入性大，最后我们放弃fast版本，使用for循环的版本。
