# 📘 Topic 01 — Object-Oriented Programming (OOP) in Python

OOP is the backbone of professional Python development. Every major framework (Django, FastAPI, SQLAlchemy, Pydantic) is built on OOP principles. Mastering OOP means understanding not just *how* to write classes, but *why* they exist and *when* to use each feature.

---

## 📂 Sub-Topics

| File | Concept |
|------|---------|
| [classes-and-objects.md](classes-and-objects.md) | What classes and objects are at the CPython level |
| [init-constructors.md](init-constructors.md) | `__init__`, `__new__`, object lifecycle |
| [instance-class-static-methods.md](instance-class-static-methods.md) | Method types and when to choose each |
| [instance-class-static-variables.md](instance-class-static-variables.md) | Variable scoping inside classes |
| [encapsulation.md](encapsulation.md) | Name mangling, private/protected conventions |
| [abstraction.md](abstraction.md) | ABCs, `abc` module, abstract methods |
| [inheritance.md](inheritance.md) | Single, multiple, multi-level inheritance |
| [mro-and-super.md](mro-and-super.md) | C3 linearization, `super()` internals |
| [polymorphism.md](polymorphism.md) | Duck typing, method overriding, protocols |
| [method-overloading.md](method-overloading.md) | Python's approach to overloading (spoiler: no native overloading) |
| [composition-vs-inheritance.md](composition-vs-inheritance.md) | "Favor composition" — when and why |
| [property-getters-setters.md](property-getters-setters.md) | `@property`, `fset`, `fdel` |
| [classmethod-staticmethod.md](classmethod-staticmethod.md) | Deep dive into `@classmethod` and `@staticmethod` |
| [dunder-methods-overview.md](dunder-methods-overview.md) | Overview of Python's special methods |
| [slots.md](slots.md) | `__slots__` — memory optimization and tradeoffs |
| [metaclasses-intro.md](metaclasses-intro.md) | What metaclasses are, `type`, custom metaclasses |
| [dataclasses-interplay.md](dataclasses-interplay.md) | How `@dataclass` interacts with OOP concepts |
| [solid-in-python.md](solid-in-python.md) | SOLID principles with Python examples |

---

## 🏋️ Coding Challenges

See [coding-challenges.md](coding-challenges.md) for 15 scenario-rich OOP coding challenges.

---

## 🔑 Key Themes in This Topic

- Python uses **duck typing** — if it walks like a duck and quacks like a duck, it's a duck.
- Python's **MRO (Method Resolution Order)** uses C3 linearization — you must be able to trace it manually.
- `__slots__` is a memory optimization that removes `__dict__` from instances.
- **Composition is usually better than inheritance** — prefer "has-a" over "is-a" when in doubt.
- **Metaclasses** are "classes of classes" — rarely needed but important to understand for framework interviews.
- SOLID principles are language-agnostic but Python has idiomatic ways to implement each.
