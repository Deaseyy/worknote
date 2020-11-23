# session中间件

`django.contrib.sessions.middleware.SessionMiddleware`

```python
class SessionMiddleware(MiddlewareMixin):
    def __init__(self, get_response=None):
        self.get_response = get_response
        engine = import_module(settings.SESSION_ENGINE)
        self.SessionStore = engine.SessionStore

    def process_request(self, request):
        # 从请求的cookie中获取 sessionid 对应的 session_key（即数据库session表保存的键）
        session_key = request.COOKIES.get(settings.SESSION_COOKIE_NAME)
        # 将session_key放入SessionStore初始化，获取保存在session_key中的信息设置到该次请求的session对象
        request.session = self.SessionStore(session_key)

    def process_response(self, request, response):
        """
        If request.session was modified, or if the configuration is to save the
        session every time, save the changes and set a session cookie or delete
        the session cookie if the session has been emptied.
        """
        try:
            # 该次请求视图中使用了request.session['userid'] = user.id等，
            # accessed，modified值会为True，empty 则会为False
            accessed = request.session.accessed  # 是否存取
            modified = request.session.modified  # 是否修改，设置或删除时为True
            # Returns True when there is no session_key and the session is empty
            empty = request.session.is_empty()   # 即原本就为空，本次请求也无设置
        except AttributeError:
            pass
        else:
            # First check if we need to delete this cookie.
            # The session should be deleted only if the session is entirely empty
            if settings.SESSION_COOKIE_NAME in request.COOKIES and empty:
                response.delete_cookie(
                    settings.SESSION_COOKIE_NAME,
                    path=settings.SESSION_COOKIE_PATH,
                    domain=settings.SESSION_COOKIE_DOMAIN,
                )
            else:
                if accessed:
                    patch_vary_headers(response, ('Cookie',))
                if (modified or settings.SESSION_SAVE_EVERY_REQUEST) and not empty:
                    #在浏览器关闭的时候 session 的有效期设 None ， 过期时间设 None
                    if request.session.get_expire_at_browser_close():
                        max_age = None
                        expires = None
                    else:
                        # 获取设置的session过期时间 request.session.set_expiry(1111)
                        max_age = request.session.get_expiry_age()
                        expires_time = time.time() + max_age
                        expires = cookie_date(expires_time)
                    # Save the session data and refresh the client cookie.
                    # Skip session save for 500 responses, refs #3881.
                    if response.status_code != 500:
                        try:
                            # 将session保存到数据库
                            # session值存到session_key，session_key不存在则创建再保存
                            request.session.save()
                        except UpdateError:
                            raise SuspiciousOperation(
                                "The request's session was deleted before the "
                                "request completed. The user may have logged "
                                "out in a concurrent request, for example."
                            )
                        response.set_cookie(
                            settings.SESSION_COOKIE_NAME, # 'sessionid'
                            # 将session_key设置到cookie，及有效时间
                            request.session.session_key, max_age=max_age,
                            expires=expires, domain=settings.SESSION_COOKIE_DOMAIN,
                            path=settings.SESSION_COOKIE_PATH,
                            secure=settings.SESSION_COOKIE_SECURE or None,
                            httponly=settings.SESSION_COOKIE_HTTPONLY or None,
                        )
        return response
```





待续。。。。。。





