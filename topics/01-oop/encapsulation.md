# Encapsulation

## 1. Concept (Deep Explanation)

### What Is Encapsulation?

Encapsulation is the OOP principle of **bundling data (attributes) and behaviour (methods) together into a single unit (the class)** while **controlling access to the internal state** from outside code. It exists for several reasons:

1. **Data integrity** — Prevent external code from putting the object in an invalid state.
2. **Implementation hiding** — Allow the internal representation to change without breaking callers.
3. **API stability** — Define a clear public interface and keep everything else private.

In strongly-typed languages like Java and C++, encapsulation is enforced by the runtime: the compiler refuses to compile code that accesses `private` members. Python takes a fundamentally different approach.

### Python's Philosophy: "We're All Consenting Adults"

Python's culture holds that programmers should be trusted to respect documented conventions. The language provides **hints** about access levels but enforces nothing at runtime (with one minor exception: name mangling). This is sometimes called the *consenting adults* or *open kimono* philosophy.

The practical result:
- A single leading underscore `_` signals *"internal, use at your own risk"*.
- A double leading underscore `__` triggers **name mangling** — a syntactic transformation, not a true access restriction.

### Single Underscore: The "Protected" Convention

```python
class BankAccount:
    def __init__(self, balance: float) -> None:
        self._balance = balance   # "protected" by convention

    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Must be positive")
        self._balance += amount

acc = BankAccount(100.0)
# Python does NOT prevent this — it's just impolite:
acc._balance = -999.0   # works fine, but breaks the invariant
```

Single-underscore attributes are visible to subclasses and are *expected* to be accessed within the class hierarchy. They signal "not part of the public API, but not deeply private either."

### Double Underscore: Name Mangling (NOT True Private)

When an identifier has **two or more leading underscores** and **at most one trailing underscore** inside a class body, CPython renames it:

```
__attr  →  _ClassName__attr
```

This happens at **compile time** during the parsing of the class body. It is a purely lexical/syntactic transformation — not an access check at runtime.

```python
class Vault:
    def __init__(self, secret: str) -> None:
        self.__secret = secret     # stored as _Vault__secret

    def reveal(self) -> str:
        return self.__secret       # inside the class, __secret is fine
                                   # CPython transforms this to self._Vault__secret

v = Vault("my-secret")

# Attempting direct access fails:
try:
    print(v.__secret)              # AttributeError
except AttributeError as e:
    print(e)

# But the mangled name is fully accessible:
print(v._Vault__secret)            # "my-secret" — NOT truly private

# Inspect the instance dict:
print(vars(v))   # {'_Vault__secret': 'my-secret'}
```

Name mangling's **intended purpose** is to **avoid accidental override in subclasses**, not to enforce secrecy:

```python
class Base:
    def __init__(self) -> None:
        self.__id = 42   # stored as _Base__id

class Child(Base):
    def __init__(self) -> None:
        super().__init__()
        self.__id = 99   # stored as _Child__id — different attribute!

c = Child()
print(vars(c))   # {'_Base__id': 42, '_Child__id': 99}
```

This is why Python uses name mangling: a subclass that accidentally defines `__id` will not silently shadow the parent's `__id`.

### The `property` Decorator: Pythonic Encapsulation

`property` is the canonical Python mechanism for controlled attribute access. It turns method calls into attribute-style access, letting you add validation or computation without changing the external API.

```python
class Temperature:
    def __init__(self, celsius: float = 0.0) -> None:
        self._celsius = celsius   # internal storage

    @property
    def celsius(self) -> float:
        return self._celsius

    @celsius.setter
    def celsius(self, value: float) -> None:
        if value < -273.15:
            raise ValueError(f"Temperature {value}°C below absolute zero")
        self._celsius = value

    @celsius.deleter
    def celsius(self) -> None:
        self._celsius = 0.0

    @property
    def fahrenheit(self) -> float:
        return self._celsius * 9 / 5 + 32

    @fahrenheit.setter
    def fahrenheit(self, value: float) -> None:
        self.celsius = (value - 32) * 5 / 9   # reuses celsius setter validation

t = Temperature(25.0)
print(t.celsius)     # 25.0
print(t.fahrenheit)  # 77.0
t.fahrenheit = 32.0
print(t.celsius)     # 0.0

try:
    t.celsius = -300.0
except ValueError as e:
    print(e)  # "Temperature -300.0°C below absolute zero"
```

**How `property` works under the hood:** `property` is itself a **data descriptor** (it defines `__get__`, `__set__`, and `__delete__`). Because data descriptors take priority over `instance.__dict__`, they always intercept access — you cannot bypass a property by direct dict manipulation like you can with a plain instance attribute.

```python
# property is a descriptor in the class dict:
print(type(Temperature.__dict__['celsius']))   # <class 'property'>
print(Temperature.__dict__['celsius'].fget)    # <function Temperature.celsius at ...>
```

### __slots__ for Attribute Restriction

`__slots__` prevents adding arbitrary attributes to instances:

```python
class StrictPoint:
    __slots__ = ("x", "y")

    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y

p = StrictPoint(1.0, 2.0)
try:
    p.z = 3.0   # AttributeError: 'StrictPoint' object has no attribute 'z'
except AttributeError as e:
    print(e)
```

This is a mild form of encapsulation — the class's *structure* is fixed.

### Python vs Java/C++ Encapsulation

| Aspect | Java/C++ | Python |
|---|---|---|
| `private` keyword | Hard enforced at compile/run time | No equivalent (`__` is name mangling only) |
| `protected` keyword | Compile-time restricted to subclasses | `_` is a convention, not enforced |
| `public` keyword | Explicit | Default (no leading underscore) |
| Runtime enforcement | Yes (JVM / OS protection) | No (everything is accessible with the right name) |
| Philosophy | Enforce access by language | Trust developer convention |
| Getter/setter idiom | Explicit `getName()`/`setName()` | `@property` — looks like a plain attribute |

### Practical Encapsulation Patterns

**Pattern 1 — Start public, add property later without breaking callers:**

```python
# v1 — simple attribute
class Circle:
    def __init__(self, radius: float) -> None:
        self.radius = radius

# v2 — add validation; callers still use circle.radius = x
class Circle:
    def __init__(self, radius: float) -> None:
        self.radius = radius   # this calls the setter!

    @property
    def radius(self) -> float:
        return self._radius

    @radius.setter
    def radius(self, value: float) -> None:
        if value < 0:
            raise ValueError("Radius must be non-negative")
        self._radius = value
```

The public API (`circle.radius = x`) didn't change. Internal behaviour did. This is the key encapsulation win.

**Pattern 2 — Read-only property:**

```python
class ImmutablePoint:
    def __init__(self, x: float, y: float) -> None:
        self._x = x
        self._y = y

    @property
    def x(self) -> float:
        return self._x

    @property
    def y(self) -> float:
        return self._y
    # No setters → x and y are read-only
```

**Pattern 3 — Computed property (no storage):**

```python
class Rectangle:
    def __init__(self, width: float, height: float) -> None:
        self.width  = width
        self.height = height

    @property
    def area(self) -> float:
        return self.width * self.height   # computed on demand, no storage

    @property
    def perimeter(self) -> float:
        return 2 * (self.width + self.height)
```

---

## 2. Conceptual Interview Questions (10)

**Q1: What is encapsulation and why is it important in Python?**

**A:** Encapsulation bundles data and behaviour into a class unit and restricts direct access to internal state. In Python, it is important because:
1. It allows the internal representation to change without breaking external callers (e.g., switching from a plain attribute to a `property` with validation).
2. It communicates intent — `_protected` tells subclass authors what is internal, `__mangled` says "don't touch this in subclasses".
3. It enforces invariants — `@property` setters can reject invalid values.

Unlike Java, Python's encapsulation is convention-based. The community relies on documentation and naming conventions rather than compiler enforcement.

---

**Q2: What exactly does name mangling do? Is __ truly private?**

**A:** Name mangling is a **compile-time syntactic transformation**: any identifier of the form `__name` (two or more leading underscores, at most one trailing underscore) inside a class body is rewritten to `_ClassName__name`. CPython performs this during the compilation of the class body to bytecode.

It is **not** true privacy. The transformed name is fully accessible:

```python
class Secret:
    def __init__(self):
        self.__x = 42

s = Secret()
print(s._Secret__x)   # 42 — completely accessible
print(vars(s))         # {'_Secret__x': 42}
```

The purpose is to avoid accidental name collisions in subclasses, not to protect data from determined access.

---

**Q3: What is the purpose of a single underscore prefix?**

**A:** A single leading underscore (`_name`) is a **convention** that signals "this is intended for internal use." It has no runtime enforcement whatsoever. It tells:
- Users of the class: "don't rely on this; it may change".
- Subclass authors: "you may access this within the hierarchy, but be careful".

It also affects `from module import *`: names with a leading underscore are excluded from `*` imports unless explicitly listed in `__all__`.

---

**Q4: How does `property` implement encapsulation in Python?**

**A:** `property` is a **data descriptor** (defines `__get__`, `__set__`, `__delete__`). Because data descriptors take precedence over instance `__dict__` entries, a `property` defined in the class always intercepts attribute access — clients cannot bypass it.

This enables:
1. **Validation** in the setter.
2. **Computed attributes** in the getter.
3. **Read-only attributes** (getter only, no setter).
4. **Backwards-compatible API changes** — convert a plain attribute to a property without changing the calling code.

---

**Q5: Why might you prefer a property over a getX()/setX() pattern?**

**A:** Python's property allows the clean, attribute-style syntax `obj.x = value` while still running validation or computation. This follows the Zen of Python: "Beautiful is better than ugly." Java-style `getX()`/`setX()` is idiomatic in Java but looks ceremonial and verbose in Python.

Properties also integrate naturally with Python's ecosystem: `dataclass`, `attrs`, `pydantic`, `mypy`, and IDEs all understand the property protocol.

```python
# Java style — explicit but verbose
class JavaStyle:
    def getRadius(self): return self._r
    def setRadius(self, v): self._r = v

# Pythonic
class Pythonic:
    @property
    def radius(self): return self._r
    @radius.setter
    def radius(self, v): self._r = v

# Usage:
# JavaStyle: c.setRadius(5)  vs Pythonic: c.radius = 5
```

---

**Q6: Can name-mangled attributes be accessed from outside the class?**

**A:** Yes — Python does not prevent it. The mangled name `_ClassName__attr` is simply a regular string key in `instance.__dict__`. Knowing the class name gives you the mangled name. This is intentional: Python's philosophy is that determined programmers can always find a way in. The mangling serves as a *speed bump* for accidental access, not a wall.

```python
class Vault:
    def __init__(self): self.__pin = 1234

v = Vault()
print(v._Vault__pin)   # 1234 — no error, just impolite
```

---

**Q7: What happens if a subclass defines an attribute with the same double-underscore name as the parent?**

**A:** Name mangling creates *separate* attributes for each class because the class name is incorporated:

```python
class Parent:
    def __init__(self):
        self.__value = "parent"   # stored as _Parent__value

class Child(Parent):
    def __init__(self):
        super().__init__()
        self.__value = "child"    # stored as _Child__value — different!

c = Child()
print(vars(c))   # {'_Parent__value': 'parent', '_Child__value': 'child'}
```

This is the *intended* use of name mangling: preventing subclasses from accidentally overriding internal attributes of a parent class.

---

**Q8: How do you create a truly immutable object in Python?**

**A:** Full immutability requires multiple mechanisms working together:

```python
class ImmutablePoint:
    __slots__ = ("_x", "_y")

    def __init__(self, x: float, y: float) -> None:
        object.__setattr__(self, "_x", x)   # bypass any __setattr__
        object.__setattr__(self, "_y", y)

    @property
    def x(self) -> float: return self._x

    @property
    def y(self) -> float: return self._y

    def __setattr__(self, name: str, value: object) -> None:
        raise AttributeError("ImmutablePoint is immutable")

    def __delattr__(self, name: str) -> None:
        raise AttributeError("ImmutablePoint is immutable")

p = ImmutablePoint(1.0, 2.0)
print(p.x, p.y)   # 1.0 2.0
try:
    p.x = 3.0     # AttributeError
except AttributeError as e:
    print(e)
```

Alternatively, use `@dataclass(frozen=True)` which generates `__setattr__` and `__delattr__` that raise `FrozenInstanceError`.

---

**Q9: What is the difference between `_attr`, `__attr`, and `attr` in terms of accessibility and convention?**

**A:**

| Name | Convention | Runtime enforcement | Use case |
|---|---|---|---|
| `attr` | Public API | None | Intended for external use |
| `_attr` | Internal/protected | None (only `*` import exclusion) | Internal use, OK in subclasses |
| `__attr` | Strongly internal | Name mangling (syntactic) | Avoid subclass name collisions |
| `__attr__` | Dunder/magic | None | Python protocol methods |

---

**Q10: When should you NOT use name mangling (double underscore)?**

**A:** Avoid `__name` when:
1. The attribute is part of a class hierarchy and subclasses legitimately need to access or override it — use `_name` instead.
2. You're writing a mixin or utility class that interoperates with unknown subclasses — mangling makes cooperation harder.
3. You're in a simple, small class where the protection ceremony adds noise.

Use `__name` when:
- You're writing a base class and want to guarantee that a particular internal attribute cannot be accidentally shadowed by a subclass.
- Frameworks like tkinter use `__name` in widget internals precisely for this reason.

---

## 3. Scenario-Based Interview Questions (10)

**Scenario 1: A junior developer directly modifies `account._balance` in test code. Your code review catches it. What do you say?**

**Q:** How do you explain the problem and what alternatives do you suggest?

**A:** Direct manipulation of `_balance` in tests bypasses the class's invariants (e.g., balance >= 0), making tests brittle and encouraging bad patterns. Instead, use the public API or provide a test-friendly factory:

```python
class BankAccount:
    def __init__(self, initial: float = 0.0) -> None:
        if initial < 0:
            raise ValueError("Initial balance cannot be negative")
        self._balance = initial

    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Deposit must be positive")
        self._balance += amount

    @property
    def balance(self) -> float:
        return self._balance

# In tests: use the public interface
acc = BankAccount(500.0)
acc.deposit(100.0)
assert acc.balance == 600.0   # ✓ uses public API
```

If you need to pre-populate state for testing, provide a `@classmethod` factory.

---

**Scenario 2: You are refactoring a class. The old code used `obj.radius` as a plain attribute. You need to add validation. How do you do this without breaking existing callers?**

**Q:** Show the refactoring.

**A:** Introduce a `@property` with the same public name. Callers continue to use `obj.radius = x` unmodified.

```python
# Before
class Circle:
    def __init__(self, radius: float) -> None:
        self.radius = radius

# After — same public API, added validation
class Circle:
    def __init__(self, radius: float) -> None:
        self.radius = radius   # calls setter

    @property
    def radius(self) -> float:
        return self._radius

    @radius.setter
    def radius(self, value: float) -> None:
        if value < 0:
            raise ValueError(f"radius must be >= 0, got {value}")
        self._radius = value

c = Circle(5.0)   # existing code unchanged
c.radius = 3.0    # existing code unchanged, now validated
```

---

**Scenario 3: A teammate says "I'll use __pin to make the PIN truly secure in our BankCard class." Is this correct?**

**Q:** Evaluate and explain.

**A:** Name mangling is NOT a security mechanism. The PIN stored as `self.__pin` becomes `self._BankCard__pin`, which is trivially accessible:

```python
class BankCard:
    def __init__(self, pin: int) -> None:
        self.__pin = pin

card = BankCard(1234)
print(card._BankCard__pin)  # 1234 — no protection

import json
print(json.dumps(vars(card)))  # {"_BankCard__pin": 1234}
```

For actual security, never store PINs in plain Python objects in memory without encryption. Use dedicated secrets management, hashing (`bcrypt`/`argon2`), and limit access at the architecture level. Name mangling is purely a namespacing tool.

---

**Scenario 4: You need a class attribute that can be read by subclasses but cannot be set or deleted from outside. Implement it.**

**Q:** Design the read-only class attribute.

**A:** Use a `property` on the class (via a custom descriptor or metaclass). A simpler approach for instance-level read-only:

```python
class Config:
    _DEBUG: bool = False

    @classmethod
    def debug(cls) -> bool:
        return cls._DEBUG

# Or using a property in a metaclass for class-level read-only:
class ReadOnlyMeta(type):
    @property
    def debug(cls) -> bool:
        return cls._DEBUG

class Config(metaclass=ReadOnlyMeta):
    _DEBUG: bool = False

print(Config.debug)   # False
try:
    Config.debug = True   # AttributeError — no setter
except AttributeError as e:
    print(e)
```

---

**Scenario 5: You're building a library and want users to know which methods are public API vs internal. What conventions do you apply?**

**Q:** Describe your approach.

**A:** 

```python
# public_module.py

__all__ = ["PublicClass"]   # restricts 'from module import *'

class PublicClass:
    """Public API — stable, versioned."""

    def public_method(self) -> str:           # public: part of API contract
        return self._internal_helper()

    def _internal_helper(self) -> str:        # protected: may change, stable within hierarchy
        return self.__core_logic()

    def __core_logic(self) -> str:            # name-mangled: not for subclasses
        return "result"

class _InternalClass:                          # _ prefix on class: not for external use
    pass
```

Supplement with docstrings marking `.. deprecated::` or `.. versionadded::`, and use `warnings.warn` with `DeprecationWarning` when public methods are deprecated.

---

**Scenario 6: You have a legacy class that stores temperature as Fahrenheit internally but you need to expose a Celsius API. Implement with properties.**

**Q:** Show the implementation.

**A:**

```python
class LegacyThermometer:
    def __init__(self, fahrenheit: float) -> None:
        self._fahrenheit = fahrenheit   # internal storage, legacy format

    @property
    def fahrenheit(self) -> float:
        return self._fahrenheit

    @fahrenheit.setter
    def fahrenheit(self, value: float) -> None:
        if value < -459.67:
            raise ValueError("Below absolute zero")
        self._fahrenheit = value

    @property
    def celsius(self) -> float:
        return (self._fahrenheit - 32) * 5 / 9

    @celsius.setter
    def celsius(self, value: float) -> None:
        self.fahrenheit = value * 9 / 5 + 32   # reuses fahrenheit validation

    @property
    def kelvin(self) -> float:
        return self.celsius + 273.15

t = LegacyThermometer(212.0)
print(t.celsius)     # 100.0
t.celsius = 0.0
print(t.fahrenheit)  # 32.0
print(t.kelvin)      # 273.15
```

---

**Scenario 7: A class has a property getter but no setter. What happens when a user tries to set it?**

**Q:** What is the error and how do you customise the message?

**A:**

```python
class ReadOnlyDemo:
    def __init__(self, value: int) -> None:
        self._value = value

    @property
    def value(self) -> int:
        return self._value
    # No setter defined

obj = ReadOnlyDemo(42)
print(obj.value)   # 42

try:
    obj.value = 100
except AttributeError as e:
    print(e)   # "property 'value' of 'ReadOnlyDemo' object has no setter"

# Customised message via explicit setter:
class BetterReadOnly:
    @property
    def value(self) -> int:
        return self._value

    @value.setter
    def value(self, _: object) -> None:
        raise AttributeError("'value' is read-only; create a new instance instead")
```

---

**Scenario 8: You want to log every time an attribute is accessed or set on a class. How do you implement this without modifying every attribute?**

**Q:** Use a custom `__getattribute__` and `__setattr__`.

**A:**

```python
import logging

logger = logging.getLogger(__name__)

class Audited:
    def __getattribute__(self, name: str) -> object:
        value = super().__getattribute__(name)
        if not name.startswith("_"):
            logger.debug("GET %s.%s = %r", type(self).__name__, name, value)
        return value

    def __setattr__(self, name: str, value: object) -> None:
        if not name.startswith("_"):
            logger.debug("SET %s.%s = %r", type(self).__name__, name, value)
        super().__setattr__(name, value)

logging.basicConfig(level=logging.DEBUG)

class Order(Audited):
    def __init__(self, total: float) -> None:
        self.total = total

o = Order(99.99)
_ = o.total
# DEBUG: SET Order.total = 99.99
# DEBUG: GET Order.total = 99.99
```

---

**Scenario 9: How would you implement a password field that hashes on assignment and never reveals the plain-text value?**

**Q:** Show a safe property implementation.

**A:**

```python
import hashlib
import secrets


class User:
    def __init__(self, username: str) -> None:
        self.username    = username
        self._pw_hash: str | None = None

    @property
    def password(self) -> None:
        raise AttributeError("Password is write-only")

    @password.setter
    def password(self, plain: str) -> None:
        salt = secrets.token_hex(16)
        self._pw_hash = salt + ":" + hashlib.sha256(
            (salt + plain).encode()
        ).hexdigest()

    def verify_password(self, plain: str) -> bool:
        if self._pw_hash is None:
            return False
        salt, digest = self._pw_hash.split(":", 1)
        return hashlib.sha256((salt + plain).encode()).hexdigest() == digest

u = User("alice")
u.password = "s3cret"
print(u.verify_password("s3cret"))  # True
print(u.verify_password("wrong"))   # False
try:
    print(u.password)               # AttributeError: Password is write-only
except AttributeError as e:
    print(e)
```

---

**Scenario 10: A colleague wants to use name mangling to prevent monkeypatching in production. Is this a valid use case?**

**Q:** Evaluate the approach.

**A:** No — name mangling does not prevent monkeypatching. Any determined code can still patch the mangled name:

```python
class Locked:
    def __init__(self): self.__secret = 42
    def get(self): return self.__secret

import unittest.mock as mock

obj = Locked()
with mock.patch.object(obj, "_Locked__secret", 99):
    print(obj.get())  # 99 — successfully monkeypatched

```

For production protection against tampering, the solutions are architectural (process isolation, read-only file systems, signing/verification). Name mangling is purely a *namespacing* tool.

---

## 4. Quick Recap / Cheatsheet

- **Encapsulation** = bundling data + behaviour in a class + controlling access to internal state.
- **Python philosophy**: "We're all consenting adults" — convention over enforcement.
- **`attr`** (no prefix) = public API. Stable, documented.
- **`_attr`** (single underscore) = internal/protected by convention. Accessible but marked "don't rely on this."
- **`__attr`** (double underscore) = name mangling: `_ClassName__attr`. NOT truly private — prevents accidental subclass shadowing.
- **Name mangling** is a compile-time syntactic transformation, not a runtime access control.
- **`property`** = the Pythonic getter/setter mechanism. Allows validation, computed values, read-only attributes, and backwards-compatible API changes.
- **Data descriptor** (property) takes precedence over instance `__dict__` — cannot be bypassed by direct dict manipulation.
- **`__slots__`** = prevents arbitrary attribute creation, mild structural encapsulation.
- **Immutability**: `@dataclass(frozen=True)` or custom `__setattr__` + `__delattr__` raising `AttributeError`.
- **Refactoring tip**: converting `self.x` to `@property x` is non-breaking if you keep the same name.
- **Security note**: name mangling and properties are NOT security features. Use architecture-level controls for security.
- **`__all__`** in modules restricts `*` imports — complementary encapsulation at the module level.
