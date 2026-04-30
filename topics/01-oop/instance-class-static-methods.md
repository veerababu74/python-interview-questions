# Instance, Class, and Static Methods

## 1. Concept (Deep Explanation)

### What They Are and Why They Exist

Python methods come in three flavours, each serving a distinct purpose in object-oriented design:

- **Instance methods** receive `self` as their first argument — a reference to the specific object on which the method is called. They can read and mutate both instance state and class state.
- **Class methods** receive `cls` as their first argument — a reference to the *class itself* (not an instance). They are declared with `@classmethod` and can access or mutate class-level state. They are the standard pattern for *factory/alternate constructors*.
- **Static methods** receive neither `self` nor `cls`. They are declared with `@staticmethod` and are essentially plain functions that live inside a class namespace for organisational cohesion.

### CPython Internals: The Descriptor Protocol

Understanding *why* these three types behave differently requires understanding Python's **descriptor protocol**. A descriptor is any object that defines `__get__`, `__set__`, or `__delete__`.

When you write `instance.method`, Python does not simply return the function stored in `Class.__dict__['method']`. Instead it calls `Class.__dict__['method'].__get__(instance, Class)`.

For a **regular function**, `function.__get__(instance, owner)` returns a **bound method** — an object that wraps the function and pre-fills the first argument with `instance`.

```python
class Foo:
    def bar(self):
        return self

f = Foo()
# These are equivalent:
f.bar()
Foo.bar(f)

# Inspecting the descriptor protocol directly:
raw_func = Foo.__dict__['bar']          # plain function object
bound    = raw_func.__get__(f, Foo)     # bound method
print(bound)          # <bound method Foo.bar of <Foo object ...>>
print(bound.__self__) # f  — the instance that was bound
print(bound.__func__)  # raw_func
```

`classmethod` is itself a **descriptor**. Its `__get__` ignores the instance and instead binds `cls` to the class:

```python
class Meta:
    @classmethod
    def info(cls):
        return cls

raw = Meta.__dict__['info']       # <classmethod object>
bound = raw.__get__(None, Meta)   # callable bound to Meta
print(bound())                    # <class '__main__.Meta'>
```

`staticmethod` is also a descriptor, but its `__get__` simply returns the wrapped function unchanged — no binding occurs at all:

```python
class Utils:
    @staticmethod
    def add(a, b):
        return a + b

raw = Utils.__dict__['add']       # <staticmethod object>
fn  = raw.__get__(None, Utils)    # returns the original function, unbound
print(fn)  # <function Utils.add at 0x...>
```

### Progressive Code Examples

#### 1. Basic Usage

```python
class BankAccount:
    interest_rate: float = 0.05          # class variable

    def __init__(self, owner: str, balance: float = 0.0) -> None:
        self.owner   = owner             # instance variable
        self.balance = balance           # instance variable

    # --- Instance method: accesses both self and class state ---
    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Deposit must be positive")
        self.balance += amount

    def apply_interest(self) -> None:
        self.balance += self.balance * BankAccount.interest_rate

    # --- Class method: alternate constructor (factory pattern) ---
    @classmethod
    def from_dict(cls, data: dict) -> "BankAccount":
        return cls(owner=data["owner"], balance=data.get("balance", 0.0))

    # --- Static method: utility that belongs here conceptually ---
    @staticmethod
    def validate_amount(amount: float) -> bool:
        return isinstance(amount, (int, float)) and amount > 0

acc = BankAccount("Alice", 1000.0)
acc.deposit(200.0)
acc.apply_interest()

acc2 = BankAccount.from_dict({"owner": "Bob", "balance": 500.0})
print(BankAccount.validate_amount(-10))  # False
```

#### 2. Factory Pattern with @classmethod and Subclassing

The `@classmethod` factory pattern is *subclass-aware* because `cls` holds the actual class. This is its main advantage over using `__init__` directly.

```python
from __future__ import annotations
import json
from datetime import date


class Event:
    def __init__(self, name: str, date_: date) -> None:
        self.name  = name
        self.date_ = date_

    @classmethod
    def from_iso(cls, name: str, iso_str: str) -> Event:
        return cls(name, date.fromisoformat(iso_str))

    @classmethod
    def from_json(cls, json_str: str) -> Event:
        data = json.loads(json_str)
        return cls(data["name"], date.fromisoformat(data["date"]))

    def __repr__(self) -> str:
        return f"{type(self).__name__}({self.name!r}, {self.date_})"


class ConferenceEvent(Event):
    def __init__(self, name: str, date_: date, venue: str) -> None:
        super().__init__(name, date_)
        self.venue = venue

    def __repr__(self) -> str:
        return f"ConferenceEvent({self.name!r}, {self.date_}, {self.venue!r})"


# Crucially, cls is ConferenceEvent here, so from_iso returns a ConferenceEvent
# (though venue won't be set — you'd override from_iso if needed)
e = Event.from_iso("PyCon", "2025-05-15")
print(e)  # Event('PyCon', 2025-05-15)
```

#### 3. @staticmethod vs Module-Level Function

```python
# Option A: module-level function (perfectly valid)
def _parse_csv_row(row: str) -> list[str]:
    return [cell.strip() for cell in row.split(",")]

class CSVParser:
    def parse(self, text: str) -> list[list[str]]:
        return [_parse_csv_row(row) for row in text.splitlines()]

# Option B: @staticmethod (groups helper with class, avoids polluting namespace)
class CSVParserV2:
    @staticmethod
    def _parse_row(row: str) -> list[str]:
        return [cell.strip() for cell in row.split(",")]

    def parse(self, text: str) -> list[list[str]]:
        return [CSVParserV2._parse_row(row) for row in text.splitlines()]
```

When the helper is *only* used by this class, `@staticmethod` improves cohesion. When it might be reused elsewhere, a module-level function is more appropriate.

### Pitfalls and Interview Traps

**Trap 1 — @staticmethod cannot be overridden polymorphically:**

```python
class Animal:
    @staticmethod
    def sound() -> str:
        return "..."

class Dog(Animal):
    @staticmethod
    def sound() -> str:
        return "Woof"

a: Animal = Dog()
# There is NO dynamic dispatch — static methods don't use cls
# The following calls Dog.sound() ONLY because Python looks up the type:
print(a.sound())   # "Woof"  — but only because a is actually a Dog

# With @classmethod:
class Cat(Animal):
    @classmethod
    def make_sound(cls) -> str:
        return f"{cls.__name__} says Meow"
```

**Trap 2 — Calling a classmethod via an instance vs via the class:**

```python
class Counter:
    count = 0

    @classmethod
    def increment(cls) -> None:
        cls.count += 1

c = Counter()
c.increment()         # works, cls == Counter
Counter.increment()   # also works, cls == Counter
print(Counter.count)  # 2
```

Both forms work, but the idiom for class methods is to call them on the class.

**Trap 3 — Accidentally omitting self/cls:**

```python
class Bad:
    def oops():   # missing self → works as a static, but is NOT a @staticmethod
        return 42

Bad.oops()   # OK (unbound call)
Bad().oops() # TypeError: oops() takes 0 positional arguments but 1 was given
```

Always use `@staticmethod` explicitly if you don't need `self` or `cls`.

### Best Practices

- Use **instance methods** as the default.
- Use **@classmethod** for factory constructors (`from_json`, `from_dict`, `from_env`).
- Use **@classmethod** to manipulate class-level state (counters, caches shared across instances).
- Use **@staticmethod** only for utilities that are logically part of the class but need no instance or class reference.
- Prefer a **module-level function** when the logic is reusable outside the class.
- Never use `@staticmethod` just to avoid passing `self` — that is a design smell indicating the method may not belong in the class.

---

## 2. Conceptual Interview Questions (10)

**Q1: What is the difference between an instance method, a class method, and a static method?**

**A:** An **instance method** takes `self` as its first parameter and operates on a specific instance. It can access and modify both instance state (`self.x`) and class state (`type(self).y`).

A **class method** is decorated with `@classmethod` and takes `cls` as its first parameter. `cls` refers to the class itself — or the subclass if the method is called on a subclass. This makes classmethods ideal for factory constructors because the returned object will be of the correct subclass type.

A **static method** is decorated with `@staticmethod` and takes neither `self` nor `cls`. It cannot access instance or class state without explicit passing. It is logically grouped with the class but behaves like a plain function.

```python
class Demo:
    value = 10

    def inst(self):          return self.value   # reads from instance or class
    @classmethod
    def clsm(cls):           return cls.value    # reads from class (or subclass)
    @staticmethod
    def stat():              return Demo.value   # hard-coded class reference
```

---

**Q2: How does Python's descriptor protocol power @classmethod and @staticmethod?**

**A:** Both `classmethod` and `staticmethod` are **descriptor objects** — they implement `__get__`. When Python evaluates `instance.method` or `Class.method`, the attribute lookup finds the descriptor in `Class.__dict__` and calls its `__get__(instance_or_None, owner_class)`.

- For `classmethod.__get__`, the returned callable always binds `cls` to the owner class, ignoring the instance.
- For `staticmethod.__get__`, the returned callable is the raw underlying function with no binding whatsoever.

```python
class Desc:
    @classmethod
    def cm(cls): return cls
    @staticmethod
    def sm(): return 42

# Inspecting the raw descriptors:
print(type(Desc.__dict__['cm']))  # <class 'classmethod'>
print(type(Desc.__dict__['sm']))  # <class 'staticmethod'>

# Invoking __get__ manually:
print(Desc.__dict__['cm'].__get__(None, Desc)())  # <class '__main__.Desc'>
print(Desc.__dict__['sm'].__get__(None, Desc)())  # 42
```

---

**Q3: Why is @classmethod preferred over @staticmethod for alternate constructors?**

**A:** Because `@classmethod` receives `cls` — the actual class on which the method was called, including subclasses. This makes the factory **subclass-aware**:

```python
class Shape:
    def __init__(self, color: str) -> None:
        self.color = color

    @classmethod
    def red(cls) -> "Shape":
        return cls("red")   # cls is Shape or any subclass

class Circle(Shape):
    pass

s = Shape.red()   # Shape("red")
c = Circle.red()  # Circle("red")  ← correct type, no extra code needed
```

If `red` were a `@staticmethod`, `cls` would not be available and it would always create a `Shape`, breaking subclass-based factories.

---

**Q4: Can you call a @classmethod on an instance? What is cls in that case?**

**A:** Yes. When you call `instance.classmethod()`, Python still passes the *class* of `instance` as `cls`, not the instance itself. This is identical to calling `type(instance).classmethod()`.

```python
class Base:
    @classmethod
    def who(cls) -> str:
        return cls.__name__

class Child(Base):
    pass

c = Child()
print(c.who())     # "Child"  — cls is Child, not Base
print(Base.who())  # "Base"
```

---

**Q5: What happens if you define a method without self and without @staticmethod?**

**A:** Python stores it as a plain function in `Class.__dict__`. Calling it on the class works (it's just a function call), but calling it on an instance raises `TypeError` because Python will try to pass the instance as the first argument, and the function accepts none.

```python
class Broken:
    def no_self():
        return 42

Broken.no_self()   # 42 — works, no instance passed
Broken().no_self() # TypeError: no_self() takes 0 positional arguments but 1 was given
```

Always use `@staticmethod` to make intent explicit and avoid this trap.

---

**Q6: How do @classmethod and inheritance interact?**

**A:** When a subclass inherits a `@classmethod`, calling it on the subclass passes the subclass as `cls`. This allows the classmethod to create instances of the subclass, access subclass-level class variables, and so on — without any override needed in the subclass.

```python
class Animal:
    sound = "..."

    @classmethod
    def describe(cls) -> str:
        return f"{cls.__name__} says {cls.sound}"

class Dog(Animal):
    sound = "Woof"

print(Dog.describe())   # "Dog says Woof"
print(Animal.describe()) # "Animal says ..."
```

---

**Q7: Is there any performance difference between the three method types?**

**A:** The overhead differences are negligible in practice. Creating a bound method involves a small allocation. `@staticmethod` avoids binding overhead since `__get__` returns the raw function. However, Python 3.7+ caches bound methods better than older versions, and the real cost is always the function body, not the binding. Micro-optimising method type choice for performance is almost never worthwhile.

---

**Q8: When should you NOT use @staticmethod?**

**A:** 

1. When the logic needs to access class state or be overridable per-subclass → use `@classmethod`.
2. When the function is reused in multiple classes or outside the class → use a module-level function.
3. When you're tempted to use `@staticmethod` to work around a design issue (e.g., a method that doesn't naturally belong in the class) → refactor.

Static methods are hardest to mock in tests (no polymorphism, no `cls`). If testability matters, a module-level function is easier to patch.

---

**Q9: How do you call a super() classmethod in a subclass?**

**A:** `super()` works naturally with `@classmethod`. Inside a subclass classmethod, `super().method()` resolves via MRO and still passes the correct `cls`.

```python
class Base:
    @classmethod
    def build(cls, x: int):
        obj = cls.__new__(cls)
        obj.x = x
        return obj

class Enhanced(Base):
    @classmethod
    def build(cls, x: int, y: int):
        obj = super().build(x)   # cls is still Enhanced here
        obj.y = y
        return obj

e = Enhanced.build(1, 2)
print(e.x, e.y)  # 1 2
print(type(e))   # <class '__main__.Enhanced'>
```

---

**Q10: What is the difference between @classmethod and @staticmethod in terms of inheritance and overriding?**

**A:** A `@classmethod` participates naturally in inheritance: the `cls` argument provides the actual subclass, and overriding the classmethod in a subclass replaces it with proper MRO. A `@staticmethod` can also be overridden, but since it receives no `cls`, the decision of which implementation runs depends solely on which class is used for the lookup — there is no runtime polymorphic dispatch via `cls`.

```python
class Parent:
    @classmethod
    def cm(cls): return f"Parent.cm via {cls.__name__}"
    @staticmethod
    def sm(): return "Parent.sm"

class Child(Parent):
    @classmethod
    def cm(cls): return f"Child.cm via {cls.__name__}"
    @staticmethod
    def sm(): return "Child.sm"

p: Parent = Child()
print(p.cm())  # "Child.cm via Child"  — dynamic dispatch on cls
print(p.sm())  # "Child.sm"  — resolved from type(p) == Child
```

---

## 3. Scenario-Based Interview Questions (10)

**Scenario 1: You are building a `Config` class that reads settings from environment variables, JSON files, or a dict. How do you structure the constructors?**

**Q:** Design the `Config` class with multiple creation paths without overloading `__init__`.

**A:** Use `@classmethod` factory methods as alternate constructors. Each factory method handles parsing and delegates to `__init__`.

```python
import json
import os
from typing import Any


class Config:
    def __init__(self, data: dict[str, Any]) -> None:
        self._data = data

    @classmethod
    def from_env(cls, prefix: str = "APP_") -> "Config":
        data = {
            k[len(prefix):].lower(): v
            for k, v in os.environ.items()
            if k.startswith(prefix)
        }
        return cls(data)

    @classmethod
    def from_json(cls, path: str) -> "Config":
        with open(path) as fh:
            return cls(json.load(fh))

    @classmethod
    def from_dict(cls, data: dict[str, Any]) -> "Config":
        return cls(dict(data))

    def get(self, key: str, default: Any = None) -> Any:
        return self._data.get(key, default)
```

This keeps `__init__` minimal and each factory is independently testable and readable.

---

**Scenario 2: A teammate defined a helper as an instance method, but it never uses `self`. The code review bot warns about this. What do you do?**

**Q:** How do you fix the method and why?

**A:** Convert it to `@staticmethod` (or move it to module level if reused elsewhere). Instance methods that ignore `self` mislead readers into thinking they depend on instance state.

```python
# Before (misleading)
class MathUtils:
    def square(self, x: float) -> float:   # self is never used
        return x * x

# After (correct)
class MathUtils:
    @staticmethod
    def square(x: float) -> float:
        return x * x

# Or if reused outside:
def square(x: float) -> float:
    return x * x
```

---

**Scenario 3: You need a class that tracks how many instances have been created. Implement it safely.**

**Q:** Where does the counter live, and how do you update it?

**A:** Use a class variable updated inside `__init__`. Optionally expose a `@classmethod` to reset or inspect it.

```python
class InstanceTracker:
    _count: int = 0

    def __init__(self) -> None:
        InstanceTracker._count += 1

    @classmethod
    def instance_count(cls) -> int:
        return cls._count

    @classmethod
    def reset(cls) -> None:
        cls._count = 0

a, b, c = InstanceTracker(), InstanceTracker(), InstanceTracker()
print(InstanceTracker.instance_count())  # 3
InstanceTracker.reset()
print(InstanceTracker.instance_count())  # 0
```

---

**Scenario 4: You inherit a `Payment` base class and need `Payment.from_stripe_event(event)` to always return the correct subclass based on a field in the event. How do you implement this?**

**Q:** Design the dispatch factory.

**A:** Use a `@classmethod` registry pattern:

```python
from __future__ import annotations


class Payment:
    _registry: dict[str, type[Payment]] = {}

    def __init_subclass__(cls, payment_type: str = "", **kwargs: object) -> None:
        super().__init_subclass__(**kwargs)
        if payment_type:
            Payment._registry[payment_type] = cls

    @classmethod
    def from_event(cls, event: dict) -> Payment:
        kind   = event["type"]
        klass  = cls._registry.get(kind, cls)
        return klass._from_event_data(event)

    @classmethod
    def _from_event_data(cls, event: dict) -> Payment:
        obj = cls.__new__(cls)
        obj.raw = event
        return obj


class ChargePayment(Payment, payment_type="charge"):
    pass

class RefundPayment(Payment, payment_type="refund"):
    pass

ev = {"type": "charge", "amount": 100}
p  = Payment.from_event(ev)
print(type(p))  # <class '__main__.ChargePayment'>
```

---

**Scenario 5: A junior developer is using `@staticmethod` for a validation function, but the validation rules differ per subclass. What do you recommend?**

**Q:** Explain the problem and the fix.

**A:** A `@staticmethod` cannot be overridden polymorphically via `cls`. Switch to `@classmethod`:

```python
class Validator:
    MIN_LENGTH = 3

    @classmethod
    def validate(cls, value: str) -> bool:
        return len(value) >= cls.MIN_LENGTH

class StrictValidator(Validator):
    MIN_LENGTH = 10

print(Validator.validate("hi"))        # False
print(StrictValidator.validate("hi"))  # False (uses StrictValidator.MIN_LENGTH=10)
```

Now the validation rule is polymorphically resolved via `cls`.

---

**Scenario 6: You need to write a unit test that checks a @classmethod factory. How do you approach it?**

**Q:** Write a test for `BankAccount.from_dict`.

**A:**

```python
import pytest
from bank import BankAccount   # hypothetical module

def test_from_dict_creates_correct_instance():
    data = {"owner": "Alice", "balance": 500.0}
    acc  = BankAccount.from_dict(data)
    assert isinstance(acc, BankAccount)
    assert acc.owner   == "Alice"
    assert acc.balance == 500.0

def test_from_dict_default_balance():
    acc = BankAccount.from_dict({"owner": "Bob"})
    assert acc.balance == 0.0
```

Classmethods are easy to test: call on the class, assert the result. No mocking needed for the simple factory pattern.

---

**Scenario 7: You're converting a procedural module full of functions into a class. Some functions don't use any shared state. How do you decide method type?**

**Q:** Provide a decision framework.

**A:** 

1. Does the function access or modify `self` (instance state)? → **instance method**
2. Does it access or modify class-level state, or need to return instances of the correct subclass? → **@classmethod**
3. Does it logically belong to the class conceptually but need no state? → **@staticmethod**
4. Is it useful outside this class? → **module-level function**

```python
class EmailService:
    _smtp_host: str = "smtp.example.com"

    def __init__(self, sender: str) -> None:
        self.sender = sender

    def send(self, to: str, body: str) -> None:         # instance method
        ...

    @classmethod
    def set_host(cls, host: str) -> None:               # class method
        cls._smtp_host = host

    @staticmethod
    def is_valid_email(email: str) -> bool:              # static: pure utility
        return "@" in email and "." in email.split("@")[-1]
```

---

**Scenario 8: How would you implement a Singleton using a classmethod?**

**Q:** Show the implementation and its limitations.

**A:**

```python
class Singleton:
    _instance: "Singleton | None" = None

    def __new__(cls) -> "Singleton":
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    @classmethod
    def get_instance(cls) -> "Singleton":
        if cls._instance is None:
            cls._instance = cls()
        return cls._instance

s1 = Singleton.get_instance()
s2 = Singleton.get_instance()
print(s1 is s2)  # True
```

Limitation: not thread-safe without a lock; subclassing Singleton can lead to unexpected shared `_instance`.

---

**Scenario 9: You have a data pipeline where each stage is a class. You want each stage to log its class name without hardcoding it. How?**

**Q:** Use the appropriate method type.

**A:** Use a `@classmethod` so that `cls.__name__` dynamically gives the correct class name even in subclasses:

```python
import logging

logger = logging.getLogger(__name__)

class PipelineStage:
    @classmethod
    def log_start(cls) -> None:
        logger.info("Starting stage: %s", cls.__name__)

    def process(self, data: list) -> list:
        self.log_start()
        return self._transform(data)

    def _transform(self, data: list) -> list:
        return data

class FilterStage(PipelineStage):
    def _transform(self, data: list) -> list:
        return [x for x in data if x is not None]

FilterStage().process([1, None, 2])
# Logs: "Starting stage: FilterStage"
```

---

**Scenario 10: A colleague says "just use @staticmethod everywhere to avoid the overhead of self". How do you respond?**

**Q:** Is this good advice?

**A:** No. The performance difference is negligible (nanoseconds), and this advice sacrifices:
1. **Polymorphism** — classmethods and instance methods participate in Python's method resolution and can be overridden meaningfully.
2. **Readability** — `self` communicates that the method depends on instance state; removing it misrepresents intent.
3. **Testability** — instance methods can be patched via `mock.patch.object`; static methods are harder to intercept.
4. **Design correctness** — if a method truly doesn't need `self` or `cls`, it probably belongs at module level.

Use the right tool for the job, not the one with the smallest constant-factor overhead.

---

## 4. Quick Recap / Cheatsheet

- **Instance method**: first arg `self`, accesses instance + class state. Default choice.
- **@classmethod**: first arg `cls` (the class), decorated with `@classmethod`. Use for factory constructors, class-level state mutation.
- **@staticmethod**: no `self`/`cls`, decorated with `@staticmethod`. Use for utilities logically grouped with the class but needing no state.
- **Descriptor protocol**: `function.__get__(instance, owner)` → bound method; `classmethod.__get__` → `cls`-bound callable; `staticmethod.__get__` → raw function.
- **Subclass factory**: `@classmethod` returns `cls(...)` so subclasses automatically get the right type.
- **@staticmethod pitfall**: not polymorphic via `cls` — can't adapt per subclass without hardcoding the class name.
- **Design rule**: if a method doesn't use `self` or `cls`, it smells like a module-level function.
- **Testing**: classmethods and instance methods are easier to mock/patch than staticmethods.
- **Calling classmethod on instance**: `instance.cm()` → `cls` = `type(instance)`, identical to `type(instance).cm()`.
- **super() with classmethod**: works naturally; `cls` in the child classmethod propagates correctly through `super()`.
