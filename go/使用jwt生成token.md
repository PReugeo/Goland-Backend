## 使用jwt生成token

### JWT是什么

Json web token (JWT), 是为了在网络应用环境间传递声明而执行的一种基于JSON的开放标准（[(RFC 7519](https://link.jianshu.com/?t=https://tools.ietf.org/html/rfc7519)).该token被设计为紧凑且安全的，特别适用于分布式站点的单点登录（SSO）场景。JWT的声明一般被用来在身份提供者和服务提供者间传递被认证的用户身份信息，以便于从资源服务器获取资源，也可以增加一些额外的其它业务逻辑所必须的声明信息，该token也可直接被用于认证，也可被加密。

### JWT组成部分

1. Header(默认标识, 和加密算法)

2. Claims(载荷)

    ```go
    Audience  string `json:"aud,omitempty"` //token接收者
    ExpiresAt int64  `json:"exp,omitempty"`	//过期时间
    Id        string `json:"jti,omitempty"`	//自定义id号
    IssuedAt  int64  `json:"iat,omitempty"`	//签名发行时间
    Issuer    string `json:"iss,omitempty"`	//签名发行者
    NotBefore int64  `json:"nbf,omitempty"`	//token信息生效时间
    Subject   string `json:"sub,omitempty"` //签名面向的用户
    ```

3. Signature(加密签名)



### go中应用

使用到的开源库

```go
go get github.com/urfave/negroni  //web中间件
go get github.com/dgrijalva/jwt-go
go get github.com/dgrijalva/jwt-go/request
```

生成token代码(根据[Wangjiaxing123](https://github.com/Wangjiaxing123)/**JwtDemo**  修改)

```go
//定义载荷
type CustomClaims struct {
	Username string  `json:"username"` //1
	ExpireTime int64 `json:"expire_time"` //2这些可以自己随意设置
	jwt.StandardClaims //过期时间等初始化定义在这个结构体中
}

type JWT struct {
	SigningKey []byte
}

//初始化设置sign
func NewJWT() *JWT {
	return &JWT{
		[]byte(GetSignKey()),
	}
}

//根据载荷创建一个token
func (j *JWT) CreatToken(claims CustomClaims) (string, error) {
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(j.SigningKey)
}
```

验证token 

```go
//错误信息常量
var (
	TokenExpired     = errors.New("Token is expired")
	TokenNotValidYet = errors.New("Token not active yet")
	TokenMalformed   = errors.New("That's not even a token")
	TokenInvalid     = errors.New("Couldn't handle this token:")
	SignKey          = "lottery"
)

//解析token
func (j *JWT) ParseToken(tokenString string) (*CustomClaims, error) {
    //根据jwt-go库代码解析token
	token, err := jwt.ParseWithClaims(tokenString, &CustomClaims{}, func(token *jwt.Token) (i interface{}, e error) {
		return j.SigningKey, nil
	})
	//输出错误信息
	if err != nil {
		if ve, ok := err.(*jwt.ValidationError); ok {
			if ve.Errors & jwt.ValidationErrorMalformed != 0 {
	return nil, TokenMalformed
			}else if ve.Errors & jwt.ValidationErrorExpired != 0 {
				return nil, TokenExpired
			}else if ve.Errors & jwt.ValidationErrorNotValidYet != 0 {
				return nil, TokenNotValidYet
			}else {
				return nil, TokenInvalid
			}
		}
	}
	//如果验证成功则返回载体
	if Claims, ok := token.Claims.(*CustomClaims); ok && token.Valid {
		return Claims, nil
	}
	return nil, TokenInvalid
}
```

