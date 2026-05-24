# Metaclasses (Introduction)

## 1. Concept (Deep Explanation)

### A Metaclass Is "A Class of a Class"

In Python, everything is an object — including classes. A **metaclass** is the class of a class: it controls how classes are created, just as a class controls how instances are created.

```python
# Every object has a type:
x = 42
print(type(x))        # <class 'int'>
print(type(int))      # <class 'type'>   — int's metaclass is 'type'
print(type(str))      # <class 'type'>
print(type(type))     # <class 'type'>   — type is its own metaclass!

class MyClass:
    pass

print(type(MyClass))  # <class 'type'>   — default metaclass
```

The metaclass chain:
```
instance → class (type of instance) → metaclass (type of class)
42        → int                      → type
MyClass() → MyClass                  → type
```

When you write `class Foo:`, Python calls `type('Foo', (object,), namespace_dict)` to create the class object. By providing a custom metaclass, you intercept this process.

### `type` Is the Default Metaclass

`type` serves a dual role:
1. As a **function**: `type(obj)` returns the type of `obj`
2. As a **metaclass**: `type(name, bases, namespace)` creates a new class dynamically

```python
# Dynamic class creation — equivalent to writing a class statement:
MyDynamic = type(
    'MyDynamic',              # class name
    (object,),                # base classes tuple
    {                         # class namespace / attributes
        'x': 42,
        'greet': lambda self: f"Hello from {type(self).__name__}",
    }
)

obj = MyDynamic()
print(obj.greet())    # "Hello from MyDynamic"
print(MyDynamic.x)    # 42
print(type(MyDynamic))  # <class 'type'>

# This is exactly what Python does internally for every class statement!
```

### How Class Creation Works Step by Step

When Python encounters a `class` statement:

1. **Determine the metaclass**: look for `metaclass=` keyword; if absent, inherit from bases; if still absent, use `type`
2. **Call `metaclass.__prepare__(name, bases, **kwargs)`**: returns the namespace dict (usually an empty `dict`, but can be an `OrderedDict` or custom mapping)
3. **Execute the class body** in the returned namespace
4. **Call `metaclass(name, bases, namespace, **kwargs)`**: this is `metaclass.__new__` + `metaclass.__init__`

```python
class Meta(type):
    @classmethod
    def __prepare__(mcs, name: str, bases: tuple, **kwargs) -> dict:
        print(f"  __prepare__({name!r})")
        return {}  # could return an OrderedDict, custom dict, etc.

    def __new__(mcs, name: str, bases: tuple, namespace: dict, **kwargs):
        print(f"  __new__({name!r})")
        cls = super().__new__(mcs, name, bases, namespace)
        return cls

    def __init__(cls, name: str, bases: tuple, namespace: dict, **kwargs) -> None:
        print(f"  __init__({name!r})")
        super().__init__(name, bases, namespace)

print("Creating class...")
class Widget(metaclass=Meta):
    x = 42

# Output:
#   __prepare__('Widget')
#   __new__('Widget')
#   __init__('Widget')
```

### Custom Metaclass: `__new__` and `__init__`

```python
from typing import Any

class SingletonMeta(type):
    """Metaclass that makes every class a singleton."""
    _instances: dict[type, Any] = {}

    def __call__(cls, *args, **kwargs):
        """__call__ on the metaclass is invoked when you call the CLASS (ClassName())."""
        if cls not in cls._instances:
            # First time: create normally
            instance = super().__call__(*args, **kwargs)
            cls._instances[cls] = instance
        return cls._instances[cls]


class Database(metaclass=SingletonMeta):
    def __init__(self, url: str = "sqlite:///:memory:") -> None:
        self.url = url
        print(f"Database initialised with {url}")


db1 = Database("postgresql://localhost/mydb")
db2 = Database()
print(db1 is db2)   # True — same instance!
```

### Use Cases: ORMs, Frameworks, Singletons, Auto-Registration

**ORM-style metaclass (Django-like model field registration):**

```python
class FieldDescriptor:
    def __init__(self, field_type: type) -> None:
        self.field_type = field_type
        self.name: str = ""

    def __set_name__(self, owner: type, name: str) -> None:
        self.name = name

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return obj.__dict__.get(self.name)

    def __set__(self, obj, value) -> None:
        if not isinstance(value, self.field_type):
            raise TypeError(
                f"Field {self.name!r} must be {self.field_type.__name__}, "
                f"got {type(value).__name__}"
            )
        obj.__dict__[self.name] = value


class ModelMeta(type):
    """Metaclass that collects Field descriptors and generates __init__."""

    def __new__(mcs, name: str, bases: tuple, namespace: dict):
        fields: dict[str, FieldDescriptor] = {}

        # Collect field descriptors from the namespace:
        for key, value in namespace.items():
            if isinstance(value, FieldDescriptor):
                fields[key] = value

        namespace['_fields'] = fields
        cls = super().__new__(mcs, name, bases, namespace)
        return cls


class Model(metaclass=ModelMeta):
    def __init__(self, **kwargs) -> None:
        for name, field in self._fields.items():
            value = kwargs.get(name)
            if value is not None:
                setattr(self, name, value)

    def __repr__(self) -> str:
        fields = ", ".join(
            f"{name}={getattr(self, name, None)!r}"
            for name in self._fields
        )
        return f"{type(self).__name__}({fields})"


class User(Model):
    name = FieldDescriptor(str)
    age = FieldDescriptor(int)
    email = FieldDescriptor(str)


u = User(name="Alice", age=30, email="alice@example.com")
print(u)           # User(name='Alice', age=30, email='alice@example.com')
u.age = "thirty"   # TypeError: Field 'age' must be int, got str
```

### `__prepare__`: Returning a Custom Namespace

`__prepare__` is called before the class body is executed. It returns the dict-like object that the class body will write into. This enables powerful use cases:

```python
class OrderedMeta(type):
    """Preserves method definition order (pre-Python 3.7 style; now default)."""

    @classmethod
    def __prepare__(mcs, name: str, bases: tuple, **kwargs) -> dict:
        from collections import OrderedDict
        return OrderedDict()  # preserves insertion order

    def __new__(mcs, name: str, bases: tuple, namespace, **kwargs):
        cls = super().__new__(mcs, name, bases, dict(namespace))
        cls._method_order = list(namespace.keys())
        return cls


class MyAPI(metaclass=OrderedMeta):
    def get(self) -> None: pass
    def post(self) -> None: pass
    def put(self) -> None: pass
    def delete(self) -> None: pass


print(MyAPI._method_order)
# ['__module__', '__qualname__', 'get', 'post', 'put', 'delete']
```

### Class Decorators as a Lighter Alternative

For most metaclass use cases, a **class decorator** is simpler, more readable, and sufficient:

```python
from functools import wraps

def auto_repr(cls):
    """Add a __repr__ based on __init__ parameters — no metaclass needed."""
    import inspect
    params = list(inspect.signature(cls.__init__).parameters.keys())[1:]  # skip self

    def __repr__(self) -> str:
        attrs = ", ".join(f"{p}={getattr(self, p, '?')!r}" for p in params)
        return f"{type(self).__name__}({attrs})"

    cls.__repr__ = __repr__
    return cls


@auto_repr
class Point:
    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

print(Point(1.0, 2.0))   # Point(x=1.0, y=2.0)
```

Class decorators run **after** the class is created (unlike metaclasses which run **during** creation). This is why metaclasses are needed for things like modifying the class namespace before the body executes (`__prepare__`), or controlling which base classes are used.

### `__init_subclass__`: Even Lighter Than Metaclasses

Added in Python 3.6, `__init_subclass__` is called on the parent class whenever a new subclass is created. It replaces many metaclass use cases elegantly:

```python
class PluginBase:
    _plugins: dict[str, type] = {}

    def __init_subclass__(cls, /, plugin_name: str = "", **kwargs) -> None:
        super().__init_subclass__(**kwargs)
        name = plugin_name or cls.__name__.lower()
        PluginBase._plugins[name] = cls
        print(f"Registered plugin: {name!r}")

    @classmethod
    def get(cls, name: str) -> "PluginBase":
        return cls._plugins[name]()


class CSVPlugin(PluginBase, plugin_name="csv"):
    def process(self) -> str:
        return "CSV processing"

class JSONPlugin(PluginBase, plugin_name="json"):
    def process(self) -> str:
        return "JSON processing"


plugin = PluginBase.get("csv")
print(plugin.process())   # "CSV processing"
```

### When NOT to Use Metaclasses

Python's Zen: "Simple is better than complex." Metaclasses are complex. Before reaching for a metaclass, ask:

1. Can a **class decorator** do this? (Usually yes — prefer it)
2. Can `__init_subclass__` do this? (Often yes — even cleaner)
3. Can `__set_name__` (descriptor) do this? (For attribute-level customisation)
4. Is this actually needed at class-creation time, or can it be deferred?

Metaclasses are appropriate for:
- Frameworks that need to modify class creation in ways decorators can't (e.g., `__prepare__`)
- Enforcing interface contracts across an entire class hierarchy automatically
- ORMs where the metaclass needs to inspect annotations/descriptors before the class exists
- Abstract Base Class machinery (`ABCMeta`)

Do NOT use metaclasses for:
- Simple attribute injection → use class decorator
- Plugin registration → use `__init_subclass__`
- Singleton pattern → use `__new__` or a class decorator
- Validation that can run at `__init__` time

### Metaclass Conflict Resolution

When two metaclasses need to coexist (e.g., using ABCMeta + a custom metaclass):

```python
from abc import ABCMeta, abstractmethod

class MyMeta(type):
    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        print(f"MyMeta created {name}")
        return cls

# Conflict: both ABCMeta and MyMeta want to be metaclass
class CombinedMeta(MyMeta, ABCMeta):
    """Resolve conflict by explicitly combining metaclasses."""
    pass

class AbstractBase(metaclass=CombinedMeta):
    @abstractmethod
    def process(self) -> str: ...

class Concrete(AbstractBase):
    def process(self) -> str:
        return "processed"

c = Concrete()
print(c.process())
```

### Pitfalls and Gotchas

- **Metaclass conflict**: if two parent classes have incompatible metaclasses, Python raises `TypeError`
- **`__prepare__` must be a classmethod** (explicitly, since metaclass `__prepare__` is `classmethod`-like by default in practice — but be careful)
- **Performance**: metaclass `__new__` runs only at class creation, not instance creation — no runtime cost
- **Debugging complexity**: stack traces through metaclass machinery are hard to read
- **`type(cls)` returns the metaclass** — be aware when doing introspection
- **`isinstance` and `issubclass` work correctly** with custom metaclasses because they check the full MRO

---

## 2. Conceptual Interview Questions (10)

**Q1: What is a metaclass and how does it differ from a class?**

**A:** A class defines how instances are created (via `__init__`) and what attributes/methods they have. A metaclass defines how **classes** are created and what attributes/methods the class objects themselves have.

In Python's type system, every object is an instance of its type. Classes are objects, so they too are instances of their type — the metaclass. The default metaclass is `type`. When you define a class, `type.__new__` is called to allocate and configure the class object.

```python
class Foo:
    bar = 42

# Foo is an instance of type:
print(isinstance(Foo, type))   # True
print(type(Foo))               # <class 'type'>

# A custom metaclass makes classes into instances of that metaclass:
class Meta(type):
    pass

class FooWithMeta(metaclass=Meta):
    pass

print(type(FooWithMeta))       # <class '__main__.Meta'>
print(isinstance(FooWithMeta, type))  # True — Meta inherits from type
```

**Q2: Explain the class creation sequence in detail.**

**A:** When Python encounters `class Foo(Base, metaclass=Meta):`, it:

1. **Finds the metaclass**: checks `metaclass=` keyword, then `type(Base)`, then defaults to `type`
2. **Calls `Meta.__prepare__(name, bases, **kwargs)`**: gets the namespace dict
3. **Executes the class body** in that namespace (all `def` and assignments go into it)
4. **Calls `Meta.__new__(Meta, name, bases, namespace, **kwargs)`**: creates the class object
5. **Calls `Meta.__init__(cls, name, bases, namespace, **kwargs)`**: initialises the class object
6. **Runs `__set_name__`** on all descriptors in the namespace
7. **Calls `Base.__init_subclass__(cls, **kwargs)`** on all bases

```python
class TraceMeta(type):
    @classmethod
    def __prepare__(mcs, name, bases, **kw):
        print(f"1. __prepare__({name!r})")
        return super().__prepare__(name, bases, **kw)

    def __new__(mcs, name, bases, ns, **kw):
        print(f"2. __new__({name!r})")
        return super().__new__(mcs, name, bases, ns)

    def __init__(cls, name, bases, ns, **kw):
        print(f"3. __init__({name!r})")
        super().__init__(name, bases, ns)


class MyClass(metaclass=TraceMeta):
    x = 42
    def hello(self): pass
```

**Q3: What is `__prepare__` and what are its use cases?**

**A:** `__prepare__` is a classmethod on the metaclass called before the class body is executed. It returns the mapping (dict-like object) into which class body definitions will be written.

Use cases:
1. **Ordered namespace** (pre-Python 3.7) — return an `OrderedDict` to preserve definition order
2. **Validation namespace** — return a custom dict that validates or transforms definitions as they're added
3. **Namespace injection** — pre-populate the namespace with constants or helper functions available inside the class body

```python
class ValidatedNamespace(dict):
    """Rejects duplicate method definitions."""
    def __setitem__(self, key: str, value) -> None:
        if key in self and callable(value):
            raise ValueError(f"Method {key!r} already defined!")
        super().__setitem__(key, value)

class StrictMeta(type):
    @classmethod
    def __prepare__(mcs, name, bases, **kwargs):
        return ValidatedNamespace()

class Strict(metaclass=StrictMeta):
    def hello(self) -> str:
        return "hello"
    # def hello(self) -> str:   # ValueError: Method 'hello' already defined!
    #     return "oops"
```

**Q4: How do `__init_subclass__` and class decorators compare to metaclasses?**

**A:** All three can modify classes, but at different points and with different capabilities:

| Feature | Metaclass | Class Decorator | `__init_subclass__` |
|---|---|---|---|
| When it runs | During class creation | After class creation | When subclass is defined |
| Can modify namespace pre-execution | ✓ (via `__prepare__`) | ✗ | ✗ |
| Can change base classes | ✓ | ✗ | ✗ |
| Affects all subclasses | ✓ | ✗ (apply manually) | ✓ |
| Complexity | High | Low | Medium |

Prefer in order: `__init_subclass__` → class decorator → metaclass.

```python
# __init_subclass__ — simplest for plugin registration:
class Plugin:
    _registry = {}
    def __init_subclass__(cls, name="", **kw):
        super().__init_subclass__(**kw)
        Plugin._registry[name or cls.__name__] = cls

# Class decorator — clean for adding methods/properties:
def singleton(cls):
    instances = {}
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance
```

**Q5: How does `ABCMeta` use the metaclass mechanism?**

**A:** `ABCMeta` is the metaclass behind `ABC`. It tracks abstract methods via `__abstractmethods__` — a frozenset of method names that must be implemented. At class creation (`__new__`), `ABCMeta` scans the namespace for `@abstractmethod`-decorated functions and sets `cls.__abstractmethods__`.

When `cls()` is called, `type.__call__` checks if `cls.__abstractmethods__` is non-empty and raises `TypeError` if it is.

```python
from abc import ABCMeta, abstractmethod, ABC

class Shape(ABC):   # ABC is a class with metaclass=ABCMeta
    @abstractmethod
    def area(self) -> float: ...

print(Shape.__abstractmethods__)   # frozenset({'area'})

try:
    Shape()   # TypeError: Can't instantiate abstract class Shape with abstract method area
except TypeError as e:
    print(e)

class Circle(Shape):
    def __init__(self, r: float) -> None:
        self.r = r
    def area(self) -> float:
        import math
        return math.pi * self.r ** 2

print(Circle.__abstractmethods__)  # frozenset() — all implemented
c = Circle(5)  # Works!
```

**Q6: Explain the metaclass conflict and how to resolve it.**

**A:** A metaclass conflict occurs when a class has multiple bases whose metaclasses are not in a compatible relationship (one must be a subclass of the other). Python raises `TypeError: metaclass conflict`.

```python
class Meta1(type): pass
class Meta2(type): pass

class Base1(metaclass=Meta1): pass
class Base2(metaclass=Meta2): pass

# class Broken(Base1, Base2): pass  # TypeError: metaclass conflict

# Resolution: create a combined metaclass that inherits from both:
class CombinedMeta(Meta1, Meta2): pass

class Fixed(Base1, Base2, metaclass=CombinedMeta): pass
print(type(Fixed))  # <class '__main__.CombinedMeta'>
```

The rule: the metaclass of a derived class must be a subtype of the metaclasses of all its bases. If not, Python cannot determine which metaclass to use.

**Q7: Can you add methods to a class dynamically via its metaclass?**

**A:** Yes. The metaclass `__new__` or `__init__` runs at class creation and can inject any attribute into the class:

```python
class AutoPropertyMeta(type):
    """Automatically creates properties for all private attributes."""

    def __new__(mcs, name, bases, namespace):
        # Find all _private attributes set in __init__
        # (In a real ORM, you'd look at annotations or descriptors)
        auto_props = {
            k.lstrip('_'): k
            for k in namespace
            if k.startswith('_') and not k.startswith('__')
        }

        for public, private in auto_props.items():
            def make_prop(priv: str):
                return property(
                    fget=lambda self: getattr(self, priv),
                    fset=lambda self, v: setattr(self, priv, v),
                )
            namespace[public] = make_prop(private)

        return super().__new__(mcs, name, bases, namespace)


class Config(metaclass=AutoPropertyMeta):
    _host: str = "localhost"
    _port: int = 8080

    def __init__(self, host: str, port: int) -> None:
        self._host = host
        self._port = port


cfg = Config("example.com", 443)
print(cfg.host)   # "example.com" — auto-generated property!
print(cfg.port)   # 443
```

**Q8: How does `type.__call__` work and how do metaclasses interact with instance creation?**

**A:** When you call `MyClass(args)`, Python invokes `type(MyClass).__call__(MyClass, args)`. Since `type(MyClass)` is the metaclass (usually `type`), `type.__call__` runs. It:

1. Calls `MyClass.__new__(MyClass, args)` → creates and returns the instance
2. If the result is an instance of `MyClass`, calls `instance.__init__(args)`
3. Returns the instance

By overriding `__call__` in the metaclass, you can intercept **instance creation**:

```python
class CachedMeta(type):
    _caches: dict = {}

    def __call__(cls, *args, **kwargs):
        key = (cls, args, tuple(sorted(kwargs.items())))
        if key not in CachedMeta._caches:
            CachedMeta._caches[key] = super().__call__(*args, **kwargs)
        return CachedMeta._caches[key]


class Connection(metaclass=CachedMeta):
    def __init__(self, host: str, port: int) -> None:
        self.host = host
        self.port = port

c1 = Connection("localhost", 5432)
c2 = Connection("localhost", 5432)
print(c1 is c2)   # True — cached!
```

**Q9: What is the relationship between `type.__new__` and `type.__init__` in class creation?**

**A:**

- `type.__new__(mcs, name, bases, namespace)`: allocates the class object in memory, sets `__dict__`, processes `__slots__`, resolves the MRO, installs descriptors. Returns the new class object.
- `type.__init__(cls, name, bases, namespace)`: additional initialisation. In the default `type`, this is mostly a no-op (the real work is in `__new__`).

In custom metaclasses, it's conventional to do structural changes (modifying namespace, adding methods) in `__new__`, and post-creation configuration in `__init__`.

```python
class Meta(type):
    def __new__(mcs, name, bases, namespace):
        # Add a class-level constant to every class:
        namespace['CREATED_BY'] = 'Meta'
        return super().__new__(mcs, name, bases, namespace)

    def __init__(cls, name, bases, namespace) -> None:
        # Register the class in a global registry:
        super().__init__(name, bases, namespace)
        print(f"Registered {name!r}")

class Foo(metaclass=Meta): pass
print(Foo.CREATED_BY)   # 'Meta'
```

**Q10: How do you check if a class uses a specific metaclass?**

**A:** Use `type()` or `isinstance()` with the metaclass:

```python
class CustomMeta(type): pass
class MyClass(metaclass=CustomMeta): pass

print(type(MyClass))                    # <class '__main__.CustomMeta'>
print(type(MyClass) is CustomMeta)      # True
print(isinstance(MyClass, CustomMeta))  # True — MyClass is an instance of its metaclass
print(isinstance(MyClass, type))        # True — CustomMeta inherits from type

# Alternatively, check __class__:
print(MyClass.__class__)   # <class '__main__.CustomMeta'>
```

---

## 3. Scenario-Based Interview Questions (10)

**S1: You need to enforce that all methods in a class hierarchy have type annotations. Use a metaclass.**

**Q:** Implement annotation enforcement.

**A:**

```python
import inspect
import typing

class AnnotatedMeta(type):
    """Ensures all non-dunder methods have return type annotations."""

    def __new__(mcs, name: str, bases: tuple, namespace: dict):
        for attr_name, value in namespace.items():
            if attr_name.startswith('__'):
                continue
            if callable(value) and not isinstance(value, (classmethod, staticmethod)):
                hints = typing.get_type_hints(value)
                if 'return' not in hints:
                    raise TypeError(
                        f"{name}.{attr_name} is missing a return type annotation"
                    )
        return super().__new__(mcs, name, bases, namespace)


class StrictAPI(metaclass=AnnotatedMeta):
    def get_user(self, user_id: int) -> dict:   # OK
        return {}

    # def bad_method(self):  # Would raise TypeError!
    #     pass
```

**S2: Django-style models use a metaclass to convert class-level field definitions into instance attributes. Show a simplified version.**

**Q:** Implement model metaclass.

**A:**

```python
from __future__ import annotations

class Field:
    def __init__(self, field_type: type, required: bool = True) -> None:
        self.field_type = field_type
        self.required = required
        self.name: str = ""

    def __set_name__(self, owner: type, name: str) -> None:
        self.name = name


class ModelMeta(type):
    def __new__(mcs, name: str, bases: tuple, namespace: dict):
        fields: dict[str, Field] = {}

        # Inherit fields from parent models:
        for base in bases:
            if hasattr(base, '_fields'):
                fields.update(base._fields)

        # Collect new fields:
        for key, val in namespace.items():
            if isinstance(val, Field):
                val.name = key
                fields[key] = val

        namespace['_fields'] = fields

        # Auto-generate __init__:
        def __init__(self, **kwargs):
            for fname, field in self._fields.items():
                val = kwargs.get(fname)
                if field.required and val is None:
                    raise ValueError(f"Field {fname!r} is required")
                if val is not None and not isinstance(val, field.field_type):
                    raise TypeError(
                        f"Field {fname!r} must be {field.field_type.__name__}"
                    )
                object.__setattr__(self, fname, val)

        namespace['__init__'] = __init__

        def __repr__(self) -> str:
            attrs = ", ".join(
                f"{k}={getattr(self, k, None)!r}" for k in self._fields
            )
            return f"{type(self).__name__}({attrs})"

        namespace['__repr__'] = __repr__

        return super().__new__(mcs, name, bases, namespace)


class Model(metaclass=ModelMeta):
    pass


class User(Model):
    name = Field(str)
    age = Field(int)
    email = Field(str, required=False)


u = User(name="Alice", age=30)
print(u)             # User(name='Alice', age=30, email=None)
u2 = User(name="Bob", age="thirty")  # TypeError
```

**S3: Implement a metaclass that adds automatic logging to every method call.**

**Q:** Auto-wrap all methods with logging.

**A:**

```python
import functools
import logging
from typing import Callable

logger = logging.getLogger(__name__)

def log_call(func: Callable) -> Callable:
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        cls_name = type(args[0]).__name__ if args else ""
        logger.debug("CALL %s.%s args=%r kwargs=%r", cls_name, func.__name__, args[1:], kwargs)
        try:
            result = func(*args, **kwargs)
            logger.debug("RETURN %s.%s -> %r", cls_name, func.__name__, result)
            return result
        except Exception as exc:
            logger.error("ERROR %s.%s: %s", cls_name, func.__name__, exc)
            raise
    return wrapper


class LoggingMeta(type):
    def __new__(mcs, name: str, bases: tuple, namespace: dict):
        for attr_name, value in namespace.items():
            if callable(value) and not attr_name.startswith('__'):
                namespace[attr_name] = log_call(value)
        return super().__new__(mcs, name, bases, namespace)


logging.basicConfig(level=logging.DEBUG)

class PaymentService(metaclass=LoggingMeta):
    def charge(self, amount: float, card: str) -> bool:
        return True

    def refund(self, transaction_id: str) -> dict:
        return {"status": "refunded", "id": transaction_id}


svc = PaymentService()
svc.charge(99.99, "4111111111111111")
svc.refund("txn-123")
```

**S4: A team wants to use metaclasses for an API rate limiter that applies to all methods of service classes.**

**Q:** Design the rate-limiting metaclass.

**A:**

```python
import time
import functools
from collections import deque

class RateLimitDescriptor:
    def __init__(self, func, calls: int, period: float) -> None:
        self.func = func
        self.calls = calls
        self.period = period
        self._timestamps: deque = deque()
        functools.update_wrapper(self, func)

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        @functools.wraps(self.func)
        def method(*args, **kwargs):
            now = time.monotonic()
            # Remove timestamps outside the window:
            while self._timestamps and now - self._timestamps[0] > self.period:
                self._timestamps.popleft()
            if len(self._timestamps) >= self.calls:
                raise RuntimeError(
                    f"Rate limit: {self.calls} calls per {self.period}s exceeded"
                )
            self._timestamps.append(now)
            return self.func(obj, *args, **kwargs)
        return method


def rate_limit(calls: int = 10, period: float = 1.0):
    """Decorator factory to mark a method for rate limiting."""
    def decorator(func):
        func._rate_limit = (calls, period)
        return func
    return decorator


class RateLimitedMeta(type):
    def __new__(mcs, name: str, bases: tuple, namespace: dict):
        for attr_name, value in list(namespace.items()):
            if callable(value) and hasattr(value, '_rate_limit'):
                calls, period = value._rate_limit
                namespace[attr_name] = RateLimitDescriptor(value, calls, period)
        return super().__new__(mcs, name, bases, namespace)


class ExternalAPI(metaclass=RateLimitedMeta):
    @rate_limit(calls=3, period=1.0)
    def fetch_data(self, endpoint: str) -> dict:
        return {"endpoint": endpoint, "data": "..."}


api = ExternalAPI()
for i in range(3):
    print(api.fetch_data(f"/items/{i}"))

try:
    api.fetch_data("/too_many")
except RuntimeError as e:
    print(f"Rate limited: {e}")
```

**S5: Explain why `__init_subclass__` is preferred over a metaclass for plugin registration.**

**Q:** Compare the two approaches.

**A:**

```python
# METACLASS APPROACH — more complex:
class PluginMeta(type):
    _registry: dict = {}

    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if bases:  # Skip the base class itself
            mcs._registry[name] = cls
        return cls

class PluginBase(metaclass=PluginMeta):
    pass

class CsvPlugin(PluginBase):  # auto-registered
    pass


# __init_subclass__ APPROACH — simpler and preferred:
class Plugin:
    _registry: dict[str, type] = {}

    def __init_subclass__(cls, /, plugin_type: str = "", **kwargs) -> None:
        super().__init_subclass__(**kwargs)
        key = plugin_type or cls.__name__.lower()
        Plugin._registry[key] = cls

    @classmethod
    def create(cls, plugin_type: str) -> "Plugin":
        return cls._registry[plugin_type]()


class JSONPlugin(Plugin, plugin_type="json"):
    def run(self) -> str:
        return "JSON"

class CSVPlugin(Plugin, plugin_type="csv"):
    def run(self) -> str:
        return "CSV"


print(Plugin._registry)   # {'json': JSONPlugin, 'csv': CSVPlugin}
print(Plugin.create("json").run())  # "JSON"

# __init_subclass__ wins because:
# 1. No metaclass complexity
# 2. No metaclass conflict risk when combining with ABCMeta or other metaclasses
# 3. Cleaner syntax with keyword arguments
# 4. Easier to understand and debug
```

**S6: Implement a metaclass that enforces the Singleton pattern for specific classes.**

**Q:** Metaclass-based singleton with thread safety.

**A:**

```python
import threading
from typing import Any

class ThreadSafeSingletonMeta(type):
    _instances: dict[type, Any] = {}
    _lock: threading.Lock = threading.Lock()

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            with cls._lock:
                if cls not in cls._instances:
                    instance = super().__call__(*args, **kwargs)
                    cls._instances[cls] = instance
        return cls._instances[cls]

    @classmethod
    def reset(mcs, cls: type) -> None:
        """For testing — clear the singleton cache."""
        with mcs._lock:
            mcs._instances.pop(cls, None)


class AppConfig(metaclass=ThreadSafeSingletonMeta):
    def __init__(self, env: str = "production") -> None:
        self.env = env
        self.settings: dict = {}
        print(f"AppConfig initialised for {env}")

    def get(self, key: str, default=None):
        return self.settings.get(key, default)


cfg1 = AppConfig("staging")
cfg2 = AppConfig("production")  # Same instance — "production" ignored!
print(cfg1 is cfg2)   # True
print(cfg1.env)       # "staging"

# In tests:
ThreadSafeSingletonMeta.reset(AppConfig)
cfg3 = AppConfig("test")
print(cfg3 is cfg1)   # False — fresh instance
```

**S7: Use `__prepare__` to implement a class where methods can be defined with SQL-style syntax.**

**Q:** Creative use of `__prepare__`.

**A:**

```python
class SQLNamespace(dict):
    """A namespace that accepts SQL-style method annotations."""

    def __init__(self) -> None:
        super().__init__()
        self._queries: dict[str, str] = {}

    def __setitem__(self, key: str, value) -> None:
        if isinstance(value, str) and key.upper() == key:
            # All-uppercase → treat as a SQL query constant
            self._queries[key] = value
        super().__setitem__(key, value)


class QueryMeta(type):
    @classmethod
    def __prepare__(mcs, name: str, bases: tuple, **kwargs) -> SQLNamespace:
        return SQLNamespace()

    def __new__(mcs, name: str, bases: tuple, namespace: SQLNamespace):
        # Copy queries to class:
        queries = getattr(namespace, '_queries', {})
        cls = super().__new__(mcs, name, bases, dict(namespace))
        cls._queries = queries
        return cls


class UserRepository(metaclass=QueryMeta):
    GET_BY_ID = "SELECT * FROM users WHERE id = :id"
    GET_ALL = "SELECT * FROM users ORDER BY name"
    DELETE = "DELETE FROM users WHERE id = :id"

    def find(self, query_name: str, **params) -> str:
        sql = self._queries.get(query_name, "")
        for k, v in params.items():
            sql = sql.replace(f":{k}", repr(v))
        return sql


repo = UserRepository()
print(repo.find("GET_BY_ID", id=42))
print(UserRepository._queries)
```

**S8: How would you implement a metaclass that validates class-level configuration at class creation time?**

**Q:** Enforce class-level invariants.

**A:**

```python
class ConfigurableMeta(type):
    """Validates required class-level configuration at class creation."""

    REQUIRED_ATTRS = ('table_name', 'primary_key')

    def __new__(mcs, name: str, bases: tuple, namespace: dict):
        cls = super().__new__(mcs, name, bases, namespace)

        # Skip validation for the base class itself:
        if not any(isinstance(b, mcs) for b in bases):
            return cls

        errors = []
        for attr in mcs.REQUIRED_ATTRS:
            if not hasattr(cls, attr):
                errors.append(f"Missing required class attribute: {attr!r}")

        if errors:
            raise TypeError(
                f"Class {name!r} is invalid:\n" + "\n".join(f"  - {e}" for e in errors)
            )
        return cls


class Model(metaclass=ConfigurableMeta):
    """Base model — validation is skipped for this class."""
    pass


class UserModel(Model):
    table_name = "users"
    primary_key = "id"

    def find(self, pk: int) -> dict:
        return {"id": pk}


try:
    class BrokenModel(Model):
        pass   # Missing table_name and primary_key!
except TypeError as e:
    print(e)
```

**S9: Explain how you would use metaclasses to implement an automatic `__repr__` and `__eq__` similar to what `@dataclass` does.**

**Q:** Implement dataclass-like metaclass.

**A:**

```python
import inspect

class DataMeta(type):
    """Generates __repr__, __eq__, and __init__ from type annotations."""

    def __new__(mcs, name: str, bases: tuple, namespace: dict):
        annotations: dict[str, type] = {}

        # Collect annotations from parent classes:
        for base in reversed(bases):
            annotations.update(getattr(base, '__annotations__', {}))
        annotations.update(namespace.get('__annotations__', {}))

        namespace['_data_fields'] = list(annotations.keys())

        # Generate __init__:
        def __init__(self, **kwargs):
            for field_name, field_type in annotations.items():
                val = kwargs.get(field_name)
                if val is not None and not isinstance(val, field_type):
                    raise TypeError(
                        f"{field_name!r} must be {field_type.__name__}"
                    )
                setattr(self, field_name, val)
        namespace['__init__'] = __init__

        # Generate __repr__:
        def __repr__(self) -> str:
            fields = ", ".join(
                f"{f}={getattr(self, f, None)!r}"
                for f in type(self)._data_fields
            )
            return f"{type(self).__name__}({fields})"
        namespace['__repr__'] = __repr__

        # Generate __eq__:
        def __eq__(self, other: object) -> bool:
            if type(self) is not type(other):
                return NotImplemented
            return all(
                getattr(self, f) == getattr(other, f)
                for f in type(self)._data_fields
            )
        namespace['__eq__'] = __eq__

        def __hash__(self) -> int:
            return hash(tuple(getattr(self, f) for f in type(self)._data_fields))
        namespace['__hash__'] = __hash__

        return super().__new__(mcs, name, bases, namespace)


class DataClass(metaclass=DataMeta):
    pass


class Point(DataClass):
    x: float
    y: float


p1 = Point(x=1.0, y=2.0)
p2 = Point(x=1.0, y=2.0)
p3 = Point(x=3.0, y=4.0)

print(p1)          # Point(x=1.0, y=2.0)
print(p1 == p2)    # True
print(p1 == p3)    # False
print({p1, p2})    # {Point(x=1.0, y=2.0)} — hashable, deduped
```

**S10: A senior developer argues that metaclasses are overused and `__init_subclass__` or class decorators should be preferred. Do you agree?**

**Q:** Form an argument.

**A:**

```python
# Agree — with nuance. Here's the decision matrix:

# CASE: Plugin auto-registration — __init_subclass__ wins
class Plugin:
    _registry: dict[str, type] = {}

    def __init_subclass__(cls, /, name: str = "", **kw) -> None:
        super().__init_subclass__(**kw)
        Plugin._registry[name or cls.__name__] = cls


# CASE: Adding __repr__/__eq__ — class decorator wins
from dataclasses import dataclass
@dataclass  # Far simpler than writing a metaclass for this!
class Config:
    host: str
    port: int


# CASE: Modifying namespace DURING class body execution — metaclass REQUIRED
class OrderedMeta(type):
    @classmethod
    def __prepare__(mcs, name, bases, **kw):
        return {}   # or OrderedDict, or a validation-capturing dict

    def __new__(mcs, name, bases, ns):
        # Add a method that references something only in the namespace:
        ns['method_count'] = sum(1 for v in ns.values() if callable(v))
        return super().__new__(mcs, name, bases, ns)

class API(metaclass=OrderedMeta):
    def get(self): pass
    def post(self): pass

print(API.method_count)   # 2 — only metaclass can do this cleanly

# VERDICT: Use metaclass ONLY when:
# 1. You need __prepare__ (namespace modification before class body runs)
# 2. The operation cannot be cleanly done by __init_subclass__ or a decorator
# 3. The benefit clearly outweighs the added complexity
# In all other cases, prefer the simpler alternatives.
```

---

## 4. Quick Recap / Cheatsheet

- **Metaclass = class of a class**: controls how classes are created, just as classes control instances
- **Default metaclass is `type`**: all classes (unless otherwise specified) are instances of `type`
- **`type(name, bases, namespace)`**: dynamic class creation — what Python does for every `class` statement
- **`metaclass=` keyword**: specify custom metaclass in class definition
- **Class creation order**: `__prepare__` → class body executes → `__new__` → `__init__` → `__set_name__` → `__init_subclass__`
- **`__prepare__`**: returns the namespace mapping; only in metaclasses; enables ordered/validated namespaces
- **`type.__call__`**: override in metaclass to intercept instance creation (before `__new__`/`__init__`)
- **`__init_subclass__`**: simpler alternative to metaclasses for subclass notification (Python 3.6+)
- **Class decorator**: runs after class creation; simpler than metaclass for most attribute-injection tasks
- **Prefer order**: `__init_subclass__` → class decorator → metaclass
- **Metaclass conflict**: two parents with incompatible metaclasses → `TypeError`; fix by creating a combined metaclass
- **`ABCMeta`**: metaclass behind `ABC`; tracks `__abstractmethods__`; prevents instantiation of abstract classes
- **`isinstance(MyClass, MyMeta)`**: always `True` — classes are instances of their metaclass
- **Use metaclasses for**: `__prepare__`, framework internals (ORMs, test runners), enforcing constraints across entire hierarchies
- **Avoid metaclasses for**: singleton (use `__new__`), plugin registration (use `__init_subclass__`), attribute injection (use decorator)
