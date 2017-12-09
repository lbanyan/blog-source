---
title: CSRF防御
date: 2017-09-25 17:36
tags:
    - CSRF
    - HTTP Referer
---

### 前期参考
关于CSRF的基本知识及防御手段请参考：[CSRF 攻击的应对之道](https://www.ibm.com/developerworks/cn/web/1102_niugang_csrf/)

### 防御方法

#### 方法一：验证 HTTP Referer 字段
以下是使用验证 HTTP Referer 字段的方式实现的预防方法：
``` java
/**
 * 基于referer进行CSRF验证
 * @param  request
 * @param  refererDomain 服务端要求的域名源
 * @return true：CSRF攻击，false：合法访问
 */
public static boolean isCsrf(HttpServletRequest request, String refererDomain) {

	String referer = request.getHeader("Referer");
	if (StringUtils.isEmpty(referer)) {
		return true;
	}
	int domainStart = referer.indexOf("://");
	if (domainStart == -1) {
		domainStart = 0;
	} else {
		domainStart += 3;
	}

	/* 如果带端口,则冒号结尾,否则斜线结尾 */
	int domainEnd = referer.indexOf(':', domainStart);
	if (domainEnd == -1) {
		domainEnd = referer.indexOf('/', domainStart);
		if (domainEnd == -1) {
			domainEnd = referer.length();
		}
	}

	return domainEnd < refererDomain.length() || !refererDomain.equals(referer.substring(domainStart, domainEnd));
}
```
此方法对于IE6等低版本的浏览器存在风险，如果服务必须支持低版本浏览器，则该方式不可用。
这种方式主要是利用了使用CSRF的网站无法改变其网站Refer。

<!--more-->

#### 方法二：token
网络中的方法比较僵化，以下介绍一种简单的方法：
用户登录验证成功后，将用户 session 以随机 UUID 为 key 存入缓存中并设置到 Cookie 中，用户在访问非登录页面的服务时，验证该 Cookie 中的 UUID 是否为当前 Session 缓存中的。
此种方法有两个好处：一是验证了该用户是否已登录，二是作为了CSRF的一种防御手段。
这种方式主要是利用了使用CSRF的网站无法获取你网站的Cookie的弱点。

#### 方法三：验证码
使用验证码的方式很多：
- 以手机短信或邮件的方式获取动态验证码来操作
- 以自己的登录密码或设置专门的密码（如专门的支付密码）作为验证码来操作；
- 以动态口令，通常有硬件口令设备、软件APP应用提供，例如QQ安全中心，不过它主要是用于登录验证的来操作。

上述就可分为动态验证码和静态验证码，验证还可以根据时间有效性分为一次验证有效，即每次操作均需提供验证码（对于动态验证码是每个验证码仅能使用一次），或是一次验证后一段时间内有效，该时段内无需再次提供验证码。
使用一次验证有效的验证码是可以极大防御CSRF攻击的，当然它也可以防止用户误操作或是其他类型的黑客攻击，防御能力极强，但是用户体验较差，通常仅用在关键步骤，如用户的支付行为。
注意使用验证码验证很容易犯一个错误，就是把整个过程分为了两个请求来执行，一个请求用来验证验证码的正确性，另一个请求用来执行相关的操作。验证和后期的操作应该是一体的，分开就体现不出验证的安全意义了。
安全是一种评估后的权衡，没有百分百的安全，应该在合适的地方运用合适的手段来预防攻击。

