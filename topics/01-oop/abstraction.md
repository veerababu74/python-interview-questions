# Abstraction

## 1. Concept (Deep Explanation)

### What Is Abstraction?

Abstraction is the OOP principle of **hiding implementation complexity behind a simple, consistent interface**. It allows callers to work with objects through a contract (a set of defined operations) without knowing *how* those operations are implemented internally.

Abstraction answers: *"What does this object do?"* — Encapsulation answers: *"How does it protect its data?"*

In Python, abstraction is primarily achieved through:
1. **Abstract Base Classes (ABC)** — formal contracts enforced at instantiation time.
2. **Protocols (PEP 544)** — structural subtyping, checked statically by type checkers.
3. **Duck typing** — informal abstraction via shared method names.

### The `abc` Module

The `abc` module (Abstract Base Classes) provides the machinery to define formal abstract interfaces.

**Key components:**

| Component | Purpose |
|---|---|
| `ABC` | Convenience base class (uses `ABCMeta` as metaclass) |
| `ABCMeta` | Metaclass that tracks abstract methods |
| `@abstractmethod` | Marks a method as abstract (must be overridden) |
| `@abstractclassmethod` | Deprecated — use `@classmethod @abstractmethod` |
| `@abstractstaticmethod` | Deprecated — use `@staticmethod @abstractmethod` |
| `@abstractproperty` | Deprecated — use `@property @abstractmethod` |

```python
from abc import ABC, abstractmethod


class Shape(ABC):
    """Abstract base class for all shapes."""

    @abstractmethod
    def area(self) -> float:
        """Return the area of the shape."""
        ...

    @abstractmethod
    def perimeter(self) -> float:
        """Return the perimeter of the shape."""
        ...

    def describe(self) -> str:   # concrete method — shared behaviour
        return (
            f"{type(self).__name__}: "
            f"area={self.area():.2f}, perimeter={self.perimeter():.2f}"
        )


class Circle(Shape):
    def __init__(self, radius: float) -> None:
        self.radius = radius

    def area(self) -> float:
        import math
        return math.pi * self.radius ** 2

    def perimeter(self) -> float:
        import math
        return 2 * math.pi * self.radius


class Square(Shape):
    def __init__(self, side: float) -> None:
        self.side = side

    def area(self) -> float:
        return self.side ** 2

    def perimeter(self) -> float:
        return 4 * self.side


# Attempting to instantiate the abstract class:
try:
    s = Shape()
except TypeError as e:
    print(e)   # Can't instantiate abstract class Shape with abstract methods area, perimeter

c = Circle(5.0)
print(c.describe())   # Circle: area=78.54, perimeter=31.42
```

### How ABCMeta.__new__ Prevents Instantiation

When a class uses `ABCMeta` as its metaclass, two things happen during class creation:

1. `ABCMeta.__new__` collects all methods decorated with `@abstractmethod` into a frozenset stored in `cls.__abstractmethods__`.
2. When `type.__call__` is invoked to create an instance (i.e., `Shape()`), it checks `cls.__abstractmethods__`. If the frozenset is non-empty, it raises `TypeError`.

```python
from abc import ABC, abstractmethod

class Animal(ABC):
    @abstractmethod
    def speak(self) -> str: ...

print(Animal.__abstractmethods__)   # frozenset({'speak'})

class Dog(Animal):
    def speak(self) -> str: return "Woof"

print(Dog.__abstractmethods__)      # frozenset()  — all abstract methods overridden

# Manually clearing __abstractmethods__ bypasses the check (not recommended):
Animal.__abstractmethods__ = frozenset()
a = Animal()   # Works now (but is a terrible idea)
```

This mechanism lives entirely in CPython's `typeobject.c` in the `type_call` function.

### Abstract Class Method and Static Method

```python
from abc import ABC, abstractmethod


class Serializable(ABC):
    @classmethod
    @abstractmethod
    def from_dict(cls, data: dict) -> "Serializable":
        """Concrete subclasses must implement this factory."""
        ...

    @staticmethod
    @abstractmethod
    def format_name() -> str:
        """Return the serialisation format name."""
        ...

    @property
    @abstractmethod
    def schema_version(self) -> int:
        """Return the schema version."""
        ...


class JSONRecord(Serializable):
    def __init__(self, data: dict) -> None:
        self._data = data

    @classmethod
    def from_dict(cls, data: dict) -> "JSONRecord":
        return cls(data)

    @staticmethod
    def format_name() -> str:
        return "JSON"

    @property
    def schema_version(self) -> int:
        return 2
```

**Important ordering note:** `@classmethod @abstractmethod` must have `@classmethod` *outer* and `@abstractmethod` *inner* (Python 3.3+). The deprecated `@abstractclassmethod` combined both, but is removed in Python 3.11+.

### Virtual Subclasses with ABC.register()

ABCs support **virtual subclasses** — classes that are considered subclasses of an ABC without actually inheriting from it. This bridges formal ABCs with duck typing.

```python
from abc import ABC, abstractmethod


class Drawable(ABC):
    @abstractmethod
    def draw(self) -> None: ...


class LegacyWidget:
    """Third-party class we cannot modify."""
    def draw(self) -> None:
        print("LegacyWidget draws itself")


# Register as a virtual subclass:
Drawable.register(LegacyWidget)

w = LegacyWidget()
print(isinstance(w, Drawable))    # True — virtual subclass
print(issubclass(LegacyWidget, Drawable))  # True

# But ABCMeta does NOT verify that LegacyWidget actually has .draw():
class FakeWidget:
    pass

Drawable.register(FakeWidget)
f = FakeWidget()
print(isinstance(f, Drawable))    # True — registered but draw() not implemented!
# f.draw()  → AttributeError at runtime
```

Virtual subclasses are a power tool for integrating with legacy or third-party code. They trust the programmer to ensure the contract is met.

### __subclasshook__ for Structural Checking

`__subclasshook__` allows an ABC to define custom `issubclass` logic without requiring `register()`. This enables *structural subtype checking* directly in the ABC:

```python
from abc import ABC, abstractmethod


class Closeable(ABC):
    @abstractmethod
    def close(self) -> None: ...

    @classmethod
    def __subclasshook__(cls, subclass: type) -> bool | type(NotImplemented):
        if cls is Closeable:
            # Structural check: does the class have a callable 'close' attribute?
            if any("close" in B.__dict__ for B in subclass.__mro__):
                return True
        return NotImplemented


class FileHandle:
    def close(self) -> None:
        print("File closed")


print(issubclass(FileHandle, Closeable))   # True — structural check passes
print(isinstance(FileHandle(), Closeable)) # True
```

`__subclasshook__` is what makes Python's built-in ABCs like `collections.abc.Iterable` work: any class with `__iter__` is considered an `Iterable`.

### Protocol (PEP 544) vs ABC

Python 3.8+ introduced `Protocol` as a cleaner structural subtyping mechanism:

```python
from typing import Protocol, runtime_checkable


@runtime_checkable
class Drawable(Protocol):
    def draw(self) -> None: ...


class Canvas:
    def draw(self) -> None:
        print("Canvas draws")


class Button:
    def draw(self) -> None:
        print("Button draws")


class TextLabel:   # No inheritance from Drawable
    def draw(self) -> None:
        print("TextLabel draws")


def render_all(items: list[Drawable]) -> None:
    for item in items:
        item.draw()

render_all([Canvas(), Button(), TextLabel()])  # All work — structural typing
print(isinstance(Canvas(), Drawable))   # True (runtime_checkable)
```

| Aspect | ABC | Protocol |
|---|---|---|
| Enforcement mechanism | `ABCMeta` + `__abstractmethods__` | Type checker (mypy, pyright) |
| Runtime `isinstance` | Yes | Only with `@runtime_checkable` |
| Requires inheritance | Yes (or `register()`) | No |
| Prevents instantiation | Yes (abstract methods) | No |
| Best use | Class hierarchies, frameworks | Duck typing with type safety |

### When to Use ABC vs Protocol

Use **ABC** when:
- You're building a framework or plugin system where concrete implementations must inherit from the base.
- You want to provide shared concrete methods alongside abstract methods.
- You need runtime `isinstance` checks as part of dispatch logic.

Use **Protocol** when:
- You're defining an interface for structural/duck typing.
- You want type safety without requiring inheritance.
- You're working with third-party or legacy code that you can't modify.
- You want the interface to live with the *consumer*, not the *producer*.

---

## 2. Conceptual Interview Questions (10)

**Q1: What is the difference between abstraction and encapsulation?**

**A:** They are complementary but distinct:

- **Encapsulation** is about *protection and bundling*: keeping data and behaviour together and hiding internal state from outside access.
- **Abstraction** is about *simplification and interface*: exposing only the essential features and hiding the *how*.

A class can encapsulate without abstracting (a concrete utility class with private state). A class can define an abstract interface without encapsulating much (an ABC with only abstract methods and no data).

```python
# Encapsulation: hides _balance, exposes deposit/withdraw
class BankAccount:
    def __init__(self): self._balance = 0.0
    def deposit(self, amount): self._balance += amount

# Abstraction: defines interface without implementation
from abc import ABC, abstractmethod
class Payable(ABC):
    @abstractmethod
    def process_payment(self, amount: float) -> bool: ...
```

---

**Q2: What happens internally when you try to instantiate an abstract class?**

**A:** CPython's `type.__call__` (in `typeobject.c`) checks `cls.__abstractmethods__` before calling `cls.__new__`. If the frozenset is non-empty, it raises `TypeError` with a message listing the unimplemented abstract methods.

```python
from abc import ABC, abstractmethod

class Base(ABC):
    @abstractmethod
    def run(self) -> None: ...
    @abstractmethod
    def stop(self) -> None: ...

try:
    Base()
except TypeError as e:
    print(e)  # Can't instantiate abstract class Base with abstract methods run, stop

class Partial(Base):
    def run(self) -> None: pass
    # stop is still abstract

print(Partial.__abstractmethods__)  # frozenset({'stop'})
try:
    Partial()
except TypeError as e:
    print(e)  # Can't instantiate abstract class Partial with abstract method stop
```

---

**Q3: What is `ABC.register()` and when would you use it?**

**A:** `register(subclass)` declares a class as a *virtual subclass* of the ABC. `isinstance(obj, ABC)` and `issubclass(cls, ABC)` return `True` for registered classes, even without inheritance. Use it to integrate third-party or legacy code with ABCs without modifying the third-party class.

```python
from abc import ABC
class Printable(ABC): pass

class ThirdPartyDoc:  # can't modify this
    def print(self): print("doc")

Printable.register(ThirdPartyDoc)
print(issubclass(ThirdPartyDoc, Printable))   # True
```

---

**Q4: What is __subclasshook__ and how does it enable structural typing in ABCs?**

**A:** `__subclasshook__` is a classmethod on ABC subclasses that `ABCMeta.__instancecheck__` and `ABCMeta.__subclasscheck__` call before the standard check. If it returns `True` or `False`, that result is used immediately; if it returns `NotImplemented`, the normal check proceeds.

```python
from abc import ABC
class Sized(ABC):
    @classmethod
    def __subclasshook__(cls, C):
        if cls is Sized:
            return any("__len__" in B.__dict__ for B in C.__mro__)
        return NotImplemented

print(issubclass(list, Sized))   # True — list has __len__
print(issubclass(int, Sized))    # False — int has no __len__
```

---

**Q5: What is the difference between Protocol and ABC?**

**A:** ABC requires inheritance (or `register()`). Protocol relies on structural subtyping — a class satisfies a Protocol if it has the required methods, regardless of inheritance. ABCs are checked at runtime by default; Protocols are checked statically by type checkers (unless `@runtime_checkable` is used). ABCs can contain concrete methods; Protocols should be interface-only.

Use ABC for framework hierarchies; use Protocol for flexible, duck-typing-friendly interfaces that shouldn't constrain the implementor's class hierarchy.

---

**Q6: Can an abstract class have concrete methods? Why is this useful?**

**A:** Yes. Abstract base classes often provide *partial implementations* — concrete utility methods that rely on the abstract methods:

```python
from abc import ABC, abstractmethod
import math

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

    def is_larger_than(self, other: "Shape") -> bool:
        return self.area() > other.area()   # concrete method using abstract area()

class Circle(Shape):
    def __init__(self, r): self.r = r
    def area(self): return math.pi * self.r ** 2

class Square(Shape):
    def __init__(self, s): self.s = s
    def area(self): return self.s ** 2

print(Circle(5).is_larger_than(Square(8)))   # True — 78.5 > 64
```

This is the **Template Method** design pattern: define the skeleton of an algorithm in the base class with some steps delegated to subclasses.

---

**Q7: How do you enforce that a subclass implements a property (not just a method)?**

**A:** Combine `@property` and `@abstractmethod`:

```python
from abc import ABC, abstractmethod

class DataStore(ABC):
    @property
    @abstractmethod
    def connection_string(self) -> str: ...

class PostgresStore(DataStore):
    @property
    def connection_string(self) -> str:
        return "postgresql://localhost/mydb"

try:
    DataStore()   # TypeError — abstract property
except TypeError: pass

p = PostgresStore()
print(p.connection_string)   # "postgresql://localhost/mydb"
```

---

**Q8: What happens if a subclass implements only some abstract methods?**

**A:** The subclass remains abstract. Its `__abstractmethods__` frozenset contains only the unimplemented methods. Instantiating it raises `TypeError`.

```python
from abc import ABC, abstractmethod

class I(ABC):
    @abstractmethod
    def a(self): ...
    @abstractmethod
    def b(self): ...

class Partial(I):
    def a(self): pass   # b still abstract

print(Partial.__abstractmethods__)  # frozenset({'b'})
try: Partial()
except TypeError as e: print(e)   # abstract method b
```

---

**Q9: Can you use `abc.ABC` and `typing.Protocol` together?**

**A:** They serve different purposes and can coexist in a codebase, but combining them in a single class is unusual and not recommended. If a class inherits from both `ABC` and `Protocol`, ABCMeta becomes the metaclass and Protocol's structural checking still works, but the semantics become confusing. Prefer choosing one abstraction mechanism per interface.

---

**Q10: How does `collections.abc` use ABCMeta to provide built-in abstract interfaces?**

**A:** `collections.abc` defines ABCs like `Iterable`, `Iterator`, `Sequence`, `Mapping`, etc. They use `ABCMeta` with `__subclasshook__` to provide structural typing. For example, any class defining `__iter__` is considered an `Iterable` without inheriting from it:

```python
from collections.abc import Iterable, Mapping

class MyIter:
    def __iter__(self):
        return iter([1, 2, 3])

print(isinstance(MyIter(), Iterable))  # True — __subclasshook__ checks for __iter__
print(isinstance({}, Mapping))         # True
print(isinstance([], Mapping))         # False
```

---

## 3. Scenario-Based Interview Questions (10)

**Scenario 1: You are building a payment processing library that supports Stripe, PayPal, and custom providers. How do you define the interface?**

**Q:** Design the abstract interface.

**A:** Use an ABC with abstract methods defining the contract, and concrete helper methods:

```python
from abc import ABC, abstractmethod
from decimal import Decimal


class PaymentGateway(ABC):
    """Abstract interface for payment providers."""

    @abstractmethod
    def charge(self, amount: Decimal, currency: str, token: str) -> str:
        """Charge a customer. Return transaction ID."""
        ...

    @abstractmethod
    def refund(self, transaction_id: str, amount: Decimal) -> bool:
        """Refund a transaction. Return success status."""
        ...

    @abstractmethod
    def get_status(self, transaction_id: str) -> str:
        """Return transaction status: 'pending', 'completed', 'failed'."""
        ...

    def charge_and_log(self, amount: Decimal, currency: str, token: str) -> str:
        import logging
        txn_id = self.charge(amount, currency, token)
        logging.info("Charged %s %s → txn %s", amount, currency, txn_id)
        return txn_id


class StripeGateway(PaymentGateway):
    def charge(self, amount, currency, token): return f"stripe-{token[:8]}"
    def refund(self, txn_id, amount): return True
    def get_status(self, txn_id): return "completed"
```

---

**Scenario 2: You have a plugin system where plugins are discovered dynamically at runtime. How do you verify a plugin satisfies the interface before using it?**

**Q:** Use ABC with isinstance check.

**A:**

```python
from abc import ABC, abstractmethod
import importlib


class Plugin(ABC):
    @abstractmethod
    def execute(self, data: dict) -> dict: ...

    @abstractmethod
    def name(self) -> str: ...


def load_plugin(module_name: str, class_name: str) -> Plugin:
    mod = importlib.import_module(module_name)
    cls = getattr(mod, class_name)
    if not (isinstance(cls, type) and issubclass(cls, Plugin)):
        raise TypeError(f"{class_name} must be a Plugin subclass")
    instance = cls()
    return instance
```

---

**Scenario 3: You need to integrate a third-party HTTP client library into your abstraction layer. The library has a `.send(request)` method but doesn't inherit from your `HttpClient` ABC. How do you integrate it?**

**Q:** Use register() or __subclasshook__.

**A:**

```python
from abc import ABC, abstractmethod


class HttpClient(ABC):
    @abstractmethod
    def send(self, request: dict) -> dict: ...

    @classmethod
    def __subclasshook__(cls, C: type) -> bool | type(NotImplemented):
        if cls is HttpClient:
            if any("send" in B.__dict__ for B in C.__mro__):
                return True
        return NotImplemented


# Third-party library class (not modifiable):
class RequestsAdapter:
    def send(self, request: dict) -> dict:
        return {"status": 200}


print(issubclass(RequestsAdapter, HttpClient))    # True — structural check
client: HttpClient = RequestsAdapter()
print(client.send({"url": "https://api.example.com"}))
```

---

**Scenario 4: How do you combine ABC with dataclass to create abstract data models?**

**Q:** Implement an abstract dataclass hierarchy.

**A:**

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass


@dataclass
class BaseEvent(ABC):
    event_id: str
    timestamp: float

    @abstractmethod
    def to_payload(self) -> dict: ...

    @classmethod
    @abstractmethod
    def event_type(cls) -> str: ...


@dataclass
class ClickEvent(BaseEvent):
    x: int
    y: int

    def to_payload(self) -> dict:
        return {
            "type": self.event_type(),
            "id":   self.event_id,
            "x":    self.x,
            "y":    self.y,
        }

    @classmethod
    def event_type(cls) -> str:
        return "click"


e = ClickEvent(event_id="evt-001", timestamp=1_700_000_000.0, x=100, y=200)
print(e.to_payload())
```

---

**Scenario 5: A team member says "just use duck typing, ABCs are Java-style boilerplate." How do you respond?**

**Q:** Compare the trade-offs.

**A:** Both have valid use cases:

```python
# Duck typing — simple, flexible, no boilerplate
def process(thing):
    thing.execute()   # works for any object with execute()

# ABC — enforced contract, clear error at class definition time
from abc import ABC, abstractmethod

class Executable(ABC):
    @abstractmethod
    def execute(self) -> None: ...

class BadPlugin(Executable):
    pass   # TypeError immediately: abstract method 'execute' not implemented
```

ABCs provide:
1. **Early error detection** — at class definition, not first use.
2. **Documentation** — the interface is explicit and IDE-visible.
3. **`isinstance` support** — enables runtime type checking for dispatch.

Duck typing provides:
1. **Flexibility** — no inheritance required.
2. **Less coupling** — producers and consumers are decoupled.

For a public library API, ABCs document and enforce contracts. For internal utility functions, duck typing is usually sufficient.

---

**Scenario 6: You need a sorted container ABC that works with the built-in `sorted()` function. Implement it.**

**Q:** Design using ABC + Protocol hybrid.

**A:**

```python
from abc import ABC, abstractmethod
from typing import Iterator


class SortedCollection(ABC):
    @abstractmethod
    def __iter__(self) -> Iterator: ...

    @abstractmethod
    def __len__(self) -> int: ...

    @abstractmethod
    def add(self, item) -> None: ...

    def to_sorted_list(self) -> list:
        return sorted(self)   # relies on __iter__ being abstract-enforced


class SortedList(SortedCollection):
    def __init__(self) -> None:
        self._items: list = []

    def __iter__(self) -> Iterator:
        return iter(self._items)

    def __len__(self) -> int:
        return len(self._items)

    def add(self, item) -> None:
        import bisect
        bisect.insort(self._items, item)


sc = SortedList()
sc.add(3)
sc.add(1)
sc.add(2)
print(sc.to_sorted_list())   # [1, 2, 3]
```

---

**Scenario 7: You need to verify at type-check time (not runtime) that all implementations of an interface provide a specific set of methods. Should you use ABC or Protocol?**

**Q:** Choose and justify.

**A:** Use `Protocol` for static (type-checker-time) verification, as it works structurally without requiring inheritance:

```python
from typing import Protocol


class Serializer(Protocol):
    def serialize(self, obj: object) -> bytes: ...
    def deserialize(self, data: bytes) -> object: ...


class JSONSerializer:
    def serialize(self, obj: object) -> bytes:
        import json
        return json.dumps(obj).encode()

    def deserialize(self, data: bytes) -> object:
        import json
        return json.loads(data)


def save(serializer: Serializer, obj: object, path: str) -> None:
    with open(path, "wb") as f:
        f.write(serializer.serialize(obj))

# mypy/pyright will verify JSONSerializer satisfies Serializer at type-check time
save(JSONSerializer(), {"key": "value"}, "output.bin")
```

---

**Scenario 8: How do you prevent certain abstract methods from being accidentally implemented as class methods or static methods when they should be instance methods?**

**Q:** Show the pattern.

**A:** This is enforced by `@abstractmethod` — the descriptor protocol ensures the implementing method has the correct binding. However, Python doesn't distinguish *kind* of method by default. You can add runtime checks in `__init_subclass__`:

```python
from abc import ABC, abstractmethod
import inspect


class StrictABC(ABC):
    def __init_subclass__(cls, **kwargs: object) -> None:
        super().__init_subclass__(**kwargs)
        for name in getattr(cls, "__abstractmethods__", set()) - set(cls.__dict__):
            pass  # still abstract
        for name, method in cls.__dict__.items():
            if name == "compute" and isinstance(method, staticmethod):
                raise TypeError(f"{cls.__name__}.compute must be an instance method")

    @abstractmethod
    def compute(self, x: int) -> int: ...


class GoodImpl(StrictABC):
    def compute(self, x: int) -> int: return x * 2

try:
    class BadImpl(StrictABC):
        @staticmethod
        def compute(x: int) -> int: return x * 2
except TypeError as e:
    print(e)
```

---

**Scenario 9: You need a collection of heterogeneous objects that all share the same interface to be processed in a pipeline. How do you design and type this?**

**Q:** Design the pipeline with abstraction.

**A:**

```python
from abc import ABC, abstractmethod
from typing import TypeVar

T = TypeVar("T")


class Transformer(ABC):
    @abstractmethod
    def transform(self, data: list) -> list: ...


class FilterTransformer(Transformer):
    def __init__(self, predicate) -> None:
        self.predicate = predicate

    def transform(self, data: list) -> list:
        return [x for x in data if self.predicate(x)]


class MapTransformer(Transformer):
    def __init__(self, func) -> None:
        self.func = func

    def transform(self, data: list) -> list:
        return [self.func(x) for x in data]


class Pipeline:
    def __init__(self, stages: list[Transformer]) -> None:
        self.stages = stages

    def run(self, data: list) -> list:
        for stage in self.stages:
            data = stage.transform(data)
        return data


result = Pipeline([
    FilterTransformer(lambda x: x > 0),
    MapTransformer(lambda x: x * 2),
]).run([-1, 1, 2, -3, 4])
print(result)   # [2, 4, 8]
```

---

**Scenario 10: When would you choose to use `ABC` over `Protocol` in a public library?**

**Q:** Provide a justified recommendation.

**A:** Choose **ABC** for a public library when:
1. You provide a *partial implementation* (concrete methods) that subclasses can reuse.
2. You want instantiation errors to surface at *class definition time*, not at *first use*.
3. You need `isinstance`/`issubclass` to work reliably at runtime (e.g., a plugin system).
4. Users of the library are *expected to inherit*, and the inheritance hierarchy is part of the documented API.

Choose **Protocol** when:
1. You want to accept any structurally compatible object without forcing inheritance.
2. You're writing consumer-side type hints (you consume objects, not extend them).
3. You want third-party classes to satisfy your interface without modification.

```python
# Library ABC — users inherit, get concrete helpers, get early errors
class BaseHandler(ABC):
    @abstractmethod
    def handle(self, request: dict) -> dict: ...

    def log_and_handle(self, req: dict) -> dict:   # concrete helper
        import logging; logging.info(req)
        return self.handle(req)

# Library Protocol — users pass in anything compatible
from typing import Protocol
class Handler(Protocol):
    def handle(self, request: dict) -> dict: ...
```

---

## 4. Quick Recap / Cheatsheet

- **Abstraction** = hiding implementation behind a simple interface; defines *what*, not *how*.
- **`from abc import ABC, abstractmethod`** — the standard imports.
- **`ABC`** = base class that uses `ABCMeta` as metaclass.
- **`@abstractmethod`** — marks a method that concrete subclasses MUST implement.
- **`__abstractmethods__`** — frozenset on the class; CPython checks this in `type.__call__`.
- **Instantiation prevention**: if `cls.__abstractmethods__` is non-empty, `type.__call__` raises `TypeError`.
- **Abstract property**: `@property @abstractmethod` (order matters — `@property` outer).
- **Abstract classmethod**: `@classmethod @abstractmethod`.
- **`ABC.register(cls)`** — declares a virtual subclass without inheritance; no method verification.
- **`__subclasshook__`** — customise `isinstance`/`issubclass` logic structurally.
- **`Protocol` (PEP 544)** — structural subtyping, no inheritance required, static checking by mypy/pyright.
- **`@runtime_checkable`** — makes Protocol work with `isinstance` at runtime (checks only method presence, not signatures).
- **ABC vs Protocol**: ABC for hierarchies/frameworks; Protocol for flexible duck typing with type safety.
- **Template Method pattern**: ABC provides concrete methods that call abstract methods = partial implementation.
- **Partial subclass**: if a subclass doesn't implement all abstract methods, it remains abstract.
- **`collections.abc`** — rich set of built-in ABCs (`Iterable`, `Mapping`, `Sequence`, etc.) using `__subclasshook__`.
