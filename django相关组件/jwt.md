##### jwt_decode_handler      jwt 解密

##### jwt_encode_handler      

##### jwt_payload_handler    



## 使用jwt生成token

```python
# 生成带有用户信息的载荷payload
payload = jwt_payload_handler(user)  #  传入用户对象

# 将带有用户信息的payload加密，生成token
token = jwt_encode_handler(payload)

# 前端携带token访问时验证token信息
payload = jwt_decode_handler(token) # payload为带有用户信息的字典
```

例子：

```python
def login_user(self, validate_attr):
    jwt_payload_handler = api_settings.JWT_PAYLOAD_HANDLER
    jwt_encode_handler = api_settings.JWT_ENCODE_HANDLER
    user_account = validate_attr.get('user_account')
    password = validate_attr.get('password')
    username_en = validate_attr.get('username_en')
    user = User(user_account, password)
    user.username = user_account
    user.username_en = username_en
    payload = self.jwt_payload_handler(user)
    token = jwt_encode_handler(payload)
    return token
```









