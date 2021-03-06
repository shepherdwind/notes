title: "一次有意思的内部讨论-sku组合查询算法探索"
id: 339
date: 2012-04-27 23:20:05
tags: 
- 算法
categories: 
- 电脑网络
---

<link rel="stylesheet" href="http://ued.taobao.org/blog/wp-content/uploads/2012/07/hanwen/path.css" />

在前端领域，很少会遇到算法问题，这不能说不是一种遗憾。不过，随着前端处理的任务越来越复杂和重要，偶尔，也能遇到一些算法上的问题。本文，所要讨论的，就是这样一样问题。

### 什么是SKU

问题来自垂直导购线周会的一次讨论，sku组合查询，这个题目比较俗，是我自己取得。首先，看下什么是sku，来自维基百科的解释：
> 最小存货单位([Stock Keeping Unit](http://en.wikipedia.org/wiki/Stock-keeping_unit))在连锁零售门店中有时称单品为一个SKU，定义为保存库存控制的最小可用单位，例如纺织品中一个SKU通常表示规格、颜色、款式。

让我们假设在淘宝上，有这么一个手机，如下表格所示：
<table><tbody><tr><th>颜色</th><th>容量</th><th>保修期限</th><th>屏幕大小</th><th>电池容量</th></tr><tr><td>红色</td><td>4G</td><td>1 month</td><td>**3.7**</td><td>1500mAh</td></tr><tr><td>**白色**</td><td>8G</td><td>**3 month**</td><td>4</td><td>1900mAh</td></tr><tr><td>黑色</td><td>**16G**</td><td>6 month</td><td>4.3</td><td>**2100mAh**</td></tr><tr><td>黄色</td><td>64G</td><td>1 year</td><td>&nbsp;</td><td>2500mAh</td></tr><tr><td>蓝色</td><td>128G</td><td>&nbsp;</td><td>&nbsp;</td><td>&nbsp;</td></tr></tbody></table>sku: 白色 + 16G + 3 month + 3.7 + 2100mAh就这么一款可以提供5种颜色，5种容量，4种保修期限， 3种屏幕尺寸，4种电池容量的手机，我们假设它存在，叫xphone。表格中，加粗的5种属性，组合在一起，构成一个sku。现在，应该清楚什么是sku了吧。可以把xphone的规格参数看成一个JS的构造器，每一个sku，对xphone函数进行实例化，返回的一个对象就是一个sku。不过，这和一部手机的概念有一些区别，一个sku对应多个手机，sku是描述手机的最小单位，比如说学校，在学校里面最小教学单位是班级，那么一个班级可以看做一个sku。

### 问题描述

下面，为了描述问题，我首先假设一个产品属性组合2x2，用`[[a, A], [b, B]]`表示，那么，sku组合为`[ab, Ab, Ab, AB]`，也是2x2，4个sku。现在我们知道sku对应的数目和价格，依然用js对象来描述为：

```
{
    ab: {amount: 10, price: 20}
    aB: {amount: 10, price: 30}
    AB: {amount: 10, price: 40}
}
```

这个的数据说明了，Ab是没有货存的，ab, aB, AB分别有10个货源在。那么，当用户对商品进行选择的时候，如果首先选择A，那么，b应该显示为不可选择状态，因为Ab是没有货的。同样，如果选择了b，那么A应为灰掉，因为Ab还是没有值的。可能的几种状态如下：
<div class="alert alert-info"><div class="ui-inline-block">  初始状态  <div class="group">    <span class="text">属性1：</span>    <button class="btn" id="A">A</button>    <button class="btn" id="a">a</button>  </div>  <div class="group">    <span class="text">属性2：</span>    <button class="btn" id="B">B</button>    <button class="btn" id="b">b</button>  </div></div><div class="ui-inline-block">  1\. 选中b，A禁止  <div class="group">    <span class="text">属性1：</span>    <button class="btn disabled" id="A">A</button>    <button class="btn" id="a">a</button>  </div>  <div class="group">    <span class="text">属性2：</span>    <button class="btn" id="B">B</button>    <button class="btn active btn-warning" id="b">b</button>  </div></div><div class="ui-inline-block">  2\. 选中A，b禁止  <div class="group">    <span class="text">属性1：</span>    <button class="btn active btn-warning" id="A">A</button>    <button class="btn" id="a">a</button>  </div>  <div class="group">    <span class="text">属性2：</span>    <button class="btn" id="B">B</button>    <button class="btn disabled" id="b">b</button>  </div></div><div class="ui-inline-block">  3\. 选中AB，价格为40  <div class="group">    <span class="text">属性1：</span>    <button class="btn active btn-warning" id="A">A</button>    <button class="btn" id="a">a</button>  </div>  <div class="group">    <span class="text">属性2：</span>    <button class="btn active btn-warning" id="B">B</button>    <button class="btn disabled" id="b">b</button>  </div></div></div>  

问题：用户选择某个属性后，如何判断哪些属性是可以被选择的。当sku属性只是2x2的时候，还是很容易计算的。但是，如果情况变得复杂，比如4x4x4x5这样的情况，要判断用户的那些行为是可行的，还是会复杂很多的。下面看算法实现吧，还是用2x2这种最简单的形式作为参考。为了方便描述，下面使用`result = {ab: ...}`表示sku对应的价格和数目的数据对象，使用`item`表示一个sku属性下的一个元素，`items = [[a, A], [b, B]]`表示所有sku属性元素。

### 算法演示

首先来一个演示吧，仅支持高级浏览器。对于第一算法，使用正则匹配，不是很完善，有些不准，仅仅是演示，正则表达式写的不好，不用在意。

下面灰色按钮表示不可选，白色表示可选，红色为选中状态。演示框最下面是可用的sku组合。
<div class="alert alert-info">

<label for="J_input">item数组</label><input type="text" value="[8, 8, 8, 8, 8, 8, 8]" name="a" id="J_input"><label for="J_nums">随机数范围</label><input type="text" value="500" name="num" id="J_nums"><button class="btn" id="J_sub">重新生成</button>
第一种算法[正则]：
<div id="J_demo" class="ui-inline-block">共进行323次运算，耗时2ms<div class="group"><span class="text">属性1：</span><button class="btn" data-id="2">002</button><button class="btn" data-id="3">003</button><button class="btn" data-id="5">005</button><button class="btn" data-id="7">007</button><button class="btn" data-id="11">011</button><button class="btn" data-id="13">013</button><button class="btn" data-id="17">017</button><button class="btn" data-id="19">019</button></div><div class="group"><span class="text">属性2：</span><button class="btn" data-id="23">023</button><button class="btn" data-id="29">029</button><button class="btn" data-id="31">031</button><button class="btn" data-id="37">037</button><button class="btn" data-id="41">041</button><button class="btn" data-id="43">043</button><button class="btn" data-id="47">047</button><button class="btn" data-id="53">053</button></div><div class="group"><span class="text">属性3：</span><button class="btn" data-id="59">059</button><button class="btn" data-id="61">061</button><button class="btn" data-id="67">067</button><button class="btn" data-id="71">071</button><button class="btn" data-id="73">073</button><button class="btn" data-id="79">079</button><button class="btn" data-id="83">083</button><button class="btn" data-id="89">089</button></div><div class="group"><span class="text">属性4：</span><button class="btn" data-id="97">097</button><button class="btn" data-id="101">101</button><button class="btn" data-id="103">103</button><button class="btn" data-id="107">107</button><button class="btn" data-id="109">109</button><button class="btn" data-id="113">113</button><button class="btn" data-id="127">127</button><button class="btn" data-id="131">131</button></div><div class="group"><span class="text">属性5：</span><button class="btn" data-id="137">137</button><button class="btn disabled" data-id="139">139</button><button class="btn" data-id="149">149</button><button class="btn" data-id="151">151</button><button class="btn" data-id="157">157</button><button class="btn" data-id="163">163</button><button class="btn" data-id="167">167</button><button class="btn" data-id="173">173</button></div><div class="group"><span class="text">属性6：</span><button class="btn" data-id="179">179</button><button class="btn" data-id="181">181</button><button class="btn" data-id="191">191</button><button class="btn" data-id="193">193</button><button class="btn" data-id="197">197</button><button class="btn" data-id="199">199</button><button class="btn" data-id="211">211</button><button class="btn" data-id="223">223</button></div><div class="group"><span class="text">属性7：</span><button class="btn" data-id="227">227</button><button class="btn" data-id="229">229</button><button class="btn" data-id="233">233</button><button class="btn" data-id="239">239</button><button class="btn" data-id="241">241</button><button class="btn" data-id="251">251</button><button class="btn" data-id="257">257</button><button class="btn" data-id="263">263</button></div></div>
第一种算法优化方式[除法]：
<div id="J_demo_imporve" class="ui-inline-block">共进行387次运算，耗时1ms. result乘积最大为67672188866017<div class="group"><span class="text">属性1：</span><button class="btn" data-id="2">002</button><button class="btn" data-id="3">003</button><button class="btn disabled" data-id="5">005</button><button class="btn" data-id="7">007</button><button class="btn" data-id="11">011</button><button class="btn" data-id="13">013</button><button class="btn" data-id="17">017</button><button class="btn" data-id="19">019</button></div><div class="group"><span class="text">属性2：</span><button class="btn" data-id="23">023</button><button class="btn" data-id="29">029</button><button class="btn" data-id="31">031</button><button class="btn" data-id="37">037</button><button class="btn" data-id="41">041</button><button class="btn" data-id="43">043</button><button class="btn" data-id="47">047</button><button class="btn" data-id="53">053</button></div><div class="group"><span class="text">属性3：</span><button class="btn" data-id="59">059</button><button class="btn" data-id="61">061</button><button class="btn" data-id="67">067</button><button class="btn" data-id="71">071</button><button class="btn" data-id="73">073</button><button class="btn" data-id="79">079</button><button class="btn" data-id="83">083</button><button class="btn" data-id="89">089</button></div><div class="group"><span class="text">属性4：</span><button class="btn" data-id="97">097</button><button class="btn" data-id="101">101</button><button class="btn" data-id="103">103</button><button class="btn" data-id="107">107</button><button class="btn" data-id="109">109</button><button class="btn" data-id="113">113</button><button class="btn" data-id="127">127</button><button class="btn" data-id="131">131</button></div><div class="group"><span class="text">属性5：</span><button class="btn" data-id="137">137</button><button class="btn disabled" data-id="139">139</button><button class="btn" data-id="149">149</button><button class="btn" data-id="151">151</button><button class="btn" data-id="157">157</button><button class="btn" data-id="163">163</button><button class="btn" data-id="167">167</button><button class="btn" data-id="173">173</button></div><div class="group"><span class="text">属性6：</span><button class="btn" data-id="179">179</button><button class="btn" data-id="181">181</button><button class="btn" data-id="191">191</button><button class="btn" data-id="193">193</button><button class="btn" data-id="197">197</button><button class="btn" data-id="199">199</button><button class="btn" data-id="211">211</button><button class="btn" data-id="223">223</button></div><div class="group"><span class="text">属性7：</span><button class="btn" data-id="227">227</button><button class="btn" data-id="229">229</button><button class="btn" data-id="233">233</button><button class="btn" data-id="239">239</button><button class="btn" data-id="241">241</button><button class="btn" data-id="251">251</button><button class="btn" data-id="257">257</button><button class="btn" data-id="263">263</button></div></div><div id="J_open_way">可选择的路线: 
<div id="3:23:83:101:137:179:227" class="way ui-inline-block">3:23:83:101:137:179:227</div><div id="11:53:67:131:157:191:251" class="way ui-inline-block">11:53:67:131:157:191:251</div><div id="7:29:73:127:167:211:257" class="way ui-inline-block">7:29:73:127:167:211:257</div><div id="17:47:71:103:151:199:241" class="way ui-inline-block">17:47:71:103:151:199:241</div><div id="3:41:89:107:151:223:257" class="way ui-inline-block">3:41:89:107:151:223:257</div><div id="19:53:83:109:163:199:229" class="way ui-inline-block">19:53:83:109:163:199:229</div><div id="13:43:79:101:173:211:239" class="way ui-inline-block">13:43:79:101:173:211:239</div><div id="19:47:67:127:157:191:229" class="way ui-inline-block">19:47:67:127:157:191:229</div><div id="3:37:61:107:167:223:251" class="way ui-inline-block">3:37:61:107:167:223:251</div><div id="2:31:89:97:151:199:257" class="way ui-inline-block">2:31:89:97:151:199:257</div><div id="13:29:59:113:149:193:227" class="way ui-inline-block">13:29:59:113:149:193:227</div><div id="11:41:73:103:173:181:263" class="way ui-inline-block">11:41:73:103:173:181:263</div><div id="19:43:89:103:151:197:263" class="way ui-inline-block">19:43:89:103:151:197:263</div><div id="3:53:83:107:149:199:241" class="way ui-inline-block">3:53:83:107:149:199:241</div><div id="11:29:71:131:157:211:233" class="way ui-inline-block">11:29:71:131:157:211:233</div><div id="13:31:61:113:173:181:229" class="way ui-inline-block">13:31:61:113:173:181:229</div><div id="2:37:83:127:137:211:251" class="way ui-inline-block">2:37:83:127:137:211:251</div><div id="17:53:89:97:151:191:229" class="way ui-inline-block">17:53:89:97:151:191:229</div><div id="3:31:73:103:167:193:239" class="way ui-inline-block">3:31:73:103:167:193:239</div></div></div>

### 第一种算法
<dl><dt>初始条件</dt><dd>已知所有sku属性的数组`items`和sku所对应的价格信息`result`</dd><dd>用户选择了`item` B，使用数组`selected=['B']`表示，`selected`可以为空数组</dd></dl><dl><dt>算法过程</dt><dd>1\. 循环所有sku属性`forEach(result, (curitems, attr)-&gt;)`，使curitems等于属性对应的所有元素，attr等于属性id。</dd><dd>2\. 克隆数据`attrSelected = selected`</dd><dd>3\. 判断属性`attr`中是否有元素在数组`attrSelected`中，如果存在，从`attrSelected`去掉存在的元素</dd><dd>4\. 循环属性下的元素`forEach(curitems, (item)-&gt;`，使得item等于单个属性的值</dd><dd>5\. 把 `attrSelected`和`item`组合成sku</dd><dd>6\. 循环`result`，判断第五组成的sku在result中是否存在，如果存在，退出循环4，返回true，进入步骤8</dd><dd>7\. 当前`item`设置为灰色，标志不可选择</dd><dd>8\. 当前`item`为可选属性元素</dd><dd>9\. 循环4和循环1完成，所有`item`状态标注完成，算法结束</dd></dl>

这个方式是最普通的算法实现了，非常直接，一个一个判断所有的`item`是否可以被选中，判断依据是`item`和`selected`的元素组合的sku是否在`result`数组中存在。在我们上面的例子中，在初始化的情况下，用户没有选中任何元素，那么循环过程，只需要判断`a, b, A, B`在`selected`是否存在。如果，用户选中了`b`，那么循环过程中，依次判断的sku组合是`ab, Ab, B`，存在的sku组合是`ab, aB, AB`，所以因为Ab组合没有能知道，所以，A需要标注为不可点。组合sku判断的时候，需要注意的是，因为B和选中的b在同一个属性中，所以组合的时候，需要去掉b，然后组合成B，这是第3步所主要完成的过程。

这样的算法，很简单，但很繁琐，循环嵌套循环，可以简单分析一下算法复杂度。如果sku属性组合元素的总和数用m表示，结果数据长度为n，那么每次选择后，需要的算法大致步骤是m * n。这似乎不是很复杂，m * n而已，不过，每次判断一个sku组合是否和result中的组合匹配，却不是一个简单的过程，实际上，这可以看做是一个字符串匹配的一个算法了，最简单的还是使用正则匹配，m * n次正则匹配，这样就不怎么快了吧。正则表达式很不稳定，万一sku组合中有一些特殊字符，就可能导致一个正则匹配没能匹配到我们想要的表达式。

### 第一种算法的优化

经过讨论，第一种算法，有了优化的算法思路。 就第一种算法而言，正则匹配不够优雅，而且比较慢，而我们想要做的事情是比较一个组合是否包含于另外一个组合，用数学语言来描述，就是一个集合是否是另一个集合的子集，怎么来做这样的快速判断呢。

现在问题可以简化为：假设一个集合A{a, b, c}和另外一个集合B{a, e}，如何快速判断B是否是A的子集。这个问题比较简单的方法是用B中所有元素依次和A中的元素进行比较，还是简单而粗暴的方式，比正则稍微快一些。对于集合中的元素，它们都以唯一的，通过这样的特性，我们可以把所有字母转换为一个质数，那么 **集合A可以表示为集合元素(质数)的积，B同样， B是否是A的子集，这个只需要将B除以A，看看是否可以整除** ，如果可以那么说明，B是A的子集。

现在处理字符串就转换为处理乘法算法了，有了以上的分析，我们可以整理下算法过程：

1.  数据预处理，生成一组随机数，把所有item一一对应一个质数，把item组合转换为一几个  质数的积
2.  根据用户已经选择的item进行扫描所有的item，如果item已经被选中，则退出，如果没有，  则和所有已经选择的item进行相乘(特别注意，以选中的item需要去掉和当前匹配的item  在同一个类目中的item，因为一个组合不可能出现两个类目相同的item) ，这个乘机就是  上文中的集合B
3.  把集合B依次和sku组合构成的积(相当于上文中的集合A)进行相除，比较，如果整除，则  退出，当前匹配的sku可以被选中，如果一直到最好还没有匹配上，则不能被整除。  

这样优化了一下看起来比较简单的思路，但是实现起来却一点都不容易，代码在[这里](https://gist.github.com/2141756)。算法也算简化了不少，不过这个预处理过程还是比较麻烦的，而且实际上，和第一种方案的解决的算法复杂度差不多，只是比较的时候使用的是乘除法，而第一种是正则匹配罢了。

### 第二种算法

后来又过了一周，这个问题被当成一个方案来继续讨论了。大家此时差不多都无话可说了，算法都有实现了，似乎没有什么其他可说的了。就在这个问题就如此结束的时候，正豪站出来了，说不管是第一种还是第一种方案的优化方案，每次用户进行选择，都需要重复计算一遍，这样实在太麻烦了。每次都对所有spu进行扫描，这样不是很好，能不能有其他的方式呢，能否更加直接判断出一个sku是否可以被选择呢。前面的算法，一个sku是否可以被选择，需要依次循环sku 组合的所有元素才可以判断的，这样的过程一定需要吗？

第三种算法就这样诞生了，考虑到JavaScript中的对象属性访问是最快的了，那么对于如果能够直接从一个对象中读取到以选择的sku和需要匹配的sku组合对应的数目，那这样的算法简直就是不用时间啊。下面来详细描述。

下面把问题初始条件假设如下：
<div class="alert alert-info"><div class="ui-inline-block">  初始状态，选中A1  <div class="group">    <span class="text">属性1：</span>    <button class="btn active btn-warning" id="A">A1</button>    <button class="btn" id="a">A2</button>    <button class="btn">A3</button>    <button class="btn">A4</button>  </div>  <div class="group">    <span class="text">属性2：</span>    <button class="btn" id="B">B1</button>    <button class="btn" id="b">B2</button>    <button class="btn">B3</button>  </div>  <div class="group">    <span class="text">属性3：</span>    <button class="btn">C1</button>    <button class="btn">C2</button>    <button class="btn">C3</button>  </div></div></div>

假如已经选中item为A1，那么现在要计算B1是否可以被选择，那么如果我们能够直接获取到A1和B1组合的所有商品数目，那么就能知道B1是否可以被选择了。A1和B1的组合是这样计算的，在上面描述的问题空间中，A1和B1的组合，可能有以下几种： A1+B1+C1, A1+B1+C2,A1+B1+C3。这些组合就可以直接从已知的sku组合中获取信息啦，同样是对象属性查找，快得不得了。示例如下：
<div class="alert alert-info">  <div class="group">    A1选中状态下，判断B1是否可用，只需要查找A1 B1
    <button class="btn active btn-warning" id="A">A1</button>    <button class="btn ">B1</button>    =     <button class="btn active btn-warning" id="A">A1</button>    <button class="btn ">B1</button>    <button class="btn ">C1</button>    +     <button class="btn active btn-warning" id="A">A1</button>    <button class="btn ">B1</button>    <button class="btn ">C2</button>    +    <button class="btn active btn-warning" id="A">A1</button>    <button class="btn ">B1</button>    <button class="btn ">C3</button>  </div>  A1+B1+C1这样的组合，结果可以可以直接从result中获得数据结果。</div>

实际上， 对于任何一个sku和其他sku的组合都是可以通过同样的方式递归查找来实现获取其组合后的商品数目。这样的算法最大的优势是，计算过程是可以缓存的，比如计算A1是否可以被选中，那么肯定需要计算除A1+B1组合的数目，A1的数目是由A1+B1，A1+B2，A1+B3三个子集构成，这三个子集又可以拆分为更细的组合，然后这些所有的组合对应的商品数目都可以获取到了，下次需要判断A1+B2组合，则无需重复计算了。此外，我们可以清晰的获取组合相关的信息，比如某个sku下面可以有的商品数目。

算法实现[这里](https://gist.github.com/3074516)，[jsfiddle](http://jsfiddle.net/cctvu/9Y54x/)。

### 复杂度分析

第二种算法思路非常有趣，使用动态规划法，将原问题分解为相似的子问题，在求解的过程中通过子问题的解求出原问题的解。而且，最终判断一个item是否可以被选择，直接从对象中查找，属于字典查找算法了，应该是很快。但是，乍一看，还是有些问题，递归查找，数据贮存在变量中，这些都是通过空间来换取时间的做法，递归会堆栈溢出吗？查找次数到底多少？

第一个种算法的复杂度还是很容易计算的，首先假设一个n * n的矩阵构成sku属性，比如10x10表示，有10个属性，每个属性有10个元素。假设可选择的result长度是m，那么，第一种算法的复杂度大概是 n * n * m，这样的算法还是很快的。只是，如果每一个步骤，都使用正则表达式匹配，根据上面的演示，确实会有一些些慢，不过正则表达式的是模糊匹配，可能不是那么稳定。不过除法方式判断需要生成足够的质数，当几个数的乘积太大的时候，可能导致计算机无法运算，所有，使用第1种算法的优化算法，也是有一定限制的。js里面，能够处理的最大数字大概是19位，这个范围内可以操作的范围还是比较大的，这个就不做推算了。此外，通用可以引入负数，这样就可以把质数的范围增大一倍，计算量也小一些，可以处理更大的输入规模了。

第二种算法复杂度，同样对于n * n的数据输入，从第一排算起，第一排第一个A1，组合为A1 + B1, A1 + B2 ...函数递归进入第二层，第二层从第一个B1开始，组合为A1 + B1+ C1, A1 + B1 + C2 ...进入第三层，以此类推，函数每增加一层，需要的计算量是上一层的n倍，总数是 n + n<sup><small>2</small></sup> + n<sup><small>3</small></sup> + ... + n<sup><small>n</small></sup>，这个数目是非常庞大了，算法复杂度用n<sup><small>n</small></sup>来描述了，如果是10x10的sku属性组合，初始化需要100亿次计算，有些吓人了，这还需要一个同样庞大的内存数组。

### 第二种算法的优化

经过上面的算法分析，似乎第二种算法是错误的，无法执行。不过，仔细想想，第二种方法第一初始化的时候算法复杂度非常高，几乎是浏览器无法承受的。但是，一旦数据初始化完成，后面的过程就非常简单了，同样对于n * n规模的输入，每次用户选择，这个时候，需要进行的操作是把所有数据遍历一遍，然后直接查询是否存可以被选中。算法复杂度是n * n。比起上面第一种算法的优化算法要快，现在主要的问题是，初始化如果使用自上而下，不断拆分问题，这样运算复杂度指数级增加，不过，算法本身是可行的，数据初始化过程，还是需要进一步优化。

第二种算法，把问题一层一层拆分，查找过程分解太过于琐碎，有很多的组合，是完全不可能存在的，算法非常浪费。如果，直接从获得的result数组中读取数据组合，只需要把result循环一遍，所有可能的组合就都可以计算出来了。举个例子，从最上面的2x2的result中，我们知道result对象

```
    ab: {amount: 10, price: 20}
    aB: {amount: 10, price: 30}
    AB: {amount: 10, price: 40}
```

计算过程，循环result

1.  第一次分解ab，a = 10, ab = 10, b = 10
2.  第二次分解aB, a = a + 10 = 20, aB = 10, B = 10
3.  第三次分解AB, A = 10, AB = 10, B = B + 10 = 20

三次循环，得到一个新的数据结构`var map = {a: 20, ab: 10, b: 10, aB: 10, AB: 10, A: 10, B: 10}`通过这个对象，就可以判断任何情况了。比如，初始化的时候，需要查找a, b, c,d，直接查找map对象中是否存在a, b, c, d。如果选择了a，那么需要判断aB, ab，统一直接查找的方式。

经过这样的优化，初始化的时候计算量也不大，这样第二种算法的实现就可以很好的完成任务了。可能这个map对象，可能还是会有点大。

### 结论

总的来说，比较好的方式是第一种算法的优化(也就是除法判断)和第二种算法。各自有其特点，都有其特色之处，除法判断把 **字符串匹配转换为数字运算** ，第二种算法使用 **字典查找** 。并且都能 **快速准确** 的计算出结果。

从算法速度来说，第一种算法复杂度是n * n * m，当然需要一个比较繁琐负责的质数对应转换过程，第二种算法复杂度是 n * n，其初始化过程比较复杂，最初的方式是n<sup><small>n</small></sup>，经过优化，可以提高到n!，n的阶乘。从理论上而言，n<sup><small>n</small></sup>或者n!都是不可用的算法了，就实际情况而言，sku组合大多在，6x6以下，第二种算法还是非常快的。

从算法本身而言，第二种算法想法非常奇妙，容易理解，实现代码优雅。只是初始化比较慢，在初始化可以接受的情况下，还是非常推荐的，比如淘宝线上的sku判断。此外，第二种算法获得的结果比起第一种更具有价值，第二种方式直接取得组合对应的数目，价格信息，而第一种只是判断是否可以组合，从实际应用角度而言，第二种方式还是剩下不少事的。

感觉只要善于去发现，还能能够找到一些有意思的解决问题思路的。

<script src="http://ued.taobao.org/blog/wp-content/uploads/2012/07/hanwen/path.js" charset="utf-8"></script>
