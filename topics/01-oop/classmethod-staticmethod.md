# @classmethod and @staticmethod In Depth

## 1. Concept (Deep Explanation)

### What They Are and Why They Exist

Python functions defined inside a class body are, by default, **instance methods** — their first parameter (`self`) is automatically bound to the instance when called. But sometimes a method belongs to the class conceptually, not to individual instances. Python provides two built-in decorators to handle these cases:

- `@classmethod` — the first argument is the **class** (`cls`), not the instance
- `@staticmethod` — no automatic binding at all; it is a plain function that lives in the class namespace

Both exist to improve **cohesion** (keeping related logic together) and **correctness** (especially `classmethod` for inheritance-aware factory methods).

### How `@classmethod` Works as a Descriptor

Like `property`, both `classmethod` and `staticmethod` are descriptor objects (implemented in C in CPython, in `Objects/funcobject.c`). Understanding their `__get__` is key:

```python
# Python equivalent of classmethod's __get__:
class classmethod_equivalent:
    def __init__(self, func):
        self.func = func

    def __get__(self, obj, cls=None):
        if cls is None:
            cls = type(obj)
        # Returns a partial that binds cls as the first argument
        import functools
        return functools.partial(self.func, cls)


# Python equivalent of staticmethod's __get__:
class staticmethod_equivalent:
    def __init__(self, func):
        self.func = func

    def __get__(self, obj, cls=None):
        return self.func  # No binding whatsoever — returns func unchanged
```

This descriptor protocol means:
- `obj.class_method()` → `__get__(obj, type(obj))` → callable bound to `type(obj)`
- `Class.class_method()` → `__get__(None, Class)` → callable bound to `Class`
- `obj.static_method()` → `__get__(obj, type(obj))` → the raw function
- `Class.static_method()` → `__get__(None, Class)` → the raw function

```python
class Foo:
    @classmethod
    def cm(cls) -> str:
        return f"cls is {cls.__name__}"

    @staticmethod
    def sm() -> str:
        return "no binding"

foo = Foo()

# Both access patterns work identically for classmethod:
print(foo.cm())   # "cls is Foo"
print(Foo.cm())   # "cls is Foo"

# Both access patterns work identically for staticmethod:
print(foo.sm())   # "no binding"
print(Foo.sm())   # "no binding"
```

### `@classmethod` as Factory Methods (Inheritance-Aware)

This is the most important use case. A factory classmethod uses `cls()` instead of `ClassName()`, so subclasses automatically get the right return type:

```python
from __future__ import annotations
from datetime import datetime, date
import json

class Event:
    def __init__(self, name: str, timestamp: datetime) -> None:
        self.name = name
        self.timestamp = timestamp

    @classmethod
    def from_dict(cls, data: dict) -> "Event":
        """Factory: construct from a dict. Inheritance-aware."""
        return cls(
            name=data["name"],
            timestamp=datetime.fromisoformat(data["timestamp"]),
        )

    @classmethod
    def from_json(cls, json_str: str) -> "Event":
        return cls.from_dict(json.loads(json_str))

    @classmethod
    def now(cls, name: str) -> "Event":
        return cls(name, datetime.now())

    def __repr__(self) -> str:
        return f"{type(self).__name__}({self.name!r}, {self.timestamp})"


class AuditEvent(Event):
    """Subclass — factory methods still return AuditEvent, not Event."""
    def __init__(self, name: str, timestamp: datetime, user: str = "system") -> None:
        super().__init__(name, timestamp)
        self.user = user

    @classmethod
    def from_dict(cls, data: dict) -> "AuditEvent":
        obj = super().from_dict(data)
        obj.user = data.get("user", "system")
        return obj


data = {"name": "login", "timestamp": "2024-01-15T10:30:00", "user": "alice"}

event = Event.from_dict(data)
print(type(event))   # <class 'Event'>

audit = AuditEvent.from_dict(data)
print(type(audit))   # <class 'AuditEvent'> — correct type!
print(audit.user)    # "alice"
```

If `from_dict` used `Event(...)` instead of `cls(...)`, `AuditEvent.from_dict(data)` would return an `Event`, not an `AuditEvent` — a silent bug.

### `@classmethod` for Class-Level Operations (Registries, Counters)

```python
class Plugin:
    _registry: dict[str, type["Plugin"]] = {}

    def __init_subclass__(cls, /, plugin_name: str = "", **kwargs) -> None:
        super().__init_subclass__(**kwargs)
        name = plugin_name or cls.__name__.lower()
        Plugin._registry[name] = cls

    @classmethod
    def get_plugin(cls, name: str) -> type["Plugin"]:
        try:
            return cls._registry[name]
        except KeyError:
            raise ValueError(f"No plugin registered: {name!r}")

    @classmethod
    def list_plugins(cls) -> list[str]:
        return list(cls._registry.keys())


class CSVPlugin(Plugin, plugin_name="csv"):
    def process(self) -> str:
        return "processing CSV"


class JSONPlugin(Plugin, plugin_name="json"):
    def process(self) -> str:
        return "processing JSON"


print(Plugin.list_plugins())          # ['csv', 'json']
plugin_cls = Plugin.get_plugin("csv")
print(plugin_cls().process())         # "processing CSV"
```

### `@staticmethod` for Utility Functions

A static method is just a regular function that conceptually belongs with the class:

```python
import re
import hashlib

class PasswordManager:
    MIN_LENGTH = 12
    _SPECIAL_RE = re.compile(r'[!@#$%^&*(),.?":{}|<>]')

    def __init__(self, hashed_password: str) -> None:
        self._hash = hashed_password

    @staticmethod
    def hash_password(plain: str) -> str:
        """Pure function — no class or instance state needed."""
        return hashlib.sha256(plain.encode()).hexdigest()

    @staticmethod
    def validate_strength(password: str) -> bool:
        """Validation logic — could be a module function, but lives here for cohesion."""
        has_upper = any(c.isupper() for c in password)
        has_digit = any(c.isdigit() for c in password)
        has_special = bool(PasswordManager._SPECIAL_RE.search(password))
        return (
            len(password) >= PasswordManager.MIN_LENGTH
            and has_upper
            and has_digit
            and has_special
        )

    def verify(self, plain: str) -> bool:
        return self._hash == self.hash_password(plain)


print(PasswordManager.validate_strength("MyP@ssw0rd123!"))  # True
mgr = PasswordManager(PasswordManager.hash_password("MyP@ssw0rd123!"))
print(mgr.verify("MyP@ssw0rd123!"))  # True
```

### `@classmethod` Called on an Instance

A key fact that trips up candidates: calling a classmethod *on an instance* still passes the **class**, not the instance:

```python
class Foo:
    count: int = 0

    @classmethod
    def increment(cls) -> None:
        cls.count += 1
        print(f"cls is {cls.__name__}, count is {cls.count}")

foo = Foo()
foo.increment()     # cls is Foo — NOT the instance!
Foo.increment()     # cls is Foo
```

This is because `classmethod.__get__` ignores `obj` — it always binds to `type(obj)` when called on an instance.

### `@staticmethod` vs Module-Level Function

Both are just functions. The choice is about organisation:

| Factor | `@staticmethod` | Module-level function |
|---|---|---|
| Cohesion | High — lives next to related class | Lower — separated from class |
| Namespace | `ClassName.func()` | `module.func()` |
| Inheritance | Can be overridden in subclasses | Cannot |
| Testability | Same | Same |
| When to prefer | Logically belongs to the class concept | Genuinely unrelated to any class |

```python
# staticmethod CAN be overridden in a subclass:
class Formatter:
    @staticmethod
    def format(value: float) -> str:
        return f"{value:.2f}"

class PercentFormatter(Formatter):
    @staticmethod
    def format(value: float) -> str:  # overrides parent's
        return f"{value * 100:.1f}%"

print(Formatter.format(3.14))        # "3.14"
print(PercentFormatter.format(3.14)) # "314.0%"
```

### Introspection: `classmethod.__func__` and `staticmethod.__func__`

Both wrappers expose the underlying function:

```python
class Foo:
    @classmethod
    def cm(cls) -> None:
        pass

    @staticmethod
    def sm() -> None:
        pass

# Access the raw function:
print(Foo.__dict__['cm'])          # <classmethod object at 0x...>
print(Foo.__dict__['cm'].__func__) # <function Foo.cm at 0x...>
print(Foo.__dict__['sm'])          # <staticmethod object at 0x...>
print(Foo.__dict__['sm'].__func__) # <function Foo.sm at 0x...>

# Calling __func__ directly requires passing cls manually:
Foo.__dict__['cm'].__func__(Foo)   # equivalent to Foo.cm()
```

### Overriding `@classmethod` in Subclasses

```python
class Animal:
    sound: str = "..."

    @classmethod
    def speak(cls) -> str:
        return f"A {cls.__name__} says {cls.sound}"


class Dog(Animal):
    sound = "woof"

    @classmethod
    def speak(cls) -> str:
        # super() in a classmethod: cls is still Dog
        base = super().speak()
        return f"{base}! {base}!"


print(Animal.speak())   # "A Animal says ..."
print(Dog.speak())      # "A Dog says woof! A Dog says woof!"
```

### Pitfalls and Gotchas

- **Forgetting `cls` as first parameter** in a classmethod raises `TypeError`
- **Using `ClassName()` instead of `cls()`** in factory classmethods breaks subclass inheritance
- **Calling `@classmethod` on an instance** passes the class, not instance — never access `obj` attributes via `cls`
- **`@staticmethod` doesn't receive `self` or `cls`** — cannot access instance or class state without an explicit argument
- **Overriding a `@classmethod` with a regular method** (or vice versa) changes the first-arg binding — easy to introduce bugs

### Best Practices

- Use `@classmethod` for: alternative constructors (factories), operations on class-level state, registry management
- Use `@staticmethod` for: helper/utility functions that are logically tied to the class but need no class or instance data
- In factory classmethods, always use `cls(...)` not `ClassName(...)` to support subclassing
- Prefer `@staticmethod` over module-level functions when the function's purpose is inseparable from the class concept
- Avoid overusing `@staticmethod` — if a method doesn't use the class or instance at all and isn't logically tied to the class, it belongs at module level

---

## 2. Conceptual Interview Questions (10)

**Q1: What is the difference between `@classmethod`, `@staticmethod`, and a regular instance method?**

**A:** Three types of methods, differentiated by binding:

- **Instance method**: first arg is the instance (`self`). Bound to the instance via the descriptor protocol. Can access both instance state (`self.x`) and class state (`type(self).x` or `self.__class__.x`).
- **@classmethod**: first arg is the class (`cls`). Bound to the class, not the instance. Can access and modify class-level state. Inheritance-aware (receives the actual subclass when called on subclass).
- **@staticmethod**: no automatic binding. No `self` or `cls`. A plain function in the class namespace. Cannot access instance or class state without explicit parameters.

```python
class Example:
    class_var: int = 0

    def __init__(self, x: int) -> None:
        self.x = x

    def instance_method(self) -> str:
        return f"instance: self.x={self.x}, class_var={self.class_var}"

    @classmethod
    def class_method(cls) -> str:
        return f"class: cls={cls.__name__}, class_var={cls.class_var}"

    @staticmethod
    def static_method(n: int) -> int:
        return n * 2   # no access to self or cls

e = Example(42)
print(e.instance_method())   # "instance: self.x=42, class_var=0"
print(e.class_method())      # "class: cls=Example, class_var=0"
print(e.static_method(5))    # 10
```

**Q2: Why should factory classmethods use `cls()` instead of `ClassName()`?**

**A:** Using `cls()` makes the factory polymorphic — when called on a subclass, it creates an instance of the subclass, not the parent. Using `ClassName()` always returns the parent type, making the factory useless for subclasses.

```python
class Animal:
    def __init__(self, name: str) -> None:
        self.name = name

    @classmethod
    def create(cls, name: str) -> "Animal":
        return cls(name)  # cls() — correct!
        # return Animal(name)  # WRONG — always returns Animal, never Dog


class Dog(Animal):
    def bark(self) -> str:
        return "Woof!"


dog = Dog.create("Rex")
print(type(dog))   # <class 'Dog'> — correct
print(dog.bark())  # "Woof!"
```

**Q3: Explain how `@classmethod` and `@staticmethod` use the descriptor protocol.**

**A:** Both are descriptor objects stored in the class `__dict__`. Their `__get__` controls what happens when they are accessed:

- `classmethod.__get__(obj, cls)` returns a **bound method** where `cls` is pre-filled as the first argument. If `obj` is not `None`, it uses `type(obj)` as the class.
- `staticmethod.__get__(obj, cls)` returns the **raw function** unchanged — no binding occurs.

This is why the behaviour is consistent whether you call on the class or an instance:

```python
class Foo:
    @classmethod
    def cm(cls) -> type:
        return cls

    @staticmethod
    def sm() -> str:
        return "static"

foo = Foo()
assert foo.cm() is Foo      # True — cls = type(foo) = Foo
assert Foo.cm() is Foo      # True — cls = Foo
assert foo.sm() == "static" # True — no binding
assert Foo.sm() == "static" # True — no binding
```

**Q4: When should you use `@staticmethod` vs a module-level function?**

**A:** Use `@staticmethod` when the function is *conceptually coupled* to the class — a reader would naturally look for it there — but does not need class or instance state. Use a module-level function when the function is genuinely general-purpose and not tied to any particular class.

```python
# Good staticmethod: only makes sense in the context of User
class User:
    @staticmethod
    def hash_password(plain: str) -> str:
        import hashlib
        return hashlib.sha256(plain.encode()).hexdigest()


# Module-level is better: this is general math, unrelated to any class
def clamp(value: float, low: float, high: float) -> float:
    return max(low, min(high, value))
```

Also, `@staticmethod` can be overridden in subclasses; module-level functions cannot.

**Q5: What is the output of this code and why?**

```python
class Base:
    @classmethod
    def make(cls) -> "Base":
        return cls()

class Child(Base):
    pass

obj = Child.make()
print(type(obj))
```

**A:** The output is `<class '__main__.Child'>`. When `Child.make()` is called, Python's descriptor protocol binds `cls` to `Child` (since `classmethod.__get__` uses the class through which the method is accessed). Inside `make`, `cls()` is therefore `Child()`, returning a `Child` instance. This is the inheritance-aware factory pattern.

**Q6: Can you call a `@classmethod` via `super()` inside a `@classmethod`?**

**A:** Yes. `super()` inside a classmethod works just as in instance methods. `super().some_classmethod()` will look up `some_classmethod` in the MRO above the current class, but `cls` will still be the original class (not the class where the method is defined):

```python
class A:
    @classmethod
    def greet(cls) -> str:
        return f"Hello from A, cls={cls.__name__}"

class B(A):
    @classmethod
    def greet(cls) -> str:
        parent_greeting = super().greet()
        return f"B extends: {parent_greeting}"

print(B.greet())
# "B extends: Hello from A, cls=B"
# Note: cls is B inside A.greet, not A!
```

**Q7: What is `__func__` on a classmethod and when would you use it?**

**A:** `__func__` is the underlying unbound function that the `classmethod` wraps. It is accessed via the descriptor object stored in the class `__dict__`:

```python
class Foo:
    @classmethod
    def cm(cls, x: int) -> int:
        return x * 2

# Accessing the descriptor object:
descriptor = Foo.__dict__['cm']
print(descriptor.__func__)  # <function Foo.cm at 0x...>

# Calling __func__ manually (must pass cls explicitly):
result = descriptor.__func__(Foo, 5)
print(result)  # 10
```

Use cases: framework code that inspects decorated methods, copying/wrapping classmethods, serialisation of methods for distributed computing.

**Q8: How does `@staticmethod` differ from `@classmethod` when both are inherited?**

**A:** Both can be inherited, but the key difference is in what information flows to the method:

```python
class Parent:
    label: str = "parent"

    @classmethod
    def class_info(cls) -> str:
        return f"cls.label = {cls.label}"

    @staticmethod
    def static_info() -> str:
        return "static — no cls available!"

class Child(Parent):
    label = "child"

print(Parent.class_info())   # "cls.label = parent"
print(Child.class_info())    # "cls.label = child" — polymorphic!
print(Child.static_info())   # "static — no cls available!" — same result
```

`@classmethod` is *polymorphic on the class* — `cls` changes depending on which class it's called from. `@staticmethod` is not — it always runs the same code with no class context.

**Q9: What happens if you accidentally define a classmethod without the `cls` parameter?**

**A:** Python doesn't validate the signature at class-creation time. The error occurs when the method is called, because Python tries to pass the class as the first argument but the function takes none:

```python
class Broken:
    @classmethod
    def broken() -> None:   # missing cls!
        print("This will fail")

Broken.broken()
# TypeError: Broken.broken() takes 0 positional arguments but 1 was given
```

This is a common typo that IDEs and type checkers (mypy) will flag. Always name the first parameter `cls` by convention (it's not enforced by Python, but it is universally expected).

**Q10: Is there a performance difference between instance methods, classmethods, and staticmethods?**

**A:** There is a marginal performance difference, but it is rarely significant in practice. A staticmethod call is slightly cheaper than a classmethod call because no binding occurs — the function is called directly. An instance method involves binding (creating a bound method object). However, Python 3.8+ caches bound methods in some cases, and modern CPython has optimised these paths.

```python
import timeit

class Foo:
    def instance(self) -> None: pass
    @classmethod
    def class_m(cls) -> None: pass
    @staticmethod
    def static_m() -> None: pass

f = Foo()
# Timeit comparison (results vary by machine):
# instance method: ~70-80ns
# classmethod:     ~80-90ns (extra descriptor work)
# staticmethod:    ~60-70ns (no binding)
```

Choose based on design correctness, not micro-optimisation.

---

## 3. Scenario-Based Interview Questions (10)

**S1: You are building a `Date` class and want to support construction from a string like `"2024-01-15"` and from a timestamp integer. Implement factory classmethods.**

**Q:** Design the constructors.

**A:**

```python
from __future__ import annotations
from datetime import datetime, timezone


class Date:
    def __init__(self, year: int, month: int, day: int) -> None:
        self.year = year
        self.month = month
        self.day = day

    @classmethod
    def from_string(cls, date_str: str) -> "Date":
        """Parse 'YYYY-MM-DD' format."""
        parts = date_str.split("-")
        if len(parts) != 3:
            raise ValueError(f"Expected YYYY-MM-DD, got {date_str!r}")
        return cls(int(parts[0]), int(parts[1]), int(parts[2]))

    @classmethod
    def from_timestamp(cls, ts: float) -> "Date":
        """Construct from a Unix timestamp."""
        dt = datetime.fromtimestamp(ts, tz=timezone.utc)
        return cls(dt.year, dt.month, dt.day)

    @classmethod
    def today(cls) -> "Date":
        now = datetime.now()
        return cls(now.year, now.month, now.day)

    def __repr__(self) -> str:
        return f"{type(self).__name__}({self.year:04d}-{self.month:02d}-{self.day:02d})"


class BusinessDate(Date):
    """Subclass — factories return BusinessDate, not Date."""
    def is_weekday(self) -> bool:
        import calendar
        return calendar.weekday(self.year, self.month, self.day) < 5


d = Date.from_string("2024-01-15")
bd = BusinessDate.from_string("2024-01-15")
print(type(d))    # <class 'Date'>
print(type(bd))   # <class 'BusinessDate'>
print(bd.is_weekday())  # True
```

**S2: A `Registry` class should track all subclasses automatically. Use a classmethod to manage the registry.**

**Q:** Implement an auto-registration system.

**A:**

```python
from __future__ import annotations

class Handler:
    _handlers: dict[str, type["Handler"]] = {}

    def __init_subclass__(cls, handler_type: str = "", **kwargs) -> None:
        super().__init_subclass__(**kwargs)
        key = handler_type or cls.__name__
        Handler._handlers[key] = cls
        print(f"Registered handler: {key!r}")

    @classmethod
    def create(cls, handler_type: str, **config) -> "Handler":
        try:
            handler_cls = cls._handlers[handler_type]
        except KeyError:
            available = list(cls._handlers.keys())
            raise ValueError(f"Unknown handler {handler_type!r}. Available: {available}")
        return handler_cls(**config)

    @classmethod
    def available(cls) -> list[str]:
        return list(cls._handlers.keys())

    def handle(self, event: dict) -> None:
        raise NotImplementedError


class FileHandler(Handler, handler_type="file"):
    def __init__(self, path: str = "/var/log/app.log") -> None:
        self.path = path
    def handle(self, event: dict) -> None:
        print(f"Writing to {self.path}: {event}")


class HTTPHandler(Handler, handler_type="http"):
    def __init__(self, url: str = "https://logs.example.com") -> None:
        self.url = url
    def handle(self, event: dict) -> None:
        print(f"Posting to {self.url}: {event}")


h = Handler.create("file", path="/tmp/debug.log")
h.handle({"level": "ERROR", "msg": "Oops"})
print(Handler.available())  # ['file', 'http']
```

**S3: A `Matrix` class needs a `@classmethod` for creating identity matrices and a `@staticmethod` for validating matrix dimensions.**

**Q:** Implement both types of methods appropriately.

**A:**

```python
from __future__ import annotations

class Matrix:
    def __init__(self, data: list[list[float]]) -> None:
        Matrix.validate_shape(data)  # static call for validation
        self._data = data
        self.rows = len(data)
        self.cols = len(data[0])

    @staticmethod
    def validate_shape(data: list[list[float]]) -> None:
        """Pure validation — no class or instance state needed."""
        if not data or not data[0]:
            raise ValueError("Matrix cannot be empty")
        row_len = len(data[0])
        for i, row in enumerate(data):
            if len(row) != row_len:
                raise ValueError(
                    f"Row {i} has {len(row)} elements, expected {row_len}"
                )

    @classmethod
    def identity(cls, size: int) -> "Matrix":
        """Factory: creates an identity matrix of given size."""
        data = [[1.0 if i == j else 0.0 for j in range(size)] for i in range(size)]
        return cls(data)

    @classmethod
    def zeros(cls, rows: int, cols: int) -> "Matrix":
        return cls([[0.0] * cols for _ in range(rows)])

    @classmethod
    def from_flat(cls, flat: list[float], rows: int, cols: int) -> "Matrix":
        if len(flat) != rows * cols:
            raise ValueError(f"Expected {rows*cols} elements, got {len(flat)}")
        return cls([flat[i*cols:(i+1)*cols] for i in range(rows)])

    def __repr__(self) -> str:
        return f"{type(self).__name__}({self._data})"


I = Matrix.identity(3)
print(I)
Z = Matrix.zeros(2, 3)
print(Z)
Matrix.validate_shape([[1,2],[3]])   # ValueError: row mismatch
```

**S4: A team argues whether `validate_email` should be a `@staticmethod` on the `User` class or a module-level function. How do you decide?**

**Q:** Make the design decision.

**A:**

```python
import re

# Module-level: appropriate if used by multiple unrelated classes
_EMAIL_PATTERN = re.compile(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')

def validate_email(email: str) -> bool:
    return bool(_EMAIL_PATTERN.match(email))


# @staticmethod: appropriate when only User validates emails AND
# you want to allow overriding in subclasses

class User:
    @staticmethod
    def validate_email(email: str) -> bool:
        return bool(_EMAIL_PATTERN.match(email))

    def __init__(self, email: str) -> None:
        if not self.validate_email(email):
            raise ValueError(f"Invalid email: {email!r}")
        self.email = email


class CorporateUser(User):
    """Corporate users must use company domain."""
    @staticmethod
    def validate_email(email: str) -> bool:
        return User.validate_email(email) and email.endswith("@corp.example.com")


# Calling validate_email via self allows polymorphism:
try:
    cu = CorporateUser("alice@gmail.com")   # ValidationError — wrong domain
except ValueError as e:
    print(e)
```

Decision: use `@staticmethod` if subclasses might need to specialise validation; use module-level if validation is truly universal.

**S5: You need a Singleton class. Show how `@classmethod` is used to implement it, and discuss the trade-offs.**

**Q:** Implement singleton via classmethod.

**A:**

```python
from __future__ import annotations
from threading import Lock

class Singleton:
    _instance: "Singleton | None" = None
    _lock: Lock = Lock()

    def __new__(cls) -> "Singleton":
        if cls._instance is None:
            with cls._lock:
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance

    @classmethod
    def get_instance(cls) -> "Singleton":
        """Alternative factory accessor — makes intent explicit."""
        return cls()

    @classmethod
    def reset(cls) -> None:
        """Useful for testing — reset singleton state."""
        with cls._lock:
            cls._instance = None


s1 = Singleton.get_instance()
s2 = Singleton.get_instance()
print(s1 is s2)  # True

# Trade-offs:
# + Thread-safe via double-checked locking
# - Global state — hard to test without reset()
# - Inheritance breaks singleton guarantees unless carefully managed
# - Consider: dependency injection is usually a better alternative
```

**S6: A CLI tool has a `Command` base class. Each subcommand should register itself. Use `@classmethod` and `__init_subclass__` to implement this.**

**Q:** Auto-register subcommands.

**A:**

```python
from __future__ import annotations
import sys

class Command:
    _commands: dict[str, type["Command"]] = {}
    name: str = ""
    description: str = ""

    def __init_subclass__(cls, **kwargs) -> None:
        super().__init_subclass__(**kwargs)
        if cls.name:
            Command._commands[cls.name] = cls

    @classmethod
    def dispatch(cls, argv: list[str]) -> int:
        if not argv or argv[0] not in cls._commands:
            print(f"Available commands: {', '.join(cls._commands)}")
            return 1
        cmd_cls = cls._commands[argv[0]]
        return cmd_cls().run(argv[1:])

    def run(self, args: list[str]) -> int:
        raise NotImplementedError


class BuildCommand(Command):
    name = "build"
    description = "Build the project"

    def run(self, args: list[str]) -> int:
        print(f"Building with args: {args}")
        return 0


class TestCommand(Command):
    name = "test"
    description = "Run tests"

    def run(self, args: list[str]) -> int:
        print(f"Testing with args: {args}")
        return 0


exit_code = Command.dispatch(["build", "--release"])
print(f"Exit: {exit_code}")
Command.dispatch(["unknown"])   # Lists available commands
```

**S7: Explain why this code is buggy and fix it.**

```python
class Shape:
    @classmethod
    def create_pair(cls) -> tuple:
        return Shape(), Shape()   # Bug!
```

**Q:** What is the bug and how do you fix it?

**A:**

```python
# BUG: Using Shape() hardcodes the return type.
# If Circle.create_pair() is called, you get (Shape(), Shape()), not (Circle(), Circle())

class Shape:
    @classmethod
    def create_pair(cls) -> tuple["Shape", "Shape"]:
        return cls(), cls()   # FIX: use cls() for polymorphism

class Circle(Shape):
    def __repr__(self) -> str:
        return "Circle()"

class Square(Shape):
    def __repr__(self) -> str:
        return "Square()"

print(Shape.create_pair())    # (Shape(), Shape()) — if Shape has __repr__
print(Circle.create_pair())   # (Circle(), Circle()) — correct with fix
print(Square.create_pair())   # (Square(), Square()) — correct with fix
```

**S8: You want to unit-test a classmethod that creates database connections. How do you make it testable?**

**Q:** Design for testability.

**A:**

```python
from typing import Protocol
from unittest.mock import MagicMock

class Connection(Protocol):
    def execute(self, sql: str) -> list: ...
    def close(self) -> None: ...


class Repository:
    _connection_factory = None   # injectable at class level

    @classmethod
    def set_connection_factory(cls, factory) -> None:
        cls._connection_factory = factory

    @classmethod
    def create_connection(cls) -> Connection:
        if cls._connection_factory:
            return cls._connection_factory()
        import sqlite3
        return sqlite3.connect(":memory:")  # real default

    def __init__(self) -> None:
        self._conn = self.create_connection()

    def find_all(self) -> list:
        return self._conn.execute("SELECT * FROM items")


# In tests:
mock_conn = MagicMock()
mock_conn.execute.return_value = [{"id": 1, "name": "test"}]

Repository.set_connection_factory(lambda: mock_conn)
repo = Repository()
results = repo.find_all()
print(results)  # [{'id': 1, 'name': 'test'}]
Repository.set_connection_factory(None)  # reset
```

**S9: A data pipeline uses `@staticmethod` for transformation functions. When should these be moved to classmethods?**

**Q:** Discuss the evolution.

**A:**

```python
class Pipeline:
    @staticmethod
    def normalise(value: float, min_val: float, max_val: float) -> float:
        """Pure function — no class state needed. staticmethod is correct."""
        return (value - min_val) / (max_val - min_val)

    @staticmethod
    def clip(value: float, low: float = 0.0, high: float = 1.0) -> float:
        return max(low, min(high, value))


# Evolving requirement: different Pipeline configurations need different defaults
class NormalisedPipeline(Pipeline):
    clip_low: float = 0.0
    clip_high: float = 1.0

    @classmethod
    def clip(cls, value: float, low: float | None = None, high: float | None = None) -> float:
        # Now needs class-level defaults → must become classmethod
        low = low if low is not None else cls.clip_low
        high = high if high is not None else cls.clip_high
        return max(low, min(high, value))

class StrictPipeline(NormalisedPipeline):
    clip_low = 0.1
    clip_high = 0.9

print(NormalisedPipeline.clip(1.5))   # 1.0
print(StrictPipeline.clip(1.5))       # 0.9 — uses subclass defaults
```

Move from `@staticmethod` to `@classmethod` when you need access to class-level configuration or subclass customisation.

**S10: Show how `classmethod` enables an "abstract class method" pattern before Python 3.3.**

**Q:** Implement abstract classmethod.

**A:**

```python
from abc import ABC, abstractmethod

class Serialiser(ABC):
    @classmethod
    @abstractmethod
    def from_bytes(cls, data: bytes) -> "Serialiser":
        """Abstract factory — subclasses MUST implement."""
        ...

    @abstractmethod
    def to_bytes(self) -> bytes:
        ...


class JSONSerialiser(Serialiser):
    def __init__(self, data: dict) -> None:
        self._data = data

    @classmethod
    def from_bytes(cls, data: bytes) -> "JSONSerialiser":
        import json
        return cls(json.loads(data))

    def to_bytes(self) -> bytes:
        import json
        return json.dumps(self._data).encode()


# Cannot instantiate Serialiser (abstract):
# Serialiser.from_bytes(b"")  # TypeError

s = JSONSerialiser.from_bytes(b'{"key": "value"}')
print(s.to_bytes())  # b'{"key": "value"}'
```

Note: `@classmethod @abstractmethod` is the correct order (since Python 3.3). The `@abstractmethod` must be the innermost decorator.

---

## 4. Quick Recap / Cheatsheet

- **`@classmethod`**: first arg is `cls` (the class). Defined via descriptor that binds to `type(obj)` on access.
- **`@staticmethod`**: no automatic binding — returns the raw function unchanged.
- **Factory pattern**: always use `cls()` inside `@classmethod`, never `ClassName()` — enables polymorphism in subclasses.
- **Instance calls classmethod**: `obj.classmethod()` still receives the class, not the instance.
- **Descriptor internals**: `classmethod.__get__` returns a partial bound to `cls`; `staticmethod.__get__` returns the raw function.
- **`__func__`**: access the underlying function via `Foo.__dict__['cm'].__func__` for both classmethod and staticmethod.
- **Override in subclass**: both can be overridden. Classmethod overrides remain class-polymorphic; staticmethod overrides are independent.
- **Abstract classmethod**: use `@classmethod` + `@abstractmethod` (in that order, abstractmethod innermost).
- **staticmethod vs module function**: prefer `@staticmethod` if it logically belongs to the class; module-level if genuinely general.
- **When classmethod**: factory construction, class-level state management, plugin registries, inheritance-aware operations.
- **When staticmethod**: utilities tightly coupled to the class concept but requiring no class/instance state.
- **`super()` in classmethod**: works as expected; `cls` is still the original (most-derived) class throughout the chain.
- **No `self`/`cls` → no access**: `@staticmethod` cannot access class or instance state — pass everything explicitly.
