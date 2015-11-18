title: XSS防御
date: 2015-11-09 23:24:48
tags: Web安全
---

## XSS的定义
- - -
XSS 全称Cross Site Script，跨站脚本攻击，HTML注入。

通过前端代码注入篡改网页，插入恶意脚本，从而在用户浏览网页时，控制用户浏览器。

带来cookie劫持问题、窃取用户信息、模拟用户身份执行操作等。

## 防御
- - -
#### HttpOnly（有助于缓解XSS攻击）

浏览器禁止页面的脚本访问带有HttpOnly属性的Cookie（严格来讲它解决的是Cookie劫持攻击）

一旦窃取用户的Cookie，就可以登录它的账户，但如果为Cookie设置了HttpOnly，这种攻击就会失败。

在所有set-cookie的地方，给关键cookie都加上HttpOnly。

HttpOnly并不是万能的，添加了不等于解决了XSS问题，它有助于缓解XSS攻击，但仍然需要其他能够解决XSS漏洞的方案。

#### 输入检查（没有结合语境，不够智能）

输入检查，很多时候被用于格式检查，必须放在服务器端代码中实现。

在XSS的防御上，输入检查一般是检查用户输入的数据中是否包含一些特殊字符，如 < > ' " 等，如果发现则过滤或编码。

比较智能的输入检查可能还会匹配XSS特征，称为“XSS Filter”

全局的XSS Filter无法看到用户数据的输出语境，可能会漏报或者改变用户数据的语义，比如1+1 > 2 ，如果XSS Filter不够智能的话，就会粗暴的把 > 给过滤了。

输入数据，还可能被展示在多个地方呢，每个地方的语境可能各不相同，如果使用单一的替换操作，则可能会出现问题。

#### 输出检查

除了富文本的输出外，在变量输出到HTML页面时，可以使用编码或转义的方式来防御XSS攻击。

为了对抗XSS，在HtmlEncode中要求至少转换以下字符：
& -> `&amp;`  
< -> `&lt;`  
&gt; -> `&gt;`  
" -> `&quot;`  
' -> `&#x27;`  
/ -> `&#x2F;`

JavaScript的编码方式可以使用JavascriptEncode，并且在对抗XSS时，还要求输出的变量必须在括号内部，以避免造成安全问题。

比较下面两种写法：
```
var x = escapeJavascript($evil);
var y = '"'+escapeJavascript($evil)+'"';
```

上面的两行代码可能会变成：
```
var x = 1;alert('xxs');
var y = "1;alert('xxs');";
```

前者执行了而外的代码，后者则是安全的。
在“Apache Common Lang”的“StringEscapeUtils”里，提供了许多的escape函数。
需要注意的是，编码后的长度可能会发生改变，从而影响某些有字符长度限制的功能。

## 正确地防御XSS
- - -
XSS的本质还是一种“HTML注入”，用户的数据被当成了HTML代码的一部分来执行。

使用MVC架构的网站，XSS发生在View层，在应用拼接变量到HTML页面时产生。所以在用户提交数据处进行输入检查的方案，其实并不是在真正发生攻击的地方做防御。

要根治XSS问题，可以列出所有XSS可能发生的场景，再一一解决。

#### 在HTML标签中输出

	<div>$var</div>
	<a href="#" >$var</a>

攻击方式：

	<div> <script>alert(/xss/)</script> </div>
	<a href="#" > <img src=# onerror=alert(1) /> </a>

防御方法： 对变量使用HtmlEncode。

#### 在HTML属性中输出

	<div id="abc" name="$var"></div>

攻击方式：

	<div id="abc" name=""><script>alert(/xss/)</script><""></div>

防御方法： 对变量使用HtmlEncode。

在OWASP ESAPI中推荐了一种更严格的HtmlEncode：除了字母、数字外，其它所有的特殊字符都被编码成HTMLEntities。

	String sage = ESAPI.encoder().encodeForHTMLAttribute(request.getParameter("input"));

这种严格的编码方式，可以保证不会出现任何安全问题。

#### 在`<script>`标签中输出

首先应该确保输出的变量在引号中：`<script>var x = "$var";</script>`
攻击者需要先闭合引号才能实施XSS攻击。
防御方法： 使用JavascriptEncode。

#### 在事件中输出

	<a href=# onclick="func('$var')">test</a>

攻击方式：

	<a href=# onclick="func('');alert(/xss/);//')">test</a>

防御方法： 使用JavascriptEncode。

#### 在地址中输出 

	[Protocal][Host][Path][Search][Hash]

在URL的path或者search中输出

	<a href="http://www.evil.com/?test=$var">test</a>

攻击方式：

	<a href="http://www.evil.com/?test=" onclick=alert(1)"">test</a>

防御方法： 使用URLEncode。

整个URL能够被用户完全控制
此时不能使用URLEncode，否则会改变URL的语义。

	<a href="$var">test</a>

攻击方式：构造伪协议实施攻击

	<a href="javascript:alert(1);">test</a>

防御方法：
1. 先检查变量是否以http开头，如果不是则自动添加，以保证不会出现伪协议类的XSS攻击
2. 再对变量进行URLEncode，即可保证不会有此类的XSS发生了。