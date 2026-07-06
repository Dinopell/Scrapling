# 查询元素
Scrapling 目前仅支持解析 HTML 页面（不支持 XML 订阅源），因为自适应（adaptive）功能无法与 XML 配合使用。

在 Scrapling 中，有五种主要方式查找元素：

1. CSS3 选择器
2. XPath 选择器
3. 基于过滤器/条件查找元素
4. 查找直接文本内容包含特定文本的元素
5. 查找直接文本内容匹配特定正则表达式的元素

还有其他间接查找元素的方式。Scrapling 也可以查找与给定元素相似的元素；参见[查找相似元素](#finding-similar-elements)。

## CSS/XPath 选择器

### 什么是 CSS 选择器？
[CSS](https://en.wikipedia.org/wiki/CSS) 是一种为 HTML 文档应用样式的语言。它定义了选择器，用于将样式与特定的 HTML 元素关联。

Scrapling 实现了 [W3C 规范](http://www.w3.org/TR/2011/REC-css3-selectors-20110929/) 中描述的 CSS3 选择器。CSS 选择器支持来自 `cssselect`，因此最好阅读 [cssselect 支持哪些选择器](https://cssselect.readthedocs.io/en/latest/#supported-selectors) 以及伪函数/伪元素的相关说明。

此外，Scrapling 还实现了一些非标准伪元素，例如：

* 选择文本节点，使用 ``::text``。
* 选择属性值，使用 ``::attr(name)``，其中 name 是你想要获取值的属性名

选择器逻辑遵循与 Scrapy/Parsel 相同的约定。

使用 CSS 选择器选择元素时，请使用 `css` 方法，该方法返回 `Selectors`。使用 `[0]` 获取第一个元素，或使用 `.get()` / `.getall()` 从文本/属性伪选择器中提取文本值。

### 什么是 XPath 选择器？
[XPath](https://en.wikipedia.org/wiki/XPath) 是一种在 XML 文档中选择节点的语言，也可用于 HTML。[这份速查表](https://devhints.io/xpath) 是学习 [XPath](https://en.wikipedia.org/wiki/XPath) 的好资源。Scrapling 通过 [lxml](https://lxml.de/) 直接添加了 XPath 选择器。

逻辑遵循与 Scrapy/Parsel 相同的约定。不过，Scrapling 并未像 Scrapy/Parsel 那样实现 XPath 扩展函数 `has-class`，而是在返回的元素上提供了 `has_class` 方法。

使用 XPath 选择器选择元素时，请使用 `xpath` 方法，其逻辑与上述 CSS 选择器方法相同。

> 注意，`css` 和 `xpath` 的每个方法都有额外参数，但此处未作说明，因为它们都与自适应（adaptive）功能相关。自适应功能将在后续单独页面中详细描述。

### 选择器示例
下面看一些 CSS 和 XPath 选择器的通用示例。

选择所有 class 为 `product` 的元素。
```python
products = page.css('.product')
products = page.xpath('//*[@class="product"]')
```
**注意：** 如果存在其他 class，XPath 版本可能不准确；按 class 选择时始终优先使用 CSS。

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
这与以下写法相同
```python
title = page.css('h1')[0].text
title = page.xpath('//h1')[0].text
```
获取第一个 `a` 标签元素的 `href` 属性
```python
link = page.css('a::attr(href)').get()
link = page.xpath('//a/@href').get()
```
选择 class 为 `product` 的元素下、文本包含 `Phone` 的第一个 `h1` 标签元素的文本。
```python
title = page.css('.product h1:contains("Phone")::text').get()
title = page.xpath('//*[@class="product"]//h1[contains(text(),"Phone")]/text()').get()
```
只要返回结果，你可以任意嵌套和链式调用选择器
```python
page.css('.product')[0].css('h1:contains("Phone")::text').get()
page.xpath('//*[@class="product"]')[0].xpath('//h1[contains(text(),"Phone")]/text()').get()
page.xpath('//*[@class="product"]')[0].css('h1:contains("Phone")::text').get()
```
另一个示例

所有 `href` 属性中包含 'image' 的链接
```python
links = page.css('a[href*="image"]')
links = page.xpath('//a[contains(@href, "image")]')
for index, link in enumerate(links):
    link_value = link.attrib['href']  # Cleaner than link.css('::attr(href)').get()
    link_text = link.text
    print(f'Link number {index} points to this url {link_value} with text content as "{link_text}"')
```

## 基于文本内容的选择
Scrapling 提供两种方式，根据元素的直接文本内容选择元素：

1. 通过 `find_by_text` 方法，以多种选项选择直接文本内容包含给定文本的元素。
2. 通过 `find_by_regex` 方法，以多种选项选择直接文本内容匹配给定正则表达式的元素。

`find_by_text` 能实现的功能，`find_by_regex` 也能实现，但两者都提供是为了方便使用。

使用 `find_by_text` 时，文本作为第一个参数传入；使用 `find_by_regex` 时，正则表达式作为第一个参数传入。两种方法共享以下参数：

* **first_match**：若为 `True`（默认值），方法将返回找到的第一个结果。
* **case_sensitive**：若为 `True`，将区分字母大小写。
* **clean_match**：若为 `True`，匹配前会将所有空白和连续空格替换为单个空格。

默认情况下，Scrapling 会精确匹配传入 `find_by_text` 的文本/模式，因此目标元素的文本内容必须**仅**为你输入的文本，但为此还提供了一个额外参数：

* **partial**：若启用，`find_by_text` 将返回包含输入文本的元素，即不再要求精确匹配

**注意：** `find_by_regex` 方法的第一个参数既可以是普通字符串，也可以是已编译的正则表达式。

### 查找相似元素
Scrapling 可以查找与给定元素相似的元素，灵感来自 AutoScraper 库，但可用于通过任意方法找到的元素。

给定一个元素（例如通过标题找到的商品），在其上调用 `.find_similar()` 后，Scrapling 会：

1. 查找页面上所有与该元素 DOM 树深度相同的元素。
2. 检查所有找到的元素，丢弃标签名、父标签名、祖父标签名不一致的元素。
3. 作为最终检查，Scrapling 使用模糊匹配丢弃属性与原始元素属性不相似的元素。此步骤可通过可配置的百分比控制（见下方参数）。

`find_similar()` 的参数：

* **similarity_threshold**：比较元素属性的相似度百分比（第 3 步）。默认值为 0.2（标签属性至少需 20% 相似）。设为 0 可完全禁用此检查。
* **ignore_attributes**：传入的属性名在最后一步匹配属性时将被忽略。默认值为 `('href', 'src',)`，因为 URL 在不同元素间可能差异很大，不可靠。
* **match_text**：若为 `True`，匹配时将考虑元素的文本内容（第 3 步）。在典型场景中不推荐使用此参数，但视情况而定。

### 示例
通过原始文本、正则表达式和 `find_similar` 查找元素的示例。
```python
from scrapling.fetchers import Fetcher
page = Fetcher.get('https://books.toscrape.com/index.html')
```
查找文本完全匹配此文本的第一个元素
```python
>>> page.find_by_text('Tipping the Velvet')
<data='<a href="catalogue/tipping-the-velvet_99...' parent='<h3><a href="catalogue/tipping-the-velve...'>
```
结合 `page.urljoin` 从相对 `href` 返回完整 URL。
```python
>>> page.find_by_text('Tipping the Velvet').attrib['href']
'catalogue/tipping-the-velvet_999/index.html'
>>> page.urljoin(page.find_by_text('Tipping the Velvet').attrib['href'])
'https://books.toscrape.com/catalogue/tipping-the-velvet_999/index.html'
```
若有多个匹配，获取全部（注意返回的是列表）
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
搜索默认不区分大小写，因此上述结果包含 `The`，而不仅是小写 `the`。若限制为精确大小写：
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
传入已编译的正则表达式效果相同；Scrapling 会检测输入类型并据此处理：
```python
>>> import re
>>> regex = re.compile(r'£[\d\.]+')
>>> page.find_by_regex(regex)
<data='<p class="price_color">£51.77</p>' parent='<div class="product_price"> <p class="pr...'>
>>> page.find_by_regex(regex).text
'£51.77'
```
获取所有匹配该正则的元素
```python
>>> page.find_by_regex(r'£[\d\.]+', first_match=False)
[<data='<p class="price_color">£51.77</p>' parent='<div class="product_price"> <p class="pr...'>,
 <data='<p class="price_color">£53.74</p>' parent='<div class="product_price"> <p class="pr...'>,
 <data='<p class="price_color">£50.10</p>' parent='<div class="product_price"> <p class="pr...'>,
 <data='<p class="price_color">£47.82</p>' parent='<div class="product_price"> <p class="pr...'>,
 ...]
```
以此类推...

查找与当前元素在位置和属性上相似的所有元素。本例中，匹配时忽略 'title' 属性
```python
>>> element = page.find_by_text('Tipping the Velvet')
>>> element.find_similar(ignore_attributes=['title'])
[<data='<a href="catalogue/a-light-in-the-attic_...' parent='<h3><a href="catalogue/a-light-in-the-at...'>,
 <data='<a href="catalogue/soumission_998/index....' parent='<h3><a href="catalogue/soumission_998/in...'>,
 <data='<a href="catalogue/sharp-objects_997/ind...' parent='<h3><a href="catalogue/sharp-objects_997...'>,
...]
```
元素数量为 19 而非 20，因为当前元素不包含在结果中：
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
以该元素为起点获取所有图书数据：
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
使用 `find_similar` 方法的高级示例：

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
受 BeautifulSoup 的 `find_all` 函数启发，可以使用 `find_all` 和 `find` 方法查找元素。两种方法都接受多个过滤器，并返回页面上所有满足全部过滤条件的元素。

更具体地说：

* 传入的任何字符串都被视为标签名。
* 传入的任何可迭代对象（如 List/Tuple/Set）都被视为标签名的可迭代集合。
* 任何字典都被视为 HTML 元素、属性名与属性值的映射。
* 传入的任何正则表达式用于按内容过滤元素，与 `find_by_regex` 方法类似
* 传入的任何函数用于过滤元素
* 传入的任何关键字参数都被视为 HTML 元素属性及其值。

它会收集所有传入的参数和关键字，每个过滤器以瀑布式过滤系统将结果传递给下一个过滤器。

它按以下顺序过滤当前页面/元素中的所有元素：

1. 收集所有具有传入标签名的元素。
2. 收集所有匹配全部传入属性的元素；若已使用上一过滤器，则对先前收集的元素进行过滤。
3. 收集所有匹配全部传入正则表达式的元素；若已使用上一过滤器，则对先前收集的元素进行过滤。
4. 收集所有满足全部传入函数的元素；若已使用上一过滤器，则对先前收集的元素进行过滤。

**注意：**

1. 过滤过程始终从上述过滤顺序中找到的第一个过滤器开始。若未传入标签名但传入了属性，则从第 2 步开始，以此类推。
2. 传入参数的顺序无关紧要。唯一考虑的顺序是上文说明的顺序。

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
与上面相同。
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
查找 class 等于 `quote` 且包含 `.text` 元素（其内容包含单词 'world'）的所有 div 元素。
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
查找内容中包含单词 'world' 的所有元素。
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
查找 class 为 'quote' 的所有 div 和 span 元素（没有这样的 span，因此只返回 div）
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
进阶技巧：查找 `href` 属性值以 'Einstein' 结尾的所有元素。
```python
>>> page.find_all({'href$': 'Einstein'})
[<data='<a href="/author/Albert-Einstein">(about...' parent='<span>by <small class="author" itemprop=...'>,
 <data='<a href="/author/Albert-Einstein">(about...' parent='<span>by <small class="author" itemprop=...'>,
 <data='<a href="/author/Albert-Einstein">(about...' parent='<span>by <small class="author" itemprop=...'>]
```
另一个进阶技巧：查找 `href` 属性值包含 '/author/' 的所有元素
```python
>>> page.find_all({'href*': '/author/'})
[<data='<a href="/author/Albert-Einstein">(about...' parent='<span>by <small class="author" itemprop=...'>,
 <data='<a href="/author/J-K-Rowling">(about)</a...' parent='<span>by <small class="author" itemprop=...'>,
 <data='<a href="/author/Albert-Einstein">(about...' parent='<span>by <small class="author" itemprop=...'>,
...]
```
以此类推...

## 生成选择器
无论通过何种方法找到元素，都可以为其生成 CSS/XPath 选择器。

为 `url_element` 元素生成短 CSS 选择器（若可能则生成短选择器，否则为完整选择器）
```python
>>> url_element = page.find({'href*': '/author/'})
>>> url_element.generate_css_selector
'body > div > div:nth-of-type(2) > div > div > span:nth-of-type(2) > a'
```
从页面起点为 `url_element` 元素生成完整 CSS 选择器
```python
>>> url_element.generate_full_css_selector
'body > div > div:nth-of-type(2) > div > div > span:nth-of-type(2) > a'
```
为 `url_element` 元素生成短 XPath 选择器（若可能则生成短选择器，否则为完整选择器）
```python
>>> url_element.generate_xpath_selector
'//body/div/div[2]/div/div/span[2]/a'
```
从页面起点为 `url_element` 元素生成完整 XPath 选择器
```python
>>> url_element.generate_full_xpath_selector
'//body/div/div[2]/div/div/span[2]/a'
```
**注意：** 生成短选择器时，Scrapling 会尝试找到唯一元素（例如带有 `id` 属性的元素）作为停止点。若不存在，短选择器与完整选择器将相同。

## 将选择器与正则表达式配合使用
与 `parsel`/`scrapy` 类似，`re` 和 `re_first` 方法可用于通过正则表达式提取数据。这些方法存在于 `Selector`、`Selectors`、`TextHandler` 和 `TextHandlers` 中，因此即使未选择文本节点，也可直接在元素上使用。详见 [TextHandler](main_classes_CN.md#texthandler) 类。

示例：
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
有关正则方法的更多详情，请参阅 [TextHandler](main_classes_CN.md#texthandler) 类。
