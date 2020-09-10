# 一，视图集

### 1.@action（）装饰器

装饰自定义的视图函数

路由： 默认url为方法名，可使用url_path参数指定

还可接收：

- methods 对应请求方式
- detail 声明该action的路径是否与单一资源对应，及是否是`xxx/<pk>/action方法名/`
  - True 表示路径格式是`xxx/<pk>/action方法名/`
  - False 表示路径格式是`xxx/action方法名/`

```python
 @action(methods=['GET'], detail=True)
 def related_detail(self, request, *args, **kwargs):
    。。。。。。
 # 对应url：视图集类名/pk/related_detail  不能用detail作为名字（貌似重名报错）
```





# 二，认证

#### 1.继承BaseAuthentication的自定义auth类

**1.必须重写authentication() 方法**

**2.方法必须返回一个元组(user，token)**   

- 返回的user会被设置到request.user中, 默认的request.user为匿名用户
- 返回的token会被设置到request.auth中



# 三，权限

## 自定义权限

#### 1.继承BasePermission的自定义权限类

**必须重写下列方法之一或两种：**

#### a.  has_permission()  ：视图访问权限

使用视图访问权限，需使用认证类认证：否则权限禁止时返回提示： Authentication credentials were not provided，而不会正确返回自定义message信息

```python
class MyPermission(permissions.BasePermission):
    message = '没有权限访问'  # 自定义权限被禁止时的信息
    #方法返回False说明权限认证失败，True说明认证成功
    def has_permission(self,request,view):
            # 如果user_type = 0 普通用户, 返回False 
            if request.user.user_type == 0:
                return False
            # 返回True 就是权限认证成功
            return True
```

**例子：**

```python
def has_permission(self, request, view):
    """针对通用视图中某一个接口设置权限"""
    if view.action == 'update' and 。。。。:
        return False
    return True

def has_permission(self,request,view):
	# 检测只读权限 SAFE_METHODS = ('GET', 'HEAD', 'OPTIONS')
    if request.method in permissions.SAFE_METHODS:
        # Check permissions for read-only request
    else:
        # Check permissions for write request
    
def has_permission(self, request, view):
    # 黑名单检测
    ip_addr = request.META['REMOTE_ADDR']
    blacklisted = Blacklist.objects.filter(ip_addr=ip_addr).exists()
    return not blacklisted
```

#### b.  has_objects_permission()  ： 对象访问权限

```python
# 定义对象级权限
class Obj_perm(permissions.BasePermission):
    def has_object_permission(self, request, view, obj):
         # 当创建人是登录用户是才能访问对象
        if obj.owner == request.user:
            return True
        return False
    
# 使用对象级权限
# 自己写的视图在get_object()方法中显示调用该方法来检验权限
def get_object(self)
	obj = ......
	self.check_object_permissions(self.request, obj)
    return obj
# 通用视图get_object()方法中已默认调用该方法
```

**注意**：`has_object_permission`仅在视图级别`has_permission`检查已经通过的情况下，才调用实例级别的方法。还要注意，为了运行实例级检查，视图代码应显式调用`.check_object_permissions(request, obj)`。如果您使用的是通用视图，则默认情况下get_object()方法中会为您处理。（基于功能的视图将需要显式检查对象权限，从而`PermissionDenied`导致失败。）

## 使用权限

##### 局部使用：

```python
permission_classes = [MyPremission]
# 使用标准的Python按位运算符来组合权限，它支持＆（和），|。（或）和〜（不是）。
permission_classes = [权限a | 权限b]   # 任一权限通过即可
```

##### 配置全局权限：

在settings.py中，配置如下：

```python
REST_FRAMEWORK = {
   "DEFAULT_PERMISSION_CLASSES":['api.utils.permission.MyPermission'], # 路径 + 权限类
}
```



# 四，节流

#### 1.继承BaseThrottle的自定义类

**必须覆盖allow_request() 方法**

```python
class VisitThrottle(BaseThrottle):
    message = ''
    """30秒内只能访问三次"""
    def __init__(self):
        self.history = None # 初始化访问记录

    def allow_request(self, request, view):
        # 获取用户ip
        remote_addr = self.get_ident(request) # 获取ip
        ctime = time.time()
        # 如果当前IP不在访问记录里面，就添加到记录
        if remote_addr not in VISIT_RECORD:
            VISIT_RECORD[remote_addr] = [ctime]
            return True  # 允许访问

        history = VISIT_RECORD.get(remote_addr)
        self.history = history
        
        # 有访问记录，最早一次记录(列表最后)离当前时间超过30s则删除
        # 只要为True，就一直循环删除最早的一次访问记录
        while history and history[-1] < ctime - 30:
            history.pop()
        if len(history) < 3:
            history.insert(0, ctime)
            return True
        
    def wait(self):
        """还需要等多久才能访问"""
        ctime = time.time()
        message = 30 - (ctime - self.history[-1])
        # message = 30 - (ctime - self.history[0])
        return message
```

#### 2.继承SimpleRateThrottle的自定义类

**和上面的节流限制相同，封装好的更加简单**

- **必须覆盖get_cache_key()方法**
- **必须设置scope属性**

```python
class VisitThrottle2(SimpleRateThrottle):
    scope = 'rate1'  # 使用settings全局配置中的rate1对应频率
    def get_cache_key(self, request, view):
        return self.get_ident(request)
```

##### 节流全局配置：

```python
"DEFAULT_THROTTLE_CLASSES" : ["snippets.Throttles.VisitThrottle",
                             "snippets.Throttles.VisitThrottle2"],
"DEFAULT_THROTTLE_RATES":{  # 配置访问频率
    "rate1":"5/m",   # 每分钟允许访问5次
    "rate2":"5/h"  # 可以填写多个，每个视图使用不同rate
},
```



# 五，筛选过滤

## 1.过滤FilterSet

### a.覆盖get_queryset()方法

### b.filter_queryset方法过滤

#### 设置过滤器后端

- **全局设置**

```python
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ['django_filters.rest_framework.DjangoFilterBackend']
}
```

- **基于每个视图设置**

```python
class UserListView(generics.ListAPIView):
    ......
    filter_backends = [django_filters.rest_framework.DjangoFilterBackend]
```

#### 使用过滤器

- **基于等式的简单过滤**

  可以在视图或视图集上设置一个属性`filterset_fields`，列出要过滤的字段集。这将自动为给定的字段创建一个类`FilterSet`：

```python
class ProductList(generics.ListAPIView):
    ......
    filter_backends = [DjangoFilterBackend]  # 可直接全局配置
    filterset_fields = ['category', 'in_stock']
```

请求url中指定过滤条件即可：

```
http://example.com/api/products?category=clothing&in_stock=True
```

- **自定义过滤类**

  可以自定义继承`FilterSet`的过滤类，然后指定视图使用该过滤类

自定义过滤器：

```python
class MyFilter(django_filters.rest_framework.FilterSet):
    ......
```

视图中指定该类：

```python
class ProductList(generics.ListAPIView):
    ......
    filter_backends = [DjangoFilterBackend]  # 可直接全局配置
    filter_class = MyFilter
```



## 2.搜索SearchFilter

`SearchFilter`支持简单的查询参数基于搜索和基于该[admin界面的搜索功能](https://docs.djangoproject.com/en/stable/ref/contrib/admin/#django.contrib.admin.ModelAdmin.search_fields)。

```
	只有设置了search_fields属性的视图才会使用SearchFilter类，该`search_fields`属性应该是模型上文本类型字段名称的列表，例如`CharField`或`TextField`。
```

```python
from rest_framework import filters

class UserListView(generics.ListAPIView):
    ......
    filter_backends = [filters.SearchFilter]
    search_fields = ['username', 'email']
```

通过以下请求查询过滤列表中的项目：

```python
search_fields = ['username', 'email', 'profile__profession']
```

```
	默认情况下，搜索将使用不区分大小写的部分匹配。搜索参数可以包含多个搜索词，将其用空格或逗号分隔。如果使用多个搜索词，则仅当所有提供的词都匹配时，对象才会在列表中返回。
	例如：http://example.com/api/products?category=a,b
```

可以通过在前面加上各种字符来限制搜索行为`search_fields`。

- '^'开头匹配。 
- '='完全匹配。
- '@'全文搜索。（当前仅支持Django的MySQL后端。）
- '$'正则表达式搜索。

例如：

```python
search_fields = ['=username', '=email']
# 过滤字段列表中字段值以搜索参数值开头的对象
search_fields = ['^username', '^email']
```

```python
	默认情况下，搜索参数名为`'search'`，但是此参数可能会被`SEARCH_PARAM`设置覆盖。
	要根据请求内容动态更改搜索字段，可以对`SearchFilter`进行子类化并覆盖该`get_search_fields()`函数。例如，以下子类当查询参数`title_only`在请求参数中时则只在`title`上搜索：
```

```python
from rest_framework import filters

class CustomSearchFilter(filters.SearchFilter):
    def get_search_fields(self, view, request):
        # 如果能获取到title_only参数，直接只对title字段过滤
        if request.query_params.get('title_only'):
            return ['title']
        return super(CustomSearchFilter, self).get_search_fields(view, request)
    	
    	"""父类方法：
    	def get_search_fields(self, view, request):
        	return getattr(view, 'search_fields', None)
    	"""
```



## 3.排序OrderingFilter

默认情况下，查询参数名为`'ordering'`，但这可以被`ORDERING_PARAM`设置覆盖。

例如，按用户名订购用户：

```
http://example.com/api/users?ordering=username  # asc
http://example.com/api/users?ordering=-username  # desc
```

也可以指定多个字段排序：

```
http://example.com/api/users?ordering=account,username
```

### 可以限定只能针对哪些字段进行排序

建议明确指定API应在排序过滤器中允许的字段。通过`ordering_fields`在视图上设置属性：

```python
class UserListView(generics.ListAPIView):
    ......
    filter_backends = [filters.OrderingFilter]
    ordering_fields = ['username', 'email']
```

```python
	这有助于防止意外的数据泄漏，例如允许用户针对密码哈希字段或其他敏感数据进行排序。
	如果未ordering_fields在视图上指定属性，则过滤器类将默认为允许用户对由serializer_class属性指定的序列化器上的任何可读字段进行过滤。

	如果您确信该视图使用的查询集不包含任何敏感数据，则还可以通过使用special值明确指定一个视图应允许对任何模型字段或查询集集合进行排序'__all__'。
  示例：  ordering_fields = '__all__'
```

### 指定默认排序

视图上设置`ordering`属性，将用作默认排序，该`ordering`属性可以是字符串，也可以是字符串列表/元组。

通常，也可以通过`order_by`在初始查询集上进行设置来控制此操作

```python
class UserListView(generics.ListAPIView):
    ......
    filter_backends = [filters.OrderingFilter]
    ordering_fields = ['username', 'email']
    ordering = ['username']
```



## 4.自定义通用过滤器后端

```
继承`BaseFilterBackend`，并重写`.filter_queryset(self, request, queryset, view)`方法。该方法应返回一个经过过滤的新查询集。

除允许客户端执行搜索和过滤外，通用过滤器后端对于限制对任何给定请求或用户应可见的对象也很有用。
```

**例如**，您可能需要限制用户只能看到他们创建的对象:

```python
class IsOwnerFilterBackend(filters.BaseFilterBackend):
    """Filter that only allows users to see their own objects."""
    
    def filter_queryset(self, request, queryset, view):
        return queryset.filter(owner=request.user)
```

```
可以通过覆盖`get_queryset()`视图来实现相同的行为，但是使用过滤器后端可以使您更轻松地将此限制添加到多个视图，或者将其应用于整个API。
```



# 六，分页

## 1.设置分页样式

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    # "next": "http://127.0.0.1:8001/api-auth/snippet/?limit=2&offset=2"
    
    # 'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination',
    # "next": "http://127.0.0.1:8001/api-auth/snippet/?page=2"
    'PAGE_SIZE': 100
}
```

可以使用`pagination_class`属性在单个视图上设置分页类：

```
class BillingRecordsView(generics.ListAPIView):
    ......
    pagination_class = rest_framework.pagination.PageNumberPagination

通常，可能希望在整个API中使用相同的分页样式，尽管您可能希望根据每个视图改变分页的各个方面，例如默认或最大页面大小。

```

## 2.修改分页样式

如果要修改分页样式的特定方面，则需要覆盖其中一个分页类，并设置要更改的属性。

```
class LargeResultsSetPagination(PageNumberPagination):
    page_size = 1000
    page_size_query_param = 'page_size'
    max_page_size = 10000

```

然后，您可以使用`pagination_class`属性将新样式应用于视图或全局配置。

## 3.自定义分页样式

```
继承`pagination.BasePagination`并覆盖`paginate_queryset(self, queryset, request, view=None)`和`get_paginated_response(self, data)`方法：

```

- 该`paginate_queryset`方法将传递给初始查询集，并且应返回仅包含所请求页面中数据的可迭代对象。
- 该`get_paginated_response`方法将传递序列化的页面数据，并且应返回一个`Response`实例。

注意，该`paginate_queryset`方法可以在分页实例上设置状态，以后可以由该`get_paginated_response`方法使用。

```python
class CustomPagination(pagination.PageNumberPagination):
    def get_paginated_response(self, data):
        return Response({
            'links': {
                'next': self.get_next_link(),
                'previous': self.get_previous_link()
            },
            'count': self.page.paginator.count,
            'results': data
        })
```

然后，我们需要在配置中设置自定义类：

```python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'my_project.apps.core.pagination.CustomPagination',
    'PAGE_SIZE': 100
}
```



# 七，drf 缓存

### 1-安装

pip install drf-extensions

### 2-配置

在settings里面增加两项配置

```
# drf扩展
REST_FRAMEWORK_EXTENSIONS = {
    # 缓存时间
    "DEFAULT_CACHE_RESPONSE_TIMEOUT": 60 * 60,
    # 使用缓存配置（default是settings里面配置好的caches里面的一项配置）
    "DEFAULT_USER_CACHE": "default"
}

```

### 3-使用

- **法1：使用cache_response装饰器**

  ```python
  from rest_framework_extensions.cache.decorators import cache_response
  
  class CityView(views.APIView):
      @cache_response(timeout=60*60, cache='default') # 缓存使用（即CACHES配置中的键名称）
      def get(self, request, *args, **kwargs):
          pass
  ```

  如果在使用cache_response装饰器时未指明timeout或者cache参数，则会使用配置文件中的默认配置，可以通过在settings.py中进行全局方法配置。

  **注意，cache_response装饰器既可以装饰在类视图中的get方法上，也可以装饰在REST framework扩展类提供的list或retrieve方法上。使用cache_response装饰器无需使用method_decorator进行转换。**

- **法2：使用扩展类Mixin(前提使用了视图集ViewSet)**

在需要缓存的视图类，继承CacheResponseMixin

```python
from rest_framework_extensions.cache.mixins import CacheResponseMixin

#继承顺序一定在ViewSet前,其实必须在对应的mixin前
class Views(CacheResponseMixin,ReadOnlyModelViewSet): 
　　"""描述"""
　　pass
```

**ListCacheResponseMixin**:用于缓存返回列表数据的视图，与ListModelMixin扩展类配合使用，实际是为list方法添加了cache_response装饰器

 **RetrieveCacheResponseMixin**:用于缓存返回单一数据的视图，与RetrieveModelMixin扩展类配合使用，实际是为retrieve方法添加了cache_response装饰器 

**CacheResponseMixin**:为视图集同时补充List和Retrieve两种缓存，与ListModelMixin和RetrieveModelMixin一起配合使用。

# Caching errors

默认情况,请求遇到错误时候,下一次还是会返回缓存的错误,例如:

```python
class CityView(views.APIView):
    @cache_response()
    def get(self, request, *args, **kwargs):
        raise Exception("500 error comes from here")1234
```

需要改变这种状况时,可以在装饰器中指明`cache_errors=False`

```python
class CityView(views.APIView):
    @cache_response(cache_errors=False)
    def get(self, request, *args, **kwargs):
        raise Exception("500 error comes from here")1234
```

也可以修改`DEFAULT_CACHE_ERRORS`的设置

```python
REST_FRAMEWORK_EXTENSIONS = {
    'DEFAULT_CACHE_ERRORS': False
}123
```

   

