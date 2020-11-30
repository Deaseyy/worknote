# BaseHandler

```python
class BaseHandler:
    _view_middleware = None # view中间件
    _template_response_middleware = None  # template_response中间件
    _exception_middleware = None  # exception中间件
    _middleware_chain = None

    def load_middleware(self):
        """WSGIHandler类在初始化时,从settings.MIDDLEWARE加载中间件列表；"""
        # 这里只加载以下三种，请求和响应中间件最下面有其他机制。
        self._view_middleware = []
        self._template_response_middleware = []
        self._exception_middleware = []
        
        # 将_get_response函数包装在异常转换装饰器中：捕获请求流程中的异常，将其转换成合适的响应
        # handler相当于在_get_response加了一层处理响应异常的逻辑
        handler = convert_exception_to_response(self._get_response)
        # 遍历settings配置文件中的中间件：从最后一个开始
        for middleware_path in reversed(settings.MIDDLEWARE):
            middleware = import_string(middleware_path) # 导入该中间件类
            try:
                mw_instance = middleware(handler) # 中间件实例化
            except MiddlewareNotUsed as exc:
                if settings.DEBUG:
                    if str(exc):
                        logger.debug('MiddlewareNotUsed(%r): %s', middleware_path, exc)
                    else:
                        logger.debug('MiddlewareNotUsed: %r', middleware_path)
                continue

            if mw_instance is None:
                raise ImproperlyConfigured(
                    'Middleware factory %s returned None.' % middleware_path
                )
				
            if hasattr(mw_instance, 'process_view'):
                # 加载中间件类的 process_view
                self._view_middleware.insert(0, mw_instance.process_view)
            if hasattr(mw_instance, 'process_template_response'):
                # 加载中间件类的 process_template_response
                self._template_response_middleware.append(mw_instance.process_template_response)	         
            if hasattr(mw_instance, 'process_exception'):
                # 加载中间件类的 process_exception
                self._exception_middleware.append(mw_instance.process_exception)
			
            # mw_instance实现了__call__方法，相当于加了一层逻辑的get_response
            # 这里对_request_middleware和_response_middleware进行了处理。
            handler = convert_exception_to_response(mw_instance)

        # 只在初始化完成时对它进行赋值，因为它被用作初始化完成的标志。
        # _middleware_chain是一个修饰器函数，内层就是_get_response，外层则是那些中间件逻辑
        self._middleware_chain = handler
    
    def get_response(self, request):
        """为传入的HttpRequest返回一个HttpResponse对象
        重点：该函数是请求传入到返回响应的整个逻辑框架
        """
        # 为这个线程设置默认的url解析器
        set_urlconf(settings.ROOT_URLCONF)
        
        # 核心代码块！！！！！！！！！！！！！！！！！
		# 调用包装后的_get_response()
        response = self._middleware_chain(request)

        response._closable_objects.append(request)
        # 如果异常处理程序返回尚未渲染的TemplateResponse，则强制渲染它。
        if not getattr(response, 'is_rendered', True) and callable(getattr(response, 'render', None)):
            response = response.render()

        if response.status_code >= 400:
            log_response(
                '%s: %s', response.reason_phrase, request.path,
                response=response,
                request=request,
            )
        return response
    
    def _get_response(self, request):
        """解析并调用视图，然后应用view、exception和template_response中间件。
        此方法是在 request/response中间件内发生的所有事情"""
        response = None
		
        # 获取url解析器
        if hasattr(request, 'urlconf'):
            urlconf = request.urlconf
            set_urlconf(urlconf)
            resolver = get_resolver(urlconf)
        else:
            resolver = get_resolver()
		
        # 获取url匹配的视图函数；以命名元组类型返回
        resolver_match = resolver.resolve(request.path_info)
        # (视图函数，位置参数，关键字参数)
        callback, callback_args, callback_kwargs = resolver_match
        request.resolver_match = resolver_match

        """1.使用视图中间件 view middleware。"""
        # 执行process_view
        for middleware_method in self._view_middleware:
            response = middleware_method(request, callback, callback_args, callback_kwargs)
            # 若process_view返回response对象，则后面中间件的process_view不再执行
            if response:
                break
		
        if response is None:
            # 将视图函数原子化
            wrapped_callback = self.make_view_atomic(callback)
            try:
                # 执行视图函数 ！！！！！！！！！！！！！！！！！
                response = wrapped_callback(request, *callback_args, **callback_kwargs)
            except Exception as e:
                """2.使用异常处理中间件 exception middleware"""
                # 执行出错，执行process_exception处理异常
                response = self.process_exception_by_middleware(e, request)

        # 如果视图返回None就会报错(常见的错误)。
        if response is None:
            if isinstance(callback, types.FunctionType):    # FBV
                view_name = callback.__name__
            else:                                           # CBV
                view_name = callback.__class__.__name__ + '.__call__'

            raise ValueError(
                "The view %s.%s didn't return an HttpResponse object. It "
                "returned None instead." % (callback.__module__, view_name)
            )
            
		"""3.使用模板响应中间件 template response middleware。""" 
        # 如果响应支持延迟渲染，则执行process_template_response，然后呈现响应。
        elif hasattr(response, 'render') and callable(response.render):
            for middleware_method in self._template_response_middleware:
                response = middleware_method(request, response)
                # 如果模板响应中间件返回None就会报错(一种常见错误)
                if response is None:
                    raise ValueError(
                        "%s.process_template_response didn't return an "
                        "HttpResponse object. It returned None instead."
                        % (middleware_method.__self__.__class__.__name__)
                    )

            try:
                # 结果渲染！！！！！！！！！！！！！！！！
                response = response.render()
            except Exception as e:
                response = self.process_exception_by_middleware(e, request)

        return response
    
    def make_view_atomic(self, view):
        """使视图原子化"""
        non_atomic_requests = getattr(view, '_non_atomic_requests', set())
        for db in connections.all():
            if db.settings_dict['ATOMIC_REQUESTS'] and db.alias not in non_atomic_requests:
                view = transaction.atomic(using=db.alias)(view)
        return view
    
    def process_exception_by_middleware(self, exception, request):
        """将异常传递给异常中间件。如果没有中间件为此异常返回响应，则引发该异常。"""
        for middleware_method in self._exception_middleware:
            response = middleware_method(request, exception)
            if response:
                return response
        raise
```

可以看到`self.load_middleware()`是加载中间件的过程。

可以看到主循环是对配置文件里中间件进行逆序遍历，`_view_middleware`列表是执行的`insert(0)`操作，所以这个列表最后的顺序是和配置文件的顺序相同。而`_template_response_middleware`和`_exception_middleware`都是`append`的操作，所以最后生成的列表顺序是和配置文件的顺序相反。这里面比较特殊的是`_request_middleware`和`_response_middleware`，并没有使用其他3种中间件那样的列表，而是在`MiddlewareMixin`的`__call__`方法中将其和`get_response`进行了包裹，外层的中间件包装着里面的中间件，最里面的是`_get_response`。其实这两类中间件都会继承自`MiddlewareMixin`。



# WSGIHandler

Django应用最开始的入口处在`wsgi.py`，这里是从web服务器转发请求到Django应用的地方，同时也是Django应用返回响应给web服务器的地方

```python
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest # wsgi请求类

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # 从settings.MIDDLEWARE加载中间件列表，只加载三种
        self.load_middleware() 

    def __call__(self, environ, start_response):
        """"""
        set_script_prefix(get_script_name(environ))
        # 发送请求开始的signal
        signals.request_started.send(sender=self.__class__, environ=environ)
        # 根据web服务器的传参，创建wsgi请求对象（django中的request）
        request = self.request_class(environ)
        # 生成响应对象
        response = self.get_response(request)

        response._handler_class = self.__class__

        status = '%d %s' % (response.status_code, response.reason_phrase)
        response_headers = list(response.items())
        for c in response.cookies.values():
            response_headers.append(('Set-Cookie', c.output(header='')))
            
        # 执行wsgi协议的回调start_response，通过它来设置:响应状态码，响应头
        start_response(status, response_headers)
        if getattr(response, 'file_to_stream', None) is not None and environ.get('wsgi.file_wrapper'):
            response = environ['wsgi.file_wrapper'](response.file_to_stream)
        return response
```



# 相关工具

### MiddlewareMixin

```python
class MiddlewareMixin:
    def __init__(self, get_response=None):
        self.get_response = get_response
        super().__init__()
	
    """BaseHandler/load_middleware()方法中有调用：
    handler = convert_exception_to_response(mw_instance)
    """ 
    def __call__(self, request):
        """声明中间件实例是可调用的, 如：mw_instance();
        相当于装饰器，在get_response上又加了一层逻辑；
        理解其实例就是一个get_response函数，返回一个response；
        """
        response = None
        if hasattr(self, 'process_request'):
            response = self.process_request(request)
        response = response or self.get_response(request)
        if hasattr(self, 'process_response'):
            response = self.process_response(request, response)
        return response
```



### 异常转换装饰器

包装`_get_response`：捕获请求流程中的异常，将其转换成合适的响应。

```python
def convert_exception_to_response(get_response):
    """
    将给定的get_response可调用包装在异常到响应的转换中。

    所有异常都会被转换。所有已知的4xx异常(Http404，
    权限拒绝、多部分解析器错误、可疑操作)将
    转换为适当的响应，所有其他例外都将
    转换为500响应。

    此装饰器自动应用于所有中间件，以确保
    没有中间件泄漏异常，堆栈中的下一个中间件
    可以依赖得到响应而不是异常。
    """
    @wraps(get_response)
    def inner(request):
        try:
            response = get_response(request)
        except Exception as exc:
            response = response_for_exception(request, exc)
        return response
    return inner
```



# 源码学习总结

### 1.请求流程步骤

web服务器发送请求进来，接着由WSGIHandler来处理，初始化时，先将配置文件中的所有中间件，加载到内存列表，分为：request，response，view， exception，template_response；

接着使用WSGIRequest对请求进行封装，再调用get_response来返回响应对象。

```python
class WSGIHandler(base.BaseHandler):
    request_class = WSGIRequest

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # 加载中间件列表
        self.load_middleware()

    def __call__(self, environ, start_response):
       	# 发送请求开始处理信号
        signals.request_started.send(sender=self.__class__, environ=environ)
        # 生成wsgi请求对象
        request = self.request_class(environ)
        # 根据传入的请求获取响应对象
        response = self.get_response(request)
		......
        # 获取状态码和请求头
        status = '%d %s' % (response.status_code, response.reason_phrase)
        response_headers = list(response.items())
        ......
        # 请求头设置cookie
        response_headers.append(('Set-Cookie', c.output(header='')))
        # 执行wsgi协议的回调
        start_response(status, response_headers)
        # 返回响应体
        return response
```



### 2.请求处理的简单原型

从源码中可以看出，从请求进来到返回响应，简单来说，就是下面这个方法：

```python
def get_response(request):
    """为传入的HttpRequest返回一个HttpResponse对象"""
	return response
```

而`_get_response()`才是生成响应对象的关键方法， 它被层层包装，添加额外功能，封装中间件l列表；最后变成了`_middleware_chain`（其内部还是`_get_response()`），

```python
def get_response(self, request):
    # 调用包装了_get_response()后的_middleware_chain
    response = self._middleware_chain(request)
    return response
```



### 3.返回响应的核心方法

从`BaseHandler`源码可以看出，`_get_response()` 是返回响应对象的关键方法:

```python
def _get_response(self, request):
    """解析并调用视图，然后应用view、exception和template_response中间件。
    此方法是在 request/response中间件内发生的所有事情"""
    response = None
    # 1. 获取url解析器
    # 2. 获取url匹配的视图函数
    # 3. 遍历self._view_middleware列表，执行 process_view
    if response is None:
        try:
            # 4.执行视图函数
        except:
            # 5.执行 process_exception 处理异常
    if response is None:
        # 视图返回None就会报错(常见的错误)。
        
    # 6. 如果响应支持延迟渲染，则执行process_template_response
    try:
        # 7.结果渲染
    except:
        # 8.执行 process_exception 处理异常
    return response
```



#### `_get_response()`的包装过程

`_get_response()`经过了多层包装（添加额外功能，封装中间件l列表），每一步返回的其实都是包装后的`_get_response()` 。

```python
"""BaseHandler/load_middleware()""" 
# 1.异常转换装饰器包装
handler = convert_exception_to_response(self._get_response)
# 2.中间件类实例的__call__方法包装
mw_instance = middleware(handler)
# 3.异常转换装饰器 继续包装
handler = convert_exception_to_response(mw_instance)
# 4.赋值给WSGIHandler实例变量
self._middleware_chain = handler
```

从以上可以看出，`_middleware_chain`就是封装了中间件的`_get_response`方法。



# 其他收获

在源码中调试发现，csrf 校验使用的是`CorsMiddleware`中间件的`process_view`方法。

之前以为是process_request ...

