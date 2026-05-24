# MRO and super()

## 1. Concept (Deep Explanation)

### The Method Resolution Order (MRO)

When Python looks up an attribute on an instance, it must search through the class hierarchy to find it. In a single-inheritance chain, this is trivial — search the class, then its parent, then the grandparent, etc. In **multiple inheritance**, the order in which classes are searched is the **Method Resolution Order (MRO)**.

Python 2.2 used a depth-first, left-to-right search (DFS). This had a known flaw with diamond inheritance: the common base appeared at different positions for different branches, producing inconsistent results. Python 2.3 replaced DFS with the **C3 Linearisation Algorithm**, which remains in use today.

### C3 Linearisation: Step-by-Step

The C3 algorithm computes a **linearisation** (a flat, ordered list) of a class hierarchy. The notation is:

```
L(C) = C + merge(L(B1), L(B2), ..., [B1, B2, ...])
```

Where `B1, B2, ...` are the direct bases, and `merge` picks the first element of each list whose value does not appear in the *tail* (any position except the first) of any other list.

#### Full Diamond Example

```
    O (object)
   / \
  A   B
   \ /
    C
   / \
  D   E
   \ /
    F
```

```python
class O: pass           # object
class A(O): pass
class B(O): pass
class C(A, B): pass
class D(C): pass
class E(C): pass
class F(D, E): pass

print(F.__mro__)
# (<class 'F'>, <class 'D'>, <class 'E'>, <class 'C'>,
#  <class 'A'>, <class 'B'>, <class 'O'>, <class 'object'>)
```

Let's trace the C3 merge for `C(A, B)` first:

```
L(O) = [O, object]
L(A) = [A] + merge(L(O), [O]) = [A, O, object]
L(B) = [B] + merge(L(O), [O]) = [B, O, object]

L(C) = [C] + merge(L(A), L(B), [A, B])
     = [C] + merge([A, O, object], [B, O, object], [A, B])

Step 1: Head=A. A in tail of [B, O, object]? No. A in tail of [A, B]? No (it's head). → Take A.
        Remaining: merge([O, object], [B, O, object], [B])

Step 2: Head=O. O in tail of [B, O, object]? Yes (position 1). Skip.
        Next list head=B. B in tail of [O, object]? No. B in tail of [B]? No. → Take B.
        Remaining: merge([O, object], [O, object], [])

Step 3: Head=O. O not in any tail. → Take O.
        Remaining: merge([object], [object], [])

Step 4: Take object.

L(C) = [C, A, B, O, object]
```

Now `F(D, E)`:

```
L(D) = [D, C, A, B, O, object]
L(E) = [E, C, A, B, O, object]

L(F) = [F] + merge([D,C,A,B,O,object], [E,C,A,B,O,object], [D,E])

Step 1: Head=D. D not in any tail. → Take D.
Step 2: Head=E (from second list, D consumed). E not in any tail. → Take E.
         ... continuing: C, A, B, O, object
```

Result matches what Python computes: `[F, D, E, C, A, B, O, object]`.

#### When C3 Fails (Inconsistent Hierarchy)

C3 will raise `TypeError` if the hierarchy is inconsistent (cannot be linearised without violating local precedence):

```python
class X: pass
class Y: pass
class A(X, Y): pass
class B(Y, X): pass

try:
    class C(A, B): pass   # TypeError: Cannot create a consistent MRO
except TypeError as e:
    print(e)
# A says X before Y; B says Y before X — irreconcilable contradiction
```

### Inspecting the MRO

```python
class Mixin: pass
class Base: pass
class Child(Mixin, Base): pass

# Three ways to inspect MRO:
print(Child.__mro__)          # tuple — most common
print(Child.mro())            # list — method, same content
print(type.mro(Child))        # also works, same as Child.mro()

# Useful in debugging:
for i, cls in enumerate(Child.__mro__):
    print(f"{i}: {cls.__name__}")
```

### super(): The Proxy Object

`super()` does NOT return the parent class. It returns a **proxy object** that intercepts attribute lookup and begins searching the MRO *after* the class specified.

#### Two Forms of super()

**Zero-argument form (Python 3, recommended):**
```python
class Child(Parent):
    def method(self):
        return super().method()   # Python 3 magic: __class__ cell variable
```

**Explicit form:**
```python
class Child(Parent):
    def method(self):
        return super(Child, self).method()   # equivalent to zero-arg in most cases
```

The zero-argument form works via a **`__class__` cell variable** — a closure implicitly created by CPython when `super()` (or `__class__`) appears inside a method body. The `__class__` cell holds the *class where the method was defined*, not `type(self)`.

```python
class Child(Parent):
    def method(self):
        print(__class__)          # <class '__main__.Child'>
        print(type(self))         # could be a further subclass!
        return super().method()
```

This is why `super()` works correctly even in deep hierarchies where `type(self)` ≠ the class that defines the method.

#### super() in a Classmethod

```python
class Base:
    @classmethod
    def build(cls) -> "Base":
        obj = cls.__new__(cls)
        return obj

class Child(Base):
    @classmethod
    def build(cls) -> "Child":
        obj = super().build()   # cls is propagated correctly
        obj.extra = True
        return obj

c = Child.build()
print(type(c))     # <class '__main__.Child'>
print(c.extra)     # True
```

#### super() in a Staticmethod

`super()` inside a `@staticmethod` cannot use the zero-argument form (no `__class__` cell). Use the explicit form:

```python
class Base:
    @staticmethod
    def helper() -> str:
        return "Base helper"

class Child(Base):
    @staticmethod
    def helper() -> str:
        # super().helper()  ← RuntimeError: zero-argument super is not allowed in static methods
        return super(Child, Child).helper() + " + Child"  # or just Base.helper()
```

#### super() with __new__

```python
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

s1 = Singleton()
s2 = Singleton()
print(s1 is s2)   # True
```

### Cooperative Multiple Inheritance

The full power of `super()` emerges in **cooperative multiple inheritance**, where every class in the hierarchy calls `super()`. The MRO ensures each class is called exactly once:

```python
class LogMixin:
    def __init__(self, **kwargs: object) -> None:
        print(f"LogMixin.__init__ called, kwargs={kwargs}")
        super().__init__(**kwargs)

class TimestampMixin:
    def __init__(self, **kwargs: object) -> None:
        print(f"TimestampMixin.__init__ called")
        super().__init__(**kwargs)
        import time
        self.timestamp = time.time()

class Base:
    def __init__(self, name: str = "", **kwargs: object) -> None:
        print(f"Base.__init__ called, name={name!r}")
        super().__init__(**kwargs)
        self.name = name

class MyClass(LogMixin, TimestampMixin, Base):
    def __init__(self, name: str) -> None:
        super().__init__(name=name)

# MRO: MyClass → LogMixin → TimestampMixin → Base → object
obj = MyClass("Alice")
# Prints:
# LogMixin.__init__ called, kwargs={'name': 'Alice'}
# TimestampMixin.__init__ called
# Base.__init__ called, name='Alice'
```

### Common Pitfalls

**Pitfall 1 — Wrong arguments to explicit super():**
```python
class A:
    def method(self): return "A"

class B(A):
    def method(self):
        # Wrong: super(A, self) skips B and goes to A's parent (object)
        return super(A, self).method()   # searches from AFTER A in the MRO

b = B()
print(b.method())   # AttributeError or unexpected: skips A, finds object.method
```

**Pitfall 2 — Forgetting super() in one class breaks cooperative chain:**
```python
class A:
    def __init__(self, **kw): super().__init__(**kw); self.a = 1

class B(A):
    def __init__(self, **kw):
        # Forgot super().__init__(**kw) — A.__init__ never runs!
        self.b = 2

class C(B, A):
    def __init__(self): super().__init__()

c = C()
print(hasattr(c, "a"))   # False — A.__init__ skipped!
print(hasattr(c, "b"))   # True
```

**Pitfall 3 — super() in module-level functions (not methods):**
```python
# super() only works inside method bodies (needs __class__ cell):
def standalone():
    return super()   # RuntimeError: super() used outside of a class body
```

---

## 2. Conceptual Interview Questions (10)

**Q1: What is the MRO and why does Python use C3 linearisation instead of depth-first search?**

**A:** The MRO defines the order in which Python searches classes for attributes and methods. C3 linearisation was chosen over DFS because DFS produces inconsistent orderings in diamond inheritance hierarchies — specifically, it could visit the same class multiple times and produce different results depending on the inheritance order.

C3 guarantees three properties:
1. **Local precedence**: if A is listed before B in the base class list, A appears before B in the MRO.
2. **Monotonicity**: if A before B in the MRO of C, then A before B in the MRO of any subclass of C.
3. **Consistency**: common ancestors appear after all their descendants.

```python
class X: pass
class Y(X): pass
class Z(X): pass
class W(Y, Z): pass
print(W.__mro__)  # W → Y → Z → X → object
```

---

**Q2: What does super() actually return?**

**A:** `super()` returns a **proxy object** (an instance of the built-in `super` type). It does not return the parent class. The proxy overrides `__getattribute__` to search the MRO starting from the class *after* the one specified in `super()`, using the MRO of the actual instance (`self`).

```python
class A:
    def hello(self): return "A"

class B(A):
    def hello(self):
        s = super()
        print(type(s))          # <class 'super'>
        print(s.__thisclass__)  # <class '__main__.B'>
        print(s.__self_class__) # <class '__main__.B'> (or subclass if called on subclass instance)
        return s.hello()

print(B().hello())   # "A"
```

---

**Q3: How does the zero-argument super() form work internally?**

**A:** When Python compiles a method that uses `super()` or `__class__`, it creates an implicit **closure cell** variable called `__class__` that holds the class in which the method is defined. `super()` with no arguments reads `__class__` from this cell and `self` (or `cls`) from the first argument.

```python
import dis

class Foo:
    def bar(self):
        return super()

# The bytecode will show LOAD_DEREF for __class__:
dis.dis(Foo.bar)
# ... LOAD_DEREF __class__ ...
# ... LOAD_FAST self ...
# ... CALL super() ...
```

This is why `super()` works correctly even when the method is inherited by a subclass — `__class__` always refers to the *defining* class, not the runtime type.

---

**Q4: What is the difference between `super(ChildClass, self)` and `super(ParentClass, self)`?**

**A:** The first argument to `super(type, obj_or_type)` specifies the *starting point* — the search begins from the class *after* `type` in `type(obj).__mro__`.

- `super(ChildClass, self)`: searches MRO of `type(self)` starting after `ChildClass` — this is the normal usage.
- `super(ParentClass, self)`: searches starting after `ParentClass` — effectively skips `ParentClass` and any classes before it in the MRO.

```python
class A:
    def method(self): return "A"

class B(A):
    def method(self): return "B"

class C(B):
    def method(self):
        normal = super(C, self).method()     # → B.method() = "B"
        skip_b = super(B, self).method()     # → A.method() = "A"
        return f"C normal={normal} skip_b={skip_b}"

print(C().method())   # C normal=B skip_b=A
```

---

**Q5: In cooperative multiple inheritance, what happens if one class in the chain doesn't call super()?**

**A:** The chain is broken. All classes *after* the non-cooperative class in the MRO have their `__init__` (or whatever method) silently skipped.

```python
class A:
    def __init__(self, **kw):
        super().__init__(**kw)
        self.a = True

class B(A):
    def __init__(self, **kw):
        # Missing super().__init__(**kw)!
        self.b = True

class C(B, A):
    def __init__(self):
        super().__init__()

c = C()
print(c.b)           # True
print(hasattr(c, "a"))  # False — A.__init__ never called
```

This is why the cooperative pattern requires **every** class in the chain to call `super()`.

---

**Q6: Can super() be used in __new__? How?**

**A:** Yes. `__new__` is a static method (it receives the class as first argument, not an instance). The zero-argument `super()` works inside `__new__` because CPython still creates the `__class__` cell.

```python
class Base:
    def __new__(cls, *args, **kwargs):
        print(f"Base.__new__ called with cls={cls.__name__}")
        instance = super().__new__(cls)   # allocates memory
        return instance

class Child(Base):
    def __new__(cls, *args, **kwargs):
        print(f"Child.__new__ called")
        return super().__new__(cls, *args, **kwargs)

c = Child()
# Child.__new__ called
# Base.__new__ called with cls=Child
```

---

**Q7: What is `__class__` and how is it different from `type(self)`?**

**A:** `__class__` inside a method is the **class in which the method was defined** — it's a closure variable injected by CPython at compile time. `type(self)` is the **runtime type of the instance**, which may be a subclass.

```python
class Base:
    def show(self):
        print(f"__class__: {__class__.__name__}")   # always 'Base'
        print(f"type(self): {type(self).__name__}")  # varies

class Child(Base):
    pass

Child().show()
# __class__: Base    ← compile-time: defined in Base
# type(self): Child  ← runtime: the actual object type
```

`super()` uses `__class__` (not `type(self)`), which is what makes it correctly navigate the MRO.

---

**Q8: What happens when you call super() in a classmethod defined on a subclass and the method involves cls?**

**A:** `super()` in a `@classmethod` creates a proxy that searches from after the defining class in the MRO, but `cls` still refers to the *original* class the method was called on. This allows the factory pattern to work correctly through `super()`.

```python
class Animal:
    @classmethod
    def create(cls) -> "Animal":
        print(f"Animal.create called, cls={cls.__name__}")
        return cls()

class Dog(Animal):
    @classmethod
    def create(cls) -> "Dog":
        print(f"Dog.create called")
        obj = super().create()   # calls Animal.create, but cls is still Dog
        return obj

d = Dog.create()
# Dog.create called
# Animal.create called, cls=Dog
print(type(d))   # <class '__main__.Dog'>
```

---

**Q9: What is a TypeError related to MRO and when does it occur?**

**A:** Python raises `TypeError: Cannot create a consistent method resolution order` when the C3 algorithm cannot linearise the hierarchy because the bases contradict the local precedence ordering.

```python
class A: pass
class B: pass
class C(A, B): pass
class D(B, A): pass   # B before A

try:
    class E(C, D): pass   # C says A before B; D says B before A — contradiction!
except TypeError as e:
    print(e)
# TypeError: Cannot create a consistent method resolution order (MRO)
# for bases A, B
```

The fix: restructure the hierarchy so no circular precedence constraints exist.

---

**Q10: How does super() relate to __mro_entries__ and generic aliases in Python 3.9+?**

**A:** In Python 3.9+, `__class_getitem__` supports `GenericAlias` syntax (`list[int]`, `dict[str, int]`). When used as a base class, `__mro_entries__` is called to unpack the generic alias to actual classes for MRO computation. `super()` is not directly affected — it always works on the resolved MRO of actual classes, not generic aliases.

```python
class MyList(list):   # inheriting from built-in list
    def first(self):
        return self[0] if self else None

ml = MyList([3, 1, 4])
print(ml.first())   # 3
print(MyList.__mro__)  # MyList → list → object
```

---

## 3. Scenario-Based Interview Questions (10)

**Scenario 1: You have a diamond hierarchy where two branches both update a counter. Using super(), ensure the counter is incremented exactly once.**

**Q:** Implement cooperative super() to call each __init__ exactly once.

**A:**

```python
class Base:
    counter: int = 0

    def __init__(self, **kwargs: object) -> None:
        super().__init__(**kwargs)
        Base.counter += 1
        print(f"Base.__init__ — counter={Base.counter}")

class Left(Base):
    def __init__(self, **kwargs: object) -> None:
        super().__init__(**kwargs)
        print("Left.__init__")

class Right(Base):
    def __init__(self, **kwargs: object) -> None:
        super().__init__(**kwargs)
        print("Right.__init__")

class Diamond(Left, Right):
    def __init__(self) -> None:
        super().__init__()
        print("Diamond.__init__")

Base.counter = 0
d = Diamond()
print(f"Counter = {Base.counter}")
# MRO: Diamond → Left → Right → Base → object
# Base.__init__ called exactly once: counter = 1
```

---

**Scenario 2: You're debugging a method call that seems to call an unexpected class. How do you trace the MRO?**

**Q:** Show the debugging technique.

**A:**

```python
class A:
    def process(self):
        print(f"A.process — self type: {type(self).__name__}")

class B(A):
    def process(self):
        print(f"B.process")
        super().process()

class C(A):
    def process(self):
        print(f"C.process")
        super().process()

class D(B, C):
    def process(self):
        print(f"D.process — MRO: {[c.__name__ for c in type(self).__mro__]}")
        super().process()

D().process()
# D.process — MRO: ['D', 'B', 'C', 'A', 'object']
# B.process
# C.process
# A.process — self type: D

# To inspect where each method is defined:
for cls in D.__mro__:
    if "process" in cls.__dict__:
        print(f"{cls.__name__} defines process()")
```

---

**Scenario 3: You need to extend a classmethod factory in a subclass while preserving the correct return type.**

**Q:** Implement the pattern.

**A:**

```python
from __future__ import annotations
import json


class Record:
    def __init__(self, data: dict) -> None:
        self._data = data

    @classmethod
    def from_json(cls, json_str: str) -> Record:
        return cls(json.loads(json_str))

    @classmethod
    def from_file(cls, path: str) -> Record:
        with open(path) as f:
            return cls.from_json(f.read())

    def get(self, key: str):
        return self._data.get(key)


class AuditedRecord(Record):
    def __init__(self, data: dict) -> None:
        super().__init__(data)
        self.accessed: list[str] = []

    @classmethod
    def from_json(cls, json_str: str) -> AuditedRecord:
        obj = super().from_json(json_str)   # cls is AuditedRecord → correct type
        return obj

    def get(self, key: str):
        self.accessed.append(key)
        return super().get(key)


ar = AuditedRecord.from_json('{"name": "Alice", "age": 30}')
print(type(ar))         # <class '__main__.AuditedRecord'>
ar.get("name")
print(ar.accessed)     # ['name']
```

---

**Scenario 4: A mixin's __init__ uses super(), but the class it's mixed into doesn't define __init__. Does the cooperative chain still work?**

**Q:** Show what happens.

**A:** Yes. If the class doesn't define `__init__`, Python finds the mixin's `__init__` via MRO. `super().__init__(**kwargs)` in the mixin eventually reaches `object.__init__`, which accepts no extra keyword args (Python 3.3+ raises `TypeError` if unexpected kwargs are passed to `object.__init__`):

```python
class TimestampMixin:
    def __init__(self, **kwargs: object) -> None:
        super().__init__(**kwargs)
        import time
        self.ts = time.time()

class SimpleModel(TimestampMixin):
    pass   # no __init__

m = SimpleModel()   # MRO: SimpleModel → TimestampMixin → object
print(m.ts)         # timestamp — works because TimestampMixin.__init__ is found

# With extra args — object.__init__ complains:
class BadModel(TimestampMixin):
    pass

try:
    BadModel(extra="value")   # TimestampMixin passes **kwargs to object.__init__
except TypeError as e:
    print(e)   # object.__init__() takes exactly one argument (the instance to initialize)
```

---

**Scenario 5: You need to call a grandparent's method while skipping the parent. Is this possible and should you do it?**

**Q:** Show how and explain why it's usually a bad idea.

**A:**

```python
class GrandParent:
    def method(self): return "GrandParent"

class Parent(GrandParent):
    def method(self): return f"Parent + {super().method()}"

class Child(Parent):
    def method(self):
        # Skipping Parent: call GrandParent directly
        direct = GrandParent.method(self)
        # Or via explicit super:
        skip   = super(Parent, self).method()   # starts after Parent in MRO
        return f"Child (direct={direct}, skip={skip})"

print(Child().method())
# Child (direct=GrandParent, skip=GrandParent)
```

**Why it's usually a bad idea:** Skipping a parent class breaks the cooperative inheritance chain and violates the Liskov Substitution Principle. If `Parent.method()` does important work (validation, logging, state updates), skipping it creates hidden bugs. The only legitimate use is if you know exactly what you're doing (e.g., avoiding an infinite recursion bug in a base class).

---

**Scenario 6: You're writing a framework where users subclass your Handler class. How do you ensure users always call super() in their __init__?**

**Q:** Implement a warning or enforcement mechanism.

**A:**

```python
import warnings


class HandlerMeta(type):
    def __init__(cls, name, bases, namespace):
        super().__init__(name, bases, namespace)
        if "__init__" in namespace and bases:
            original_init = namespace["__init__"]
            import functools

            @functools.wraps(original_init)
            def checked_init(self, *args, **kwargs):
                self._super_called = False
                original_init(self, *args, **kwargs)
                if not getattr(self, "_super_called", True):
                    warnings.warn(
                        f"{type(self).__name__}.__init__ did not call super().__init__()",
                        stacklevel=2,
                    )

            cls.__init__ = checked_init


class Handler(metaclass=HandlerMeta):
    def __init__(self):
        self._super_called = True
        self.initialized = True

class GoodHandler(Handler):
    def __init__(self):
        super().__init__()
        self.extra = True

class BadHandler(Handler):
    def __init__(self):
        self.extra = True   # forgot super().__init__()

GoodHandler()   # no warning
BadHandler()    # UserWarning: BadHandler.__init__ did not call super().__init__()
```

---

**Scenario 7: Two mixins both override the same method and both call super(). Show that both execute.**

**Q:** Prove cooperative super() chains both mixins.

**A:**

```python
results: list[str] = []

class MixinA:
    def process(self) -> None:
        results.append("MixinA")
        super().process()

class MixinB:
    def process(self) -> None:
        results.append("MixinB")
        super().process()

class Base:
    def process(self) -> None:
        results.append("Base")

class MyClass(MixinA, MixinB, Base):
    pass

MyClass().process()
print(results)   # ['MixinA', 'MixinB', 'Base']
# MRO: MyClass → MixinA → MixinB → Base → object
# MixinA.process → super() → MixinB.process → super() → Base.process
```

---

**Scenario 8: How does super() behave inside __init_subclass__?**

**Q:** Demonstrate proper usage.

**A:** `__init_subclass__` is a classmethod implicitly. Inside it, `super().__init_subclass__(**kwargs)` must be called to allow cooperative multi-level registration:

```python
class PluginBase:
    _plugins: list[type] = []

    def __init_subclass__(cls, register: bool = True, **kwargs: object) -> None:
        super().__init_subclass__(**kwargs)
        if register:
            PluginBase._plugins.append(cls)

class LogPlugin(PluginBase, register=True): pass
class CachePlugin(PluginBase, register=True): pass
class InternalHelper(PluginBase, register=False): pass

print([p.__name__ for p in PluginBase._plugins])
# ['LogPlugin', 'CachePlugin']
```

---

**Scenario 9: You are refactoring code that calls Parent.method(self) everywhere instead of super().method(). What are the risks?**

**Q:** Explain and show the correct pattern.

**A:** Hardcoded parent class calls bypass MRO. If the hierarchy changes (a new mixin is inserted, the parent is renamed), all hardcoded calls must be updated. More critically, in multiple inheritance, hardcoded calls break the cooperative chain — classes between the hardcoded parent and the original caller are skipped.

```python
class A:
    def save(self): print("A.save")

class B(A):
    def save(self):
        A.save(self)    # hardcoded — works now
        print("B.save")

class C(A):
    def save(self): print("C.save")

class D(B, C):   # MRO: D → B → C → A
    def save(self):
        super().save()   # calls B.save → which hardcodes A, skipping C!

D().save()
# A.save
# B.save
# C.save never called!

# Fix B to use super():
class BFixed(A):
    def save(self):
        super().save()   # cooperative — will call C.save in context of D
        print("BFixed.save")
```

---

**Scenario 10: How do you write a mixin that works correctly whether it's placed first or last in the base class list?**

**Q:** Design a position-independent mixin.

**A:** A well-designed mixin calls `super()` and accepts `**kwargs`, making it position-independent in the MRO:

```python
class RetryMixin:
    _max_retries: int = 3

    def execute(self, *args, **kwargs):
        last_exc = None
        for attempt in range(self._max_retries):
            try:
                return super().execute(*args, **kwargs)
            except Exception as e:
                last_exc = e
                print(f"Attempt {attempt + 1} failed: {e}")
        raise last_exc

class APIClient:
    def execute(self, url: str) -> dict:
        # Simulate a call
        return {"url": url}

class RetryingClient(RetryMixin, APIClient):   # Mixin first — wraps execute
    pass

class AltRetryingClient(APIClient, RetryMixin):  # Mixin last — still works via MRO
    pass

c = RetryingClient()
print(c.execute("https://api.example.com"))
```

The key requirement: every class in the chain must define `execute` cooperatively and call `super().execute(...)`.

---

## 4. Quick Recap / Cheatsheet

- **MRO** = the ordered tuple of classes Python searches for attribute lookup; stored in `__mro__`.
- **C3 linearisation**: guarantees local precedence, monotonicity, each class appears exactly once.
- **`Class.__mro__`**: tuple. **`Class.mro()`**: list. Same content.
- **Diamond inheritance**: C3 ensures the shared ancestor appears last (after all its descendants).
- **`super()`**: returns a **proxy object**, NOT the parent class.
- **Zero-argument `super()`**: uses `__class__` cell (implicitly created by CPython) + `self`/`cls`.
- **`__class__` cell**: always the class where the method is *defined*, not `type(self)` (which may be a subclass).
- **`super(ChildClass, self)`**: searches MRO of `type(self)` starting AFTER `ChildClass`.
- **`super(ParentClass, self)`**: skips `ParentClass` and everything before it — use cautiously.
- **Cooperative multiple inheritance**: EVERY class must call `super().__init__(**kwargs)` and accept `**kwargs`.
- **Breaking the chain**: one class not calling `super()` silently skips all subsequent classes in the MRO.
- **`super()` in `@classmethod`**: works naturally; `cls` propagates correctly.
- **`super()` in `@staticmethod`**: zero-arg form fails (no `__class__` cell); use `super(ClassName, ClassName)`.
- **`super()` in `__new__`**: works; `cls` is available as first arg.
- **MRO TypeError**: raised when C3 cannot linearise the hierarchy (contradictory precedence constraints).
- **Hardcoded parent call** (`Parent.method(self)`): bypasses MRO, breaks cooperative chain in multiple inheritance.
- **`__init_subclass__`**: must call `super().__init_subclass__(**kwargs)` for cooperative multi-level hooks.
