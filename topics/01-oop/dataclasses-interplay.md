# Dataclasses and OOP Interplay

## 1. Concept (Deep Explanation)

### What `@dataclass` Does

`@dataclass` (introduced in Python 3.7, PEP 557) is a class decorator that inspects **type annotations** in a class body and auto-generates boilerplate special methods: primarily `__init__`, `__repr__`, and `__eq__`. It does this at class decoration time, not at import time of the `dataclasses` module.

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float
    label: str = "origin"

# Equivalent handwritten class would be:
class PointManual:
    def __init__(self, x: float, y: float, label: str = "origin") -> None:
        self.x = x
        self.y = y
        self.label = label

    def __repr__(self) -> str:
        return f"Point(x={self.x!r}, y={self.y!r}, label={self.label!r})"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, PointManual):
            return NotImplemented
        return (self.x, self.y, self.label) == (other.x, other.y, other.label)

p = Point(1.0, 2.0)
print(p)           # Point(x=1.0, y=2.0, label='origin')
print(p == Point(1.0, 2.0))   # True
```

`@dataclass` inspects `__annotations__` (a dict of `name → type`) and processes only fields that have type annotations. Non-annotated attributes are ignored.

### `field()` — Fine-Grained Control

The `field()` function provides per-field customisation:

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class Article:
    title: str
    content: str
    tags: list[str] = field(default_factory=list)    # mutable default — MUST use factory
    word_count: int = field(init=False, repr=False)   # not in __init__, not in __repr__
    created_at: datetime = field(default_factory=datetime.now, compare=False)
    _cache: dict = field(default_factory=dict, init=False, repr=False, compare=False)
    score: float = field(default=0.0, metadata={"unit": "points", "range": (0, 100)})

    def __post_init__(self) -> None:
        self.word_count = len(self.content.split())  # computed from content

a = Article(
    title="Python Deep Dive",
    content="This is a comprehensive article about Python dataclasses.",
    tags=["python", "oop"],
)
print(a)
# Article(title='Python Deep Dive', content='...', tags=['python', 'oop'], score=0.0)
# word_count NOT in repr (repr=False)

# Access field metadata:
from dataclasses import fields
for f in fields(Article):
    if f.metadata:
        print(f.name, f.metadata)
```

**Key `field()` parameters:**
- `default` / `default_factory` — provide a default value or a callable that produces one (use factory for mutables like `list`, `dict`)
- `init=False` — exclude from `__init__`; set in `__post_init__`
- `repr=False` — exclude from `__repr__`
- `compare=False` — exclude from `__eq__` (and `__lt__` etc. if `order=True`)
- `hash=None/True/False` — control whether this field is included in `__hash__`
- `metadata` — read-only mapping of arbitrary metadata for the field

### `ClassVar` to Exclude from `__init__`

Fields annotated with `ClassVar` are class-level variables — not instance fields. `@dataclass` ignores them for `__init__` generation:

```python
from dataclasses import dataclass
from typing import ClassVar

@dataclass
class Config:
    DEFAULT_TIMEOUT: ClassVar[int] = 30   # class variable — no __init__ param
    DEFAULT_RETRIES: ClassVar[int] = 3

    host: str
    port: int = 8080
    timeout: int = DEFAULT_TIMEOUT        # references class variable

cfg = Config("localhost")
print(cfg.host)            # "localhost"
print(cfg.timeout)         # 30
print(Config.DEFAULT_TIMEOUT)  # 30 — class variable
```

Without `ClassVar`, `DEFAULT_TIMEOUT: int = 30` would create an instance field with a default of 30.

### `InitVar` — Init-Only Parameters

`InitVar[T]` fields appear in `__init__` but are not stored as instance attributes. They are passed to `__post_init__` for processing:

```python
from dataclasses import dataclass, InitVar
from typing import Optional

@dataclass
class DBConnection:
    host: str
    port: int
    # Init-only: used to configure the connection, not stored:
    password: InitVar[str] = ""
    _connection = None   # Not annotated → not a field

    def __post_init__(self, password: str) -> None:
        """password is passed here, not stored on self."""
        if password:
            self._connection = self._connect(password)

    def _connect(self, password: str) -> str:
        return f"Connected to {self.host}:{self.port}"


conn = DBConnection("db.example.com", 5432, password="secret")
print(conn.host)         # "db.example.com"
# print(conn.password)   # AttributeError — not stored!
```

### `__post_init__` Hook

Called at the end of the generated `__init__`. Use it for:
- Validation
- Derived field computation
- Processing `InitVar` arguments

```python
from dataclasses import dataclass, field
import math

@dataclass
class Circle:
    radius: float

    def __post_init__(self) -> None:
        if self.radius <= 0:
            raise ValueError(f"Radius must be positive, got {self.radius}")

    @property
    def area(self) -> float:
        return math.pi * self.radius ** 2

    @property
    def circumference(self) -> float:
        return 2 * math.pi * self.radius

c = Circle(5.0)
print(c.area)           # 78.539...
Circle(-1)              # ValueError
```

### `frozen=True` — Immutable Dataclasses

`frozen=True` generates `__setattr__` and `__delattr__` that raise `FrozenInstanceError`, making instances effectively immutable. It also generates `__hash__` (since instances are now safely hashable):

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class ImmutablePoint:
    x: float
    y: float

p = ImmutablePoint(1.0, 2.0)
p.x = 99.0    # FrozenInstanceError: cannot assign to field 'x'

# Hashable — can be used in sets and as dict keys:
points = {ImmutablePoint(0, 0), ImmutablePoint(1, 1), ImmutablePoint(0, 0)}
print(points)  # {ImmutablePoint(x=0.0, y=0.0), ImmutablePoint(x=1.0, y=1.0)}

d = {p: "origin"}
print(d[ImmutablePoint(1.0, 2.0)])  # "origin" — hash matches!
```

**Important**: `frozen=True` uses `object.__setattr__` internally to set fields during `__init__` (since the generated `__setattr__` would block it). If you call `super().__init__()` from a non-frozen dataclass into a frozen parent, this can cause subtle issues.

### `slots=True` (Python 3.10+) — Memory Efficiency

```python
from dataclasses import dataclass
import sys

@dataclass
class NormalPoint:
    x: float
    y: float

@dataclass(slots=True)
class SlottedPoint:
    x: float
    y: float

n = NormalPoint(1.0, 2.0)
s = SlottedPoint(1.0, 2.0)

print(sys.getsizeof(n) + sys.getsizeof(n.__dict__))  # ~288 bytes
print(sys.getsizeof(s))  # ~56 bytes — no __dict__!
print(hasattr(n, '__dict__'))  # True
print(hasattr(s, '__dict__'))  # False
```

`slots=True` creates a new class with `__slots__` set to all field names (because `__slots__` must be set at class creation, the decorator creates a fresh class).

### `kw_only=True` (Python 3.10+)

Forces all fields (or specific fields via `field(kw_only=True)`) to be keyword-only in `__init__`:

```python
from dataclasses import dataclass, field

@dataclass(kw_only=True)
class APIConfig:
    host: str
    port: int = 8080
    timeout: int = 30
    retries: int = 3

# All arguments must be keyword arguments:
cfg = APIConfig(host="localhost", timeout=60)
# APIConfig("localhost", 8080)  # TypeError — no positional args!

# Per-field kw_only (Python 3.10+):
@dataclass
class MixedArgs:
    required_pos: str        # positional
    optional_pos: int = 0   # positional with default
    kw_field: str = field(default="", kw_only=True)  # keyword-only
```

### Inheritance with Dataclasses: MRO and Field Ordering

Field ordering in inherited dataclasses follows the MRO. Parent fields come first, child fields come after:

```python
from dataclasses import dataclass

@dataclass
class Base:
    x: int
    y: int = 0

@dataclass
class Child(Base):
    z: int = 0   # OK — child field with default after parent field with default

# Child.__init__ signature: (self, x: int, y: int = 0, z: int = 0)

c = Child(x=1, z=3)
print(c)   # Child(x=1, y=0, z=3)

# PROBLEM: A parent field with a default, then a child field WITHOUT a default
# This would create an invalid signature: __init__(self, x: int, y: int = 0, z: int)
@dataclass
class GoodBase:
    x: int       # no default

@dataclass
class BadChild(GoodBase):
    z: int       # also no default — fine: x then z
    y: int = 0   # default — fine: z... wait, x=int, z=int, y=int=0 is OK actually

# But this FAILS:
# @dataclass
# class Base2:
#     x: int = 0   # has default
# @dataclass
# class Child2(Base2):
#     y: int      # NO default — TypeError: non-default follows default!
```

**Frozen parent + non-frozen child is illegal:**

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class FrozenBase:
    x: int

# @dataclass
# class MutableChild(FrozenBase):  # TypeError! Cannot make mutable subclass of frozen
#     y: int
```

Both parent and child must agree on `frozen`.

### Comparison: `dataclass` vs `NamedTuple` vs `TypedDict` vs Plain Class

| Feature | `@dataclass` | `NamedTuple` | `TypedDict` | Plain class |
|---|---|---|---|---|
| Mutability | Mutable (default) | Immutable | N/A (dict) | Mutable |
| `__init__` | Auto-generated | Auto-generated | N/A | Manual |
| `__repr__` | Auto-generated | Auto-generated | N/A | Manual |
| `__eq__` | Auto-generated | Auto-generated | N/A | Manual |
| Hashable | `frozen=True` | Yes (values hashable) | No (it's a dict) | Manual |
| Memory | Standard / `slots=True` | Tuple-efficient | Dict-overhead | Standard |
| Unpacking | No | Yes (`a, b = point`) | No | No |
| Inheritance | Full OOP | Limited | No | Full OOP |
| Type checking | Full | Full | dict-style | Full |
| `__post_init__` | Yes | No | N/A | N/A |

```python
from typing import NamedTuple, TypedDict
from dataclasses import dataclass

# NamedTuple — tuple-based, immutable, unpackable
class PointNT(NamedTuple):
    x: float
    y: float

pnt = PointNT(1.0, 2.0)
x, y = pnt   # unpacking works!
print(pnt[0])  # index access works — it IS a tuple

# TypedDict — dict with type hints (not a class instance)
class ConfigTD(TypedDict):
    host: str
    port: int

cfg: ConfigTD = {"host": "localhost", "port": 8080}

# @dataclass — most flexible for OOP
@dataclass
class PointDC:
    x: float
    y: float

    def distance(self) -> float:
        return (self.x**2 + self.y**2) ** 0.5
```

### When to Use vs When Plain Class or Pydantic Is Better

**Use `@dataclass` when:**
- You need a class with named fields and auto-generated dunder methods
- You want mutable instances with type hints
- You need inheritance, computed properties, or OOP methods
- Performance matters (use `slots=True` for memory efficiency)

**Use plain class when:**
- Complex `__init__` logic that doesn't map to simple field assignment
- Heavy computation or database interaction in `__init__`
- You're wrapping a legacy API

**Use Pydantic `BaseModel` when:**
- You need runtime validation (dataclasses validate only with `__post_init__` — no built-in type enforcement)
- You're parsing JSON/API data
- You need `.model_dump()`, `.model_validate()`, JSON serialisation
- You need nested validation with coercion

```python
# @dataclass does NOT validate types at runtime:
from dataclasses import dataclass

@dataclass
class BadConfig:
    port: int

bc = BadConfig(port="not-an-int")   # No error! dataclass doesn't coerce/validate
print(bc.port)  # "not-an-int" — surprise!

# Pydantic DOES validate:
# from pydantic import BaseModel
# class Config(BaseModel):
#     port: int
# Config(port="not-an-int")  # ValidationError — converted to int automatically
```

---

## 2. Conceptual Interview Questions (10)

**Q1: How does `@dataclass` know which fields to include in `__init__`?**

**A:** `@dataclass` inspects `cls.__annotations__` — a dictionary mapping attribute names to their type annotations. Only annotated names are treated as fields. Non-annotated class-level attributes (or those typed with `ClassVar`) are ignored.

```python
from dataclasses import dataclass
from typing import ClassVar

@dataclass
class Demo:
    x: int              # field — in __init__
    y: str = "default"  # field with default — in __init__
    CLASS_CONST: ClassVar[str] = "constant"  # ClassVar — NOT in __init__
    helper = lambda self: None  # no annotation — NOT a field

# Inspect:
print(Demo.__annotations__)  # {'x': int, 'y': str, 'CLASS_CONST': ClassVar[str]}

# Fields (ClassVar excluded):
from dataclasses import fields
print([f.name for f in fields(Demo)])  # ['x', 'y']

# __init__ signature: (self, x: int, y: str = 'default')
```

**Q2: Why does using a mutable default (like `list`) directly cause a `ValueError` in `@dataclass`?**

**A:** If you write `tags: list = []` in a dataclass, all instances would share the same list object (a classic Python mutable default anti-pattern). `@dataclass` detects this at decoration time and raises `ValueError`:

```python
from dataclasses import dataclass, field

# @dataclass
# class Bad:
#     tags: list = []   # ValueError: mutable default <class 'list'> is not allowed

@dataclass
class Good:
    tags: list[str] = field(default_factory=list)  # each instance gets a new list

a = Good()
b = Good()
a.tags.append("python")
print(b.tags)   # [] — independent!
```

The `default_factory` is called once per instance creation (inside the generated `__init__`), ensuring each instance gets a fresh mutable object.

**Q3: What does `frozen=True` actually generate and how does it make instances immutable?**

**A:** `frozen=True` generates two methods:

```python
def __setattr__(self, name, value):
    raise dataclasses.FrozenInstanceError('cannot assign to field {name!r}')

def __delattr__(self, name):
    raise dataclasses.FrozenInstanceError('cannot delete field {name!r}')
```

But wait — if `__setattr__` raises, how does `__init__` set the fields? The generated `__init__` uses `object.__setattr__(self, name, value)` directly, bypassing the custom `__setattr__`:

```python
# Generated __init__ for frozen=True:
def __init__(self, x: float, y: float) -> None:
    object.__setattr__(self, 'x', x)  # bypasses the frozen __setattr__
    object.__setattr__(self, 'y', y)
```

Additionally, `frozen=True` generates `__hash__` from all fields (unless `unsafe_hash=True` is also used or a field has `hash=False`).

**Q4: How does `__post_init__` interact with `InitVar`?**

**A:** `InitVar[T]` fields are passed to `__init__` but not stored on the instance. After all regular fields are set, the generated `__init__` calls `self.__post_init__(initvar1, initvar2, ...)` with the `InitVar` values as positional arguments, in the order they appear in the class.

```python
from dataclasses import dataclass, InitVar

@dataclass
class HashedValue:
    value: str
    salt: InitVar[str]   # passed to __post_init__, not stored
    _hash: str = ""      # computed in __post_init__

    def __post_init__(self, salt: str) -> None:
        import hashlib
        combined = self.value + salt
        self._hash = hashlib.sha256(combined.encode()).hexdigest()


hv = HashedValue("secret", salt="random_salt_xyz")
print(hv.value)   # "secret"
print(hv._hash)   # sha256 hash
# print(hv.salt)  # AttributeError — InitVar not stored!
```

**Q5: Explain the field ordering rules when inheriting from multiple dataclasses.**

**A:** Dataclass inheritance follows MRO for field ordering. Fields from base classes appear first, in MRO order, followed by child class fields. The constraint: a field with a default value cannot be followed by a field without a default (since this would produce an invalid function signature).

```python
from dataclasses import dataclass

@dataclass
class A:
    a: int        # no default

@dataclass
class B(A):
    b: int = 0   # default — OK, 'a' (no default) comes first

@dataclass
class C(B):
    c: int = 0   # default — OK, ordering: a, b, c

# C.__init__(self, a: int, b: int = 0, c: int = 0)
print(C(1))       # C(a=1, b=0, c=0)

# PROBLEM: parent has default, child wants no default
@dataclass
class Parent:
    x: int = 0   # has default

# @dataclass
# class Problem(Parent):
#     y: int     # no default — TypeError! non-default arg after default arg
```

In Python 3.10+, `kw_only=True` solves this problem by making all fields keyword-only (no positional ordering constraints).

**Q6: What is the difference between `eq=True` (default) and `order=True` in `@dataclass`?**

**A:**
- `eq=True` (default): generates `__eq__` and `__ne__`, comparing all fields
- `order=True`: additionally generates `__lt__`, `__le__`, `__gt__`, `__ge__` by comparing field tuples in definition order

```python
from dataclasses import dataclass

@dataclass(order=True)
class Version:
    major: int
    minor: int
    patch: int

versions = [Version(1, 3, 0), Version(1, 2, 5), Version(2, 0, 0)]
print(sorted(versions))
# [Version(major=1, minor=2, patch=5), Version(major=1, minor=3, patch=0), Version(major=2, minor=0, patch=0)]

# If order=True AND frozen=False, __hash__ is NOT generated (mutable + orderable is unusual)
# Set unsafe_hash=True if you really need it
```

**Q7: How does `@dataclass(slots=True)` work under the hood in Python 3.10+?**

**A:** `slots=True` cannot retroactively add `__slots__` to an existing class (slots must be set at class creation). So the decorator:

1. Creates the class normally (without `__slots__`)
2. Collects all field names as the slot set
3. Creates a **new class** with `__slots__` set to those names, copying all methods and class variables from the original class
4. Replaces the class with the new one

```python
from dataclasses import dataclass
import sys

@dataclass(slots=True)
class Config:
    host: str
    port: int

# Verify:
print(Config.__slots__)          # ('host', 'port')
print(hasattr(Config(), '__dict__'))  # False
print(sys.getsizeof(Config("localhost", 8080)))  # ~56 bytes

# To include __weakref__ support:
# Unfortunately, slots=True doesn't add __weakref__ automatically;
# add it via field if needed (it's tricky with dataclasses)
```

**Q8: Can you combine `frozen=True` and `slots=True`?**

**A:** Yes — this is the most memory-efficient and immutable combination:

```python
from dataclasses import dataclass
import sys

@dataclass(frozen=True, slots=True)
class ImmutablePoint:
    x: float
    y: float

    def distance_to(self, other: "ImmutablePoint") -> float:
        return ((self.x - other.x)**2 + (self.y - other.y)**2) ** 0.5

p1 = ImmutablePoint(0.0, 0.0)
p2 = ImmutablePoint(3.0, 4.0)

print(p1.distance_to(p2))   # 5.0
p1.x = 1.0                  # FrozenInstanceError

# Hashable + memory-efficient:
s = {p1, p2, ImmutablePoint(0.0, 0.0)}
print(len(s))               # 2 — p1 deduplicated
print(sys.getsizeof(p1))    # Compact!
```

**Q9: What is `unsafe_hash=True` and when would you use it?**

**A:** By default, if `eq=True` and `frozen=False`, `@dataclass` sets `__hash__ = None` (making instances unhashable) because mutable objects with custom equality are unsafe to hash (the hash could change after insertion into a set/dict).

`unsafe_hash=True` generates a `__hash__` anyway, with the understanding that you (the developer) guarantee not to mutate the object after hashing:

```python
from dataclasses import dataclass

@dataclass(unsafe_hash=True)
class CacheKey:
    """We promise not to mutate this after using it as a dict key."""
    query: str
    page: int

cache: dict[CacheKey, list] = {}
key = CacheKey("SELECT * FROM users", 1)
cache[key] = [{"id": 1}]
print(cache[key])  # Works!

# If you do this, you're on your own:
# key.page = 2   # Hash changes! Dict lookup will fail silently!
```

**Q10: What are the advantages of Pydantic `BaseModel` over `@dataclass` for API models?**

**A:** The key differences:

| Feature | `@dataclass` | Pydantic `BaseModel` |
|---|---|---|
| Runtime type validation | No | Yes |
| Type coercion | No | Yes |
| JSON serialisation | No | `.model_dump()` |
| Custom validators | `__post_init__` | `@field_validator` |
| Nested model validation | No | Yes |
| Error messages | Exceptions | `ValidationError` with details |
| Performance | Faster (no validation) | Slower (validation overhead) |

```python
from dataclasses import dataclass

@dataclass
class DCUser:
    name: str
    age: int

u = DCUser(name=123, age="thirty")   # No error! Types not enforced.
print(u.name, type(u.name))  # 123 <class 'int'>

# from pydantic import BaseModel
# class PydanticUser(BaseModel):
#     name: str
#     age: int
# PydanticUser(name=123, age="30")   # name coerced to "123", age to 30
# PydanticUser(name="Alice", age="not-a-number")  # ValidationError!

# Use dataclass for pure Python OOP / performance-critical code
# Use Pydantic for API request/response parsing, configuration, any external data
```

---

## 3. Scenario-Based Interview Questions (10)

**S1: You need a `Rectangle` dataclass where `area` and `perimeter` are computed properties. Implement it.**

**Q:** Combine dataclass with properties.

**A:**

```python
from dataclasses import dataclass
import math

@dataclass
class Rectangle:
    width: float
    height: float

    def __post_init__(self) -> None:
        if self.width <= 0 or self.height <= 0:
            raise ValueError(f"Dimensions must be positive: {self.width}x{self.height}")

    @property
    def area(self) -> float:
        return self.width * self.height

    @property
    def perimeter(self) -> float:
        return 2 * (self.width + self.height)

    @property
    def diagonal(self) -> float:
        return math.sqrt(self.width**2 + self.height**2)

    def scale(self, factor: float) -> "Rectangle":
        """Returns a new rectangle — immutability-style API."""
        return Rectangle(self.width * factor, self.height * factor)


r = Rectangle(3.0, 4.0)
print(r.area)        # 12.0
print(r.diagonal)    # 5.0
print(r.scale(2))    # Rectangle(width=6.0, height=8.0)
Rectangle(-1, 4)     # ValueError
```

**S2: Design a `User` dataclass hierarchy for an authentication system with admin and regular users.**

**Q:** Implement inheritance between dataclasses.

**A:**

```python
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum, auto

class Role(Enum):
    USER = auto()
    MODERATOR = auto()
    ADMIN = auto()

@dataclass
class BaseUser:
    username: str
    email: str
    created_at: datetime = field(default_factory=datetime.now, compare=False, repr=False)
    is_active: bool = True

    def __post_init__(self) -> None:
        self.email = self.email.lower().strip()
        if not self.username.strip():
            raise ValueError("Username cannot be blank")

    def can_read(self) -> bool:
        return self.is_active

    def __repr__(self) -> str:
        return f"{type(self).__name__}(username={self.username!r}, email={self.email!r})"


@dataclass
class RegularUser(BaseUser):
    subscription: str = "free"

    def can_write(self) -> bool:
        return self.is_active and self.subscription in ("pro", "enterprise")


@dataclass
class AdminUser(BaseUser):
    role: Role = Role.ADMIN
    managed_users: list[str] = field(default_factory=list, compare=False)

    def can_write(self) -> bool:
        return self.is_active

    def can_manage(self) -> bool:
        return self.is_active and self.role == Role.ADMIN


u = RegularUser("alice", "Alice@Example.COM", subscription="pro")
a = AdminUser("superuser", "admin@corp.com")

print(u)              # RegularUser(username='alice', email='alice@example.com')
print(u.can_write())  # True
print(a.can_manage()) # True
print(u.email)        # "alice@example.com" — normalised in __post_init__
```

**S3: You need to serialise a dataclass to and from JSON. Implement a mixin.**

**Q:** Add JSON serialisation to dataclasses.

**A:**

```python
import json
from dataclasses import dataclass, fields, asdict, astuple
from datetime import datetime
from typing import Any

class JSONMixin:
    """Adds JSON serialisation to dataclasses."""

    def to_json(self, indent: int | None = None) -> str:
        return json.dumps(self._to_dict(), indent=indent, default=self._json_default)

    def _to_dict(self) -> dict[str, Any]:
        return asdict(self)   # recursive dict conversion (handles nested dataclasses)

    @staticmethod
    def _json_default(obj: Any) -> Any:
        if isinstance(obj, datetime):
            return obj.isoformat()
        raise TypeError(f"Object of type {type(obj).__name__} is not JSON serialisable")

    @classmethod
    def from_json(cls, json_str: str) -> "JSONMixin":
        data = json.loads(json_str)
        return cls(**data)


@dataclass
class Product(JSONMixin):
    id: int
    name: str
    price: float
    tags: list[str] = None

    def __post_init__(self) -> None:
        if self.tags is None:
            self.tags = []


p = Product(1, "Widget", 9.99, ["new", "sale"])
json_str = p.to_json(indent=2)
print(json_str)

p2 = Product.from_json('{"id": 2, "name": "Gadget", "price": 19.99, "tags": []}')
print(p2)
```

**S4: A `Config` dataclass should be immutable after creation but support "functional update" (creating a modified copy).**

**Q:** Implement frozen dataclass with `replace()`.

**A:**

```python
from dataclasses import dataclass, replace

@dataclass(frozen=True, slots=True)
class ServerConfig:
    host: str = "localhost"
    port: int = 8080
    timeout: int = 30
    max_retries: int = 3
    debug: bool = False

    def with_debug(self) -> "ServerConfig":
        """Returns a debug-enabled copy."""
        return replace(self, debug=True, timeout=5)

    def with_port(self, port: int) -> "ServerConfig":
        """Returns a copy with a different port."""
        if port < 1 or port > 65535:
            raise ValueError(f"Invalid port: {port}")
        return replace(self, port=port)

    def for_environment(self, env: str) -> "ServerConfig":
        """Returns an environment-appropriate copy."""
        if env == "production":
            return replace(self, debug=False, max_retries=5, timeout=60)
        elif env == "staging":
            return replace(self, debug=True, timeout=10)
        return replace(self, debug=True, timeout=5)


cfg = ServerConfig()
prod_cfg = cfg.for_environment("production")
debug_cfg = prod_cfg.with_debug()

print(cfg)       # ServerConfig(host='localhost', port=8080, ...)
print(prod_cfg)  # ServerConfig(..., debug=False, timeout=60, max_retries=5)
print(debug_cfg) # ServerConfig(..., debug=True)

# Immutability verified:
try:
    cfg.port = 9090
except Exception as e:
    print(f"Immutable: {e}")

# Use as dict keys (hashable):
configs = {cfg: "default", prod_cfg: "production"}
```

**S5: Implement a dataclass-based event system with typed events.**

**Q:** Design events using frozen dataclasses.

**A:**

```python
from dataclasses import dataclass, field
from datetime import datetime
from typing import Callable, ClassVar
from abc import ABC

@dataclass(frozen=True)
class Event(ABC):
    event_id: str
    timestamp: datetime = field(default_factory=datetime.now, compare=False)

    _handlers: ClassVar[dict[type, list[Callable]]] = {}

    @classmethod
    def subscribe(cls, event_type: type, handler: Callable) -> None:
        cls._handlers.setdefault(event_type, []).append(handler)

    @classmethod
    def publish(cls, event: "Event") -> None:
        for handler in cls._handlers.get(type(event), []):
            handler(event)


@dataclass(frozen=True)
class UserCreated(Event):
    username: str
    email: str

@dataclass(frozen=True)
class OrderPlaced(Event):
    order_id: str
    total: float
    items: tuple[str, ...] = field(default_factory=tuple)

# Register handlers:
def on_user_created(event: UserCreated) -> None:
    print(f"Welcome email sent to {event.email}")

def on_order_placed(event: OrderPlaced) -> None:
    print(f"Processing order {event.order_id} for ${event.total:.2f}")

Event.subscribe(UserCreated, on_user_created)
Event.subscribe(OrderPlaced, on_order_placed)

# Publish events:
Event.publish(UserCreated(event_id="evt-001", username="alice", email="alice@example.com"))
Event.publish(OrderPlaced(event_id="evt-002", order_id="ORD-123", total=49.99, items=("Widget",)))
```

**S6: Compare the memory usage of `@dataclass`, `@dataclass(slots=True)`, and `NamedTuple` for a high-volume scenario.**

**Q:** Benchmark and recommend.

**A:**

```python
import sys
from dataclasses import dataclass
from typing import NamedTuple

@dataclass
class RegularPoint:
    x: float
    y: float
    z: float

@dataclass(slots=True)
class SlottedPoint:
    x: float
    y: float
    z: float

class NTPoint(NamedTuple):
    x: float
    y: float
    z: float

# Per-instance sizes:
rp = RegularPoint(1.0, 2.0, 3.0)
sp = SlottedPoint(1.0, 2.0, 3.0)
ntp = NTPoint(1.0, 2.0, 3.0)

rp_size = sys.getsizeof(rp) + sys.getsizeof(rp.__dict__)
sp_size = sys.getsizeof(sp)
ntp_size = sys.getsizeof(ntp)

print(f"Regular dataclass: {rp_size} bytes")   # ~288 bytes
print(f"Slotted dataclass: {sp_size} bytes")   # ~56 bytes
print(f"NamedTuple:        {ntp_size} bytes")  # ~64 bytes

# For 1,000,000 instances:
print(f"\nFor 1M instances:")
print(f"Regular: {rp_size * 1_000_000 / 1024**2:.1f} MB")
print(f"Slotted: {sp_size * 1_000_000 / 1024**2:.1f} MB")
print(f"NTuple:  {ntp_size * 1_000_000 / 1024**2:.1f} MB")

# Recommendation:
# - Pure data, immutable, unpackable → NamedTuple
# - Mutable with OOP methods → @dataclass(slots=True)
# - Rarely instantiated → @dataclass (simpler)
```

**S7: A `TreeNode` dataclass needs to support recursive structures. Show how to handle default values and self-referential types.**

**Q:** Implement recursive dataclass.

**A:**

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class TreeNode:
    value: int
    left: Optional[TreeNode] = None
    right: Optional[TreeNode] = None

    def insert(self, value: int) -> None:
        if value < self.value:
            if self.left is None:
                self.left = TreeNode(value)
            else:
                self.left.insert(value)
        else:
            if self.right is None:
                self.right = TreeNode(value)
            else:
                self.right.insert(value)

    def inorder(self) -> list[int]:
        result = []
        if self.left:
            result.extend(self.left.inorder())
        result.append(self.value)
        if self.right:
            result.extend(self.right.inorder())
        return result

    def __repr__(self) -> str:
        parts = [f"value={self.value}"]
        if self.left:
            parts.append(f"left=TreeNode({self.left.value})")
        if self.right:
            parts.append(f"right=TreeNode({self.right.value})")
        return f"TreeNode({', '.join(parts)})"


root = TreeNode(5)
for v in [3, 7, 1, 4, 6, 8]:
    root.insert(v)

print(root.inorder())   # [1, 3, 4, 5, 6, 7, 8]
```

**S8: Implement a `@dataclass`-based configuration system that loads from environment variables.**

**Q:** Integrate dataclasses with external configuration.

**A:**

```python
import os
from dataclasses import dataclass, field, fields
from typing import Any, get_type_hints

@dataclass
class AppConfig:
    DATABASE_URL: str = "sqlite:///:memory:"
    SECRET_KEY: str = "dev-secret-key"
    DEBUG: bool = False
    MAX_CONNECTIONS: int = 10
    ALLOWED_HOSTS: list[str] = field(default_factory=list)

    def __post_init__(self) -> None:
        # Validate critical settings:
        if not self.DEBUG and self.SECRET_KEY == "dev-secret-key":
            raise ValueError("SECRET_KEY must be changed in production!")

    @classmethod
    def from_env(cls) -> "AppConfig":
        """Construct config from environment variables."""
        type_hints = get_type_hints(cls)
        kwargs: dict[str, Any] = {}

        for f in fields(cls):
            env_val = os.environ.get(f.name)
            if env_val is not None:
                hint = type_hints.get(f.name, str)
                kwargs[f.name] = cls._coerce(env_val, hint)

        return cls(**kwargs)

    @staticmethod
    def _coerce(value: str, target_type: type) -> Any:
        if target_type is bool:
            return value.lower() in ('true', '1', 'yes', 'on')
        if target_type is int:
            return int(value)
        if target_type is float:
            return float(value)
        if hasattr(target_type, '__origin__') and target_type.__origin__ is list:
            return [v.strip() for v in value.split(',') if v.strip()]
        return value


# Simulate environment:
os.environ.update({
    "DEBUG": "true",
    "MAX_CONNECTIONS": "50",
    "ALLOWED_HOSTS": "localhost,127.0.0.1,example.com",
})

cfg = AppConfig.from_env()
print(cfg.DEBUG)          # True
print(cfg.MAX_CONNECTIONS)  # 50
print(cfg.ALLOWED_HOSTS)  # ['localhost', '127.0.0.1', 'example.com']
```

**S9: How do you make a dataclass work with `functools.lru_cache` or as a dict key?**

**Q:** Make dataclass hashable.

**A:**

```python
from dataclasses import dataclass
import functools

# Option 1: frozen=True (recommended for truly immutable data)
@dataclass(frozen=True)
class QueryParams:
    table: str
    limit: int = 100
    offset: int = 0
    order_by: str = "id"

# Now it's hashable:
params = QueryParams("users", limit=20)
cache: dict[QueryParams, list] = {params: [{"id": 1}]}
print(cache[QueryParams("users", limit=20)])  # [{'id': 1}]

# Option 2: unsafe_hash=True for mutable dataclasses (use carefully!)
@dataclass(unsafe_hash=True)
class CacheKey:
    resource: str
    version: int

    # Promise: won't mutate after using as a dict key!

@functools.lru_cache(maxsize=128)
def expensive_computation(key: CacheKey) -> str:
    return f"Result for {key.resource} v{key.version}"

k = CacheKey("users", 1)
print(expensive_computation(k))   # Computed
print(expensive_computation(k))   # Cached
```

**S10: Design a versioned, immutable configuration system using dataclass `replace()`.**

**Q:** Implement version-tracked configs.

**A:**

```python
from dataclasses import dataclass, replace, field
from datetime import datetime
from typing import Any

@dataclass(frozen=True)
class ConfigVersion:
    version: int
    config: "DatabaseConfig"
    changed_by: str
    changed_at: datetime = field(default_factory=datetime.now)
    change_notes: str = ""


@dataclass(frozen=True)
class DatabaseConfig:
    host: str = "localhost"
    port: int = 5432
    database: str = "mydb"
    pool_size: int = 5
    max_overflow: int = 10

class ConfigHistory:
    def __init__(self, initial: DatabaseConfig, owner: str) -> None:
        self._history: list[ConfigVersion] = [
            ConfigVersion(version=1, config=initial, changed_by=owner,
                         change_notes="Initial configuration")
        ]

    @property
    def current(self) -> DatabaseConfig:
        return self._history[-1].config

    @property
    def version(self) -> int:
        return self._history[-1].version

    def update(self, changed_by: str, notes: str = "", **changes: Any) -> DatabaseConfig:
        new_config = replace(self.current, **changes)
        new_version = ConfigVersion(
            version=self.version + 1,
            config=new_config,
            changed_by=changed_by,
            change_notes=notes,
        )
        self._history.append(new_version)
        return new_config

    def rollback(self) -> DatabaseConfig:
        if len(self._history) <= 1:
            raise ValueError("Cannot rollback: already at initial version")
        self._history.pop()
        return self.current

    def audit_log(self) -> list[dict]:
        return [
            {"version": cv.version, "by": cv.changed_by,
             "at": cv.changed_at.isoformat(), "notes": cv.change_notes}
            for cv in self._history
        ]


db = ConfigHistory(DatabaseConfig(), owner="ops-team")
db.update("alice", notes="Scale up pool", pool_size=20, max_overflow=40)
db.update("bob", notes="Point to prod DB", host="db.prod.example.com", port=5433)

print(f"Current host: {db.current.host}")      # db.prod.example.com
print(f"Current pool: {db.current.pool_size}") # 20

db.rollback()
print(f"After rollback: {db.current.host}")    # still prod, pool_size=20

for entry in db.audit_log():
    print(entry)
```

---

## 4. Quick Recap / Cheatsheet

- **`@dataclass`** inspects `__annotations__` and generates `__init__`, `__repr__`, `__eq__` automatically
- **`field()`** controls per-field behaviour: `default_factory`, `init`, `repr`, `compare`, `hash`, `metadata`
- **Mutable defaults MUST use `default_factory=list/dict/...`** — direct `= []` raises `ValueError`
- **`ClassVar[T]`**: class-level variable — excluded from `__init__`, `__repr__`, and comparison
- **`InitVar[T]`**: appears in `__init__` only; passed to `__post_init__`; not stored as instance attribute
- **`__post_init__`**: called after generated `__init__` completes — use for validation and derived fields
- **`frozen=True`**: generates `__setattr__`/`__delattr__` that raise; uses `object.__setattr__` in `__init__`; generates `__hash__`
- **`slots=True`** (3.10+): creates a new class with `__slots__` — memory-efficient; no `__dict__`
- **`kw_only=True`** (3.10+): forces all fields to be keyword-only — solves default-ordering problems in inheritance
- **`order=True`**: generates `__lt__`, `__le__`, `__gt__`, `__ge__` for sorting
- **`unsafe_hash=True`**: generates `__hash__` even for mutable classes — use only if you won't mutate after hashing
- **`replace(instance, field=new_value)`**: functional update — returns modified copy; key for frozen dataclasses
- **`asdict(instance)`**: converts dataclass to nested dict (handles nested dataclasses recursively)
- **`fields(cls)`**: returns tuple of `Field` objects for inspection
- **Inheritance**: parent fields first, child fields after; fields with defaults cannot precede fields without (use `kw_only` to escape)
- **Frozen parent + non-frozen child = `TypeError`** — all classes must agree on `frozen`
- **vs `NamedTuple`**: NamedTuple is immutable, unpackable, tuple-based; dataclass is flexible OOP
- **vs Pydantic**: dataclass does NOT validate types at runtime; Pydantic does — use Pydantic for external data
