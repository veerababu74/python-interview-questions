# Composition vs Inheritance

## 1. Concept (Deep Explanation)

### What It Is and Why It Exists

Inheritance and composition are the two primary mechanisms for code reuse and relationship modelling in object-oriented programming. Understanding when to use each — and why — is one of the most important design skills an OOP developer can possess.

**Inheritance** models an **"is-a"** relationship. A `Dog` *is an* `Animal`. When you inherit from a class, you receive all of its methods and attributes, and the subclass is treated as a specialisation of the parent. Python uses a Method Resolution Order (MRO) via the C3 linearisation algorithm to determine which implementation to call when multiple parents are involved.

**Composition** models a **"has-a"** relationship. A `Car` *has an* `Engine`. The composing class holds a reference to another object and delegates work to it. The composed object is just an attribute; there is no inheritance relationship.

The Gang of Four principle **"Favour composition over inheritance"** (from *Design Patterns*, 1994) advises that most re-use problems are better solved by combining objects than by extending class hierarchies — because composition is more flexible, easier to test, and avoids the pitfalls of deep hierarchies.

### Internals / CPython Specifics

When Python looks up an attribute on an instance, it follows the MRO chain stored in `__mro__`. Each class in the hierarchy is searched in order. This is computed at class-creation time using the C3 linearisation:

```python
class A: pass
class B(A): pass
class C(A): pass
class D(B, C): pass

print(D.__mro__)
# (<class '__main__.D'>, <class '__main__.B'>, <class '__main__.C'>, <class '__main__.A'>, <class 'object'>)
```

Every class definition triggers `type.__new__`, which calculates the `__mro__` tuple and stores it on the class object. Attribute lookup (`LOAD_ATTR` bytecode) calls `type.__getattribute__`, which walks the MRO.

With composition, there is no MRO involved — attribute delegation is explicit Python code (a method call), which makes the flow completely transparent and debuggable.

### The Fragile Base Class Problem

This is the most serious pitfall of deep inheritance hierarchies. When a parent class changes its implementation, subclasses that relied on internal behaviour silently break.

```python
class Base:
    def process(self) -> list[int]:
        results = []
        for item in self._get_items():
            results.append(self._transform(item))
        return results

    def _get_items(self) -> list[int]:
        return [1, 2, 3]

    def _transform(self, item: int) -> int:
        return item * 2


class Child(Base):
    # Overrides _transform expecting it to be called by process()
    def _transform(self, item: int) -> int:
        return item * 3


# Works fine — prints [3, 6, 9]
print(Child().process())

# Now the library author refactors Base internally:
class Base:
    def process(self) -> list[int]:
        # Optimised — no longer calls _transform
        return [item * 2 for item in self._get_items()]

# Child._transform is now SILENTLY ignored — the override is dead code
# Child().process() returns [2, 4, 6], not [3, 6, 9]
```

The fragile base class problem occurs because inheritance exposes implementation details. Any internal method that a subclass overrides becomes an implicit part of the public API, whether the library author intends it or not.

### Interface Explosion with Deep Hierarchies

As inheritance hierarchies deepen, they tend to combine concerns in unnatural ways, leading to an explosion of subclasses.

```python
# Forced to create a cartesian product of classes:
class Animal: ...
class FlyingAnimal(Animal): ...
class SwimmingAnimal(Animal): ...
class FlyingSwimmingAnimal(FlyingAnimal, SwimmingAnimal): ...  # multiple inheritance needed!

# With composition:
class Animal:
    def __init__(self, locomotion: list["Locomotion"]) -> None:
        self.locomotion = locomotion

class Locomotion:
    def move(self) -> str: ...

class Flying(Locomotion):
    def move(self) -> str:
        return "flying"

class Swimming(Locomotion):
    def move(self) -> str:
        return "swimming"

duck = Animal(locomotion=[Flying(), Swimming()])
```

### Composition Patterns

**Containment / Delegation:**

```python
class Engine:
    def start(self) -> str:
        return "Engine started"

    def status(self) -> dict[str, str]:
        return {"rpm": "3000", "temp": "normal"}


class Car:
    def __init__(self) -> None:
        self._engine = Engine()  # Car HAS-AN Engine

    def start(self) -> str:
        return self._engine.start()  # delegation

    def engine_status(self) -> dict[str, str]:
        return self._engine.status()
```

**Protocol-based composition (structural subtyping):**

```python
from typing import Protocol

class Serialisable(Protocol):
    def to_dict(self) -> dict: ...

class JSONLogger:
    def log(self, obj: Serialisable) -> None:
        import json
        print(json.dumps(obj.to_dict()))

class User:
    def __init__(self, name: str, age: int) -> None:
        self.name = name
        self.age = age

    def to_dict(self) -> dict:
        return {"name": self.name, "age": self.age}

# User satisfies Serialisable without inheriting from anything
logger = JSONLogger()
logger.log(User("Alice", 30))
```

**Mixin as a middle ground:**

Mixins use inheritance syntax but model capabilities, not identity. They should not be instantiated on their own and should avoid `__init__`.

```python
class TimestampMixin:
    """Adds created_at / updated_at tracking — purely additive."""
    from datetime import datetime

    def touch(self) -> None:
        self.updated_at = self.datetime.now()  # type: ignore[attr-defined]


class JSONMixin:
    """Adds JSON serialisation capability."""
    import json

    def to_json(self) -> str:
        return self.json.dumps(self.__dict__)  # type: ignore[attr-defined]


class Article(TimestampMixin, JSONMixin):
    def __init__(self, title: str) -> None:
        self.title = title

a = Article("Python Deep Dive")
print(a.to_json())
```

Mixins work best when they:
- Do not inherit from the application-domain base class
- Are narrow and single-purpose
- Rely on duck typing rather than `super()` chains for cooperative multiple inheritance

### When Inheritance IS Appropriate

Use inheritance when:
1. The relationship is a **genuine is-a** — not just code reuse convenience
2. The **Liskov Substitution Principle (LSP)** holds
3. You want to expose a consistent interface (e.g., overriding abstract methods in an ABC)
4. The hierarchy is **shallow** (≤ 2 levels is a strong heuristic)

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

    @abstractmethod
    def perimeter(self) -> float: ...

class Circle(Shape):
    def __init__(self, radius: float) -> None:
        self.radius = radius

    def area(self) -> float:
        import math
        return math.pi * self.radius ** 2

    def perimeter(self) -> float:
        import math
        return 2 * math.pi * self.radius
```

This is correct use of inheritance: `Circle` IS-A `Shape`, the abstraction is stable, and LSP holds.

### Liskov Substitution Principle

LSP states: if `S` is a subtype of `T`, objects of type `T` may be replaced by objects of type `S` without altering the correctness of the program. Violations are a sure sign that inheritance is being misused.

```python
class Rectangle:
    def __init__(self, width: float, height: float) -> None:
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height


class Square(Rectangle):
    """LSP VIOLATION: Square IS-NOT-A Rectangle in a behavioural sense."""
    def __init__(self, side: float) -> None:
        super().__init__(side, side)

    @Rectangle.width.setter  # type: ignore
    def width(self, value: float) -> None:
        self._width = value
        self._height = value  # must keep sides equal


def double_width(rect: Rectangle) -> float:
    rect.width = rect.width * 2
    return rect.area()  # For Rectangle: expects height unchanged

r = Rectangle(4, 5)
print(double_width(r))   # 40.0 — correct

s = Square(4)
print(double_width(s))   # 64.0 — NOT 40.0, LSP violated
```

The fix: don't inherit. Model both as separate implementors of a `Shape` protocol.

### Real-World Refactoring: Inheritance → Composition

```python
# BEFORE: Inheritance-based notification system
class Notifier:
    def send(self, message: str) -> None:
        raise NotImplementedError

class EmailNotifier(Notifier):
    def send(self, message: str) -> None:
        print(f"Email: {message}")

class SMSNotifier(Notifier):
    def send(self, message: str) -> None:
        print(f"SMS: {message}")

class EmailAndSMSNotifier(EmailNotifier, SMSNotifier):
    """Multiple inheritance just to get both — fragile!"""
    def send(self, message: str) -> None:
        EmailNotifier.send(self, message)
        SMSNotifier.send(self, message)


# AFTER: Composition-based — clean and extensible
from typing import Protocol

class NotificationChannel(Protocol):
    def send(self, message: str) -> None: ...

class EmailChannel:
    def send(self, message: str) -> None:
        print(f"Email: {message}")

class SMSChannel:
    def send(self, message: str) -> None:
        print(f"SMS: {message}")

class SlackChannel:
    def send(self, message: str) -> None:
        print(f"Slack: {message}")

class NotificationService:
    def __init__(self, channels: list[NotificationChannel]) -> None:
        self._channels = channels

    def notify(self, message: str) -> None:
        for channel in self._channels:
            channel.send(message)

# Easily combine any number of channels without new subclasses
service = NotificationService([EmailChannel(), SMSChannel(), SlackChannel()])
service.notify("Deployment complete!")
```

### Pitfalls and Gotchas

- **Overriding `__init__` without calling `super().__init__`** breaks cooperative multiple inheritance
- **Diamond problem** — if multiple parents define the same method, MRO determines which wins; this can be surprising
- **`super()` in multiple inheritance** — `super()` follows MRO, not "call the parent"; each class in the chain must cooperatively call `super()` for it to work properly
- **Composition adds indirection** — sometimes a simple inheritance is clearer for truly stable, shallow hierarchies
- **Mixin ordering matters** — mixins should be listed before the main base class in the class definition

### Best Practices

- Default to composition; reach for inheritance only when you can clearly articulate an is-a relationship and LSP holds
- Keep hierarchies ≤ 2 levels deep
- Use `ABC` and abstract methods to define interfaces for inheritance
- Use `Protocol` for structural (duck-typed) composition
- Prefer mixins over multiple inheritance of concrete classes
- Make composed objects injectable (dependency injection) for easier testing

---

## 2. Conceptual Interview Questions (10)

**Q1: What does "favour composition over inheritance" actually mean in practice?**

**A:** It is a design heuristic from the Gang of Four advising that when you want to reuse behaviour, you should first consider whether the code can be placed in a separate object that is referenced as an attribute, rather than extending a parent class. Inheritance creates a tight coupling between parent and child at the language level — any change to the parent can silently affect children, and the child is forever tied to one inheritance chain. Composition keeps objects loosely coupled through well-defined interfaces.

In practice this means: before writing `class Dog(Animal)`, ask "is this genuinely an is-a, or am I just trying to reuse Animal's code?" If it's the latter, create a separate object (e.g., `Behaviour`, `MovementStrategy`) and inject it.

```python
# Instead of: class LoggingRepository(Repository): ...
# Prefer:
class LoggingRepository:
    def __init__(self, repo: Repository, logger: Logger) -> None:
        self._repo = repo
        self._logger = logger

    def save(self, entity) -> None:
        self._logger.info(f"Saving {entity}")
        self._repo.save(entity)
        self._logger.info("Saved")
```

**Q2: Explain the fragile base class problem with a concrete example.**

**A:** The fragile base class problem occurs when a parent class is changed in a way that breaks subclasses, even if the subclass code itself is untouched. It happens because inheritance exposes implementation details — private methods that subclasses override become implicit contracts.

Consider a base class `Collection` that calls a helper method `_notify()` internally. A subclass overrides `_notify()` to add logging. If the library author renames `_notify()` to `_dispatch()`, the subclass override becomes a dead method — but there is no error, no warning, just a silent behaviour change. This is particularly insidious in large codebases where the parent and child are maintained by different teams.

**Q3: What is the Liskov Substitution Principle and how does it guide the inheritance vs composition choice?**

**A:** LSP states that objects of a subtype must be substitutable for objects of their supertype without altering the correctness of the program. In other words, anywhere you use a `Shape`, you should be able to drop in a `Circle` or `Rectangle` without the calling code needing to know the difference or behaving incorrectly.

LSP violations are a strong signal that inheritance is being misused. The classic `Square extends Rectangle` example shows this: if you resize a Rectangle's width, you expect its height to stay the same; but a Square must keep both equal, breaking the expectation. The fix is to not use inheritance — either use composition or have both implement a common `Shape` interface.

If you cannot satisfy LSP, inheritance is the wrong tool. Use composition or a Protocol instead.

**Q4: How does Python's `Protocol` (from `typing`) support composition over inheritance?**

**A:** `Protocol` enables structural subtyping — an object satisfies a Protocol if it has the required methods and attributes, regardless of its inheritance tree. This means you can write functions and classes that accept "any object with a `.send()` method" without requiring that object to inherit from a specific base class.

```python
from typing import Protocol

class Sendable(Protocol):
    def send(self, message: str) -> None: ...

def broadcast(channels: list[Sendable], msg: str) -> None:
    for ch in channels:
        ch.send(msg)

class PushNotification:  # No inheritance needed
    def send(self, message: str) -> None:
        print(f"Push: {message}")

broadcast([PushNotification()], "Hello")
```

This is ideal for composition because the composed object just needs to satisfy the protocol; there is no coupling to a class hierarchy.

**Q5: What are mixins and how do they differ from regular inheritance?**

**A:** Mixins are classes designed to add specific functionality to other classes via multiple inheritance without representing an is-a relationship. They differ from regular inheritance in intent and design:

- A mixin should not be instantiated on its own
- A mixin should add a narrow, single capability
- A mixin typically does not call `__init__` or hold state (though it can)
- Mixins are listed first in the class definition (before the main base class) so their methods take precedence in the MRO

```python
class ReprMixin:
    def __repr__(self) -> str:
        attrs = ", ".join(f"{k}={v!r}" for k, v in self.__dict__.items())
        return f"{type(self).__name__}({attrs})"

class ValidationMixin:
    def validate(self) -> bool:
        return all(v is not None for v in self.__dict__.values())

class Product(ReprMixin, ValidationMixin):
    def __init__(self, name: str, price: float) -> None:
        self.name = name
        self.price = price

p = Product("Widget", 9.99)
print(repr(p))        # Product(name='Widget', price=9.99)
print(p.validate())   # True
```

**Q6: When is inheritance genuinely appropriate?**

**A:** Inheritance is appropriate when:
1. The is-a relationship is genuine and stable (won't change as the domain evolves)
2. LSP is satisfied — subclasses can stand in for the parent without surprises
3. The hierarchy is shallow (ideally one or two levels)
4. You are implementing a framework/library where users are expected to subclass and override abstract methods
5. The parent defines a stable interface (Abstract Base Class) that subclasses specialise

Example: `Exception` hierarchy in Python's stdlib is a good use of inheritance. `ValueError` IS-A `Exception`. The hierarchy is shallow. LSP holds. You can catch `Exception` and get a `ValueError` transparently.

**Q7: What is the diamond problem in multiple inheritance and how does Python handle it?**

**A:** The diamond problem occurs when a class inherits from two classes that both inherit from a common ancestor, creating a diamond-shaped inheritance graph. The question is: which ancestor's method gets called?

```python
class A:
    def hello(self) -> str:
        return "A"

class B(A):
    def hello(self) -> str:
        return "B"

class C(A):
    def hello(self) -> str:
        return "C"

class D(B, C):
    pass

print(D().hello())  # "B" — follows MRO: D -> B -> C -> A
print(D.__mro__)    # D, B, C, A, object
```

Python resolves this with C3 linearisation, which produces a consistent MRO. The rule: left-to-right, depth-first, but no class appears before all its subclasses. This makes multiple inheritance deterministic, but understanding what `super()` will call requires understanding the MRO.

**Q8: How does dependency injection relate to composition?**

**A:** Dependency Injection (DI) is a pattern where an object receives its dependencies from the outside rather than creating them internally. It is the practical application of composition: instead of a class hard-coding which implementation it uses, you inject the dependency (composed object) at construction time.

```python
class ReportGenerator:
    def __init__(self, formatter: "Formatter", storage: "Storage") -> None:
        self._formatter = formatter  # injected — no coupling to concrete type
        self._storage = storage

    def generate(self, data: dict) -> None:
        content = self._formatter.format(data)
        self._storage.save(content)

# Easy to swap implementations or mock in tests:
gen = ReportGenerator(
    formatter=CSVFormatter(),
    storage=S3Storage(bucket="reports"),
)
```

DI makes classes easier to test (inject mocks), easier to extend (inject new implementations), and avoids the fragile base class problem entirely.

**Q9: What are the trade-offs of composition over inheritance in terms of verbosity?**

**A:** Composition can be more verbose because delegation methods must be written explicitly. If a composed object has 20 methods and the composing class wants to expose all of them, you either write 20 delegation methods or use `__getattr__` magic.

```python
class LoggingList:
    def __init__(self) -> None:
        self._list: list = []

    # Must explicitly delegate each method we want to expose:
    def append(self, item) -> None:
        print(f"Appending {item}")
        self._list.append(item)

    def __len__(self) -> int:
        return len(self._list)

    def __getitem__(self, index):
        return self._list[index]
    # ... and so on
```

Python's `__getattr__` can reduce boilerplate, but it loses static type-checking benefits. The verbosity of composition is often considered a feature — it makes delegation explicit and intentional, unlike inheritance which silently exposes the entire parent API.

**Q10: How do Abstract Base Classes (ABCs) interact with the composition vs inheritance decision?**

**A:** ABCs define interfaces — they specify what methods a class *must* implement. Using `ABC` with abstract methods is one of the legitimate uses of inheritance: you're not reusing code, you're enforcing a contract.

```python
from abc import ABC, abstractmethod

class Repository(ABC):
    @abstractmethod
    def find_by_id(self, id: int) -> dict: ...

    @abstractmethod
    def save(self, entity: dict) -> None: ...

class SQLRepository(Repository):
    def find_by_id(self, id: int) -> dict: ...
    def save(self, entity: dict) -> None: ...
```

The key insight: `ABCMeta` (the metaclass behind `ABC`) keeps a registry of abstract methods and raises `TypeError` at instantiation time if any are unimplemented. This is a useful enforcement mechanism. However, Python's `Protocol` (from `typing`) achieves similar results without requiring inheritance at all, which is often preferable for composition.

---

## 3. Scenario-Based Interview Questions (10)

**S1: You are maintaining a payment processing library. The current design has `CreditCardPayment`, `PayPalPayment`, `CryptoPayment` all inheriting from `Payment`. A new requirement adds retry logic to all of them. How do you add this without duplication?**

**Q:** Design the retry mechanism.

**A:** Instead of adding retry to the base class (which breaks SRP) or duplicating across subclasses, compose a `RetryingPayment` wrapper:

```python
from typing import Protocol
import time

class PaymentProcessor(Protocol):
    def charge(self, amount: float) -> bool: ...

class RetryingPaymentProcessor:
    def __init__(
        self,
        processor: PaymentProcessor,
        max_attempts: int = 3,
        delay: float = 1.0,
    ) -> None:
        self._processor = processor
        self._max_attempts = max_attempts
        self._delay = delay

    def charge(self, amount: float) -> bool:
        for attempt in range(1, self._max_attempts + 1):
            try:
                return self._processor.charge(amount)
            except Exception as exc:
                if attempt == self._max_attempts:
                    raise
                print(f"Attempt {attempt} failed: {exc}. Retrying...")
                time.sleep(self._delay)
        return False

class CreditCardProcessor:
    def charge(self, amount: float) -> bool:
        print(f"Charging ${amount} to credit card")
        return True

# Compose — no subclassing needed
processor = RetryingPaymentProcessor(CreditCardProcessor(), max_attempts=3)
processor.charge(99.99)
```

**S2: Your codebase has `class AdminUser(User)`. The `User` class is being refactored to add async support, and every method signature is changing. How does this illustrate the fragile base class problem and what is a better design?**

**Q:** Redesign without tight coupling.

**A:** `AdminUser` is fragile because it relies on `User`'s internal method signatures. A better design uses composition:

```python
from dataclasses import dataclass
from typing import Protocol

class UserData(Protocol):
    name: str
    email: str

@dataclass
class User:
    name: str
    email: str

    def display(self) -> str:
        return f"{self.name} <{self.email}>"

class AdminPermissions:
    def __init__(self, user: User) -> None:
        self._user = user
        self.can_delete = True
        self.can_manage_users = True

    @property
    def display_name(self) -> str:
        return f"[Admin] {self._user.display()}"

# Now refactoring User doesn't break AdminPermissions unless the Protocol changes
admin_permissions = AdminPermissions(User("Alice", "alice@example.com"))
print(admin_permissions.display_name)
```

**S3: A junior developer created `class Stack(list)` to implement a stack data structure. What are the problems with this?**

**Q:** Analyse the design and propose a fix.

**A:** Inheriting from `list` violates LSP and exposes the entire list API (`.insert()`, `.sort()`, index access), violating encapsulation. Users can bypass the stack abstraction entirely.

```python
# PROBLEM:
class Stack(list):
    def push(self, item) -> None:
        self.append(item)
    def pop_top(self):
        return self.pop()

s = Stack()
s.push(1)
s.insert(0, 99)  # Bypasses stack discipline — inserts at arbitrary position!
s.sort()         # Completely nonsensical for a stack

# BETTER: Composition
class Stack:
    def __init__(self) -> None:
        self._data: list = []

    def push(self, item) -> None:
        self._data.append(item)

    def pop(self):
        if not self._data:
            raise IndexError("pop from empty stack")
        return self._data.pop()

    def peek(self):
        if not self._data:
            raise IndexError("peek at empty stack")
        return self._data[-1]

    def __len__(self) -> int:
        return len(self._data)

    def __bool__(self) -> bool:
        return bool(self._data)
```

**S4: You are designing a logging system. Different log handlers (File, Console, Network) need to be combined. Show two approaches and explain which is better.**

**Q:** Inheritance vs Composition for log handlers.

**A:**

```python
# COMPOSITION APPROACH — much better
from typing import Protocol

class LogHandler(Protocol):
    def emit(self, record: str) -> None: ...

class FileHandler:
    def __init__(self, path: str) -> None:
        self._path = path
    def emit(self, record: str) -> None:
        with open(self._path, "a") as f:
            f.write(record + "\n")

class ConsoleHandler:
    def emit(self, record: str) -> None:
        print(record)

class MultiHandler:
    def __init__(self, handlers: list[LogHandler]) -> None:
        self._handlers = handlers
    def emit(self, record: str) -> None:
        for handler in self._handlers:
            handler.emit(record)

class Logger:
    def __init__(self, handler: LogHandler) -> None:
        self._handler = handler
    def info(self, msg: str) -> None:
        self._handler.emit(f"INFO: {msg}")

logger = Logger(MultiHandler([ConsoleHandler(), FileHandler("app.log")]))
logger.info("Application started")
```

Composition wins because adding a `NetworkHandler` requires no change to existing code. An inheritance approach would require a new subclass for every combination.

**S5: Your team uses `class ServiceBase` with dozens of helper methods, and every service inherits it just to access `self.db`, `self.cache`, `self.logger`. Is this appropriate use of inheritance?**

**Q:** What is wrong and how would you fix it?

**A:** This is "inheritance for convenience" — a common anti-pattern. Services aren't `ServiceBase`; they just need access to shared resources. The fix is dependency injection:

```python
from dataclasses import dataclass

@dataclass
class ServiceDependencies:
    db: "Database"
    cache: "Cache"
    logger: "Logger"

class UserService:
    def __init__(self, deps: ServiceDependencies) -> None:
        self._db = deps.db
        self._cache = deps.cache
        self._logger = deps.logger

    def get_user(self, user_id: int) -> dict:
        cached = self._cache.get(f"user:{user_id}")
        if cached:
            return cached
        user = self._db.query("SELECT * FROM users WHERE id=?", user_id)
        self._cache.set(f"user:{user_id}", user)
        return user
```

Now `UserService` has no inheritance coupling. It's easy to test with mocked dependencies.

**S6: Describe a situation where you had to choose between a mixin and full inheritance. What was your decision process?**

**Q:** Walk through the decision.

**A:**

```python
# Situation: Multiple model classes need JSON serialisation.
# Option A: Inheritance from JsonModel base
class JsonModel:
    def to_json(self) -> str:
        import json
        return json.dumps(self.__dict__)

class User(JsonModel): ...  # Takes up the one "superclass slot"

# Option B: Mixin
class JsonMixin:
    def to_json(self) -> str:
        import json
        return json.dumps(self.__dict__)

class TimestampMixin:
    from datetime import datetime
    created_at: "datetime"

class User(JsonMixin, TimestampMixin):  # Can combine freely
    def __init__(self, name: str) -> None:
        self.name = name
```

Decision process: Is the class IS-A JsonModel? No — being JSON-serialisable is a capability, not an identity. Mixin is appropriate. This also preserves the single inheritance slot for a genuine domain base class (e.g., `Model` if using an ORM).

**S7: You are reviewing code where `HTTPSConnection` inherits from `HTTPConnection`, which inherits from `Socket`. What questions would you ask to validate this design?**

**Q:** How do you evaluate the inheritance hierarchy?

**A:** Questions to ask:
1. Does LSP hold at every level? Can I substitute `HTTPConnection` wherever `Socket` is expected?
2. Is `HTTPSConnection` IS-A `HTTPConnection` or does it just share connection setup code?
3. Does the deep hierarchy cause the fragile base class problem? If `Socket.connect()` changes signature, does it ripple through?
4. Would the code be clearer if `HTTPSConnection` composed an `HTTPConnection` rather than inheriting from it?

```python
# If HTTPS just wraps HTTP with TLS, composition is cleaner:
import ssl

class HTTPConnection:
    def __init__(self, host: str, port: int) -> None:
        self.host = host
        self.port = port

    def connect(self) -> "socket.socket":
        import socket
        sock = socket.create_connection((self.host, self.port))
        return sock

class HTTPSConnection:
    def __init__(self, host: str, port: int = 443) -> None:
        self._http = HTTPConnection(host, port)
        self._context = ssl.create_default_context()

    def connect(self) -> ssl.SSLSocket:
        sock = self._http.connect()
        return self._context.wrap_socket(sock, server_hostname=self._http.host)
```

**S8: How would you implement the Strategy pattern using composition in Python?**

**Q:** Implement a sorting service with swappable strategies.

**A:**

```python
from typing import Protocol

class SortStrategy(Protocol):
    def sort(self, data: list[int]) -> list[int]: ...

class BubbleSort:
    def sort(self, data: list[int]) -> list[int]:
        arr = data.copy()
        n = len(arr)
        for i in range(n):
            for j in range(0, n - i - 1):
                if arr[j] > arr[j + 1]:
                    arr[j], arr[j + 1] = arr[j + 1], arr[j]
        return arr

class QuickSort:
    def sort(self, data: list[int]) -> list[int]:
        if len(data) <= 1:
            return data
        pivot = data[len(data) // 2]
        left = [x for x in data if x < pivot]
        mid = [x for x in data if x == pivot]
        right = [x for x in data if x > pivot]
        return self.sort(left) + mid + self.sort(right)

class Sorter:
    def __init__(self, strategy: SortStrategy) -> None:
        self._strategy = strategy

    def set_strategy(self, strategy: SortStrategy) -> None:
        self._strategy = strategy  # Hot-swap at runtime!

    def sort(self, data: list[int]) -> list[int]:
        return self._strategy.sort(data)

sorter = Sorter(QuickSort())
print(sorter.sort([5, 3, 1, 4, 2]))
sorter.set_strategy(BubbleSort())
print(sorter.sort([5, 3, 1, 4, 2]))
```

**S9: A codebase has `class BaseView`, `class ListView(BaseView)`, `class DetailView(BaseView)`, `class CreateView(BaseView)` — Django-style class-based views. Is this good design?**

**Q:** Evaluate and suggest improvements.

**A:** This is a legitimate use of inheritance because:
- `ListView` IS-A `BaseView` in the domain sense
- The hierarchy is shallow
- `BaseView` is an intentional extension point
- LSP holds — any view can be used where a view is expected

However, the CBV pattern still has pitfalls:

```python
from abc import ABC, abstractmethod

class BaseView(ABC):
    def dispatch(self, request: dict) -> dict:
        method = request.get("method", "GET").lower()
        handler = getattr(self, f"handle_{method}", self.method_not_allowed)
        return handler(request)

    def method_not_allowed(self, request: dict) -> dict:
        return {"status": 405, "body": "Method Not Allowed"}

    @abstractmethod
    def handle_get(self, request: dict) -> dict: ...

class TemplateResponseMixin:
    template: str = ""

    def render(self, context: dict) -> str:
        return f"<html>Template: {self.template}, context: {context}</html>"

class ListView(TemplateResponseMixin, BaseView):
    model: type
    template = "list.html"

    def handle_get(self, request: dict) -> dict:
        items = []  # would query self.model
        return {"status": 200, "body": self.render({"items": items})}
```

The mixin approach (as Django does) adds capabilities without deepening the core hierarchy.

**S10: A candidate says "I always use inheritance because it's more Pythonic than composition." How do you respond?**

**Q:** Is this statement correct?

**A:** This is incorrect. Python strongly supports both patterns. Python's duck typing and `Protocol` actually *favour* composition — you don't need a shared base class to interoperate.

Evidence that Python's ecosystem favours composition:
- The standard library uses composition extensively (e.g., `io.TextIOWrapper` wraps `io.BufferedReader`)
- Python's `typing.Protocol` enables composition without inheritance overhead
- `functools.wraps`, decorators, and context managers are all composition patterns
- Django's class-based views use mixins (limited inheritance) + composition

The truly Pythonic approach is: **choose the right tool for the relationship**. Use inheritance for genuine is-a relationships with stable hierarchies. Use composition for code reuse, capability addition, and flexible runtime behaviour.

---

## 4. Quick Recap / Cheatsheet

- **Inheritance = is-a**, **Composition = has-a** — always verify the semantic relationship first
- **"Favour composition over inheritance"** = default to composition; justify inheritance explicitly
- **Fragile base class**: inheritance exposes internals; parent refactors silently break children
- **LSP**: if you can't substitute a subclass everywhere the parent is used, inheritance is wrong
- **Deep hierarchies** (>2 levels) are a red flag — consider flattening with composition
- **Mixins** add capabilities via multiple inheritance without modelling identity — keep them narrow and stateless
- **Protocol** (structural typing) enables composition without requiring a shared ancestor
- **Dependency injection** is composition in practice — inject dependencies, don't inherit them
- **Diamond problem** resolved by Python's C3 MRO — but avoid by preferring composition
- **When inheritance IS right**: shallow, stable, genuine is-a, ABC-enforced interface, LSP satisfied
- **Test signal**: code using inheritance is often harder to unit-test (tight coupling); composition enables easy mocking
- **`super()` follows MRO**, not "direct parent" — critical for cooperative multiple inheritance with mixins
