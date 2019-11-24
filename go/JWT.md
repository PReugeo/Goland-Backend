# JSON Web Tokens

## 概念

​	JWT是个开放的定义了一种紧凑且自包含的方式, 用于在各方之间作为JSON对象安全地传输信息的开放标准.

## 什么时候使用JWT

1.  Authorization(登录授权):　一旦用户登录，每个后续请求将包括JWT, 从而允许用户访问该令牌允许的域名, 服务和资源(route, services, resource), 普遍用于SSO(单点登录)
2.  信息交换: JSON Web令牌是在各方之间安全地传输信息的好方法。 因为可以对JWT进行签名（例如，使用公钥/私钥对），所以您可以确定发件人是他们所说的人。 另外，由于签名是使用标头和有效负载计算的，因此您还可以验证内容是否未被篡改。

## JWT结构

JWT由三部分组成, 这些部分有 . 分割, 分别为:

-   Header: 通常以两部分组成, 令牌类型(JWT)和签名所使用的算法

    -   ```json
        { "alg": "HS256", "typ": "JWT" }
        ```

-   Payload(有效载荷): 包含claims(声明), 通常是用户数据结构和其他数据的声明. 声明有三种类型: registered, public, private

    -   registered: 这是一组非强制性但是建议使用的预定义claims, 以提供一组有用的, 可互操作的claims, 可以包含: issuer(发出者), exp(到期时间), sub(主题), aud(受众)等.

    -   public: 可随意定义. 但是为避免冲突，应在[IANA JSON Web令牌注册表](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com&sl=auto&sp=nmt4&tl=zh-CN&u=https://www.iana.org/assignments/jwt/jwt.xhtml&xid=17259,1500003,15700022,15700186,15700190,15700256,15700259,15700262,15700265,15700271,15700283&usg=ALkJrhj7Nu6hC0lBtFF4eikwfwtEogQ0LA)中定义它们，或将其定义为包含抗冲突名称空间的URI。

    -   private: 这些是为在同意使用它们的各方之间共享信息而创建的自定义声明，既不是*注册*声明也不是*公共*声明。

    -   ```json
        { "sub": "1234567890", "name": "John Doe", "admin": true }
        ```

        

-   Signature(签名): 签名用于验证消息在整个过程中没有更改，并且对于使用私钥进行签名的令牌，它还可以验证JWT的发送者是它所说的真实身份。

## JWT如何工作

在身份验证中，当用户使用其凭据成功登录时，将返回JSON Web令牌。 由于令牌是凭据，因此必须格外小心以防止安全问题。 通常，令牌的保留时间不应超过要求的时间。

[由于缺乏安全性，](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com&sl=auto&sp=nmt4&tl=zh-CN&u=https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html&xid=17259,1500003,15700022,15700186,15700190,15700256,15700259,15700262,15700265,15700271,15700283&usg=ALkJrhi_7cvWHyn4UK2nU4IbU4yTL4WFxw#local-storage)您也不[应该将敏感的会话数据存储在浏览器中](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com&sl=auto&sp=nmt4&tl=zh-CN&u=https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html&xid=17259,1500003,15700022,15700186,15700190,15700256,15700259,15700262,15700265,15700271,15700283&usg=ALkJrhi_7cvWHyn4UK2nU4IbU4yTL4WFxw#local-storage) 。

每当用户想要访问受保护的路由或资源时，用户代理(前端)应该在授权头中使用Bearer模式发送JWT。 标头的内容应如下所示：

>   ```http
>   Authorization: Bearer <token>
>   ```

