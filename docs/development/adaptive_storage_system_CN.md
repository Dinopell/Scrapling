# 编写你的检索系统

Scrapling 默认使用 SQLite，本教程展示如何编写自己的存储系统，为 `adaptive` 功能存储元素属性。

例如，你可能想使用 Firebase，并在不同机器上的多个 Spider 之间共享数据库。使用此类在线数据库是个好主意，因为 Spider 可以彼此共享自适应数据。

首先，要使存储类正常工作，必须满足三大要求：

1. 继承抽象类 `scrapling.core.storage.StorageSystemMixin`，并接受一个字符串参数，即 `url` 参数，以维持库的逻辑。
2. 在类上使用装饰器 `functools.lru_cache`，以遵循与其他类相同的单例设计模式。
3. 实现 `save` 与 `retrieve` 方法，如类型提示所示：
    - `save` 方法无返回值，库会传入两个参数
        * 第一个参数类型为 `lxml.html.HtmlElement`，即元素本身。必须使用子模块 `scrapling.core.utils._StorageTools` 中的 `element_to_dict` 函数转换为字典以保持相同格式，然后按你的方式保存到数据库。
        * 第二个参数为字符串，即用于检索的标识符。该标识符与初始化时的 `url` 参数组合后，每一行必须唯一，否则 `adaptive` 数据会混乱。
    - `retrieve` 方法接受一个字符串标识符；结合初始化时传入的 `url`，从数据库检索元素字典并返回；若不存在则返回 `None`。

> 若说明不够清晰，可参考我在 [storage_adaptors](https://github.com/D4Vinci/Scrapling/blob/main/scrapling/core/storage.py) 文件中使用 SQLite3 的实现

若你的类满足上述条件，其余就很简单了。若计划在多线程应用中使用该库，请确保你的类支持多线程。默认使用的类是线程安全的。

抽象类中还添加了一些辅助函数供你使用。直接在[代码](https://github.com/D4Vinci/Scrapling/blob/main/scrapling/core/storage.py)中查看更方便，注释很详细 :)


## 真实示例：Redis 存储

以下是由 AI 生成的更实用示例，使用 Redis：

```python
import redis
import orjson
from functools import lru_cache
from scrapling.core.storage import StorageSystemMixin
from scrapling.core.utils import _StorageTools

@lru_cache(None)
class RedisStorage(StorageSystemMixin):
    def __init__(self, host='localhost', port=6379, db=0, url=None):
        super().__init__(url)
        self.redis = redis.Redis(
            host=host,
            port=port,
            db=db,
            decode_responses=False
        )
        
    def save(self, element, identifier: str) -> None:
        # Convert element to dictionary
        element_dict = _StorageTools.element_to_dict(element)
        
        # Create key
        key = f"scrapling:{self._get_base_url()}:{identifier}"
        
        # Store as JSON
        self.redis.set(
            key,
            orjson.dumps(element_dict)
        )
        
    def retrieve(self, identifier: str) -> dict | None:
        # Get data
        key = f"scrapling:{self._get_base_url()}:{identifier}"
        data = self.redis.get(key)
        
        # Parse JSON if exists
        if data:
            return orjson.loads(data)
        return None
```
