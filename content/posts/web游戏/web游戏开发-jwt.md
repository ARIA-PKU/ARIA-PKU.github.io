---
title: Web游戏开发 Jwt
date: 2022-05-29T16:49:35+08:00
lastmod: 2022-05-29T16:49:35+08:00
cover: https://oss.surfaroundtheworld.top/blog-pictures/reverse_world.jpg
categories:
  - web游戏开发
tags:
  - JWT
  - REST
# nolastmod: true
draft: true
---

Jwt的介绍以及在django中如何实现Jwt的部署

<!--more-->

JWT全称Json Web Token。是跨域身份认证的一种方式。JWT的声明一般是用来身份提供者与服务提供者之间传递被认证的用户身份信息。便于从服务器获取到资源。它也可以用来记录用户信息。与token不同的是，它不用查询数据库，只要在服务端使用密钥完成校验就行。

# 跨域身份验证问题的解决

跨域身份验证问题一般有两种常用方式，一是通过session验证，二是通过JWT验证。

## session实现方式

1. 浏览器发起请求登陆，向服务器发送用户名和密码
2. 服务端验证身份，生成身份验证信息（用户角色、登录时间等），存储在服务端 的 session 中
3. 服务器向用户返回 session_id，session_id 信息都会写入到用户的 Cookie中，这里一般考虑到安全问题，会设置成http-only防止被恶意的js调用。
4. 用户的每个后续请求都将通过在 Cookie 中取出 session_id 传给服务器
5. 服务器收到 session_id 并对比之前保存的数据，确认用户的身份
6. 服务器返回用户请求的内容

## Jwt的实现方式

以下是在网上找到的流程图：

![在这里插入图片描述](..\..\..\static\pics\jwt1.png)

1. 浏览器发起请求登陆
2. 服务端验证身份，根据算法，将用户标识符打包生成 JWT, 并且返回给浏览器，这里一般存放到localStorage中
3. 浏览器发起请求获取用户资料，把刚刚拿到的 JWT 一起发送给服务器
4. 服务器发现数据中有 JWT，进行验证
5. 服务器返回用户请求的内容

## 两种方式的比较

1. 如果前端使用ajax调用，由于cookie被设置为http-only所以无法获得session_id进行验证
2. session 存储在服务端占用服务器资源，而 JWT 存储在客户端
3. session 存储在 Cookie 中，存在伪造跨站请求伪造攻击的风险，而 JWT 则通过参数传递
4. session 只存在一台服务器上，那么下次请求就必须请求这台服务器，不利于分布式应用
5. 存储在客户端的 JWT 比存储在服务端的 session 更具有扩展性

# JWT的构成

jwt是由三部分组成，分别是header，payload和signature这三部分。

**Header**
jwt的头部承载着两部分信息。一是声明类型，这里是jwt，二是声明加密的算法，通常就使用HMAC和SHA256，例如这样的一个完整的JSON。

```
{
     'typ': 'jwt',
     'alg': 'HS256'
}
```


对它的头部进行base64加密。就构成了jwt的第一部分:

`ewogICAgICd0eXAnOiAnand0JywKICAgICAnYWxnJzogJ0hTMjU2Jwp9`

**Payload**

这一部分就是存放有效信息的地方。这些有效信息包含三个部分。

标准注册的声明，公共的声明和私有的声明。标准注册的声明并不强制使用。

标准注册的声明有

```
iss: jwt签发者
sub: jwt所面向的用户
aud: 接收jwt的一方
exp: jwt的过期时间，这个过期时间必须要大于签发时间
nbf: 定义在什么时间之前，该jwt都是不可用的.
iat: jwt的签发时间
jti: jwt的唯一身份标识，主要用来作为一次性token,从而回避重放攻击。
```


公共的声明是可以添加任何信息的，一般添加用户的相关信息或者其他业务需要的必要信息。这一部分客户端是可见的，所以**不建议存放敏感信息**。

私有声明是提供者和消费者所共同定义的声明，不建议存放敏感信息。用base64加密等于明文。

举例一个payload:

```
{
  "sub": "1234567890",
  "name": "XiLitter",
  "admin": true
}
```


同样对payload这一部分进行base64加密:

`ewogICJzdWIiOiAiMTIzNDU2Nzg5MCIsCiAgIm5hbWUiOiAiWGlMaXR0ZXIiLAogICJhZG1pbiI6IHRydWUKfQ==`

还是那句话，payload这一部分只进行base64加密，里面的信息对于任何人都是可见的。所以尽量别把敏感信息放在这个部分。

**signature**

这一部分是对前两部分的签名，防止数据被纂改。

首先是需要这个密钥的，这个密钥只有服务器知道，不会泄露给用户。然后就使用header头里的签名算法生成签名。相关公式：

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```


默认采用的签名算法是HMAC SHA256.产生签名后，这三部分就用点(.)将这三部分连接起来，就组成了完整的jwt。这里的secret是服务器端的密钥，只有服务器端知道（如果泄露则会毫无安全性)。

实际的jwt(在百度图片上找到的炫酷图片）:

![在这里插入图片描述](..\..\..\static\pics\jwt2.jpg)



# JWT的安全问题

**敏感信息泄露**

上面也说了，jwt的payload是任何人可见的，所以如果这一部分添加有用户的敏感性息，那就会造成敏感信息泄露，（比如账号密码）很好理解。

**将加密算法修改为none**

其实jwt是支持将算法修改为none的，这样的话，jwt中的签名就会被清空，所以就没有什么安全性可言了。任何token都是可以的，最初由这样的设定是为了方便调试。所以一定要在真实环境中关闭该功能，不然的话，攻击者就可以伪造任意的token来进行任意用户登录网站。

**密钥混合攻击**

那就要用到JWT两种最常见的算法。HMAC和RSA。HMAC是对称加密算法，使用同一密钥对token进行签名和认证。而RSA是非对称加密算法，它是需要两个密钥，用私钥对token进行加密生成jwt，再用公钥进行认证。基础知识了解，再说说攻击思路：我们可以伪造解密的算法，由RS256改为HS256，那么被用于HS256解密的密钥就变成了RS256的公钥。为什么要这么做？HS256加密解密都用一个密钥，所以安全起见，服务端是不会让你知道HS356的密钥的，而RS256是由服务端用私钥加密，再用公钥解密，所以私钥用户是得不到的，而公钥无所谓得到，因为没有私钥加密，就伪造不了token。一旦我们更改解密算法，我们是可以获取到RS256的公钥，又因为改算法为HS256，它会将RS256的公钥作为密钥，那么不言而喻，我们可以伪造任意的token了。这种攻击的预防也很容易，**禁止同时使用这两种算法就行**。

# Django中的JWT实现

django中的simple-jwt的实现是有两个token的，分别是access token和refresh token。在验证权限的时候，会验证access token，如果过期，会通过refresh token申请新的access token，如果refresh token也过期，则需要重新登录。这里的access token和refresh token的时长可以在django的settings文件中配置，一般access token的有效期会较短，refresh token有效期会较长。

这样实现的原因在于，access token会通过get请求频繁地暴露出来，有效时间过长则安全性较低，需要短时间内刷新；refresh token请求access token的过程使用的是post请求，较为安全。

## 配置安装

因为django中jwt被集成到了restframework中，所以需要一同部署

```
pip install djangorestframework
pip install pyjwt  # 解码使用
pip install djangorestframework-simplejwt  # 最终版，另一个djangorestframework-jwt不维护了
```

在每次安装django相关的插件，都需要将其添加到`settings.py`文件中`INSTALLED_APPS`中：

```
INSTALLED_APPS = [
    ...
    'rest_framework',
    'rest_framework_simplejwt',
    ...
]
```

将默认的验证方式改为jwt：

```
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    )
}
```

增加jwt的配置文件(在simple-jwt官网有参数详细信息：https://django-rest-framework-simplejwt.readthedocs.io/en/latest/）：

```
from datetime import timedelta

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=5),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=14),
    'ROTATE_REFRESH_TOKENS': False,
    'BLACKLIST_AFTER_ROTATION': False,  # 黑名单，可以实现显式退出jwt
    'UPDATE_LAST_LOGIN': False,

    'ALGORITHM': 'HS256',
    'SIGNING_KEY': 'fasdlkfjalsd', // 这个就是上文介绍的服务器端的secret,自定义随机值并定期更换
    'VERIFYING_KEY': None,
    'AUDIENCE': None,
    'ISSUER': None,
    'JWK_URL': None,
    'LEEWAY': 0,

    'AUTH_HEADER_TYPES': ('Bearer',),
    'AUTH_HEADER_NAME': 'HTTP_AUTHORIZATION',
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',
    'USER_AUTHENTICATION_RULE': 'rest_framework_simplejwt.authentication.default_user_authentication_rule',

    'AUTH_TOKEN_CLASSES': ('rest_framework_simplejwt.tokens.AccessToken',),
    'TOKEN_TYPE_CLAIM': 'token_type',
    'TOKEN_USER_CLASS': 'rest_framework_simplejwt.models.TokenUser',

    'JTI_CLAIM': 'jti',

    'SLIDING_TOKEN_REFRESH_EXP_CLAIM': 'refresh_exp',
    'SLIDING_TOKEN_LIFETIME': timedelta(minutes=5),
    'SLIDING_TOKEN_REFRESH_LIFETIME': timedelta(days=1),
}
```

**注意：**安装完app，要`python3 manage.py collectstatic`将安装的app同步过来。

## 添加路由

将获取acess和refresh的token的路由添加进来：

```
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
)

urlpatterns = [
    ...
    # 类的方法需要使用as_view()进行转换
    path('token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    ...
]
```

然后就能通过路由访问到token的页面，这里是restframework包中实现好的页面。

在jwt官网：https://jwt.io/就能解码内容，进行验证查看结果。

## REST框架

###后端部分

官方给出的例子：

```
from rest_framework.views import APIView
from rest_framework.response import Response

class SnippetDetail(APIView):
    def get(self, request):  # 查找
        ...
        return Response(...)

    def post(self, request):  # 创建
        ...
        return Response(...)

    def put(self, request, pk):  # 修改
        ...
        return Response(...)

    def delete(self, request, pk):  # 删除
        ...
        return Response(...)
```

根据格式修改自己的代码：

```
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from game.models.player.player import Player

class InfoView(APIView):
    permission_classes = ([IsAuthenticated])  # 验证权限  

    def get(self, request):
        user = request.user
        player = Player.objects.get(user=user)
        return Response({
            'result': "success",
            'username': user.username,
            'photo': player.photo,
        })

```

**手动获取jwt**

    from rest_framework_simplejwt.tokens import RefreshToken
    
    def get_tokens_for_user(user):
        refresh = RefreshToken.for_user(user)
    
    return {
        'refresh': str(refresh),
        'access': str(refresh.access_token),
    }
### 前端部分

前端的一般会存放到localStoage中，按照前面介绍的逻辑进行验证，登录和刷新。

这里先实现只存放到内存中，设置周期函数，定期刷新access token。

```
 getinfo() {
        $.ajax({
            url: "https://bloodborne.shop/settings/getinfo/",
            type: "get",
            data: {
                platform: this.platform,
            },
            headers: {  # 配合后端的permission_classes验证需要加这个，字段也是在settings中设定的
                "Authorization": "Bearer " + this.root.access,
            },
            success: resp => {
                console.log(resp);
                if (resp.result === "success") {
                    this.username = resp.username;
                    this.photo = resp.photo;
                    this.hide();
                    this.root.menu.show();
                } else {
                    this.login();
                }
            }
        });
    }
```

### WebSocket端的验证

将jwt集成到django_channels中，

首先，创建文件`channelsmiddleware.py`

```
"""General web socket middlewares
"""

from channels.db import database_sync_to_async
from django.contrib.auth import get_user_model
from django.contrib.auth.models import AnonymousUser
from rest_framework_simplejwt.exceptions import InvalidToken, TokenError
from rest_framework_simplejwt.tokens import UntypedToken
from rest_framework_simplejwt.authentication import JWTTokenUserAuthentication
from channels.middleware import BaseMiddleware
from channels.auth import AuthMiddlewareStack
from django.db import close_old_connections
from urllib.parse import parse_qs
from jwt import decode as jwt_decode
from django.conf import settings
@database_sync_to_async
def get_user(validated_token):
    try:
        user = get_user_model().objects.get(id=validated_token["user_id"])
        # return get_user_model().objects.get(id=toke_id)
        return user

    except:
        return AnonymousUser()



class JwtAuthMiddleware(BaseMiddleware):
    def __init__(self, inner):
        self.inner = inner

    async def __call__(self, scope, receive, send):
       # Close old database connections to prevent usage of timed out connections
        close_old_connections()

        # Try to authenticate the user
        try:
            # Get the token
            token = parse_qs(scope["query_string"].decode("utf8"))["token"][0]

            # This will automatically validate the token and raise an error if token is invalid
            UntypedToken(token)
        except:
            # Token is invalid

            scope["user"] = AnonymousUser()
        else:
            #  Then token is valid, decode it
            decoded_data = jwt_decode(token, settings.SIMPLE_JWT["SIGNING_KEY"], algorithms=["HS256"])

            # Get the user using ID
            scope["user"] = await get_user(validated_token=decoded_data)
        return await super().__call__(scope, receive, send)


def JwtAuthMiddlewareStack(inner):
    return JwtAuthMiddleware(AuthMiddlewareStack(inner))
```

然后，修改`asgi.py`文件，将验证方式改为上面的中间件：

```
from game.channelsmiddleware import JwtAuthMiddlewareStack

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": JwtAuthMiddlewareStack(URLRouter(websocket_urlpatterns))
})
```

最后，要在新建socket连接的时候，在连接中加上token字段：

```
this.ws = new WebSocket("wss://bloodborne.shop/wss/multiplayer/?token=" + playground.root.access);
```



