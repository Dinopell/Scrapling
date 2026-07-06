# 使用 Scrapling 的自定义类型

> 你可以利用 Scrapling 的自定义类型，在库外单独使用。毕竟这比复制代码更好 :)

### 所有当前类型均可单独导入，如下所示
```python
from scrapling.core.custom_types import TextHandler, AttributesHandler

somestring = TextHandler('{}')
somestring.json()  # '{}'
somedict_1 = AttributesHandler({'a': 1})
somedict_2 = AttributesHandler(a=1)
```

请注意，`TextHandler` 是 Python `str` 的子类，因此所有适用于 Python 字符串的标准操作/方法均可使用。
若要在代码中检查类型，最好使用 Python 内置的 `issubclass` 函数。

`AttributesHandler` 类是 `collections.abc.Mapping` 的子类，因此不可变（只读），所有操作均继承自它。传入的数据之后可通过 `_data` 属性访问，但请注意其类型为 `types.MappingProxyType`，同样不可变（只读）（比 `collections.abc.Mapping` 快若干分之一秒）。

因此，为便于理解：若你刚接触 Python，Python 标准 `dict` 类型的所有操作与方法在 `AttributesHandler` 上均可用，唯独尝试修改实际数据的操作除外。

若需修改 `AttributesHandler` 内的数据，须先使用 `dict` 函数等将其转换为字典，再在外部修改。
