# Google Python Style Guide 中文整理版（超详细）

> 说明
>
> 本文依据 Google 官方页面 [Python Style Guide](https://google.github.io/styleguide/pyguide.html) 整理而成，目标是尽可能完整地保留原网页的章节结构、规则意图、判断标准和示例场景，并用中文重新组织成便于阅读的 Markdown 版本。
>
> 这是一份超详细中文整理稿，不是官方逐段逐句直译稿；其中示例代码块为便于理解而重写或改写，尽量对应原文示例想表达的规则。

---

## 1. 背景

- Python 是 Google 主要使用的动态语言之一。
- 团队需要一套统一风格，以降低审查成本、减少维护分歧、提升协作效率。
- 这份风格指南的目标不是“追求某种抽象上的完美形式”，而是尽量让代码：
  - 好读
  - 好改
  - 好审查
  - 好维护
  - 适合多人长期协作
- 页面同时假设工程中会结合自动化工具使用，例如格式化工具和静态检查工具，以减少低价值的样式争论。

---

## 2. Python 语言规则

这一部分关注“可以用哪些语言特性、哪些不推荐、哪些必须谨慎”。核心不是“能不能写出来”，而是“写出来后是否会给团队带来维护负担”。

### 2.1 Lint

#### 核心要求

- 所有代码都应运行 `pylint`。
- `pylint` 可以帮助发现 Python 这类动态语言中容易遗漏的问题，例如：
  - 名字拼错
  - 局部变量先使用后赋值
  - 导入未使用
  - 参数未使用
  - 某些危险或不一致的写法

#### 为什么需要

- Python 在运行前不会像静态类型语言那样暴露大量错误。
- 因此越早发现问题越好，lint 是最便宜的一层保护。

#### 使用原则

- `pylint` 不是完美的，允许按需禁用某些告警。
- 但禁用必须尽量精确，不能大面积、无理由地关闭。
- 推荐写法是行内的：

```python
def get_name(user) -> str:
    return user.full_name  # pylint: disable=no-member
```

- 如果告警名称不能清楚说明原因，应补一小段解释：

```python
def run(argv: list[str]) -> None:
    del argv  # Unused; required by framework callback signature.
    start_server()
```

#### 关于未使用参数

- Google 风格不再鼓励通过命名成 `_`、`unused_x` 这类方式规避告警。
- 更推荐在函数体开头显式 `del` 掉未使用参数，这样表达更直接，也更方便静态工具理解你的意图。

**为什么接口规定必须有？**
- **一致性**：接口/基类定义了调用契约，所有实现类必须遵守，确保调用者能够以统一的方式调用不同的实现。
- **框架要求**：许多框架（如 Web 路由、GUI 回调）会自动传递固定的参数集，如果不接受这些参数，程序会报错。

**推荐示例：**

```python
def handle_event(event_id, timestamp, payload):
    """处理事件的回调函数。"""
    # 实际上由于当前业务逻辑限制，我们不需要 event_id 和 timestamp，但接口规定必须有
    del event_id, timestamp
    
    process_data(payload)
```

#### 实践建议

- 把 lint 看作默认流程的一部分，而不是“最后才跑一次的额外步骤”。
- 如果某条规则经常需要被禁用，应先判断：
  - 是代码确实特殊？
  - 还是项目配置应调整？

---

### 2.2 Imports

#### 核心要求

- `import` 的主要用途是导入模块或包。
- 不鼓励为了省几个字符而到处直接导入函数、类、常量。
- 导入方式应优先保持来源清晰。

#### 推荐形式

```python
import os
import requests
from mypkg import util
from mypkg import parser as config_parser
```

#### 一般不推荐的形式

```python
from mypkg.util import parse_config
from mypkg.util import *
```

#### 原则解释

- 导入模块而不是零散符号，读代码时更容易知道“这个名字来自哪里”。
- 模块名前缀还能减少命名冲突。
- 使用 `from x import y` 时，`y` 最好本身还是一个模块，而不是某个函数或类。

#### 允许别名的场景

- 名称冲突
- 名称过长
- 缩写已经成为事实标准

例如：

```python
import numpy as np
from long_package_name.deep.module import parser as text_parser
```

#### 明确禁止

- 不要使用 `from x import *`
- 不要依赖相对导入

不推荐：

```python
from . import helpers
from ..common import util
```

推荐：

```python
from my_project.feature import helpers
from my_project.common import util
```

#### 例外

- 类型系统相关的导入是允许“直接导入符号”的重要例外。
- 常见包括：
  - `typing`
  - `collections.abc`
  - `typing_extensions`
  - 某些兼容层导入

例如：

```python
from collections.abc import Mapping, Sequence
from typing import Any, TYPE_CHECKING
```

---

### 2.3 Packages

#### 核心要求

- 新代码应通过完整包路径来导入模块。

#### 原因

- 完整路径更稳定。
- 可以避免不同目录下的同名模块被错误导入。
- 可以避免依赖当前运行入口位置或 `sys.path` 细节。

#### 推荐示例

```python
from analytics.jobs import scheduler
from analytics.storage import client
```

#### 不推荐示例

```python
import scheduler
import client
```

#### 理解重点

- “它在我本地能跑” 不足以说明导入写法没问题。
- 导入必须考虑：
  - 测试环境
  - 打包环境
  - 不同入口脚本
  - 不同工作目录

---

### 2.4 Exceptions

#### 核心要求

- 可以使用异常，但必须克制且精确。

#### 推荐做法

- 优先使用内建异常类型。
- 如果参数不合法，优先抛 `ValueError`。
- 如果状态不允许，可能用 `RuntimeError` 或更具体的异常。
- 自定义异常应继承合适的现有异常类，名字以 `Error` 结尾。

例如：

```python
class ConfigError(ValueError):
    """Configuration content is invalid."""
```

#### 关于 `assert`

- 不要把 `assert` 当成业务逻辑的输入校验工具。
- `assert` 只适合表达“理论上必须成立的内部不变量”。
- 在优化模式中，`assert` 可能被移除，因此不能依赖它保护外部输入。

不推荐：

```python
def set_age(age: int) -> None:
    assert age >= 0
```

推荐：

```python
def set_age(age: int) -> None:
    if age < 0:
        raise ValueError("age must be non-negative")
```

#### 捕获异常时的原则

- 只捕获你明确能处理的异常。
- 不要写裸 `except:`
- 不要随意捕获 `Exception`

不推荐：

```python
try:
    run_job()
except:
    return None
```

更合理：

```python
try:
    run_job()
except ConnectionError:
    retry_later()
```

#### 为什么要缩小 `try` 范围

- `try` 块过大会把不相关的错误也吞进去。
- 审查者难以判断你到底想处理哪一步的失败。

不推荐：

```python
try:
    data = load()
    result = transform(data)
    save(result)
except ValueError:
    report_bad_data()
```

更好：

```python
data = load()
try:
    result = transform(data)
except ValueError:
    report_bad_data()
    return
save(result)
```

#### `finally`

- 当你必须无论成功失败都释放资源、恢复状态或记录信息时，用 `finally`。

```python
lock.acquire()
try:
    update_index()
finally:
    lock.release()
```

---

### 2.5 可变全局状态

#### 核心要求

- 避免可变的模块级或类级全局状态。

#### 为什么危险

- 会制造隐式耦合。
- 调用顺序会影响结果。
- 测试之间容易互相污染。
- 并发场景下更容易出错。
- 导入模块时可能就改变程序状态。

#### 不推荐示例

```python
cache = {}


def get_user(user_id: str) -> dict:
    if user_id not in cache:
        cache[user_id] = load_user(user_id)
    return cache[user_id]
```

#### 更稳妥的方向

- 封装到对象里
- 通过明确接口访问
- 在文档中说明生命周期和并发约束

例如：

```python
class UserRepository:
    def __init__(self) -> None:
        self._cache: dict[str, dict] = {}

    def get_user(self, user_id: str) -> dict:
        if user_id not in self._cache:
            self._cache[user_id] = load_user(user_id)
        return self._cache[user_id]
```

#### 允许的例外

- 模块级常量是允许的，而且鼓励这么做。

```python
DEFAULT_TIMEOUT_SECONDS = 30
MAX_BATCH_SIZE = 500
```

#### 如果不得不用可变全局状态

- 至少应做到：
  - 命名上表明它是内部实现
  - 不直接暴露给外部任意修改
  - 通过函数或类方法统一访问
  - 写清楚为何必须这样做

---

### 2.6 嵌套 / 局部 / 内部类与函数

#### 核心要求

- 可以使用，但应限制在确实有价值的场景。

#### 合理场景

- 闭包需要捕获局部变量
- 装饰器内部包装函数
- 某段辅助逻辑只在一个小作用域内使用

例如：

```python
def make_multiplier(factor: int):
    def multiply(value: int) -> int:
        return value * factor

    return multiply
```

#### 风险

- 测试不方便
- 阅读时跳转和定位不方便
- 容易让外层函数过长

#### 不要为了“假装私有”而嵌套

- 如果只是想表示“模块内部使用”，更推荐定义在模块顶层，并用前导下划线命名：

```python
def _normalize_name(value: str) -> str:
    return value.strip().lower()
```

这样测试和复用都会更容易。

---

### 2.7 推导式与生成器表达式

#### 核心要求

- 简单、直观时可以使用。
- 一旦推导式开始变复杂，就改回普通循环。

#### 推荐示例

```python
active_ids = [user.user_id for user in users if user.is_active]
name_set = {item.name for item in items}
```

#### 不推荐示例

```python
matrix = [
    transform(x, y)
    for group in groups
    for x, y in group
    if should_keep(x, y) and extra_rule(x, y)
]
```

这种写法虽然合法，但阅读成本太高。

#### 改写后更清楚

```python
matrix = []
for group in groups:
    for x, y in group:
        if should_keep(x, y) and extra_rule(x, y):
            matrix.append(transform(x, y))
```

#### 判断标准

- 如果审查者一眼看不懂，说明推导式已经太复杂。

---

### 2.8 默认迭代器与运算符

#### 核心要求

- 对支持默认迭代或成员判断的对象，优先使用 Python 原生语法。

#### 推荐写法

```python
for key in user_map:
    ...

if target in names:
    ...

for line in input_file:
    ...

for key, value in user_map.items():
    ...
```

#### 不推荐写法

```python
for key in user_map.keys():
    ...

if target in names.keys():
    ...

for line in input_file.readlines():
    ...
```

#### 原因

- 原生形式更短、更清晰、也更符合 Python 惯例。
- `dict.keys()` 在很多场景下是多余的。
- `readlines()` 会一次性把内容全部读入内存。

#### 注意

- 遍历容器时不要修改这个容器本身。

不推荐：

```python
for key in data:
    if should_remove(key):
        del data[key]
```

更稳妥：

```python
for key in list(data):
    if should_remove(key):
        del data[key]
```

---

### 2.9 Generators

#### 核心要求

- 可以使用生成器，但要理解它们保留状态、延迟计算和资源管理的特性。

#### 优点

- 内存效率高
- 数据按需产生
- 某些流式处理场景更自然

#### 风险

- 生成器中的局部状态会持续存在
- 如果生成器依赖外部资源，资源释放时机可能不明显
- 调试堆栈有时不如普通函数直观

#### 示例

```python
def iter_even_numbers(limit: int):
    for number in range(limit):
        if number % 2 == 0:
            yield number
```

#### 文档写法

- 生成器函数的 docstring 应用 `Yields:`，而不是 `Returns:`。

```python
def iter_even_numbers(limit: int):
    """Generate even numbers below the given limit.

    Args:
        limit: Upper bound, exclusive.

    Yields:
        Even integers from 0 to limit - 1.
    """
    for number in range(limit):
        if number % 2 == 0:
            yield number
```

#### 资源管理提醒

- 如果生成器内部打开文件、连接或锁，要格外注意关闭策略。

---

### 2.10 Lambda

#### 核心要求

- 只在逻辑非常简单时使用 `lambda`。

#### 推荐场景

```python
items.sort(key=lambda item: item.priority)
```

#### 不推荐场景

```python
result = map(
    lambda row: (
        normalize(row.name),
        calculate_score(row.values),
        build_metadata(row),
    ),
    rows,
)
```

#### 更好写法

```python
def convert_row(row):
    return (
        normalize(row.name),
        calculate_score(row.values),
        build_metadata(row),
    )


result = map(convert_row, rows)
```

#### 原则

- 单表达式且一眼能懂，可以。
- 一旦超过一行或开始需要解释，就改成普通函数。
- 某些简单运算直接用 `operator` 模块更清楚。

---

### 2.11 条件表达式

#### 核心要求

- `x if cond else y` 可以用，但必须保持简洁。

#### 推荐示例

```python
status = "ready" if is_ready else "pending"
```

#### 不推荐示例

```python
status = (
    compute_ready_status(item)
    if should_run(item) and check_permission(item) and verify_context(context)
    else build_fallback_status(item, context, cache)
)
```

#### 判断标准

- 条件、真分支、假分支三部分只要有一部分明显变复杂，就改成正常 `if/else`。

```python
if should_run(item) and check_permission(item) and verify_context(context):
    status = compute_ready_status(item)
else:
    status = build_fallback_status(item, context, cache)
```

---

### 2.12 默认参数值

#### 核心要求

- 不要把可变对象写成默认参数值。

#### 错误示例

```python
def add_item(item, bucket=[]):
    bucket.append(item)
    return bucket
```

问题在于默认值只会在函数定义时求值一次，后续调用会共享同一个列表。

#### 正确示例

```python
def add_item(item, bucket=None):
    if bucket is None:
        bucket = []
    bucket.append(item)
    return bucket
```

#### 同样应避免的情况

- 在默认值里调用会随时间或外部状态变化的表达式。

不推荐：

```python
def log_event(created_at=time.time()):
    ...
```

因为 `time.time()` 会在函数定义时执行，而不是每次调用时执行。

#### 更好写法

```python
def log_event(created_at=None):
    if created_at is None:
        created_at = time.time()
```

---

### 2.13 Properties

#### 核心要求

- `property` 适合做轻量、自然、符合属性语义的访问控制。
- 不要把复杂、昂贵或副作用明显的逻辑藏进属性访问里。

#### 合理场景

- 读取派生值
- 验证赋值
- 提供只读属性
- 在不改外部接口名称的前提下调整内部实现

#### 推荐示例

```python
class Rectangle:
    def __init__(self, width: float, height: float) -> None:
        self.width = width
        self.height = height

    @property
    def area(self) -> float:
        return self.width * self.height
```

#### 另一个合理示例

```python
class User:
    def __init__(self) -> None:
        self._age = 0

    @property
    def age(self) -> int:
        return self._age

    @age.setter
    def age(self, value: int) -> None:
        if value < 0:
            raise ValueError("age must be non-negative")
        self._age = value
```

#### 不推荐场景

- 每次访问属性都触发昂贵 I/O
- 每次访问属性都会修改对象状态
- 子类将来很可能需要覆写并扩展复杂计算

例如，下面虽然语法合法，但语义上更像方法：

```python
class Report:
    @property
    def pdf_bytes(self) -> bytes:
        return render_pdf_from_remote_service()
```

更合理：

```python
class Report:
    def render_pdf(self) -> bytes:
        return render_pdf_from_remote_service()
```

---

### 2.14 True / False 判断

#### 核心要求

- 合理利用 Python 的真假值规则，但在语义容易混淆时必须显式。

#### 推荐规则

- 判断 `None` 时，用 `is None` / `is not None`
- 判断空序列时，优先写 `if seq:` 或 `if not seq:`
- 判断布尔值时，不要写 `== True` 或 `== False`

#### 推荐示例

```python
if value is None:
    return

if not users:
    return []

if not is_valid:
    raise ValueError("invalid state")
```

#### 不推荐示例

```python
if value == None:
    ...

if len(users) == 0:
    ...

if is_valid == False:
    ...
```

#### 需要特别小心的情形

- 如果一个变量可能同时取 `None`、`0`、空字符串、空列表等值，隐式真假值可能掩盖语义差别。

例如：

```python
score = get_score()
if not score:
    ...
```

这里无法区分：

- `score is None`
- `score == 0`

更明确：

```python
score = get_score()
if score is None:
    ...
elif score == 0:
    ...
```

#### 关于 `x = x or default`

- 这种写法虽然常见，但有时会把合法的“假值”也错误替换掉。

不推荐：

```python
timeout = timeout or 30
```

因为 `timeout=0` 时会被错误改成 `30`。

更好：

```python
if timeout is None:
    timeout = 30
```

#### 特殊类型提醒

- 例如 `numpy.ndarray` 在布尔上下文中可能报错，因此不要想当然地写 `if array:`。
- 更适合依据 `.size` 或更明确的条件判断。

---

### 2.16 词法作用域

#### 核心要求

- Python 的闭包和词法作用域可以放心使用。
- 但要警惕循环变量绑定和重绑定造成的可读性问题。

#### 常见陷阱

```python
callbacks = []
for i in range(3):
    callbacks.append(lambda: i)
```

许多人以为返回的是 `0, 1, 2`，但实际闭包引用的是同一个 `i`，最终通常都会得到最后一个值。

#### 更稳妥写法

```python
callbacks = []
for i in range(3):
    callbacks.append(lambda i=i: i)
```

或者：

```python
def make_callback(index: int):
    return lambda: index


callbacks = [make_callback(i) for i in range(3)]
```

---

### 2.17 函数与方法装饰器

#### 核心要求

- 装饰器可以用，但只在收益清晰时用。

#### 为什么要谨慎

- 装饰器会隐藏调用过程。
- 它会在定义时执行，而模块级定义通常意味着“导入时执行”。
- 如果装饰器很重、很神秘，排错成本会显著升高。

#### 适合使用的场景

- 注册回调
- 统一日志包装
- 缓存
- 明确的访问控制
- 标记语义

#### 示例

```python
def traced(func):
    def wrapper(*args, **kwargs):
        print(f"calling {func.__name__}")
        return func(*args, **kwargs)

    return wrapper


@traced
def load_user(user_id: str) -> dict:
    return {"user_id": user_id}
```

#### 对装饰器本身的要求

- 要有清晰文档
- 要能被单独测试
- 不要在定义阶段依赖脆弱的外部资源

#### 关于 `staticmethod`

- 除非为了兼容既有接口，不要优先选择 `staticmethod`。
- 很多场景下，把函数写成模块级普通函数会更简单直接。

#### 关于 `classmethod`

- 常用于命名构造器
- 或真正需要操作类级共享状态时

示例：

```python
class User:
    def __init__(self, name: str) -> None:
        self.name = name

    @classmethod
    def from_dict(cls, data: dict[str, str]) -> "User":
        return cls(name=data["name"])
```

---

### 2.18 Threading

#### 核心要求

- 不要依赖内建类型操作的“看起来像原子”的行为来保证线程安全。

#### 为什么

- 某些操作在特定实现上可能看起来安全，但这并不是良好并发设计的基础。
- 代码可移植性、未来重构和组合行为都会让这类假设变脆弱。

#### 推荐做法

- 线程间传递数据时，优先使用 `queue.Queue`
- 同步共享状态时，使用明确的锁或条件变量

例如：

```python
import queue
import threading

work_queue: queue.Queue[int] = queue.Queue()


def worker() -> None:
    while True:
        item = work_queue.get()
        process(item)
        work_queue.task_done()
```

#### 不推荐思路

- “字典更新通常是原子的，所以我们就不加锁了”
- “列表 append 看起来没出问题，所以应该没事”

这些推理都不可靠。

---

### 2.19 Power Features

#### 核心要求

- 避免使用炫技型、魔法感过强的高级特性。

#### 典型例子

- 复杂元类
- 字节码修改
- 运行时编译
- 动态改写继承层次
- 深度自省并改写对象内部行为
- 依赖 `__del__` 做关键清理
- 导入系统黑魔法

#### 不推荐的根本原因

- 很难读
- 很难调试
- 很难让新同事接手
- 工具链支持通常更差

#### 可以接受的情况

- 某些标准库或成熟框架内部使用了这些能力，但对外接口稳定、简单、可预测。
- 例如：
  - `abc`
  - `enum`
  - `dataclasses`

原则是：使用稳定抽象，不要自己发明复杂机制。

---

### 2.20 现代 Python：`from __future__ import ...`

#### 核心要求

- 鼓励按需使用 `from __future__ import ...`

#### 作用

- 让当前文件提前采用较新的语言行为
- 帮助代码跨版本迁移
- 减少“同样语法在不同版本下语义不同”的问题

#### 实践理解

- 如果项目需要兼容一段时间的旧运行时，不要因为“当前环境似乎已经够新”就过早删掉这些导入。

例如：

```python
from __future__ import annotations
```

这在类型注解前向引用和延迟求值上非常常见。

---

### 2.21 类型标注代码

#### 核心要求

- 强烈鼓励在新增或更新代码时启用类型标注和类型检查。

#### 为什么重要

- 更清楚地表达 API 契约
- 更早暴露错误
- 更利于 IDE 和重构工具工作
- 更便于大型工程协作

#### 推荐策略

- 公共 API 优先补类型
- 正在修改的代码顺手补类型
- 构建系统中启用对应类型检查工具

#### 现实原则

- 如果存在工具链阻碍，不要求一步到位覆盖全部代码。
- 但最好保留明确的后续计划或 TODO，说明为什么还没做。

---

## 3. Python 风格规则

这一部分关注“代码长什么样子更利于团队协作”，包括格式、注释、命名、类型标注风格等。

### 3.1 分号

- 行尾不要加分号。
- 不要用分号把多条语句挤在同一行。

不推荐：

```python
x = 1;
y = 2; z = 3
```

推荐：

```python
x = 1
y = 2
z = 3
```

---

### 3.2 行长度

#### 核心要求

- 每行最多 80 个字符。

#### 常见例外

- 很长的 import
- 注释里的 URL、文件路径、命令行参数
- 很难自然拆分的长字符串常量
- 某些工具注释

#### 换行原则

- 优先借助括号、方括号、花括号做隐式续行
- 不要依赖反斜杠续行

不推荐：

```python
result = first_value + second_value + third_value + \
    fourth_value
```

推荐：

```python
result = (
    first_value
    + second_value
    + third_value
    + fourth_value
)
```

#### 注释也要遵守可读性

- URL 太长时，可以把它单独放在一行注释里，而不是强行切碎。
- docstring 的摘要行也应遵守 80 列限制。

---

### 3.3 括号

#### 核心要求

- 不要使用没有意义的额外括号。

不推荐：

```python
if (is_ready):
    return (value)
```

推荐：

```python
if is_ready:
    return value
```

#### 什么时候括号是好的

- 为了隐式换行
- 为了表达元组
- 为了增强复杂表达式的可读性

---

### 3.4 缩进

#### 核心要求

- 使用 4 个空格
- 不要使用 Tab

#### 悬挂缩进示例

```python
result = some_function(
    first_argument,
    second_argument,
    third_argument,
)
```

#### 对齐式示例

```python
result = some_function(first_argument,
                       second_argument,
                       third_argument)
```

第一种通常更常见，也更容易配合自动格式化工具。

#### 结束括号位置

- 可以跟最后一个元素同一行
- 也可以单独成行
- 如果单独成行，应与开括号所在语句对齐

#### 3.4.1 末尾逗号

- 在多行容器字面量、参数列表、导入列表中，尾逗号常常是好事。
- 它有几个优点：
  - 后续增删元素 diff 更小
  - 自动格式化更稳定
  - 单元素元组不会看起来像普通括号表达式

示例：

```python
coordinates = (
    10,
)

config = {
    "host": "localhost",
    "port": 8080,
}
```

---

### 3.5 空行

#### 核心规则

- 顶层函数和类定义之间：空两行
- 类内部方法之间：空一行
- 类 docstring 与第一个方法之间：空一行

#### 目标

- 用空白表达结构
- 让视觉分组更清楚

#### 示例

```python
def load_data() -> list[str]:
    return []


def save_data(values: list[str]) -> None:
    del values


class User:
    """User model."""

    def __init__(self, name: str) -> None:
        self.name = name

    def rename(self, name: str) -> None:
        self.name = name
```

---

### 3.6 空白字符

#### 核心规则

- 采用标准、节制的空格风格
- 不为了视觉对齐插入多余空格

#### 推荐

```python
x = 1
name = "Alice"
if x == 1:
    print(name)
```

#### 不推荐

```python
x    = 1
name =    "Alice"
if(x == 1):
    print( name )
```

#### 参数默认值的细节

- 普通默认值写法，不在 `=` 两边留空格：

```python
def connect(host="localhost", port=8080):
    ...
```

- 如果参数带类型标注，则 `=` 两边留空格：

```python
def connect(host: str = "localhost", port: int = 8080):
    ...
```

#### 不要做纵向对齐

不推荐：

```python
short_name   = "a"
display_name = "b"
user_id      = "c"
```

推荐：

```python
short_name = "a"
display_name = "b"
user_id = "c"
```

---

### 3.7 Shebang

#### 核心要求

- 普通被导入的 `.py` 文件一般不需要 shebang。
- 只有打算作为命令直接执行的脚本主文件才需要。

#### 示例

```python
#!/usr/bin/env python3
```

#### 实际理解

- 库模块不是 shell 命令，不要机械地给每个文件都加 shebang。

---

### 3.8 注释与文档字符串

### 3.8.1 Docstrings

#### 核心要求

- 统一使用三重双引号 `"""`。
- 摘要行应简洁、完整、可独立理解。
- 复杂对象应在摘要之后空一行，再补充细节。

#### 推荐结构

```python
def normalize_name(value: str) -> str:
    """Return a normalized display name.

    Removes leading and trailing whitespace and collapses
    repeated spaces inside the string.
    """
```

#### 常见问题

- 摘要行太长
- 第一行就写成大段描述
- 没说明接口契约，只重复函数名

---

### 3.8.2 模块

#### 核心要求

- 文件顶部应有许可证头
- 模块应有模块级 docstring

#### 模块 docstring 应描述什么

- 这个模块做什么
- 核心对外对象有哪些
- 使用方式或特殊注意事项

#### 示例

```python
"""Utilities for reading and validating job configuration files.

This module exposes helpers for loading YAML configuration,
normalizing defaults, and reporting validation errors.
"""
```

#### 3.8.2.1 测试模块

- 测试文件的模块 docstring 不是硬性要求。
- 只有在需要解释运行方式、环境依赖或测试组织策略时才值得写。
- 如果 docstring 只是“Tests for foo”，价值不大。

---

### 3.8.3 函数与方法

#### 哪些函数必须写 docstring

- 公共 API
- 非常长的函数
- 行为不明显的函数
- 有副作用、边界条件或异常约束的函数

#### 应写哪些内容

- 这个函数做什么
- 参数含义
- 返回值语义
- 迭代器 / 生成器产出什么
- 哪些异常是调用者应该关注的

#### 示例

```python
def read_user(user_id: str) -> dict[str, str]:
    """Load one user record by id.

    Args:
        user_id: Stable unique user identifier.

    Returns:
        A dictionary containing user fields.

    Raises:
        KeyError: If the user does not exist.
    """
```

#### `@property` 的文档

- 应像描述属性，而不是像描述方法返回值。

推荐：

```python
class Session:
    @property
    def is_open(self) -> bool:
        """Whether the network session is currently open."""
```

#### 3.8.3.1 覆写方法

- 若使用 `@override` 且只是继承父类同样契约，可以省略 docstring。
- 但如果行为、副作用、异常约束发生变化，仍应重新写清楚。

---

### 3.8.4 类

#### 核心要求

- 类 docstring 应说明“这个类的实例表示什么”。
- 公共数据属性应写在 `Attributes:` 中。

#### 示例

```python
class UserProfile:
    """Represents one user's public profile.

    Attributes:
        user_id: Stable unique identifier.
        display_name: Name shown in the UI.
        timezone: IANA timezone string.
    """
```

#### 关于异常类

- 异常类 docstring 应描述“这种异常代表什么问题”，而不是“在哪抛出”。

---

### 3.8.5 块注释与行尾注释

#### 核心要求

- 注释应解释“为什么”，而不只是“代码正在做什么”。
- 复杂逻辑前写块注释
- 非显然的小补充可写行尾注释

#### 推荐示例

```python
# The API occasionally returns duplicated ids during backfill.
# Deduplicate here so downstream counters remain stable.
unique_ids = set(api_ids)
```

#### 行尾注释示例

```python
deadline_seconds = 5  # Tight timeout to avoid blocking request threads.
```

#### 不推荐示例

```python
count += 1  # Add one to count.
```

这种注释没有提供额外信息。

---

### 3.8.6 标点、拼写、语法

- 注释也是工程产物，应该像自然语言一样尽量通顺。
- 注意首字母大小写、句号、拼写和语法。
- 注释质量差会直接降低整个模块的专业度和可信度。

---

### 3.10 字符串

#### 字符串格式化

- 可以使用：
  - f-string
  - `%` 格式化
  - `.format()`

#### 不要为了格式化而用 `+`

不推荐：

```python
message = "Hello, " + user_name + "."
```

推荐：

```python
message = f"Hello, {user_name}."
```

#### 关于字符串累加

- 在循环里反复 `+=` 拼接字符串通常不好。

不推荐：

```python
result = ""
for item in items:
    result += format_item(item)
```

推荐：

```python
parts = []
for item in items:
    parts.append(format_item(item))
result = "".join(parts)
```

或：

```python
import io

buffer = io.StringIO()
for item in items:
    buffer.write(format_item(item))
result = buffer.getvalue()
```

#### 引号风格

- 同一文件中尽量保持一致。
- 为了减少转义，可以局部切换单引号或双引号。

#### 多行字符串

- 多行字符串优先使用三重双引号。
- 如果不想把代码缩进带进字符串，可考虑相邻字符串拼接或 `textwrap.dedent()`。

---

### 3.10.1 Logging

#### 核心要求

- 给日志 API 传“格式字符串 + 参数”，而不是提前把字符串拼好。

不推荐：

```python
logger.info(f"User {user_id} logged in from {ip_address}")
```

推荐：

```python
logger.info("User %s logged in from %s", user_id, ip_address)
```

#### 这样做的好处

- 只有在日志真正输出时才做格式化
- 日志平台更容易做模板聚合
- 对结构化分析更友好

---

### 3.10.2 错误消息

#### 好的错误消息应具备

- 条件准确
- 信息具体
- 可搜索
- 尽量不要依赖模糊上下文

#### 不推荐

```python
raise ValueError("bad input")
```

#### 更好

```python
raise ValueError(f"invalid user_id: {user_id!r}")
```

#### 原则

- 错误消息应帮助人和程序都能快速定位问题。

---

### 3.11 文件、Socket 及类似有状态资源

#### 核心要求

- 这类资源使用完必须及时关闭。

#### 包括但不限于

- 文件句柄
- socket
- 数据库连接
- mmap
- 某些图形资源
- 第三方库对象，只要它有显式关闭语义

#### 推荐写法

```python
with open(path, "r", encoding="utf-8") as f:
    content = f.read()
```

#### 如果对象不是上下文管理器

```python
from contextlib import closing

with closing(open_resource()) as resource:
    use(resource)
```

#### 为什么不能依赖析构

- 析构时机不稳定
- 循环引用会影响回收
- 不同运行时行为可能不同

---

### 3.12 TODO 注释

#### 核心要求

- `TODO` 用来标注“当前不完美，但短期可接受”的代码。
- 推荐把 TODO 绑定到明确的上下文，例如 bug、issue 或后续任务链接。

#### 推荐格式

```python
# TODO: crbug.com/123456 - Remove fallback after service migration.
```

#### 好的 TODO 应说明

- 还欠了什么
- 为什么现在先这样
- 什么条件下可以删除或完成

#### 不推荐

```python
# TODO: fix this later
```

这类 TODO 没有责任边界，也没有关闭条件。

---

### 3.13 Import 格式

#### 核心要求

- import 放在文件顶部
- 一般一行一个 import
- 按组排序

#### 推荐分组顺序

1. `__future__`
2. 标准库
3. 第三方库
4. 本仓库内部导入

#### 示例

```python
from __future__ import annotations

import os
import pathlib

import requests

from my_project import config
from my_project.jobs import runner
```

#### 例外

- `typing` 和 `collections.abc` 中的多个符号可以写在同一行。

```python
from collections.abc import Iterable, Mapping, Sequence
from typing import Any, TYPE_CHECKING
```

#### 排序原则

- 组内按完整导入路径字典序排序
- 忽略大小写

---

### 3.14 Statements

#### 核心要求

- 原则上一行一条语句。

#### 可接受的小例外

```python
if ready: run()
```

只有在测试条件和动作都极其简单时，这样的单行写法才勉强可接受。

#### 明显不推荐

```python
if ready: run()
else: stop()
```

```python
try: run()
except ValueError: recover()
```

这些结构压成一行后，可读性会明显下降。

---

### 3.15 Accessors

#### 核心要求

- 只有在 getter / setter 本身携带语义价值时才值得写。

#### 适合写 accessor 的情形

- 设置值时需要校验
- 设置值时需要同步其他状态
- 读取值时需要计算或懒加载

#### 不适合的情形

- 只是机械地包一层读写，没有任何额外语义

不推荐：

```python
class User:
    def __init__(self) -> None:
        self._name = ""

    def get_name(self) -> str:
        return self._name

    def set_name(self, value: str) -> None:
        self._name = value
```

更 Pythonic：

```python
class User:
    def __init__(self) -> None:
        self.name = ""
```

#### 如果接口要从属性迁移到方法

- 应让旧调用点尽快暴露问题，而不是静悄悄做兼容魔法。

---

### 3.16 命名

#### 核心原则

- 名称要描述意图，不要只描述类型。
- 名称要帮助读者建立模型，而不是制造歧义。

#### 3.16.1 应避免的名字

- 无意义单字符
- 令人困惑的缩写
- 带短横线的模块名
- 双前后下划线风格的自造名称
- 把类型硬编码进变量名

不推荐：

```python
user_list_dict = {}
str_name = "alice"
```

更好：

```python
users_by_id = {}
display_name = "alice"
```

#### 允许的简写例外

- 小范围循环变量：`i`、`j`
- 异常对象：`e`
- 文件句柄：`f`
- 数学上下文中的标准记号

#### 3.16.2 命名约定

- 包和模块：`lower_with_under`
- 类：`CapWords`
- 函数和方法：`lower_with_under`
- 常量：`CAPS_WITH_UNDER`
- 内部对象：前导 `_`

#### 示例

```python
MAX_RETRY_COUNT = 5


class UserProfile:
    ...


def load_user_profile() -> UserProfile:
    ...


def _normalize_name() -> str:
    ...
```

#### 关于双下划线

- `__name` 会触发名称改写，不是真正私有。
- 一般不鼓励，除非你非常清楚这么做的目的。

#### 3.16.3 文件命名

- Python 文件必须以 `.py` 结尾
- 文件名中不要使用 `-`

#### 3.16.4 Guido 风格命名映射

- 包：`lower_with_under`
- 模块：`lower_with_under`
- 类：`CapWords`
- 异常：`CapWords`
- 函数 / 方法：`lower_with_under`
- 常量：`CAPS_WITH_UNDER`
- 变量 / 参数：`lower_with_under`

#### 3.16.5 数学记号

- 在数学或算法密集的局部场景中，可以保留传统短符号。
- 但应在注释、docstring 或上下文中解释清楚。

例如：

```python
def update_state(x_k: float, alpha_k: float) -> float:
    """Perform one optimization step following the notation in the paper."""
    return x_k - alpha_k * gradient(x_k)
```

---

### 3.17 Main

#### 核心要求

- 模块应当可安全导入。
- 脚本入口逻辑应放进 `main()`。

#### 推荐模式

```python
def main() -> None:
    run_server()


if __name__ == "__main__":
    main()
```

#### 为什么

- 导入模块时不应自动执行重逻辑
- 这样测试更方便
- 主流程更清晰

#### 使用 `absl` 的场景

- 常见写法是 `app.run(main)`，由框架处理参数和生命周期。

---

### 3.18 函数长度

#### 核心原则

- 函数应尽量短、小、聚焦。

#### 经验判断

- 当一个函数接近或超过约 40 行时，应认真思考是否应该拆分。

#### 不是机械规则

- 并非超过 40 行就一定错。
- 但超长函数更容易同时具备以下问题：
  - 多个职责混杂
  - 局部变量太多
  - 条件嵌套太深
  - 审查时难以完整验证

#### 拆分信号

- 需要很多注释来解释一段步骤
- 某一部分可以被命名成独立动作
- 某一部分可以单独测试

---

### 3.19 类型注解

这一节是页面中信息密度很高的一部分，主要讲“怎么把类型标注写得既现代又可维护”。

#### 3.19.1 通则

- 尽量熟练使用 type hints。
- 公共 API 优先加类型。
- 不需要为了“凑满覆盖率”而给每个局部变量都机械标注。
- 重点给：
  - 易错接口
  - 难懂代码
  - 稳定 API

#### 示例

```python
def build_index(records: list[str]) -> dict[str, int]:
    return {record: i for i, record in enumerate(records)}
```

#### 关于 `self` / `cls`

- 一般不必显式标注。
- 需要表达返回自身实例时，可以考虑 `Self`。

---

#### 3.19.2 换行

- 当函数签名因类型变长时，优先一行一个参数。

推荐：

```python
def build_report(
    user_id: str,
    rows: Sequence[Mapping[str, str]],
    options: ReportOptions,
) -> Report:
    ...
```

- 不要为了挤进一行而牺牲可读性。

#### 如果类型本身很长

- 优先定义类型别名，而不是把签名写成一大串。

```python
type RowMapping = Mapping[str, Sequence[tuple[str, int]]]
```

---

#### 3.19.3 前向声明

- 若类型引用的类稍后才定义：
  - 优先 `from __future__ import annotations`
  - 或使用字符串注解

例如：

```python
from __future__ import annotations


class Node:
    def __init__(self, next_node: Node | None = None) -> None:
        self.next_node = next_node
```

或：

```python
class Node:
    def __init__(self, next_node: "Node | None" = None) -> None:
        self.next_node = next_node
```

---

#### 3.19.4 默认值

- 类型标注参数带默认值时，`=` 两边保留空格。

```python
def connect(host: str = "localhost", port: int = 8080) -> None:
    ...
```

---

#### 3.19.5 NoneType

- 如果参数允许 `None`，必须把它写进类型里。

推荐：

```python
def greet(name: str | None = None) -> str:
    if name is None:
        return "Hello"
    return f"Hello, {name}"
```

#### 不要依赖旧式隐式可空语义

不推荐：

```python
def greet(name: str = None) -> str:
    ...
```

---

#### 3.19.6 类型别名

- 对复杂类型可定义别名，提升可读性。

```python
from collections.abc import Mapping

UserRecord = Mapping[str, str]
```

- 公开别名用 `CapWords`
- 模块内部别名前导 `_`

---

#### 3.19.7 忽略类型

- 必要时可用：
  - `# type: ignore`
  - 或类型工具支持的更细粒度忽略

#### 使用原则

- 尽量局部
- 尽量写在真正有问题的那一行
- 如果原因不明显，补简短说明

---

#### 3.19.8 变量类型

- 当局部变量类型不易推断时，优先用现代注解形式。

推荐：

```python
user_map: dict[str, User] = {}
```

- 不推荐再新增旧式行尾类型注释：

```python
user_map = {}  # type: dict[str, User]
```

---

#### 3.19.9 Tuple 与 List

- `list[T]` 表示同构列表
- `tuple[T, ...]` 表示元素类型一致的元组
- `tuple[int, str]` 表示固定位置异构结构

示例：

```python
names: list[str] = ["a", "b"]
point: tuple[int, int] = (10, 20)
record: tuple[int, str, float] = (1, "ok", 0.5)
```

---

#### 3.19.10 类型变量

- 泛型场景下使用 `TypeVar`、`ParamSpec`
- 名称应反映用途

推荐：

```python
from typing import TypeVar

SortableItem = TypeVar("SortableItem")
```

如果它有明确边界，上名字应更具体：

```python
SupportsCloseType = TypeVar("SupportsCloseType", bound="SupportsClose")
```

#### 什么时候简写 `_T` 可以接受

- 只有在模块内部、没有约束、不会对外暴露时。

---

#### 3.19.11 字符串类型

- 文本用 `str`
- 二进制用 `bytes`
- 新代码不要使用 `typing.Text`

---

#### 3.19.12 用于类型的导入

- 从 `typing` 或 `collections.abc` 直接导入符号是被鼓励的。

```python
from collections.abc import Iterable, Mapping, Sequence
from typing import Any, Generic, TYPE_CHECKING, cast
```

#### 容器类型选择原则

- 函数参数尽量写抽象接口，而不是具体实现。

更好：

```python
def summarize(values: Sequence[int]) -> int:
    return sum(values)
```

不必强行写成：

```python
def summarize(values: list[int]) -> int:
    return sum(values)
```

除非你真的依赖 `list` 的可变语义。

---

#### 3.19.13 条件导入

- 仅在确有需要时，把只用于类型检查的导入放进 `if TYPE_CHECKING:`

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from my_project.storage import StorageClient
```

#### 这样做的意义

- 避免运行时循环依赖
- 避免昂贵导入
- 保持类型工具可见

---

#### 3.19.14 循环依赖

- 如果因为类型标注出现循环依赖，首先应思考设计是否需要重构。
- 若短期无法解决，可用：
  - `TYPE_CHECKING`
  - 字符串注解
  - 过渡性的 `Any`

#### 但重点是

- 不要把“循环依赖”当成长期合理状态。

---

#### 3.19.15 Generics

- 使用泛型时，把类型参数写完整。

不推荐：

```python
items: list = []
```

推荐：

```python
items: list[str] = []
```

#### 原因

- 省略类型参数后，很多工具会回退成 `Any`
- 这会显著削弱类型系统的价值

---

## 4. 结束语

### 最重要的原则：一致性

- 一致性比个人偏好更重要。
- 当你修改已有代码时，先看周围代码已经在使用什么约定。
- 如果局部已经形成稳定风格，通常应优先融入局部风格。

### 但一致性不是借口

- 不能因为“这里以前一直这么写”就为明显更差的风格背书。
- 如果某种旧写法已明显落后、容易误导或和工具链不兼容，应考虑在合理范围内推动改进。

### 实际落地时的建议

- 先保证代码正确
- 再保证接口清晰
- 再保证风格统一
- 最后用自动化工具把机械问题交给工具

---

## 附：这份风格指南可以压缩成哪些核心心法

如果把整份页面浓缩成几条实践准则，大概就是：

1. 写给别人看的代码，优先可读性而不是技巧性。
2. 优先使用清晰、普通、可预测的 Python 写法。
3. 复杂特性只有在收益明确时才使用。
4. 避免隐藏状态、隐式行为和魔法。
5. 类型、文档、注释和资源管理都属于代码质量的一部分。
6. 团队一致性比个人习惯更重要。

