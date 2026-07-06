## 选择你的路径

不确定从哪里开始？选择与你目标匹配的路径：

| 我想... | 从这里开始 |
|:---|:---|
| **解析已有 HTML** | [查询元素](parsing/selection_CN.md)：CSS、XPath 与基于文本的选择 |
| **快速抓取页面**并原型验证 | 选择 [Fetcher](fetching/choosing_CN.md) 立即测试，或启动[交互式 Shell](cli/interactive-shell_CN.md) |
| **构建可扩展爬虫** | [Spiders](spiders/getting-started_CN.md)：并发、多会话爬取，支持暂停/恢复 |
| **不写代码抓取** | [CLI extract 命令](cli/extract-commands_CN.md)，或将 [MCP 服务器](ai/mcp-server_CN.md) 接入你喜爱的 AI 工具 |
| **从其他库迁移** | [从 BeautifulSoup 迁移](tutorials/migrating_from_beautifulsoup_CN.md) 或 [与 Scrapy 对比](spiders/architecture_CN.md#comparison-with-scrapy) |

---

我们将先快速回顾解析能力，然后使用自定义浏览器获取网站、发起请求并解析响应。

以下是由 ChatGPT 生成的 HTML 文档，将作为本页示例贯穿使用：
```html
<html>
  <head>
    <title>Complex Web Page</title>
    <style>
      .hidden { display: none; }
    </style>
  </head>
  <body>
    <header>
      <nav>
        <ul>
          <li> <a href="#home">Home</a> </li>
          <li> <a href="#about">About</a> </li>
          <li> <a href="#contact">Contact</a> </li>
        </ul>
      </nav>
    </header>
    <main>
      <section id="products" schema='{"jsonable": "data"}'>
        <h2>Products</h2>
        <div class="product-list">
          <article class="product" data-id="1">
            <h3>Product 1</h3>
            <p class="description">This is product 1</p>
            <span class="price">$10.99</span>
            <div class="hidden stock">In stock: 5</div>
          </article>

          <article class="product" data-id="2">
            <h3>Product 2</h3>
            <p class="description">This is product 2</p>
            <span class="price">$20.99</span>
            <div class="hidden stock">In stock: 3</div>
          </article>

          <article class="product" data-id="3">
            <h3>Product 3</h3>
            <p class="description">This is product 3</p>
            <span class="price">$15.99</span>
            <div class="hidden stock">Out of stock</div>
          </article>
        </div>
      </section>
      
      <section id="reviews">
        <h2>Customer Reviews</h2>
        <div class="review-list">
          <div class="review" data-rating="5">
            <p class="review-text">Great product!</p>
            <span class="reviewer">John Doe</span>
          </div>
          <div class="review" data-rating="4">
            <p class="review-text">Good value for money.</p>
            <span class="reviewer">Jane Smith</span>
          </div>
        </div>
      </section>
    </main>
    <script id="page-data" type="application/json">
      {
        "lastUpdated": "2024-09-22T10:30:00Z",
        "totalProducts": 3
      }
    </script>
  </body>
</html>
```
从如下方式加载上述原始 HTML 开始
```python
from scrapling.parser import Selector
page = Selector(html_doc)
page  # <data='<html><head><title>Complex Web Page</tit...'>
```
递归获取页面上所有文本内容
```python
page.get_all_text(ignore_tags=('script', 'style'))
# 'Complex Web Page\nHome\nAbout\nContact\nProducts\nProduct 1\nThis is product 1\n$10.99\nIn stock: 5\nProduct 2\nThis is product 2\n$20.99\nIn stock: 3\nProduct 3\nThis is product 3\n$15.99\nOut of stock\nCustomer Reviews\nGreat product!\nJohn Doe\nGood value for money.\nJane Smith'
```

## 查找元素
只要页面上有你想要的元素，你就能找到它！唯一限制是你的创造力！

查找第一个 HTML `section` 元素
```python
section_element = page.find('section')
# <data='<section id="products" schema='{"jsonabl...' parent='<main><section id="products" schema='{"j...'>
```
查找所有 `section` 元素
```python
section_elements = page.find_all('section')
# [<data='<section id="products" schema='{"jsonabl...' parent='<main><section id="products" schema='{"j...'>, <data='<section id="reviews"><h2>Customer Revie...' parent='<main><section id="products" schema='{"j...'>]
```
查找 `id` 属性值为 `products` 的所有 `section` 元素。
```python
section_elements = page.find_all('section', {'id':"products"})
# 等同于
section_elements = page.find_all('section', id="products")
# [<data='<section id="products" schema='{"jsonabl...' parent='<main><section id="products" schema='{"j...'>]
```
查找 `id` 属性值包含 `product` 的所有 `section` 元素。
```python
section_elements = page.find_all('section', {'id*':"product"})
```
查找文本内容匹配正则 `Product \d` 的所有 `h3` 元素
```python
page.find_all('h3', re.compile(r'Product \d'))
# [<data='<h3>Product 1</h3>' parent='<article class="product" data-id="1"><h3...'>, <data='<h3>Product 2</h3>' parent='<article class="product" data-id="2"><h3...'>, <data='<h3>Product 3</h3>' parent='<article class="product" data-id="3"><h3...'>]
```
查找文本内容仅匹配正则 `Product` 的所有 `h3` 和 `h2` 元素
```python
page.find_all(['h3', 'h2'], re.compile(r'Product'))
# [<data='<h3>Product 1</h3>' parent='<article class="product" data-id="1"><h3...'>, <data='<h3>Product 2</h3>' parent='<article class="product" data-id="2"><h3...'>, <data='<h3>Product 3</h3>' parent='<article class="product" data-id="3"><h3...'>, <data='<h2>Products</h2>' parent='<section id="products" schema='{"jsonabl...'>]
```
查找文本内容恰好为 `Products` 的所有元素（空白字符不计入）
```python
page.find_by_text('Products', first_match=False)
# [<data='<h2>Products</h2>' parent='<section id="products" schema='{"jsonabl...'>]
```
或查找文本内容匹配正则 `Product \d` 的所有元素
```python
page.find_by_regex(r'Product \d', first_match=False)
# [<data='<h3>Product 1</h3>' parent='<article class="product" data-id="1"><h3...'>, <data='<h3>Product 2</h3>' parent='<article class="product" data-id="2"><h3...'>, <data='<h3>Product 3</h3>' parent='<article class="product" data-id="3"><h3...'>]
```
查找与目标元素相似的所有元素
```python
target_element = page.find_by_regex(r'Product \d', first_match=True)
# <data='<h3>Product 1</h3>' parent='<article class="product" data-id="1"><h3...'>
target_element.find_similar()
# [<data='<h3>Product 2</h3>' parent='<article class="product" data-id="2"><h3...'>, <data='<h3>Product 3</h3>' parent='<article class="product" data-id="3"><h3...'>]
```
查找匹配 CSS 选择器的第一个元素
```python
page.css('.product-list [data-id="1"]')[0]
# <data='<article class="product" data-id="1"><h3...' parent='<div class="product-list"> <article clas...'>
```
查找匹配 CSS 选择器的所有元素
```python
page.css('.product-list article')
# [<data='<article class="product" data-id="1"><h3...' parent='<div class="product-list"> <article clas...'>, <data='<article class="product" data-id="2"><h3...' parent='<div class="product-list"> <article clas...'>, <data='<article class="product" data-id="3"><h3...' parent='<div class="product-list"> <article clas...'>]
```
查找匹配 XPath 选择器的第一个元素
```python
page.xpath("//*[@id='products']/div/article")[0]
# <data='<article class="product" data-id="1"><h3...' parent='<div class="product-list"> <article clas...'>
```
查找匹配 XPath 选择器的所有元素
```python
page.xpath("//*[@id='products']/div/article")
# [<data='<article class="product" data-id="1"><h3...' parent='<div class="product-list"> <article clas...'>, <data='<article class="product" data-id="2"><h3...' parent='<div class="product-list"> <article clas...'>, <data='<article class="product" data-id="3"><h3...' parent='<div class="product-list"> <article clas...'>]
```

以上只是这些功能的冰山一角；这些选择方法的更多高级选项将在后文展示。
## 访问元素数据
非常简单：
```python
>>> section_element.tag
'section'
>>> print(section_element.attrib)
{'id': 'products', 'schema': '{"jsonable": "data"}'}
>>> section_element.attrib['schema'].json()  # 若属性值可转换为 json，则使用 `.json()` 转换
{'jsonable': 'data'}
>>> section_element.text  # 直接文本内容
''
>>> section_element.get_all_text()  # 递归获取所有文本内容
'Products\nProduct 1\nThis is product 1\n$10.99\nIn stock: 5\nProduct 2\nThis is product 2\n$20.99\nIn stock: 3\nProduct 3\nThis is product 3\n$15.99\nOut of stock'
>>> section_element.html_content  # 元素的 HTML 内容
'<section id="products" schema=\'{"jsonable": "data"}\'><h2>Products</h2>\n        <div class="product-list">\n          <article class="product" data-id="1"><h3>Product 1</h3>\n            <p class="description">This is product 1</p>\n            <span class="price">$10.99</span>\n            <div class="hidden stock">In stock: 5</div>\n          </article><article class="product" data-id="2"><h3>Product 2</h3>\n            <p class="description">This is product 2</p>\n            <span class="price">$20.99</span>\n            <div class="hidden stock">In stock: 3</div>\n          </article><article class="product" data-id="3"><h3>Product 3</h3>\n            <p class="description">This is product 3</p>\n            <span class="price">$15.99</span>\n            <div class="hidden stock">Out of stock</div>\n          </article></div>\n      </section>'
>>> print(section_element.prettify())  # 美化后的版本
'''
<section id="products" schema='{"jsonable": "data"}'><h2>Products</h2>
    <div class="product-list">
      <article class="product" data-id="1"><h3>Product 1</h3>
        <p class="description">This is product 1</p>
        <span class="price">$10.99</span>
        <div class="hidden stock">In stock: 5</div>
      </article><article class="product" data-id="2"><h3>Product 2</h3>
        <p class="description">This is product 2</p>
        <span class="price">$20.99</span>
        <div class="hidden stock">In stock: 3</div>
      </article><article class="product" data-id="3"><h3>Product 3</h3>
        <p class="description">This is product 3</p>
        <span class="price">$15.99</span>
        <div class="hidden stock">Out of stock</div>
      </article>
    </div>
</section>
'''
>>> section_element.path  # 该元素在 DOM 树中的所有祖先
[<data='<main><section id="products" schema='{"j...' parent='<body> <header><nav><ul><li> <a href="#h...'>,
 <data='<body> <header><nav><ul><li> <a href="#h...' parent='<html><head><title>Complex Web Page</tit...'>,
 <data='<html><head><title>Complex Web Page</tit...'>]
>>> section_element.generate_css_selector
'#products'
>>> section_element.generate_full_css_selector
'body > main > #products > #products'
>>> section_element.generate_xpath_selector
"//*[@id='products']"
>>> section_element.generate_full_xpath_selector
"//body/main/*[@id='products']"
```

## 导航
使用上文找到的元素

```python
>>> section_element.parent
<data='<main><section id="products" schema='{"j...' parent='<body> <header><nav><ul><li> <a href="#h...'>
>>> section_element.parent.tag
'main'
>>> section_element.parent.parent.tag
'body'
>>> section_element.children
[<data='<h2>Products</h2>' parent='<section id="products" schema='{"jsonabl...'>,
 <data='<div class="product-list"> <article clas...' parent='<section id="products" schema='{"jsonabl...'>]
>>> section_element.siblings
[<data='<section id="reviews"><h2>Customer Revie...' parent='<main><section id="products" schema='{"j...'>]
>>> section_element.next  # 获取下一个元素，逻辑与 `quote.previous` 相同。
<data='<section id="reviews"><h2>Customer Revie...' parent='<main><section id="products" schema='{"j...'>
>>> section_element.children.css('h2::text').getall()
['Products']
>>> page.css('[data-id="1"]')[0].has_class('product')
True
```
若需要超出元素父级之外的操作，可遍历任意元素的完整祖先树，如下所示
```python
for ancestor in section_element.iterancestors():
    # do something with it...
```
可搜索满足条件的特定祖先元素；只需传入一个以 `Selector` 对象为参数、条件满足时返回 `True`、否则返回 `False` 的函数，如下：
```python
>>> section_element.find_ancestor(lambda ancestor: ancestor.css('nav'))
<data='<body> <header><nav><ul><li> <a href="#h...' parent='<html><head><title>Complex Web Page</tit...'>
```

## 获取网站
除了将原始 HTML 传给 Scrapling，你也可以通过 HTTP 请求或浏览器直接获取网站响应。

每种用例都有对应的 Fetcher。

### HTTP 请求
对于简单 HTTP 请求，可使用 `Fetcher` 类，如下导入并使用：
```python
from scrapling.fetchers import Fetcher
page = Fetcher.get('https://scrapling.requestcatcher.com/get', impersonate="chrome")
```
在此基础上，所有 HTTP 方法用法如下：
```python
from scrapling.fetchers import Fetcher
page = Fetcher.get('https://scrapling.requestcatcher.com/get', stealthy_headers=True)
page = Fetcher.post('https://scrapling.requestcatcher.com/post', data={'key': 'value'}, proxy='http://username:password@localhost:8030')
page = Fetcher.put('https://scrapling.requestcatcher.com/put', data={'key': 'value'})
page = Fetcher.delete('https://scrapling.requestcatcher.com/delete')
```
对于异步请求，将导入替换为：
```python
from scrapling.fetchers import AsyncFetcher
page = await AsyncFetcher.get('https://scrapling.requestcatcher.com/get', stealthy_headers=True)
page = await AsyncFetcher.post('https://scrapling.requestcatcher.com/post', data={'key': 'value'}, proxy='http://username:password@localhost:8030')
page = await AsyncFetcher.put('https://scrapling.requestcatcher.com/put', data={'key': 'value'})
page = await AsyncFetcher.delete('https://scrapling.requestcatcher.com/delete')
```

!!! note "说明："

    1. 有 `stealthy_headers` 参数，启用后请求会生成真实浏览器请求头并使用，包括 Google referer 头。默认启用。
    2. `impersonate` 参数可伪造特定浏览器版本的 TLS 指纹。
    3. 还有 `http3` 参数，启用后 Fetcher 使用 HTTP/3 发起请求，使请求更真实

这只是该 Fetcher 的冰山一角；更多内容见[此处](fetching/static_CN.md)

### 动态加载
若你处理的是当今常见的动态网站，我们也能满足！

`DynamicFetcher` 类（原 `PlayWrightFetcher`）为基于 Chromium 的浏览器获取与加载网页提供多种选项。
```python
from scrapling.fetchers import DynamicFetcher
page = DynamicFetcher.fetch('https://quotes.toscrape.com/js/', disable_resources=True, block_ads=True)
print(len(page.css(".quote")))  # -> 10

# fetch 的异步版本
page = await DynamicFetcher.async_fetch('https://quotes.toscrape.com/js/', disable_resources=True, block_ads=True)
print(len(page.css(".quote")))  # -> 10
```
它构建于 [Playwright](https://playwright.dev/python/) 之上，目前提供两种主要运行选项，可按需组合：

- 除你所选修改外不做任何改动的原生 Playwright。使用 Chromium 浏览器。
- 传入 `real_chrome` 参数使用真实 Chrome 浏览器，或传入 CDP URL 由 Fetcher 控制你的浏览器，多数选项均可启用。


同样，这只是该 Fetcher 的冰山一角。完整参数列表与全部细节见[此处](fetching/dynamic_CN.md)。

### 动态反防护加载
若你处理的是带有烦人反防护的动态网站，我们同样能应对！

`StealthyFetcher` 类使用上文 `DynamicFetcher` 的隐蔽版本。

它所做的一些事情：

1. 可轻松自动绕过所有类型的 Cloudflare Turnstile/Interstitial。
2. 绕过 CDP 运行时泄漏与 WebRTC 泄漏。
3. 隔离 JS 执行，移除许多 Playwright 指纹，并阻止通过部分已知机器人行为进行的检测。
4. 生成 canvas 噪声以防止通过 canvas 进行指纹识别。
5. 自动修补已知的无头模式检测方法，并提供选项以抵御时区不匹配攻击。
6. 以及其他反防护选项...

```python
from scrapling.fetchers import StealthyFetcher
page = StealthyFetcher.fetch('https://www.browserscan.net/bot-detection')  # 默认无头运行
page.status == 200  # -> True

page = StealthyFetcher.fetch('https://nopecha.com/demo/cloudflare', solve_cloudflare=True)  # 若出现 Cloudflare 验证码则自动解决
page.status == 200  # -> True

page = StealthyFetcher.fetch('https://www.browserscan.net/bot-detection', block_webrtc=True, hide_canvas=True, dns_over_https=True) # 以及其余参数...
# fetch 的异步版本
page = await StealthyFetcher.async_fetch('https://www.browserscan.net/bot-detection')
page.status == 200  # -> True
```

同样，这只是该 Fetcher 的冰山一角。完整参数列表与全部细节见[此处](fetching/stealthy_CN.md)。

---

以上就是 Scrapling 概览。若想了解更多，请继续阅读下一节。
