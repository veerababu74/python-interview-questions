# SOLID Principles in Python

## 1. Concept (Deep Explanation)

### S — Single Responsibility Principle

A class should have only one reason to change. One class = one job.

```python
# VIOLATION: UserService does authentication, email, AND persistence
class UserServiceBad:
    def __init__(self, db_url: str) -> None:
        import sqlite3
        self._conn = sqlite3.connect(db_url)

    def register(self, username: str, password: str, email: str) -> None:
        hashed = self._hash(password)
        self._conn.execute("INSERT INTO users VALUES (?, ?, ?)", (username, hashed, email))
        self._send_welcome_email(email)   # WHY is email here?!

    def _hash(self, password: str) -> str:
        import hashlib
        return hashlib.sha256(password.encode()).hexdigest()

    def _send_welcome_email(self, email: str) -> None:
        print(f"Sending email to {email}")   # Email logic inside user service!


# FIX: Split responsibilities
import hashlib

class PasswordHasher:
    def hash(self, plain: str) -> str:
        return hashlib.sha256(plain.encode()).hexdigest()

    def verify(self, plain: str, hashed: str) -> bool:
        return self.hash(plain) == hashed


class EmailService:
    def send_welcome(self, email: str) -> None:
        print(f"Sending welcome email to {email}")


class UserRepository:
    def save(self, username: str, hashed_pw: str, email: str) -> None:
        print(f"Saving user {username} to DB")


class UserRegistrationService:
    def __init__(
        self,
        repo: UserRepository,
        hasher: PasswordHasher,
        email_svc: EmailService,
    ) -> None:
        self._repo = repo
        self._hasher = hasher
        self._email = email_svc

    def register(self, username: str, password: str, email: str) -> None:
        hashed = self._hasher.hash(password)
        self._repo.save(username, hashed, email)
        self._email.send_welcome(email)
```

### O — Open/Closed Principle

Open for extension, closed for modification. Add behaviour without changing existing code.

```python
# VIOLATION: Adding a new discount type requires modifying existing code
class DiscountCalculatorBad:
    def calculate(self, order_type: str, price: float) -> float:
        if order_type == "regular":
            return price
        elif order_type == "vip":
            return price * 0.9
        elif order_type == "employee":
            return price * 0.7
        # Adding "student" requires editing this method → violation!


# FIX: Strategy/plugin pattern
from typing import Protocol

class DiscountStrategy(Protocol):
    def apply(self, price: float) -> float: ...

class RegularDiscount:
    def apply(self, price: float) -> float:
        return price

class VIPDiscount:
    def apply(self, price: float) -> float:
        return price * 0.9

class EmployeeDiscount:
    def apply(self, price: float) -> float:
        return price * 0.7

class StudentDiscount:    # NEW — no existing code changed!
    def apply(self, price: float) -> float:
        return price * 0.85

class DiscountCalculator:
    def __init__(self, strategy: DiscountStrategy) -> None:
        self._strategy = strategy

    def calculate(self, price: float) -> float:
        return self._strategy.apply(price)

calc = DiscountCalculator(VIPDiscount())
print(calc.calculate(100.0))   # 90.0
```

### L — Liskov Substitution Principle

Subtypes must be behaviorally substitutable for their base types.

```python
# VIOLATION: Square "is-a" Rectangle breaks LSP
class Rectangle:
    def __init__(self, width: float, height: float) -> None:
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height

class Square(Rectangle):
    def __init__(self, side: float) -> None:
        super().__init__(side, side)

    @Rectangle.width.setter  # type: ignore
    def width(self, v: float) -> None:
        self._width = v
        self._height = v  # must keep equal

def resize_and_check(rect: Rectangle) -> float:
    rect.width = 10
    return rect.area()   # Expects height unchanged

r = Rectangle(5, 3)
print(resize_and_check(r))   # 30 — correct

s = Square(5)
print(resize_and_check(s))   # 100, NOT 30 — LSP violated!


# FIX: Separate shapes, share a common Protocol
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class FixedRectangle(Shape):
    def __init__(self, width: float, height: float) -> None:
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height

class FixedSquare(Shape):
    def __init__(self, side: float) -> None:
        self.side = side

    def area(self) -> float:
        return self.side ** 2
```

### I — Interface Segregation Principle

Many small, focused interfaces are better than one large interface. Clients should not be forced to depend on methods they don't use.

```python
# VIOLATION: One fat interface forces all implementors to define everything
from abc import ABC, abstractmethod

class WorkerBad(ABC):
    @abstractmethod
    def work(self) -> None: ...
    @abstractmethod
    def eat(self) -> None: ...
    @abstractmethod
    def sleep(self) -> None: ...

class Robot(WorkerBad):
    def work(self) -> None:
        print("Robot working")
    def eat(self) -> None:
        raise NotImplementedError("Robots don't eat!")   # forced to stub!
    def sleep(self) -> None:
        raise NotImplementedError("Robots don't sleep!")


# FIX: Small, focused protocols
from typing import Protocol

class Workable(Protocol):
    def work(self) -> None: ...

class Eatable(Protocol):
    def eat(self) -> None: ...

class Sleepable(Protocol):
    def sleep(self) -> None: ...

class HumanWorker:
    def work(self) -> None: print("Human working")
    def eat(self) -> None: print("Human eating")
    def sleep(self) -> None: print("Human sleeping")

class RobotWorker:
    def work(self) -> None: print("Robot working")
    # No eat/sleep — doesn't need to stub them!

def manage_shift(worker: Workable) -> None:
    worker.work()

manage_shift(HumanWorker())
manage_shift(RobotWorker())   # Works — only Workable required
```

### D — Dependency Inversion Principle

High-level modules should not depend on low-level modules. Both should depend on abstractions. Abstractions should not depend on details.

```python
from typing import Protocol

# VIOLATION: High-level OrderService directly depends on low-level MySQLDatabase
class MySQLDatabase:
    def save_order(self, order: dict) -> None:
        print(f"MySQL: saving {order}")

class OrderServiceBad:
    def __init__(self) -> None:
        self._db = MySQLDatabase()   # tightly coupled!

    def place_order(self, order: dict) -> None:
        self._db.save_order(order)


# FIX: Depend on abstractions
class OrderDatabase(Protocol):
    def save_order(self, order: dict) -> None: ...

class MySQLOrderDB:
    def save_order(self, order: dict) -> None:
        print(f"MySQL: {order}")

class InMemoryOrderDB:
    def __init__(self) -> None:
        self._orders: list[dict] = []
    def save_order(self, order: dict) -> None:
        self._orders.append(order)

class OrderService:
    def __init__(self, db: OrderDatabase) -> None:   # depends on abstraction
        self._db = db

    def place_order(self, order: dict) -> None:
        # Validate, enrich...
        self._db.save_order(order)

# Inject whichever implementation fits:
svc = OrderService(MySQLOrderDB())
svc_test = OrderService(InMemoryOrderDB())   # easy to swap for tests
```

### How Python's Duck Typing Affects SOLID

Python's duck typing (and `Protocol`) means ISP and DIP are naturally lightweight — you don't need abstract base classes everywhere. A function that accepts any object with a `.save()` method already implements ISP/DIP via structural typing.

```python
# Python's duck typing makes DIP implicit:
def process(items, storage) -> None:
    """storage just needs .save(). No explicit Protocol required."""
    for item in items:
        storage.save(item)
```

However, explicit `Protocol` or `ABC` is still valuable for:
- Documentation of intent
- IDE autocompletion
- `isinstance()` checks
- Runtime validation with `runtime_checkable`

---

## 2. Conceptual Interview Questions (10)

**Q1: What is the Single Responsibility Principle and how do you identify a violation?**

**A:** SRP states a class should have only one reason to change. A violation exists when a class changes for multiple independent reasons — e.g., a `Report` class that changes when the reporting format changes AND when the data source changes.

Detection heuristics:
- The class name includes "and" or "manager" or "handler" (often a smell)
- Methods fall into clearly distinct groups (data access, formatting, notification)
- Changes to one feature require editing the class even though the feature seems unrelated

```python
# Classic violation — Report has 3 reasons to change:
class Report:
    def fetch_data(self) -> list: ...    # changes if DB schema changes
    def format_html(self) -> str: ...    # changes if HTML template changes
    def send_email(self) -> None: ...    # changes if email provider changes

# After SRP:
class ReportDataFetcher:
    def fetch(self) -> list: ...

class HTMLFormatter:
    def format(self, data: list) -> str: ...

class EmailSender:
    def send(self, content: str, to: str) -> None: ...
```

**Q2: Explain the Open/Closed Principle with a real-world Python example.**

**A:** OCP says software entities should be open for extension but closed for modification. Adding new behaviour should not require modifying existing, tested code.

In Python, the primary mechanisms for OCP are:
- **Strategy pattern**: inject a new strategy object
- **Inheritance with ABCs**: create a new subclass
- **Plugin/registry systems**: register new handlers

```python
from abc import ABC, abstractmethod

class NotificationChannel(ABC):
    @abstractmethod
    def send(self, message: str, recipient: str) -> None: ...

class EmailChannel(NotificationChannel):
    def send(self, message: str, recipient: str) -> None:
        print(f"Email to {recipient}: {message}")

class SlackChannel(NotificationChannel):
    def send(self, message: str, recipient: str) -> None:
        print(f"Slack to {recipient}: {message}")

# Adding SMS requires NO changes to existing code — just a new class:
class SMSChannel(NotificationChannel):
    def send(self, message: str, recipient: str) -> None:
        print(f"SMS to {recipient}: {message}")

class Notifier:
    def __init__(self, channels: list[NotificationChannel]) -> None:
        self._channels = channels

    def notify(self, message: str, recipient: str) -> None:
        for channel in self._channels:
            channel.send(message, recipient)
```

**Q3: How does LSP guide the decision between inheritance and composition?**

**A:** LSP is the litmus test for inheritance correctness. If a subclass cannot be substituted wherever its parent is expected — without altering correctness — the inheritance relationship is wrong.

The `Square extends Rectangle` problem is the classic example: a Rectangle's width can be set independently; a Square's cannot. Code that resizes width and checks area breaks for Squares.

Decision rule: if you cannot guarantee LSP, use composition or a shared Protocol instead of inheritance.

```python
from typing import Protocol

class Resizable(Protocol):
    def set_width(self, w: float) -> None: ...
    def area(self) -> float: ...

# Now Rectangle and Square implement Resizable independently
# — no inheritance, no LSP violation
```

**Q4: What is Interface Segregation in Python without formal interfaces?**

**A:** Python doesn't have Java-style interfaces, but ISP applies via `Protocol` (structural typing) or `ABC` (nominal typing). The principle: define small, focused protocols rather than one large one.

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Readable(Protocol):
    def read(self, n: int = -1) -> bytes: ...

@runtime_checkable
class Writable(Protocol):
    def write(self, data: bytes) -> int: ...

@runtime_checkable
class Seekable(Protocol):
    def seek(self, pos: int) -> int: ...

# Consumers only declare what they need:
def copy_data(src: Readable, dst: Writable) -> None:
    dst.write(src.read())

# A read-only stream satisfies Readable without needing write/seek stubs
class NetworkStream:
    def read(self, n: int = -1) -> bytes:
        return b"data"
    # No write/seek needed!

copy_data(NetworkStream(), open("out.bin", "wb"))
```

**Q5: Explain Dependency Inversion and how it enables testability.**

**A:** DIP says high-level modules (business logic) should depend on abstractions, not concrete implementations. This enables:
1. Easy swapping of implementations (prod vs test)
2. Parallel development (define the interface, implement independently)
3. Unit testing without infrastructure

```python
from typing import Protocol
from unittest.mock import MagicMock

class UserRepository(Protocol):
    def find_by_id(self, user_id: int) -> dict | None: ...
    def save(self, user: dict) -> None: ...

class UserService:
    def __init__(self, repo: UserRepository) -> None:
        self._repo = repo   # depends on abstraction

    def deactivate(self, user_id: int) -> None:
        user = self._repo.find_by_id(user_id)
        if user is None:
            raise ValueError(f"User {user_id} not found")
        user["active"] = False
        self._repo.save(user)

# Unit test — no real DB needed:
mock_repo = MagicMock()
mock_repo.find_by_id.return_value = {"id": 1, "name": "Alice", "active": True}
svc = UserService(mock_repo)
svc.deactivate(1)
mock_repo.save.assert_called_once()
```

**Q6: Which SOLID principles are most relevant in Python vs statically typed languages?**

**A:** All five matter, but with nuances:
- **SRP**: universally important — language-agnostic
- **OCP**: very relevant; Python's duck typing makes it naturally easy via Protocol/injection
- **LSP**: critical — Python has no compile-time enforcement, so violations are silent bugs
- **ISP**: Python's `Protocol` makes this effortless — small structural protocols are idiomatic
- **DIP**: Python's duck typing partially implements this implicitly; explicit Protocol/ABC makes it formal

LSP is arguably *more* important in Python because the type checker won't catch substitution violations without explicit typing and mypy/pyright.

**Q7: Show a common SRP violation in a Django/Flask view and how to fix it.**

**A:**

```python
# VIOLATION: View does validation, business logic, AND formatting
def create_order_bad(request: dict) -> dict:
    # Validation
    if "product_id" not in request:
        return {"error": "product_id required", "status": 400}
    if request.get("quantity", 0) <= 0:
        return {"error": "quantity must be positive", "status": 400}
    # Business logic
    import random
    order_id = random.randint(1000, 9999)
    total = request["quantity"] * 9.99  # hardcoded price!
    # Persistence
    print(f"DB: INSERT order {order_id}")
    # Email
    print("Email: order confirmation sent")
    return {"order_id": order_id, "total": total, "status": 201}


# FIX: Each function/class has one job
from dataclasses import dataclass

@dataclass
class OrderRequest:
    product_id: int
    quantity: int

    def validate(self) -> list[str]:
        errors = []
        if self.quantity <= 0:
            errors.append("quantity must be positive")
        return errors

class PricingService:
    def calculate_total(self, product_id: int, quantity: int) -> float:
        prices = {1: 9.99, 2: 19.99}
        return prices.get(product_id, 9.99) * quantity

class OrderRepository:
    def create(self, product_id: int, quantity: int, total: float) -> int:
        import random
        order_id = random.randint(1000, 9999)
        print(f"DB: saving order {order_id}")
        return order_id

def create_order(request: dict, pricing: PricingService, repo: OrderRepository) -> dict:
    try:
        order_req = OrderRequest(**{k: request[k] for k in ("product_id", "quantity")})
    except KeyError as e:
        return {"error": f"Missing field: {e}", "status": 400}

    errors = order_req.validate()
    if errors:
        return {"error": errors, "status": 400}

    total = pricing.calculate_total(order_req.product_id, order_req.quantity)
    order_id = repo.create(order_req.product_id, order_req.quantity, total)
    return {"order_id": order_id, "total": total, "status": 201}
```

**Q8: How does the Strategy pattern implement OCP?**

**A:** The Strategy pattern is the canonical implementation of OCP. The context class (high-level) never changes when new behaviours are added — only new strategy classes are created.

```python
from typing import Protocol

class SortStrategy(Protocol):
    def sort(self, data: list) -> list: ...

class BubbleSort:
    def sort(self, data: list) -> list:
        arr = data[:]
        for i in range(len(arr)):
            for j in range(len(arr)-i-1):
                if arr[j] > arr[j+1]:
                    arr[j], arr[j+1] = arr[j+1], arr[j]
        return arr

class QuickSort:
    def sort(self, data: list) -> list:
        if len(data) <= 1:
            return data
        pivot = data[len(data)//2]
        return (
            self.sort([x for x in data if x < pivot]) +
            [x for x in data if x == pivot] +
            self.sort([x for x in data if x > pivot])
        )

# Context never changes — only strategies are added/swapped:
class DataProcessor:
    def __init__(self, sorter: SortStrategy) -> None:
        self._sorter = sorter

    def process(self, data: list) -> list:
        return self._sorter.sort(data)

processor = DataProcessor(QuickSort())
print(processor.process([3, 1, 4, 1, 5]))
```

**Q9: What is "Tell, Don't Ask" and how does it relate to SRP?**

**A:** "Tell, Don't Ask" is a related principle: instead of asking an object for its data and making decisions outside, tell the object to do something. It prevents logic from leaking out of classes (SRP violation).

```python
# VIOLATION (asking): Controller decides what to do with order's data
class Order:
    def __init__(self, items: list, status: str) -> None:
        self.items = items
        self.status = status

def process_order_bad(order: Order) -> None:
    if order.status == "pending" and len(order.items) > 0:
        # Controller knows too much about Order internals!
        order.status = "processing"
        print("Processing...")

# TELL (better): Order knows how to process itself
class Order:
    def __init__(self, items: list, status: str = "pending") -> None:
        self.items = items
        self.status = status

    def can_process(self) -> bool:
        return self.status == "pending" and bool(self.items)

    def process(self) -> None:
        if not self.can_process():
            raise ValueError("Order cannot be processed")
        self.status = "processing"
        print("Order is processing")

order = Order([{"sku": "WIDGET"}])
order.process()   # Tell the order to process itself
```

**Q10: How do you apply DIP in a Python application without a DI framework?**

**A:** Python doesn't need a DI framework — constructor injection with Protocol types is sufficient:

```python
from typing import Protocol
from dataclasses import dataclass

class Cache(Protocol):
    def get(self, key: str) -> bytes | None: ...
    def set(self, key: str, value: bytes, ttl: int = 300) -> None: ...

class Logger(Protocol):
    def info(self, msg: str) -> None: ...
    def error(self, msg: str) -> None: ...

class ProductService:
    def __init__(self, cache: Cache, logger: Logger) -> None:
        self._cache = cache
        self._logger = logger

    def get_product(self, product_id: int) -> dict:
        key = f"product:{product_id}"
        cached = self._cache.get(key)
        if cached:
            import json
            self._logger.info(f"Cache hit for {key}")
            return json.loads(cached)
        product = {"id": product_id, "name": "Widget"}  # would query DB
        import json
        self._cache.set(key, json.dumps(product).encode())
        return product

# Wire up in application factory:
class InMemoryCache:
    def __init__(self) -> None:
        self._store: dict = {}
    def get(self, key: str) -> bytes | None:
        return self._store.get(key)
    def set(self, key: str, value: bytes, ttl: int = 300) -> None:
        self._store[key] = value

class PrintLogger:
    def info(self, msg: str) -> None: print(f"INFO: {msg}")
    def error(self, msg: str) -> None: print(f"ERROR: {msg}")

svc = ProductService(InMemoryCache(), PrintLogger())
print(svc.get_product(1))
```

---

## 3. Scenario-Based Interview Questions (10)

**S1: You see a `PaymentProcessor` class with 500 lines handling card validation, currency conversion, fraud detection, and logging. Refactor it.**

**Q:** Apply SRP to split the class.

**A:**

```python
from typing import Protocol
from dataclasses import dataclass

@dataclass
class PaymentRequest:
    amount: float
    currency: str
    card_number: str
    cvv: str

class CardValidator:
    def validate(self, card: str, cvv: str) -> bool:
        return len(card) == 16 and card.isdigit() and len(cvv) == 3

class CurrencyConverter:
    _rates = {"USD": 1.0, "EUR": 0.85, "GBP": 0.73}

    def to_usd(self, amount: float, currency: str) -> float:
        rate = self._rates.get(currency, 1.0)
        return amount / rate

class FraudDetector:
    def is_suspicious(self, amount: float, card: str) -> bool:
        return amount > 10_000  # simplified

class PaymentLogger:
    def log_attempt(self, req: PaymentRequest, success: bool) -> None:
        status = "SUCCESS" if success else "FAILED"
        print(f"Payment {status}: ${req.amount} {req.currency}")

class PaymentGateway(Protocol):
    def charge(self, amount_usd: float, card: str) -> bool: ...

class PaymentProcessor:
    def __init__(
        self,
        validator: CardValidator,
        converter: CurrencyConverter,
        fraud: FraudDetector,
        logger: PaymentLogger,
        gateway: PaymentGateway,
    ) -> None:
        self._validator = validator
        self._converter = converter
        self._fraud = fraud
        self._logger = logger
        self._gateway = gateway

    def process(self, req: PaymentRequest) -> bool:
        if not self._validator.validate(req.card_number, req.cvv):
            self._logger.log_attempt(req, False)
            return False
        amount_usd = self._converter.to_usd(req.amount, req.currency)
        if self._fraud.is_suspicious(amount_usd, req.card_number):
            self._logger.log_attempt(req, False)
            return False
        success = self._gateway.charge(amount_usd, req.card_number)
        self._logger.log_attempt(req, success)
        return success
```

**S2: Your `ReportGenerator` has a long `if/elif` chain for different output formats. Apply OCP.**

**Q:** Refactor using OCP.

**A:**

```python
from typing import Protocol
from abc import ABC, abstractmethod

class ReportFormatter(Protocol):
    def format(self, data: list[dict]) -> str: ...

class CSVFormatter:
    def format(self, data: list[dict]) -> str:
        if not data:
            return ""
        headers = ",".join(data[0].keys())
        rows = [",".join(str(v) for v in row.values()) for row in data]
        return "\n".join([headers] + rows)

class JSONFormatter:
    def format(self, data: list[dict]) -> str:
        import json
        return json.dumps(data, indent=2)

class HTMLFormatter:
    def format(self, data: list[dict]) -> str:
        if not data:
            return "<table></table>"
        headers = "".join(f"<th>{k}</th>" for k in data[0].keys())
        rows = "".join(
            "<tr>" + "".join(f"<td>{v}</td>" for v in row.values()) + "</tr>"
            for row in data
        )
        return f"<table><thead><tr>{headers}</tr></thead><tbody>{rows}</tbody></table>"

# Markdown added without touching existing code:
class MarkdownFormatter:
    def format(self, data: list[dict]) -> str:
        if not data:
            return ""
        headers = " | ".join(data[0].keys())
        sep = " | ".join(["---"] * len(data[0]))
        rows = [" | ".join(str(v) for v in row.values()) for row in data]
        return "\n".join([headers, sep] + rows)

class ReportGenerator:
    def __init__(self, formatter: ReportFormatter) -> None:
        self._formatter = formatter

    def generate(self, data: list[dict]) -> str:
        return self._formatter.format(data)

data = [{"name": "Alice", "score": 95}, {"name": "Bob", "score": 87}]
gen = ReportGenerator(MarkdownFormatter())
print(gen.generate(data))
```

**S3: A `Bird` class has a `fly()` method. `Penguin` subclasses it but throws `NotImplementedError` from `fly()`. Fix the LSP violation.**

**Q:** Redesign to satisfy LSP.

**A:**

```python
from typing import Protocol
from abc import ABC, abstractmethod

# VIOLATION:
class BirdBad(ABC):
    @abstractmethod
    def fly(self) -> str: ...

class PenguinBad(BirdBad):
    def fly(self) -> str:
        raise NotImplementedError("Penguins can't fly!")  # LSP violated!


# FIX: Separate capability from identity
class Bird(ABC):
    @abstractmethod
    def eat(self) -> str: ...
    @abstractmethod
    def breathe(self) -> str: ...

class FlyingBird(Bird):
    @abstractmethod
    def fly(self) -> str: ...

class SwimmingBird(Bird):
    @abstractmethod
    def swim(self) -> str: ...

class Eagle(FlyingBird):
    def eat(self) -> str: return "Eagle eating"
    def breathe(self) -> str: return "Eagle breathing"
    def fly(self) -> str: return "Eagle soaring"

class Penguin(SwimmingBird):
    def eat(self) -> str: return "Penguin eating"
    def breathe(self) -> str: return "Penguin breathing"
    def swim(self) -> str: return "Penguin swimming"

# Now no code that requires FlyingBird can receive a Penguin — LSP enforced!
def make_fly(bird: FlyingBird) -> str:
    return bird.fly()

print(make_fly(Eagle()))
# make_fly(Penguin())  # Type error — Penguin is not FlyingBird
```

**S4: A logging library has a `Logger` interface with `debug`, `info`, `warning`, `error`, `critical`, and `format_json`, `format_csv`, `write_file`, `write_network`. ISP says this is wrong — fix it.**

**Q:** Apply ISP.

**A:**

```python
from typing import Protocol

class LogEmitter(Protocol):
    def debug(self, msg: str) -> None: ...
    def info(self, msg: str) -> None: ...
    def warning(self, msg: str) -> None: ...
    def error(self, msg: str) -> None: ...

class LogFormatter(Protocol):
    def format(self, level: str, msg: str) -> str: ...

class LogWriter(Protocol):
    def write(self, data: str) -> None: ...

class JSONLogFormatter:
    def format(self, level: str, msg: str) -> str:
        import json
        return json.dumps({"level": level, "msg": msg})

class FileLogWriter:
    def __init__(self, path: str) -> None:
        self._path = path
    def write(self, data: str) -> None:
        with open(self._path, "a") as f:
            f.write(data + "\n")

class ConsoleLogWriter:
    def write(self, data: str) -> None:
        print(data)

class Logger:
    def __init__(self, formatter: LogFormatter, writer: LogWriter) -> None:
        self._formatter = formatter
        self._writer = writer

    def info(self, msg: str) -> None:
        self._writer.write(self._formatter.format("INFO", msg))

    def error(self, msg: str) -> None:
        self._writer.write(self._formatter.format("ERROR", msg))

logger = Logger(JSONLogFormatter(), ConsoleLogWriter())
logger.info("Application started")
logger.error("Something went wrong")
```

**S5: Your service class directly instantiates its database connection. Unit tests are slow because of real DB calls. Apply DIP to fix this.**

**Q:** Refactor for testability.

**A:**

```python
from typing import Protocol
from unittest.mock import MagicMock

class UserStore(Protocol):
    def find(self, user_id: int) -> dict | None: ...
    def update(self, user: dict) -> None: ...

class UserService:
    def __init__(self, store: UserStore) -> None:
        self._store = store

    def activate(self, user_id: int) -> bool:
        user = self._store.find(user_id)
        if user is None:
            return False
        user["active"] = True
        self._store.update(user)
        return True

# Production:
class PostgresUserStore:
    def find(self, user_id: int) -> dict | None:
        return {"id": user_id, "active": False}  # would query real DB
    def update(self, user: dict) -> None:
        print(f"DB: updating user {user['id']}")

# Test — no DB:
mock_store = MagicMock(spec=UserStore)
mock_store.find.return_value = {"id": 1, "name": "Alice", "active": False}
svc = UserService(mock_store)
result = svc.activate(1)
assert result is True
mock_store.update.assert_called_once()
print("All tests passed!")
```

**S6: An analytics service computes metrics but also sends them to Datadog, logs them to a file, and sends Slack alerts. Which principles are violated and how do you fix?**

**Q:** Identify and fix SRP + DIP violations.

**A:**

```python
from typing import Protocol

class MetricsPublisher(Protocol):
    def publish(self, metric: str, value: float, tags: dict) -> None: ...

class AlertService(Protocol):
    def alert(self, message: str, severity: str) -> None: ...

class MetricsLogger(Protocol):
    def log(self, metric: str, value: float) -> None: ...

class DatadogPublisher:
    def publish(self, metric: str, value: float, tags: dict) -> None:
        print(f"Datadog: {metric}={value} {tags}")

class SlackAlerter:
    def alert(self, message: str, severity: str) -> None:
        print(f"Slack [{severity}]: {message}")

class FileMetricsLogger:
    def log(self, metric: str, value: float) -> None:
        print(f"FILE: {metric}={value}")

class AnalyticsService:
    def __init__(
        self,
        publisher: MetricsPublisher,
        alerter: AlertService,
        logger: MetricsLogger,
    ) -> None:
        self._publisher = publisher
        self._alerter = alerter
        self._logger = logger

    def record_latency(self, endpoint: str, latency_ms: float) -> None:
        self._publisher.publish("http.latency", latency_ms, {"endpoint": endpoint})
        self._logger.log(f"http.latency.{endpoint}", latency_ms)
        if latency_ms > 1000:
            self._alerter.alert(f"High latency on {endpoint}: {latency_ms}ms", "warning")

svc = AnalyticsService(DatadogPublisher(), SlackAlerter(), FileMetricsLogger())
svc.record_latency("/api/users", 1500)
```

**S7: Show how `__init_subclass__` can enforce OCP for a plugin system.**

**Q:** Implement OCP-compliant plugin registry.

**A:**

```python
from abc import ABC, abstractmethod

class DataProcessor(ABC):
    _processors: dict[str, type["DataProcessor"]] = {}

    def __init_subclass__(cls, format: str = "", **kwargs) -> None:
        super().__init_subclass__(**kwargs)
        if format:
            DataProcessor._processors[format] = cls

    @abstractmethod
    def process(self, data: bytes) -> dict: ...

    @classmethod
    def for_format(cls, fmt: str) -> "DataProcessor":
        if fmt not in cls._processors:
            raise ValueError(f"No processor for {fmt!r}. Available: {list(cls._processors)}")
        return cls._processors[fmt]()


class JSONProcessor(DataProcessor, format="json"):
    def process(self, data: bytes) -> dict:
        import json
        return json.loads(data)

class CSVProcessor(DataProcessor, format="csv"):
    def process(self, data: bytes) -> dict:
        lines = data.decode().strip().split("\n")
        headers = lines[0].split(",")
        return {"headers": headers, "rows": len(lines) - 1}

# Adding XML — zero changes to existing code:
class XMLProcessor(DataProcessor, format="xml"):
    def process(self, data: bytes) -> dict:
        return {"raw_xml": data.decode()}

proc = DataProcessor.for_format("json")
print(proc.process(b'{"key": "value"}'))
```

**S8: A `Shape` hierarchy has concrete classes. The `AreaCalculator` has if/elif for each type. Apply OCP.**

**Q:** Eliminate type-checking with OCP.

**A:**

```python
from abc import ABC, abstractmethod
import math

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class Circle(Shape):
    def __init__(self, radius: float) -> None:
        self.radius = radius
    def area(self) -> float:
        return math.pi * self.radius ** 2

class Rectangle(Shape):
    def __init__(self, w: float, h: float) -> None:
        self.w = w
        self.h = h
    def area(self) -> float:
        return self.w * self.h

class Triangle(Shape):
    def __init__(self, base: float, height: float) -> None:
        self.base = base
        self.height = height
    def area(self) -> float:
        return 0.5 * self.base * self.height

# OCP-compliant calculator — never needs to change for new shapes:
class AreaCalculator:
    def total_area(self, shapes: list[Shape]) -> float:
        return sum(s.area() for s in shapes)

    def largest(self, shapes: list[Shape]) -> Shape:
        return max(shapes, key=lambda s: s.area())

calc = AreaCalculator()
shapes: list[Shape] = [Circle(5), Rectangle(3, 4), Triangle(6, 8)]
print(f"Total area: {calc.total_area(shapes):.2f}")
print(f"Largest: {type(calc.largest(shapes)).__name__}")
```

**S9: Explain how Python's `typing.Protocol` with `@runtime_checkable` helps implement ISP.**

**Q:** Show runtime-checkable Protocol for ISP.

**A:**

```python
from typing import Protocol, runtime_checkable

@runtime_checkable
class Closeable(Protocol):
    def close(self) -> None: ...

@runtime_checkable
class Flushable(Protocol):
    def flush(self) -> None: ...

@runtime_checkable
class Readable(Protocol):
    def read(self, n: int = -1) -> bytes: ...

# Consumers declare minimal requirements:
def safe_close(resource: Closeable) -> None:
    resource.close()

def flush_if_possible(resource) -> None:
    if isinstance(resource, Flushable):  # runtime check!
        resource.flush()

class NetworkSocket:
    def read(self, n: int = -1) -> bytes:
        return b"data"
    def close(self) -> None:
        print("Socket closed")
    # No flush — and that's fine!

class BufferedFile:
    def read(self, n: int = -1) -> bytes:
        return b"file data"
    def flush(self) -> None:
        print("Buffer flushed")
    def close(self) -> None:
        print("File closed")

sock = NetworkSocket()
buf = BufferedFile()

print(isinstance(sock, Readable))   # True
print(isinstance(sock, Flushable))  # False — no flush method
print(isinstance(buf, Flushable))   # True

flush_if_possible(sock)   # Does nothing — not Flushable
flush_if_possible(buf)    # "Buffer flushed"
safe_close(sock)          # "Socket closed"
```

**S10: A candidate claims "Python's duck typing means SOLID doesn't apply." How do you respond?**

**Q:** Evaluate the claim.

**A:** The claim is wrong. SOLID is about design principles for maintainable, extensible, testable software — they apply regardless of language. Duck typing doesn't eliminate the need for good design; it changes the *mechanism* but not the *goal*.

Specifically:
- **SRP**: still violated when a class has multiple reasons to change, regardless of typing
- **OCP**: duck typing makes it *easier* (no need for explicit inheritance for extension) but the principle still guides when to use inheritance vs composition
- **LSP**: *more* critical in Python because violations are silent at runtime without type checkers
- **ISP**: Protocol makes small interfaces effortless — duck typing *enables* ISP, not eliminates it
- **DIP**: Python's duck typing means `def process(storage)` implicitly accepts any storage — an informal DIP. Explicit Protocol makes contracts clear and documented.

```python
# Duck typing gives you DIP "for free" — but explicit Protocol is better for teams:
from typing import Protocol

class Storage(Protocol):
    def save(self, data: dict) -> None: ...  # explicit contract

class S3Storage:
    def save(self, data: dict) -> None:
        print(f"S3: {data}")

class LocalStorage:
    def save(self, data: dict) -> None:
        print(f"Local: {data}")

def process(data: dict, storage: Storage) -> None:  # DIP via Protocol
    storage.save(data)

# Duck typing: ANY object with .save() works — no isinstance checks needed
# But the Protocol documents the contract explicitly
```

---

## 4. Quick Recap / Cheatsheet

- **SRP**: one reason to change; split classes by domain concept (validation, persistence, notification)
- **OCP**: extend via new classes/strategies, not by modifying existing code; Strategy pattern is the key tool
- **LSP**: subclasses must be substitutable for parents; if LSP breaks, use composition or Protocol instead
- **ISP**: many small `Protocol`s > one large ABC; consumers declare only what they need
- **DIP**: high-level depends on abstractions (`Protocol`/`ABC`), not concrete classes; inject dependencies
- **Python tools for SOLID**: `Protocol` (ISP, DIP), `ABC` (LSP enforcement, OCP), decorators (OCP), `__init_subclass__` (OCP plugin systems)
- **Duck typing aids OCP and DIP** — any conforming object is accepted without inheritance
- **LSP is hardest to enforce** in Python without static type checking (mypy/pyright)
- **Prefer `Protocol` over `ABC`** for DIP when you don't own the implementations
- **Detect SRP violation**: class name has "and", "manager", "handler"; methods group into unrelated buckets
- **`isinstance` anti-pattern**: if/elif on types usually means OCP or LSP violation — use polymorphism
- **Test signal**: SRP violations are hard to unit-test (too many dependencies); DIP violations require real infrastructure in tests
- **`@runtime_checkable Protocol`**: enables `isinstance()` checks against structural types — key for ISP
- **SOLID is orthogonal to duck typing**: typing style changes the mechanism, not the design principle
