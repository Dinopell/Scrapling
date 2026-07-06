# 解析核心类

[Selector](#selector) 类是 Scrapling 的核心解析引擎，提供 HTML 解析与元素选择能力。你可以通过以下任一方式导入：
```python
from scrapling import Selector
from scrapling.parser import Selector
```
用法：
```python
page = Selector(
    '<html>...</html>',
    url='https://example.com'
)

# Then select elements as you like
elements = page.css('.product')
```
在 Scrapling 中，传入 HTML 源码或抓取网站后，你主要操作的对象当然是 [Selector](#selector) 对象。你执行的任何操作（如选择、导航等），若结果是页面中的元素/多个元素，将返回 [Selector](#selector) 或 [Selectors](#selectors) 对象，而非文本等类型。

主页面是一个 [Selector](#selector) 对象，其中的元素也是 [Selector](#selector) 对象。任何文本（元素内的文本内容或属性值）都是 [TextHandler](#texthandler) 对象，元素属性则存储为 [AttributesHandler](#attributeshandler)。

## Selector
### 参数说明
最重要的是 `content`，用于传入要解析的 HTML 代码，接受 `str` 或 `bytes` 类型的 HTML 内容。

参数 `url`、`adaptive`、`storage` 和 `storage_args` 是与 `adaptive` 功能配合使用的设置，详见 [adaptive](adaptive_CN.md) 功能页面。

解析调整相关参数：

- **encoding**：解析 HTML 时使用的编码。默认为 `UTF-8`。
- **keep_comments**：告知库解析页面时是否保留 HTML 注释。默认禁用，因为可能在多种情况下影响抓取。
- **keep_cdata**：与 HTML 注释逻辑相同。[cdata](https://stackoverflow.com/questions/7092236/what-is-cdata-in-html) 默认移除，以获得更干净的 HTML。

参数 `huge_tree` 和 `root` 是本文未涉及的高级功能。

主页面及其元素上的大多数属性都是懒加载的（访问时才初始化），这有助于提升 Scrapling 的速度。

### 属性
遍历相关属性见下文 [traversal](#traversal) 部分。

以下 HTML 页面作为示例进行解析：
```html
<html>
  <head>
    <title>Some page</title>
  </head>
  <body>
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

    <script id="page-data" type="application/json">
      {
        "lastUpdated": "2024-09-22T10:30:00Z",
        "totalProducts": 3
      }
    </script>
  </body>
</html>
```
如前所示直接加载页面：
```python
from scrapling import Selector
page = Selector(html_doc)
```
递归获取页面上所有文本内容
```python
>>> page.get_all_text()
'Some page\n\n    \n\n      \nProduct 1\nThis is product 1\n$10.99\nIn stock: 5\nProduct 2\nThis is product 2\n$20.99\nIn stock: 3\nProduct 3\nThis is product 3\n$15.99\nOut of stock'
```
获取第一个 article（全文示例均以此为例）：
```python
article = page.find('article')
```
同样逻辑，递归获取元素上所有文本内容
```python
>>> article.get_all_text()
'Product 1\nThis is product 1\n$10.99\nIn stock: 5'
```
但若尝试获取直接文本内容，将为空，因为上述 HTML 中没有直接文本
```python
>>> article.text
''
```
`get_all_text` 方法有以下可选参数：

1. **separator**：收集到的所有字符串将使用此分隔符拼接。默认为 '\n'。
2. **strip**：若启用，拼接前会对字符串执行 strip。默认禁用。
3. **ignore_tags**：要在最终结果中忽略的所有标签名元组，并忽略嵌套在其中的任何元素。默认为 `('script', 'style',)`。
4. **valid_values**：若启用，方法仅收集有实际值的元素，因此文本为空或仅含空白的元素将被忽略。默认启用

返回的文本是 [TextHandler](#texthandler)，而非普通字符串。若文本内容可序列化为 JSON，可对其调用 `.json()`：
```python
>>> script = page.find('script')
>>> script.json()
{'lastUpdated': '2024-09-22T10:30:00Z', 'totalProducts': 3}
```
继续获取元素标签
```python
>>> article.tag
'article'
```
直接在页面上使用时，操作的是根 `html` 元素：
```python
>>> page.tag
'html'
```
获取元素属性
```python
>>> print(article.attrib)
{'class': 'product', 'data-id': '1'}
```
通过以下任一方式访问特定属性
```python
article.attrib['class']
article.attrib.get('class')
article['class']  # new in v0.3
```
通过以下任一方法检查属性是否包含特定属性
```python
'class' in article.attrib
'class' in article  # new in v0.3
```
获取元素的 HTML 内容
```python
>>> article.html_content
'<article class="product" data-id="1"><h3>Product 1</h3>\n        <p class="description">This is product 1</p>\n        <span class="price">$10.99</span>\n        <div class="hidden stock">In stock: 5</div>\n      </article>'
```
获取元素 HTML 内容的格式化版本
```python
print(article.prettify())
```
```html
<article class="product" data-id="1"><h3>Product 1</h3>
    <p class="description">This is product 1</p>
    <span class="price">$10.99</span>
    <div class="hidden stock">In stock: 5</div>
</article>
```
使用 `.body` 属性获取页面原始内容。从 v0.4 起，在 fetcher 返回的 `Response` 对象上使用时，`.body` 始终返回 `bytes`。
```python
>>> page.body
'<html>\n  <head>\n    <title>Some page</title>\n  </head>\n  ...'
```
获取该元素在 DOM 树中的所有祖先
```python
>>> article.path
[<data='<div class="product-list"> <article clas...' parent='<body> <div class="product-list"> <artic...'>,
 <data='<body> <div class="product-list"> <artic...' parent='<html><head><title>Some page</title></he...'>,
 <data='<html><head><title>Some page</title></he...'>]
```
若可能则生成 CSS 短选择器，否则生成完整选择器
```python
>>> article.generate_css_selector
'body > div > article'
>>> article.generate_full_css_selector
'body > div > article'
```
XPath 同理
```python
>>> article.generate_xpath_selector
"//body/div/article"
>>> article.generate_full_xpath_selector
"//body/div/article"
```

### Traversal
用于在页面上导航元素的属性和方法。

`html` 元素是网站树的根。`head` 和 `body` 等元素是 `html` 的「子元素」，而 `html` 是它们的「父元素」。`body` 元素是 `head` 的「兄弟元素」，反之亦然。

访问元素的父元素
```python
>>> article.parent
<data='<div class="product-list"> <article clas...' parent='<body> <div class="product-list"> <artic...'>
>>> article.parent.tag
'div'
```
支持链式调用，与所有类似属性/方法相同：
```python
>>> article.parent.parent.tag
'body'
```
获取元素的子元素
```python
>>> article.children
[<data='<h3>Product 1</h3>' parent='<article class="product" data-id="1"><h3...'>,
 <data='<p class="description">This is product 1...' parent='<article class="product" data-id="1"><h3...'>,
 <data='<span class="price">$10.99</span>' parent='<article class="product" data-id="1"><h3...'>,
 <data='<div class="hidden stock">In stock: 5</d...' parent='<article class="product" data-id="1"><h3...'>]
```
获取元素下的所有元素。它是 `children` 属性的嵌套版本
```python
>>> article.below_elements
[<data='<h3>Product 1</h3>' parent='<article class="product" data-id="1"><h3...'>,
 <data='<p class="description">This is product 1...' parent='<article class="product" data-id="1"><h3...'>,
 <data='<span class="price">$10.99</span>' parent='<article class="product" data-id="1"><h3...'>,
 <data='<div class="hidden stock">In stock: 5</d...' parent='<article class="product" data-id="1"><h3...'>]
```
该元素返回与 `children` 相同的结果，因为其子元素没有子元素。

使用 class 为 `product-list` 的元素示例，可清楚看出 `children` 与 `below_elements` 的区别
```python
>>> products_list = page.css('.product-list')[0]
>>> products_list.children
[<data='<article class="product" data-id="1"><h3...' parent='<div class="product-list"> <article clas...'>,
 <data='<article class="product" data-id="2"><h3...' parent='<div class="product-list"> <article clas...'>,
 <data='<article class="product" data-id="3"><h3...' parent='<div class="product-list"> <article clas...'>]

>>> products_list.below_elements
[<data='<article class="product" data-id="1"><h3...' parent='<div class="product-list"> <article clas...'>,
 <data='<h3>Product 1</h3>' parent='<article class="product" data-id="1"><h3...'>,
 <data='<p class="description">This is product 1...' parent='<article class="product" data-id="1"><h3...'>,
 <data='<span class="price">$10.99</span>' parent='<article class="product" data-id="1"><h3...'>,
 <data='<div class="hidden stock">In stock: 5</d...' parent='<article class="product" data-id="1"><h3...'>,
 <data='<article class="product" data-id="2"><h3...' parent='<div class="product-list"> <article clas...'>,
...]
```
获取元素的兄弟元素
```python
>>> article.siblings
[<data='<article class="product" data-id="2"><h3...' parent='<div class="product-list"> <article clas...'>,
 <data='<article class="product" data-id="3"><h3...' parent='<div class="product-list"> <article clas...'>]
```
获取当前元素的下一个元素
```python
>>> article.next
<data='<article class="product" data-id="2"><h3...' parent='<div class="product-list"> <article clas...'>
```
`previous` 属性逻辑相同
```python
>>> article.previous  # It's the first child, so it doesn't have a previous element
>>> second_article = page.css('.product[data-id="2"]')[0]
>>> second_article.previous
<data='<article class="product" data-id="1"><h3...' parent='<div class="product-list"> <article clas...'>
```
检查元素是否具有特定 class 名：
```python
>>> article.has_class('product')
True
```
遍历任意元素的完整祖先树：
```python
for ancestor in article.iterancestors():
    # do something with it...
```
搜索满足搜索函数的特定祖先。传入一个接受 [Selector](#selector) 对象并返回 `True`/`False` 的函数：
```python
>>> article.find_ancestor(lambda ancestor: ancestor.has_class('product-list'))
<data='<div class="product-list"> <article clas...' parent='<body> <div class="product-list"> <artic...'>

>>> article.find_ancestor(lambda ancestor: ancestor.css('.product-list'))  # Same result, different approach
<data='<div class="product-list"> <article clas...' parent='<body> <div class="product-list"> <artic...'>
```
## Selectors
`Selectors` 类是 [Selector](#selector) 类的「列表」版本。它继承自 Python 标准 `List` 类型，因此拥有所有 `List` 的属性和方法，并额外添加方法，使对其中 [Selector](#selector) 实例的操作更简便。

在 [Selector](#selector) 类中，所有应返回一组元素的方法/属性都会以 [Selectors](#selectors) 类实例返回。

从 v0.4 起，所有选择方法一致返回 [Selector](#selector)/[Selectors](#selectors) 对象，包括文本节点和属性值。文本节点（通过 `::text`、`/text()`、`::attr()`、`/@attr` 选择）被包装为 [Selector](#selector) 对象。这些文本节点选择器的 `tag` 设为 `"#text"`，其 `text` 属性返回文本值。你仍可直接访问文本值，其他属性会优雅地返回空/默认值。

```python
page.css('a::text')              # -> Selectors (of text node Selectors)
page.xpath('//a/text()')         # -> Selectors
page.css('a::text').get()        # -> TextHandler (the first text value)
page.css('a::text').getall()     # -> TextHandlers (all text values)
page.css('a::attr(href)')        # -> Selectors
page.xpath('//a/@href')          # -> Selectors
page.css('.price_color')         # -> Selectors
```

### 数据提取方法
从 v0.4 起，[Selector](#selector) 和 [Selectors](#selectors) 均提供 `get()`、`getall()` 及其别名 `extract_first` 和 `extract`（遵循 Scrapy 约定）。旧的 `get_all()` 方法已移除。

**在 [Selector](#selector) 对象上：**

- `get()` 返回 `TextHandler`：对文本节点选择器返回文本值；对 HTML 元素选择器返回序列化的 outer HTML。
- `getall()` 返回包含单个序列化字符串的 `TextHandlers` 列表。
- `extract_first` 是 `get()` 的别名，`extract` 是 `getall()` 的别名。

```python
>>> page.css('h3')[0].get()        # Outer HTML of the element
'<h3>Product 1</h3>'

>>> page.css('h3::text')[0].get()  # Text value of the text node
'Product 1'
```

**在 [Selectors](#selectors) 对象上：**

- `get(default=None)` 返回**第一个**元素的序列化字符串，若列表为空则返回 `default`。
- `getall()` 序列化**所有**元素并返回 `TextHandlers` 列表。
- `extract_first` 是 `get()` 的别名，`extract` 是 `getall()` 的别名。

```python
>>> page.css('.price::text').get()      # First price text
'$10.99'

>>> page.css('.price::text').getall()   # All price texts
['$10.99', '$20.99', '$15.99']

>>> page.css('.price::text').get('')    # With default value
'$10.99'
```

这些方法与所有选择类型（CSS、XPath、`find` 等）无缝配合，是以 Scrapy 兼容风格提取文本和属性值的推荐方式。

### 属性
除 Python 列表的标准操作（迭代、切片等）外，还提供以下操作：

CSS 和 XPath 选择器可直接在 [Selector](#selector) 实例上执行，返回类型与 [Selector](#selector) 的 `css` 和 `xpath` 方法相同。参数类似，但 `adaptive` 参数不可用。这使链式调用非常简便：
```python
>>> page.css('.product_pod a')
[<data='<a href="catalogue/a-light-in-the-attic_...' parent='<div class="image_container"> <a href="c...'>,
 <data='<a href="catalogue/a-light-in-the-attic_...' parent='<h3><a href="catalogue/a-light-in-the-at...'>,
 <data='<a href="catalogue/tipping-the-velvet_99...' parent='<div class="image_container"> <a href="c...'>,
 <data='<a href="catalogue/tipping-the-velvet_99...' parent='<h3><a href="catalogue/tipping-the-velve...'>,
 <data='<a href="catalogue/soumission_998/index....' parent='<div class="image_container"> <a href="c...'>,
 <data='<a href="catalogue/soumission_998/index....' parent='<h3><a href="catalogue/soumission_998/in...'>,
...]

>>> page.css('.product_pod').css('a')  # Returns the same result
[<data='<a href="catalogue/a-light-in-the-attic_...' parent='<div class="image_container"> <a href="c...'>,
 <data='<a href="catalogue/a-light-in-the-attic_...' parent='<h3><a href="catalogue/a-light-in-the-at...'>,
 <data='<a href="catalogue/tipping-the-velvet_99...' parent='<div class="image_container"> <a href="c...'>,
 <data='<a href="catalogue/tipping-the-velvet_99...' parent='<h3><a href="catalogue/tipping-the-velve...'>,
 <data='<a href="catalogue/soumission_998/index....' parent='<div class="image_container"> <a href="c...'>,
 <data='<a href="catalogue/soumission_998/index....' parent='<h3><a href="catalogue/soumission_998/in...'>,
...]
```
`re` 和 `re_first` 方法可直接运行。参数与 [Selector](#selector) 类相同。在此类中，`re_first` 对每个内部的 [Selector](#selector) 运行 `re` 并返回第一个有结果的；`re` 方法返回合并所有匹配的 [TextHandlers](#texthandlers) 对象：
```python
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
```
`search` 方法在可用的 [Selector](#selector) 实例中搜索。传入的函数必须接受 [Selector](#selector) 实例作为第一个参数并返回 True/False。返回第一个匹配的 [Selector](#selector) 实例，或 `None`：
```python
# Find all the products with price '53.23'.
>>> search_function = lambda p: float(p.css('.price_color').re_first(r'[\d\.]+')) == 54.23
>>> page.css('.product_pod').search(search_function)
<data='<article class="product_pod"><div class=...' parent='<li class="col-xs-6 col-sm-4 col-md-3 co...'>
```
`filter` 方法接受与 `search` 类似的函数，但返回所有匹配的 [Selector](#selector) 实例组成的 `Selectors`：
```python
# Find all products with prices over $50
>>> filtering_function = lambda p: float(p.css('.price_color').re_first(r'[\d\.]+')) > 50
>>> page.css('.product_pod').filter(filtering_function)
[<data='<article class="product_pod"><div class=...' parent='<li class="col-xs-6 col-sm-4 col-md-3 co...'>,
 <data='<article class="product_pod"><div class=...' parent='<li class="col-xs-6 col-sm-4 col-md-3 co...'>,
 <data='<article class="product_pod"><div class=...' parent='<li class="col-xs-6 col-sm-4 col-md-3 co...'>,
...]
```
安全访问第一个或最后一个元素，避免索引错误：
```python
>>> page.css('.product').first   # First Selector or None
<data='<article class="product" data-id="1"><h3...'>
>>> page.css('.product').last    # Last Selector or None
<data='<article class="product" data-id="3"><h3...'>
>>> page.css('.nonexistent').first  # Returns None instead of raising IndexError
```

获取 [Selectors](#selectors) 实例中 [Selector](#selector) 实例的数量：
```python
page.css('.product_pod').length
```
等价于
```python
len(page.css('.product_pod'))
```

## TextHandler
所有返回字符串的方法/属性都返回 `TextHandler`，返回字符串列表的则返回 [TextHandlers](#texthandlers)。

TextHandler 是标准 Python 字符串的子类，因此支持所有标准字符串操作。

TextHandler 在标准 Python 字符串之外还提供额外方法和属性。所有类中返回字符串的方法/属性都返回 TextHandler，便于链式调用和更简洁的代码。它也可直接导入并用于任意字符串。
### 用法
所有操作（切片、索引等）和方法（`split`、`replace`、`strip` 等）都返回 `TextHandler`，因此可以链式调用。

`re` 和 `re_first` 方法也存在于 [Selector](#selector)、[Selectors](#selectors) 和 [TextHandlers](#texthandlers) 中，接受相同参数。

- `re` 方法以字符串/已编译正则表达式作为第一个参数。它在数据中搜索所有匹配正则的字符串，并以 [TextHandlers](#texthandlers) 实例返回。`re_first` 方法参数相同，但仅返回第一个结果作为 `TextHandler` 实例。
    
    此外还有其他实用参数：
    
    - **replace_entities**：默认启用。将字符实体引用替换为对应字符。
    - **clean_match**：默认禁用。匹配时忽略所有空白（包括连续空格）。
    - **case_sensitive**：默认启用。顾名思义，禁用时编译正则时不区分字母大小写。
  
    因使用 `re` 方法，返回结果为 [TextHandlers](#texthandlers)：
    ```python
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
    ```
    演示其他参数的自定义字符串示例：
    ```python
    >>> from scrapling import TextHandler
    >>> test_string = TextHandler('hi  there')  # Hence the two spaces
    >>> test_string.re('hi there')
    >>> test_string.re('hi there', clean_match=True)  # Using `clean_match` will clean the string before matching the regex
    ['hi there']
    
    >>> test_string2 = TextHandler('Oh, Hi Mark')
    >>> test_string2.re_first('oh, hi Mark')
    >>> test_string2.re_first('oh, hi Mark', case_sensitive=False)  # Hence disabling `case_sensitive`
    'Oh, Hi Mark'
    
    # Mixing arguments
    >>> test_string.re('hi there', clean_match=True, case_sensitive=False)
    ['hi There']
    ```
    由于 `html_content` 返回 `TextHandler`，可直接对 HTML 内容应用正则：
    ```python
    >>> page.html_content.re('div class=".*">(.*)</div')
    ['In stock: 5', 'In stock: 3', 'Out of stock']
    ```

- `.json()` 方法在可能时将内容转换为 JSON 对象，否则抛出错误：
  ```python
  >>> page.css('#page-data::text').get()
    '\n      {\n        "lastUpdated": "2024-09-22T10:30:00Z",\n        "totalProducts": 3\n      }\n    '
  >>> page.css('#page-data::text').get().json()
    {'lastUpdated': '2024-09-22T10:30:00Z', 'totalProducts': 3}
  ```
  若选择元素时未指定文本节点，会自动选择文本内容：
  ```python
  >>> page.css('#page-data')[0].json()
  {'lastUpdated': '2024-09-22T10:30:00Z', 'totalProducts': 3}
  ```
  [Selector](#selector) 类还添加了额外行为。给定此页面：
  ```html
  <html>
      <body>
          <div>
            <script id="page-data" type="application/json">
              {
                "lastUpdated": "2024-09-22T10:30:00Z",
                "totalProducts": 3
              }
            </script>
          </div>
      </body>
  </html>
  ```
  [Selector](#selector) 类有 `get_all_text` 方法，返回 `TextHandler`。例如：
  ```python
  >>> page.css('div::text').get().json()
  ```
  这会抛出错误，因为 `div` 标签没有直接文本内容。`get_all_text` 方法可处理此情况：
  ```python
  >>> page.css('div')[0].get_all_text(ignore_tags=[]).json()
    {'lastUpdated': '2024-09-22T10:30:00Z', 'totalProducts': 3}
  ```
  此处使用 `ignore_tags` 参数，因为其默认值为 `('script', 'style',)`。

  处理 JSON 响应时：
  ```python
  >>> page = Selector("""{"some_key": "some_value"}""")
  ```
  [Selector](#selector) 类针对 HTML 优化，因此将此视为损坏的 HTML 响应并包装。`html_content` 属性显示：
  ```python
  >>> page.html_content
  '<html><body><p>{"some_key": "some_value"}</p></body></html>'
  ```
  可直接使用 `json` 方法：
  ```python
  >>> page.json()
  {'some_key': 'some_value'}
  ```
  对于 JSON 响应，[Selector](#selector) 类会保留接收内容的原始副本。调用 `.json()` 时，先检查该原始副本并转换为 JSON。若原始副本不可用（如子元素），则检查当前元素的文本内容，再回退到 `get_all_text`。

- `.clean()` 方法移除所有空白和连续空格，返回新的 `TextHandler` 实例：
```python
>>> TextHandler('\n wonderful  idea, \reh?').clean()
'wonderful idea, eh?'
```
`remove_entities` 参数会使 `clean` 将 HTML 实体替换为对应字符。

- `.sort()` 方法对字符串字符排序：
```python
>>> TextHandler('acb').sort()
'abc'
```
或反向排序：
```python
>>> TextHandler('acb').sort(reverse=True)
'cba'
```

库中几乎所有返回字符串的地方都用此类替代。

## TextHandlers
此类继承自标准列表，新增 `re` 和 `re_first` 方法。

`re_first` 方法对每个 [TextHandler](#texthandler) 运行 `re` 并返回第一个结果，或 `None`。

## AttributesHandler
这是 Python 标准字典（`dict`）的只读版本，专门用于存储每个元素/[Selector](#selector) 实例的属性。
```python
>>> print(page.find('script').attrib)
{'id': 'page-data', 'type': 'application/json'}
>>> type(page.find('script').attrib).__name__
'AttributesHandler'
```
由于是只读的，它比标准字典占用更少资源。仍具有相同的字典方法和属性，但不包括允许修改/覆盖数据的方法。

目前额外添加两个简单方法：

- `search_values` 方法

    按值（而非键）搜索当前属性，并返回每个匹配项的字典。
    
    简单示例：
    ```python
    >>> for i in page.find('script').attrib.search_values('page-data'):
            print(i)
    {'id': 'page-data'}
    ```
    此方法还提供 `partial` 参数，允许按值的一部分搜索：
    ```python
    >>> for i in page.find('script').attrib.search_values('page', partial=True):
            print(i)
    {'id': 'page-data'}
    ```
    更实用的示例是与 `find_all` 配合，查找属性中具有特定值的所有元素：
    ```python
    >>> page.find_all(lambda element: list(element.attrib.search_values('product')))
    [<data='<article class="product" data-id="1"><h3...' parent='<div class="product-list"> <article clas...'>,
     <data='<article class="product" data-id="2"><h3...' parent='<div class="product-list"> <article clas...'>,
     <data='<article class="product" data-id="3"><h3...' parent='<div class="product-list"> <article clas...'>]
    ```
    这些元素的 `class` 属性值均为 'product'。
    
    此处使用 `list` 是因为 `search_values` 返回生成器，否则对所有元素都会为 `True`。

- `json_string` 属性

    若属性可 JSON 序列化，此属性将当前属性转换为 JSON 字符串，否则抛出错误。
  
    ```python
    >>>page.find('script').attrib.json_string
    b'{"id":"page-data","type":"application/json"}'
    ```
