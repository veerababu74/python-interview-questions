# 🐍 Python Interview Prep — Complete Knowledge Base

> **Master Python from fundamentals to advanced internals. Crack technical interviews with deep concepts, 10+10 Q&A per sub-topic, and 15 coding challenges per topic.**

---

## 🎯 Motivation

This repository exists because Python interviews are hard — not because Python is hard, but because interviewers probe deeply: *Why does `__slots__` exist? What happens in CPython when you call `super()`? How does the GIL affect `asyncio`?* Surface-level answers fail. This repo gives you the depth to excel.

Every sub-topic contains:
- **Deep conceptual explanation** with CPython internals, pitfalls, and best practices
- **10 conceptual interview Q&A** (multi-paragraph answers, code examples)
- **10 scenario-based interview Q&A** (realistic engineering situations)
- **Quick recap / cheatsheet**

Every topic contains a `coding-challenges.md` with **15 scenario-rich coding challenges** (no solutions — solve them yourself to build real skill).

---

## 🗂️ How to Use This Repo

1. **Pick your weak area** from the table of contents below.
2. **Read the sub-topic files** in order — they build on each other.
3. **Answer the interview questions** out loud or in writing before reading the answers.
4. **Solve the coding challenges** without looking at hints first.
5. **Review the cheatsheet** the day before your interview.

Suggested daily practice: 1 topic folder per day, 2–3 sub-topics deep.

---

## 📚 Table of Contents

| # | Topic | Status | Folder |
|---|-------|--------|--------|
| 01 | Object-Oriented Programming (OOP) | ✅ Full | [topics/01-oop/](topics/01-oop/) |
| 02 | Data Structures & Algorithms (DSA) | ✅ Full | [topics/02-dsa/](topics/02-dsa/) |
| 03 | File Handling (text, JSON, CSV) | 🔧 Skeleton | [topics/03-file-handling/](topics/03-file-handling/) |
| 04 | Design Principles (SOLID, DRY, etc.) | 🔧 Skeleton | [topics/04-design-principles/](topics/04-design-principles/) |
| 05 | Exception Handling | ✅ Full | [topics/05-exception-handling/](topics/05-exception-handling/) |
| 06 | Context Managers | ✅ Full | [topics/06-context-managers/](topics/06-context-managers/) |
| 07 | Threading | ✅ Full | [topics/07-threading/](topics/07-threading/) |
| 08 | Multiprocessing | 🔧 Skeleton | [topics/08-multiprocessing/](topics/08-multiprocessing/) |
| 09 | Async / Asyncio | ✅ Full | [topics/09-async/](topics/09-async/) |
| 10 | Dictionaries (all concepts) | 🔧 Skeleton | [topics/10-dictionaries/](topics/10-dictionaries/) |
| 11 | Lists (all concepts) | 🔧 Skeleton | [topics/11-lists/](topics/11-lists/) |
| 12 | Sets (all concepts) | 🔧 Skeleton | [topics/12-sets/](topics/12-sets/) |
| 13 | Comprehensions | 🔧 Skeleton | [topics/13-comprehensions/](topics/13-comprehensions/) |
| 14 | Dataclasses | 🔧 Skeleton | [topics/14-dataclasses/](topics/14-dataclasses/) |
| 15 | Enums | 🔧 Skeleton | [topics/15-enums/](topics/15-enums/) |
| 16 | *args, **kwargs | 🔧 Skeleton | [topics/16-args-kwargs/](topics/16-args-kwargs/) |
| 17 | map, filter, reduce | 🔧 Skeleton | [topics/17-map-filter-reduce/](topics/17-map-filter-reduce/) |
| 18 | Closures & Lambdas | 🔧 Skeleton | [topics/18-closures-lambdas/](topics/18-closures-lambdas/) |
| 19 | Generators & Iterators | ✅ Full | [topics/19-generators-iterators/](topics/19-generators-iterators/) |
| 20 | Memory Management | 🔧 Skeleton | [topics/20-memory-management/](topics/20-memory-management/) |
| 21 | String Manipulation | 🔧 Skeleton | [topics/21-string-manipulation/](topics/21-string-manipulation/) |
| 22 | OS Concepts | 🔧 Skeleton | [topics/22-os-concepts/](topics/22-os-concepts/) |
| 23 | Pydantic | ✅ Full | [topics/23-pydantic/](topics/23-pydantic/) |
| 24 | Magic Methods (Dunder) | ✅ Full | [topics/24-magic-methods/](topics/24-magic-methods/) |
| 25 | Decorators | ✅ Full | [topics/25-decorators/](topics/25-decorators/) |
| 26 | Typing & Type Hints | 🔧 Skeleton | [topics/26-typing-hints/](topics/26-typing-hints/) |
| 27 | functools | 🔧 Skeleton | [topics/27-functools/](topics/27-functools/) |
| 28 | itertools | 🔧 Skeleton | [topics/28-itertools/](topics/28-itertools/) |
| 29 | collections module | 🔧 Skeleton | [topics/29-collections/](topics/29-collections/) |
| 30 | Logging | 🔧 Skeleton | [topics/30-logging/](topics/30-logging/) |
| 31 | Testing (unittest, pytest) | 🔧 Skeleton | [topics/31-testing/](topics/31-testing/) |
| 32 | Packaging & Modules | 🔧 Skeleton | [topics/32-packaging/](topics/32-packaging/) |
| 33 | GIL & Execution Model | 🔧 Skeleton | [topics/33-gil-execution-model/](topics/33-gil-execution-model/) |
| 34 | Descriptors & Metaclasses | 🔧 Skeleton | [topics/34-descriptors-metaclasses/](topics/34-descriptors-metaclasses/) |
| 35 | Scope & Namespaces (LEGB) | 🔧 Skeleton | [topics/35-scope-namespaces/](topics/35-scope-namespaces/) |
| 36 | Mutable vs Immutable & Deep Copy | 🔧 Skeleton | [topics/36-mutable-immutable/](topics/36-mutable-immutable/) |
| 37 | Database Basics (SQLAlchemy, raw SQL) | 🔧 Skeleton | [topics/37-database-basics/](topics/37-database-basics/) |
| 38 | REST API Basics (FastAPI/Flask) | 🔧 Skeleton | [topics/38-rest-api-basics/](topics/38-rest-api-basics/) |
| 39 | Design Patterns in Python | 🔧 Skeleton | [topics/39-design-patterns/](topics/39-design-patterns/) |
| 40 | Security Basics | 🔧 Skeleton | [topics/40-security-basics/](topics/40-security-basics/) |

---

## 🗺️ Suggested Study Order

### Beginner → Intermediate
1. 35 — Scope & Namespaces
2. 36 — Mutable vs Immutable
3. 10 — Dictionaries
4. 11 — Lists
5. 12 — Sets
6. 13 — Comprehensions
7. 16 — *args / **kwargs
8. 18 — Closures & Lambdas
9. 21 — String Manipulation
10. 17 — map / filter / reduce

### Intermediate → Advanced
11. 01 — OOP (core)
12. 05 — Exception Handling
13. 06 — Context Managers
14. 19 — Generators & Iterators
15. 25 — Decorators
16. 24 — Magic Methods
17. 14 — Dataclasses
18. 15 — Enums
19. 26 — Typing & Type Hints
20. 23 — Pydantic

### Advanced / Concurrency
21. 33 — GIL & Execution Model
22. 07 — Threading
23. 08 — Multiprocessing
24. 09 — Async / Asyncio
25. 20 — Memory Management
26. 34 — Descriptors & Metaclasses

### CS Fundamentals & Interview Patterns
27. 02 — DSA
28. 27 — functools
29. 28 — itertools
30. 29 — collections

### Professional Skills
31. 04 — Design Principles
32. 39 — Design Patterns
33. 03 — File Handling
34. 22 — OS Concepts
35. 30 — Logging
36. 31 — Testing
37. 32 — Packaging
38. 37 — Database Basics
39. 38 — REST API Basics
40. 40 — Security Basics

---

## 📊 Status

| Status | Meaning |
|--------|---------|
| ✅ Full | All sub-topic files present, deep content (200+ lines each), complete Q&A |
| 🔧 Skeleton | Folder + README + coding-challenges.md only; sub-topic deep-dives pending |

### Fully Populated Topics
- ✅ 01 — OOP
- ✅ 02 — DSA
- ✅ 05 — Exception Handling
- ✅ 06 — Context Managers
- ✅ 07 — Threading
- ✅ 09 — Async / Asyncio
- ✅ 19 — Generators & Iterators
- ✅ 23 — Pydantic
- ✅ 24 — Magic Methods
- ✅ 25 — Decorators

### Skeleton Topics (content pending)
All remaining topics (03, 04, 08, 10–18, 20–22, 26–40) have:
- `README.md` listing planned sub-topics
- `coding-challenges.md` with 15 real challenges

See individual topic READMEs for planned sub-topic lists and follow-up issue links.

---

## 🤝 Contributing

Found an error? Want to add a sub-topic deep-dive? Open a PR! Please follow the per-sub-topic template in the `CONTRIBUTING.md` (coming soon).

---

*Built to help Python developers crack technical interviews with confidence. Study hard, practice daily, and go get that job! 🚀*
