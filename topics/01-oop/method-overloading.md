# Method Overloading

## 1. Concept (Deep Explanation)

### What Is Method Overloading?

In languages like C++ and Java, **method overloading** means defining multiple methods with the **same name** but **different parameter signatures** (number or type of parameters). The compiler resolves which implementation to call at *compile time* based on argument types.

```cpp
// C++ — compile-time overloading:
int add(int a, int b) { return a + b; }
double add(double a, double b) { return a + b; }
std::string add(std::string a, std::string b) { return a + b; }
```

**Python does NOT support this.** If you define a function with the same name twice, the second definition simply *replaces* the first. There is no error, no disambiguation — the later binding wins.

```python
class Calculator:
    def add(self, a, b):
        return a + b

    def add(self, a, b, c):        # REPLACES the previous add!
        return a + b + c

calc = Calculator()
print(calc.add(1, 2, 3))           # 6 — works
# calc.add(1, 2)                   # TypeError: add() missing 1 required argument
```

Python's dynamic nature (one namespace, late binding) means there is only one `add` in `Calculator.__dict__` at any time.

### Why Python Doesn't Need Traditional Overloading

Python achieves the same goals through more flexible mechanisms:
1. **Default parameter values**
2. **`*args` and `**kwargs`**
3. **Runtime `isinstance` checks** (simple but not recommended for complex dispatch)
4. **`functools.singledispatch`** — type-based dispatch for standalone functions
5. **`functools.singledispatchmethod`** — type-based dispatch for methods (Python 3.8+)
6. **`typing.overload`** — type-checker hints only, no runtime effect
7. **Multiple dispatch libraries** (e.g., `multipledispatch`)

### Approach 1: Default Parameters

```python
class Greeter:
    def greet(self, name: str, greeting: str = "Hello", punctuation: str = "!") -> str:
        return f"{greeting}, {name}{punctuation}"

g = Greeter()
print(g.greet("Alice"))                         # "Hello, Alice!"
print(g.greet("Bob", "Hi"))                     # "Hi, Bob!"
print(g.greet("Carol", "Hey", "."))             # "Hey, Carol."
```

Limitations: all parameters must be compatible types; you can't have completely different behaviours.

### Approach 2: *args and **kwargs

```python
class Adder:
    def add(self, *args: int | float) -> int | float:
        if not args:
            raise TypeError("add() requires at least one argument")
        return sum(args)

a = Adder()
print(a.add(1, 2))           # 3
print(a.add(1, 2, 3))        # 6
print(a.add(1, 2, 3, 4, 5))  # 15
```

### Approach 3: isinstance Dispatch (Simple Cases)

```python
class Formatter:
    def format(self, value: int | float | str | list) -> str:
        if isinstance(value, int):
            return f"INT({value})"
        elif isinstance(value, float):
            return f"FLOAT({value:.2f})"
        elif isinstance(value, str):
            return f"STR({value!r})"
        elif isinstance(value, list):
            items = ", ".join(self.format(v) for v in value)
            return f"LIST[{items}]"
        else:
            raise TypeError(f"Unsupported type: {type(value).__name__}")

f = Formatter()
print(f.format(42))                   # INT(42)
print(f.format(3.14))                 # FLOAT(3.14)
print(f.format([1, "abc", 2.0]))     # LIST[INT(1), STR('abc'), FLOAT(2.00)]
```

This pattern violates the Open/Closed Principle — adding a new type requires modifying `format()`.

### Approach 4: functools.singledispatch (Standalone Functions)

`@singledispatch` is the canonical Python solution for type-based dispatch on standalone functions:

```python
from functools import singledispatch
from datetime import date, datetime
from decimal import Decimal


@singledispatch
def serialize(obj: object) -> str:
    """Default implementation — raises for unknown types."""
    raise TypeError(f"Cannot serialize {type(obj).__name__}")


@serialize.register(int)
def _(obj: int) -> str:
    return str(obj)


@serialize.register(float)
def _(obj: float) -> str:
    return f"{obj:.6f}"


@serialize.register(str)
def _(obj: str) -> str:
    return f'"{obj}"'


@serialize.register(date)
def _(obj: date) -> str:
    return obj.isoformat()


@serialize.register(datetime)  # more specific than date — registered separately
def _(obj: datetime) -> str:
    return obj.isoformat(sep="T")


@serialize.register(list)
def _(obj: list) -> str:
    return "[" + ", ".join(serialize(item) for item in obj) + "]"


@serialize.register(Decimal)
def _(obj: Decimal) -> str:
    return str(obj)


print(serialize(42))                     # "42"
print(serialize(3.14))                   # "3.140000"
print(serialize("hello"))               # '"hello"'
print(serialize(date(2025, 1, 1)))      # "2025-01-01"
print(serialize([1, "a", 3.14]))        # '[1, "a", 3.140000]'

# MRO-aware dispatch: subclass resolution
print(serialize.dispatch(int))           # the registered int handler
print(serialize.dispatch(bool))          # bool subclasses int — finds int handler
print(bool in serialize.registry)        # False — but dispatch walks MRO
```

**Key features of singledispatch:**
- Dispatches on the **type of the first argument**.
- MRO-aware: if `bool` is not registered but `int` is, `bool` maps to the `int` handler.
- `@singledispatch` with `Union` types (Python 3.11+): register with `register(int | str)`.
- `serialize.registry` exposes the dispatch table.
- `serialize.dispatch(some_type)` returns the implementation for a given type.

### Approach 5: functools.singledispatchmethod (Python 3.8+)

For **methods** on a class, use `singledispatchmethod`:

```python
from functools import singledispatchmethod


class Printer:
    @singledispatchmethod
    def print_value(self, arg: object) -> None:
        raise NotImplementedError(f"Cannot print {type(arg).__name__}")

    @print_value.register(int)
    def _(self, arg: int) -> None:
        print(f"Integer: {arg}")

    @print_value.register(str)
    def _(self, arg: str) -> None:
        print(f"String: {arg!r}")

    @print_value.register(list)
    def _(self, arg: list) -> None:
        print(f"List with {len(arg)} items")


p = Printer()
p.print_value(42)            # "Integer: 42"
p.print_value("hello")       # "String: 'hello'"
p.print_value([1, 2, 3])    # "List with 3 items"
```

**`singledispatchmethod` with classmethod:** Since Python 3.8, you can stack it with `@classmethod`:

```python
from functools import singledispatchmethod


class Parser:
    @singledispatchmethod
    @classmethod
    def parse(cls, data: object):
        raise NotImplementedError

    @parse.register(str)
    @classmethod
    def _(cls, data: str):
        return {"source": "string", "value": data}

    @parse.register(bytes)
    @classmethod
    def _(cls, data: bytes):
        return {"source": "bytes", "value": data.decode()}
```

### Approach 6: typing.overload (Type Checkers Only)

`@typing.overload` is purely for **type checker hints** — it has **no runtime effect**. Use it to give mypy/pyright precise knowledge about which overload corresponds to which argument types:

```python
from typing import overload


class Converter:
    @overload
    def convert(self, value: int) -> str: ...

    @overload
    def convert(self, value: str) -> int: ...

    def convert(self, value: int | str) -> str | int:
        """Actual implementation — handles both overloads."""
        if isinstance(value, int):
            return str(value)
        return int(value)


c = Converter()
result1: str = c.convert(42)     # mypy knows this returns str
result2: int = c.convert("99")   # mypy knows this returns int
print(result1, result2)          # "42" 99
```

At runtime, only the un-decorated `convert` function exists. The `@overload` decorated versions are discarded.

**Important:** The `@overload` signatures must come *before* the actual implementation. The implementation must NOT have the `@overload` decorator.

### Comparison with C++ and Java

**C++ Overloading (compile-time):**
```cpp
// Resolved at compile time based on argument types — zero runtime cost
int   square(int x)   { return x * x; }
float square(float x) { return x * x; }

square(5);    // → int version
square(5.0f); // → float version
```

**Java Overloading (compile-time):**
```java
// Method selected at compile time; runtime dispatches via vtable for virtual methods only
void print(int x) { System.out.println("int: " + x); }
void print(String x) { System.out.println("str: " + x); }
```

**Python Equivalents:**
- Python's `*args`/`**kwargs` handles variable arity.
- `singledispatch` handles type-based dispatch at *runtime* (not compile-time).
- `typing.overload` provides the type-checker-visible "overloaded" view.

### Multiple Dispatch Libraries

The `multipledispatch` library (not stdlib) extends `singledispatch` to dispatch on **multiple argument types**:

```python
# pip install multipledispatch
from multipledispatch import dispatch


@dispatch(int, int)
def add(a, b):
    return a + b


@dispatch(str, str)
def add(a, b):
    return a + b


@dispatch(int, str)
def add(a, b):
    return str(a) + b


print(add(1, 2))         # 3
print(add("a", "b"))     # "ab"
print(add(1, "b"))       # "1b"
```

### When Overloading Is Needed vs Separate Methods

Prefer **separate, clearly named methods** over overloading when:
- The behaviours are sufficiently different to deserve different names.
- The number of type combinations is small.
- Clarity matters more than API uniformity.

```python
# Clearer with separate methods:
class DataStore:
    def store_string(self, key: str, value: str) -> None: ...
    def store_bytes(self, key: str, value: bytes) -> None: ...
    def store_json(self, key: str, value: dict) -> None: ...

# Use overload/singledispatch when: the operation is genuinely the same concept:
class DataStore:
    @singledispatchmethod
    def store(self, key: str, value: object) -> None:
        raise NotImplementedError

    @store.register(str)
    def _(self, key: str, value: str) -> None: ...

    @store.register(bytes)
    def _(self, key: str, value: bytes) -> None: ...
```

---

## 2. Conceptual Interview Questions (10)

**Q1: Does Python support method overloading? What happens if you define the same method name twice?**

**A:** Python does **not** support traditional method overloading. A class body is executed as a dictionary assignment; when you define `def foo(self, a, b)` and then `def foo(self, a, b, c)`, the second definition simply overwrites the first in the class's `__dict__`.

```python
class Oops:
    def greet(self, name: str) -> str:
        return f"Hello, {name}!"

    def greet(self, name: str, times: int) -> str:   # replaces previous!
        return f"Hello, {name}! " * times

o = Oops()
# o.greet("Alice")         # TypeError — needs 2 args now
print(o.greet("Bob", 2))  # "Hello, Bob! Hello, Bob! "
```

Python achieves overloading-like behaviour through default arguments, `*args`, `**kwargs`, `singledispatch`, or `typing.overload`.

---

**Q2: What is `functools.singledispatch` and how does it work?**

**A:** `@singledispatch` is a decorator that transforms a function into a *single-dispatch generic function*. It maintains an internal registry (`{type: callable}`) and dispatches to the registered implementation based on the *type of the first argument*.

If the exact type is not registered, it walks the type's MRO until it finds a match (falling back to the base implementation if none is found).

```python
from functools import singledispatch

@singledispatch
def process(obj):
    return f"default: {obj!r}"

@process.register(int)
def _(obj: int):
    return f"int: {obj * 2}"

@process.register(float)
def _(obj: float):
    return f"float: {obj:.2f}"

print(process(5))       # "int: 10"
print(process(3.14))    # "float: 3.14"
print(process("hi"))    # "default: 'hi'"
print(process(True))    # "int: 2"  — bool subclasses int, uses int handler
```

---

**Q3: What is the difference between `functools.singledispatch` and `functools.singledispatchmethod`?**

**A:** `singledispatch` works on **module-level functions**. It dispatches on the type of the *first* argument. However, if used directly as a decorator on a method, `self` would be the first argument — and you'd be dispatching on `self`'s type, not the data argument.

`singledispatchmethod` (Python 3.8+) is designed for **instance methods** and **classmethods**. It correctly handles `self`/`cls` and dispatches on the *second* argument (the first after `self`/`cls`).

```python
from functools import singledispatch, singledispatchmethod


# singledispatch on a method — WRONG (dispatches on self):
class Bad:
    @singledispatch
    def process(self, value):           # dispatches on type(self), not type(value)!
        return "default"


# singledispatchmethod — CORRECT:
class Good:
    @singledispatchmethod
    def process(self, value):
        return "default"

    @process.register(int)
    def _(self, value: int):
        return f"int: {value}"

g = Good()
print(g.process(42))      # "int: 42"
print(g.process("hi"))    # "default"
```

---

**Q4: What does `@typing.overload` do? Does it have any runtime effect?**

**A:** `@typing.overload` has **no runtime effect**. The `@overload`-decorated versions are stored and then discarded; only the final implementation (without `@overload`) is installed in the class/module namespace.

Its purpose is to give **type checkers** (mypy, pyright) precise information about the relationship between argument types and return types:

```python
from typing import overload

@overload
def parse(value: str) -> int: ...
@overload
def parse(value: bytes) -> str: ...

def parse(value: str | bytes) -> int | str:
    if isinstance(value, str):
        return int(value)
    return value.decode()

# At runtime: only one 'parse' function exists
# Type checker: knows parse(str) -> int and parse(bytes) -> str
reveal_type(parse("42"))     # mypy: int
reveal_type(parse(b"hello")) # mypy: str
```

---

**Q5: How does singledispatch handle inheritance? If bool is not registered, what handler runs?**

**A:** `singledispatch` walks the MRO of the argument type to find the nearest registered ancestor. Since `bool` is a subclass of `int`, and `int` is typically registered, `bool` values use the `int` handler.

```python
from functools import singledispatch

@singledispatch
def describe(x):
    return f"object: {x}"

@describe.register(int)
def _(x: int): return f"int: {x}"

print(describe(True))   # "int: True"  — bool subclasses int
print(describe(False))  # "int: False"

# Register bool explicitly to override:
@describe.register(bool)
def _(x: bool): return f"bool: {x}"

print(describe(True))   # "bool: True"  — now has specific handler
```

---

**Q6: How do you implement overloading based on the number of arguments in Python?**

**A:** Use default parameters or `*args`:

```python
class Geometry:
    def area(self, *args: float) -> float:
        match len(args):
            case 1:
                import math
                return math.pi * args[0] ** 2    # circle: area(radius)
            case 2:
                return args[0] * args[1]         # rectangle: area(w, h)
            case 3:
                # triangle: area(a, b, c) via Heron's formula
                a, b, c = args
                s = (a + b + c) / 2
                return (s * (s-a) * (s-b) * (s-c)) ** 0.5
            case _:
                raise TypeError(f"area() takes 1, 2, or 3 arguments, got {len(args)}")

g = Geometry()
print(f"{g.area(5):.2f}")         # circle: 78.54
print(g.area(4, 6))               # rectangle: 24.0
print(f"{g.area(3, 4, 5):.2f}")   # triangle: 6.00
```

Using `match/case` (Python 3.10+) on `len(args)` is clean and explicit.

---

**Q7: What is multiple dispatch and how do you implement it in Python?**

**A:** Multiple dispatch selects the implementation based on the types of *multiple* arguments (not just the first). Python's stdlib only supports single dispatch. Multiple dispatch requires either:
1. Manual `isinstance` checks in a single function.
2. The `multipledispatch` library.
3. A custom dispatch table.

```python
# Manual multiple dispatch:
def combine(a, b):
    if isinstance(a, int) and isinstance(b, int):
        return a + b
    if isinstance(a, str) and isinstance(b, str):
        return a + b
    if isinstance(a, int) and isinstance(b, str):
        return str(a) + b
    raise TypeError(f"Cannot combine {type(a).__name__} and {type(b).__name__}")

# With multipledispatch library:
# from multipledispatch import dispatch
# @dispatch(int, int)
# def combine(a, b): return a + b
# @dispatch(str, str)
# def combine(a, b): return a + b
```

---

**Q8: When is it better to use separate, explicitly-named methods instead of overloading?**

**A:** Separate methods are better when:
1. The behaviour is meaningfully different — different names communicate intent.
2. The type combinations are limited and well-known.
3. The methods have very different signatures that `@overload` can't cleanly unify.
4. The callers benefit from explicit naming (reduces accidental misuse).

```python
# Overload disguises what should be explicit:
class Connection:
    def send(self, data: str | bytes | dict) -> None:
        if isinstance(data, str): ...
        elif isinstance(data, bytes): ...
        elif isinstance(data, dict): ...

# Separate methods — clearer:
class Connection:
    def send_text(self, data: str) -> None: ...
    def send_bytes(self, data: bytes) -> None: ...
    def send_json(self, data: dict) -> None: ...
```

---

**Q9: How does `typing.overload` interact with `@classmethod` and `@staticmethod`?**

**A:** Stack the decorators carefully:

```python
from typing import overload


class Parser:
    @overload
    @classmethod
    def parse(cls, value: str) -> int: ...

    @overload
    @classmethod
    def parse(cls, value: bytes) -> str: ...

    @classmethod
    def parse(cls, value: str | bytes) -> int | str:
        if isinstance(value, str):
            return int(value)
        return value.decode()

    @overload
    @staticmethod
    def validate(value: str) -> bool: ...

    @overload
    @staticmethod
    def validate(value: int) -> bool: ...

    @staticmethod
    def validate(value: str | int) -> bool:
        if isinstance(value, str): return len(value) > 0
        return value >= 0
```

---

**Q10: What are the performance implications of singledispatch vs isinstance checks?**

**A:** `singledispatch` uses a dictionary lookup (O(1) for exact type, O(MRO depth) for inherited types), which is fast but has a small overhead for the dispatch mechanism itself. Direct `isinstance` chains are slightly faster for very small numbers of types but become O(n) for n types. `singledispatch` scales better for many types.

In hot paths, consider:
1. For 2–3 types: explicit `isinstance` is readable and fast.
2. For 4+ types: `singledispatch` is cleaner and scales better.
3. For performance-critical code: profile before optimising; the overhead of both is usually negligible compared to I/O or computation.

```python
import timeit

from functools import singledispatch

@singledispatch
def sd(x): return "obj"

@sd.register(int)
def _(x): return "int"

def ischeck(x):
    if isinstance(x, int): return "int"
    return "obj"

# Microbenchmark (rough):
print(timeit.timeit(lambda: sd(42), number=1_000_000))     # ~0.15s
print(timeit.timeit(lambda: ischeck(42), number=1_000_000)) # ~0.07s
# singledispatch ~2x slower per call, but negligible in real workloads
```

---

## 3. Scenario-Based Interview Questions (10)

**Scenario 1: You're writing a library function `read_data(source)` that should accept a file path string, a file-like object, or raw bytes. Implement it cleanly.**

**Q:** Use singledispatch to handle the three cases.

**A:**

```python
from functools import singledispatch
from pathlib import Path
import io


@singledispatch
def read_data(source: object) -> bytes:
    raise TypeError(f"Unsupported source type: {type(source).__name__}")


@read_data.register(str)
@read_data.register(Path)
def _(source: str | Path) -> bytes:
    with open(source, "rb") as f:
        return f.read()


@read_data.register(bytes)
def _(source: bytes) -> bytes:
    return source


@read_data.register(io.RawIOBase)
@read_data.register(io.BufferedIOBase)
def _(source) -> bytes:
    return source.read()


# Usage:
data1 = read_data("config.json")                  # from file path
data2 = read_data(b"\x00\x01\x02")               # raw bytes
data3 = read_data(io.BytesIO(b"hello world"))     # file-like object
```

---

**Scenario 2: A team member insists on using method overloading via isinstance for a class with 5 different input types. You suggest singledispatch. Show the refactoring.**

**Q:** Before and after.

**A:**

```python
# Before: one big method with isinstance checks (hard to extend)
class Renderer:
    def render(self, data):
        if isinstance(data, str):
            return f"<p>{data}</p>"
        elif isinstance(data, int):
            return f"<code>{data}</code>"
        elif isinstance(data, list):
            items = "".join(f"<li>{self.render(item)}</li>" for item in data)
            return f"<ul>{items}</ul>"
        elif isinstance(data, dict):
            rows = "".join(f"<tr><td>{k}</td><td>{self.render(v)}</td></tr>" for k, v in data.items())
            return f"<table>{rows}</table>"
        else:
            return f"<span>{data!r}</span>"

# After: singledispatchmethod (open to extension)
from functools import singledispatchmethod

class RendererV2:
    @singledispatchmethod
    def render(self, data) -> str:
        return f"<span>{data!r}</span>"

    @render.register(str)
    def _(self, data: str) -> str:
        return f"<p>{data}</p>"

    @render.register(int)
    def _(self, data: int) -> str:
        return f"<code>{data}</code>"

    @render.register(list)
    def _(self, data: list) -> str:
        items = "".join(f"<li>{self.render(item)}</li>" for item in data)
        return f"<ul>{items}</ul>"

    @render.register(dict)
    def _(self, data: dict) -> str:
        rows = "".join(f"<tr><td>{k}</td><td>{self.render(v)}</td></tr>" for k, v in data.items())
        return f"<table>{rows}</table>"


r = RendererV2()
print(r.render("Hello"))
print(r.render(42))
print(r.render(["a", "b", "c"]))
```

---

**Scenario 3: You're using a third-party library that calls `process(value)` where value can be different types. You want type-specific behaviour. Implement using typing.overload for type-safety.**

**Q:** Show the `@typing.overload` implementation with a dispatch body.

**A:**

```python
from typing import overload


@overload
def process(value: int) -> str: ...
@overload
def process(value: float) -> str: ...
@overload
def process(value: str) -> int: ...
@overload
def process(value: list[int]) -> int: ...


def process(value):
    """Runtime dispatch — overload decorators inform type checkers only."""
    match value:
        case int():   return f"integer:{value}"
        case float(): return f"float:{value:.3f}"
        case str():   return int(value)
        case list():  return sum(value)
        case _:       raise TypeError(f"Unsupported: {type(value)}")


# Type checkers see:
# process(42)     → str
# process(3.14)   → str
# process("99")   → int
# process([1,2])  → int

print(process(42))         # "integer:42"
print(process(3.14159))    # "float:3.142"
print(process("123"))      # 123
print(process([10, 20]))   # 30
```

---

**Scenario 4: You need to implement `__add__` for a `Matrix` class that supports `Matrix + Matrix`, `Matrix + int` (scalar addition), and `Matrix + list` (element-wise). Use overload for type hints.**

**Q:** Show the implementation.

**A:**

```python
from __future__ import annotations
from typing import overload


class Matrix:
    def __init__(self, data: list[list[float]]) -> None:
        self._data = data
        self.rows  = len(data)
        self.cols  = len(data[0]) if data else 0

    @overload
    def __add__(self, other: "Matrix") -> "Matrix": ...
    @overload
    def __add__(self, other: int | float) -> "Matrix": ...
    @overload
    def __add__(self, other: list) -> "Matrix": ...

    def __add__(self, other):
        if isinstance(other, Matrix):
            if self.rows != other.rows or self.cols != other.cols:
                raise ValueError("Shape mismatch")
            return Matrix([
                [self._data[r][c] + other._data[r][c] for c in range(self.cols)]
                for r in range(self.rows)
            ])
        elif isinstance(other, (int, float)):
            return Matrix([
                [self._data[r][c] + other for c in range(self.cols)]
                for r in range(self.rows)
            ])
        elif isinstance(other, list):
            flat = [x for row in other for x in row]
            k = 0
            result = []
            for row in self._data:
                result.append([v + flat[k + i] for i, v in enumerate(row)])
                k += len(row)
            return Matrix(result)
        return NotImplemented

    def __repr__(self) -> str:
        return f"Matrix({self._data!r})"


m1 = Matrix([[1, 2], [3, 4]])
m2 = Matrix([[5, 6], [7, 8]])
print(m1 + m2)      # Matrix([[6, 8], [10, 12]])
print(m1 + 10)      # Matrix([[11, 12], [13, 14]])
print(m1 + [[1, 1], [1, 1]])  # Matrix([[2, 3], [4, 5]])
```

---

**Scenario 5: A logging system needs to accept different argument types in `log(message)` — string, exception, or dict. Implement with singledispatch.**

**Q:** Design the multi-type logger.

**A:**

```python
from functools import singledispatch
import traceback
import json


@singledispatch
def log(message: object) -> None:
    print(f"[LOG] {message!r}")


@log.register(str)
def _(message: str) -> None:
    print(f"[LOG] {message}")


@log.register(Exception)
def _(message: Exception) -> None:
    tb = "".join(traceback.format_exception(type(message), message, message.__traceback__))
    print(f"[ERROR] {type(message).__name__}: {message}\n{tb}")


@log.register(dict)
def _(message: dict) -> None:
    print(f"[LOG-JSON] {json.dumps(message, default=str)}")


log("Application started")
log({"user": "alice", "action": "login"})

try:
    1 / 0
except ZeroDivisionError as e:
    log(e)
```

---

**Scenario 6: You need to write a `convert(value, target_type)` function that behaves like overloading on the target type. How?**

**Q:** Show the implementation.

**A:**

```python
from functools import singledispatch


@singledispatch
def _convert_to(target_type: type, value: object):
    raise TypeError(f"Cannot convert {type(value)} to {target_type}")


def convert(value: object, target_type: type):
    """Convert value to target_type using registered converters."""
    return _convert_to(target_type, value)


@_convert_to.register(int)
def _(target_type: type, value: object) -> int:
    return int(value)


@_convert_to.register(float)
def _(target_type: type, value: object) -> float:
    return float(value)


@_convert_to.register(str)
def _(target_type: type, value: object) -> str:
    return str(value)


@_convert_to.register(bool)
def _(target_type: type, value: object) -> bool:
    return bool(value)


print(convert("42", int))     # 42
print(convert(3.14, str))     # "3.14"
print(convert(0, bool))       # False
```

---

**Scenario 7: You need to implement a `save` function in a file manager that behaves differently based on the type of data being saved (text, binary, JSON). Use singledispatch.**

**Q:** Implement the save function.

**A:**

```python
from functools import singledispatch
from pathlib import Path
import json


@singledispatch
def save(data: object, path: str | Path) -> None:
    raise TypeError(f"Cannot save {type(data).__name__}")


@save.register(str)
def _(data: str, path: str | Path) -> None:
    Path(path).write_text(data, encoding="utf-8")
    print(f"Saved text to {path}")


@save.register(bytes)
def _(data: bytes, path: str | Path) -> None:
    Path(path).write_bytes(data)
    print(f"Saved binary to {path}")


@save.register(dict)
@save.register(list)
def _(data: dict | list, path: str | Path) -> None:
    Path(path).write_text(json.dumps(data, indent=2), encoding="utf-8")
    print(f"Saved JSON to {path}")


# save("Hello World", "output.txt")
# save(b"\x89PNG", "image.png")
# save({"key": "value"}, "config.json")
```

---

**Scenario 8: How do you use `typing.overload` to correctly type a function that returns different types based on a boolean flag?**

**Q:** Show the pattern.

**A:** This is a classic use case for `@overload` — a flag that selects the return type:

```python
from typing import overload, Literal


@overload
def parse_number(value: str, as_float: Literal[True]) -> float: ...
@overload
def parse_number(value: str, as_float: Literal[False] = ...) -> int: ...


def parse_number(value: str, as_float: bool = False) -> int | float:
    if as_float:
        return float(value)
    return int(value)


n: int   = parse_number("42")          # mypy: int
f: float = parse_number("3.14", True)  # mypy: float
print(n, f)   # 42 3.14
```

Without `@overload`, mypy would infer the return type as `int | float` for all calls. With `@overload` and `Literal`, mypy knows the exact return type based on the flag value.

---

**Scenario 9: A data pipeline stage needs to process either a single item or a list of items. Implement with singledispatch and handle the Sequence abstraction.**

**Q:** Show the implementation.

**A:**

```python
from functools import singledispatch
from collections.abc import Sequence


@singledispatch
def process(data: object) -> list:
    """Default: treat as single item."""
    return [_process_one(data)]


@process.register(list)
@process.register(tuple)
def _(data: Sequence) -> list:
    """Handle any sequence — process each item."""
    return [_process_one(item) for item in data]


def _process_one(item) -> dict:
    return {"processed": item, "type": type(item).__name__}


print(process(42))               # [{'processed': 42, 'type': 'int'}]
print(process([1, 2, 3]))        # list of 3 processed items
print(process((10, 20)))         # tuple handled same as list
```

---

**Scenario 10: You're reviewing code that uses `*args` with isinstance checks for every branch. Suggest a refactoring to singledispatch.**

**Q:** Show before and after.

**A:**

```python
# Before: fragile, hard to extend
class EventHandler:
    def handle(self, *args):
        if not args:
            raise ValueError("No arguments")
        first = args[0]
        if isinstance(first, str):
            print(f"String event: {first}")
        elif isinstance(first, int):
            print(f"Integer event: {first * 2}")
        elif isinstance(first, dict):
            print(f"Dict event: {list(first.keys())}")
        else:
            raise TypeError(f"Unknown event type: {type(first)}")


# After: clean, extensible
from functools import singledispatchmethod


class EventHandlerV2:
    @singledispatchmethod
    def handle(self, event: object) -> None:
        raise TypeError(f"Unknown event type: {type(event)}")

    @handle.register(str)
    def _(self, event: str) -> None:
        print(f"String event: {event}")

    @handle.register(int)
    def _(self, event: int) -> None:
        print(f"Integer event: {event * 2}")

    @handle.register(dict)
    def _(self, event: dict) -> None:
        print(f"Dict event: {list(event.keys())}")


h = EventHandlerV2()
h.handle("click")
h.handle(42)
h.handle({"user": "alice", "action": "login"})
# Adding new event type: just register, no existing code touched
```

---

## 4. Quick Recap / Cheatsheet

- **Python does NOT have native method overloading** — the last definition wins.
- **Default parameters**: simplest workaround for optional arguments.
- **`*args` / `**kwargs`**: handle variable arity and keyword arguments.
- **`isinstance` dispatch**: works but violates Open/Closed Principle; use for 2–3 types only.
- **`functools.singledispatch`**: type-based dispatch on standalone functions; dispatches on first argument type.
- **`functools.singledispatchmethod`** (Python 3.8+): same, but for instance methods and classmethods; dispatches on second argument (`self` is skipped).
- **`typing.overload`**: pure type-checker hint — **no runtime effect**; the `@overload` bodies are discarded.
- **`@overload` ordering**: decorated overloads must precede the actual implementation.
- **singledispatch MRO**: if `bool` is not registered, the `int` handler is used (walks MRO).
- **singledispatch registry**: `func.registry` exposes the dispatch table; `func.dispatch(type)` returns the handler.
- **Union types in singledispatch** (Python 3.11+): `@func.register(int | str)` registers one handler for multiple types.
- **Multiple dispatch**: not stdlib — use `multipledispatch` library or manual isinstance matrix.
- **When to use named methods instead**: different operations with genuinely different semantics benefit from distinct names (`send_text` vs `send_bytes` vs `send_json`).
- **`match/case` (Python 3.10+)**: structural pattern matching can replace complex isinstance chains.
- **Performance**: singledispatch is O(1) for registered types, slightly slower than `isinstance` for very few types; scales better.
- **C++ vs Python**: C++ overloading is compile-time (zero runtime cost); Python singledispatch is runtime (small lookup cost).
