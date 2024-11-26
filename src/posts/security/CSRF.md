---
date: 2024-11-26
category:
  - 安全
tag:
  - 安全
excerpt: CSRF 简介
order: 99
---

# CSRF（Cross-site request forgery，跨站请求伪造）

CSRF 利用的是网站对用户网页浏览器的信任，攻击者通过某些技术手段，欺骗用户的浏览器去访问一个认证过的网站并执行一些操作

- CSRF 并不能直接获取用户的账户控制权和信息，他能做是欺骗用户的浏览器，让其以用户的名义执行操作

例如在恶意网站在某个图片的地址中，加入跳转到已认证网站的链接并附带恶意参数

```html
<img src="https://www.example.com/index.php?action=delete&id=123" />
```

CSRF 利用的是自动的身份验证，浏览器会随着所有请求发送 Cookie，如果不使用 Cookie，也不依赖 Cookie 进行身份验证，那么 CSRF 攻击也就不存在了，只要身份验证不是自动的即可

Spring Security 默认是开启 CSRF 防护的，用户登录时会发放一个 CSRF Token，之后的每个请求过来都需附带一个 CSRF Token，当然更推荐的是关闭 CSRF 防护，转而使用无状态的 JWT

另外别忘了验证请求的合法性，例如使用数字签名、时间戳等。请求头中有个 Referer 字段，表示当前请求的来源页面，从 A 网站访问 B 网站的页面，Referer 指向的就是 A，后端可通过此字段过滤不合法的请求

## 参考

- [Should I use CSRF protection on Rest API endpoints?](https://security.stackexchange.com/questions/166724/should-i-use-csrf-protection-on-rest-api-endpoints/166798)
- [Do I need CSRF token if I'm using Bearer JWT?](https://security.stackexchange.com/questions/170388/do-i-need-csrf-token-if-im-using-bearer-jwt)
- [跨站请求伪造](https://zh.wikipedia.org/wiki/%E8%B7%A8%E7%AB%99%E8%AF%B7%E6%B1%82%E4%BC%AA%E9%80%A0)
- [跨站请求伪造](https://developer.mozilla.org/zh-CN/docs/Glossary/CSRF)
- [SpringSecurity（10）——Csrf防护](https://blog.csdn.net/peng_gx/article/details/135714325)
- [CSRF 详解：攻击，防御，Spring Security应用等](https://www.cnblogs.com/pengdai/p/12164754.html)
