# __slots__

## 1. Concept (Deep Explanation)

### What `__slots__` Is

By default, every Python instance stores its attributes in a per-instance dictionary (`__dict__`). This dictionary is flexible — you can add any attribute at runtime — but it carries significant memory overhead. `__slots__` is a class-level declaration that replaces this per-instance `__dict__` with a fixed set of **slot descriptors**, reducing memory usage and (slightly) improving attribute access speed.

```python
class Normal:
    def __init__(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

class Slotted:
    __slots__ = ('x', 'y')

    def __init__(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

n = Normal(1, 2)
s = Slotted(1, 2)

print(n.__dict__)           # {'x': 1, 'y': 2}
# print(s.__dict__)         # AttributeError — no __dict__!

s.z = 99                    # AttributeError — can't add new attributes
n.z = 99                    # Works fine — flexible __dict__
```

### CPython Implementation: `member_descriptor` Objects

When CPython processes a class definition with `__slots__`, it does the following for each slot name:

1. Calculates an offset into a fixed-size memory block (rather than a hash table)
2. Creates a `member_descriptor` object (a C-level data descriptor) and stores it in the class `__dict__`
3. Does NOT create a `__dict__` attribute on instances (unless `'__dict__'` is explicitly included in `__slots__`)

```python
class Point:
    __slots__ = ('x', 'y')

    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

# Inspect the slot descriptors:
print(type(Point.__dict__['x']))   # <class 'member_descriptor'>
print(Point.__dict__['x'])         # <member 'x' of 'Point' objects>

# The descriptor knows its offset in the C struct:
import inspect
print(inspect.ismemberdescriptor(Point.__dict__['x']))  # True
```

The `member_descriptor` implements the data descriptor protocol (`__get__`, `__set__`, `__delete__`) and directly accesses a fixed-offset memory location in the instance's C-level struct, bypassing the hash-table lookup of `__dict__`.

### Memory Savings: Exact Numbers

The overhead of a per-instance `__dict__` is substantial:

```python
import sys

class WithDict:
    def __init__(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

class WithSlots:
    __slots__ = ('x', 'y')

    def __init__(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

wd = WithDict(1, 2)
ws = WithSlots(1, 2)

print(sys.getsizeof(wd))          # ~56 bytes (just the object, not __dict__)
print(sys.getsizeof(wd.__dict__)) # ~232 bytes (the dict itself)
print(sys.getsizeof(ws))          # ~56 bytes (object with 2 slots)
# ws has NO __dict__, saving ~200-400 bytes per instance!

# For 1 million instances:
# __dict__ approach:  ~288 MB  (56 + 232 per instance)
# __slots__ approach: ~56 MB   (56 per instance, 2 slots baked in)
# Savings: ~83%!
```

Real-world memory analysis:

```python
import tracemalloc

tracemalloc.start()

# Allocate 100,000 instances
instances_dict = [WithDict(i, i*2) for i in range(100_000)]
snapshot = tracemalloc.take_snapshot()
stats = snapshot.statistics('lineno')
# Shows ~28-30 MB for dict-based

tracemalloc.clear_traces()

instances_slots = [WithSlots(i, i*2) for i in range(100_000)]
snapshot2 = tracemalloc.take_snapshot()
stats2 = snapshot2.statistics('lineno')
# Shows ~5-7 MB for slots-based — significant reduction
```

### Performance: Attribute Access Speed

Slot access is faster than dict lookup because:
1. No hash computation needed — fixed offset into C struct
2. No collision resolution
3. Direct C-level memory read

```python
import timeit

wd = WithDict(1, 2)
ws = WithSlots(1, 2)

dict_time = timeit.timeit(lambda: wd.x, number=10_000_000)
slot_time = timeit.timeit(lambda: ws.x, number=10_000_000)

print(f"Dict access: {dict_time:.3f}s")   # ~0.45s
print(f"Slot access: {slot_time:.3f}s")   # ~0.35s
print(f"Speedup: {dict_time/slot_time:.2f}x")  # ~1.3x faster
```

The speedup is modest (20-30%) but accumulates in tight loops with millions of attribute accesses.

### Restrictions: Cannot Add Arbitrary Attributes

With `__slots__`, the instance has no `__dict__` by default. Attempting to set an undeclared attribute raises `AttributeError`:

```python
class Config:
    __slots__ = ('host', 'port')

    def __init__(self, host: str, port: int) -> None:
        self.host = host
        self.port = port

cfg = Config("localhost", 8080)
cfg.timeout = 30   # AttributeError: 'Config' object has no attribute 'timeout'

# To inspect what slots exist:
print(Config.__slots__)    # ('host', 'port')
```

### Inheritance and `__slots__`: The Critical Gotcha

If a **parent class does not define `__slots__`**, the child will have `__dict__` regardless — making `__slots__` in the child mostly useless for memory savings:

```python
class Base:
    # No __slots__ — has __dict__ per instance
    def __init__(self) -> None:
        self.base_attr = "base"

class Child(Base):
    __slots__ = ('child_attr',)  # Almost pointless! Base still gives __dict__

    def __init__(self) -> None:
        super().__init__()
        self.child_attr = "child"

c = Child()
print(c.__dict__)     # {'base_attr': 'base'} — still exists from Base!
c.anything = "ok"     # WORKS — __dict__ from Base allows arbitrary attrs
```

For `__slots__` to be effective throughout a hierarchy, **every class in the chain must define `__slots__`**:

```python
class SlottedBase:
    __slots__ = ('base_attr',)

    def __init__(self) -> None:
        self.base_attr = "base"

class SlottedChild(SlottedBase):
    __slots__ = ('child_attr',)   # Don't repeat parent slots!

    def __init__(self) -> None:
        super().__init__()
        self.child_attr = "child"

sc = SlottedChild()
# sc.__dict__ doesn't exist — full memory savings!
print(sc.base_attr)    # "base" — inherited slot
print(sc.child_attr)   # "child" — own slot
sc.other = "x"         # AttributeError — no __dict__
```

**Do NOT re-declare parent slots in children** — it creates a duplicate slot descriptor that shadows the parent's without benefit.

### Pitfall: Forgetting `__weakref__`

By default, Python objects support weak references via the `__weakref__` slot. When you define `__slots__`, this slot is not automatically created:

```python
import weakref

class WithDict:
    pass

class WithSlots:
    __slots__ = ('x',)

wd = WithDict()
ref = weakref.ref(wd)   # Works fine

ws = WithSlots()
ref2 = weakref.ref(ws)  # TypeError: cannot create weak reference to 'WithSlots' object
```

Fix: include `'__weakref__'` in `__slots__`:

```python
class ProperSlots:
    __slots__ = ('x', '__weakref__')

ps = ProperSlots()
ref = weakref.ref(ps)   # Works!
```

### `__slots__` with `__dict__`: Hybrid Behaviour

You can include `'__dict__'` in `__slots__` to get both the fixed slots AND a per-instance dict:

```python
class Hybrid:
    __slots__ = ('x', 'y', '__dict__')

    def __init__(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

h = Hybrid(1, 2)
h.x = 10          # uses the slot descriptor — fast
h.extra = "hello"  # uses __dict__ — flexible
print(h.__dict__)  # {'extra': 'hello'} — only non-slot attrs
```

This gives you fast access for declared attributes plus flexibility for optional ones, at the cost of some memory overhead (the `__dict__` still exists).

### `__slots__` with `dataclasses(slots=True)` (Python 3.10+)

Python 3.10 added `slots=True` to `@dataclass`, which automatically generates the `__slots__` declaration:

```python
from dataclasses import dataclass, field

@dataclass(slots=True)
class Vector:
    x: float
    y: float
    z: float = 0.0
    _magnitude: float = field(default=0.0, init=False, repr=False)

    def __post_init__(self) -> None:
        self._magnitude = (self.x**2 + self.y**2 + self.z**2) ** 0.5

v = Vector(3.0, 4.0)
print(v.x)           # 3.0
print(v._magnitude)  # 5.0
# v.extra = "x"      # AttributeError — slots enforced!

import sys
print(sys.getsizeof(v))  # Much smaller than without slots
```

**How `slots=True` works internally**: Python 3.10 creates a new class with `__slots__` set to all annotated fields, then copies methods/classmethods from the original class. This is necessary because `__slots__` cannot be added to a class after creation.

### When to Use `__slots__`

Use `__slots__` when:
1. **High-volume objects** — millions of instances in memory simultaneously (IoT events, point clouds, ML feature vectors, game entities)
2. **Memory-constrained environments** — embedded systems, microcontrollers
3. **Well-defined, stable attribute set** — the object's attributes won't change
4. **Performance-critical** — tight loops with millions of attribute accesses

Do NOT use `__slots__` when:
- The class is rarely instantiated
- You need dynamic attributes (or use `__dict__` in `__slots__` for hybrid)
- Rapid prototyping / the attribute set is still evolving
- You use multiple inheritance with parents that don't define `__slots__`

### Pitfalls Summary

```python
# PITFALL 1: Redefining a slot in a subclass shadows it (but doesn't override it well)
class Parent:
    __slots__ = ('x',)
class Child(Parent):
    __slots__ = ('x', 'y')  # WRONG: creates a second 'x' slot — confusing!
    # Correct: __slots__ = ('y',)  only new slots

# PITFALL 2: __slots__ in class with multiple inheritance can conflict
class A:
    __slots__ = ('x',)
class B:
    __slots__ = ('x',)
class C(A, B):
    __slots__ = ()  # Both A.x and B.x exist — which does 'x' map to?
    # Python allows this but it's confusing; x maps to A.x

# PITFALL 3: class variables vs slots — a class variable with the same name as a slot
class Bad:
    __slots__ = ('x',)
    x = 10  # This replaces the slot descriptor! Now instances can't use x as a slot.

b = Bad()
b.x = 5  # AttributeError — slot descriptor was overwritten by class variable!
```

### Best Practices

- Always include `'__weakref__'` in `__slots__` if instances might be weakly referenced
- In hierarchies, define `__slots__` at every level; never redeclare parent slots
- Use `@dataclass(slots=True)` for modern code — it's cleaner and less error-prone
- Keep `__slots__` a tuple of strings (not a list or set — order doesn't matter functionally, but tuple is conventional)
- Add `'__dict__'` to `__slots__` if you need occasional dynamic attributes alongside fixed ones

---

## 2. Conceptual Interview Questions (10)

**Q1: What exactly is created in the class namespace when you define `__slots__`?**

**A:** For each name in `__slots__`, CPython creates a `member_descriptor` object and stores it in the class `__dict__`. These are C-level data descriptors (implementing `__get__`, `__set__`, `__delete__`) that know the fixed offset of their slot within the instance's C-level struct (a `PyObject` with extra memory for the slots).

```python
class Point:
    __slots__ = ('x', 'y')

print(type(Point.__dict__['x']))  # <class 'member_descriptor'>

# It IS a data descriptor:
import inspect
print(inspect.ismemberdescriptor(Point.__dict__['x']))  # True
print(hasattr(Point.__dict__['x'], '__get__'))   # True
print(hasattr(Point.__dict__['x'], '__set__'))   # True
```

Additionally, CPython does NOT create a `tp_dictoffset` in the C struct (the field that tells Python where to find `__dict__`), so instances have no `__dict__`.

**Q2: Why does `__slots__` not work as expected when a parent class doesn't define it?**

**A:** Because the parent class allocates `__dict__` for each instance (via `tp_dictoffset`), and child instances inherit this allocation. The child's `__slots__` still creates slot descriptors, but the instance also has `__dict__` from the parent. This means:
- The extra slot descriptors work for declared attributes (faster access)
- But arbitrary attributes can still be added via `__dict__`
- Memory savings from removing `__dict__` are NOT achieved

```python
class NaiveBase:
    pass  # has __dict__ per instance

class SlottedChild(NaiveBase):
    __slots__ = ('x',)

sc = SlottedChild()
sc.x = 1        # uses slot — OK
sc.random = 99  # uses inherited __dict__ — still works!
print(sc.__dict__)  # {'random': 99}
```

**Q3: What happens to `__hash__` and `__weakref__` when you use `__slots__`?**

**A:** `__weakref__` is a slot that enables weak references. When `__slots__` is defined and `'__weakref__'` is not included, the type does not allocate the weakref slot, making instances non-weakly-referenceable.

`__hash__` is not affected by `__slots__` — it's a method, not a slot. However, if you define `__eq__`, Python sets `__hash__ = None` regardless of slots.

```python
import weakref

class NoWeakRef:
    __slots__ = ('x',)

class WithWeakRef:
    __slots__ = ('x', '__weakref__')

nwr = NoWeakRef()
wwr = WithWeakRef()

try:
    weakref.ref(nwr)
except TypeError as e:
    print(e)   # cannot create weak reference to 'NoWeakRef' object

ref = weakref.ref(wwr)   # Works!
```

**Q4: Can you use `__slots__` with multiple inheritance?**

**A:** Yes, but carefully. If multiple parent classes each define a non-empty `__slots__`, Python issues no error, but slot layout can be complex. The safest patterns are:

1. Multiple inheritance where only one parent has non-empty `__slots__` — other parents use empty `__slots__ = ()`
2. Mix-in classes that define `__slots__ = ()` (no slots, preventing `__dict__`)

```python
class Mixin:
    __slots__ = ()   # Empty — no new slots, but also no __dict__

    def describe(self) -> str:
        return repr(self)

class Base:
    __slots__ = ('x',)

class Combined(Mixin, Base):
    __slots__ = ('y',)

    def __init__(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

c = Combined(1, 2)
print(c.describe())  # works — mixin method
print(c.x, c.y)     # works — inherited and own slots
```

**Q5: How do `__slots__` interact with `__init_subclass__` and class decorators?**

**A:** `__init_subclass__` and class decorators run after the class is created. By this point, `__slots__` has already been processed by the metaclass (`type.__new__`). Adding new slot-like attributes after class creation is not possible — `__slots__` is a one-time setup at class creation.

Class decorators that add attributes to the class dict (like `@dataclass` does) need to be careful: adding a class variable with the same name as a slot will overwrite the slot descriptor.

```python
def add_method(cls):
    def extra(self) -> str:
        return f"extra on {self.x}"
    cls.extra = extra
    return cls

@add_method
class Slotted:
    __slots__ = ('x',)

    def __init__(self, x: int) -> None:
        self.x = x

s = Slotted(42)
print(s.extra())   # "extra on 42" — method addition is fine
```

But adding a *class variable* that shadows a slot:
```python
Slotted.x = "shadowed"   # Replaces slot descriptor with string — BAD
```

**Q6: How does `@dataclass(slots=True)` implement slots internally?**

**A:** Because `__slots__` must be set at class creation and cannot be added retroactively, `@dataclass(slots=True)` cannot simply add `__slots__` to the decorated class. Instead, it:

1. Collects all annotated fields
2. Creates a **new class** with `__slots__` set to those field names
3. Copies all methods, class variables, and decorators from the original class to the new one
4. Returns the new class (replacing the original)

```python
from dataclasses import dataclass

@dataclass(slots=True)
class Config:
    host: str
    port: int = 8080

# Verify:
print(Config.__slots__)         # ('host', 'port')
cfg = Config("localhost")
# cfg.extra = "x"               # AttributeError
print(hasattr(cfg, '__dict__')) # False
```

**Q7: What is the difference in memory between `__slots__` and `__dict__`-based instances when many attributes are stored?**

**A:** The break-even point depends on the number of attributes. `__dict__` has a fixed overhead (~200-300 bytes) regardless of how many attributes are stored (up to the initial allocation). For very many attributes, `__dict__` may actually be more memory-efficient per-attribute than expanding a slotted layout, because `__dict__` uses a hash table that amortises well.

For typical small objects (2-10 attributes), `__slots__` wins significantly. For objects with 50+ attributes, the advantage diminishes.

```python
import sys

class Many:
    __slots__ = tuple(f'attr_{i}' for i in range(50))

class ManyDict:
    pass

m = Many()
md = ManyDict()
for i in range(50):
    setattr(m, f'attr_{i}', i)
    setattr(md, f'attr_{i}', i)

print(sys.getsizeof(m))           # ~456 bytes (50 slots * ~8 bytes + header)
print(sys.getsizeof(md))          # ~56 bytes (object shell)
print(sys.getsizeof(md.__dict__)) # ~2264 bytes (dict with 50 entries)
```

**Q8: Can `__slots__` hold mutable values?**

**A:** Yes — `__slots__` controls *where* values are stored (slot vs `__dict__`), not what *types* of values are allowed. You can store any Python object in a slot, including mutable ones:

```python
class Container:
    __slots__ = ('items', 'metadata')

    def __init__(self) -> None:
        self.items = []        # mutable list — fine!
        self.metadata = {}     # mutable dict — fine!

c = Container()
c.items.append(1)
c.metadata['key'] = 'value'
print(c.items)     # [1]
print(c.metadata)  # {'key': 'value'}
```

**Q9: How do you safely delete a slot value?**

**A:** Use `del obj.attr` — the slot descriptor's `__delete__` is called, which sets the slot to the "empty" sentinel (similar to C's `NULL`). Accessing an unset slot raises `AttributeError`, just like accessing an undefined attribute:

```python
class Node:
    __slots__ = ('value', 'next')

    def __init__(self, value: int) -> None:
        self.value = value
        # 'next' is NOT set — its slot is "empty"

n = Node(1)
try:
    print(n.next)    # AttributeError — slot never set
except AttributeError as e:
    print(e)

n.next = Node(2)
del n.next
try:
    print(n.next)    # AttributeError — slot deleted
except AttributeError:
    print("Slot is empty after deletion")
```

**Q10: When would you use `__slots__` with `__dict__` included?**

**A:** Include `'__dict__'` in `__slots__` when you want:
- Fast access for a *known, hot* set of attributes (via slots)
- Flexibility to add arbitrary attributes for less frequent or optional data (via `__dict__`)

This is a hybrid pattern useful in frameworks where core attributes are known upfront but plugins or extensions may add custom data:

```python
class FlexibleModel:
    __slots__ = ('id', 'name', '__dict__', '__weakref__')

    def __init__(self, id: int, name: str) -> None:
        self.id = id     # slot — fast, no __dict__ entry
        self.name = name  # slot — fast

m = FlexibleModel(1, "Alice")
m.extra_field = "plugin data"   # goes to __dict__
print(m.__dict__)               # {'extra_field': 'plugin data'}
print(m.id)                     # 1 — uses slot, not __dict__
```

---

## 3. Scenario-Based Interview Questions (10)

**S1: You are building a spatial indexing system that stores millions of 3D points. Memory is critical. Design the Point class.**

**Q:** Implement memory-efficient 3D point.

**A:**

```python
import sys
from dataclasses import dataclass

@dataclass(slots=True, frozen=True)
class Point3D:
    x: float
    y: float
    z: float

    def distance_to(self, other: "Point3D") -> float:
        return ((self.x-other.x)**2 + (self.y-other.y)**2 + (self.z-other.z)**2) ** 0.5

    def __add__(self, other: "Point3D") -> "Point3D":
        return Point3D(self.x+other.x, self.y+other.y, self.z+other.z)


# Memory comparison:
@dataclass
class Point3D_Dict:
    x: float
    y: float
    z: float

p_slot = Point3D(1.0, 2.0, 3.0)
p_dict = Point3D_Dict(1.0, 2.0, 3.0)

print(sys.getsizeof(p_slot))           # ~56 bytes
print(sys.getsizeof(p_dict))           # ~56 bytes (shell)
print(sys.getsizeof(p_dict.__dict__))  # ~232 bytes (dict overhead)

# For 10 million points:
# slots: ~560 MB
# dict:  ~2.88 GB — 5x more!

pts = [Point3D(float(i), float(i), float(i)) for i in range(100_000)]
print(f"Created {len(pts):,} points")
print(f"Sample: {pts[0].distance_to(pts[1]):.4f}")
```

**S2: A game has thousands of Entity objects, each with position, velocity, and health. The game loop accesses these attributes millions of times per second. Show the impact of `__slots__`.**

**Q:** Benchmark with and without slots.

**A:**

```python
import timeit

class EntityDict:
    def __init__(self, x: float, y: float, vx: float, vy: float, hp: int) -> None:
        self.x = x
        self.y = y
        self.vx = vx
        self.vy = vy
        self.hp = hp

    def update(self, dt: float) -> None:
        self.x += self.vx * dt
        self.y += self.vy * dt

class EntitySlot:
    __slots__ = ('x', 'y', 'vx', 'vy', 'hp')

    def __init__(self, x: float, y: float, vx: float, vy: float, hp: int) -> None:
        self.x = x
        self.y = y
        self.vx = vx
        self.vy = vy
        self.hp = hp

    def update(self, dt: float) -> None:
        self.x += self.vx * dt
        self.y += self.vy * dt


N = 10_000
entities_dict = [EntityDict(float(i), float(i), 1.0, 1.0, 100) for i in range(N)]
entities_slot = [EntitySlot(float(i), float(i), 1.0, 1.0, 100) for i in range(N)]

def run_dict():
    for e in entities_dict:
        e.update(0.016)

def run_slot():
    for e in entities_slot:
        e.update(0.016)

dict_time = timeit.timeit(run_dict, number=1000)
slot_time = timeit.timeit(run_slot, number=1000)

print(f"Dict: {dict_time:.3f}s")
print(f"Slot: {slot_time:.3f}s")
print(f"Speedup: {dict_time/slot_time:.2f}x")
```

**S3: You are implementing a linked list node. Should it use `__slots__`?**

**Q:** Analyse and implement.

**A:**

```python
import sys

class NodeDict:
    def __init__(self, value: int) -> None:
        self.value = value
        self.next: "NodeDict | None" = None

class NodeSlot:
    __slots__ = ('value', 'next', '__weakref__')

    def __init__(self, value: int) -> None:
        self.value = value
        self.next: "NodeSlot | None" = None


# A linked list with 1 million nodes:
# NodeDict: ~1M * (56 + 232) = ~288 MB
# NodeSlot: ~1M * 56 = ~56 MB (just the object, no dict)

# Build and measure:
def build_list_dict(n: int) -> NodeDict:
    head = NodeDict(0)
    curr = head
    for i in range(1, n):
        curr.next = NodeDict(i)
        curr = curr.next
    return head

def build_list_slot(n: int) -> NodeSlot:
    head = NodeSlot(0)
    curr = head
    for i in range(1, n):
        curr.next = NodeSlot(i)
        curr = curr.next
    return head

n_dict = NodeDict(42)
n_slot = NodeSlot(42)
n_slot.next = NodeSlot(43)

print(sys.getsizeof(n_dict) + sys.getsizeof(n_dict.__dict__))  # ~288 bytes
print(sys.getsizeof(n_slot))                                    # ~56 bytes

# Decision: YES, use __slots__ for nodes — they're created in huge quantities
# and have a fixed, well-known structure.
```

**S4: A developer defines `__slots__` in a subclass but not the parent. They are surprised that arbitrary attributes can still be added. Explain and fix.**

**Q:** Debug the issue.

**A:**

```python
# THE PROBLEM:
class Animal:
    def __init__(self, name: str) -> None:
        self.name = name   # goes to __dict__

class Dog(Animal):
    __slots__ = ('breed',)  # Useless! Animal still provides __dict__

    def __init__(self, name: str, breed: str) -> None:
        super().__init__(name)
        self.breed = breed

d = Dog("Rex", "Labrador")
d.random_attr = "still works!"   # Shouldn't work, but does!
print(d.__dict__)  # {'name': 'Rex', 'random_attr': 'still works!'}
print(d.breed)     # 'Labrador' (uses the slot descriptor)

# THE FIX: Add __slots__ to the parent too
class AnimalFixed:
    __slots__ = ('name', '__weakref__')

    def __init__(self, name: str) -> None:
        self.name = name

class DogFixed(AnimalFixed):
    __slots__ = ('breed',)   # Only NEW slots — don't repeat 'name'

    def __init__(self, name: str, breed: str) -> None:
        super().__init__(name)
        self.breed = breed

df = DogFixed("Rex", "Labrador")
df.random_attr = "fails now"   # AttributeError — no __dict__
print(df.name)     # 'Rex'
print(df.breed)    # 'Labrador'
```

**S5: Implement a `Record` class using `__slots__` that acts like a lightweight named tuple with mutable fields.**

**Q:** Design a mutable, memory-efficient record.

**A:**

```python
from __future__ import annotations
import sys

class Record:
    """Mutable, memory-efficient record using __slots__."""
    __slots__ = ('_fields', '_values')

    def __init__(self, **kwargs) -> None:
        # Store field names and values as tuples (immutable — fast)
        object.__setattr__(self, '_fields', tuple(kwargs.keys()))
        object.__setattr__(self, '_values', list(kwargs.values()))

    def __getattr__(self, name: str):
        try:
            idx = self._fields.index(name)
            return self._values[idx]
        except ValueError:
            raise AttributeError(f"No field {name!r}")

    def __setattr__(self, name: str, value) -> None:
        if name.startswith('_'):
            object.__setattr__(self, name, value)
            return
        try:
            idx = self._fields.index(name)
            self._values[idx] = value
        except ValueError:
            raise AttributeError(f"Cannot add new field {name!r}")

    def __repr__(self) -> str:
        fields = ", ".join(f"{k}={v!r}" for k, v in zip(self._fields, self._values))
        return f"Record({fields})"


r = Record(name="Alice", age=30, score=95.5)
print(r)           # Record(name='Alice', age=30, score=95.5)
print(r.name)      # Alice
r.age = 31
print(r.age)       # 31
r.new_field = "x"  # AttributeError — can't add new fields
print(sys.getsizeof(r))  # Much smaller than a dict-based approach
```

**S6: Show how `__slots__` can break `pickle` and how to fix it.**

**Q:** Diagnose and fix pickle incompatibility.

**A:**

```python
import pickle

class Broken:
    __slots__ = ('x', 'y')

    def __init__(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

b = Broken(1, 2)
try:
    data = pickle.dumps(b)
    b2 = pickle.loads(data)
    print(b2.x)    # Actually works in Python 3 for simple cases!
except Exception as e:
    print(f"Error: {e}")

# Pickle CAN handle __slots__ in modern Python, BUT needs __getstate__/__setstate__
# for complex cases (inheritance, mix of slots and __dict__):

class FixedPickle:
    __slots__ = ('x', 'y')

    def __init__(self, x: int, y: int) -> None:
        self.x = x
        self.y = y

    def __getstate__(self) -> dict:
        return {'x': self.x, 'y': self.y}

    def __setstate__(self, state: dict) -> None:
        self.x = state['x']
        self.y = state['y']

fp = FixedPickle(10, 20)
data = pickle.dumps(fp)
fp2 = pickle.loads(data)
print(fp2.x, fp2.y)   # 10, 20
```

**S7: Can `__slots__` be used with `@dataclass`? Show the evolution from Python 3.9 to 3.10.**

**Q:** Implement slots dataclass.

**A:**

```python
# Python 3.9 and earlier — manual workaround needed:
from dataclasses import dataclass

@dataclass
class OldStyle:
    x: float
    y: float

    def __init_subclass__(cls, **kwargs):
        pass

# Manually add slots (Python 3.9 style):
OldStyle.__slots__ = ('x', 'y')
# This doesn't actually work properly — slots need to be set at class creation!

# Python 3.10+ — clean and correct:
from dataclasses import dataclass, field

@dataclass(slots=True, frozen=True)
class Point3D_Modern:
    x: float
    y: float
    z: float = 0.0
    tags: list[str] = field(default_factory=list)

p = Point3D_Modern(1.0, 2.0, 3.0, tags=["a", "b"])
print(p.x)          # 1.0
print(p.__slots__)  # ('x', 'y', 'z', 'tags')

# frozen=True + slots=True = fully immutable and memory-efficient
try:
    p.x = 99.0   # FrozenInstanceError
except Exception as e:
    print(e)

import sys
print(sys.getsizeof(p))   # Compact!
```

**S8: Implement a `WeakValueCache` that holds weak references to slotted objects.**

**Q:** Use weak references with slotted objects.

**A:**

```python
import weakref
from typing import TypeVar, Generic

T = TypeVar('T')

class CacheableObject:
    """Must include __weakref__ in __slots__ for weak reference support."""
    __slots__ = ('key', 'data', '__weakref__')

    def __init__(self, key: str, data) -> None:
        self.key = key
        self.data = data

    def __repr__(self) -> str:
        return f"CacheableObject({self.key!r})"


class WeakValueCache(Generic[T]):
    """Cache that doesn't prevent garbage collection of values."""

    def __init__(self) -> None:
        self._store: dict[str, weakref.ref] = {}

    def put(self, key: str, obj: T) -> None:
        self._store[key] = weakref.ref(obj, lambda ref: self._remove(key))

    def get(self, key: str) -> T | None:
        ref = self._store.get(key)
        if ref is None:
            return None
        return ref()   # dereference — returns None if GC'd

    def _remove(self, key: str) -> None:
        self._store.pop(key, None)
        print(f"Evicted {key!r} from cache (GC'd)")


cache: WeakValueCache[CacheableObject] = WeakValueCache()

obj = CacheableObject("key1", {"important": "data"})
cache.put("key1", obj)
print(cache.get("key1"))   # CacheableObject('key1')

del obj   # reference count drops
import gc
gc.collect()
print(cache.get("key1"))   # None — evicted!
```

**S9: A developer says "I'll just add `__slots__` to improve performance" without any analysis. What risks does this introduce?**

**Q:** Identify risks and provide guidance.

**A:**

```python
# RISK 1: Breaking dynamic attribute assignment (common in testing/mocking)
class Service:
    __slots__ = ('_db',)

    def __init__(self, db) -> None:
        self._db = db

# In tests:
# service = Service(mock_db)
# service.mock_extra = "something"  # AttributeError! Can't mock easily

# RISK 2: Breaking inheritance with __dict__-based parent
class ThirdPartyBase:
    pass  # has __dict__

class MyClass(ThirdPartyBase):
    __slots__ = ('x',)  # __dict__ still exists from ThirdPartyBase!
    # Memory savings: NONE

# RISK 3: Forgetting __weakref__ breaks libraries that use weak refs
import weakref
class Bad:
    __slots__ = ('data',)

b = Bad()
try:
    weakref.ref(b)
except TypeError as e:
    print(f"Weak ref failed: {e}")

# RISK 4: __slots__ with __dict__ class variable collision
class Confusing:
    __slots__ = ('status',)
    status = "class-level"  # This OVERWRITES the slot descriptor!

c = Confusing()
try:
    c.status = "instance"   # AttributeError — no slot! class var replaced it
except AttributeError as e:
    print(f"Slot overwritten: {e}")

# GUIDANCE: Profile memory first (tracemalloc), then add __slots__ only to
# hot paths. Always include __weakref__. Test with your framework/ORM.
```

**S10: Implement a high-performance event system where millions of events are created per second.**

**Q:** Design memory-efficient events with `__slots__`.

**A:**

```python
from __future__ import annotations
from dataclasses import dataclass, field
from datetime import datetime
from typing import Any

@dataclass(slots=True)
class Event:
    """Base event — slotted for memory efficiency."""
    event_type: str
    timestamp: float
    source: str

@dataclass(slots=True)
class ClickEvent(Event):
    x: int
    y: int
    button: str = "left"

@dataclass(slots=True)
class KeyEvent(Event):
    key: str
    modifiers: tuple[str, ...] = field(default_factory=tuple)

@dataclass(slots=True)
class NetworkEvent(Event):
    host: str
    port: int
    bytes_transferred: int
    success: bool = True


import sys
import time

# Benchmark event creation:
def benchmark_creation(n: int = 100_000) -> None:
    t0 = time.perf_counter()
    events = [
        ClickEvent(
            event_type="click",
            timestamp=time.time(),
            source="mouse",
            x=100,
            y=200,
        )
        for _ in range(n)
    ]
    t1 = time.perf_counter()

    print(f"Created {n:,} events in {(t1-t0)*1000:.1f}ms")
    print(f"Per-event size: {sys.getsizeof(events[0])} bytes")
    print(f"Total list size: {sys.getsizeof(events):,} bytes")

benchmark_creation()
```

---

## 4. Quick Recap / Cheatsheet

- **`__slots__`** replaces per-instance `__dict__` with fixed C-level slot descriptors (`member_descriptor`)
- **Memory savings**: ~200-400 bytes per instance (no `__dict__`); crucial for millions of instances
- **Speed**: slot access is ~20-30% faster than dict lookup (no hashing, direct C-level offset)
- **Restriction**: cannot add arbitrary attributes; `AttributeError` on undeclared names
- **Always include `'__weakref__'`** if instances need to support `weakref.ref()`
- **Inheritance**: ALL classes in the hierarchy must define `__slots__` for savings to apply
- **Never redeclare parent slots** in subclass — creates duplicate descriptors
- **`'__dict__'` in `__slots__`**: hybrid mode — slots for hot attrs, `__dict__` for occasional dynamic ones
- **`@dataclass(slots=True)`** (Python 3.10+): automatic and correct way to use slots with dataclasses
- **Breaks**: mocking with instance-level attributes, dynamic attribute assignment, `__dict__`-based serialisation
- **Pickle**: works in Python 3 for simple cases; add `__getstate__`/`__setstate__` for robustness
- **When to use**: millions of instances, fixed attribute set, performance-critical code, embedded systems
- **When NOT to use**: prototyping, small instance counts, hierarchies with uncontrolled parents, dynamic systems
- **Checking**: `hasattr(obj, '__dict__')` — `False` means slots-only; `obj.__slots__` to see declared slots
- **`@dataclass(slots=True, frozen=True)`**: fully immutable + memory-efficient — best of both worlds
