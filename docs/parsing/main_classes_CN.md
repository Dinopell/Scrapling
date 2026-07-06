# 解析核心类

!!! success "前置条件"

    - 你已完成或阅读过[查询元素](../parsing/selection_CN.md)页面，了解如何从 [Selector](../parsing/main_classes_CN.md#selector) 对象中查找/提取元素。

在探索了 Scrapling 中各种元素选择方式及其相关特性之后，让我们退一步，从整体角度审视 [Selector](#selector) 类及其他对象，以便更好地理解解析引擎。

[Selector](#selector) 类是 Scrapling 的核心解析引擎，提供 HTML 解析与元素选择能力。你可以通过以下任一方式导入：
```python
from scrapling import Selector
from scrapling.parser import Selector
```
然后像在[概览](../overview_CN.md)页面中学到的那样直接使用：
```python
page = Selector(
    '<html>...</html>',
    url='https://example.com'
)

# Then select elements as you like
elements = page.css('.product')
```
在 Scrapling 中，传入 HTML 源码或抓取网站后，你主要操作的对象当然是 [Selector](#selector) 对象。你执行的任何操作（如选择、导航等）都会返回 [Selector](#selector) 对象或 [Selectors](#selectors) 对象——前提是结果是页面中的元素，而非文本等。

换句话说，主页面是 [Selector](#selector) 对象，其中的元素也是 [Selector](#selector) 对象，以此类推。任何文本（如元素内的文本内容或属性中的文本）都是 [TextHandler](#texthandler) 对象，每个元素的属性则存储在 [AttributesHandler](#attributeshandler) 中。我们稍后会回到这两个对象，现在先聚焦 [Selector](#selector) 对象。

## Selector
### 参数说明
最重要的参数是 `content`，用于传入要解析的 HTML 代码，接受 `str` 或 `bytes` 类型的 HTML 内容。

此外还有 `url`、`adaptive`、`storage` 和 `storage_args` 参数。这些均与 `adaptive` 特性相关；若不使用该特性，它们没有影响，可暂时忽略，我们会在 [adaptive](adaptive_CN.md) 特性页面中说明。

然后是用于解析调整或在库解析 HTML 时调整/操作内容的参数：

- **encoding**：解析 HTML 时使用的编码，默认为 `UTF-8`。
- **keep_comments**：是否保留 HTML 注释。默认关闭，因为注释可能在多种情况下影响爬取。
- **keep_cdata**：逻辑与 HTML 注释相同。[cdata](https://stackoverflow.com/questions/7092236/what-is-cdata-in-html) 默认移除，以获得更干净的 HTML。

我刻意略过了 `huge_tree` 和 `root` 参数，以免本页过于复杂。
你会发现我经常这样做，因为它们涉及高级特性，使用库时不必了解。若你非常感兴趣，开发章节会涵盖这些遗漏部分。

之后，主页面及其元素上的大多数属性均为惰性加载。也就是说，它们在你实际使用前不会初始化（例如页面/元素的文本内容），这也是 Scrapling 速度快的原因之一 :)

### 属性
你已在[概览](../overview_CN.md)页面见过其中大部分内容；若尚未了解也不必担心。我们将用更高级的方法/用法更全面地回顾。为清晰起见，遍历相关属性单独放在下方 [遍历](#traversal) 一节。

为便于说明，假设我们解析以下 HTML 页面：
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
递归获取页面上全部文本内容
```python
>>> page.get_all_text()
'Some page\n\n    \n\n      \nProduct 1\nThis is product 1\n$10.99\nIn stock: 5\nProduct 2\nThis is product 2\n$20.99\nIn stock: 3\nProduct 3\nThis is product 3\n$15.99\nOut of stock'
```
获取第一个 article，如前所述；我们将用它作为示例
```python
article = page.find('article')
```
同理，递归获取该元素上的全部文本内容
```python
>>> article.get_all_text()
'Product 1\nThis is product 1\n$10.99\nIn stock: 5'
```
但若尝试获取直接文本内容，结果为空，因为上述 HTML 中没有直接文本
```python
>>> article.text
''
```
`get_all_text` 方法有以下可选参数：

1. **separator**：收集到的字符串将用此分隔符拼接，默认为 `'\n'`。
2. **strip**：若启用，拼接前会对字符串执行 strip。默认关闭。
3. **ignore_tags**：要在最终结果中忽略的标签名元组，及其内部嵌套元素。默认为 `('script', 'style',)`。
4. **valid_values**：若启用，方法仅收集有实际值的元素，忽略空文本或仅含空白符的元素。默认启用。

顺便说明，此处返回的文本不是普通字符串，而是 [TextHandler](#texthandler)；稍后详述。若文本内容可序列化为 JSON，对其调用 `.json()` 即可
```python
>>> script = page.find('script')
>>> script.json()
{'lastUpdated': '2024-09-22T10:30:00Z', 'totalProducts': 3}
```
继续获取元素标签名
```python
>>> article.tag
'article'
```
若直接在页面对象上使用，会发现你操作的是根 `html` 元素
```python
>>> page.tag
'html'
```
至此，`page`/`element` 的概念应已足够清晰，不再重复。

获取元素属性
```python
>>> print(article.attrib)
{'class': 'product', 'data-id': '1'}
```
可用以下任一方式访问特定属性
```python
article.attrib['class']
article.attrib.get('class')
article['class']  # new in v0.3
```
可用以下方法检查属性是否包含特定键
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
使用 `.body` 属性获取页面原始内容。自 v0.4 起，在抓取器返回的 `Response` 对象上，`.body` 始终返回 `bytes`。
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
若可能则生成缩短的 CSS 选择器，否则生成完整选择器
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
基于上文找到的元素，我们将详细说明在页面上移动的属性/方法。

若你不熟悉 DOM 树或树形数据结构，以下遍历部分可能令人困惑。建议先在线查阅这些概念以便更好理解。

若懒得搜索，这里有个简要说明。<br/>
简单说，`html` 元素是网站树的根，因为每个页面都以 `html` 元素开始。<br/>
该元素直接位于 `head`、`body` 等元素之上。它们是 `html` 的「子元素」，`html` 是它们的「父元素」。`body` 与 `head` 互为「兄弟元素」。

访问元素的父元素
```python
>>> article.parent
<data='<div class="product-list"> <article clas...' parent='<body> <div class="product-list"> <artic...'>
>>> article.parent.tag
'div'
```
可以任意链式调用，下文类似属性/方法均适用。
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
获取元素下方的所有元素，相当于 `children` 的嵌套版本
```python
>>> article.below_elements
[<data='<h3>Product 1</h3>' parent='<article class="product" data-id="1"><h3...'>,
 <data='<p class="description">This is product 1...' parent='<article class="product" data-id="1"><h3...'>,
 <data='<span class="price">$10.99</span>' parent='<article class="product" data-id="1"><h3...'>,
 <data='<div class="hidden stock">In stock: 5</d...' parent='<article class="product" data-id="1"><h3...'>]
```
此元素与 `children` 结果相同，因为其子元素没有子元素。

再用带 `product-list` 类的元素举例，可看出 `children` 与 `below_elements` 的区别
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
可快速检查元素是否包含特定类名
```python
>>> article.has_class('product')
True
```
若需要不止父元素，可遍历任意元素的整个祖先树，例如：
```python
for ancestor in article.iterancestors():
    # do something with it...
```
可搜索满足条件的特定祖先元素；传入一个接受 [Selector](#selector) 并返回 `True`/`False` 的函数即可，例如：
```python
>>> article.find_ancestor(lambda ancestor: ancestor.has_class('product-list'))
<data='<div class="product-list"> <article clas...' parent='<body> <div class="product-list"> <artic...'>

>>> article.find_ancestor(lambda ancestor: ancestor.css('.product-list'))  # Same result, different approach
<data='<div class="product-list"> <article clas...' parent='<body> <div class="product-list"> <artic...'>
```
## Selectors
`Selectors` 类是 [Selector](#selector) 的「列表」版本。它继承 Python 标准 `List` 类型，因此拥有所有 `List` 的属性和方法，并额外提供方法，便于对内部的 [Selector](#selector) 实例执行操作。

在 [Selector](#selector) 类中，所有应返回一组元素的方法/属性都会以 [Selectors](#selectors) 实例返回。

自 v0.4 起，所有选择方法一致返回 [Selector](#selector)/[Selectors](#selectors) 对象，包括文本节点和属性值。文本节点（通过 `::text`、`/text()`、`::attr()`、`/@attr` 选择）会包装为 [Selector](#selector) 对象。这些文本节点选择器的 `tag` 为 `"#text"`，`text` 属性返回文本值。你仍可直取文本值，其他属性会优雅地返回空/默认值。

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
自 v0.4 起，[Selector](#selector) 和 [Selectors](#selectors) 均提供 `get()`、`getall()` 及其别名 `extract_first` 和 `extract`（遵循 Scrapy 约定）。旧的 `get_all()` 方法已移除。

**在 [Selector](#selector) 对象上：**

- `get()` 返回 `TextHandler`：对文本节点选择器返回文本值；对 HTML 元素选择器返回序列化后的 outer HTML。
- `getall()` 返回包含单个序列化字符串的 `TextHandlers` 列表。
- `extract_first` 是 `get()` 的别名，`extract` 是 `getall()` 的别名。

```python
>>> page.css('h3')[0].get()        # Outer HTML of the element
'<h3>Product 1</h3>'

>>> page.css('h3::text')[0].get()  # Text value of the text node
'Product 1'
```

**在 [Selectors](#selectors) 对象上：**

- `get(default=None)` 返回**第一个**元素的序列化字符串；列表为空时返回 `default`。
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

下面看看 [Selectors](#selectors) 类还提供了什么。
### 属性
除 Python 列表的标准操作（如迭代、切片）外，你还可以：

直接在所包含的 [Selector](#selector) 实例上执行 CSS 和 XPath 选择，返回类型与 [Selector](#selector) 的 `css`、`xpath` 方法相同。参数类似，但此处没有 `adaptive` 参数。这使方法链式调用非常直观。
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
直接运行 `re` 和 `re_first` 方法，参数与 [Selector](#selector) 类相同。这两个方法的说明放在下方 [TextHandler](#texthandler) 一节。

在此类中，`re_first` 行为不同：它对每个 [Selector](#selector) 运行 `re` 并返回第一个有结果者。`re` 方法照常返回 [TextHandlers](#texthandlers) 对象，合并所有 [TextHandler](#texthandler) 实例。
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
使用 `search` 方法可在可用 [Selector](#selector) 实例中快速搜索。传入的函数必须以 [Selector](#selector) 为第一参数并返回 True/False。方法返回第一个满足条件的 [Selector](#selector)，否则返回 `None`。
```python
# Find all the products with price '53.23'.
>>> search_function = lambda p: float(p.css('.price_color').re_first(r'[\d\.]+')) == 54.23
>>> page.css('.product_pod').search(search_function)
<data='<article class="product_pod"><div class=...' parent='<li class="col-xs-6 col-sm-4 col-md-3 co...'>
```
也可使用 `filter` 方法，参数与 `search` 类似，但返回满足条件的所有 [Selector](#selector) 实例组成的 `Selectors` 对象
```python
# Find all products with prices over $50
>>> filtering_function = lambda p: float(p.css('.price_color').re_first(r'[\d\.]+')) > 50
>>> page.css('.product_pod').filter(filtering_function)
[<data='<article class="product_pod"><div class=...' parent='<li class="col-xs-6 col-sm-4 col-md-3 co...'>,
 <data='<article class="product_pod"><div class=...' parent='<li class="col-xs-6 col-sm-4 col-md-3 co...'>,
 <data='<article class="product_pod"><div class=...' parent='<li class="col-xs-6 col-sm-4 col-md-3 co...'>,
...]
```
可安全访问第一个或最后一个元素，无需担心索引错误：
```python
>>> page.css('.product').first   # First Selector or None
<data='<article class="product" data-id="1"><h3...'>
>>> page.css('.product').last    # Last Selector or None
<data='<article class="product" data-id="3"><h3...'>
>>> page.css('.nonexistent').first  # Returns None instead of raising IndexError
```

若像我一样懒得数 [Selectors](#selectors) 实例中有多少个 [Selector](#selector)，可以这样做：
```python
page.css('.product_pod').length
```
等价于
```python
len(page.css('.product_pod'))
```
没错，就像 JavaScript :)

## TextHandler
理解此类很重要：所有应返回字符串的方法/属性都会返回 `TextHandler`，应返回字符串列表的则返回 [TextHandlers](#texthandlers)。

TextHandler 是标准 Python 字符串的子类，因此字符串能做的它都能做。那为何需要单独命名？

TextHandler 提供了标准字符串不具备的额外方法和属性。下面逐一说明，但请记住：所有返回字符串的类的方法/属性都返回 TextHandler，这为创意用法打开了大门，使代码更短更干净。你也可以直接导入并在任意字符串上使用，详见[后文](../development/scrapling_custom_types_CN.md)。
### 用法
在讨论新增方法前，需知对其的一切操作（切片、索引等）以及 `split`、`replace`、`strip` 等方法都会再次返回 `TextHandler`，可任意链式调用。若发现某方法/属性返回标准字符串而非 `TextHandler`，请提 issue，我们会一并覆盖。

首先是 `re` 和 `re_first` 方法，与其他类（[Selector](#selector)、[Selectors](#selectors)、[TextHandlers](#texthandlers)）中的同名方法相同，参数一致。

- `re` 方法第一个参数为字符串或已编译的正则，在数据中搜索所有匹配并作为 [TextHandlers](#texthandlers) 返回。`re_first` 参数相同，但只返回第一个结果，类型为 `TextHandler`。
    
    另有实用参数：
    
    - **replace_entities**：默认启用，将字符实体引用替换为对应字符。
    - **clean_match**：默认关闭，匹配时忽略所有空白（含连续空格）。
    - **case_sensitive**：默认启用，关闭后编译正则时忽略大小写。
  
    以下示例见过；因使用 `re` 方法，返回 [TextHandlers](#texthandlers)。
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
    为更好说明其他参数，下面用自定义字符串举例
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
    处处用 `TextHandler` 替换字符串的另一好处是：`html_content` 等属性返回 `TextHandler`，可直接对 HTML 内容做正则：
    ```python
    >>> page.html_content.re('div class=".*">(.*)</div')
    ['In stock: 5', 'In stock: 3', 'Out of stock']
    ```

- 还有 `.json()` 方法，在可能时快速将内容转为 JSON 对象，否则抛出错误
  ```python
  >>> page.css('#page-data::text').get()
    '\n      {\n        "lastUpdated": "2024-09-22T10:30:00Z",\n        "totalProducts": 3\n      }\n    '
  >>> page.css('#page-data::text').get().json()
    {'lastUpdated': '2024-09-22T10:30:00Z', 'totalProducts': 3}
  ```
  因此，选择元素时若未指定文本节点（如文本内容或属性文本），会自动选择文本内容，例如：
  ```python
  >>> page.css('#page-data')[0].json()
  {'lastUpdated': '2024-09-22T10:30:00Z', 'totalProducts': 3}
  ```
  [Selector](#selector) 类在此还有一处补充；假设工作页面如下：
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
  [Selector](#selector) 类有 `get_all_text` 方法，你应已熟悉。该方法当然返回 `TextHandler`。<br/><br/>
  因此，若执行类似操作
  ```python
  >>> page.css('div::text').get().json()
  ```
  会报错，因为 `div` 标签没有可序列化为 JSON 的直接文本，实际上没有任何直接文本。<br/><br/>
  此时 `get_all_text` 可救场：
  ```python
  >>> page.css('div')[0].get_all_text(ignore_tags=[]).json()
    {'lastUpdated': '2024-09-22T10:30:00Z', 'totalProducts': 3}
  ```
  此处使用 `ignore_tags` 是因为其默认值为 `('script', 'style',)`，如前所述。<br/><br/>
  另一相关行为与任意抓取器有关，稍后说明。若有如下 JSON 响应：
  ```python
  >>> page = Selector("""{"some_key": "some_value"}""")
  ```
  因 [Selector](#selector) 针对 HTML 页面优化，会将其当作破损 HTML 并修复，故使用 `html_content` 会得到：
  ```python
  >>> page.html_content
  '<html><body><p>{"some_key": "some_value"}</p></body></html>'
  ```
  此时可直接使用 `json` 方法：
  ```python
  >>> page.json()
  {'some_key': 'some_value'}
  ```
  你可能会疑惑：`html` 标签没有直接文本，这是如何做到的。<br/>
  对于 JSON 响应等情况，我让 [Selector](#selector) 保留接收内容的原始副本。使用 `.json()` 时先检查该副本再转 JSON；若无原始副本（如元素），则检查当前元素文本，否则直接使用 `get_all_text`。<br/>

- 另一实用方法是 `.clean()`，移除所有空白和连续空格并返回新的 `TextHandler` 实例
```python
>>> TextHandler('\n wonderful  idea, \reh?').clean()
'wonderful idea, eh?'
```
也可传入 `remove_entities` 参数，使 `clean` 将 HTML 实体替换为对应字符。

- 某些情况下 `.sort()` 也有用，像对列表一样排序字符串
```python
>>> TextHandler('acb').sort()
'abc'
```
或反向：
```python
>>> TextHandler('acb').sort(reverse=True)
'cba'
```

更多方法和属性会陆续添加，但请记住：库中几乎所有应返回字符串的地方都返回此类。

## TextHandlers
你大概能猜到：此类类似 [Selectors](#selectors) 和 [Selector](#selector)，但继承标准列表的逻辑与方法，仅新增 `re` 和 `re_first`。

唯一区别是此处的 `re_first` 对每个 [TextHandler](#texthandler) 运行 `re` 并返回第一个结果或 `None`。无需额外说明，新方法会陆续添加。

## AttributesHandler
这是 Python 标准字典 `dict` 的只读版本，专门存储每个元素/[Selector](#selector) 实例的属性。
```python
>>> print(page.find('script').attrib)
{'id': 'page-data', 'type': 'application/json'}
>>> type(page.find('script').attrib).__name__
'AttributesHandler'
```
只读设计比标准字典更省资源，拥有相同的字典方法和属性，但不包括修改/覆盖数据的方法。

目前额外提供两个简单方法：

- `search_values` 方法

    标准字典可用 `dict.get("key_name")` 检查键是否存在；若按值而非键搜索，需要额外代码。此方法替你完成，按值搜索当前属性并返回每个匹配项的字典。
    
    简单示例：
    ```python
    >>> for i in page.find('script').attrib.search_values('page-data'):
            print(i)
    {'id': 'page-data'}
    ```
    还提供 `partial` 参数，支持按值的一部分搜索：
    ```python
    >>> for i in page.find('script').attrib.search_values('page', partial=True):
            print(i)
    {'id': 'page-data'}
    ```
    实际场景更可能是与 `find_all` 配合，查找属性中包含特定值的所有元素：
    ```python
    >>> page.find_all(lambda element: list(element.attrib.search_values('product')))
    [<data='<article class="product" data-id="1"><h3...' parent='<div class="product-list"> <article clas...'>,
     <data='<article class="product" data-id="2"><h3...' parent='<div class="product-list"> <article clas...'>,
     <data='<article class="product" data-id="3"><h3...' parent='<div class="product-list"> <article clas...'>]
    ```
    这些元素的 `class` 属性值均为 `product`。
    
    此处使用 `list` 是因为 `search_values` 返回生成器，对所有元素都会为 `True`。

- `json_string` 属性

    若属性可 JSON 序列化，此属性将当前属性转为 JSON 字符串，否则抛出错误。
  
    ```python
    >>>page.find('script').attrib.json_string
    b'{"id":"page-data","type":"application/json"}'
    ```
