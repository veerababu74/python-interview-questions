# `__init__` and Constructors

## 1. Concept (Deep Explanation)

### What It Is and Why It Exists

`__init__` is the **initialiser** method called automatically after a new instance is created. It is *not* the constructor in the C++/Java sense — Python splits construction into two phases:

1. **`__new__(cls, *args, **kwargs)`** — the true constructor. A static method that allocates memory and returns a new (possibly uninitialised) instance of the class.
2. **`__init__(self, *args, **kwargs)`** — the initialiser. Receives the already-allocated instance and populates it with initial state.

The separation exists because Python needs `__new__` for immutable types (`str`, `int`, `tuple`, `frozenset`) where attributes must be set at allocation time, since you cannot reassign `self` inside `__init__` on an immutable object.

### CPython Internals: What Happens During `Foo(1, 2)`

```
Foo(1, 2)
  → Foo.__call__(1, 2)            # type.__call__ is invoked
      → obj = Foo.__new__(Foo, 1, 2)
      → if isinstance(obj, Foo):
            Foo.__init__(obj, 1, 2)
      → return obj
```

The key is `type.__call__`. When you write `Foo(1, 2)`, you're actually calling the **class** like a function, which invokes `type.__call__` (since `Foo`'s type is `type`). `type.__call__` orchestrates `__new__` and `__init__`.

### `__init__` Signature and Type Annotations

```python
from __future__ import annotations

class BankAccount:
    def __init__(
        self,
        owner: str,
        balance: float = 0.0,
        *,
        currency: str = "USD",
    ) -> None:
        self.owner = owner
        self.balance = balance
        self.currency = currency
        self._transactions: list[float] = []
```

**`__init__` must return `None`**. Returning anything else raises `TypeError`.

### Multiple Initialisers via Class Methods (Factory Pattern)

Python doesn't support method overloading by signature, so the idiomatic way to provide multiple construction paths is via `@classmethod` factories:

```python
from datetime import date

class Employee:
    def __init__(self, name: str, birth_year: int) -> None:
        self.name = name
        self.birth_year = birth_year

    @classmethod
    def from_string(cls, emp_str: str) -> "Employee":
        name, birth_year = emp_str.split("-")
        return cls(name, int(birth_year))

    @classmethod
    def from_birth_date(cls, name: str, birth_date: date) -> "Employee":
        return cls(name, birth_date.year)

e1 = Employee("Alice", 1990)
e2 = Employee.from_string("Bob-1985")
e3 = Employee.from_birth_date("Carol", date(1992, 6, 15))
```

Real-world examples: `datetime.fromtimestamp()`, `dict.fromkeys()`, `int.from_bytes()`.

### Calling `super().__init__()` in Inheritance

When a subclass defines `__init__`, it should almost always call `super().__init__()` to ensure parent classes are properly initialised. This is especially critical in multiple-inheritance hierarchies where Python's MRO dictates call order.

```python
class Vehicle:
    def __init__(self, make: str, model: str) -> None:
        self.make = make
        self.model = model

class ElectricVehicle(Vehicle):
    def __init__(self, make: str, model: str, range_km: int) -> None:
        super().__init__(make, model)   # must call or Vehicle.__init__ is skipped
        self.range_km = range_km

ev = ElectricVehicle("Tesla", "Model 3", 500)
print(ev.make, ev.range_km)  # Tesla 500
```

### `__new__` Deep Dive

`__new__` is needed for:
- **Immutable types subclassing** (you can't change a `str`'s value in `__init__`, it's already allocated)
- **Singleton patterns**
- **Object pooling / flyweight**
- **Custom metaclass behaviour**

```python
# Subclassing an immutable type
class UpperStr(str):
    def __new__(cls, value: str) -> "UpperStr":
        instance = super().__new__(cls, value.upper())
        return instance

s = UpperStr("hello")
print(s)          # HELLO
print(type(s))    # <class '__main__.UpperStr'>
print(s + " world")  # HELLO world (still a str)
```

```python
# __new__ returning a different type — __init__ is skipped
class Tricky:
    def __new__(cls):
        return 42   # returns an int, not a Tricky instance

    def __init__(self):
        print("__init__ called")  # never runs

obj = Tricky()
print(obj)           # 42
print(type(obj))     # <class 'int'>
```

### `__init_subclass__` — Hook for Subclass Creation

Python 3.6+ allows you to intercept subclass creation with `__init_subclass__`:

```python
class Plugin:
    _registry: dict[str, type] = {}

    def __init_subclass__(cls, plugin_name: str = "", **kwargs) -> None:
        super().__init_subclass__(**kwargs)
        if plugin_name:
            Plugin._registry[plugin_name] = cls

class AuthPlugin(Plugin, plugin_name="auth"):
    pass

class LogPlugin(Plugin, plugin_name="log"):
    pass

print(Plugin._registry)
# {'auth': <class 'AuthPlugin'>, 'log': <class 'LogPlugin'>}
```

### Post-Init Pattern with `dataclasses`

`dataclasses.dataclass` generates `__init__` automatically. You can run post-initialisation logic in `__post_init__`:

```python
from dataclasses import dataclass, field

@dataclass
class Rectangle:
    width: float
    height: float
    area: float = field(init=False)

    def __post_init__(self) -> None:
        if self.width <= 0 or self.height <= 0:
            raise ValueError("Dimensions must be positive")
        self.area = self.width * self.height

r = Rectangle(4.0, 5.0)
print(r.area)   # 20.0
```

### Pitfalls / Gotchas / Interview Traps

**Mutable default arguments:**
```python
# BUG — the list is created once and shared
def __init__(self, items=[]):   # never do this
    self.items = items

# Correct
def __init__(self, items=None):
    self.items = items if items is not None else []
```

**Forgetting `super().__init__()` in multiple inheritance:**
```python
class A:
    def __init__(self): self.x = 1

class B(A):
    def __init__(self):
        # forgot super().__init__()
        self.y = 2

class C(B):
    def __init__(self):
        super().__init__()  # calls B.__init__ only via MRO
        self.z = 3

c = C()
print(hasattr(c, 'x'))  # False — A.__init__ was never called
```

**`__init__` returning a value:**
```python
class Broken:
    def __init__(self):
        return 42   # TypeError: __init__() should return None
```

**Over-initialising — doing too much in `__init__`:**  
Network calls, file I/O, heavy computation in `__init__` makes testing and subclassing painful. Use lazy initialisation or factory methods.

### Best Practices

- Keep `__init__` focused on setting attributes; avoid side effects
- Use type annotations for all parameters with `-> None` return type
- Validate inputs early and raise descriptive exceptions
- Use `@classmethod` factories for alternate construction paths
- Always call `super().__init__()` in subclasses
- Prefer keyword-only arguments (`*`) for clarity when there are many parameters

---

## 2. Conceptual Interview Questions (10)

**Q1: What is the difference between `__new__` and `__init__`?**

**A:** `__new__` is responsible for **creating** a new instance (memory allocation), while `__init__` is responsible for **initialising** the already-created instance. `__new__` is a static method that receives the class as its first argument and must return an object. If it returns an instance of the class, `__init__` is then called on that instance. `__init__` must return `None`. You typically only override `__new__` when subclassing immutable types or implementing patterns like Singleton, where you need to control object creation at the allocation level.

```python
class Demo:
    def __new__(cls):
        print(f"__new__ called, cls={cls}")
        return super().__new__(cls)

    def __init__(self):
        print(f"__init__ called, self={self}")

d = Demo()
# __new__ called, cls=<class '__main__.Demo'>
# __init__ called, self=<__main__.Demo object at 0x...>
```

---

**Q2: What does `super().__init__()` do and why is it important?**

**A:** `super()` returns a proxy object that delegates method calls to the next class in the MRO. Calling `super().__init__()` ensures that the parent class's initialisation code runs, which may set up critical attributes or register the object in some container. In multiple-inheritance scenarios, Python's MRO (C3 linearisation) ensures that `super().__init__()` calls proceed cooperatively through the entire hierarchy, calling each class's `__init__` exactly once — but only if every class in the chain consistently calls `super().__init__()`.

---

**Q3: Can `__init__` be a generator or async function?**

**A:** No, and no. `__init__` must be a regular function returning `None`. If it's a generator, it returns a generator object (not `None`), causing `TypeError`. For async initialisation, the pattern is to use a factory `@classmethod` that is `async`:

```python
class AsyncResource:
    def __init__(self, data): self.data = data

    @classmethod
    async def create(cls) -> "AsyncResource":
        data = await some_async_fetch()
        return cls(data)

# Usage: resource = await AsyncResource.create()
```

---

**Q4: What happens if `__new__` doesn't return an instance of the class?**

**A:** If `__new__` returns an object that is *not* an instance of `cls`, then `__init__` is **not called**. Python's `type.__call__` checks `isinstance(obj, cls)` before calling `__init__`. This is a subtle but important detail.

```python
class Weird:
    def __new__(cls):
        return "a string, not a Weird"

    def __init__(self):
        print("This never runs")

obj = Weird()
print(type(obj))   # <class 'str'>
```

---

**Q5: How do you implement alternative constructors in Python?**

**A:** Using `@classmethod` as a factory method is the idiomatic approach. The classmethod receives the class as its first argument (`cls`) instead of the instance, making it subclass-friendly — a subclass will receive itself as `cls`, so the factory creates the right type.

```python
class Color:
    def __init__(self, r: int, g: int, b: int) -> None:
        self.r, self.g, self.b = r, g, b

    @classmethod
    def from_hex(cls, hex_str: str) -> "Color":
        hex_str = hex_str.lstrip("#")
        r, g, b = (int(hex_str[i:i+2], 16) for i in (0, 2, 4))
        return cls(r, g, b)

    @classmethod
    def from_tuple(cls, t: tuple[int, int, int]) -> "Color":
        return cls(*t)

c = Color.from_hex("#FF5733")
print(c.r, c.g, c.b)  # 255 87 51
```

---

**Q6: Why is using a mutable object as a default parameter in `__init__` dangerous?**

**A:** Default parameter values in Python are evaluated **once** at function definition time, not on each call. A mutable default (like `[]` or `{}`) is shared across all calls that don't pass a value. Any mutation affects all future calls that use the default.

```python
class Tag:
    def __init__(self, name: str, attributes: dict = {}):  # danger!
        self.name = name
        self.attributes = attributes

t1 = Tag("p")
t1.attributes["class"] = "bold"
t2 = Tag("span")
print(t2.attributes)  # {'class': 'bold'} — shared default!

# Fix:
class Tag:
    def __init__(self, name: str, attributes: dict | None = None):
        self.name = name
        self.attributes = attributes if attributes is not None else {}
```

---

**Q7: How does `__init_subclass__` work and what is it used for?**

**A:** `__init_subclass__` is a class method called on the parent class **whenever** it is subclassed. It is used to register subclasses automatically, enforce constraints, inject behaviour, or build plugin systems without metaclasses. It was introduced in PEP 487 (Python 3.6).

```python
class Validator:
    _validators: dict[str, type] = {}

    def __init_subclass__(cls, field: str, **kwargs):
        super().__init_subclass__(**kwargs)
        Validator._validators[field] = cls

class EmailValidator(Validator, field="email"):
    def validate(self, val): return "@" in val

print(Validator._validators)  # {'email': <class 'EmailValidator'>}
```

---

**Q8: What is `__post_init__` and when is it used?**

**A:** `__post_init__` is a hook specific to Python's `dataclasses`. When `@dataclass` generates `__init__`, it calls `self.__post_init__()` at the end of the generated `__init__`. This allows you to run additional initialisation logic (validation, derived-field computation) without overriding the generated `__init__`. Fields with `field(init=False)` are not passed to `__init__` but can be computed in `__post_init__`.

---

**Q9: Is it possible to make `__init__` behave differently based on the type of arguments passed?**

**A:** Yes, through several approaches. You can use `isinstance` checks, `*args`/`**kwargs` introspection, or more elegantly, the `singledispatch` pattern. However, the cleanest Python approach is to use `@classmethod` factories for distinct construction signatures, which keeps the intent clear.

```python
from functools import singledispatch

class DataWrapper:
    def __init__(self, data):
        self.data = self._normalise(data)

    @singledispatch
    def _normalise(self, data):
        return data

    # Alternatively, use classmethods:
    @classmethod
    def from_json(cls, json_str: str) -> "DataWrapper":
        import json
        return cls(json.loads(json_str))
```

---

**Q10: What is "cooperative multiple inheritance" and how does `super().__init__()` support it?**

**A:** Cooperative multiple inheritance means each class in an MRO chain passes control to the next class via `super()`, ensuring every base class is initialised even across a diamond-shaped hierarchy. All classes must cooperate — they all must call `super().__init__()` with appropriate `**kwargs` forwarding.

```python
class A:
    def __init__(self, **kwargs):
        print("A.__init__")
        super().__init__(**kwargs)

class B(A):
    def __init__(self, **kwargs):
        print("B.__init__")
        super().__init__(**kwargs)

class C(A):
    def __init__(self, **kwargs):
        print("C.__init__")
        super().__init__(**kwargs)

class D(B, C):
    def __init__(self, **kwargs):
        print("D.__init__")
        super().__init__(**kwargs)

D()
# D.__init__ → B.__init__ → C.__init__ → A.__init__
# Each called exactly once, following MRO: D → B → C → A → object
```

---

## 3. Scenario-Based Interview Questions (10)

**Q1: You're building a configuration class for a web server. The configuration can come from a YAML file, environment variables, or command-line arguments. How do you design the `__init__` and constructors?**

**A:** Use a central `__init__` for the normalised config and `@classmethod` factories for each source:

```python
import os
from pathlib import Path

class ServerConfig:
    def __init__(
        self,
        host: str = "0.0.0.0",
        port: int = 8080,
        debug: bool = False,
        *,
        workers: int = 4,
    ) -> None:
        self.host = host
        self.port = int(port)   # normalise type
        self.debug = bool(debug)
        self.workers = int(workers)

    @classmethod
    def from_yaml(cls, path: str | Path) -> "ServerConfig":
        import yaml
        with open(path) as f:
            data = yaml.safe_load(f)
        return cls(**data)

    @classmethod
    def from_env(cls) -> "ServerConfig":
        return cls(
            host=os.environ.get("HOST", "0.0.0.0"),
            port=int(os.environ.get("PORT", 8080)),
            debug=os.environ.get("DEBUG", "false").lower() == "true",
        )

cfg = ServerConfig.from_env()
```

---

**Q2: A legacy codebase has a class with 15 parameters in `__init__`. How do you refactor this?**

**A:** Options in order of preference:
1. Group related parameters into smaller config objects (composition)
2. Use `@dataclass` to reduce boilerplate and add defaults
3. Use the Builder pattern for complex construction

```python
from dataclasses import dataclass, field

@dataclass
class DatabaseConfig:
    host: str = "localhost"
    port: int = 5432
    name: str = "mydb"

@dataclass
class CacheConfig:
    url: str = "redis://localhost"
    ttl: int = 300

@dataclass
class AppConfig:
    db: DatabaseConfig = field(default_factory=DatabaseConfig)
    cache: CacheConfig = field(default_factory=CacheConfig)
    debug: bool = False
```

---

**Q3: You need to subclass Python's built-in `list` to create a list that only accepts integers. How do you do this?**

**A:** Subclass `list` and override `append`, `extend`, `__setitem__`, and `__init__`:

```python
class IntList(list):
    def __init__(self, iterable=()) -> None:
        super().__init__(self._validate(iterable))

    def _validate(self, iterable):
        result = []
        for item in iterable:
            if not isinstance(item, int):
                raise TypeError(f"Expected int, got {type(item).__name__}")
            result.append(item)
        return result

    def append(self, item: int) -> None:
        if not isinstance(item, int):
            raise TypeError(f"Expected int, got {type(item).__name__}")
        super().append(item)

    def __setitem__(self, index, value):
        if isinstance(index, slice):
            value = list(self._validate(value))
        elif not isinstance(value, int):
            raise TypeError(f"Expected int, got {type(value).__name__}")
        super().__setitem__(index, value)

il = IntList([1, 2, 3])
il.append(4)
print(il)        # [1, 2, 3, 4]
il.append("x")  # TypeError
```

---

**Q4: A class's `__init__` makes a network request. Tests are slow and flaky. How do you fix this?**

**A:** The problem is coupling side effects to initialisation. Refactor using dependency injection and lazy initialisation:

```python
from typing import Protocol

class HttpClient(Protocol):
    def get(self, url: str) -> dict: ...

class ApiClient:
    def __init__(self, base_url: str, http: HttpClient | None = None) -> None:
        self.base_url = base_url
        self._http = http   # injected; no network call here
        self._data: dict | None = None

    def _get_http(self) -> HttpClient:
        if self._http is None:
            import httpx
            self._http = httpx.Client()
        return self._http

    def fetch_user(self, user_id: int) -> dict:
        return self._get_http().get(f"{self.base_url}/users/{user_id}")

# In tests, inject a mock:
class MockHttp:
    def get(self, url): return {"id": 1, "name": "Alice"}

client = ApiClient("https://api.example.com", http=MockHttp())
```

---

**Q5: How do you ensure a subclass always calls `super().__init__()`?**

**A:** Use `__init_subclass__` to wrap the subclass's `__init__`, or use a metaclass check. A simpler approach is documentation + `__init_subclass__` validation:

```python
class Base:
    _initialized: bool = False

    def __init__(self) -> None:
        self._initialized = True

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        original_init = cls.__dict__.get("__init__")
        if original_init:
            def checked_init(self, *args, **kw):
                original_init(self, *args, **kw)
                if not Base._initialized:
                    raise RuntimeError(
                        f"{cls.__name__}.__init__ must call super().__init__()"
                    )
            cls.__init__ = checked_init
```

---

**Q6: Implement a class that records all arguments passed to `__init__` for later serialisation.**

**A:**

```python
import inspect

class Serialisable:
    def __init__(self, *args, **kwargs) -> None:
        sig = inspect.signature(self.__class__.__init__)
        params = list(sig.parameters.keys())[1:]  # skip 'self'
        bound = sig.bind(self, *args, **kwargs)
        bound.apply_defaults()
        self._init_kwargs = dict(list(bound.arguments.items())[1:])

    def to_dict(self) -> dict:
        return {"type": type(self).__name__, **self._init_kwargs}

class Point(Serialisable):
    def __init__(self, x: float, y: float, label: str = "P") -> None:
        super().__init__(x, y, label=label)
        self.x, self.y, self.label = x, y, label

p = Point(1.0, 2.0)
print(p.to_dict())  # {'type': 'Point', 'x': 1.0, 'y': 2.0, 'label': 'P'}
```

---

**Q7: You need to implement a class pool that reuses instances rather than creating new ones. How?**

**A:** Override `__new__` to return an existing instance from a pool:

```python
class Connection:
    _pool: list["Connection"] = []
    _max_size: int = 3

    def __new__(cls) -> "Connection":
        if cls._pool:
            return cls._pool.pop()
        return super().__new__(cls)

    def __init__(self) -> None:
        self.active = True

    def close(self) -> None:
        self.active = False
        Connection._pool.append(self)

c1 = Connection()
c1.close()
c2 = Connection()   # reuses c1's memory
print(c1 is c2)     # True
```

---

**Q8: How do you implement a class that can be constructed from JSON?**

**A:**

```python
import json
from typing import Any

class JsonDeserialisable:
    @classmethod
    def from_json(cls, json_str: str) -> "JsonDeserialisable":
        data = json.loads(json_str)
        return cls(**data)

    def to_json(self) -> str:
        return json.dumps(vars(self))

class User(JsonDeserialisable):
    def __init__(self, name: str, age: int, email: str) -> None:
        self.name = name
        self.age = age
        self.email = email

u = User.from_json('{"name": "Alice", "age": 30, "email": "alice@ex.com"}')
print(u.name, u.age)          # Alice 30
print(u.to_json())             # {"name": "Alice", "age": 30, "email": "..."}
```

---

**Q9: Describe a situation where you'd override both `__new__` and `__init__`.**

**A:** Subclassing an immutable type while also needing custom attributes:

```python
class CurrencyAmount(float):
    """An immutable float with a currency tag."""

    def __new__(cls, value: float, currency: str = "USD") -> "CurrencyAmount":
        instance = super().__new__(cls, value)   # value is baked in here
        return instance

    def __init__(self, value: float, currency: str = "USD") -> None:
        # Can't change the float value here; it's immutable
        # But we can add mutable attributes
        self.currency = currency

amount = CurrencyAmount(19.99, "EUR")
print(float(amount))    # 19.99
print(amount.currency)  # EUR
print(amount + 5)       # 24.99 (float arithmetic, loses currency)
```

---

**Q10: A class has a very expensive computation in `__init__`. You want to defer it until the result is actually needed. How do you implement lazy initialisation?**

**A:** Use a `property` backed by a sentinel value, or `functools.cached_property`:

```python
from functools import cached_property

class DataAnalyser:
    def __init__(self, filepath: str) -> None:
        self.filepath = filepath
        # No computation here

    @cached_property
    def data(self) -> list:
        print("Loading data (expensive)...")
        with open(self.filepath) as f:
            return [line.strip() for line in f]

    @cached_property
    def word_count(self) -> int:
        return sum(len(line.split()) for line in self.data)

# Usage
analyser = DataAnalyser("large_file.txt")
# No I/O yet
count = analyser.word_count  # I/O happens here, once
count2 = analyser.word_count  # served from cache
```

---

## 4. Quick Recap / Cheatsheet

- **`__new__`** creates the object (allocates memory); **`__init__`** initialises it
- `__init__` must return `None`; returning anything else raises `TypeError`
- Call `super().__init__()` in subclasses or parent initialisation is silently skipped
- Mutable default arguments in `__init__` are a classic bug — use `None` and create inside
- Provide alternative constructors via `@classmethod` factories (à la `dict.fromkeys`)
- `__init_subclass__` lets a base class react when it is subclassed — great for plugin registries
- `__post_init__` is the `dataclasses`-specific hook for post-construction logic
- Override `__new__` when subclassing immutable types (`str`, `int`, `tuple`)
- Dependency injection in `__init__` makes code testable; avoid side effects in constructors
- `functools.cached_property` enables lazy, one-time-computed attributes
- `type.__call__` orchestrates `__new__` → `__init__`; understanding this demystifies metaclasses
- For cooperative multiple inheritance, every `__init__` must call `super().__init__(**kwargs)`
