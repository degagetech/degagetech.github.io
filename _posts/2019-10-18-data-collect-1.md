---
layout: post
title: 网络信息采集案例（一）-CSS 文本显示反采集
subtitle: 采集某小说网站的文章内容。
comments: true
---



最近找工作遇到的一道试题，采集某小说网站的文章内容。

网站对文本显示做了处理，部分文字采集 CSS 样式做了替换无法直接被采集。

不过我本意不在做信息采集这块，而是通过这个获取另一个岗位的面试资格，哈哈。



😂给的比较初级的题目，我给自己定的时间是两小时内完成信息采集思路的解析以及相关代码的编写。

------

#### 分析过程

首先来看下直接获取到的文章内容信息：

```html
 <div class="rdtext" fsize="16">
 <p>公元二十年<span class="context_kw0"></span><span class="context_kw23"></span>皇元年<span class="context_kw2"></span>
</p><p>长安<span class="context_kw0"></span>奉车光禄<span class="context_kw16"></span>夫<span class="context_kw3"></span><span class="context_kw19"></span>府邸<span class="context_kw2"></span>
</p><p>密室<span class="context_kw2"></span>
```

而普通用户直接查看文章展示的内容：

```
公元二十年，地皇元年。

长安，奉车光禄大夫刘歆府邸。

密室。
```

可以看到文章中的内容被使用特定样式的元素代替了。

不用想，样式应该是 JS 动态生成的，而这部分 JS 代码应该又会做了混淆处理，先看下样式的内容。

```css
.context_kw0::before {
    content: "，";
}
```

文本的内容直接被设置在了样式的内容中，看来这个网站被采集的比较少。😂

遇到这种，时间紧迫，也不用去费劲反向混淆的 JS 代码了，直接获取文档中特定的样式对象。然后解析 cssText，把映射关系起来就行。

先在 Chrome 控制台中试下：

```javascript
var rulesObj=document.styleSheets[0].cssRules;var result=''; for(var i in rulesObj){ if(rulesObj[i].style){result+=('|'+rulesObj[i].selectorText+'='+rulesObj[i].style.cssText);} } console.log(result);
```

输出如下：

```
|.context_kw0::before=content: "，";|.context_kw1::before=content: "的";|.context_kw2::before=content: "。";|.context_kw3::before=content: "刘";|.context_kw4::before=content: "人";|.context_kw5::before=content: "一";|.context_kw6::before=content: "他";|.context_kw7::before=content: 
...
```

继续往下，为了印证想法的有效性，再次调式网站的几篇文章，发现每篇的文章的映射表不是固定的，例如：

"kw2" 在文章一中映射的为  "。"  号。而在另一篇文章中映射的为 “刘”。

*但是同一篇文章每次进去后映射表是固定的，也就是说网站有一套机制去单独设置每篇文章的映射表，而不是通过算法随机生成。*

------

#### 代码编写

好了，大致采集思路知道了就不浪费时间了，时间紧迫，赶紧实现编码。

环境：IDE  VS2019、语言 C# 、运行时 .NET Framework4.5、客户端框架 WinForm

信息采集和解析部分就不描述了，比较简单，主要是 映射表的获取，这里我由于时间原因直接用 Webbroswer 然后向其中注入 JS 代码，通过与 C# 交互传回样式文本信息，解析然后建立映射关系，替换原始采集内容中的标签。

关键代码：

```c#
    HtmlElement ele = this._wbBroswer.Document.CreateElement("script");
    ele.SetAttribute("type", "text/javascript");
    ele.SetAttribute("text", @"
    function getCssText() 
    { 
        var rulesObj=document.styleSheets[0].rules;
        var result=''; 
        for(var i in rulesObj)
        {    
          if(rulesObj[i].style)
          {
             result+=('|'+rulesObj[i].selectorText+'='+rulesObj[i].style.cssText);
           } 
         } 
         return result;
     }");
     this._wbBroswer.Document.Body.AppendChild(ele);
```

与控制台中的代码一致，需要注意的是 Webbroswer 默认是 IE 的内核，所以与在上文 Chrome 控制台中的代码有些区别，例如：rules 与 cssRules 。

最终效果如下：

![img](https://github.com/degagetech/degagetech.github.io/blob/master/img/posts/data-collect-1/demo.gif?raw=true)

若需要相关代码，请通过我的邮箱联系。

