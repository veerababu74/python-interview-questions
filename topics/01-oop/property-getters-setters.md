# Properties, Getters, and Setters

## 1. Concept (Deep Explanation)

### What It Is and Why It Exists

Python's `property` built-in allows you to define methods that are accessed like attributes. It exists to support a fundamental Python philosophy: **start with a plain attribute; upgrade to a property if you need to add behaviour — without changing the calling API**.

In languages like Java, the convention is to always write `getName()` / `setName()` from the very beginning, even when no validation or computation is needed, "just in case". Python avoids this ceremony: you start with `self.name = name`, and *if* you later need validation, caching, or computation, you swap the attribute for a `property` — and every caller is unchanged because the access syntax (`obj.name`, `obj.name = value`) stays the same.

### `property` as a Data Descriptor

`property` is not magic — it is a regular Python class that implements the **descriptor protocol**. Specifically, it is a **data descriptor** because it defines both `__get__` and `__set__` (and optionally `__delete__`).

```python
# property is literally a class defined in CPython (Objects/descrobject.c)
# Its Python equivalent would look like:

class property:
    def __init__(self, fget=None, fset=None, fdel=None, doc=None):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel
        self.__doc__ = doc or (fget.__doc__ if fget else None)

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self          # accessed on the class → return descriptor itself
        if self.fget is None:
            raise AttributeError("unreadable attribute")
        return self.fget(obj)

    def __set__(self, obj, value):
        if self.fset is None:
            raise AttributeError("can't set attribute")
        self.fset(obj, value)

    def __delete__(self, obj):
        if self.fdel is None:
            raise AttributeError("can't delete attribute")
        self.fdel(obj)

    def getter(self, fget):
        return type(self)(fget, self.fset, self.fdel, self.__doc__)

    def setter(self, fset):
        return type(self)(self.fget, fset, self.fdel, self.__doc__)

    def deleter(self, fdel):
        return type(self)(self.fget, self.fset, fdel, self.__doc__)
```

Because `property` is a **data descriptor** (has `__set__`), it takes precedence over the instance `__dict__` during attribute lookup. This is why a property defined on the class will intercept `obj.attr = value` even though that would normally write to `obj.__dict__['attr']`.

### CPython Attribute Lookup Priority

```
Instance __dict__   ← only for non-data descriptors
  ↑
Data Descriptor (class level — has __get__ AND __set__)
  ↑
Non-data Descriptor (class level — only __get__)
  ↑
class __dict__ fallback
```

Because `property` has `__set__`, it sits at the top: `obj.x = value` calls `property.__set__`, not `obj.__dict__['x'] = value`.

### `@property` Decorator Syntax

The `@property` decorator is syntactic sugar for the `property()` call:

```python
class Temperature:
    def __init__(self, celsius: float) -> None:
        self._celsius = celsius  # underscore: "internal" attribute

    @property
    def celsius(self) -> float:
        """Get temperature in Celsius."""
        return self._celsius

    @celsius.setter
    def celsius(self, value: float) -> None:
        if value < -273.15:
            raise ValueError(f"Temperature {value}°C is below absolute zero")
        self._celsius = value

    @celsius.deleter
    def celsius(self) -> None:
        print("Deleting temperature")
        del self._celsius

    @property
    def fahrenheit(self) -> float:
        """Computed property — no setter, read-only."""
        return self._celsius * 9/5 + 32


t = Temperature(25.0)
print(t.celsius)      # 25.0  — calls fget
print(t.fahrenheit)   # 77.0  — computed, no backing attribute

t.celsius = 100.0     # calls fset
print(t.fahrenheit)   # 212.0

t.celsius = -300.0    # raises ValueError!

del t.celsius         # calls fdel
```

### Full `property()` Function Form

```python
class Circle:
    def __init__(self, radius: float) -> None:
        self._radius = radius

    def _get_radius(self) -> float:
        return self._radius

    def _set_radius(self, value: float) -> None:
        if value < 0:
            raise ValueError("Radius cannot be negative")
        self._radius = value

    # Equivalent to @property + @radius.setter:
    radius = property(
        fget=_get_radius,
        fset=_set_radius,
        doc="The circle's radius. Must be non-negative.",
    )

c = Circle(5.0)
print(c.radius)   # 5.0
print(Circle.radius)  # <property object at 0x...>
help(Circle.radius)   # prints "The circle's radius. Must be non-negative."
```

### Read-Only Properties

A property with only `fget` is effectively read-only. Attempting to set raises `AttributeError`:

```python
class Immutable:
    def __init__(self, value: int) -> None:
        self._value = value

    @property
    def value(self) -> int:
        return self._value
    # No setter defined

obj = Immutable(42)
print(obj.value)   # 42
obj.value = 99     # AttributeError: can't set attribute
```

### Validation in Setters

This is the most common use case: validate data on assignment.

```python
from typing import Any

class Person:
    def __init__(self, name: str, age: int) -> None:
        # Note: go through the setter for validation on __init__ too
        self.name = name
        self.age = age

    @property
    def name(self) -> str:
        return self._name

    @name.setter
    def name(self, value: Any) -> None:
        if not isinstance(value, str):
            raise TypeError(f"name must be str, got {type(value).__name__}")
        if not value.strip():
            raise ValueError("name cannot be blank")
        self._name = value.strip()

    @property
    def age(self) -> int:
        return self._age

    @age.setter
    def age(self, value: Any) -> None:
        if not isinstance(value, int):
            raise TypeError(f"age must be int, got {type(value).__name__}")
        if not 0 <= value <= 150:
            raise ValueError(f"age {value} out of realistic range [0, 150]")
        self._age = value

p = Person("Alice", 30)
p.name = "  Bob  "   # strips whitespace
print(p.name)         # "Bob"
p.age = -1            # ValueError
```

### Property in Inheritance: Overriding Getters/Setters in Subclasses

This is a common **interview trap**. You must redefine the **entire property** in the subclass, not just the setter:

```python
class Base:
    @property
    def value(self) -> int:
        return self._value

    @value.setter
    def value(self, v: int) -> None:
        self._value = v


class Child(Base):
    # WRONG — this silently discards the setter from Base:
    # @property
    # def value(self) -> int:
    #     return super().value * 2

    # CORRECT — must recreate the full property:
    @Base.value.getter
    def value(self) -> int:
        return super().value * 2
    # This creates a NEW property with the parent's setter and this getter.


c = Child()
c.value = 5
print(c.value)   # 10 — getter doubles it; setter from Base is preserved
```

Alternatively, use explicit inheritance:

```python
class Child(Base):
    @property
    def value(self) -> int:
        return super().value * 2

    @value.setter
    def value(self, v: int) -> None:
        Base.value.fset(self, v)  # explicitly call parent's setter
```

### Computed / Cached Properties: `functools.cached_property`

`cached_property` (added in Python 3.8) computes the value once and caches it in the instance `__dict__`. It is a **non-data descriptor** (no `__set__`), so the cached value in `__dict__` takes precedence on subsequent access.

```python
import functools
import time

class DataLoader:
    def __init__(self, path: str) -> None:
        self.path = path

    @functools.cached_property
    def data(self) -> list[str]:
        """Expensive load — happens only once."""
        print(f"Loading from {self.path}...")
        time.sleep(0.1)  # simulate IO
        with open(self.path) as f:
            return f.readlines()

loader = DataLoader("data.txt")
_ = loader.data   # triggers load
_ = loader.data   # returns cached value, no load
```

Caveat: `cached_property` stores in `__dict__`, so it doesn't work with `__slots__` (no `__dict__`). For thread-safety, the computation may run more than once in a multithreaded context; use a lock or `@property` with explicit caching if needed:

```python
import threading

class ThreadSafeComputed:
    _lock = threading.Lock()

    @property
    def expensive(self) -> int:
        try:
            return self._cached
        except AttributeError:
            with self._lock:
                # double-checked locking
                if not hasattr(self, '_cached'):
                    self._cached = self._compute()
            return self._cached

    def _compute(self) -> int:
        return 42  # expensive operation
```

### `property` vs Direct Attribute Access

| Situation | Use plain attribute | Use `property` |
|---|---|---|
| Simple storage, no validation | ✓ | ✗ (over-engineering) |
| Validation needed | ✗ | ✓ |
| Computed from other attributes | ✗ | ✓ |
| Caching expensive computation | ✗ | ✓ (cached_property) |
| Backward compat when adding logic | ✗ | ✓ |
| Attribute must trigger a side effect | ✗ | ✓ |

### Python `property` vs Java Getters/Setters

In Java, the convention forces `getName()` / `setName()` everywhere from day one. Python's `property` enables the **Uniform Access Principle**: `obj.name` is syntactically identical regardless of whether `name` is a raw attribute or a property. This means:

```python
# Python — clean migration path
class User:
    def __init__(self, name: str) -> None:
        self.name = name  # plain attribute initially

# Later, without changing callers:
class User:
    def __init__(self, name: str) -> None:
        self.name = name  # still looks the same to callers

    @property
    def name(self) -> str:
        return self._name

    @name.setter
    def name(self, v: str) -> None:
        self._name = v.strip().title()
```

All existing `user.name` and `user.name = "alice"` calls work without modification.

### Pitfalls and Gotchas

- **Forgetting to use `self._x` (underscore) for backing storage** — without the underscore, `self.x = v` in the setter calls the setter recursively → `RecursionError`
- **Only overriding the getter in a subclass** silently drops the setter (see inheritance section above)
- **`cached_property` is not thread-safe** by default
- **`property` with `__slots__`**: the slot descriptor and property descriptor conflict; you cannot have both for the same name
- **`__init__` bypassing the setter**: if you write `self._name = name` directly in `__init__`, you skip validation. Use `self.name = name` to go through the setter.

### Best Practices

- Start with plain attributes; add `property` only when needed (YAGNI)
- Always use `self._attr` as the backing store to avoid recursion
- Document the property's docstring — `help(MyClass.prop)` reads `property.__doc__`
- Use `@cached_property` for expensive, idempotent computations
- Keep setters focused: one responsibility — validate and store

---

## 2. Conceptual Interview Questions (10)

**Q1: Why is `property` a data descriptor, and why does that matter?**

**A:** A descriptor is any object that defines `__get__`, `__set__`, or `__delete__`. A *data* descriptor defines both `__get__` and `__set__`. Python's attribute lookup gives data descriptors priority over the instance `__dict__`.

This matters for `property` because it means `obj.attr = value` is *always* intercepted by the property's `__set__` — even if the instance `__dict__` somehow had a key `attr`. If `property` were a non-data descriptor (only `__get__`), a simple `obj.__dict__['attr'] = 99` would shadow the property on subsequent reads, breaking the contract.

```python
class Demo:
    @property
    def x(self) -> int:
        return 42

d = Demo()
# Because property is a DATA descriptor, this won't shadow it:
d.__dict__['x'] = 99
print(d.x)  # Still 42 — data descriptor wins!
```

**Q2: What happens when you access a `property` on the class rather than an instance?**

**A:** When `obj` is `None` (i.e., accessed via the class), `property.__get__` returns the `property` object itself. This is how `@classmethod` and `@staticmethod` also behave — the descriptor's `__get__` receives `None` for `obj` and returns itself (or a bound method for classmethod).

```python
class Foo:
    @property
    def bar(self) -> int:
        return 42

print(type(Foo.bar))   # <class 'property'>
print(Foo.bar.fget)    # <function Foo.bar at 0x...>
print(Foo().bar)       # 42
```

This is useful for introspection — you can access `fget`, `fset`, `fdel`, and `__doc__` via the class.

**Q3: How does `@prop.setter` create a new property rather than mutating the existing one?**

**A:** Each call to `.setter()`, `.getter()`, or `.deleter()` creates a brand-new `property` object. This is intentional — in Python's data model, class attributes should be treated as immutable after class creation. The pattern works because:

```python
class Foo:
    @property          # step 1: creates property(fget=_x_get)
    def x(self):
        return self._x

    @x.setter          # step 2: x.setter(fset) → returns NEW property(fget, fset)
    def x(self, v):    # 'x' name is now rebound to the new property in the class dict
        self._x = v
```

Each decorator call returns a new `property`, and the `x` name in the class namespace is rebound. The final `x` in the class dict is a property with both `fget` and `fset`.

**Q4: What is the `RecursionError` trap with properties and how do you avoid it?**

**A:** If you use the same name for both the property and the backing attribute, the setter calls itself infinitely:

```python
class Bad:
    @property
    def name(self) -> str:
        return self.name   # RecursionError! calls __get__ again

    @name.setter
    def name(self, v: str) -> None:
        self.name = v   # RecursionError! calls __set__ again

class Good:
    @property
    def name(self) -> str:
        return self._name   # underscore prefix → different name → instance __dict__

    @name.setter
    def name(self, v: str) -> None:
        self._name = v   # writes to __dict__['_name'], not a property
```

The convention is to use a single leading underscore for the backing attribute name.

**Q5: How does `cached_property` differ from `property` at the descriptor protocol level?**

**A:** `cached_property` is a **non-data descriptor** — it only defines `__get__`, not `__set__`. This is deliberate: after the first call, it stores the result in `obj.__dict__[attr]`. On subsequent accesses, Python's attribute lookup finds the value in `obj.__dict__` *before* consulting the class for a non-data descriptor, so `cached_property.__get__` is never called again — the cache hit is essentially free.

```python
import functools

class Foo:
    @functools.cached_property
    def expensive(self) -> int:
        print("Computing...")
        return sum(range(1_000_000))

f = Foo()
print(f.expensive)   # "Computing..." then 499999500000
print(f.expensive)   # 499999500000 — no "Computing..."
print(f.__dict__)    # {'expensive': 499999500000}  — stored here!
```

**Q6: Can a `property` be defined on a metaclass to affect class-level attribute access?**

**A:** Yes. A `property` on a metaclass controls attribute access on the *class* object itself (since a class is an instance of its metaclass).

```python
class Meta(type):
    @property
    def species(cls) -> str:
        return f"Species: {cls.__name__}"

class Animal(metaclass=Meta):
    pass

print(Animal.species)   # "Species: Animal" — property on the class!
a = Animal()
# a.species would use normal instance lookup (no property on Animal itself)
```

This is an advanced pattern used in ORMs and frameworks to allow class-level descriptor access (e.g., `MyModel.field_name` returns a query expression, not the field value).

**Q7: What is the difference between a read-only `property` and a slot with no setter?**

**A:** A read-only `property` (no fset) raises `AttributeError: can't set attribute` when you try to assign to it. A `__slots__` entry without assignment always has a setter (the slot descriptor handles it); there is no built-in way to make a slot read-only without a property.

```python
class ReadOnlyProp:
    def __init__(self) -> None:
        self._x = 42

    @property
    def x(self) -> int:
        return self._x
    # No setter → read-only via AttributeError on set

class SlottedReadOnly:
    __slots__ = ('_x',)

    def __init__(self) -> None:
        self._x = 42

    @property
    def x(self) -> int:
        return self._x
    # __slots__ + property: slot holds _x, property exposes x read-only
```

**Q8: How do you use a `property` to implement a computed attribute that depends on other properties?**

**A:**

```python
import math

class Vector:
    def __init__(self, x: float, y: float) -> None:
        self._x = x
        self._y = y

    @property
    def x(self) -> float:
        return self._x

    @x.setter
    def x(self, v: float) -> None:
        self._x = v

    @property
    def y(self) -> float:
        return self._y

    @y.setter
    def y(self, v: float) -> None:
        self._y = v

    @property
    def magnitude(self) -> float:
        """Computed from x and y — always up-to-date."""
        return math.sqrt(self._x**2 + self._y**2)

    @property
    def angle_deg(self) -> float:
        return math.degrees(math.atan2(self._y, self._x))

v = Vector(3.0, 4.0)
print(v.magnitude)   # 5.0
v.x = 0.0
print(v.magnitude)   # 4.0 — recomputed
```

**Q9: In subclasses, what is the correct way to add a setter to a parent's read-only property?**

**A:**

```python
class Base:
    @property
    def value(self) -> int:
        return self._value


class Child(Base):
    def __init__(self, v: int) -> None:
        self._value = v

    # Correct: use the parent's property as the getter, add a setter
    @Base.value.setter
    def value(self, v: int) -> None:
        self._value = v


c = Child(10)
print(c.value)  # 10
c.value = 20
print(c.value)  # 20
```

This pattern uses `Base.value.setter(fset)` which returns a new `property` using `Base.value.fget` and the new `fset`. The new `property` is then bound to `value` in `Child`'s namespace.

**Q10: What is the `__set_name__` hook and how does it interact with `property`?**

**A:** `__set_name__` is called on descriptors when the class is created, passing the owner class and the attribute name. The built-in `property` does not use `__set_name__`, but custom descriptors can use it to self-configure:

```python
class TypedAttribute:
    def __init__(self, expected_type: type) -> None:
        self.expected_type = expected_type
        self.name: str = ""

    def __set_name__(self, owner: type, name: str) -> None:
        self.name = name  # learn our own name!

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return obj.__dict__.get(self.name)

    def __set__(self, obj, value) -> None:
        if not isinstance(value, self.expected_type):
            raise TypeError(
                f"{self.name} must be {self.expected_type.__name__}, "
                f"got {type(value).__name__}"
            )
        obj.__dict__[self.name] = value

class Config:
    host: str = TypedAttribute(str)
    port: int = TypedAttribute(int)

    def __init__(self, host: str, port: int) -> None:
        self.host = host  # validated!
        self.port = port  # validated!

c = Config("localhost", 8080)
c.port = "not-a-number"  # TypeError: port must be int, got str
```

---

## 3. Scenario-Based Interview Questions (10)

**S1: You have a `BankAccount` class with a `balance` attribute. A new requirement says the balance cannot go below zero. How do you add this without breaking existing code?**

**Q:** Implement validation using property.

**A:**

```python
class BankAccount:
    def __init__(self, owner: str, initial_balance: float = 0.0) -> None:
        self.owner = owner
        self.balance = initial_balance   # goes through setter

    @property
    def balance(self) -> float:
        return self._balance

    @balance.setter
    def balance(self, amount: float) -> None:
        if amount < 0:
            raise ValueError(f"Balance cannot be negative, got {amount}")
        self._balance = round(amount, 2)

    def withdraw(self, amount: float) -> None:
        self.balance = self.balance - amount  # setter validates automatically

    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Deposit must be positive")
        self.balance = self.balance + amount


acc = BankAccount("Alice", 100.0)
acc.deposit(50.0)
print(acc.balance)   # 150.0
acc.withdraw(200.0)  # ValueError: Balance cannot be negative
```

No callers need to change — they all access `acc.balance` the same way.

**S2: You have a `User` model where `email` should always be stored in lowercase. Implement this.**

**Q:** Use a property setter for normalisation.

**A:**

```python
import re

class User:
    _EMAIL_RE = re.compile(r'^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$')

    def __init__(self, username: str, email: str) -> None:
        self.username = username
        self.email = email

    @property
    def email(self) -> str:
        return self._email

    @email.setter
    def email(self, value: str) -> None:
        if not isinstance(value, str):
            raise TypeError("email must be a string")
        normalised = value.strip().lower()
        if not self._EMAIL_RE.match(normalised):
            raise ValueError(f"Invalid email address: {value!r}")
        self._email = normalised


u = User("alice", "Alice@Example.COM")
print(u.email)   # "alice@example.com" — normalised
u.email = "INVALID"  # ValueError
```

**S3: A `Config` class has a `timeout` attribute. Later you need to ensure it's always a positive integer and convert floats automatically. How do you refactor without breaking callers?**

**Q:** Add a property with type coercion.

**A:**

```python
class Config:
    def __init__(self, timeout: int | float = 30) -> None:
        self.timeout = timeout   # uses setter

    @property
    def timeout(self) -> int:
        return self._timeout

    @timeout.setter
    def timeout(self, value: int | float) -> None:
        value = int(value)  # coerce float → int
        if value <= 0:
            raise ValueError(f"timeout must be positive, got {value}")
        self._timeout = value

c = Config(30.9)
print(c.timeout)    # 30 — coerced
c.timeout = -1      # ValueError
c.timeout = 60.0    # Accepted, stored as 60
```

**S4: You have a `Circle` class. `radius`, `diameter`, and `area` should all be accessible as attributes and mutually consistent. Implement this.**

**Q:** Use properties for computed, bi-directional attributes.

**A:**

```python
import math

class Circle:
    def __init__(self, radius: float) -> None:
        self.radius = radius

    @property
    def radius(self) -> float:
        return self._radius

    @radius.setter
    def radius(self, v: float) -> None:
        if v < 0:
            raise ValueError("Radius must be non-negative")
        self._radius = v

    @property
    def diameter(self) -> float:
        return self._radius * 2

    @diameter.setter
    def diameter(self, v: float) -> None:
        self.radius = v / 2   # validates through radius setter

    @property
    def area(self) -> float:
        return math.pi * self._radius ** 2

    # area is read-only — it doesn't make sense to set it directly

c = Circle(5.0)
print(c.diameter)  # 10.0
c.diameter = 20.0
print(c.radius)    # 10.0
print(c.area)      # 314.159...
c.area = 100       # AttributeError: can't set attribute
```

**S5: A web framework needs a `Request` object where `body` is an expensive read from a socket, but should only be read once. Implement cached access.**

**Q:** Use `cached_property`.

**A:**

```python
import functools
from io import BytesIO

class Request:
    def __init__(self, method: str, path: str, raw_socket: BytesIO) -> None:
        self.method = method
        self.path = path
        self._socket = raw_socket

    @functools.cached_property
    def body(self) -> bytes:
        """Read body exactly once from socket."""
        print("Reading from socket...")  # would only print once
        return self._socket.read()

    @functools.cached_property
    def json(self) -> dict:
        import json
        return json.loads(self.body)  # body is cached, so no double read


import io
raw = io.BytesIO(b'{"key": "value"}')
req = Request("POST", "/api/data", raw)
print(req.body)   # reads from socket
print(req.body)   # returns cached bytes
print(req.json)   # parses cached body
```

**S6: Describe how to implement a property that logs all read and write accesses for debugging purposes.**

**Q:** Implement a debug property.

**A:**

```python
import logging
from typing import TypeVar, Generic

T = TypeVar("T")
logger = logging.getLogger(__name__)

class LoggedDescriptor(Generic[T]):
    def __set_name__(self, owner: type, name: str) -> None:
        self.public_name = name
        self.private_name = f"_{name}"

    def __get__(self, obj, objtype=None) -> T:
        if obj is None:
            return self  # type: ignore
        value = obj.__dict__.get(self.private_name, None)
        logger.debug("GET %s.%s = %r", type(obj).__name__, self.public_name, value)
        return value  # type: ignore

    def __set__(self, obj, value: T) -> None:
        old = obj.__dict__.get(self.private_name)
        logger.debug(
            "SET %s.%s: %r → %r",
            type(obj).__name__, self.public_name, old, value
        )
        obj.__dict__[self.private_name] = value

class DebuggedModel:
    name: str = LoggedDescriptor()
    score: int = LoggedDescriptor()

    def __init__(self, name: str, score: int) -> None:
        self.name = name
        self.score = score

logging.basicConfig(level=logging.DEBUG)
m = DebuggedModel("Alice", 95)
print(m.name)
m.score = 100
```

**S7: A parent class defines a read-only `id` property. A subclass `EditableEntity` should allow setting `id`. How do you implement this?**

**Q:** Override property in subclass.

**A:**

```python
class Entity:
    def __init__(self, entity_id: int) -> None:
        self._id = entity_id

    @property
    def id(self) -> int:
        return self._id
    # No setter — read-only in base


class EditableEntity(Entity):
    # Override: keep the getter from parent, add a setter
    @Entity.id.setter
    def id(self, value: int) -> None:
        if value <= 0:
            raise ValueError("id must be positive")
        self._id = value


e = EditableEntity(1)
print(e.id)   # 1
e.id = 42
print(e.id)   # 42

base = Entity(1)
base.id = 10  # AttributeError: can't set attribute
```

**S8: You have a `Temperature` class used across a codebase. You want to add Kelvin access without breaking existing `.celsius` and `.fahrenheit` usage.**

**Q:** Extend with a new property.

**A:**

```python
class Temperature:
    _ABS_ZERO = -273.15

    def __init__(self, celsius: float) -> None:
        self.celsius = celsius

    @property
    def celsius(self) -> float:
        return self._celsius

    @celsius.setter
    def celsius(self, v: float) -> None:
        if v < self._ABS_ZERO:
            raise ValueError(f"{v}°C is below absolute zero")
        self._celsius = v

    @property
    def fahrenheit(self) -> float:
        return self._celsius * 9/5 + 32

    @fahrenheit.setter
    def fahrenheit(self, v: float) -> None:
        self.celsius = (v - 32) * 5/9

    @property
    def kelvin(self) -> float:
        return self._celsius - self._ABS_ZERO

    @kelvin.setter
    def kelvin(self, v: float) -> None:
        self.celsius = v + self._ABS_ZERO  # validates through celsius setter

t = Temperature(0)
print(t.kelvin)       # 273.15
t.kelvin = 0          # absolute zero
print(t.celsius)      # -273.15
t.kelvin = -1         # ValueError: below absolute zero
```

**S9: How would you implement a property that tracks whether a value has changed since the object was created?**

**Q:** Implement a dirty-tracking property.

**A:**

```python
class DirtyTracker:
    """Descriptor that marks the object as dirty when any tracked field changes."""

    def __set_name__(self, owner: type, name: str) -> None:
        self.name = name
        self.private = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return obj.__dict__.get(self.private)

    def __set__(self, obj, value) -> None:
        if not hasattr(obj, '_dirty_fields'):
            object.__setattr__(obj, '_dirty_fields', set())
        old = obj.__dict__.get(self.private)
        if old != value:
            obj._dirty_fields.add(self.name)
        obj.__dict__[self.private] = value


class User:
    name: str = DirtyTracker()
    email: str = DirtyTracker()

    def __init__(self, name: str, email: str) -> None:
        self._dirty_fields: set[str] = set()
        self.name = name
        self.email = email
        self._dirty_fields.clear()  # initial values don't count as dirty

    @property
    def is_dirty(self) -> bool:
        return bool(self._dirty_fields)


u = User("Alice", "alice@example.com")
print(u.is_dirty)        # False
u.name = "Bob"
print(u.is_dirty)        # True
print(u._dirty_fields)   # {'name'}
```

**S10: A `Config` class needs to be frozen after initialisation — no property can be set after `__init__` completes. Implement this without `frozen=True` dataclass.**

**Q:** Use a property + flag approach.

**A:**

```python
class FrozenMixin:
    _frozen: bool = False

    def __setattr__(self, name: str, value) -> None:
        if self._frozen and not name.startswith('_frozen'):
            raise AttributeError(
                f"{type(self).__name__} is frozen — cannot set {name!r}"
            )
        super().__setattr__(name, value)

    def _freeze(self) -> None:
        object.__setattr__(self, '_frozen', True)


class Config(FrozenMixin):
    def __init__(self, host: str, port: int) -> None:
        self.host = host
        self.port = port
        self._freeze()  # lock it down


cfg = Config("localhost", 8080)
print(cfg.host)
cfg.host = "other"   # AttributeError: Config is frozen
```

---

## 4. Quick Recap / Cheatsheet

- **`property` is a data descriptor** — defines `__get__` + `__set__`, takes priority over `obj.__dict__`
- **Syntax sugar**: `@property` ≡ `foo = property(fget=foo)` at class-creation time
- **Backing attribute**: always use `self._attr` (underscore) — same name causes infinite recursion
- **Read-only**: omit the setter; assignment raises `AttributeError`
- **Computed property**: property with no backing attr — recomputed every access
- **`cached_property`**: non-data descriptor; stores result in `obj.__dict__` after first call — avoids recomputation
- **`cached_property` is NOT thread-safe** — use a lock or `@property` with explicit lock for thread safety
- **Class access**: `MyClass.prop` returns the `property` object itself (fget, fset, fdel accessible)
- **Inheritance trap**: overriding only the getter in a subclass silently drops the parent's setter — use `@Base.prop.setter` or redefine fully
- **`__init__` should use `self.x = v`** (not `self._x = v`) to invoke validation in the setter
- **`property` + `__slots__`**: must use different names for the slot and the property
- **Java comparison**: Python's Uniform Access Principle means `obj.name` syntax is identical for plain attrs and properties — no need for `getName()` boilerplate from day one
- **When to add `property`**: validation, computation, caching, backward compatibility, side effects on set/get
- **When NOT to add `property`**: simple storage with no logic — plain attribute is cleaner and faster
