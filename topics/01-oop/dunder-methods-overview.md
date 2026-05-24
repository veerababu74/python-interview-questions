# Dunder Methods Overview

## 1. Concept (Deep Explanation)

### What Dunder Methods Are and Why Python Uses Double Underscores

**Dunder methods** (double-underscore methods, also called **magic methods** or **special methods**) are methods with names surrounded by double underscores: `__init__`, `__repr__`, `__add__`, etc. They are the mechanism through which Python's built-in operators, functions, and syntactic constructs are dispatched to user-defined classes.

When you write `a + b`, Python internally calls `type(a).__add__(a, b)`. When you write `len(obj)`, Python calls `type(obj).__len__(obj)`. When you iterate with `for x in obj`, Python calls `type(obj).__iter__(obj)`. This uniform dispatch mechanism is what makes Python's data model so consistent and powerful — your custom classes can participate in all of Python's syntax.

The double underscores serve a **name-mangling adjacent** purpose: they clearly mark these as Python internals, not application code. They also prevent accidental collision with user-defined method names (you're unlikely to name a method `__add__` unless you mean it).

**Critical insight**: Python always looks up dunder methods on the **type** (class), not the instance. This means:

```python
class Foo:
    def __len__(self) -> int:
        return 42

foo = Foo()
foo.__len__ = lambda: 99   # monkey-patch on INSTANCE
print(len(foo))   # 42 — Python bypasses the instance, looks at type(foo).__len__
print(foo.__len__())  # 99 — direct call goes to instance
```

This is a subtle but important difference — dunder lookups are **implicit type lookups**, not normal attribute lookups.

---

### Category 1: Object Lifecycle — `__new__`, `__init__`, `__del__`

```python
class Managed:
    _count: int = 0

    def __new__(cls, *args, **kwargs):
        """Called before __init__. Returns the new instance.
        Rarely needed — use when subclassing immutable types (int, str, tuple)
        or implementing singletons/object pools."""
        print(f"__new__ called for {cls.__name__}")
        instance = super().__new__(cls)
        cls._count += 1
        return instance

    def __init__(self, value: int) -> None:
        """Called after __new__. Initialises the instance.
        Most common entry point."""
        print(f"__init__ called with value={value}")
        self.value = value

    def __del__(self) -> None:
        """Called when the reference count drops to zero.
        NOT a guaranteed destructor — CPython uses ref counting, but
        PyPy/Jython may call this much later or not at all.
        Never rely on __del__ for critical resource cleanup — use context managers."""
        type(self)._count -= 1
        print(f"__del__ called, remaining: {type(self)._count}")


obj = Managed(42)
# __new__ called for Managed
# __init__ called with value=42
del obj
# __del__ called, remaining: 0
```

**`__new__` for immutable type subclassing:**

```python
class PositiveInt(int):
    def __new__(cls, value: int) -> "PositiveInt":
        if value <= 0:
            raise ValueError(f"Value must be positive, got {value}")
        return super().__new__(cls, value)

    # No __init__ needed — int is immutable, value set in __new__

n = PositiveInt(5)
print(n + 3)       # 8 — arithmetic still works (int methods inherited)
PositiveInt(-1)    # ValueError
```

---

### Category 2: String Representation — `__repr__`, `__str__`, `__format__`, `__bytes__`

```python
class Point:
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

    def __repr__(self) -> str:
        """For developers. Should be unambiguous.
        Convention: return a string that could reconstruct the object."""
        return f"Point(x={self.x!r}, y={self.y!r})"

    def __str__(self) -> str:
        """For end users. Should be readable.
        If not defined, falls back to __repr__."""
        return f"({self.x}, {self.y})"

    def __format__(self, spec: str) -> str:
        """Called by format() and f-strings with format spec."""
        if spec == "polar":
            import math
            r = math.sqrt(self.x**2 + self.y**2)
            theta = math.degrees(math.atan2(self.y, self.x))
            return f"r={r:.2f}, θ={theta:.1f}°"
        return str(self)

    def __bytes__(self) -> bytes:
        """Called by bytes(obj). Encode as binary."""
        import struct
        return struct.pack("dd", self.x, self.y)


p = Point(3.0, 4.0)
print(repr(p))         # Point(x=3.0, y=4.0)
print(str(p))          # (3.0, 4.0)
print(f"{p:polar}")    # r=5.00, θ=53.1°
print(bytes(p))        # binary representation
```

**Key rule: `__repr__` should always be defined. `__str__` is optional.**

---

### Category 3: Comparison — `__eq__`, `__ne__`, `__lt__`, `__le__`, `__gt__`, `__ge__`, `__hash__`

```python
from functools import total_ordering

@total_ordering   # Generates missing comparisons from __eq__ and ONE of lt/le/gt/ge
class Version:
    def __init__(self, major: int, minor: int, patch: int) -> None:
        self.major = major
        self.minor = minor
        self.patch = patch

    def _tuple(self) -> tuple[int, int, int]:
        return (self.major, self.minor, self.patch)

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Version):
            return NotImplemented   # Not NotImplementedError!
        return self._tuple() == other._tuple()

    def __lt__(self, other: object) -> bool:
        if not isinstance(other, Version):
            return NotImplemented
        return self._tuple() < other._tuple()

    def __hash__(self) -> int:
        """Required when __eq__ is defined! Otherwise, hash(obj) raises TypeError."""
        return hash(self._tuple())

    def __repr__(self) -> str:
        return f"Version({self.major}.{self.minor}.{self.patch})"


v1 = Version(1, 2, 3)
v2 = Version(1, 3, 0)
print(v1 < v2)   # True
print(v1 <= v2)  # True (generated by @total_ordering)
print(sorted([v2, v1]))   # [Version(1.2.3), Version(1.3.0)]
print({v1, v2})  # works because __hash__ is defined
```

**The `__hash__` and `__eq__` contract:**
- If `a == b`, then `hash(a) == hash(b)` MUST be true
- If you define `__eq__`, Python automatically sets `__hash__ = None` (making instances unhashable)
- To restore hashability, explicitly define `__hash__`
- If you want a frozen/immutable type, define both; for mutable types, usually omit `__hash__`

---

### Category 4: Arithmetic — `__add__`, `__radd__`, `__iadd__`, `__mul__`, etc.

```python
class Vector:
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

    def __add__(self, other: "Vector | float") -> "Vector":
        if isinstance(other, Vector):
            return Vector(self.x + other.x, self.y + other.y)
        if isinstance(other, (int, float)):
            return Vector(self.x + other, self.y + other)
        return NotImplemented

    def __radd__(self, other: float) -> "Vector":
        """Called when other.__add__(self) returns NotImplemented.
        e.g., 5 + vec — int doesn't know about Vector, so Python tries vec.__radd__(5)"""
        return self.__add__(other)

    def __iadd__(self, other: "Vector") -> "Vector":
        """In-place addition: vec += other
        Return self (modified) to avoid unnecessary object creation."""
        if isinstance(other, Vector):
            self.x += other.x
            self.y += other.y
            return self
        return NotImplemented

    def __mul__(self, scalar: float) -> "Vector":
        return Vector(self.x * scalar, self.y * scalar)

    def __rmul__(self, scalar: float) -> "Vector":
        """3 * vec — scalar.__mul__(vec) fails, so vec.__rmul__(3) is tried."""
        return self.__mul__(scalar)

    def __neg__(self) -> "Vector":
        return Vector(-self.x, -self.y)

    def __abs__(self) -> float:
        return (self.x**2 + self.y**2) ** 0.5

    def __repr__(self) -> str:
        return f"Vector({self.x}, {self.y})"


v1 = Vector(1, 2)
v2 = Vector(3, 4)
print(v1 + v2)    # Vector(4, 6)
print(5 + v1)     # Vector(6, 7) — via __radd__
print(3 * v1)     # Vector(3, 6) — via __rmul__
print(abs(v2))    # 5.0
v1 += v2
print(v1)         # Vector(4, 6) — in-place
```

---

### Category 5: Type Conversion — `__int__`, `__float__`, `__bool__`, `__complex__`

```python
class Fraction:
    def __init__(self, numerator: int, denominator: int) -> None:
        if denominator == 0:
            raise ZeroDivisionError("Denominator cannot be zero")
        from math import gcd
        g = gcd(abs(numerator), abs(denominator))
        self.num = numerator // g
        self.den = denominator // g

    def __float__(self) -> float:
        return self.num / self.den

    def __int__(self) -> int:
        return int(float(self))

    def __bool__(self) -> bool:
        """Called by bool(), if-statements, and, or, not.
        Default (if not defined): True unless __len__ returns 0."""
        return self.num != 0

    def __complex__(self) -> complex:
        return complex(float(self), 0)

    def __repr__(self) -> str:
        return f"Fraction({self.num}/{self.den})"


f = Fraction(3, 4)
print(float(f))    # 0.75
print(int(f))      # 0
print(bool(f))     # True
print(bool(Fraction(0, 1)))  # False
if f:
    print("Non-zero fraction")
```

---

### Category 6: Container Protocol

```python
from collections.abc import MutableMapping
from typing import Iterator

class FixedDict(MutableMapping):
    """A dict that only allows pre-declared keys."""

    def __init__(self, allowed_keys: set[str], **kwargs) -> None:
        self._allowed = allowed_keys
        self._data: dict = {}
        for k, v in kwargs.items():
            self[k] = v   # validates through __setitem__

    def __getitem__(self, key: str):
        return self._data[key]

    def __setitem__(self, key: str, value) -> None:
        if key not in self._allowed:
            raise KeyError(f"Key {key!r} not allowed. Allowed: {self._allowed}")
        self._data[key] = value

    def __delitem__(self, key: str) -> None:
        del self._data[key]

    def __iter__(self) -> Iterator[str]:
        return iter(self._data)

    def __len__(self) -> int:
        return len(self._data)

    def __contains__(self, key: object) -> bool:
        """Optional override — default uses __iter__, but this is O(1)."""
        return key in self._data

    def __repr__(self) -> str:
        return f"FixedDict({self._data}, allowed={self._allowed})"


fd = FixedDict({"name", "age"}, name="Alice")
fd["age"] = 30
print(fd["name"])   # "Alice"
print(len(fd))      # 2
fd["email"] = "x"   # KeyError: Key 'email' not allowed
```

---

### Category 7: Context Manager — `__enter__`, `__exit__`

```python
import time

class Timer:
    def __enter__(self) -> "Timer":
        """Called at the start of the `with` block. Returns the context variable."""
        self._start = time.perf_counter()
        return self   # this is what `as timer` captures

    def __exit__(
        self,
        exc_type: type | None,
        exc_val: BaseException | None,
        exc_tb,
    ) -> bool:
        """Called at the end of the `with` block.
        Return True to suppress exceptions; False/None to propagate."""
        self.elapsed = time.perf_counter() - self._start
        print(f"Elapsed: {self.elapsed:.4f}s")
        if exc_type is ValueError:
            print("Suppressing ValueError!")
            return True   # suppress the exception
        return False  # propagate other exceptions

with Timer() as t:
    time.sleep(0.01)
print(t.elapsed)   # 0.01xx

with Timer():
    raise ValueError("oops")   # suppressed!
# Program continues here
```

---

### Category 8: Callable — `__call__`

```python
class Multiplier:
    """Makes instances callable — acts like a function with state."""
    def __init__(self, factor: float) -> None:
        self.factor = factor

    def __call__(self, value: float) -> float:
        return value * self.factor

double = Multiplier(2)
triple = Multiplier(3)
print(double(5))   # 10.0
print(triple(5))   # 15.0
print(callable(double))  # True

# Use case: stateful callbacks, memoization, partial application
class Memoize:
    def __init__(self, func) -> None:
        self.func = func
        self.cache: dict = {}

    def __call__(self, *args):
        if args not in self.cache:
            self.cache[args] = self.func(*args)
        return self.cache[args]

@Memoize
def fib(n: int) -> int:
    return n if n < 2 else fib(n-1) + fib(n-2)

print(fib(30))  # 832040 — fast due to memoization
```

---

### Category 9: Attribute Access — `__getattr__`, `__getattribute__`, `__setattr__`, `__delattr__`

```python
class AttrProxy:
    """Demonstrates the difference between __getattr__ and __getattribute__."""

    def __init__(self, target) -> None:
        # MUST use object.__setattr__ to avoid infinite recursion
        # (our __setattr__ would call itself)
        object.__setattr__(self, '_target', target)

    def __getattr__(self, name: str):
        """Called ONLY when normal attribute lookup fails.
        (instance __dict__ → class __dict__ → MRO → then __getattr__)
        Perfect for delegation/proxy patterns."""
        return getattr(self._target, name)

    def __setattr__(self, name: str, value) -> None:
        """Called for ALL attribute assignments — including in __init__!
        Must use object.__setattr__ for own attributes."""
        if name.startswith('_'):
            object.__setattr__(self, name, value)
        else:
            setattr(self._target, name, value)

    def __getattribute__(self, name: str):
        """Called for EVERY attribute access — even internal ones.
        Override with extreme caution; always call super().__getattribute__."""
        print(f"Accessing: {name}")
        return super().__getattribute__(name)


class Target:
    def __init__(self) -> None:
        self.data = "hello"

t = Target()
proxy = AttrProxy(t)
print(proxy.data)   # prints "Accessing: ..." then "hello"
```

---

### Category 10: Descriptor Protocol — `__get__`, `__set__`, `__delete__`, `__set_name__`

```python
class TypeEnforced:
    """Custom descriptor that enforces type on every assignment."""

    def __init__(self, expected_type: type) -> None:
        self.expected_type = expected_type
        self.attr_name: str = ""

    def __set_name__(self, owner: type, name: str) -> None:
        self.attr_name = name

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self  # class-level access → return descriptor
        return obj.__dict__.get(self.attr_name)

    def __set__(self, obj, value) -> None:
        if not isinstance(value, self.expected_type):
            raise TypeError(
                f"Attribute {self.attr_name!r} must be "
                f"{self.expected_type.__name__}, got {type(value).__name__}"
            )
        obj.__dict__[self.attr_name] = value

    def __delete__(self, obj) -> None:
        obj.__dict__.pop(self.attr_name, None)

class Record:
    name: str = TypeEnforced(str)
    score: int = TypeEnforced(int)

    def __init__(self, name: str, score: int) -> None:
        self.name = name
        self.score = score

r = Record("Alice", 95)
r.score = "not an int"   # TypeError
```

---

### `NotImplemented` vs `NotImplementedError`

This is a very common interview question:

- **`NotImplemented`** (a singleton value, not an exception) is returned from binary operator dunders (`__add__`, `__eq__`, etc.) to signal "I don't know how to handle this type — try the reflected method on the other operand". Python uses this for operator dispatch.
- **`NotImplementedError`** (an exception) is raised when an abstract method is not implemented. Used in ABCs or base classes to force subclasses to override.

```python
class Money:
    def __add__(self, other):
        if not isinstance(other, Money):
            return NotImplemented   # Correct — let Python try other.__radd__
        return Money(self.amount + other.amount)

class AbstractBase:
    def process(self) -> None:
        raise NotImplementedError("Subclasses must implement process()")
        # NOT return NotImplemented — that's wrong here
```

---

### `__repr__` vs `__str__` Contract

| Method | Audience | Goal | Fallback |
|---|---|---|---|
| `__repr__` | Developers / debuggers | Unambiguous, ideally reconstructible | `object.__repr__` (ugly) |
| `__str__` | End users | Readable | Falls back to `__repr__` |

```python
import datetime
d = datetime.date(2024, 1, 15)
print(repr(d))   # datetime.date(2024, 1, 15) — could reconstruct!
print(str(d))    # 2024-01-15 — human readable
```

In containers (lists, dicts), Python calls `__repr__` on elements:
```python
items = [d]
print(items)   # [datetime.date(2024, 1, 15)] — repr, not str
```

---

## 2. Conceptual Interview Questions (10)

**Q1: Why does Python look up dunder methods on the type, not the instance?**

**A:** This design decision ensures that the interpreter's implicit operations are consistent and cannot be broken by instance-level monkey-patching. If `len(obj)` checked `obj.__len__` (instance dict) first, a user could accidentally (or maliciously) shadow the dunder by storing a non-callable `__len__` value. By always doing `type(obj).__len__(obj)`, Python guarantees that dunder dispatch is governed by the class, not variable instance state.

```python
class Foo:
    def __len__(self) -> int:
        return 10

f = Foo()
f.__len__ = lambda: 999   # instance attribute

print(f.__len__())   # 999 — direct call uses instance
print(len(f))        # 10 — implicit call bypasses instance, uses type(f).__len__
```

This also allows CPython to optimise dunder lookups by caching them on the type object, skipping the full descriptor protocol for each call.

**Q2: What is the difference between `__getattr__` and `__getattribute__`?**

**A:**

- `__getattribute__` is called for **every** attribute access, unconditionally. If you override it, you must call `super().__getattribute__()` or `object.__getattribute__()` for the normal attributes to work; otherwise you'll get infinite recursion.
- `__getattr__` is called only **after** the normal attribute lookup has failed (not found in `obj.__dict__`, class `__dict__`, or MRO). It's a "fallback" hook.

`__getattr__` is safe and commonly used for proxies/delegation. `__getattribute__` is dangerous and should only be overridden for very specific cross-cutting concerns (like attribute access logging in a debug build).

```python
class Safe:
    def __getattr__(self, name: str):
        return f"default for {name}"  # only called when name not found

class Dangerous:
    def __getattribute__(self, name: str):
        print(f"Accessing {name}")
        return super().__getattribute__(name)  # MUST call super!
```

**Q3: Explain `NotImplemented` — when do you return it vs raise `NotImplementedError`?**

**A:** Return `NotImplemented` (the singleton, not an exception) from binary operator dunders (`__add__`, `__eq__`, `__lt__`, etc.) when the method cannot handle the given operand type. Python's operator dispatch mechanism catches this value and tries the reflected operation on the other operand:

```python
class Money:
    def __add__(self, other):
        if not isinstance(other, Money):
            return NotImplemented   # let Python try other.__radd__(self)
        return Money(self.amount + other.amount)
```

Raise `NotImplementedError` when an abstract method is not overridden — this is an exceptional condition, not a dispatch signal:

```python
class Base:
    def compute(self) -> int:
        raise NotImplementedError("Subclasses must implement compute()")
```

Returning `NotImplemented` from `compute()` would be a bug — Python wouldn't raise any error, and the `NotImplemented` singleton would propagate silently.

**Q4: When you define `__eq__`, what happens to `__hash__` and why?**

**A:** When you define `__eq__` in a class without defining `__hash__`, Python sets `__hash__` to `None`. This makes instances unhashable (you can't put them in sets or use them as dict keys). The rationale: if two objects are equal (`a == b`), they must have the same hash (`hash(a) == hash(b)`). Mutable objects that define custom equality often can't guarantee this (if you mutate the object after insertion into a dict, the hash might change), so Python conservatively makes them unhashable.

To restore hashability, define `__hash__` explicitly. For immutable objects with custom equality, derive the hash from the same fields used in `__eq__`.

```python
class Point:
    def __init__(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

    def __eq__(self, other) -> bool:
        return isinstance(other, Point) and self.x == other.x and self.y == other.y

    def __hash__(self) -> int:
        return hash((self.x, self.y))  # must be consistent with __eq__

p = Point(1, 2)
d = {p: "value"}   # works because __hash__ is defined
```

**Q5: What is the purpose of `__new__` and when would you override it?**

**A:** `__new__` is the static method called to create the object (allocate memory). It runs before `__init__`. It receives the class as its first argument and must return an instance (typically of that class).

You need `__new__` for:
1. **Subclassing immutable types** (`int`, `str`, `tuple`, `frozenset`) — the value is set in `__new__`, not `__init__`
2. **Singletons / object pools** — control instance creation
3. **Metaclass-like behaviour** without a metaclass

```python
class Cached:
    _cache: dict = {}

    def __new__(cls, value: int):
        if value in cls._cache:
            print(f"Cache hit for {value}")
            return cls._cache[value]
        instance = super().__new__(cls)
        cls._cache[value] = instance
        return instance

    def __init__(self, value: int) -> None:
        self.value = value   # Called even on cache hit!

a = Cached(1)
b = Cached(1)
print(a is b)   # True — same object
```

Note: `__init__` is called even on cached instances (since Python calls it unconditionally after `__new__`). Use `__new__` to return the cached instance and guard `__init__` with `hasattr` if needed.

**Q6: How does `@functools.total_ordering` work and when should you use it?**

**A:** `@total_ordering` is a class decorator that generates the missing comparison methods from `__eq__` and one of `__lt__`, `__le__`, `__gt__`, `__ge__`. It introspects the class and adds the missing comparisons using simple logical transformations.

```python
from functools import total_ordering

@total_ordering
class Weight:
    def __init__(self, kg: float) -> None:
        self.kg = kg

    def __eq__(self, other) -> bool:
        if not isinstance(other, Weight):
            return NotImplemented
        return self.kg == other.kg

    def __lt__(self, other) -> bool:
        if not isinstance(other, Weight):
            return NotImplemented
        return self.kg < other.kg

    # total_ordering generates: __le__, __gt__, __ge__

w1, w2 = Weight(70), Weight(80)
print(w1 <= w2)   # True — generated
print(w1 > w2)    # False — generated
```

**When to use**: when you want all comparison operators without writing all 6. Trade-off: slightly slower than hand-coded comparisons (the generated methods call `__lt__` and `__eq__` indirectly).

**Q7: Explain the iteration protocol: `__iter__` and `__next__`.**

**A:** Python's iteration protocol has two parts:
- `__iter__`: returns the iterator object (often `self` for iterators; for iterables, a new iterator each time)
- `__next__`: returns the next item; raises `StopIteration` when exhausted

```python
class CountDown:
    """This is both an iterable AND an iterator."""
    def __init__(self, start: int) -> None:
        self._current = start

    def __iter__(self) -> "CountDown":
        return self   # iterators return self from __iter__

    def __next__(self) -> int:
        if self._current <= 0:
            raise StopIteration
        value = self._current
        self._current -= 1
        return value

for n in CountDown(3):
    print(n)   # 3, 2, 1
```

Distinguish:
- **Iterable**: has `__iter__` returning an iterator (e.g., `list`, `str`)
- **Iterator**: has both `__iter__` and `__next__` (e.g., `list_iterator`, generators)

An iterator's `__iter__` returns `self`, so iterators are also iterables.

**Q8: What does `__call__` enable and how is it used in practice?**

**A:** `__call__` makes an instance callable like a function (`obj(args)`). It's used for:
1. **Stateful callables** / function objects with state
2. **Decorators implemented as classes** (can hold configuration state)
3. **Memoization wrappers**
4. **Functors** and partial application

```python
class RateLimiter:
    """Callable that enforces rate limiting."""
    import time

    def __init__(self, calls_per_second: float) -> None:
        self._min_interval = 1.0 / calls_per_second
        self._last_called = 0.0

    def __call__(self, func):
        """Used as a decorator factory."""
        import functools
        import time

        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            now = time.monotonic()
            wait = self._min_interval - (now - self._last_called)
            if wait > 0:
                time.sleep(wait)
            self._last_called = time.monotonic()
            return func(*args, **kwargs)
        return wrapper

@RateLimiter(calls_per_second=2)
def api_call() -> str:
    return "response"
```

**Q9: How does `__enter__` and `__exit__` work, and what does returning `True` from `__exit__` do?**

**A:** The context manager protocol:
- `__enter__` is called at the start of a `with` block. Its return value is bound to `as` variable.
- `__exit__` receives three arguments: `exc_type`, `exc_val`, `exc_tb`. If no exception, all three are `None`. If `__exit__` returns truthy, the exception is suppressed; otherwise, it propagates.

```python
class SuppressErrors:
    def __init__(self, *exception_types: type) -> None:
        self._types = exception_types

    def __enter__(self) -> "SuppressErrors":
        return self

    def __exit__(self, exc_type, exc_val, exc_tb) -> bool:
        if exc_type is not None and issubclass(exc_type, self._types):
            print(f"Suppressed: {exc_val}")
            return True   # suppress
        return False   # re-raise


with SuppressErrors(ValueError, KeyError):
    raise ValueError("This is suppressed!")

print("Execution continues")
```

**Q10: What is the difference between `__iter__` and `__getitem__` for iteration?**

**A:** Python has two ways to make an object iterable:
1. Define `__iter__` (preferred) — returns an iterator
2. Define `__getitem__` with integer indices starting from 0 — Python will call `obj[0]`, `obj[1]`, etc., until `IndexError`

This is a legacy protocol from Python 2. Modern code should use `__iter__`:

```python
class LegacyIterable:
    """Works with for-loops via __getitem__ (legacy protocol)."""
    _data = [10, 20, 30]

    def __getitem__(self, index: int) -> int:
        return self._data[index]  # raises IndexError when out of range

for item in LegacyIterable():
    print(item)   # 10, 20, 30 — works despite no __iter__!

# BUT: iter() only works with __iter__:
# iter(LegacyIterable())  # TypeError in Python 3 if no __iter__
```

---

## 3. Scenario-Based Interview Questions (10)

**S1: Implement a `Money` class that supports arithmetic, comparison, and cannot be added to non-Money types.**

**Q:** Implement with appropriate dunders.

**A:**

```python
from decimal import Decimal
from functools import total_ordering

@total_ordering
class Money:
    def __init__(self, amount: Decimal | str | int, currency: str = "USD") -> None:
        self.amount = Decimal(str(amount))
        self.currency = currency.upper()

    def _validate_currency(self, other: "Money") -> None:
        if self.currency != other.currency:
            raise ValueError(f"Cannot mix {self.currency} and {other.currency}")

    def __add__(self, other: "Money") -> "Money":
        if not isinstance(other, Money):
            return NotImplemented
        self._validate_currency(other)
        return Money(self.amount + other.amount, self.currency)

    def __sub__(self, other: "Money") -> "Money":
        if not isinstance(other, Money):
            return NotImplemented
        self._validate_currency(other)
        return Money(self.amount - other.amount, self.currency)

    def __mul__(self, factor: int | float | Decimal) -> "Money":
        return Money(self.amount * Decimal(str(factor)), self.currency)

    def __rmul__(self, factor: int | float | Decimal) -> "Money":
        return self.__mul__(factor)

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Money):
            return NotImplemented
        return self.amount == other.amount and self.currency == other.currency

    def __lt__(self, other: "Money") -> bool:
        if not isinstance(other, Money):
            return NotImplemented
        self._validate_currency(other)
        return self.amount < other.amount

    def __hash__(self) -> int:
        return hash((self.amount, self.currency))

    def __repr__(self) -> str:
        return f"Money({self.amount!r}, {self.currency!r})"

    def __str__(self) -> str:
        return f"{self.currency} {self.amount:.2f}"

    def __bool__(self) -> bool:
        return self.amount != 0


usd10 = Money("10.00")
usd5 = Money("5.00")
print(usd10 + usd5)   # USD 15.00
print(2 * usd5)        # USD 10.00
print(usd10 > usd5)    # True
print(sorted([usd10, usd5]))  # [Money('5.00', 'USD'), Money('10.00', 'USD')]
```

**S2: Build a `LazyList` that wraps a generator and evaluates it on demand, supporting indexing and `len`.**

**Q:** Implement container dunders.

**A:**

```python
class LazyList:
    """Evaluates a generator lazily; caches evaluated items."""

    def __init__(self, gen) -> None:
        self._gen = gen
        self._cache: list = []
        self._exhausted = False

    def _load_up_to(self, index: int) -> None:
        while not self._exhausted and len(self._cache) <= index:
            try:
                self._cache.append(next(self._gen))
            except StopIteration:
                self._exhausted = True

    def _load_all(self) -> None:
        if not self._exhausted:
            self._cache.extend(self._gen)
            self._exhausted = True

    def __getitem__(self, index: int):
        if isinstance(index, slice):
            self._load_all()
            return self._cache[index]
        if index < 0:
            self._load_all()
        else:
            self._load_up_to(index)
        return self._cache[index]

    def __len__(self) -> int:
        self._load_all()
        return len(self._cache)

    def __iter__(self):
        index = 0
        while True:
            try:
                yield self[index]
                index += 1
            except IndexError:
                break

    def __contains__(self, item) -> bool:
        for elem in self:
            if elem == item:
                return True
        return False

    def __repr__(self) -> str:
        return f"LazyList(cached={self._cache!r}, exhausted={self._exhausted})"


def squares():
    for i in range(10):
        print(f"  computing {i}²")
        yield i * i

ll = LazyList(squares())
print(ll[3])   # computes 0,1,2,3 lazily
print(ll[1])   # already cached, no computation
print(len(ll)) # forces full evaluation
```

**S3: Implement a `Matrix` class with operator overloading for addition and multiplication.**

**Q:** Implement with arithmetic dunders.

**A:**

```python
class Matrix:
    def __init__(self, data: list[list[float]]) -> None:
        self._data = [row[:] for row in data]  # deep copy
        self.rows = len(data)
        self.cols = len(data[0]) if data else 0

    def __getitem__(self, pos: tuple[int, int]) -> float:
        row, col = pos
        return self._data[row][col]

    def __setitem__(self, pos: tuple[int, int], value: float) -> None:
        row, col = pos
        self._data[row][col] = value

    def __add__(self, other: "Matrix") -> "Matrix":
        if self.rows != other.rows or self.cols != other.cols:
            raise ValueError("Matrix dimensions must match for addition")
        result = [[self._data[i][j] + other._data[i][j]
                   for j in range(self.cols)]
                  for i in range(self.rows)]
        return Matrix(result)

    def __mul__(self, other: "Matrix | float") -> "Matrix":
        if isinstance(other, (int, float)):
            return Matrix([[v * other for v in row] for row in self._data])
        if self.cols != other.rows:
            raise ValueError(f"Incompatible dimensions: {self.cols} vs {other.rows}")
        result = [[sum(self._data[i][k] * other._data[k][j]
                       for k in range(self.cols))
                   for j in range(other.cols)]
                  for i in range(self.rows)]
        return Matrix(result)

    def __rmul__(self, scalar: float) -> "Matrix":
        return self.__mul__(scalar)

    def __repr__(self) -> str:
        rows_str = "\n  ".join(str(row) for row in self._data)
        return f"Matrix([\n  {rows_str}\n])"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Matrix):
            return NotImplemented
        return self._data == other._data


A = Matrix([[1, 2], [3, 4]])
B = Matrix([[5, 6], [7, 8]])
print(A + B)
print(A * B)
print(2 * A)
```

**S4: Implement a context manager that retries a block of code on failure.**

**Q:** Use `__enter__` and `__exit__` for retry logic.

**A:**

```python
import time

class Retry:
    def __init__(self, max_attempts: int = 3, delay: float = 1.0,
                 exceptions: tuple[type, ...] = (Exception,)) -> None:
        self.max_attempts = max_attempts
        self.delay = delay
        self.exceptions = exceptions
        self._attempt = 0

    def __enter__(self) -> "Retry":
        self._attempt = 0
        return self

    def __exit__(self, exc_type, exc_val, exc_tb) -> bool:
        if exc_type is None:
            return False   # no exception — exit normally
        if issubclass(exc_type, self.exceptions):
            self._attempt += 1
            if self._attempt < self.max_attempts:
                print(f"Attempt {self._attempt} failed: {exc_val}. Retrying...")
                time.sleep(self.delay)
                return True   # suppress exception, loop will retry
        return False  # out of attempts or wrong exception type

# Usage (in practice, wrap in a loop):
import random

retry = Retry(max_attempts=3, delay=0.1, exceptions=(ValueError,))
for _ in range(retry.max_attempts):
    with retry:
        if random.random() < 0.7:
            raise ValueError("Transient error")
        print("Success!")
        break
```

**S5: Create a `ReadOnlyDict` that prevents modification after creation.**

**Q:** Implement using container dunders.

**A:**

```python
class ReadOnlyDict(dict):
    """A dict subclass that prevents modification."""
    _SENTINEL = object()

    def __setitem__(self, key, value) -> None:
        raise TypeError(f"{type(self).__name__} does not support item assignment")

    def __delitem__(self, key) -> None:
        raise TypeError(f"{type(self).__name__} does not support item deletion")

    def clear(self) -> None:
        raise TypeError(f"{type(self).__name__} does not support clear()")

    def pop(self, key, default=_SENTINEL):
        raise TypeError(f"{type(self).__name__} does not support pop()")

    def update(self, *args, **kwargs) -> None:
        raise TypeError(f"{type(self).__name__} does not support update()")

    @classmethod
    def from_dict(cls, data: dict) -> "ReadOnlyDict":
        """Factory — bypass __setitem__ by calling dict.__init__."""
        instance = cls.__new__(cls)
        dict.__init__(instance, data)
        return instance

    def __repr__(self) -> str:
        return f"ReadOnlyDict({dict.__repr__(self)})"


rod = ReadOnlyDict.from_dict({"key": "value", "count": 42})
print(rod["key"])    # "value"
print(len(rod))      # 2
rod["new"] = "x"     # TypeError
rod.clear()          # TypeError
```

**S6: Implement `__format__` so that a `Duration` class can be formatted in multiple ways.**

**Q:** Use `__format__` for flexible string formatting.

**A:**

```python
class Duration:
    def __init__(self, seconds: float) -> None:
        self._seconds = seconds

    @property
    def hours(self) -> int:
        return int(self._seconds // 3600)

    @property
    def minutes(self) -> int:
        return int((self._seconds % 3600) // 60)

    @property
    def secs(self) -> float:
        return self._seconds % 60

    def __repr__(self) -> str:
        return f"Duration({self._seconds!r})"

    def __str__(self) -> str:
        return f"{self.hours:02d}:{self.minutes:02d}:{self.secs:05.2f}"

    def __format__(self, spec: str) -> str:
        if spec == "hms":
            return str(self)
        elif spec == "short":
            if self.hours:
                return f"{self.hours}h {self.minutes}m"
            elif self.minutes:
                return f"{self.minutes}m {int(self.secs)}s"
            else:
                return f"{self._seconds:.1f}s"
        elif spec == "ms":
            return f"{self._seconds * 1000:.0f}ms"
        elif not spec:
            return str(self)
        raise ValueError(f"Unknown format spec: {spec!r}")


d = Duration(3723.5)
print(f"{d}")          # 01:02:03.50
print(f"{d:hms}")      # 01:02:03.50
print(f"{d:short}")    # 1h 2m
print(f"{d:ms}")       # 3723500ms
```

**S7: Show a practical use of `__getattr__` for a configuration proxy.**

**Q:** Implement attribute-access configuration.

**A:**

```python
class Config:
    """Reads config via attribute access with dot notation."""

    def __init__(self, data: dict) -> None:
        object.__setattr__(self, '_data', data)

    def __getattr__(self, name: str):
        try:
            value = self._data[name]
        except KeyError:
            raise AttributeError(f"Config has no key {name!r}")
        if isinstance(value, dict):
            return Config(value)   # recursive — supports nested.access
        return value

    def __setattr__(self, name: str, value) -> None:
        if name.startswith('_'):
            object.__setattr__(self, name, value)
        else:
            self._data[name] = value

    def __contains__(self, key: str) -> bool:
        return key in self._data

    def __repr__(self) -> str:
        return f"Config({self._data!r})"


cfg = Config({
    "database": {
        "host": "localhost",
        "port": 5432,
        "name": "mydb",
    },
    "debug": True,
})

print(cfg.debug)              # True
print(cfg.database.host)      # "localhost"
print(cfg.database.port)      # 5432
print("database" in cfg)      # True
cfg.new_key = "added"
print(cfg.new_key)            # "added"
```

**S8: Build a class whose instances record all method calls for debugging.**

**Q:** Use `__getattribute__` to intercept method calls.

**A:**

```python
class CallRecorder:
    """Wraps another object and records all method calls."""

    def __init__(self, target) -> None:
        object.__setattr__(self, '_target', target)
        object.__setattr__(self, '_calls', [])

    def __getattr__(self, name: str):
        attr = getattr(object.__getattribute__(self, '_target'), name)
        if callable(attr):
            def recording_wrapper(*args, **kwargs):
                call_log = {'method': name, 'args': args, 'kwargs': kwargs}
                try:
                    result = attr(*args, **kwargs)
                    call_log['result'] = result
                    return result
                except Exception as exc:
                    call_log['exception'] = exc
                    raise
                finally:
                    object.__getattribute__(self, '_calls').append(call_log)
            return recording_wrapper
        return attr

    @property
    def call_log(self) -> list[dict]:
        return object.__getattribute__(self, '_calls')


wrapped = CallRecorder([1, 2, 3])
wrapped.append(4)
wrapped.append(5)
print(wrapped.call_log)
# [{'method': 'append', 'args': (4,), 'kwargs': {}, 'result': None},
#  {'method': 'append', 'args': (5,), 'kwargs': {}, 'result': None}]
```

**S9: Implement a class that supports `with` for transaction management, rolling back on exception.**

**Q:** Implement transactional context manager.

**A:**

```python
class Transaction:
    def __init__(self) -> None:
        self._log: list[tuple] = []
        self._committed = False

    def insert(self, table: str, row: dict) -> None:
        self._log.append(("INSERT", table, row))
        print(f"Staged INSERT into {table}: {row}")

    def delete(self, table: str, key: str) -> None:
        self._log.append(("DELETE", table, key))
        print(f"Staged DELETE from {table} where key={key}")

    def __enter__(self) -> "Transaction":
        print("Transaction started")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb) -> bool:
        if exc_type is None:
            self._commit()
        else:
            self._rollback()
        return False  # never suppress exceptions

    def _commit(self) -> None:
        print(f"Committing {len(self._log)} operation(s)")
        for op in self._log:
            print(f"  Executing: {op}")
        self._committed = True
        self._log.clear()

    def _rollback(self) -> None:
        print(f"Rolling back {len(self._log)} staged operation(s)")
        self._log.clear()


with Transaction() as tx:
    tx.insert("users", {"name": "Alice"})
    tx.insert("users", {"name": "Bob"})
# Commits on clean exit

try:
    with Transaction() as tx:
        tx.insert("users", {"name": "Charlie"})
        raise RuntimeError("Something went wrong!")
except RuntimeError:
    pass   # Rolled back!
```

**S10: A class needs to be picklable with custom serialisation. Which dunders are involved?**

**Q:** Implement custom pickle support.

**A:**

```python
import pickle

class SecretHolder:
    """Stores a secret — encrypts it during pickling."""

    def __init__(self, secret: str, owner: str) -> None:
        self.owner = owner
        self._secret = secret

    def __getstate__(self) -> dict:
        """Called by pickle to get the state to serialise.
        Opportunity to encrypt, exclude sensitive fields, etc."""
        state = self.__dict__.copy()
        # Encrypt the secret (simplified XOR for demo)
        encoded = bytes(b ^ 0xFF for b in self._secret.encode())
        state['_secret'] = encoded
        return state

    def __setstate__(self, state: dict) -> None:
        """Called by pickle when deserialising.
        Reverse any transformations from __getstate__."""
        # Decrypt
        decoded = bytes(b ^ 0xFF for b in state['_secret']).decode()
        state['_secret'] = decoded
        self.__dict__.update(state)

    def __repr__(self) -> str:
        return f"SecretHolder(owner={self.owner!r}, secret=***)"


sh = SecretHolder("my_password_123", "Alice")
print(sh)

data = pickle.dumps(sh)
sh2 = pickle.loads(data)
print(sh2)
print(sh2._secret)   # "my_password_123" — correctly restored
```

---

## 4. Quick Recap / Cheatsheet

- **Dunder lookup is type-based**: Python calls `type(obj).__dunder__`, not `obj.__dunder__` — instance shadowing doesn't work for implicit calls
- **`__new__`** allocates + returns instance; **`__init__`** initialises it — override `__new__` only for immutable type subclassing, singletons, or pools
- **`__repr__`** = for developers (unambiguous, ideally re-constructible); **`__str__`** = for users (readable); containers use `repr()`
- **`__format__`** powers both `format()` and f-string format specs: `f"{obj:spec}"`
- **`__eq__` + `__hash__` contract**: equal objects must have equal hashes; defining `__eq__` without `__hash__` makes instances unhashable
- **`@total_ordering`**: implement `__eq__` + ONE comparison; decorator generates the rest
- **`NotImplemented`** (value) = "try the reflected op on the other operand"; **`NotImplementedError`** (exception) = "abstract method not implemented"
- **`__radd__`** is tried when `other.__add__(self)` returns `NotImplemented` — enables `5 + MyObj()`
- **`__iadd__`** for in-place operators — should return `self` (or modified self) for mutable types
- **`__bool__`** controls truthiness; if absent, `__len__` is used; if both absent, always `True`
- **Iteration**: `__iter__` returns an iterator; `__next__` advances it; `StopIteration` signals end
- **`__enter__`** / **`__exit__`**: context manager protocol; `__exit__` returning `True` suppresses the exception
- **`__call__`** makes instances callable; use for stateful callables, decorators, memoization
- **`__getattr__`** = fallback (after normal lookup fails); **`__getattribute__`** = intercepts every access
- **Descriptors**: `__get__`, `__set__`, `__delete__`, `__set_name__` — the mechanism behind `property`, `classmethod`, `staticmethod`
- **`__getstate__` / `__setstate__`**: custom pickle serialisation
- **`__slots__` + dunders**: `__slots__` affects storage but not dunder dispatch
