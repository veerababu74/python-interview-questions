# Instance, Class, and Static Variables

## 1. Concept (Deep Explanation)

### What They Are and Why They Exist

Python variables in a class fall into two fundamental categories:

- **Instance variables** — stored in each instance's `__dict__`, unique per object.
- **Class variables** — stored in the class's `__dict__`, shared across all instances (and accessible from subclasses).

Python has no `static` keyword. What other languages call *static variables* are implemented as **class variables** in Python. Understanding the lookup chain and how assignment affects it is critical for correct OOP code.

### The Attribute Lookup Chain

When Python evaluates `instance.attr`, it applies the **descriptor protocol** and the MRO-based lookup:

1. Check `type(instance).__mro__` for a **data descriptor** (defines both `__get__` and `__set__`).
2. Check `instance.__dict__`.
3. Check `type(instance).__mro__` for a **non-data descriptor** or plain class attribute.
4. Raise `AttributeError`.

This order has a crucial consequence: **instance `__dict__` entries shadow class variables**, unless the class variable is a data descriptor (like `property`).

```python
class Dog:
    species = "Canis lupus familiaris"   # class variable

    def __init__(self, name: str) -> None:
        self.name = name                 # instance variable

rex = Dog("Rex")
fido = Dog("Fido")

print(rex.species)    # reads from Dog.__dict__ (class variable)
print(fido.species)   # same class variable

rex.species = "Wolf"  # CREATES a new instance variable on rex — does NOT modify Dog.species
print(rex.species)    # "Wolf"  — rex.__dict__["species"]
print(fido.species)   # "Canis lupus familiaris"  — still reads class variable
print(Dog.species)    # "Canis lupus familiaris"  — unchanged
```

This is the **shadow effect**: assigning to `self.x` when a class variable `x` exists creates a *new* instance variable in `self.__dict__` that hides (shadows) the class variable. The class variable itself is untouched.

### The Classic Mutable Class Variable Bug

This is one of the most common Python interview gotchas:

```python
class Student:
    grades: list[int] = []   # WRONG: mutable class variable shared by all instances

    def __init__(self, name: str) -> None:
        self.name = name

    def add_grade(self, g: int) -> None:
        self.grades.append(g)   # mutates the SHARED list — does NOT create an instance copy

alice = Student("Alice")
bob   = Student("Bob")

alice.add_grade(90)
bob.add_grade(80)

print(alice.grades)   # [90, 80]  ← Bob's grade appears here!
print(bob.grades)     # [90, 80]  ← Alice's grade appears here!
print(alice.grades is bob.grades)  # True — same list object
```

**Fix:** Always initialise mutable defaults inside `__init__`:

```python
class Student:
    def __init__(self, name: str) -> None:
        self.name   = name
        self.grades: list[int] = []   # each instance gets its own list

    def add_grade(self, g: int) -> None:
        self.grades.append(g)

alice = Student("Alice")
bob   = Student("Bob")
alice.add_grade(90)
bob.add_grade(80)
print(alice.grades)  # [90]
print(bob.grades)    # [80]
```

### Legitimate Uses of Class Variables

Class variables shine for:
- **Constants** shared across all instances (`MAX_RETRIES = 3`)
- **Instance counters** and **caches** that should persist across objects
- **Registry patterns** mapping names to subclasses

```python
class Connection:
    MAX_RETRIES: int  = 3          # constant — safe as class variable
    _pool: list       = []         # shared pool — intentionally shared
    _instance_count   = 0

    def __init__(self, host: str) -> None:
        self.host = host
        Connection._instance_count += 1   # direct class reference avoids shadow risk

    @classmethod
    def instance_count(cls) -> int:
        return cls._instance_count
```

Note: updating a class variable should always be done via `ClassName.var += ...` or `cls.var += ...` inside a classmethod — never via `self.var += ...`, because `self.var += 1` is `self.var = self.var + 1`, which *reads* the class variable but *writes* an instance variable.

```python
class Counter:
    count = 0

    def bad_increment(self):
        self.count += 1        # self.count = self.count + 1
                               # reads Counter.count, writes self.__dict__['count']

    @classmethod
    def good_increment(cls):
        cls.count += 1         # modifies the class variable correctly

c1, c2 = Counter(), Counter()
c1.bad_increment()
print(Counter.count)  # 0  ← class variable unchanged!
print(c1.count)       # 1  ← instance variable created

Counter.good_increment()
print(Counter.count)  # 1  ← class variable correctly updated
```

### __slots__: An Alternative to __dict__

By default every Python instance has a `__dict__` that allows arbitrary attribute assignment. `__slots__` replaces this with a fixed set of slot descriptors:

```python
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

p = Point(1.0, 2.0)
# p.z = 3.0   # AttributeError: 'Point' object has no attribute 'z'
# p.__dict__  # AttributeError: 'Point' has no __dict__
```

**Benefits of `__slots__`:**
- Reduced memory usage (no per-instance `__dict__` overhead — CPython saves ~200 bytes per instance for small objects).
- Faster attribute access (slot descriptors are C-level, not dict lookup).
- Prevents accidental attribute creation.

**Caveats:**
- Classes with `__slots__` cannot have class variables with the same name as a slot (the slot descriptor occupies that name in the class `__dict__`).
- Inheritance with `__slots__` is tricky: if a subclass doesn't define `__slots__`, it gets `__dict__` back, partially defeating the purpose.
- `__weakref__` must be explicitly added to `__slots__` if you need weak references.

```python
class Vector:
    __slots__ = ("x", "y", "__weakref__")

    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

    def __repr__(self) -> str:
        return f"Vector({self.x}, {self.y})"

import sys
class VectorDict:
    def __init__(self, x, y):
        self.x, self.y = x, y

print(sys.getsizeof(Vector(1.0, 2.0)))    # ~56 bytes (CPython 3.12)
print(sys.getsizeof(VectorDict(1.0, 2.0))) # ~48 bytes base + __dict__ overhead
```

### ClassVar from typing

Python's `typing` module provides `ClassVar` to distinguish class variables from instance variables in type annotations. This is purely a type-checker hint; it has no runtime effect.

```python
from typing import ClassVar


class Registry:
    # Type checkers know this is a class variable — assignment via instance is an error
    _entries: ClassVar[dict[str, type]] = {}
    name: str    # instance variable

    def __init__(self, name: str) -> None:
        self.name = name

    @classmethod
    def register(cls, key: str, value: type) -> None:
        cls._entries[key] = value
```

Type checkers like mypy and pyright will flag `instance._entries = {}` as an error because `_entries` is annotated as `ClassVar`.

### Thread Safety of Class Variables

Class variables are shared across threads. Concurrent reads are safe, but concurrent increments are not:

```python
import threading

class GlobalCounter:
    value = 0
    _lock = threading.Lock()

    @classmethod
    def increment(cls) -> None:
        with cls._lock:
            cls.value += 1

threads = [threading.Thread(target=GlobalCounter.increment) for _ in range(1000)]
for t in threads: t.start()
for t in threads: t.join()
print(GlobalCounter.value)  # 1000 — correct with lock
```

Without the lock, `value += 1` is not atomic (it's LOAD + ADD + STORE, and the GIL does not protect across these three bytecodes reliably).

---

## 2. Conceptual Interview Questions (10)

**Q1: What is the difference between an instance variable and a class variable?**

**A:** An **instance variable** lives in each object's `__dict__` and is unique per instance. A **class variable** lives in the class's `__dict__` and is shared across all instances (and subclasses, via MRO lookup).

```python
class Foo:
    class_var = "shared"

    def __init__(self):
        self.inst_var = "unique"

a, b = Foo(), Foo()
print(a.inst_var is b.inst_var)   # False — different string objects
print(a.class_var is b.class_var) # True  — same object from class dict
```

The shadowing rule: assigning `self.class_var = x` creates a new entry in `self.__dict__`, not in `Foo.__dict__`.

---

**Q2: Why is using a mutable object (list/dict) as a class variable dangerous?**

**A:** All instances share the *same* mutable object. Mutating it (via `append`, `update`, etc.) affects every instance. Replacing it (via `self.items = []`) only affects the instance that performed the assignment, but the class variable remains pointing to the original object.

```python
class Bad:
    items = []

a, b = Bad(), Bad()
a.items.append(1)   # mutates the shared list
print(b.items)      # [1] — unexpected!

a.items = []        # replaces a's reference (shadow), does NOT clear Bad.items
print(b.items)      # [1] — still the original shared list
print(Bad.items)    # [1] — class variable unchanged
```

Always use `__init__` to initialise mutable per-instance collections.

---

**Q3: What does `self.x += 1` do when x is a class variable (an integer)?**

**A:** It is equivalent to `self.x = self.x + 1`. Python first reads `self.x` (which resolves to the class variable via MRO), computes `class_var + 1`, then assigns the result to `self.x` — creating a **new instance variable** that shadows the class variable. The class variable is untouched.

```python
class C:
    x = 10

c = C()
c.x += 1
print(c.x)    # 11 — instance variable
print(C.x)    # 10 — class variable unchanged
print("x" in c.__dict__)   # True
print("x" in C.__dict__)   # True (still the original 10)
```

---

**Q4: How does __slots__ affect memory and behaviour?**

**A:** `__slots__` replaces the per-instance `__dict__` with fixed C-level slot descriptors in the class. Benefits: ~50–200 bytes saved per instance (no dict overhead), faster attribute access. Drawbacks: no arbitrary attributes, inheritance complexity, no `__dict__` (pickle/copy may need special handling), `__weakref__` must be added manually.

```python
class WithSlots:
    __slots__ = ("x", "y")
    def __init__(self, x, y): self.x, self.y = x, y

class WithoutSlots:
    def __init__(self, x, y): self.x, self.y = x, y

import sys
ws  = WithSlots(1, 2)
wos = WithoutSlots(1, 2)
# sys.getsizeof doesn't include __dict__ size, use tracemalloc for accurate comparison
print(hasattr(ws, '__dict__'))  # False
print(hasattr(wos, '__dict__')) # True
```

---

**Q5: What is ClassVar and when should you use it?**

**A:** `ClassVar[T]` from `typing` marks a class-body annotation as belonging to the class, not instances. Type checkers use this to catch accidental per-instance assignments. It has **no runtime effect** — it's purely for static analysis.

```python
from typing import ClassVar

class Config:
    default_timeout: ClassVar[int] = 30   # class variable
    host: str                              # instance variable

    def __init__(self, host: str) -> None:
        self.host = host

c = Config("localhost")
# mypy error: Cannot assign to a ClassVar from instance:
# c.default_timeout = 60   # mypy: error
Config.default_timeout = 60  # OK — assignment on the class itself
```

---

**Q6: Can class variables be inherited? How does lookup work?**

**A:** Yes. Python's MRO lookup applies to class variables. If `Child` doesn't define `x`, `child_instance.x` searches `Child.__dict__`, then `Parent.__dict__`, etc.

```python
class Animal:
    kingdom = "Animalia"

class Dog(Animal):
    pass

class Poodle(Dog):
    pass

print(Poodle.kingdom)        # "Animalia" — found in Animal.__dict__
print(Poodle.__mro__)        # (Poodle, Dog, Animal, object)
```

Assigning `Dog.kingdom = "Canidae"` creates a new entry in `Dog.__dict__` but does not affect `Animal.__dict__`. `Poodle.kingdom` would then resolve to `Dog.kingdom = "Canidae"`.

---

**Q7: How do you implement a class-level cache that is shared across instances?**

**A:**

```python
from functools import lru_cache

class ExpensiveComputation:
    _cache: dict[int, int] = {}

    def compute(self, n: int) -> int:
        if n not in ExpensiveComputation._cache:
            ExpensiveComputation._cache[n] = self._expensive(n)
        return ExpensiveComputation._cache[n]

    @staticmethod
    def _expensive(n: int) -> int:
        return n * n  # placeholder

a = ExpensiveComputation()
b = ExpensiveComputation()
a.compute(10)
print(10 in ExpensiveComputation._cache)  # True — b can reuse this
```

Using `cls._cache` in a `@classmethod` achieves the same effect and is subclass-aware.

---

**Q8: What happens with class variables in multiple inheritance?**

**A:** MRO determines which class variable is found first. Diamonds and conflicts are resolved by C3 linearisation.

```python
class A:
    x = 1

class B(A):
    x = 2

class C(A):
    x = 3

class D(B, C):
    pass

print(D.x)      # 2 — B appears before C in D's MRO
print(D.__mro__) # (D, B, C, A, object)
```

---

**Q9: How do dataclasses handle instance vs class variables?**

**A:** `@dataclass` creates `__init__` that assigns annotated fields as instance variables. If you use `ClassVar` annotation, the field is excluded from `__init__`, `__repr__`, `__eq__`, etc.

```python
from dataclasses import dataclass, field
from typing import ClassVar

@dataclass
class Inventory:
    _count: ClassVar[int] = 0    # class variable, excluded from dataclass machinery

    name: str
    quantity: int = 0

    def __post_init__(self) -> None:
        Inventory._count += 1

a = Inventory("apple", 10)
b = Inventory("banana", 5)
print(Inventory._count)  # 2
```

Using `field(default_factory=list)` is the correct way to give each instance its own mutable default:

```python
@dataclass
class ShoppingCart:
    items: list[str] = field(default_factory=list)
```

---

**Q10: How are __slots__ and class variables related? Can you have both?**

**A:** A slot descriptor lives in the class's `__dict__` under the slot name. You cannot have a class variable with the same name as a slot (the slot descriptor would be overwritten or conflict). However, you can have class variables with *different* names from slots.

```python
class Example:
    __slots__ = ("x",)
    y: int = 100   # class variable — fine, different name from slot

e = Example()
e.x = 10
print(e.y)          # 100 — class variable accessible via instance
# e.y = 200         # AttributeError — no __dict__ and 'y' is not a slot
Example.y = 200     # OK — modifying class variable directly
print(e.y)          # 200
```

---

## 3. Scenario-Based Interview Questions (10)

**Scenario 1: A teammate uses a class variable list to store user sessions. Users see each other's sessions. What's wrong and how do you fix it?**

**Q:** Diagnose and fix the bug.

**A:** The class variable is a mutable list shared by all instances. Every session appends to the same list. Fix: initialise session storage in `__init__`.

```python
# Broken
class UserSession:
    sessions: list[str] = []   # shared!

    def add(self, token: str) -> None:
        self.sessions.append(token)

# Fixed
class UserSession:
    def __init__(self) -> None:
        self.sessions: list[str] = []   # per-instance

    def add(self, token: str) -> None:
        self.sessions.append(token)

u1, u2 = UserSession(), UserSession()
u1.add("tok-abc")
u2.add("tok-xyz")
print(u1.sessions)  # ['tok-abc']  — isolated
print(u2.sessions)  # ['tok-xyz']  — isolated
```

---

**Scenario 2: You need to limit the number of DB connections to 10 across all instances of a ConnectionPool class.**

**Q:** Implement the limit using a class variable.

**A:**

```python
import threading

class ConnectionPool:
    _max_connections: int = 10
    _active: int          = 0
    _lock                 = threading.Lock()

    def __init__(self) -> None:
        with ConnectionPool._lock:
            if ConnectionPool._active >= ConnectionPool._max_connections:
                raise RuntimeError("Connection pool exhausted")
            ConnectionPool._active += 1
        self._connected = True

    def close(self) -> None:
        if self._connected:
            with ConnectionPool._lock:
                ConnectionPool._active -= 1
            self._connected = False

    @classmethod
    def active_count(cls) -> int:
        return cls._active
```

---

**Scenario 3: You are designing a plugin system where each plugin registers itself by name. Implement it using class variables.**

**Q:** Design the registry.

**A:**

```python
class Plugin:
    _registry: dict[str, type["Plugin"]] = {}

    def __init_subclass__(cls, name: str = "", **kwargs: object) -> None:
        super().__init_subclass__(**kwargs)
        if name:
            Plugin._registry[name] = cls

    @classmethod
    def get(cls, name: str) -> type["Plugin"]:
        return cls._registry[name]

class AudioPlugin(Plugin, name="audio"):
    def run(self) -> str: return "playing audio"

class VideoPlugin(Plugin, name="video"):
    def run(self) -> str: return "playing video"

plugin_cls = Plugin.get("audio")
print(plugin_cls().run())  # "playing audio"
```

---

**Scenario 4: You have a high-frequency trading system where Point objects are created millions of times per second. Memory is critical. What optimisation do you apply?**

**Q:** Implement the optimisation.

**A:** Use `__slots__` to eliminate `__dict__` overhead:

```python
class PricePoint:
    __slots__ = ("timestamp", "bid", "ask")

    def __init__(self, timestamp: float, bid: float, ask: float) -> None:
        self.timestamp = timestamp
        self.bid       = bid
        self.ask       = ask

    @property
    def spread(self) -> float:
        return self.ask - self.bid

import tracemalloc
tracemalloc.start()
points = [PricePoint(i * 0.001, 100.0 + i, 100.1 + i) for i in range(100_000)]
snap   = tracemalloc.take_snapshot()
print(snap.statistics("lineno")[0].size // 1024, "KB")
```

---

**Scenario 5: A developer uses `self.MAX = 100` inside __init__ instead of `MAX = 100` at class level. What are the consequences?**

**Q:** Explain the differences.

**A:** Setting `self.MAX = 100` in `__init__` creates an instance variable. Each instance stores its own copy, wasting memory. The intent (a constant shared value) is communicated more clearly and efficiently with a class variable. Additionally, accessing `Class.MAX` would raise `AttributeError` since it's not in the class dict.

```python
class Wrong:
    def __init__(self):
        self.MAX = 100   # stored per instance, wastes memory, not accessible as Class.MAX

class Right:
    MAX: int = 100       # stored once in class, accessible everywhere

# Right.MAX  → 100
# Wrong.MAX  → AttributeError
```

---

**Scenario 6: You use __slots__ in a base class but a subclass adds new attributes without declaring __slots__. What happens?**

**Q:** Explain the behaviour.

**A:** If a subclass doesn't declare `__slots__`, it automatically gets `__dict__`, which restores the ability to add arbitrary attributes. The parent's slots still exist (as descriptors on the parent class), but the memory savings are partially lost because the subclass instances now have both the parent's slots AND a `__dict__`.

```python
class Slotted:
    __slots__ = ("x",)

class Unslotted(Slotted):
    pass   # no __slots__ declaration

u = Unslotted()
u.x = 1      # uses parent's slot descriptor
u.z = 99     # uses __dict__ from Unslotted
print(hasattr(u, "__dict__"))  # True — savings partially lost
```

---

**Scenario 7: How do you make a class variable read-only (prevent instances from shadowing it)?**

**Q:** Implement a read-only class variable.

**A:** Use a `property` with only a getter at the class level, via a custom metaclass or a `classproperty` descriptor:

```python
class classproperty:
    def __init__(self, func):
        self._func = func
    def __get__(self, obj, objtype=None):
        return self._func(objtype)

class Config:
    _VERSION = "1.0.0"

    @classproperty
    def VERSION(cls) -> str:
        return cls._VERSION

print(Config.VERSION)   # "1.0.0"
c = Config()
print(c.VERSION)        # "1.0.0"
# c.VERSION = "2.0.0"  # AttributeError — no __set__ defined
```

---

**Scenario 8: You are writing a dataclass and need one field to be shared across all instances and another to be per-instance. How do you annotate them?**

**Q:** Show correct dataclass annotation.

**A:**

```python
from dataclasses import dataclass, field
from typing import ClassVar

@dataclass
class Server:
    MAX_WORKERS: ClassVar[int] = 8    # class variable — not in __init__
    host: str                          # instance variable
    port: int = 8080                   # instance variable with default
    tags: list[str] = field(default_factory=list)   # per-instance mutable

s1 = Server("localhost")
s2 = Server("0.0.0.0", 9090)
print(s1.MAX_WORKERS)          # 8
print(Server.MAX_WORKERS)      # 8
print(s1.tags is s2.tags)      # False — separate lists
```

---

**Scenario 9: Two threads simultaneously create objects and increment a class variable counter. Describe the race condition and fix it.**

**Q:** Diagnose and fix.

**A:** `cls.count += 1` is three bytecodes: LOAD, BINARY_ADD, STORE. The GIL can release between any of these, allowing another thread to observe a stale value. Fix with a lock:

```python
import threading

class SafeCounter:
    _count = 0
    _lock  = threading.Lock()

    def __init__(self) -> None:
        with SafeCounter._lock:
            SafeCounter._count += 1

    @classmethod
    def count(cls) -> int:
        with cls._lock:
            return cls._count

def make_many() -> None:
    for _ in range(500):
        SafeCounter()

threads = [threading.Thread(target=make_many) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(SafeCounter.count())  # 2000
```

---

**Scenario 10: You need to add type annotations to an existing class that mixes class and instance variables. How do you correctly annotate it?**

**Q:** Show the annotated version.

**A:**

```python
from typing import ClassVar

class Cache:
    # Before annotation, these were ambiguous:
    # max_size = 128
    # _store = {}

    max_size: ClassVar[int]           = 128   # shared constant
    _instances: ClassVar[int]         = 0     # shared counter
    _store: ClassVar[dict[str, object]] = {}   # shared cache (intentional)

    key: str         # instance variable
    hits: int        # instance variable

    def __init__(self, key: str) -> None:
        self.key  = key
        self.hits = 0
        Cache._instances += 1

    @classmethod
    def get(cls, key: str) -> object | None:
        return cls._store.get(key)
```

---

## 4. Quick Recap / Cheatsheet

- **Instance variable**: lives in `instance.__dict__`, unique per object. Set via `self.x = value` in `__init__`.
- **Class variable**: lives in `Class.__dict__`, shared across all instances. Defined directly in class body.
- **Shadow effect**: `self.x = val` where `x` is a class variable creates a *new* instance variable, does NOT modify the class variable.
- **`self.x += 1` trap**: reads class var, writes instance var — class var untouched.
- **Mutable class variable bug**: `items = []` at class level → all instances share the same list. Fix: use `__init__`.
- **Class variable update rule**: always use `ClassName.var = ...` or `cls.var = ...` inside `@classmethod`.
- **`__slots__`**: replaces `__dict__` with fixed descriptors — saves memory, faster access, prevents arbitrary attribute creation.
- **`ClassVar[T]`** from `typing`: hints to type checkers that a variable is class-level; no runtime effect.
- **Inheritance**: class variables are found via MRO; assigning on a subclass creates a new entry in subclass `__dict__`.
- **Thread safety**: class variable mutation requires a lock — the GIL does not make `+=` atomic.
- **`dataclass` + `ClassVar`**: fields annotated with `ClassVar` are excluded from `__init__`, `__repr__`, and `__eq__`.
- **Registry pattern**: class variables `{}` + `__init_subclass__` enable elegant plugin/subclass registries.
