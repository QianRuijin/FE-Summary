Web前端安全问题浅析!
===================


Web安全一直以来都是一个值得深究的课题，技术的快速迭代与更新的同时，新的安全问题与解决手段也层出不穷。Web前端是Web安全的第一道屏障，也是最容易被Web开发者所忽略的一个盲点。而本着“安全问题无小事”的原则，任何一个可能导致信息安全问题的事情我们都该重视，因为小概率事件最容易发生。本文尝试从Web前端的角度去浅析当前前端领域应该重视的安全问题，希望引起大家的重视。欢迎大家补充交流。

----------


> **目录:**

> - XSS.
> - CSRF.
> - SQL注入.
> - JSONP安全问题.

### XSS

#### XSS是什么？
XSS全称是Cross Site Scripting，即跨站脚本。为了与CSS(Cascading Style Sheets：层叠样式表)区别，故简称XSS。其特点是不会对服务器端直接造成任何伤害，其手法是通过一些正常的页面交互途径，例如表单填写等，提交含有 JavaScript 的内容文本。这时服务器端如果没有过滤或转义掉这些脚本，作为内容发布到了页面上，其他用户访问这个页面的时候就会运行这些脚本。
##### XSS的分类及原理
当前，网上讨论XSS的分类的时候经常做下面三个类：反射型XSS(非持久型)、存储型XSS(持久型)和DOM XSS。其实，就本质来看，分类应该只有反射型XSS(非持久型)和存储型XSS(持久型)，而DOM XSS是反射型XSS的一种。或者DOM XSS和非DOM XSS两种分类。这里，为不引起歧义，我们不过分纠结分类的问题。主要介绍下这三种XSS攻击的。

###### 1.反射型XSS(非持久型)
页面在发出请求时，XSS代码嵌入在URL中，作为请求的一部分一起提交到服务器端。服务器端解析后响应时如果未对代码进行XSS过滤，则含XSS代码随响应内容一起传回给浏览器，最后浏览器解析执行XSS代码。这个过程看起来很像是一次反射，故叫反射型XSS。
举例来看：

```html
// 访问a站点的请求
http://www.a.com?test=<script>alert('attack')</script>

// a站点的服务端代码
<?php
  echo $_GET['test'];
?>

// 那么，前端将会直接执行alert('attack')的代码。
```

###### 2.存储型XSS(持久型)
与存储型XSS有所差别的是，反射型XSS提交的代码会存储在服务器端（数据库，文件系统等后端存储介质），下次请求前端页面时不用再提交XSS代码，访问此页面将直接导致XSS代码的执行。
举例来看：
>  最典型也是最常见的例子就是评论区的XSS，用户在评论时提交一条包含XSS代码的评论到服务器的，服务器短如果没有做XSS过滤直接存储到数据库。那么下次其它任何用户在请求访问这个被XSS攻击过的页面时，那些评论的内容会从数据库查询出来并返回，浏览器就会当做正常的HTML与代码来解析执行，于是触发了XSS攻击。

###### 3.DOM XSS
 DOM-based XSS是一种基于文档对象模型（Document Object Model，DOM)的XSS漏洞。简单理解，DOM XSS就是出现在JavaScript代码中的漏洞。与普通XSS不同的是，DOM XSS是在浏览器的解析中改变页面DOM树，且恶意代码并不在返回页面源码中回显，这使我们无法通过特征匹配来检测DOM XSS，给自动化漏洞检测带来了挑战。
 > 例如：前端经常读取URL中的参数来显示到页面中，那么攻击者完全可以伪造一个带有XSS攻击代码的URL到参数当中，直接导致页面执行XSS脚本，触发XSS攻击。


```html
http://www.abc.com/test.html代码如下：
<script>
eval(location.hash.substr(1));
</script>
触发方式为：http://www.abc.com/test.html#alert(1)
```

### CSRF
CSRF（Cross-site request forgery），中文名称：跨站请求伪造。它所带来的危害包括以你名义发送消息、发送邮件，盗取你个人信息，对于一些防范不严格的网站甚至可以以你的名义进行商品交易和转账等行为，从而造成个人隐私泄漏和财产安全受到威胁的严重后果。
#### 原理
![CSRF原理](http://pic002.cnblogs.com/img/hyddd/200904/2009040916453171.jpg)

这是绝大部分讲解CSRF原理的时候都会贴出来的图片，图片[出处](http://www.cnblogs.com/hyddd/archive/2009/04/09/1432744.html)。这里大家如果有时间可以去看一下这篇文章，分析的很好。
通俗的讲，其原理就是在你登录到A站点又没有退出的情况下，又去访问了带有恶意请求的B站点，那么B站点就会向A站点发送请求。而这些请求又带了A站点留下的cookie信息，如果服务器没有做CSRF防御的话就会带来严重的恶果。
#### 举个栗子
某虚拟交易网站A，你访问并登录了A站点，它有这样一个action，``如：http://www.a.com/makeDeal.action?toId=123&fee=10000.
诱导你访问的B站点有个恶意请求，它有这样一个html结构：
``` html
<img src="http://www.a.com/makeDeal.action?toId=123&fee=10000">
```
如果A站点没有做任何CSRF的防御，那么很遗憾，你可能会损失10000块RMB。
那么为啥呢？在访问危险网站B的之前，你已经登录了网站A，而B中的<img>来请求来自A站点的资源，所以你的浏览器会带上你的网站A的Cookie发出Get请求，结果A网站服务器收到请求后，认为这是一个合理合法的请求，去处理对应的请求，所以就立刻就触发了对应的action，让你损失了1w块......
类似的例子有很多，那么如何防范CSRF呢？
#### 防御
1. 使用合理的请求方法，如敏感操作请使用POST方法，而不是GET；
2. 后端服务器也要使用合适的方法获取参数，如PHP的话使用$_POST来获取对应的参数，而不是$_REQUEST;
3. 使用token。前端在每一个请求后面加一个token，token根据skey等用户唯一值生成，然后后端用同样的算法进行校验，通过后即可。目前公司绝大部分的业务都使用到了token。

### SQL注入
所谓SQL注入式攻击，就是攻击者把含有SQL的语句插入到Web表单的输入域或页面请求的查询字符串，欺骗服务器执行恶意的SQL命令。甚至在某些特定情况下，动态合并拼凑SQL语句，以达到欺骗服务器的目的。这是很多小型网站都不太注重的问题，笔者读研期间朋友的实验室曾经被人利用SQL注入几分钟拖库20万条记录。
#### 防范
跟XSS攻击的防范一样对输入的表单字段进行过滤，过滤掉SQL注入。同时，后端对取到的传递的参数都进行过滤，防止SQL注入。

###  JSONP安全问题
同源策略限制从一个源加载的文档或脚本如何与来自另一个源的资源进行交互。这是一个用于隔离潜在恶意文件的关键的安全机制。其中同源具体是指：果协议，端口（如果指定了一个）和域名对于两个页面是相同的，则两个页面具有相同的源。
但是在很多实际应用场景中，我们不得不需要绕开浏览器的同源策略，因此出现了JSONP、图片ping等绕开同源策略的技术。其中，应用最为广泛的就是JSONP技术。
#### 什么是JSONP
JSONP 全称是 JSON with Padding ，是基于 JSON 格式的为解决跨域请求资源而产生的解决方案。他实现的基本原理是利用了 HTML 里 <script></script> 元素标签，远程调用 JSON 文件来实现数据传递。当前，前端主流的技术框架基本上都提供了JSONP的方法，如jQuery、ng等。
但随着JSONP越来越普及，其也逐渐暴露出一些安全方面的问题。
#### JSONP安全问题

##### JSON劫持
JSON 劫持又为“ JSON Hijacking ”，最开始提出这个概念大概是在 2008 年国外有安全研究人员提到这个 JSONP 带来的风险。其实这个问题属于 CSRF（ Cross-site request forgery 跨站请求伪造）攻击范畴。当某网站听过 JSONP 的方式来快域（一般为子域）传递用户认证后的敏感信息时，攻击者可以构造恶意的 JSONP 调用页面，诱导被攻击者访问来达到截取用户敏感信息的目的。
##### Callback 可定义导致的安全问题
绝大部分提供JSONP的站点都会允许自定义callback，那么这里就会带来很大的安全隐患。如：
1. 在早期 JSON 出现时候，大家都没有合格的编码习惯。再输出 JSON 时，没有严格定义好 Content-Type（ Content-Type: application/json ）然后加上 callback 这个输出点没有进行过滤直接导致了一个典型的 XSS 漏洞；
2. 对callback和输出没有进行严格的过滤，导致XSS攻击。这样的防御机制是比较传统的攻防思维，对输出点进行 xss 过滤。

##### 安全防范
1. 定义严格的 Content-Type（ Content-Type: application/json ）；
2. 对callback和输出进行严格的XSS过滤；
3. 对请求来源Referer进行严格的过滤；


#### 参考
1.[浏览器同源政策及其规避方法](http://www.ruanyifeng.com/blog/2016/04/same-origin-policy.html)
2.[JSONP 安全攻防技术](http://blog.knownsec.com/2015/03/jsonp_security_technic/)
 3.[浅谈CSRF攻击方式](http://www.cnblogs.com/hyddd/archive/2009/04/09/1432744.html)
 4.[SQL注入的基本原理和防御方法](http://www.frontopen.com/953.html)
5.[XSS攻击常识及常见的XSS攻击脚本汇总](https://www.lvtao.net/dev/xss.html)
6.其它
