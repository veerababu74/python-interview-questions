# Inheritance

## 1. Concept (Deep Explanation)

### What Is Inheritance and Why Does It Exist?

Inheritance allows a class (**subclass** / **derived class**) to acquire the attributes and methods of another class (**superclass** / **base class** / **parent class**). It serves two purposes:

1. **Code reuse** — subclasses inherit behaviour without rewriting it.
2. **Subtype relationship** — a subclass *is-a* parent class, enabling polymorphism.

```python
class Animal:
    def __init__(self, name: str) -> None:
        self.name = name

    def breathe(self) -> str:
        return f"{self.name} breathes air"

class Dog(Animal):
    def speak(self) -> str:
        return f"{self.name} says Woof"

rex = Dog("Rex")
print(rex.breathe())   # "Rex breathes air"  — inherited
print(rex.speak())     # "Rex says Woof"  — own method
print(isinstance(rex, Animal))   # True — is-a relationship
```

### Types of Inheritance

**Single inheritance:** One parent.
```python
class Vehicle: pass
class Car(Vehicle): pass
```

**Multi-level inheritance:** Chain of parents.
```python
class A: pass
class B(A): pass
class C(B): pass     # C → B → A → object
```

**Multiple inheritance:** More than one direct parent.
```python
class Flyable: pass
class Swimmable: pass
class Duck(Flyable, Swimmable): pass
```

**Hierarchical inheritance:** Multiple subclasses from one parent.
```python
class Shape: pass
class Circle(Shape): pass
class Square(Shape): pass
```

**Hybrid / Diamond inheritance:** Combination of the above, creating a diamond structure.
```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass    # diamond: D → B → C → A → object
```

### `__bases__`, `__mro__`, and `type.mro()`

Every Python class has:
- `__bases__`: tuple of its *direct* parent classes.
- `__mro__`: tuple of *all* classes in Method Resolution Order (linearised).
- `mro()`: method on the type that returns the MRO as a list.

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass

print(D.__bases__)   # (<class '__main__.B'>, <class '__main__.C'>)
print(D.__mro__)
# (<class '__main__.D'>, <class '__main__.B'>, <class '__main__.C'>,
#  <class '__main__.A'>, <class 'object'>)
print(type.mro(D))   # same as D.__mro__ but as a method call
```

### C3 Linearisation: How MRO Is Computed

Python uses the **C3 linearisation algorithm** (introduced in Python 2.3 for new-style classes). The algorithm guarantees:
1. Subclasses appear before their parents.
2. The order of base classes is preserved.
3. The monotonicity property: if C comes before D in the MRO of some class, C must come before D in the MRO of every subclass that inherits from both.

**Worked Diamond Example:**

```
    A
   / \
  B   C
   \ /
    D
```

```python
class A:
    def method(self): return "A"

class B(A):
    def method(self): return "B"

class C(A):
    def method(self): return "C"

class D(B, C):
    pass

print(D.__mro__)
# D → B → C → A → object
print(D().method())   # "B" — B is first in MRO after D
```

**C3 merge step-by-step for D(B, C):**

```
L(D) = D + merge(L(B), L(C), [B, C])
L(B) = [B, A, object]
L(C) = [C, A, object]

merge([B, A, object], [C, A, object], [B, C]):
  1. Head of first list: B. Is B in the tail of any other list? No. → Take B.
     Result so far: [D, B]
     Remaining: merge([A, object], [C, A, object], [C])

  2. Head: A. Is A in the tail of [C, A, object]? Yes (position 1). → Skip A.
     Head of next list: C. Is C in any tail? No. → Take C.
     Result so far: [D, B, C]
     Remaining: merge([A, object], [A, object], [])

  3. Head: A. Is A in any tail? No (only 'object' remains). → Take A.
     Result so far: [D, B, C, A]
     Remaining: merge([object], [object], [])

  4. Take object.
     Final: [D, B, C, A, object]
```

### `super()` and MRO

`super()` returns a **proxy object** that delegates method calls to the next class in the MRO, not necessarily the direct parent.

```python
class A:
    def greet(self) -> str:
        return "Hello from A"

class B(A):
    def greet(self) -> str:
        return super().greet() + " + B"

class C(A):
    def greet(self) -> str:
        return super().greet() + " + C"

class D(B, C):
    def greet(self) -> str:
        return super().greet() + " + D"

# D's MRO: D → B → C → A → object
# D.greet() calls B.greet() (via super in D)
# B.greet() calls C.greet() (via super in B — C is next in MRO, not A!)
# C.greet() calls A.greet() (via super in C)

print(D().greet())   # "Hello from A + C + B + D"
```

This is **cooperative multiple inheritance** — each class cooperates by calling `super()`, and the MRO ensures each class in the chain is called exactly once.

### `issubclass()` vs `isinstance()`

```python
class Animal: pass
class Dog(Animal): pass

d = Dog()

print(isinstance(d, Dog))     # True  — d is a Dog
print(isinstance(d, Animal))  # True  — Dog is a subclass of Animal
print(isinstance(d, object))  # True  — everything is an object

print(issubclass(Dog, Animal))  # True
print(issubclass(Dog, Dog))     # True  — a class is its own subclass
print(issubclass(Animal, Dog))  # False

# isinstance also works with tuples of types:
print(isinstance(d, (str, Dog, int)))   # True — checks against each type
```

### Mixins: Composable Inheritance

A **mixin** is a class designed to be mixed into other class hierarchies to add specific functionality. It should:
- Not be instantiated directly.
- Not have `__init__` that calls `super().__init__()` with specific arguments.
- Provide a well-defined, narrow piece of functionality.

```python
class LogMixin:
    def log(self, message: str) -> None:
        print(f"[{type(self).__name__}] {message}")


class JsonMixin:
    def to_json(self) -> str:
        import json
        return json.dumps(self.__dict__)


class TimestampMixin:
    def __init__(self, *args, **kwargs) -> None:
        super().__init__(*args, **kwargs)
        import time
        self.created_at = time.time()


class User(TimestampMixin, LogMixin, JsonMixin):
    def __init__(self, name: str, email: str) -> None:
        super().__init__()
        self.name  = name
        self.email = email

u = User("Alice", "alice@example.com")
u.log("User created")
print(u.to_json())
print(u.created_at)
```

### When NOT to Use Inheritance

Inheritance has a famous failure mode: **the fragile base class problem**. Changes to the base class can break subclasses in unexpected ways.

Prefer **composition** over inheritance when:
1. The subclass relationship is "has-a" rather than "is-a".
2. You only need a subset of parent behaviour.
3. The parent class is from a third-party library and may change.
4. Multiple levels of inheritance lead to deeply nested, hard-to-trace hierarchies.

```python
# Fragile inheritance:
class List:
    def append(self, item): ...
    def extend(self, items):
        for item in items:
            self.append(item)   # ← relies on append

class CountingList(list):
    def __init__(self): super().__init__(); self.count = 0
    def append(self, item):
        self.count += 1
        super().append(item)
    # extend() works correctly since list.extend() calls append() in CPython
    # but this is an internal implementation detail — fragile!

# Composition is safer:
class CountingListV2:
    def __init__(self):
        self._list = []
        self.count = 0
    def append(self, item):
        self.count += 1
        self._list.append(item)
    def __len__(self): return len(self._list)
    def __iter__(self): return iter(self._list)
```

---

## 2. Conceptual Interview Questions (10)

**Q1: What is the difference between single and multiple inheritance? What problems can multiple inheritance cause?**

**A:** Single inheritance means one direct parent; multiple inheritance means two or more. Multiple inheritance can lead to the **diamond problem** — when two parent classes share a common ancestor, ambiguity arises about which version of a method to call.

Python resolves this with the **C3 linearisation algorithm** (MRO). The MRO provides a consistent, deterministic method lookup order. The resolution is always unambiguous in Python, though the result may surprise programmers unfamiliar with MRO.

```python
class A:
    def speak(self): return "A"

class B(A):
    def speak(self): return "B"

class C(A):
    def speak(self): return "C"

class D(B, C): pass

print(D().speak())   # "B"  — B precedes C in D's MRO
print(D.__mro__)     # D → B → C → A → object
```

---

**Q2: What is the MRO and why does Python use C3 linearisation instead of depth-first search?**

**A:** MRO (Method Resolution Order) defines the sequence in which Python searches classes for a method. Python 2 originally used DFS (depth-first search), which produced counterintuitive results in diamond inheritance: A would be searched before C even when C is a more specific parent.

C3 guarantees:
1. Local precedence order (left-to-right in class definition).
2. Monotonicity (subclasses don't change the relative ordering of any two classes).
3. Good behaviour in diamond hierarchies (the shared ancestor is found last).

```python
# DFS would give: D → B → A → C → A (visits A twice, incorrect)
# C3 gives:       D → B → C → A (each class visited exactly once)
```

---

**Q3: What does `super()` return? Is it the parent class?**

**A:** `super()` returns a **proxy object** — not the parent class itself. The proxy delegates attribute lookup to the *next class in the MRO* of the instance's class, starting after the class where `super()` is called.

This distinction matters in multiple inheritance: `super()` in `B` (which is part of the MRO `D → B → C → A`) will delegate to `C`, not `A`, even though `A` is B's direct parent.

```python
class B(A):
    def method(self):
        print(super())          # <super: <class 'B'>, <__main__.D object>>
        return super().method() # delegates to C.method, not A.method
```

---

**Q4: What is the difference between `isinstance()` and `issubclass()`?**

**A:** `isinstance(obj, cls)` checks if `obj` is an instance of `cls` or any of its subclasses. `issubclass(sub, cls)` checks if `sub` is a subclass of `cls` (including `cls` itself). Both respect ABCs and registered virtual subclasses.

```python
print(isinstance([], list))       # True
print(isinstance([], object))     # True — all Python objects
print(issubclass(list, object))   # True
print(issubclass(list, list))     # True — reflexive
print(isinstance(42, (int, str))) # True — tuple shorthand
```

---

**Q5: What is a mixin and how does it differ from regular inheritance?**

**A:** A mixin is a class that provides a reusable chunk of functionality and is designed to be combined with other classes, not instantiated alone. Unlike regular base classes, mixins:
- Don't represent a complete object type.
- Are narrow in scope (one concern).
- Often don't define `__init__` or do so cooperatively.
- Are composable — multiple mixins can be combined independently.

```python
class SerializableMixin:
    def serialize(self) -> bytes:
        import pickle
        return pickle.dumps(self)

class ValidationMixin:
    def validate(self) -> bool:
        return all(v is not None for v in vars(self).values())

class Model(ValidationMixin, SerializableMixin):
    def __init__(self, x: int): self.x = x

m = Model(10)
print(m.validate())     # True
print(len(m.serialize()) > 0)  # True
```

---

**Q6: Can you call `super().__init__()` in multiple inheritance? What pattern must be followed?**

**A:** Yes, and it requires the **cooperative super() pattern**: every class in the hierarchy must call `super().__init__(**kwargs)` and accept and forward `**kwargs`. This ensures every `__init__` in the MRO chain is called exactly once.

```python
class A:
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.a = "A"

class B(A):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.b = "B"

class C(A):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.c = "C"

class D(B, C):
    def __init__(self, **kwargs):
        super().__init__(**kwargs)
        self.d = "D"

d = D()
print(vars(d))   # {'a': 'A', 'c': 'C', 'b': 'B', 'd': 'D'}
```

---

**Q7: What is the fragile base class problem?**

**A:** When a base class changes (even in seemingly backward-compatible ways), it can break subclasses that depend on internal implementation details. For example, if `Base.method_a()` calls `self.method_b()` and a subclass overrides `method_b()`, any change to how `method_a` uses `method_b` can break the subclass.

```python
class Base:
    def process(self):
        result = self._compute()   # calls overridable method
        return result * 2

    def _compute(self):
        return 10

class MyClass(Base):
    def _compute(self):   # override changes Base.process() behaviour
        return 100

# If Base later changes process() to call _compute() differently, MyClass breaks.
```

The fix: prefer composition, use private name mangling for truly internal methods, or document which methods are safe to override.

---

**Q8: What happens if you inherit from a built-in type like `list` or `dict`?**

**A:** You can inherit from built-in types. The subclass gets all built-in methods plus can add new ones or override existing ones. Caution: built-in methods may not call overridden versions consistently (CPython implements many in C and calls the C version directly in some paths).

```python
class UpperList(list):
    def append(self, item: str) -> None:  # type: ignore
        super().append(item.upper())

ul = UpperList()
ul.append("hello")
ul.extend(["world"])   # extend() calls append() in pure Python, works here
print(ul)              # ['HELLO', 'WORLD']

# But: list.sort() in C does NOT call self.append — overriding append doesn't affect sort
```

---

**Q9: How do ABCs participate in inheritance hierarchies?**

**A:** ABCs integrate seamlessly: a class can inherit from one or more ABCs plus concrete classes. The `__abstractmethods__` frozenset is inherited and a subclass must implement all abstract methods to be instantiable.

```python
from abc import ABC, abstractmethod

class Drawable(ABC):
    @abstractmethod
    def draw(self) -> None: ...

class Clickable(ABC):
    @abstractmethod
    def on_click(self) -> None: ...

class Button(Drawable, Clickable):
    def draw(self) -> None: print("Drawing button")
    def on_click(self) -> None: print("Button clicked")

b = Button()
b.draw()
b.on_click()
print(isinstance(b, Drawable))   # True
print(isinstance(b, Clickable))  # True
```

---

**Q10: What is `__init_subclass__` and how does it relate to inheritance?**

**A:** `__init_subclass__` is a classmethod called on the *parent* class whenever a new subclass is created. It's a hook for customising subclass creation without a metaclass.

```python
class Plugin:
    _registry: dict[str, type] = {}

    def __init_subclass__(cls, plugin_name: str = "", **kwargs: object) -> None:
        super().__init_subclass__(**kwargs)
        if plugin_name:
            Plugin._registry[plugin_name] = cls

class AudioPlugin(Plugin, plugin_name="audio"):
    def run(self): print("audio")

class VideoPlugin(Plugin, plugin_name="video"):
    def run(self): print("video")

print(Plugin._registry)
# {'audio': <class '__main__.AudioPlugin'>, 'video': <class '__main__.VideoPlugin'>}
```

---

## 3. Scenario-Based Interview Questions (10)

**Scenario 1: You're building a web framework where every request handler must implement `get()` and `post()`. Some handlers share authentication logic. Design the hierarchy.**

**Q:** Show the design.

**A:**

```python
from abc import ABC, abstractmethod


class BaseHandler(ABC):
    def dispatch(self, method: str, request: dict) -> dict:
        handler = getattr(self, method.lower(), None)
        if handler is None:
            return {"status": 405, "body": "Method Not Allowed"}
        return handler(request)

    @abstractmethod
    def get(self, request: dict) -> dict: ...

    @abstractmethod
    def post(self, request: dict) -> dict: ...


class AuthMixin:
    def authenticate(self, request: dict) -> bool:
        return "token" in request.get("headers", {})


class ProtectedHandler(AuthMixin, BaseHandler):
    def dispatch(self, method: str, request: dict) -> dict:
        if not self.authenticate(request):
            return {"status": 401, "body": "Unauthorized"}
        return super().dispatch(method, request)

    def get(self, request: dict) -> dict:
        return {"status": 200, "body": "secret data"}

    def post(self, request: dict) -> dict:
        return {"status": 201, "body": "created"}


h = ProtectedHandler()
print(h.dispatch("get", {"headers": {"token": "abc"}}))
print(h.dispatch("get", {}))   # {"status": 401, ...}
```

---

**Scenario 2: Your team has created a 6-level deep inheritance hierarchy. A change at level 2 breaks level 5. How do you fix this?**

**Q:** What refactoring strategy do you apply?

**A:** Flatten the hierarchy using composition. Extract shared behaviour into mixins or standalone classes:

```python
# Before: deep chain (fragile)
class L1: ...
class L2(L1): ...
class L3(L2): ...
class L4(L3): ...
class L5(L4): ...
class L6(L5): ...

# After: composition — L6 uses L1 and L3 functionality without inheriting
class L1Feature:
    def feature_a(self): ...

class L3Feature:
    def feature_b(self): ...

class L6:
    def __init__(self):
        self._a = L1Feature()
        self._b = L3Feature()

    def use_a(self): return self._a.feature_a()
    def use_b(self): return self._b.feature_b()
```

The rule: "Favour composition over inheritance" (GoF Design Patterns).

---

**Scenario 3: You need to create a `ReadOnlyDict` class that inherits from `dict` but raises an error on any modification. Implement it.**

**Q:** Show the implementation.

**A:**

```python
class ReadOnlyDict(dict):
    _MUTATING_METHODS = (
        "__setitem__", "__delitem__", "pop", "popitem",
        "clear", "update", "setdefault",
    )

    def _raise(self, *args, **kwargs) -> None:
        raise TypeError(f"{type(self).__name__} is read-only")

    def __init_subclass__(cls, **kwargs: object) -> None:
        super().__init_subclass__(**kwargs)

for method in ReadOnlyDict._MUTATING_METHODS:
    setattr(ReadOnlyDict, method, ReadOnlyDict._raise)

rod = ReadOnlyDict({"a": 1, "b": 2})
print(rod["a"])   # 1
try:
    rod["c"] = 3
except TypeError as e:
    print(e)   # ReadOnlyDict is read-only
```

---

**Scenario 4: You need to implement logging in every subclass without requiring each to call `super().__init__()`. Use `__init_subclass__`.**

**Q:** Implement the pattern.

**A:**

```python
import logging
import functools


class AutoLogger:
    def __init_subclass__(cls, **kwargs: object) -> None:
        super().__init_subclass__(**kwargs)
        cls.logger = logging.getLogger(cls.__qualname__)
        original_init = cls.__dict__.get("__init__")
        if original_init:
            @functools.wraps(original_init)
            def wrapped_init(self, *args, **kw):
                cls.logger.debug("Creating %s", type(self).__name__)
                original_init(self, *args, **kw)
            cls.__init__ = wrapped_init


class Service(AutoLogger):
    def __init__(self, name: str) -> None:
        self.name = name

class APIService(Service):
    def __init__(self, name: str, url: str) -> None:
        super().__init__(name)
        self.url = url

logging.basicConfig(level=logging.DEBUG)
api = APIService("myapi", "https://api.example.com")
```

---

**Scenario 5: Two base classes both define `__init__`. A subclass inherits from both. How do you ensure both `__init__` methods run?**

**Q:** Use cooperative `super()`.

**A:**

```python
class Timestamped:
    def __init__(self, **kwargs: object) -> None:
        super().__init__(**kwargs)
        import time
        self.created_at = time.time()

class Named:
    def __init__(self, name: str = "", **kwargs: object) -> None:
        super().__init__(**kwargs)
        self.name = name

class NamedTimestamped(Timestamped, Named):
    def __init__(self, name: str) -> None:
        super().__init__(name=name)

obj = NamedTimestamped("Alice")
print(obj.name)        # "Alice"
print(obj.created_at)  # timestamp
# MRO: NamedTimestamped → Timestamped → Named → object
# super() in NamedTimestamped → Timestamped.__init__
# super() in Timestamped → Named.__init__
# super() in Named → object.__init__
```

---

**Scenario 6: A `Vehicle` class and a `Boat` class both have `travel()` methods. A `Hovercraft` inherits from both. Which `travel()` is called and how do you call both?**

**Q:** Resolve the ambiguity.

**A:**

```python
class Vehicle:
    def travel(self) -> str: return "Vehicle travels on land"

class Boat:
    def travel(self) -> str: return "Boat travels on water"

class Hovercraft(Vehicle, Boat):
    def travel(self) -> str:
        land  = Vehicle.travel(self)
        water = Boat.travel(self)
        return f"Hovercraft: {land} AND {water}"

h = Hovercraft()
print(h.travel())
print(Hovercraft.__mro__)   # Hovercraft → Vehicle → Boat → object
# Default (via MRO): Vehicle.travel() would be called
# But we override and call both explicitly
```

---

**Scenario 7: You want to prevent a class from being subclassed. How do you implement this in Python?**

**Q:** Show the implementation.

**A:**

```python
class Final:
    def __init_subclass__(cls, **kwargs: object) -> None:
        raise TypeError(f"{Final.__name__} cannot be subclassed")

try:
    class Attempt(Final):
        pass
except TypeError as e:
    print(e)   # Final cannot be subclassed
```

Python 3.8+ also supports `@final` from `typing` for type checkers (not runtime):

```python
from typing import final

@final
class Singleton:   # type checkers will warn about subclassing
    pass
```

---

**Scenario 8: How does Python's MRO affect method resolution in a complex hierarchy? Walk through a non-trivial example.**

**Q:** Give a worked example.

**A:**

```python
class A:
    def method(self): return "A"

class B(A):
    def method(self): return f"B → {super().method()}"

class C(A):
    def method(self): return f"C → {super().method()}"

class D(B, C):
    def method(self): return f"D → {super().method()}"

# MRO: D → B → C → A → object
# D.method()  calls super() → B.method()
# B.method()  calls super() → C.method()  (C is next in D's MRO, not A!)
# C.method()  calls super() → A.method()
# A.method()  returns "A"

print(D().method())   # "D → B → C → A"
```

This shows why `super()` follows the MRO of the *original* object, not the MRO of the class where `super()` is called.

---

**Scenario 9: You inherit from a third-party class that has a bug in one method. How do you patch it safely?**

**Q:** Show the pattern.

**A:**

```python
# Third-party class (simplified):
class ThirdPartyClient:
    def fetch(self, url: str) -> dict:
        # Assume buggy — doesn't handle timeouts
        import urllib.request
        with urllib.request.urlopen(url) as r:
            import json
            return json.loads(r.read())

class SafeClient(ThirdPartyClient):
    def fetch(self, url: str, timeout: int = 10) -> dict:
        import urllib.request
        try:
            with urllib.request.urlopen(url, timeout=timeout) as r:
                import json
                return json.loads(r.read())
        except Exception as e:
            return {"error": str(e)}

# Note: composition is safer if ThirdPartyClient changes internally
class SafeClientV2:
    def __init__(self):
        self._client = ThirdPartyClient()

    def fetch(self, url: str) -> dict:
        try:
            return self._client.fetch(url)
        except Exception as e:
            return {"error": str(e)}
```

---

**Scenario 10: You're reviewing a PR that adds 4 levels of inheritance to implement a feature. The feature could be achieved with a mixin. How do you argue for the refactor?**

**Q:** Provide the refactoring argument and example.

**A:** Deep inheritance hierarchies create coupling, make testing harder, and trigger the fragile base class problem. A mixin achieves the same result with explicit, flat composition:

```python
# Before: 4-level deep hierarchy (fragile)
class Model: ...
class TimestampedModel(Model): ...
class SluggedModel(TimestampedModel): ...
class AuditedModel(SluggedModel): ...
class Article(AuditedModel): ...  # must inherit ALL of the above to get the features

# After: flat with mixins (composable)
class TimestampMixin:
    def __init__(self, **kw): super().__init__(**kw); import time; self.ts = time.time()

class SlugMixin:
    def __init__(self, title="", **kw): super().__init__(**kw); self.slug = title.lower().replace(" ", "-")

class AuditMixin:
    def __init__(self, **kw): super().__init__(**kw); self.audit_log: list = []

class Article(TimestampMixin, SlugMixin, AuditMixin):
    def __init__(self, title: str): super().__init__(title=title); self.title = title

a = Article("Hello World")
print(a.slug, a.ts, a.audit_log)
```

Benefits: each mixin is independently testable, composable in any combination, and the hierarchy is flat.

---

## 4. Quick Recap / Cheatsheet

- **Inheritance types**: single, multi-level, multiple, hierarchical, hybrid/diamond.
- **`__bases__`**: direct parents only. **`__mro__`**: full resolution order.
- **C3 linearisation**: guarantees each class in the MRO appears once, local precedence preserved, monotonic.
- **Diamond resolution**: shared ancestor appears last in MRO; `super()` chains through B → C → A.
- **`super()` is a proxy**, not the parent class — it delegates to the NEXT class in the MRO.
- **Cooperative super()**: all `__init__` methods must call `super().__init__(**kwargs)` and accept `**kwargs`.
- **`issubclass(A, B)`**: True if A is B or inherits from B (directly or transitively).
- **`isinstance(obj, cls)`**: True if `type(obj)` is `cls` or a subclass; respects ABCs and `register()`.
- **Mixin**: narrow-scope, non-instantiable class added to hierarchies for reusable features.
- **Fragile base class**: changes in parent break subclasses that rely on implementation details.
- **Composition over inheritance**: prefer "has-a" when "is-a" doesn't cleanly apply.
- **`__init_subclass__`**: hook called in parent when a subclass is created; use for registry/validation.
- **`@final`** (typing): hints to type checkers a class should not be subclassed (no runtime enforcement).
- **Builtin inheritance**: possible but C-level methods may bypass Python-level overrides.
- **ABCs in hierarchy**: `__abstractmethods__` is inherited; all abstract methods must be implemented to instantiate.
