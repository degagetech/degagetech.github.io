---
layout: post
title: 数据采集基础案例-JS分析（一）
subtitle: 采集某小说网站的文章内容。
comments: true
---



最近很多同学想要学习交流数据采集上的一些知识点，慢慢我会由易到难分享一些真实案例。

**严重声明：文中分享的代码、思路仅用于学习之用，若其他人通过此代码、思路做任何事与本人无关。**

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

代码 1-2  <a id="code-1-2"> </a>

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

我们尝试将数组中的元素代入上面实现的计算逻辑（[代码 1-2](#code-1-2)），例如  65291 ，代入后得到 65292，通过查询 UTF-8 的表后可知，此编码对应字符 "，"。正是我们在上面输出 words 中第一个元素的值。

尝试定位  secWords 声明的地方，发现如下：

```javascript
var decrypted = CryptoJS['AES'][_0xea12('0x14')](
    data,
    keywords,
    {
        'iv': iv,
        //CryptoJS.pad.ZeroPadding
        'padding': CryptoJS[_0xea12('0x0')][_0xea12('0x15')]
    });
//decrypted.toString(CryptoJS.enc.Utf8).split(',');
var secWords = decrypted[_0xea12('0x16')](CryptoJS['enc']['Utf8'])[_0xea12('0x17')](',');
//length
var words = new Array(secWords[_0xea12('0x18')]);
```

由此，牵扯出  data、keywords、iv 三个重要变量。

定位这三个变量的声明处：

```javascript
var data = _0xea12('0xc');
//CryptoJS.enc.Latin1.parse('6B0600CA9BCE5B24');
var keywords = CryptoJS[_0xea12('0xd')][_0xea12('0xe')][_0xea12('0xf')]('6B0600CA9BCE5B24');
var iv = '';
try {
    //top.window.location.href!=window.location.href
    if (top[_0xea12('0x10')][_0xea12('0x11')][_0xea12('0x12')] != window[_0xea12('0x11')]['href'])
    {
        //top.window.location.href=window.location.href;
        top['window'][_0xea12('0x11')]['href'] = window[_0xea12('0x11')][_0xea12('0x12')];
    }
    //CryptoJS.enc.Latin1.parse('6B0600CA9BCE5B24');
    iv = CryptoJS['enc'][_0xea12('0xe')]['parse']('6B0600CA9BCE5B24');
}
catch(_0x3f6f9e) {
    //CryptoJS.enc.Latin1.parse('146385F634C9CB00');
    iv = CryptoJS[_0xea12('0xd')][_0xea12('0xe')]['parse'](_0xea12('0x13'));
}
```

可以发现这三个变量数据来源要么为常量、要么来自一开始的 字符数组。

OK，逻辑大概梳理完毕，我们整理下应该实现的逻辑。

- 从原始字符串信息中获取被混淆范围的  JS 代码。
- 获取字符串数组。
- 获取数组循环操作次数，并使用此排序数组中的元素。
- 获取 data 对应的字符串信息，有可能是常量也有可能是对应在字符串数组中的索引。
- 获取 keywords 对应的字符串信息，有可能是常量也有可能是对应在字符串数组中的索引。
- 获取 iv 对应的字符串信息，有可能是常量也有可能是对应在字符串数组中的索引。
- 解密 data 对应的字符串，得到含有原始 UTF-8 编码的字符数组。
- 带入[代码1-2](#code1-2)处的计算逻辑，得到具体的字符数组。
- 映射关系建立完成。

------

### 代码编写

好了，大致采集思路知道了就不浪费时间了，时间紧迫，赶紧实现编码。

环境：IDE  VS2019、语言 C# 、运行时 .NET Framework4.5、客户端框架 WinForm。

信息采集和解析部分就不描述了，比较简单，主要是 映射关系的建立。

抽象出公共的映射关系接口，方法一、二分别继承实现即可：

```c#
public interface ISymbolTable
{
    Boolean IsReady { get; }
    Task<Boolean> ReadyAsync(CollectContextInfo context);
    Task<String> GetSymbolAsync(String key, CollectContextInfo context);
}
```

采集上下文：

```c#
public class CollectContextInfo
{
    public Uri Uri { get; set; }
    public String Original { get; set; }
    public ISymbolTable SymbolTable { get; set; }
    /// <summary>
    /// 表示当前原始内容解析的定位点
    /// </summary>
    public Int32 CurrentPosition { get; set; }
    public CollectResult Result { get; set; }

    public String[] KeywordsTable { get; set; }
}
```

采集结果：

```c#
public class CollectResult
{
    public String Title { get; set; }
    public String Content { get; set; }
    public String PreUri { get; set; }
    public String NextUri { get; set; }
}
```



#### 方法一代码：

此方法简单、快捷，直接用 Webbroswer 然后向其中注入 JS 代码，通过与 C# 交互传回样式文本信息，解析然后建立映射关系，替换原始采集内容中的标签。

声明此映射类：

```c#
  public class WebBroswerSymbolTable : ISymbolTable
  {
    //...
  }
```

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

此方法不依赖于更多的外部组件，将混淆后的 JS 代码梳理逻辑后，在 C# 中重新实现，性能比方法一要更好一些，所以将此方法作为默认 映射关系建立的方式。

声明此映射类：

```c#
  public class DefaultSymbolTable : ISymbolTable
  {
  ///...
  }
```

关键逻辑代码：

```c#

            //获取用于分析的 JS 代码
            Regex jsRegex = new Regex(@"(?<=u.AES=p._createHelper\(d\)}\)\(\);)[\s\S]+(?=</script>)");
            var jsMatch = jsRegex.Match(mapDataStr);
            if (!jsMatch.Success)
            {
                return isReady;
            }
            var jsStr = jsMatch.Value;

            //获取 strData 信息
            Regex strDataRegex = new Regex(@"(?<=var[\s\S]+\[)[\s\S]+?(?=];)");
            var strDataMatch = strDataRegex.Match(jsStr);
            if (!strDataMatch.Success)
            {
                return isReady;
            }
            Int32 startIndex = strDataMatch.Index + strDataMatch.Length;

            var strDataArray = strDataMatch.Value.Split(',');
            var strDataList = new List<String>();
            for (Int32 i = 0; i < strDataArray.Length; ++i)
            {
                strDataArray[i] = strDataArray[i].Trim('\'');
                strDataList.Add(strDataArray[i]);
            }

            //循环将第一个元素移动至末尾
            //获取循环次数
            Regex loopCountRegex = new Regex(@"(?<=push[\s\S]+,)[\s\S]+?(?=\)\))");
            var loopCountMatch = loopCountRegex.Match(jsStr, startIndex);
            if (!loopCountMatch.Success)
            {
                return isReady;
            }
            startIndex = loopCountMatch.Index + loopCountMatch.Value.Length;
            var loopCount = Convert.ToInt32(loopCountMatch.Value, 16);
            while ((--loopCount) >= 0)
            {
                var fisrt = strDataList.First();
                strDataList.RemoveAt(0);
                strDataList.Add(fisrt);
            }
            strDataArray = strDataList.ToArray();

            //获取 data 在字符数组中的索引     var data = _0xea12('0xc');
            Regex dataIndexRegex = new Regex(@"(?<=var data=.*\')[\s\S]+?(?=\'.*;)");
           var dataIndexMatch= dataIndexRegex.Match(jsStr, startIndex);
            if (!dataIndexMatch.Success)
            {
                return isReady;
            }
            startIndex = dataIndexMatch.Index + dataIndexMatch.Value.Length;

            var dataStr = dataIndexMatch.Value;
            if (dataStr.StartsWith("0x"))
            {
                var dataIndex = Convert.ToInt32(dataStr, 16);
                dataStr = strDataArray[dataIndex];
            }
           


            //获取 Key 字符串  var keywords = CryptoJS[_0xea12('0xd')][_0xea12('0xe')][_0xea12('0xf')]('6B0600CA9BCE5B24');
            Regex keyIndexRegex = new Regex(@"(?<=var keywords=CryptoJS.+]\(.*\')[\s\S]+?(?=\'.*\);)");
            var keyIndexMatch = keyIndexRegex.Match(jsStr, startIndex);
            if (!keyIndexMatch.Success)
            {
                return isReady;
            }
            startIndex = keyIndexMatch.Index + keyIndexMatch.Value.Length;
            var keyIndexStr = keyIndexMatch.Value;
            var keyStr = keyIndexStr;
            if (keyIndexStr.StartsWith("0x"))
            {
                var keyIndex = Convert.ToInt32(keyIndexStr, 16);
                keyStr = strDataArray[keyIndex];
            }

            //获取 IV 字符串
            Regex ivIndexRegex = new Regex(@"(?<=iv=CryptoJS.+]\(.*\')[\s\S]+?(?=\'.*\);)");
            var ivIndexMatch = ivIndexRegex.Match(jsStr, startIndex);
            if (!ivIndexMatch.Success)
            {
                return isReady;
            }
            startIndex = ivIndexMatch.Index + ivIndexMatch.Value.Length;
            var ivIndexStr = ivIndexMatch.Value;
            var ivStr = ivIndexStr;
            if (ivIndexStr.StartsWith("0x"))
            {
                var ivIndex = Convert.ToInt32(ivIndexStr, 16);
                ivStr = strDataArray[ivIndex];
            }

            //解密数据
            Encoding iso = Encoding.GetEncoding("ISO-8859-1");
            Byte[] key = iso.GetBytes(keyStr);
            Byte[] iv = iso.GetBytes(ivStr);

            var decryMapDataStr = String.Empty;
            using (var aesDecry = new AesManaged())
            {
                aesDecry.Padding = PaddingMode.Zeros;
                aesDecry.Mode = CipherMode.CBC;
                aesDecry.Key = key;
                aesDecry.IV = iv;
                ICryptoTransform decryptor = aesDecry.CreateDecryptor(aesDecry.Key, aesDecry.IV);
                var oriDataByte = Convert.FromBase64String(dataStr);

                using (MemoryStream msDecrypt = new MemoryStream(oriDataByte))
                {
                    using (CryptoStream csDecrypt = new CryptoStream(msDecrypt, decryptor, CryptoStreamMode.Read))
                    {
                        using (StreamReader srDecrypt = new StreamReader(csDecrypt))
                        {
                            decryMapDataStr = srDecrypt.ReadToEnd();
                        }
                    }
                }

                decryMapDataStr = decryMapDataStr.TrimEnd('\0');
                var oriWords = decryMapDataStr.Split(',');
                var targetWords = new String[oriWords.Length];
                BuildMap(oriWords, targetWords);
                //....
            }

//....
```

不在赘述，若有疑惑之处需要交流请发送邮件。

若需要相关代码，请发送邮件。（博客底部有）

------



### 程序最终效果如下：

![img](https://github.com/degagetech/degagetech.github.io/blob/master/img/posts/data-collect-1/demo.gif?raw=true)



------

拜了个拜  😄😄 😄😄 