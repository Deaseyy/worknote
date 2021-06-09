# 

当我们的接口抛出异常时，也需要返回json格式的信息到前端

```python
from flask import request, json
from werkzeug.exceptions import HTTPException

from app.utils.response_code import RET


class APIException(HTTPException):
    """定义一个API异常类，用来处理已知异常，返回json格式异常信息"""
    code = 500
    msg = 'A server error occurred!!!'
    error_code = 500

    def __init__(self, code=None, msg=None, error_code=None):
        if code:
            self.code = code
        if msg:
            self.msg = msg
        if error_code:
            self.error_code = error_code
        super().__init__(description=msg)

    def get_body(self, environ=None):
        body = dict(
            msg=self.msg,
            error_code=self.error_code,
            request=request.method + ' ' + self.get_url_no_param()
        )
        text = json.dumps(body)
        return text

    def get_headers(self, environ=None):
        """返回json格式：Get a list of headers."""
        return [('Content-Type', 'application/json')]

    @staticmethod
    def get_url_no_param():
        full_path = str(request.full_path)
        main_path = full_path.split('?')
        return main_path[0]


# 自定义已知异常类
class ParamException(APIException):
    """请求参数异常"""
    code = 400
    msg = 'invalid parameter'
    error_code = RET.PARAMERR


class PermissionDenied(APIException):
    """没有授权"""
    code = 403
    msg = 'You do not have permission to perform this action.'
    error_code = RET.FORBIDDEN
```



注意：

这种抛出的异常，对于普通flask的api 和 flask-restful的api有所不同：

```python
# 普通api
@api.route(methods=["GET"], rule="/test")
def aaa(self):
    x = request.args.get('x')
    if x =='1':
        raise ParamException(msg='参数错误')
        return jsonify('ok')

返回数据格式形如：{'error_code':4001, 'msg':'参数错误', 'request':'GET /test'}
      

#  flask-restful
rest_api.add_resource(to_maintain_views.AAA, "/test", )
class AAA(Resource):
    def get(self):
        x = request.args.get('x')
        if x =='1':
            raise ParamException(msg='参数错误')
        return jsonify('ok')
    
返回数据格式形如：{"message": "\u53c2\u6570\u9519\u8bef"} ，对'参数错误'进行了编码
前端可以直接取值 message，对应的值就是：'参数错误'
结果对异常信息 msg 进行了编码
```

