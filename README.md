Nutch AJAX page Fetch, Parse, Index Plugin
==============

### 项目简介

基于Apache Nutch 2.3和Htmlunit, Selenium WebDriver等组件扩展，实现对于AJAX加载类型页面的完整页面内容抓取，以及特定数据项的解析和索引。

According to the implementation of Apache Nutch 2.X, we can't get dynamic HTML information from fetch pages including AJAX requests as it will ignore all AJAX requests.

This plugin will use Htmlunit and Selenium WebDriver to fetch whole page content with necessary dynamic AJAX requests. 

It developed and tested with Apache Nutch 2.3, you can try it on other Nutch 2.X version or refactor the source codes as your design.

### 主要特性

* **常规的HTML页面抓取**: 对于常规的例如新闻类没有AJAX特性的页面可以直接用Nutch自带的protocol-http插件抓取。

* **常规的AJAX页面抓取**: 对于绝大部分诸如jQuery ajax加载的页面，可以直接用htmlunit扩展插件抓取。

* **特殊的AJAX请求页面抓取**: 诸如淘宝/天猫的页面采用了独特的Kissy Javascript组件，目前测试htmlunit无法正确解析，因此退而求其次采用效率低一些的Selenium WebDriver方式实现页面数据抓取。

* **基于页面滚动的AJAX请求页面抓取**: 诸如淘宝/天猫的商品详情页面会基于页面滚动发起商品描述信息的加载，通过Htmlunit或Selenium WebDriver扩展处理可以实现此类页面数据抓取。

### 运行方式

整个项目基于官方的Apache Nutch 2.3源码基础之上添加插件代码和配置，运行方式和官方指南保持一致，具体请参考：http://wiki.apache.org/nutch/

同时工程代码中提交了Eclipse的工程配置文件，可以直接import Eclipse中Run或Debug运行，Nutch工程以Ivy进行依赖管理，可采用ANT Build方式或建议在Eclipse IDE安装Apache Ivy IDE插件进行工程编译运行。

![snapshot](http://git.oschina.net/xautlx/nutch-ajax/raw/master/snapshot/eclipse-run.jpg)

![snapshot](http://git.oschina.net/xautlx/nutch-ajax/raw/master/snapshot/storage-data.jpg)

![snapshot](http://git.oschina.net/xautlx/nutch-ajax/raw/master/snapshot/parse-data.jpg)

### 插件列表

* **lib-pinyin**: 用于parse或index插件转换中文到拼音提交solr；部署用于solr dataimporthandler组件进行拼音转换的transformer扩展插件

* **lib-htmlunit**: 基于Htmlunit的多线程处理，缓存控制，请求正则控制等特性扩展插件

* **protocol-s2jh**: 基于Htmlunit和Selenium WebDriver实现的AJAX页面Fetcher插件

* **parse-s2jh**: 基于XPath解析页面元素内容; 持久化解析到的结构化数据，如MySQL，MongoDB等; 对于个别复杂类型AJAX页面定制判断页面加载完成的回调判断逻辑

* **index-s2jh**: 追加设置需要额外传递给SOLR索引的属性数据; 设定不需要索引的页面规则;

以下对各插件进一步做功能和设计实现讲解：

### lib-pinyin：汉字转拼音组件

#### 用法场景

基于Pinyin4J组件封装实现一个汉字转拼音的Convertor帮助类用于在index插件中进行拼音属性输出，以及一个定制Solr的dataimport的Transformer插件用于在从数据库提取数据转换导入拼音索引内容。

用法场景：一般在搜索引擎或电商商品搜索等功能界面，为了用户友好体验，一般会提供一个基于拼音首字母进行AutoSuggest提示效果，可通过此组件把相关的中文属性转换写入一个额外用于索引搜索的拼音属性。

在solr-4.8.1\example\solr\collection1\conf\schema.xml中添加对应属性定义，注意设置multiValued="true"，因为存在多音字处理一段汉字转换出来可能形成多组拼音组合：

```xml
  <field name="pinyin" type="string" stored="true" indexed="true" multiValued="true"/> 
```

关于Solr Auto Suggest效果后面有专门章节进一步讲解相关处理要点。

项目提供两种典型的用法参考：

#### 在IndexingFilter调用接口把中文内容转换得到拼音属性值

在IndexingFilter中操作NutchDocument对象添加写入拼音属性，可参考S2jhIndexingFilter实现：

```java
Set<String> pinyins = ChineseToPinyinConvertor.toPinyin(TableUtil.toString(page.getTitle()),
        ChineseToPinyinConvertor.MODE.capital);
for (String pinyin : pinyins) {
    doc.add("pinyin", pinyin);
}
```

#### 在Solr DataImport配置文件中添加拼音Transformer处理：

提取数据库name字段值转换添加写入pinyin属性值：

```xml
<dataConfig>
    <document name="doc">
        <entity name="item" transformer="solr.ChineseToPinyinTransformer"
            query="select name from product">            
            <field column="name" toPinyin="pinyin" mode="both"/>
        </entity>
    </document>
</dataConfig>
```

### lib-htmlunit：扩展封装Htmlunit组件接口服务

#### ExtHtmlunitCache：缓存扩展

通过一些网站测试和Htmlunit源码分析，添加对于Cache-Control=max-age此类响应头信息的缓存处理，避免不必要的重复数据下载提高爬取效率。

#### RegexHttpWebConnection：基于正则的URL请求过滤控制

Htmlunit获取解析页面时，默认机制是发起所有相关的Page，JS或CSS(如果开启的话)请求，对于一些庞大的页面可能动辄好几十个请求，但是其中绝大部分请求对于运行解析数据没有实际作用，尤其是一些三方的统计跟踪等请求。RegexHttpWebConnection扩展标准的HttpWebConnection，参考Nutch标准的regex-urlfilter.txt对应处理机制，在conf目录下定义了一个htmlunit-urlfilter.txt文件，在此文件中类似采用加减号正则表达式规则定义exclude一些不必要的htmlunit内部请求url。

htmlunit-urlfilter.txt基本配置过程：

* 默认不做任何配置执行页面FetcherJob任务，可以看到类似的日志信息：
```
Htmlunit fetching: http://www.jumeiglobal.com/deal/ht150312p1316688t1.html
Initing web client for thread: 23
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://www.jumeiglobal.com/deal/ht150312p1316688t1.html
Thread: Thread[FetcherThread0,5,main], + Http Fetching URL: http://s0.jmstatic.com/templates/jumei/js/??/jquery/jquery-1.4.2.min.js,/v44.3/jquery.all_plugins_v1.min.js,/v44.3/Jumei.Core.min.js?V0
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://a0.jmstatic.com/4cc7ce36d221af7f/lib.js
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://a0.global.jmstatic.com/7f72d7e452b1b718/ui.js
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://a2.global.jmstatic.com/77dc800b4e0f6eee/app.js
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://a3.global.jmstatic.com/0e57821a8e24b03b/detail_main.js
Thread: Thread[FetcherThread0,5,main], + Http Fetching URL: http://s0.jmstatic.com/templates/jumei/js/carousel.basic.js
Thread: Thread[FetcherThread0,5,main], + Http Fetching URL: http://s0.jmstatic.com/templates/jumei/js/lib/sea.js?v2.3
Thread: Thread[FetcherThread0,5,main], + Http Fetching URL: http://s0.jmstatic.com/templates/jumei/js/v44.3/ibar.min.js?V0
Thread: Thread[FetcherThread0,5,main], + Http Fetching URL: http://s0.jmstatic.com/templates/jumei/js/v44.3/Jumei.Search.min.js?V0
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://click.srv.jumei.com:80/global_mc.php?da7D&callback=jQuery111207272193275877097_1427164245878&_=1427164245879
Thread: Thread[FetcherThread0,5,main], + Http Fetching URL: http://s0.jmstatic.com/templates/jumei/js/jquery/dc.js
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://www.jumei.com/i/api/webinit?callback=jQuery111207272193275877097_1427164245880&_=1427164245881
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://cart.jumei.com/i/ajax/get_cart_data/?callback=jQuery111207272193275877097_1427164245882&_ajax_=1&which_cart=all&_=1427164245883
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://v3.jiathis.com/code/jia.js?uid=1405939063755704
Thread: Thread[FetcherThread0,5,main], + Http Fetching URL: http://www.google-analytics.com/__u
Thread: Thread[FetcherThread0,5,main], - Http Fetching URL: http://hm.baidu.com/h.js?884477732c15fb2f2416fb892282394b
Thread: Thread[FetcherThread0,5,main], + Http Fetching URL: http://cpro.baidu.com/cpro/ui/rt.js
Thread: Thread[FetcherThread0,5,main], + Http Fetching URL: http://s0.jmstatic.com/templates/jumei/js/v44.3/Jumei.ClickMap.min.js?V0
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://v3.jiathis.com/code/jiathis_utility.html
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://tajs.qq.com/jiathis.php?uid=1405939063755704&dm=www.jumeiglobal.com
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://click.srv.jumei.com:80/global_f
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://p0.jmstatic.com/product_report/newKoubei/js/js/12/Ajaxpage.js?_=1427164245886
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://koubei.jumei.com/ajax/reports_for_deal_newpage.json?init=1&product_id=1316688&callback=jQuery111207272193275877097_1427164245884&_=1427164245887
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://s0.jmstatic.com/templates/jumei/js/jquery/dc.js
Thread: Thread[FetcherThread0,5,main], + Http Fetching URL: http://search.jumei.com/ajax_get_default_word?callback=jsonp1427164246064
Thread: Thread[FetcherThread2,5,main], + Http Fetching URL: http://www.google-analytics.com/__utm.gif?2Fht150312p1316688t1.html&utmht=1427164248572&utmac=UA-51447284-1
Thread: Thread[FetcherThread2,5,main], - Http Excluding URL: http://hm.baidu.com/h.js?202f3ad244bdee87bc472ca5481a9267
```

* 分析上述一系列请求，初步排除一些明星无关的请求，如ttp://cpro.baidu.com，ttp://www.google-analytics.com等这些，添加到htmlunit-urlfilter.txt
```
-^http://www.google-analytics.com
-^http://cpro.baidu.com
```

* 然后再次运行FetcherJob，可以看到相关请求已经变成Excluding：
```
Thread: Thread[FetcherThread4,5,main], - Http Excluding URL: http://cpro.baidu.com/cpro/ui/rt.js
Thread: Thread[FetcherThread0,5,main], - Http Excluding URL: http://www.google-analytics.com/__utm.gif?utmwv=5.4.9&utms=1&utmn=4168
```

* 然后在进一步尝试把“感觉无用的”URL请求添加到htmlunit-urlfilter.txt排除列表中，一步步减少不必要的HTTP请求。这个过程要注意随时观察parse组件解析出来的数据是否齐备，如果在排除过程中发现由于添加了某些url排除导致解析不到数据了，说明这个url地址是用于需要的数据解析执行的就恢复保留。通过这样尝试迭代的方式不断完善过滤配制规则，例如：
```
-^http://www.google-analytics.com
-^http://cpro.baidu.com
-^http://search.jumei.com/ajax_get_default_word
-^http://click.srv.jumei.com
-^http://www.jumei.com/serverinfo.php
-^http://cart.jumei.com/i/ajax/get_cart_data
-^http://www.jumei.com/i/ajax/get_view_history
-^http://www.jumei.com/i/ajax/get_recommend_by_history
-^http://www.jumei.com/i/ajax/get_keyword_pro
-^http://eclick.baidu.com
```

**Htmlunit过滤配置原则:**

上述过程可能感觉有点繁琐，至于做到什么程度主要取决于对爬取效率的要求，如果不太在意爬取页面的速度快慢，可以完全忽略此项配置按照htmlunit默认机制发起所有http请求；如果希望提高页面获取速度，那就可以尝试尽可能的排除不必要的url请求，可以大幅提高页面爬取和解析速度。

#### HttpWebClient：基于多线程的Htmlunit页面获取接口服务

Nutch的运行基于多线程模式，HttpWebClient提供Htmlunit Page请求接口封装，其中实现每个线程一个WebClient运行模式，从而最大程度利用Htmlunit每个WebClient对重复URL请求的Cache处理特性，对于大批量相似页面的获取可以大幅减少重复URL请求耗时。

通过仔细观察Htmlunit运行日志，可以看出运行初期请求的URL数量会比较多，当所有线程的WebClient都运行一段时间后，不少不同页面的相同URL请求会被缓存使用，JS请求的数量会逐步减少到一个稳定状态。因此，如果比较关注页面爬取或解析效率参数，建议在运行一段时间稳定之后再观察相关的平均参数信息。

### protocol-s2jh：基于Htmlunit，Selenium WebDriver扩展实现页面爬取插件

基于Nuthc标准的protocol-http组件源码基础上，添加Htmlunit，Selenium WebDriver方式爬取页面:

* 首先基于默认的HTTP模式获取页面内容，如果已经拿到所需的解析数据，那直接返回；
* 如果HTTP模式未得到数据，则调用Htmlunit方式发起AJAX支持模式请求，然后再次判断如果拿到数据则返回；
* 最后再尝试Selenium WebDriver方式模拟浏览器运行获取页面，只要用户通过浏览器能看到的数据，通过这一步应该是都能获取到。

上述过程主要一个判断就是组件获取的HTML内容是否已包含后续parse组件所需内容，基本思路是遍历所有自定义的ParseFilter组件，根据每个匹配的过滤器组件内部定义的解析内容判定接口方法，直到所有匹配的过滤器组件返回确认所需数据都已满足则标识爬取成功返回，如果到最后超时都没有获取所需数据则抛出异常。具体实现可参考HttpResponse中的isParseDataFetchLoaded及关联调用的ParseFilter组件的对应方法。

注意：如果需要支持Selenium WebDriver运行模式，一般建议在Desktop类型操作系统部署，需要在操作系统层面确保已安装相关的WebDriver浏览器程序，尤其对于Linux Server一般都是以headless行模式启动运行的需要另行安装相关软件和配置，具体请参考Selenium官方网站相关指南或网上搜索Selenium Headless Linux等关键词。

### parse-s2jh：解析特定业务属性并进行持久化处理

Nutch自带一些如parse-html，parse-meta等解析插件，主要用于解析一些通用的HTML属性值，如title，meta，charset，text，links等。对于定向采集需求，一般需要再进一步提取解析页面特定元素内容，如商品价格，商品描述，新闻发布时间，新闻摘要等信息。同时，采集到这些数据后，一般都会持久化到自己特定业务系统数据库中使用，如商品信息显示，新闻发布等。

本插件提供一系列数据项解析和定制持久化的处理封装，具体接口方法详见代码注释和实现：

* AbstractHtmlParseFilter：封装的定制解析器基类
* CrawlData：解析属性结构对象定义
* XyzHtmlParseFilter：特定站点页面的定制化解析处理器

### index-s2jh：扩展的index索引处理插件

Nutch的索引处理，除了自家的Solr以外，还提供了Elastic支持可参考官方资料。本教程以Solr为例。Nutch自带一些如index-basic，index-meta等索引处理插件，主要用于提取一些通用的页面属性添加到对应的solr索引属性。

本插件提供结合parse插件组合控制只有业务相关的URL才进行索引，其余业务无关仅用于连接分析的页面忽略；取上述parse定制过滤器写入MetaData的索引属性写入处理。

* AbstractIndexingFilter：封装的定制索引处理器基类
* S2jhIndexingFilter：通用的索引处理器，可参考此实现定制特殊的索引处理器

### 许可说明

* Free Open Source

本项目所有代码完整开源，在保留标识本项目来源信息以及保证不对本项目进行非授权的销售行为的前提下，可以以任意方式自由免费使用：开源、非开源、商业及非商业。

* Charge Support Service

如果你还有兴趣在Apache Nutch/Solr/Lucene等系列技术的定制的扩展实现/技术咨询服务/毕业设计指导/二次开发项目指导等方面的合作意向，项目采用文档和服务收费模式，具体可访问淘宝店铺查阅相关商品及报价：http://shop116928923.taobao.com/

### Reference

欢迎关注作者其他项目：

* [Nutch 2.X AJAX Plugins (Active)](https://github.com/xautlx/nutch-ajax) -  基于Apache Nutch 2.3和Htmlunit, Selenium WebDriver等组件扩展，实现对于AJAX加载类型页面的完整页面内容抓取，以及特定数据项的解析和索引

* [S2JH4Net (Active)](https://github.com/xautlx/s2jh4net) -  基于Spring MVC+Spring+JPA+Hibernate的面向互联网及企业Web应用开发框架

* [S2JH (Deprecated)](https://github.com/xautlx/s2jh) -  基于Struts2+Spring+JPA+Hibernate的面向企业Web应用开发框架
 
* [Nutch 1.X AJAX Plugins (Deprecated)](https://github.com/xautlx/nutch-htmlunit) -  基于Apache Nutch 1.X和Htmlunit的扩展实现AJAX页面爬虫抓取解析插件
 
* [12306 Hunter (Deprecated)](https://github.com/xautlx/12306-hunter) - （功能已失效不可用，不过还可以当作Swing开发样列参考只用）Java Swing C/S版本12306订票助手，用处你懂的