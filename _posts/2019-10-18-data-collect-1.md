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

#### 方法一：通过浏览器对象

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

#### 方法二：梳理混淆的 JS 代码逻辑，在C# 实现其逻辑，得到映射关系

方法一确实节省时间，但并不完美。

现在我们尝试分析 JS 代码的逻辑，尤其是被混淆的那部分代码，不过总的来说这真的是个体力活，心细，做好标注就行。

好，我们开始。

- 通过  *context_kw* 字符串尝试在正文前后寻找与映射相关的 JS 代码，定位到正文后面的部分 JS 代码。

- 通过形似   _0x36ee  、 _0x1f9d8a、 _0x48e4f5  的命名，我们再次缩小需要分析的代码范围。
- 格式化确定的 JS 代码。

开始具体分析：

最开始为一个字符串数组，如下所示：

```javascript
var _0xa12e = [
    'fromCharCode', 'dCDUy', 'styleSheets', '.context_kw', '::before', 'insertRule',
    '::before{content:\x20\x22', 'cssRules', 'ZeroPadding', 'SBUEQ', 'clamp', 'sigBytes', 
    ... 				 '8RHz0u9wbbrXYJjUcstWoRU1SmEIvQZQJtdHeU9/KpK/nBtFWIzLveG63e81APFLLiBBbevCCbRPdingQfzOAFPNPBw4UJCsqrDmVXFe6+LK2CSp26aUL4S+AgWjtrByjZqnYm9H3XEWW+gLx763OGfifuNUB8AgXB7/pnNTwoLjeKDrLKzomC+pXHMGYgQJegLVezvshTGgyVrDXfw4eGSVDa3c/FpDtban34QpS3I=',
    'Latin1',
    'parse',
    'D0F33319303215D5',
    'window', 'location', 'href', 'enc',
    '146385F634C9CB00',
    'decrypt', 'toString', 'length',
    ...
    ];
```

可以看到里面有很多熟悉的变量、函数名称，常见混淆都会提取一些函数、变量、值，封装到数组中，然后通过索引访问，以达到影响阅读的目的。

~~标记，我们往下走。

```javascript
(function (_0x24d78a, _0x34d291) {
    var _0x24d797 = function (_0x48e267)
    {
        //循环重复将首元素添加到尾部 
        while (--_0x48e267) {
            _0x24d78a['push'](_0x24d78a['shift']());
        }
    };
    _0x24d797(++_0x34d291);
}(_0xa12e, 0xcc)); //204
```

这里将上面字符数组中的元素顺序加以打乱，不停将首部元素添加到尾部。关键点在于此操作循环的次数，如所示代码中的  *0xcc* 常量，我们后续应该需要尝试获取此常量。

~~继续

```javascript
var _0xea12 = function (
    _0x49c234, _0x7f6841) {
    _0x49c234 = _0x49c234 - 0x0;
    var result = _0xa12e[_0x49c234];
    return result;
};
```

通过  *var result = _0xa12e[_0x49c234];*  很明显就知道，此函数封装对数组元素的获取。第二个参数只是影响阅读，而通过  *_0x49c234 = _0x49c234 - 0x0;*   用一个变量去减一个 为 0 的数字常量，可以猜到后续应该都是传类型为 字符串的索引。

~~继续往下

```javascript
var data = _0xea12('0xc');
```

看到 data 之类命名的变量我们应该敏感一些，尝试在 Chrome 控制台中输入  *_0xea12('0xc');*  得到以下输出：

```
8RHz0u9wbbrXYJjUcstWoRU1SmEIvQZQJtdHeU9/KpK/nBtFWIzLveG63e81APFLLiBBbevCCbRPdingQfzOAFPNPBw4UJCsqrDmVXFe6+LK2CSp26aUL4S+AgWjtrByjZqnYm9H3XEWW+gLx763OGfifuNUB8AgXB7/pnNTwoLjeKDrLKzomC+pXHMGYgQJegLVezvshTGgyVrDXfw4eGSVDa3c/FpDtban34QpS3I=
```

对照后可以发现这是最开始的字符串数据中的元素。

~~继续往下，搜寻比较敏感的词汇，发现：

```javascript
var words = new Array(secWords[_0xea12('0x18')]);
```

我们尝试在控制台中输出此变量的信息：

```javascript
(24) ["，", "的", "。", "刘", "人", "一", "他", "是", "不", "在", "有", "了", "着", "”", "“", "秀", "大", "上", "道", "歆", "个", "名", "下", "地"]
```

可以发现这里面存放的都是一些映射后的词汇信息，这是很重要的信息。

现在，我们以 **words** 变量出现的地方为突破点，核心是要找到它的元素是如何被添加的，例如类似  *words[x]=xxx* 的代码。

在 JS 中快速浏览，发现以下代码（我已经对部分做了注释）：

代码 1-1

```javascript
//secWords.length
for (var i = 0x0; i < secWords[_0xea12('0x18')]; i++)
{
    //'3|5|2|4|0|1'.split('|');
    var _0x5420ee = '3|5|2|4|0|1'[_0xea12('0x17')]('|'), _0x9ff9d9 = 0x0;
    //true
    while (!![]) {
        switch (_0x5420ee[_0x9ff9d9++])
        {
            case '0': _0x423190 = _0x5796d9(_0x423190); continue;
            //String.fromCharCode();
            case '1': words[i] = String[_0xea12('0x25')](_0x423190); continue;
            //num+3
            case '2': var _0x5796d9 = function (_0x490c80)
            {
                var _0x1532b6 = {
                    'ifLSL': function _0x256992(_0x118bb, _0x36aa09)
                    {
                        return _0x118bb + _0x36aa09;
                    }
                };
                //xx.ifLSL 
                return _0x1532b6[_0xea12('0x26')](_0x490c80, 0x3 * +!(typeof document === _0xea12('0x27'))); //undefined
            }; continue;
            case '3':
                var _0x423190 = secWords[i]; continue;
            case '4': _0x423190 = _0x3e8e1e(_0x423190); continue;

            //num % 2 > 0 ? num - 2 : num - 4;
            case '5': var _0x3e8e1e = function (_0xd024e1) {
                var _0x3e40d1 = {
                    //取余
                    'mPDrG': function _0x411e6f(_0xa8939, _0x278c20)
                    { return _0xa8939 % _0x278c20; },
                    //相减
                    'DWwdv': function _0x1e0293(_0x5b15eb, _0x443876) { return _0x5b15eb - _0x443876; }
                };
                //mPDrG    DWwdv
                //obj.mPDrg(para1,2)?obj.DWwdv(para1,2):para1-4;
                return _0x3e40d1[_0xea12('0x28')](_0xd024e1, 0x2) ? _0x3e40d1[_0xea12('0x29')](_0xd024e1, 0x2) : _0xd024e1 - 0x4;
            }; continue;
        }break;
    }
}
```

结合前面给出的 字符串数据 与 Chrome 控制台，翻译此代码逻辑（C#）：

```c#
public void BuildMap(String[] oriWords, String[] targetWords)
{
     var currentWord = String.Empty; ;
    for (Int32 i = 0; i < oriWords.Length; ++i)
    {
        currentWord = oriWords[i];
        var num = Int32.Parse(currentWord);
        num =num%2>0?num+1:num-1;
        targetWords[i] = Char.ConvertFromUtf32(num);
    }
}
```

现在知道了 words 中的元素如何来的，但从 代码 1-1 的逻辑中发现   *secWords* 为原始数据的提供者，也就是说目前的重点需要解析 secWords 的数据来源过程。

尝试在 Chrome 控制台中输出  *secWords*  的信息：

```javascript
["65291", " 30339", " 12289", " 21015", " 20153", " 19967", " 20181", " 26160", " 19982", " 22311", " 26378", " 20101", " 30527", " 8222", " 8219", " 31167", " 22824", " 19977", " 36948", " 27461", " 20009", " 21518", " 19980", " 22319"]
```



------

### 代码编写

好了，大致采集思路知道了就不浪费时间了，时间紧迫，赶紧实现编码。

环境：IDE  VS2019、语言 C# 、运行时 .NET Framework4.5、客户端框架 WinForm

#### 方法一代码：

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

#### 方法二代码：







------



### 程序最终效果如下：

![img](https://github.com/degagetech/degagetech.github.io/blob/master/img/posts/data-collect-1/demo.gif?raw=true)



若需要相关代码，请通过我的邮箱联系。

