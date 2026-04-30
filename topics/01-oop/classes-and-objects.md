# Classes and Objects

## 1. Concept (Deep Explanation)

### What It Is and Why It Exists

A **class** is a blueprint for creating objects — it defines the structure (attributes) and behaviour (methods) shared by all instances. An **object** (instance) is a concrete realisation of that blueprint, living in heap memory with its own namespace.

Python's object model is deeply uniform: *everything* is an object, including classes themselves. A class is an instance of its metaclass (`type` by default). This uniformity makes Python's runtime highly introspectable and dynamic.

Classes exist to support:
- **Encapsulation** — bundling state and behaviour together
- **Reuse** — share behaviour through inheritance and composition
- **Abstraction** — hide implementation details behind a clean interface
- **Polymorphism** — different types responding to the same interface

### When to Use Classes
- When you need to maintain state across method calls
- When you have a coherent set of behaviours that logically belong together
- When you want to model real-world entities (User, BankAccount, Request)
- When you need inheritance hierarchies

### When NOT to Use Classes
- Simple utility functions with no shared state → use module-level functions
- Data containers with no behaviour → use `dataclasses`, `NamedTuple`, or plain dicts
- Single-method "classes" → often better as a closure or function
- Forced OOP just to be "object-oriented" is an anti-pattern

### CPython Internals

Every Python object is a C struct `PyObject` at minimum, containing:
- `ob_refcnt` — reference count for garbage collection
- `ob_type` — pointer to the type (class)

When you write `class Foo:`, CPython:
1. Executes the class body as a code block in a fresh namespace (a `dict`)
2. Calls `type.__new__(mcs, name, bases, namespace)` to create a new type object
3. Assigns the resulting type to the name `Foo` in the enclosing scope

Instance creation (`foo = Foo()`) invokes `Foo.__call__`, which calls:
1. `Foo.__new__(Foo)` → allocates memory, returns raw instance
2. `Foo.__init__(instance)` → initialises the instance

The instance's attributes live in `instance.__dict__`, a plain Python dict (unless `__slots__` is used).

```python
class Dog:
    species = "Canis lupus familiaris"  # class variable, in Dog.__dict__

    def __init__(self, name: str, age: int) -> None:
        self.name = name   # instance variable, in self.__dict__
        self.age = age

rex = Dog("Rex", 5)
print(rex.__dict__)        # {'name': 'Rex', 'age': 5}
print(Dog.__dict__.keys()) # dict_keys(['__module__', 'species', '__init__', ...])
print(type(Dog))           # <class 'type'>
print(isinstance(Dog, type))  # True  — Dog is itself an instance of type
```

### Attribute Lookup Order (MRO + Descriptor Protocol)

When Python evaluates `obj.attr`, it follows:
1. **Data descriptors** in the class (and bases) — objects with both `__get__` and `__set__`
2. **Instance `__dict__`**
3. **Non-data descriptors** and other class attributes

```python
class Circle:
    pi = 3.14159  # class variable

    def __init__(self, radius: float) -> None:
        self.radius = radius

    def area(self) -> float:
        return self.pi * self.radius ** 2

c = Circle(5)
print(c.area())    # 78.53975
# Lookup: c.pi → not in c.__dict__ → found in Circle.__dict__
```

### Identity, Equality and `is` vs `==`

```python
a = Dog("Buddy", 3)
b = Dog("Buddy", 3)
c = a

print(a is b)   # False — different objects in memory
print(a is c)   # True  — same object (c is just another reference)
print(id(a), id(b))  # different memory addresses

# Without __eq__, == falls back to identity check
print(a == b)   # False (default)
```

### Class Body Execution — A Subtle Detail

The class body executes *immediately* when the `class` statement is encountered:

```python
print("Before class")

class Verbose:
    print("Inside class body")   # runs at class definition time
    x = 10

print("After class")
# Output:
# Before class
# Inside class body
# After class
```

### Dynamic Class Creation

You can create classes at runtime using `type()`:

```python
# type(name, bases, namespace)
Animal = type("Animal", (object,), {
    "sound": "...",
    "speak": lambda self: print(self.sound)
})

Cat = type("Cat", (Animal,), {"sound": "meow"})
cat = Cat()
cat.speak()  # meow
```

### Pitfalls / Gotchas / Interview Traps

**Mutable default class variable shared across instances:**
```python
class BadList:
    items = []   # shared across ALL instances!

a = BadList()
b = BadList()
a.items.append(1)
print(b.items)   # [1] — surprise!

# Fix: use instance variable in __init__
class GoodList:
    def __init__(self):
        self.items = []
```

**Forgetting `self`:**
```python
class Counter:
    count = 0

    def increment():   # missing self — will fail at call time
        Counter.count += 1
```

**`__init__` vs `__new__`:** `__init__` does NOT create the object; `__new__` does.

**Class variables shadowed by instance variables:**
```python
class Foo:
    x = 10

f = Foo()
f.x = 99      # creates instance variable, shadows class variable
print(Foo.x)  # still 10
del f.x
print(f.x)    # 10 again — falls through to class variable
```

### Best Practices

- Use `ClassName` (PascalCase) for classes
- Keep classes small and focused (Single Responsibility)
- Prefer composition over deep inheritance hierarchies
- Use type hints in `__init__` signatures
- Document classes with docstrings

---

## 2. Conceptual Interview Questions (10)

**Q1: What is the difference between a class and an instance?**

**A:** A class is a template/blueprint — it exists once in memory as a type object and defines the structure and behaviour of its instances. An instance is a concrete object created from that class; each instance has its own `__dict__` namespace for its data. For example, `Dog` is the class; `rex = Dog("Rex", 5)` creates an instance. Multiple instances share the class's methods (stored once in the class's namespace) but have independent attribute storage.

```python
class Dog:
    def __init__(self, name):
        self.name = name

rex = Dog("Rex")
fido = Dog("Fido")
print(rex.__dict__)   # {'name': 'Rex'}
print(fido.__dict__)  # {'name': 'Fido'}
print(rex.bark is fido.bark)  # False — bound methods are created on access
```

---

**Q2: How does Python resolve attribute access on an instance?**

**A:** Python uses the **descriptor protocol** combined with the MRO. When `obj.attr` is evaluated:
1. Python looks for `attr` in `type(obj).__mro__` (the class hierarchy) as a **data descriptor** (has both `__get__` and `__set__`).
2. If found as a data descriptor, it takes priority.
3. Otherwise Python checks `obj.__dict__`.
4. If not found there, Python looks for a non-data descriptor or plain attribute in the class hierarchy.
5. If nothing is found, `AttributeError` is raised.

This ordering ensures that `property` (a data descriptor) overrides instance `__dict__` entries.

---

**Q3: What does `type(SomeClass)` return and why?**

**A:** It returns `<class 'type'>`. In Python, classes are themselves objects — instances of their metaclass. The default metaclass is `type`. So `Dog` is an instance of `type`, just like `rex` is an instance of `Dog`. This is the "everything is an object" principle. `type` is its own metaclass: `type(type)` is `type`.

```python
class Dog: pass
print(type(Dog))        # <class 'type'>
print(isinstance(Dog, type))  # True
print(type(type))       # <class 'type'>
```

---

**Q4: What is `__dict__` on a class vs on an instance?**

**A:** Both classes and instances have a `__dict__`, but they serve different purposes.
- `instance.__dict__` is a plain dict storing that instance's attributes only.
- `Class.__dict__` is a `mappingproxy` (read-only view of the class namespace) storing methods, class variables, and special attributes defined at the class level. It is read-only to prevent accidental mutation; you add/change class attributes via `setattr(Class, 'attr', value)` or direct assignment `Class.attr = value`.

```python
class Foo:
    x = 10
    def method(self): pass

f = Foo()
f.y = 20
print(type(Foo.__dict__))   # <class 'mappingproxy'>
print(f.__dict__)            # {'y': 20}
```

---

**Q5: What is the difference between `__new__` and `__init__`?**

**A:** `__new__` is a static method responsible for *creating* (allocating) the new instance. It receives the class as its first argument and must return an instance. `__init__` is an instance method responsible for *initialising* the already-created instance; it receives `self` and returns `None`. In normal usage, `__new__` creates, then `__init__` initialises. You override `__new__` when you need to control object creation itself — for example, implementing singletons, immutable types (subclassing `str`, `int`, `tuple`), or customising allocation.

```python
class Singleton:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        self.value = 42

a = Singleton()
b = Singleton()
print(a is b)  # True
```

---

**Q6: Can a class exist without `__init__`?**

**A:** Absolutely. If you don't define `__init__`, Python uses `object.__init__`, which accepts only `self` and does nothing. The instance is still created (via `__new__`), just with an empty `__dict__`. This is perfectly valid for classes that need no initialisation state or that rely entirely on class variables.

```python
class Empty:
    pass

e = Empty()
print(e.__dict__)   # {}
e.x = 5            # you can still add attributes dynamically
print(e.x)         # 5
```

---

**Q7: What is the difference between a class variable and an instance variable?**

**A:** A **class variable** is defined in the class body (outside any method) and is stored in the class's `__dict__`. It is shared among all instances. An **instance variable** is defined inside a method (typically `__init__`) via `self.name = value` and stored in the individual instance's `__dict__`. When an instance reads a class variable through `self.var`, it finds it in the class namespace. If it *writes* `self.var = new_value`, it creates a new entry in the instance's `__dict__`, effectively shadowing the class variable for that instance only.

---

**Q8: What does `isinstance()` do and how does it differ from `type() ==`?**

**A:** `isinstance(obj, cls)` returns `True` if `obj` is an instance of `cls` or any subclass of `cls`. `type(obj) == cls` is a strict identity check that returns `False` for subclasses. `isinstance()` is preferred in practice because it respects inheritance and works with abstract base classes.

```python
class Animal: pass
class Dog(Animal): pass

d = Dog()
print(isinstance(d, Animal))   # True
print(type(d) == Animal)       # False
print(type(d) == Dog)          # True
```

---

**Q9: How does Python handle method storage? Is a method stored per-instance?**

**A:** No. Methods (functions defined in a class body) are stored **once** in the class's `__dict__`. When accessed via an instance (`obj.method`), Python's descriptor protocol wraps the function in a **bound method** object on the fly — this is a lightweight wrapper that binds `self` to the function. That's why `obj.method is obj.method` is `False` (a new bound method is created each access), but `Dog.bark is Dog.bark` is `True` (just a plain function in the class dict).

```python
class Dog:
    def bark(self): pass

rex = Dog()
print(type(rex.bark))     # <class 'method'>
print(type(Dog.bark))     # <class 'function'>
print(rex.bark is rex.bark)  # False — new bound method each time
```

---

**Q10: What is `object` and why do all classes implicitly inherit from it?**

**A:** `object` is the root of Python's class hierarchy. Every class you define implicitly inherits from `object` (in Python 3; in Python 2, "old-style" classes did not). `object` provides default implementations of many dunder methods (`__repr__`, `__str__`, `__eq__`, `__hash__`, `__init__`, `__new__`, etc.). This guarantees that any class has a consistent baseline interface, enabling the descriptor protocol, `isinstance` checks, and much of the runtime machinery to work uniformly.

---

## 3. Scenario-Based Interview Questions (10)

**Q1: You're designing a caching layer. How do you implement a class that ensures only one instance exists throughout the program's lifetime?**

**A:** Use the Singleton pattern via `__new__`:

```python
class Cache:
    _instance: "Cache | None" = None
    _data: dict = {}

    def __new__(cls) -> "Cache":
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def set(self, key: str, value: object) -> None:
        self._data[key] = value

    def get(self, key: str) -> object:
        return self._data.get(key)

c1 = Cache()
c2 = Cache()
c1.set("user", "Alice")
print(c2.get("user"))  # Alice
print(c1 is c2)        # True
```

Discuss thread-safety: in a multi-threaded environment, add a `threading.Lock` around the `if` check.

---

**Q2: A junior developer wrote a class where all instances unexpectedly share a list. How do you diagnose and fix it?**

**A:** The bug is a mutable class variable used as a default container:

```python
# Buggy
class ShoppingCart:
    items = []           # shared across ALL instances

    def add(self, item):
        self.items.append(item)

cart1 = ShoppingCart()
cart2 = ShoppingCart()
cart1.add("apple")
print(cart2.items)  # ['apple'] — wrong!

# Fixed
class ShoppingCart:
    def __init__(self):
        self.items = []  # each instance gets its own list

    def add(self, item):
        self.items.append(item)
```

Diagnose by checking `ShoppingCart.__dict__` vs `instance.__dict__`.

---

**Q3: You need a class that tracks how many instances have been created. How do you implement this?**

**A:** Use a class variable incremented in `__init__`, decremented in `__del__`:

```python
class TrackedObject:
    _count: int = 0

    def __init__(self, name: str) -> None:
        TrackedObject._count += 1
        self.name = name

    def __del__(self) -> None:
        TrackedObject._count -= 1

    @classmethod
    def instance_count(cls) -> int:
        return cls._count

a = TrackedObject("a")
b = TrackedObject("b")
print(TrackedObject.instance_count())  # 2
del a
print(TrackedObject.instance_count())  # 1
```

Caveat: `__del__` is not guaranteed to run promptly in all implementations.

---

**Q4: How would you create a class that behaves like a named tuple but allows mutation?**

**A:** Use `__slots__` with a custom class or simply use `dataclasses`:

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

    def distance_to_origin(self) -> float:
        return (self.x**2 + self.y**2) ** 0.5

p = Point(3.0, 4.0)
print(p.distance_to_origin())  # 5.0
p.x = 6.0   # mutable
```

Or with `__slots__` for memory efficiency:
```python
class Point:
    __slots__ = ("x", "y")

    def __init__(self, x: float, y: float) -> None:
        self.x = x
        self.y = y
```

---

**Q5: You receive a class that you cannot modify but need to add a method to it. What are your options?**

**A:** Several approaches exist:
1. **Monkey-patch** (not recommended in production): `ClassName.new_method = my_function`
2. **Subclass** it and add the method
3. **Composition** — wrap the class

```python
class ThirdPartyClass:
    def greet(self):
        return "hello"

# Option 1: Subclass
class Extended(ThirdPartyClass):
    def goodbye(self):
        return "bye"

# Option 2: Monkey-patch (use sparingly, e.g., in tests)
ThirdPartyClass.goodbye = lambda self: "bye"
```

---

**Q6: Given `class B(A)`, what happens to `A.__init__` if `B` doesn't define `__init__`?**

**A:** Python's MRO will find `A.__init__` when `B()` is instantiated, so `A.__init__` is called with the new `B` instance as `self`. The instance will be of type `B`, but initialised by `A.__init__`. This is standard inheritance. If `B` does define `__init__` and doesn't call `super().__init__()`, `A`'s initialisation logic is skipped entirely.

```python
class A:
    def __init__(self):
        self.x = 10

class B(A):
    pass   # inherits A.__init__

class C(A):
    def __init__(self):
        # super().__init__() not called — x is never set
        self.y = 20

b = B()
print(b.x)   # 10
c = C()
print(hasattr(c, 'x'))  # False
```

---

**Q7: How would you implement a read-only class that prevents any attribute from being set after creation?**

**A:** Override `__setattr__` to raise `AttributeError` after construction:

```python
class Immutable:
    def __init__(self, **kwargs):
        for key, value in kwargs.items():
            object.__setattr__(self, key, value)   # bypass our override

    def __setattr__(self, name, value):
        raise AttributeError("This object is immutable")

    def __delattr__(self, name):
        raise AttributeError("This object is immutable")

p = Immutable(x=1, y=2)
print(p.x)   # 1
p.x = 5      # AttributeError
```

---

**Q8: You want to print a human-readable summary of any object's class and attributes. How would you write a generic `describe()` function?**

**A:** Use `__class__.__name__` and `vars()` (or `__dict__`):

```python
def describe(obj: object) -> str:
    class_name = type(obj).__name__
    attrs = ", ".join(f"{k}={v!r}" for k, v in vars(obj).items())
    return f"{class_name}({attrs})"

class User:
    def __init__(self, name: str, age: int) -> None:
        self.name = name
        self.age = age

print(describe(User("Alice", 30)))  # User(name='Alice', age=30)
```

`vars(obj)` returns `obj.__dict__`; note it won't work on objects using `__slots__`.

---

**Q9: A class has a method that modifies data and another that reads data. How do you ensure callers cannot accidentally call the mutating method in a read-only context?**

**A:** This is about access control and API design. Options include:
- Separate the class into a reader interface (ABC) and a mutable subclass
- Use naming conventions (`_mutate()`)
- Use `property` with only a getter

```python
from abc import ABC, abstractmethod

class ReadOnlyRepo(ABC):
    @abstractmethod
    def get(self, key: str) -> object: ...

class ReadWriteRepo(ReadOnlyRepo):
    def __init__(self): self._store = {}
    def get(self, key): return self._store.get(key)
    def set(self, key, val): self._store[key] = val

def read_only_consumer(repo: ReadOnlyRepo) -> None:
    # Type system prevents calling .set() here
    print(repo.get("key"))
```

---

**Q10: How do you introspect all methods and attributes of an unknown class at runtime?**

**A:** Use `dir()`, `inspect`, and `vars()`:

```python
import inspect

class Robot:
    kind = "AI"
    def __init__(self, name): self.name = name
    def greet(self): return f"Hi, I'm {self.name}"

r = Robot("R2D2")

# All attributes (including inherited)
print(dir(r))

# Only instance attributes
print(vars(r))

# Methods defined on the class
methods = inspect.getmembers(Robot, predicate=inspect.isfunction)
print(methods)  # [('__init__', ...), ('greet', ...)]

# Class hierarchy
print(Robot.__mro__)
```

---

## 4. Quick Recap / Cheatsheet

- **Class** = blueprint; **Instance** = concrete realisation with its own `__dict__`
- Everything in Python is an object — classes are instances of `type`
- `__new__` creates; `__init__` initialises
- Class variables live in `Class.__dict__`; instance variables live in `instance.__dict__`
- Writing `self.var = x` shadows a class variable without modifying it
- Mutable class variables are a classic shared-state bug
- `isinstance(obj, cls)` respects inheritance; `type(obj) == cls` does not
- Methods are stored once in the class; bound methods are created on-demand
- `object` is the implicit base of every Python 3 class
- Attribute lookup order: data descriptors → instance `__dict__` → non-data descriptors
- `vars(obj)` ≡ `obj.__dict__`; `dir(obj)` includes inherited names
- Use `type(name, bases, namespace)` to create classes dynamically
