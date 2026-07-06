# 查询元素
Scrapling 目前仅支持解析 HTML 页面，不支持 XML 数据源。做出这一决定是因为自适应特性不适用于 XML，但未来可能改变，敬请期待 :)

在 Scrapling 中，有五种主要方式查找元素：

1. CSS3 选择器
2. XPath 选择器
3. 基于过滤器/条件查找元素
4. 查找内容包含特定文本的元素
5. 查找内容匹配特定正则的元素

当然还有其他间接方式，此处将详细讨论主要方法。我们还会介绍 Scrapling 最突出的特性之一：查找与已有元素相似的元素；可直接跳转到[该节](#finding-similar-elements)。

若你是网络爬虫新手、几乎不会写选择器且想快速上手，建议直接学习 `find`/`find_all` 方法，见[此处](#filters-based-searching)。

## CSS/XPath 选择器

### 什么是 CSS 选择器？
[CSS](https://en.wikipedia.org/wiki/CSS) 是用于为 HTML 文档应用样式的语言，它定义选择器以将样式与特定 HTML 元素关联。

Scrapling 实现了 [W3C 规范](http://www.w3.org/TR/2011/REC-css3-selectors-20110929/) 描述的 CSS3 选择器。CSS 选择器支持来自 `cssselect`，建议阅读 [cssselect 支持的选择器](https://cssselect.readthedocs.io/en/latest/#supported-selectors) 及伪函数/伪元素。

此外，Scrapling 实现了一些非标准伪元素：

* 选择文本节点：使用 ``::text``。
* 选择属性值：使用 ``::attr(name)``，其中 name 为所需属性名

简而言之，若你来自 Scrapy/Parsel，此处选择器逻辑相同，便于上手，无需适应陌生写法 :)

使用 CSS 选择器选择元素时，调用 `css` 方法，返回 `Selectors`。用 `[0]` 获取第一个元素，或用 `.get()` / `.getall()` 从文本/属性伪选择器提取文本值。

### 什么是 XPath 选择器？
[XPath](https://en.wikipedia.org/wiki/XPath) 是用于在 XML 文档中选择节点的语言，也可用于 HTML。这份[速查表](https://devhints.io/xpath)是学习 [XPath](https://en.wikipedia.org/wiki/XPath) 的好资源。Scrapling 通过 [lxml](https://lxml.de/) 直接支持 XPath 选择器。

简而言之，与 CSS 选择器情况相同；若你来自 Scrapy/Parsel，此处逻辑一致。但 Scrapling 未像 Scrapy/Parsel 那样实现 XPath 扩展函数 `has-class`，而是提供 `has_class` 方法，可在返回的元素上使用，达到相同目的。

使用 XPath 选择器时，调用 `xpath` 方法，逻辑与上述 CSS 方法相同。

> 注意：`css` 和 `xpath` 各方法都有额外参数，此处未说明，因为它们都与自适应特性相关。自适应特性将有专门页面详细描述。

### 选择器示例
下面是 CSS 与 XPath 选择器的共用示例。

选择所有 class 为 `product` 的元素。
```python
products = page.css('.product')
products = page.xpath('//*[@class="product"]')
```
!!! info "注意："

    若存在其他 class，XPath 方式可能不准确；**按 class 选择时始终优先使用 CSS**

选择第一个 class 为 `product` 的元素。
```python
product = page.css('.product')[0]
product = page.xpath('//*[@class="product"]')[0]
```
获取第一个 `h1` 标签元素的文本
```python
title = page.css('h1::text').get()
title = page.xpath('//h1//text()').get()
```
等价于
```python
title = page.css('h1')[0].text
title = page.xpath('//h1')[0].text
```
获取第一个 `a` 标签元素的 `href` 属性
```python
link = page.css('a::attr(href)').get()
link = page.xpath('//a/@href').get()
```
选择 class 为 `product` 的元素下、文本包含 `Phone` 的第一个 `h1` 的文本。
```python
title = page.css('.product h1:contains("Phone")::text').get()
title = page.xpath('//*[@class="product"]//h1[contains(text(),"Phone")]/text()').get()
```
只要有返回结果，可任意嵌套和链式选择器
```python
page.css('.product')[0].css('h1:contains("Phone")::text').get()
page.xpath('//*[@class="product"]')[0].xpath('//h1[contains(text(),"Phone")]/text()').get()
page.xpath('//*[@class="product"]')[0].css('h1:contains("Phone")::text').get()
```
另一示例

所有 `href` 属性包含 `image` 的链接
```python
links = page.css('a[href*="image"]')
links = page.xpath('//a[contains(@href, "image")]')
for index, link in enumerate(links):
    link_value = link.attrib['href']  # Cleaner than link.css('::attr(href)').get()
    link_text = link.text
    print(f'Link number {index} points to this url {link_value} with text content as "{link_text}"')
```

## 文本内容选择
Scrapling 支持根据元素的直接文本内容选择，有两种方式：

1. 通过 `find_by_text` 方法，以多种选项查找直接文本内容包含给定文本的元素。
2. 通过 `find_by_regex` 方法，以多种选项查找直接文本内容匹配给定正则模式的元素。

若你精通正则表达式，`find_by_text` 能做的 `find_by_regex` 也能做，但我们提供更多选项以便所有用户使用。

`find_by_text` 第一个参数为文本；`find_by_regex` 第一个参数为正则模式。两方法共享以下参数：

* **first_match**：若为 `True`（默认），方法返回找到的第一个结果。
* **case_sensitive**：若为 `True`，区分字母大小写。
* **clean_match**：若为 `True`，匹配前将所有空白和连续空格替换为单个空格。

默认情况下，Scrapling 在 `find_by_text` 中搜索精确匹配，即目标元素的文本内容必须**仅**为你输入的文本，因此还有额外参数：

* **partial**：启用后，`find_by_text` 返回包含输入文本的元素，不再要求精确匹配

!!! abstract "注意："

    `find_by_regex` 的第一个参数可以是普通字符串或已编译的正则模式，如下文示例所示。

### 查找相似元素
Scrapling 最突出的新特性之一，是能查找与当前元素相似的元素。该特性灵感来自 AutoScraper 库，但在 Scrapling 中可用于通过任意方法找到的元素。多数用法可能发生在通过文本内容找到元素之后，与 AutoScraper 类似，因此在此说明很方便。

它是如何工作的？

假设你通过标题找到了一个商品，想提取同一表格/容器中的其他商品。对已有元素调用 `.find_similar()`，Scrapling 将：

1. 查找页面上与该元素 DOM 树深度相同的所有元素。
2. 检查所有找到的元素，丢弃标签名、父标签名、祖父标签名不一致者。
3. 此时我们基本（约 99%）确定这些就是目标元素；最后 Scrapling 用模糊匹配丢弃属性与目标元素不像的元素。此步骤有百分比可调，除非默认设置拿不到想要的元素，否则不建议修改。

说得较多，但需深入说明。下一节有使用示例；先介绍可传入的参数：

* **similarity_threshold**：步骤 3 中比较元素属性的相似度百分比，默认 0.2。简单说，两元素标签属性至少应 20% 相似。若要关闭此检查（即跳过步骤 3），可设为 0，但建议先了解其他参数。
* **ignore_attributes**：匹配时忽略的属性名，默认 `('href', 'src',)`，因为 URL 在不同元素间可能差异很大，不可靠。
* **match_text**：若为 `True`，匹配时会考虑元素文本内容（步骤 3）。典型场景不推荐使用，视情况而定。

下面看示例。

### 示例
下面是使用原始文本和正则查找元素的共用示例。

示例将使用 `Fetcher` 类，稍后详细说明。
```python
from scrapling.fetchers import Fetcher
page = Fetcher.get('https://books.toscrape.com/index.html')
```
查找文本完全匹配的第一个元素
```python
>>> page.find_by_text('Tipping the Velvet')
<data='<a href="catalogue/tipping-the-velvet_99...' parent='<h3><a href="catalogue/tipping-the-velve...'>
```
结合 `page.urljoin` 从相对 `href` 得到完整 URL。
```python
>>> page.find_by_text('Tipping the Velvet').attrib['href']
'catalogue/tipping-the-velvet_999/index.html'
>>> page.urljoin(page.find_by_text('Tipping the Velvet').attrib['href'])
'https://books.toscrape.com/catalogue/tipping-the-velvet_999/index.html'
```
若有多个匹配，获取全部（注意返回列表）
```python
>>> page.find_by_text('Tipping the Velvet', first_match=False)
[<data='<a href="catalogue/tipping-the-velvet_99...' parent='<h3><a href="catalogue/tipping-the-velve...'>]
```
获取所有包含单词 `the` 的元素（部分匹配）
```python
>>> results = page.find_by_text('the', partial=True, first_match=False)
>>> [i.text for i in results]
['A Light in the ...',
 'Tipping the Velvet',
 'The Requiem Red',
 'The Dirty Little Secrets ...',
 'The Coming Woman: A ...',
 'The Boys in the ...',
 'The Black Maria',
 'Mesaerion: The Best Science ...',
 "It's Only the Himalayas"]
```
搜索不区分大小写，因此结果包含 `The`，不仅是小写 `the`；将搜索限制为仅含 `the` 的元素。
```python
>>> results = page.find_by_text('the', partial=True, first_match=False, case_sensitive=True)
>>> [i.text for i in results]
['A Light in the ...',
 'Tipping the Velvet',
 'The Boys in the ...',
 "It's Only the Himalayas"]
```
获取文本内容匹配价格正则的第一个元素
```python
>>> page.find_by_regex(r'£[\d\.]+')
<data='<p class="price_color">£51.77</p>' parent='<div class="product_price"> <p class="pr...'>
>>> page.find_by_regex(r'£[\d\.]+').text
'£51.77'
```
传入已编译正则效果相同；Scrapling 会检测输入类型并相应处理：
```python
>>> import re
>>> regex = re.compile(r'£[\d\.]+')
>>> page.find_by_regex(regex)
<data='<p class="price_color">£51.77</p>' parent='<div class="product_price"> <p class="pr...'>
>>> page.find_by_regex(regex).text
'£51.77'
```
获取所有匹配正则的元素
```python
>>> page.find_by_regex(r'£[\d\.]+', first_match=False)
[<data='<p class="price_color">£51.77</p>' parent='<div class="product_price"> <p class="pr...'>,
 <data='<p class="price_color">£53.74</p>' parent='<div class="product_price"> <p class="pr...'>,
 <data='<p class="price_color">£50.10</p>' parent='<div class="product_price"> <p class="pr...'>,
 <data='<p class="price_color">£47.82</p>' parent='<div class="product_price"> <p class="pr...'>,
 ...]
```
以此类推...

查找与当前元素在位置和属性上相似的所有元素。本例匹配时忽略 `title` 属性
```python
>>> element = page.find_by_text('Tipping the Velvet')
>>> element.find_similar(ignore_attributes=['title'])
[<data='<a href="catalogue/a-light-in-the-attic_...' parent='<h3><a href="catalogue/a-light-in-the-at...'>,
 <data='<a href="catalogue/soumission_998/index....' parent='<h3><a href="catalogue/soumission_998/in...'>,
 <data='<a href="catalogue/sharp-objects_997/ind...' parent='<h3><a href="catalogue/sharp-objects_997...'>,
...]
```
注意元素数量为 19 而非 20，因为当前元素不包含在结果中。
```python
>>> len(element.find_similar(ignore_attributes=['title']))
19
```
从所有相似元素获取 `href` 属性
```python
>>> [
    element.attrib['href']
    for element in element.find_similar(ignore_attributes=['title'])
]
['catalogue/a-light-in-the-attic_1000/index.html',
 'catalogue/soumission_998/index.html',
 'catalogue/sharp-objects_997/index.html',
 ...]
```
稍增复杂度：假设要以该元素为起点获取所有图书数据
```python
>>> for product in element.parent.parent.find_similar():
        print({
            "name": product.css('h3 a::text').get(),
            "price": product.css('.price_color')[0].re_first(r'[\d\.]+'),
            "stock": product.css('.availability::text').getall()[-1].clean()
        })
{'name': 'A Light in the ...', 'price': '51.77', 'stock': 'In stock'}
{'name': 'Soumission', 'price': '50.10', 'stock': 'In stock'}
{'name': 'Sharp Objects', 'price': '47.82', 'stock': 'In stock'}
...
```
### 高级示例
更多使用 `find_similar` 的高级或真实场景示例。

电商商品提取
```python
def extract_product_grid(page):
    # Find the first product card
    first_product = page.find_by_text('Add to Cart').find_ancestor(
        lambda e: e.has_class('product-card')
    )

    # Find similar product cards
    products = first_product.find_similar()

    return [
        {
            'name': p.css('h3::text').get(),
            'price': p.css('.price::text').re_first(r'\d+\.\d{2}'),
            'stock': 'In stock' in p.text,
            'rating': p.css('.rating')[0].attrib.get('data-rating')
        }
        for p in products
    ]
```
表格行提取
```python
def extract_table_data(page):
    # Find the first data row
    first_row = page.css('table tbody tr')[0]

    # Find similar rows
    rows = first_row.find_similar()

    return [
        {
            'column1': row.css('td:nth-child(1)::text').get(),
            'column2': row.css('td:nth-child(2)::text').get(),
            'column3': row.css('td:nth-child(3)::text').get()
        }
        for row in rows
    ]
```
表单字段提取
```python
def extract_form_fields(page):
    # Find first form field container
    first_field = page.css('input')[0].find_ancestor(
        lambda e: e.has_class('form-field')
    )

    # Find similar field containers
    fields = first_field.find_similar()

    return [
        {
            'label': f.css('label::text').get(),
            'type': f.css('input')[0].attrib.get('type'),
            'required': 'required' in f.css('input')[0].attrib
        }
        for f in fields
    ]
```
从网站提取评论
```python
def extract_reviews(page):
    # Find first review
    first_review = page.find_by_text('Great product!')
    review_container = first_review.find_ancestor(
        lambda e: e.has_class('review')
    )
    
    # Find similar reviews
    all_reviews = review_container.find_similar()
    
    return [
        {
            'text': r.css('.review-text::text').get(),
            'rating': r.attrib.get('data-rating'),
            'author': r.css('.reviewer::text').get()
        }
        for r in all_reviews
    ]
```
## 基于过滤器的搜索
可以说，这是在 Scrapling 中查找元素的最佳方式，既强大又比写选择器更易让爬虫新手学习。

受 BeautifulSoup 的 `find_all` 启发，可用 `find_all` 和 `find` 查找元素。两方法均可接受多个过滤器，返回页面上同时满足所有过滤条件的元素。

更具体地说：

* 传入的字符串视为标签名。
* 传入的可迭代对象（List/Tuple/Set）视为标签名集合。
* 传入的字典视为 HTML 元素属性名与属性值的映射。
* 传入的正则模式用于按内容过滤元素，类似 `find_by_regex` 方法
* 传入的函数用于过滤元素
* 传入的关键字参数视为 HTML 元素属性及其值

收集所有传入参数和关键字，各过滤器以瀑布式将结果传给下一个过滤器。

按以下顺序过滤当前页面/元素中的所有元素：

1. 收集所有匹配传入标签名的元素。
2. 收集所有匹配传入属性的元素；若已用上一过滤器，则在上次结果上过滤。
3. 收集所有匹配传入正则模式的元素；若已用上一过滤器，则在上次结果上过滤。
4. 收集所有满足传入函数的元素；若已用上一过滤器，则在上次结果上过滤。

!!! note "注意："

    1. 如你所理解，过滤始终从上述顺序中的第一个存在的过滤器开始。若未传标签名但传了属性，则从步骤 2 开始，以此类推。
    2. 传入参数的顺序无关紧要，仅上述顺序有效。

看示例以消除疑惑 :)

### 示例
```python
from scrapling.fetchers import Fetcher
page = Fetcher.get('https://quotes.toscrape.com/')
```
查找所有标签名为 `div` 的元素。
```python
>>> page.find_all('div')
[<data='<div class="container"> <div class="row...' parent='<body> <div class="container"> <div clas...'>,
 <data='<div class="row header-box"> <div class=...' parent='<div class="container"> <div class="row...'>,
...]
```
查找 class 等于 `quote` 的所有 div 元素。
```python
>>> page.find_all('div', class_='quote')
[<data='<div class="quote" itemscope itemtype="h...' parent='<div class="col-md-8"> <div class="quote...'>,
 <data='<div class="quote" itemscope itemtype="h...' parent='<div class="col-md-8"> <div class="quote...'>,
...]
```
同上。
```python
>>> page.find_all('div', {'class': 'quote'})
[<data='<div class="quote" itemscope itemtype="h...' parent='<div class="col-md-8"> <div class="quote...'>,
 <data='<div class="quote" itemscope itemtype="h...' parent='<div class="col-md-8"> <div class="quote...'>,
...]
```
查找 class 等于 `quote` 的所有元素。
```python
>>> page.find_all({'class': 'quote'})
[<data='<div class="quote" itemscope itemtype="h...' parent='<div class="col-md-8"> <div class="quote...'>,
 <data='<div class="quote" itemscope itemtype="h...' parent='<div class="col-md-8"> <div class="quote...'>,
...]
```
查找 class 为 `quote` 且包含 `.text` 子元素、其内容含 `world` 的所有 div。
```python
>>> page.find_all('div', {'class': 'quote'}, lambda e: "world" in e.css('.text::text').get())
[<data='<div class="quote" itemscope itemtype="h...' parent='<div class="col-md-8"> <div class="quote...'>]
```
查找所有有子元素的元素。
```python
>>> page.find_all(lambda element: len(element.children) > 0)
[<data='<html lang="en"><head><meta charset="UTF...'>,
 <data='<head><meta charset="UTF-8"><title>Quote...' parent='<html lang="en"><head><meta charset="UTF...'>,
 <data='<body> <div class="container"> <div clas...' parent='<html lang="en"><head><meta charset="UTF...'>,
...]
```
查找内容包含 `world` 的所有元素。
```python
>>> page.find_all(lambda element: "world" in element.text)
[<data='<span class="text" itemprop="text">“The...' parent='<div class="quote" itemscope itemtype="h...'>,
 <data='<a class="tag" href="/tag/world/page/1/"...' parent='<div class="tags"> Tags: <meta class="ke...'>]
```
查找匹配给定正则的所有 span 元素
```python
>>> page.find_all('span', re.compile(r'world'))
[<data='<span class="text" itemprop="text">“The...' parent='<div class="quote" itemscope itemtype="h...'>]
```
查找 class 为 `quote` 的所有 div 和 span（无此类 span，故仅返回 div）
```python
>>> page.find_all(['div', 'span'], {'class': 'quote'})
[<data='<div class="quote" itemscope itemtype="h...' parent='<div class="col-md-8"> <div class="quote...'>,
 <data='<div class="quote" itemscope itemtype="h...' parent='<div class="col-md-8"> <div class="quote...'>,
...]
```
混合使用
```python
>>> page.find_all({'itemtype':"http://schema.org/CreativeWork"}, 'div').css('.author::text').getall()
['Albert Einstein',
 'J.K. Rowling',
...]
```
进阶技巧：查找 `href` 属性值以 `Einstein` 结尾的所有元素。
```python
>>> page.find_all({'href$': 'Einstein'})
[<data='<a href="/author/Albert-Einstein">(about...' parent='<span>by <small class="author" itemprop=...'>,
 <data='<a href="/author/Albert-Einstein">(about...' parent='<span>by <small class="author" itemprop=...'>,
 <data='<a href="/author/Albert-Einstein">(about...' parent='<span>by <small class="author" itemprop=...'>]
```
另一技巧：查找 `href` 属性值包含 `/author/` 的所有元素
```python
>>> page.find_all({'href*': '/author/'})
[<data='<a href="/author/Albert-Einstein">(about...' parent='<span>by <small class="author" itemprop=...'>,
 <data='<a href="/author/J-K-Rowling">(about)</a...' parent='<span>by <small class="author" itemprop=...'>,
 <data='<a href="/author/Albert-Einstein">(about...' parent='<span>by <small class="author" itemprop=...'>,
...]
```
以此类推...

## 生成选择器
可为任意元素生成可在此或其他地方复用的 CSS/XPath 选择器；无论用什么方法找到该元素都无所谓！

为 `url_element` 生成短 CSS 选择器（若可能则短，否则为完整选择器）
```python
>>> url_element = page.find({'href*': '/author/'})
>>> url_element.generate_css_selector
'body > div > div:nth-of-type(2) > div > div > span:nth-of-type(2) > a'
```
从页面起点为 `url_element` 生成完整 CSS 选择器
```python
>>> url_element.generate_full_css_selector
'body > div > div:nth-of-type(2) > div > div > span:nth-of-type(2) > a'
```
为 `url_element` 生成短 XPath 选择器（若可能则短，否则为完整选择器）
```python
>>> url_element.generate_xpath_selector
'//body/div/div[2]/div/div/span[2]/a'
```
从页面起点为 `url_element` 生成完整 XPath 选择器
```python
>>> url_element.generate_full_xpath_selector
'//body/div/div[2]/div/div/span[2]/a'
```
!!! abstract "注意："

    要求 Scrapling 生成短选择器时，会尝试找唯一元素作为生成止点（如带 `id` 的元素）；本例没有，故短选择与完整选择相同。

## 结合正则使用选择器
与 `parsel`/`scrapy` 类似，可用 `re` 和 `re_first` 通过正则提取数据。但与前者不同，这些方法几乎存在于所有类（`Selector`/`Selectors`/`TextHandler` 和 `TextHandlers`），意味着即使未选择文本节点也可直接在元素上使用。

在说明 [TextHandler](main_classes_CN.md#texthandler) 类时会深入讲解；一般用法如下：
```python
>>> page.css('.price_color')[0].re_first(r'[\d\.]+')
'51.77'

>>> page.css('.price_color').re_first(r'[\d\.]+')
'51.77'

>>> page.css('.price_color').re(r'[\d\.]+')
['51.77',
 '53.74',
 '50.10',
 '47.82',
 '54.23',
...]

>>> page.css('.product_pod h3 a::attr(href)').re(r'catalogue/(.*)/index.html')
['a-light-in-the-attic_1000',
 'tipping-the-velvet_999',
 'soumission_998',
 'sharp-objects_997',
...]

>>> filtering_function = lambda e: e.parent.tag == 'h3' and e.parent.parent.has_class('product_pod')  # As above selector
>>> page.find('a', filtering_function).attrib['href'].re(r'catalogue/(.*)/index.html')
['a-light-in-the-attic_1000']

>>> page.find_by_text('Tipping the Velvet').attrib['href'].re(r'catalogue/(.*)/index.html')
['tipping-the-velvet_999']
```
以此类推。下一页将与 [TextHandler](main_classes_CN.md#texthandler) 类一并更详细说明。
