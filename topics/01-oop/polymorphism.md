# Polymorphism

## 1. Concept (Deep Explanation)

### What Is Polymorphism?

Polymorphism (from Greek: *poly* = many, *morphe* = form) is the ability for objects of different types to respond to the **same interface** in **type-specific ways**. It is the third pillar of OOP (alongside encapsulation and inheritance).

In Python, polymorphism appears in several forms:
1. **Runtime polymorphism via method overriding** — subclasses override parent methods.
2. **Duck typing** — if it walks like a duck and quacks like a duck, it's treated as a duck.
3. **Operator overloading via dunder methods** — `+`, `*`, `[]`, etc. work on custom types.
4. **`functools.singledispatch`** — function-based polymorphism on argument type.
5. **`Protocol`-based structural polymorphism** — static structural subtyping.

### Runtime Polymorphism: Method Overriding

```python
from __future__ import annotations
import math


class Shape:
    def area(self) -> float:
        raise NotImplementedError

    def describe(self) -> str:
        return f"{type(self).__name__} with area {self.area():.2f}"


class Circle(Shape):
    def __init__(self, radius: float) -> None:
        self.radius = radius

    def area(self) -> float:
        return math.pi * self.radius ** 2


class Rectangle(Shape):
    def __init__(self, w: float, h: float) -> None:
        self.w = w
        self.h = h

    def area(self) -> float:
        return self.w * self.h


class Triangle(Shape):
    def __init__(self, base: float, height: float) -> None:
        self.base   = base
        self.height = height

    def area(self) -> float:
        return 0.5 * self.base * self.height


shapes: list[Shape] = [Circle(5), Rectangle(4, 6), Triangle(3, 8)]

for shape in shapes:
    print(shape.describe())
# Circle with area 78.54
# Rectangle with area 24.00
# Triangle with area 12.00
```

The `describe()` method is defined once in `Shape`, but each call to `self.area()` dispatches to the correct subclass — this is **runtime polymorphism**.

### Duck Typing: Python's Primary Polymorphism

Python does not require objects to share a type hierarchy. If an object has the required methods, it works:

```python
class Dog:
    def speak(self) -> str: return "Woof"

class Cat:
    def speak(self) -> str: return "Meow"

class Robot:
    def speak(self) -> str: return "Beep boop"

def make_noise(things: list) -> None:
    for thing in things:
        print(thing.speak())   # no isinstance check — duck typing!

make_noise([Dog(), Cat(), Robot()])
# Woof
# Meow
# Beep boop
```

Duck typing is more flexible than inheritance-based polymorphism because it requires no common ancestor class. It also makes code easier to extend — add a new type that implements `speak()` and it works without modification.

### Operator Overloading: Polymorphism via Dunder Methods

Dunder (double underscore) methods allow custom types to support operators, making built-in syntax polymorphic:

```python
from __future__ import annotations
from dataclasses import dataclass


@dataclass
class Vector:
    x: float
    y: float

    def __add__(self, other: "Vector") -> "Vector":
        return Vector(self.x + other.x, self.y + other.y)

    def __mul__(self, scalar: float) -> "Vector":
        return Vector(self.x * scalar, self.y * scalar)

    def __rmul__(self, scalar: float) -> "Vector":   # 3.0 * v
        return self.__mul__(scalar)

    def __abs__(self) -> float:
        return (self.x ** 2 + self.y ** 2) ** 0.5

    def __repr__(self) -> str:
        return f"Vector({self.x}, {self.y})"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Vector):
            return NotImplemented
        return self.x == other.x and self.y == other.y


v1 = Vector(1, 2)
v2 = Vector(3, 4)
print(v1 + v2)      # Vector(4, 6)
print(v1 * 3.0)     # Vector(3.0, 6.0)
print(3.0 * v1)     # Vector(3.0, 6.0) — via __rmul__
print(abs(v2))      # 5.0
print(v1 == v1)     # True
```

Important dunder methods for polymorphism:

| Operator | Dunder | Reverse |
|---|---|---|
| `+` | `__add__` | `__radd__` |
| `-` | `__sub__` | `__rsub__` |
| `*` | `__mul__` | `__rmul__` |
| `/` | `__truediv__` | `__rtruediv__` |
| `==` | `__eq__` | — |
| `<` | `__lt__` | `__gt__` |
| `[]` | `__getitem__` | — |
| `len()` | `__len__` | — |
| `str()` | `__str__` | — |
| `in` | `__contains__` | — |

### isinstance / type Checks as Anti-Patterns

Checking `type(obj)` or `isinstance(obj, SomeClass)` to dispatch behaviour is **usually a sign that polymorphism is missing**:

```python
# Anti-pattern: manual dispatch violates Open/Closed Principle
def process(shape):
    if type(shape) is Circle:
        return math.pi * shape.radius ** 2
    elif type(shape) is Rectangle:
        return shape.w * shape.h
    elif type(shape) is Triangle:
        return 0.5 * shape.base * shape.height
    else:
        raise TypeError(f"Unknown shape: {type(shape)}")

# Better: polymorphic dispatch
def process(shape: Shape) -> float:
    return shape.area()   # no type checks needed
```

When `isinstance` IS appropriate:
- Defensive programming in library code where the caller's type is unknown.
- Dunder methods returning `NotImplemented` for unknown types (correct pattern).
- Framework-level type dispatching (e.g., `functools.singledispatch`).

### functools.singledispatch: Function-Based Polymorphism

`@singledispatch` enables **function overloading** based on the type of the first argument:

```python
from functools import singledispatch
from typing import Union


@singledispatch
def serialize(obj: object) -> str:
    raise NotImplementedError(f"Cannot serialize {type(obj).__name__}")


@serialize.register(int)
def _(obj: int) -> str:
    return str(obj)


@serialize.register(float)
def _(obj: float) -> str:
    return f"{obj:.6f}"


@serialize.register(list)
def _(obj: list) -> str:
    return "[" + ", ".join(serialize(item) for item in obj) + "]"


@serialize.register(dict)
def _(obj: dict) -> str:
    pairs = ", ".join(f"{serialize(k)}: {serialize(v)}" for k, v in obj.items())
    return "{" + pairs + "}"


print(serialize(42))                   # "42"
print(serialize(3.14))                 # "3.140000"
print(serialize([1, 2.0, "skip"]))     # NotImplementedError for str
print(serialize({"a": 1, "b": 2.0}))  # "{a: 1, b: 2.000000}"

# Check registered implementations:
print(serialize.registry)
```

### Structural Polymorphism via Protocol (PEP 544)

`Protocol` enables static-analysis-time polymorphism without a class hierarchy:

```python
from typing import Protocol, runtime_checkable


@runtime_checkable
class Drawable(Protocol):
    def draw(self) -> None: ...
    def resize(self, factor: float) -> None: ...


class Widget:
    def draw(self) -> None: print("Widget drawn")
    def resize(self, factor: float) -> None: print(f"Widget resized by {factor}")


class SVGElement:
    def draw(self) -> None: print("SVG drawn")
    def resize(self, factor: float) -> None: print(f"SVG resized by {factor}")


def render(items: list[Drawable]) -> None:
    for item in items:
        item.draw()
        item.resize(1.5)


render([Widget(), SVGElement()])   # both work, no inheritance needed
print(isinstance(Widget(), Drawable))   # True (runtime_checkable)
```

### Python Polymorphism vs Java/C++ vtable

| Aspect | Python | Java / C++ |
|---|---|---|
| Dispatch mechanism | Dictionary lookup (MRO) | vtable (compile-time or JIT) |
| Duck typing | Yes — no type hierarchy needed | No — must implement interface/inherit |
| Operator overloading | Via dunder methods | Operator overloading (C++), limited (Java) |
| Performance | Slower (dict lookup per call) | Faster (vtable is a direct pointer) |
| Interface definition | ABC or Protocol | interface / abstract class |
| Runtime type checks | `isinstance`, `type()` | `instanceof`, `getClass()` |

Python's dispatch is slower per call but vastly more flexible.

---

## 2. Conceptual Interview Questions (10)

**Q1: What is polymorphism and what are its main forms in Python?**

**A:** Polymorphism means different objects respond to the same message (method call or operator) in type-specific ways. Python supports:

1. **Subtype polymorphism** — subclasses override methods.
2. **Duck typing** — structural compatibility, no inheritance required.
3. **Operator overloading** — dunder methods make custom types work with Python syntax.
4. **`singledispatch`** — type-based function dispatch.
5. **Protocol-based** — structural subtyping checked by type checkers.

```python
class Dog:
    def speak(self) -> str: return "Woof"

class Cat:
    def speak(self) -> str: return "Meow"

# Duck typing — no shared base class needed:
for animal in [Dog(), Cat()]:
    print(animal.speak())
```

---

**Q2: What is duck typing and how does it differ from inheritance-based polymorphism?**

**A:** Duck typing means Python checks for the *presence of required methods/attributes* at runtime, not the type hierarchy. An object "is-a" whatever it can *act like*.

Inheritance-based polymorphism requires a shared base class and is checked at compile/instantiation time. Duck typing is checked only at the moment of the method call.

```python
# Inheritance-based:
class Animal:
    def speak(self) -> str: ...

class Dog(Animal):
    def speak(self) -> str: return "Woof"

# Duck typing (no inheritance):
class Robot:
    def speak(self) -> str: return "Beep"

# Both work in make_noise() — duck typing doesn't care about the hierarchy
```

Duck typing is more flexible but less self-documenting. `Protocol` bridges both by providing type-safe duck typing.

---

**Q3: What does returning `NotImplemented` from a dunder method mean?**

**A:** Returning `NotImplemented` (not raising `NotImplementedError`) from a dunder method tells Python: "I don't know how to handle this type — try the reflected operation on the other operand."

```python
class Money:
    def __init__(self, amount: float, currency: str) -> None:
        self.amount   = amount
        self.currency = currency

    def __add__(self, other: "Money") -> "Money":
        if not isinstance(other, Money):
            return NotImplemented   # signals: try other.__radd__(self)
        if self.currency != other.currency:
            raise ValueError("Currency mismatch")
        return Money(self.amount + other.amount, self.currency)

m1 = Money(10.0, "USD")
m2 = Money(5.0, "USD")
print((m1 + m2).amount)   # 15.0

result = m1 + 5   # Python tries Money.__add__(m1, 5) → NotImplemented
                  # Then tries int.__radd__(5, m1) → also NotImplemented or TypeError
# TypeError: unsupported operand type(s) for +: 'Money' and 'int'
```

---

**Q4: How does `functools.singledispatch` implement polymorphism?**

**A:** `@singledispatch` maintains a registry mapping types to handler functions. When the decorated function is called, it looks up `type(arg)` (or its MRO) in the registry and dispatches to the most specific registered implementation.

```python
from functools import singledispatch

@singledispatch
def double(x):
    raise TypeError(f"Unsupported type: {type(x)}")

@double.register(int)
def _(x: int) -> int: return x * 2

@double.register(str)
def _(x: str) -> str: return x * 2

@double.register(list)
def _(x: list) -> list: return x * 2

print(double(5))          # 10
print(double("abc"))      # "abcabc"
print(double([1, 2, 3]))  # [1, 2, 3, 1, 2, 3]
```

For subclasses not explicitly registered, `singledispatch` walks the MRO to find the nearest registered ancestor.

---

**Q5: What is the difference between `__str__` and `__repr__`? Both are polymorphic — when is each called?**

**A:** Both are dunder methods enabling polymorphic string representation:

- `__repr__`: called by `repr()`, the interactive REPL, `%r` format, and as fallback for `str()`. Should be unambiguous and ideally produce a string that could `eval()` back to the object.
- `__str__`: called by `str()`, `print()`, `f"{obj}"`, `%s`. Should be human-readable.

```python
class Point:
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

    def __repr__(self) -> str:
        return f"Point({self.x!r}, {self.y!r})"

    def __str__(self) -> str:
        return f"({self.x}, {self.y})"

p = Point(1.0, 2.0)
print(repr(p))   # Point(1.0, 2.0)
print(str(p))    # (1.0, 2.0)
print(f"{p}")    # (1.0, 2.0)  — str() is called
print(f"{p!r}")  # Point(1.0, 2.0)  — repr() is called
```

---

**Q6: Why are isinstance() checks in dispatch logic often an anti-pattern?**

**A:** `isinstance` checks for dispatch violate the **Open/Closed Principle**: adding a new type requires modifying the dispatch function. Polymorphic method dispatch (or `singledispatch`) allows adding new types without touching existing code.

```python
# Anti-pattern (closed to extension):
def format_value(v):
    if isinstance(v, int):   return str(v)
    if isinstance(v, float): return f"{v:.2f}"
    if isinstance(v, list):  return "[" + ", ".join(map(format_value, v)) + "]"
    raise TypeError

# Open/closed with singledispatch:
from functools import singledispatch

@singledispatch
def format_value(v):
    raise TypeError(f"No formatter for {type(v)}")

@format_value.register(int)
def _(v): return str(v)

@format_value.register(float)
def _(v): return f"{v:.2f}"

# New type? Just register, no existing code touched:
@format_value.register(complex)
def _(v): return f"{v.real}+{v.imag}j"
```

---

**Q7: What is `__class_getitem__` and how does it relate to polymorphism?**

**A:** `__class_getitem__` is a classmethod called when a class is subscripted with `[]`. It enables **generic types**:

```python
class Stack:
    def __class_getitem__(cls, item):
        return type(f"Stack[{item.__name__}]", (cls,), {})

class IntStack(Stack[int]):   # syntactic sugar for Stack.__class_getitem__(int)
    pass

# More practically, Generic from typing uses __class_getitem__:
from typing import Generic, TypeVar
T = TypeVar("T")

class TypedStack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()

s: TypedStack[int] = TypedStack()
s.push(1)
print(s.pop())   # 1
```

---

**Q8: How does Python's `in` operator achieve polymorphism?**

**A:** `x in container` first tries `container.__contains__(x)`. If `__contains__` is not defined, Python falls back to iterating with `__iter__`. This is polymorphism via protocol:

```python
class Range:
    def __init__(self, start: int, stop: int) -> None:
        self.start = start
        self.stop  = stop

    def __contains__(self, item: int) -> bool:
        return self.start <= item < self.stop   # O(1), no iteration!

    def __iter__(self):
        return iter(range(self.start, self.stop))

r = Range(1, 100)
print(50 in r)    # True — uses __contains__ (O(1))
print(200 in r)   # False

# Fallback to __iter__ if __contains__ is absent:
class Squares:
    def __iter__(self):
        for i in range(10):
            yield i * i

print(9 in Squares())   # True — iterates and compares
```

---

**Q9: What is the difference between method overriding and method overloading in the context of polymorphism?**

**A:** 

- **Method overriding** (runtime polymorphism): a subclass provides its own implementation of a method inherited from a parent. Python resolves the method via MRO at runtime.
- **Method overloading** (compile-time polymorphism in other languages): multiple methods with the same name but different signatures. Python does **not** support this natively — later definitions replace earlier ones.

```python
# Method overriding (Python supports):
class Animal:
    def speak(self): return "..."

class Dog(Animal):
    def speak(self): return "Woof"  # overrides Animal.speak

# Method overloading (NOT supported natively):
class Calculator:
    def add(self, a, b):    return a + b
    def add(self, a, b, c): return a + b + c   # REPLACES the previous add!

print(Calculator().add(1, 2, 3))   # 6 — works
# Calculator().add(1, 2)  → TypeError: missing 1 argument
```

Python achieves similar effects via `*args`, `**kwargs`, defaults, or `singledispatch`.

---

**Q10: How does Python determine which `__add__` to call when two objects of different types are added?**

**A:** Python follows a specific protocol:
1. Call `left.__add__(right)`.
2. If it returns `NotImplemented`, call `right.__radd__(left)`.
3. If both return `NotImplemented`, raise `TypeError`.

If `right` is an instance of a *subclass* of `type(left)`, Python tries `right.__radd__` first (to allow subclasses to override parent addition).

```python
class Celsius:
    def __init__(self, v: float): self.v = v
    def __add__(self, other):
        if isinstance(other, Celsius): return Celsius(self.v + other.v)
        return NotImplemented
    def __radd__(self, other):
        if isinstance(other, (int, float)): return Celsius(self.v + other)
        return NotImplemented
    def __repr__(self): return f"Celsius({self.v})"

print(Celsius(10) + Celsius(20))   # Celsius(30)
print(5.0 + Celsius(10))           # Celsius(15.0) — via __radd__
```

---

## 3. Scenario-Based Interview Questions (10)

**Scenario 1: You're building a rendering engine that needs to draw many different shape types. How do you use polymorphism to avoid a long if-elif chain?**

**Q:** Design the polymorphic rendering system.

**A:**

```python
from abc import ABC, abstractmethod


class Renderer(ABC):
    @abstractmethod
    def draw(self, canvas) -> None: ...


class CircleRenderer(Renderer):
    def __init__(self, x: float, y: float, r: float) -> None:
        self.x, self.y, self.r = x, y, r

    def draw(self, canvas) -> None:
        canvas.draw_ellipse(self.x, self.y, self.r, self.r)


class LineRenderer(Renderer):
    def __init__(self, x1, y1, x2, y2) -> None:
        self.coords = (x1, y1, x2, y2)

    def draw(self, canvas) -> None:
        canvas.draw_line(*self.coords)


class CompositeRenderer(Renderer):
    def __init__(self, *parts: Renderer) -> None:
        self.parts = list(parts)

    def draw(self, canvas) -> None:
        for part in self.parts:
            part.draw(canvas)   # polymorphic dispatch — no type checks needed


class MockCanvas:
    def draw_ellipse(self, *args): print(f"Ellipse {args}")
    def draw_line(self, *args):    print(f"Line {args}")


c = MockCanvas()
scene = CompositeRenderer(
    CircleRenderer(0, 0, 5),
    LineRenderer(0, 0, 10, 10),
)
scene.draw(c)
```

---

**Scenario 2: You need to support both Python lists and custom linked lists in a function that processes sequences. How?**

**Q:** Use duck typing or Protocol to accept both.

**A:**

```python
from typing import Protocol, Iterator


class Sequence(Protocol):
    def __iter__(self) -> Iterator: ...
    def __len__(self) -> int: ...


class LinkedList:
    def __init__(self) -> None:
        self._items: list = []

    def append(self, item) -> None:
        self._items.append(item)

    def __iter__(self) -> Iterator:
        return iter(self._items)

    def __len__(self) -> int:
        return len(self._items)


def total(seq: Sequence) -> float:
    return sum(seq)


ll = LinkedList()
ll.append(1)
ll.append(2)
ll.append(3)

print(total([10, 20, 30]))   # 60 — built-in list
print(total(ll))             # 6  — LinkedList duck-typed as Sequence
```

---

**Scenario 3: A payment system processes charges via different gateways. Adding a new gateway should not require changing existing code. Design this.**

**Q:** Show an OCP-compliant polymorphic design.

**A:**

```python
from abc import ABC, abstractmethod
from decimal import Decimal


class Gateway(ABC):
    @abstractmethod
    def charge(self, amount: Decimal) -> str: ...


class StripeGateway(Gateway):
    def charge(self, amount: Decimal) -> str:
        return f"Stripe charged {amount}"


class PayPalGateway(Gateway):
    def charge(self, amount: Decimal) -> str:
        return f"PayPal charged {amount}"


class PaymentProcessor:
    def __init__(self, gateway: Gateway) -> None:
        self._gateway = gateway

    def process(self, amount: Decimal) -> str:
        return self._gateway.charge(amount)


# Adding Square (new gateway) — zero changes to PaymentProcessor:
class SquareGateway(Gateway):
    def charge(self, amount: Decimal) -> str:
        return f"Square charged {amount}"


for gw in [StripeGateway(), PayPalGateway(), SquareGateway()]:
    p = PaymentProcessor(gw)
    print(p.process(Decimal("99.99")))
```

---

**Scenario 4: You need to provide a pretty-printer for a data pipeline that handles ints, floats, lists, and custom objects. Use singledispatch.**

**Q:** Implement the pretty-printer.

**A:**

```python
from functools import singledispatch


@singledispatch
def pretty(obj: object) -> str:
    return f"<{type(obj).__name__}: {obj!r}>"


@pretty.register(int)
def _(obj: int) -> str:
    return f"int({obj})"


@pretty.register(float)
def _(obj: float) -> str:
    return f"float({obj:.3f})"


@pretty.register(list)
def _(obj: list) -> str:
    items = ", ".join(pretty(item) for item in obj)
    return f"[{items}]"


@pretty.register(dict)
def _(obj: dict) -> str:
    pairs = ", ".join(f"{pretty(k)}: {pretty(v)}" for k, v in obj.items())
    return f"{{{pairs}}}"


print(pretty(42))
print(pretty(3.14159))
print(pretty([1, 2.5, {"a": 10}]))
# int(42)
# float(3.142)
# [int(1), float(2.500), {str('a'): int(10)}]  (str not registered — falls back to default)
```

---

**Scenario 5: A JSON serialiser needs to handle new custom types added by users. How do you design an extensible serialiser using polymorphism?**

**Q:** Design the extensible serialiser.

**A:**

```python
from functools import singledispatch
import json
from datetime import date, datetime
from decimal import Decimal


@singledispatch
def to_json_type(obj: object):
    raise TypeError(f"Object of type {type(obj).__name__} is not JSON serializable")


@to_json_type.register(date)
def _(obj: date) -> str:
    return obj.isoformat()


@to_json_type.register(datetime)
def _(obj: datetime) -> str:
    return obj.isoformat()


@to_json_type.register(Decimal)
def _(obj: Decimal) -> float:
    return float(obj)


class CustomEncoder(json.JSONEncoder):
    def default(self, obj):
        try:
            return to_json_type(obj)
        except TypeError:
            return super().default(obj)


data = {
    "date": date(2025, 1, 1),
    "price": Decimal("19.99"),
    "name": "Widget",
}
print(json.dumps(data, cls=CustomEncoder))
# {"date": "2025-01-01", "price": 19.99, "name": "Widget"}

# User extends without modifying existing code:
class Money:
    def __init__(self, amount, currency): self.amount = amount; self.currency = currency

@to_json_type.register(Money)
def _(obj: Money) -> dict:
    return {"amount": float(obj.amount), "currency": obj.currency}
```

---

**Scenario 6: You have a `sort` function that works on any object that supports comparison operators. How does Python make this work polymorphically?**

**Q:** Explain and show.

**A:** Python's `sort()` uses `__lt__` (`<`). Any object implementing `__lt__` is sortable. This is structural polymorphism via the comparison protocol:

```python
from functools import total_ordering


@total_ordering
class Version:
    def __init__(self, version_str: str) -> None:
        self.parts = tuple(int(x) for x in version_str.split("."))

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Version): return NotImplemented
        return self.parts == other.parts

    def __lt__(self, other: "Version") -> bool:
        if not isinstance(other, Version): return NotImplemented
        return self.parts < other.parts

    def __repr__(self) -> str:
        return ".".join(map(str, self.parts))


versions = [Version("2.0.0"), Version("1.9.1"), Version("1.10.0"), Version("3.0.0")]
print(sorted(versions))   # [1.9.1, 1.10.0, 2.0.0, 3.0.0]
print(max(versions))      # 3.0.0
```

`@total_ordering` auto-derives `__le__`, `__gt__`, `__ge__` from `__eq__` and `__lt__`.

---

**Scenario 7: You're writing a function that needs to apply a transformation to a value, but the transformation differs by type. Show two approaches.**

**Q:** Compare isinstance-based and singledispatch approaches.

**A:**

```python
# Approach 1: isinstance (brittle)
def transform_v1(value):
    if isinstance(value, str):  return value.upper()
    if isinstance(value, int):  return value * 2
    if isinstance(value, list): return [transform_v1(v) for v in value]
    raise TypeError(f"Unsupported: {type(value)}")

# Approach 2: singledispatch (extensible)
from functools import singledispatch

@singledispatch
def transform(value): raise TypeError(f"Unsupported: {type(value)}")

@transform.register(str)
def _(v: str): return v.upper()

@transform.register(int)
def _(v: int): return v * 2

@transform.register(list)
def _(v: list): return [transform(item) for item in v]

print(transform("hello"))       # HELLO
print(transform(21))            # 42
print(transform([1, "abc", 2])) # [2, 'ABC', 4]

# Adding float support — no existing code modified:
@transform.register(float)
def _(v: float): return round(v, 2)
```

---

**Scenario 8: How do you implement a polymorphic `__enter__` and `__exit__` for context managers?**

**Q:** Show context manager polymorphism.

**A:**

```python
from abc import ABC, abstractmethod
from types import TracebackType
from typing import Optional, Type


class ManagedResource(ABC):
    def __enter__(self) -> "ManagedResource":
        self.open()
        return self

    def __exit__(
        self,
        exc_type: Optional[Type[BaseException]],
        exc_val: Optional[BaseException],
        exc_tb: Optional[TracebackType],
    ) -> bool:
        self.close()
        return False   # don't suppress exceptions

    @abstractmethod
    def open(self) -> None: ...

    @abstractmethod
    def close(self) -> None: ...

    @abstractmethod
    def read(self) -> str: ...


class FileResource(ManagedResource):
    def __init__(self, path: str) -> None: self.path = path
    def open(self) -> None: print(f"Opening {self.path}")
    def close(self) -> None: print(f"Closing {self.path}")
    def read(self) -> str: return "file content"


class NetworkResource(ManagedResource):
    def __init__(self, url: str) -> None: self.url = url
    def open(self) -> None: print(f"Connecting to {self.url}")
    def close(self) -> None: print(f"Disconnecting from {self.url}")
    def read(self) -> str: return "network data"


def process(resource: ManagedResource) -> None:
    with resource as r:
        print(r.read())   # polymorphic read()


process(FileResource("/tmp/data.txt"))
process(NetworkResource("https://api.example.com"))
```

---

**Scenario 9: Your team debates whether to use ABC or Protocol for a new plugin interface. They're both acceptable, but which fits better and why?**

**Q:** Provide the recommendation.

**A:**

```python
# Use ABC when: plugin authors WILL subclass your class and you provide shared behaviour
from abc import ABC, abstractmethod

class ProcessorABC(ABC):
    def __init__(self, config: dict) -> None:  # shared __init__
        self.config = config

    @abstractmethod
    def process(self, data: bytes) -> bytes: ...

    def log(self, msg: str) -> None:   # shared concrete method
        print(f"[{type(self).__name__}] {msg}")


# Use Protocol when: plugin authors control their own class hierarchy
from typing import Protocol

class ProcessorProtocol(Protocol):
    def process(self, data: bytes) -> bytes: ...


# With Protocol, a class that already inherits from something else can satisfy the interface:
class ThirdPartyProcessor:
    def process(self, data: bytes) -> bytes:
        return data.upper()

def run(processor: ProcessorProtocol, data: bytes) -> None:
    print(processor.process(data))

run(ThirdPartyProcessor(), b"hello")  # Works — structural typing
```

---

**Scenario 10: You need to add `__eq__` and `__hash__` to a class to use it in sets/dicts. Explain the polymorphism and Python's consistency requirement.**

**Q:** Implement both correctly.

**A:** Python requires: if `a == b`, then `hash(a) == hash(b)`. Objects that define `__eq__` must also define `__hash__` (Python sets `__hash__ = None` automatically if you define `__eq__` without `__hash__`, making the object unhashable).

```python
class Color:
    def __init__(self, r: int, g: int, b: int) -> None:
        self.r = r
        self.g = g
        self.b = b

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Color):
            return NotImplemented
        return (self.r, self.g, self.b) == (other.r, other.g, other.b)

    def __hash__(self) -> int:
        return hash((self.r, self.g, self.b))   # consistent with __eq__

    def __repr__(self) -> str:
        return f"Color({self.r}, {self.g}, {self.b})"


red   = Color(255, 0, 0)
red2  = Color(255, 0, 0)
blue  = Color(0, 0, 255)

print(red == red2)           # True
print(hash(red) == hash(red2))  # True — consistent
print({red, red2, blue})    # {Color(255, 0, 0), Color(0, 0, 255)} — deduped correctly
print({red: "primary"})     # dict usage
```

---

## 4. Quick Recap / Cheatsheet

- **Polymorphism** = same interface, different behaviour per type.
- **Runtime polymorphism**: subclass overrides parent method; resolved at runtime via MRO.
- **Duck typing**: Python checks method/attribute *presence*, not type hierarchy. Most Pythonic form.
- **Operator overloading**: dunder methods (`__add__`, `__lt__`, `__len__`, etc.) make custom types work with operators.
- **`NotImplemented`** (not `NotImplementedError`): return from dunder methods to signal "try the reflected operation."
- **`isinstance` dispatch anti-pattern**: violates Open/Closed Principle; prefer polymorphic methods or `singledispatch`.
- **`functools.singledispatch`**: function-based dispatch on the type of the first argument. Extensible without modifying existing code.
- **`functools.singledispatchmethod`** (Python 3.8+): same but for methods.
- **`Protocol` (PEP 544)**: structural subtyping; no inheritance required; checked statically by mypy/pyright.
- **`@runtime_checkable` Protocol**: enables `isinstance` checks (only checks method names, not signatures).
- **`@total_ordering`**: derive all 6 comparison methods from `__eq__` + `__lt__`.
- **`__eq__` + `__hash__` rule**: if A == B, then hash(A) == hash(B). Always define both together.
- **Python vs Java**: Python uses dict-based MRO lookup (flexible but slower); Java uses vtable (fast but rigid).
- **Composite pattern**: `CompositeRenderer.draw()` calls `draw()` on each child — polymorphism enables recursion without type checks.
- **Context manager polymorphism**: `__enter__`/`__exit__` protocol enables uniform `with` statement use.
