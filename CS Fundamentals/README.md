# 💻 Computer Science Fundamentals

Welcome to the **Computer Science Fundamentals** module. This directory covers core software engineering paradigms, low-level mechanics, design principles, and programming models essential for building robust, scalable systems.

---

## 📖 Curriculum

1. **[Object-Oriented Programming (OOP)](01_Object_Oriented_Programming.md):** Core concepts, state and behavior, the four pillars (Encapsulation, Abstraction, Inheritance/Composition, Polymorphism), cross-language paradigm differences, and concrete implementations in **Java**, **TypeScript**, and **Go**.
2. **[SOLID Principles](02_SOLID_Principles.md):** Architectural design for resilient software. Deep dive into Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion with full real-world scenarios across **React**, **Next.js**, **Angular**, and **Backend Services**.
3. **[Memory Management & Garbage Collection](03_Memory_Management.md):** Stack vs. Heap architecture, compiler Escape Analysis, manual memory safety in **C++** (RAII and Smart Pointers), low-latency Concurrent Tri-Color Mark-and-Sweep in **Go** (`GOMEMLIMIT`), and Generational GC in **JavaScript / TypeScript (V8 Scavenger & Full Mark-Compact)**. Deep dives into real-world memory leaks in backend microservices (unbounded maps, closure listeners, blocked goroutines) and frontend web apps (detached DOM nodes, uncleaned timers), with profiling via Chrome DevTools and `pprof`.
4. **[Error Handling & Debugging](04_Error_Handling_And_Debugging.md):** Exceptions vs. Errors as Values, reading and dissecting complex **Stack Traces** (Node.js/TypeScript source maps vs. Go panic dumps), domain error hierarchies, centralized Express/Fastify error middleware, Go error wrapping (`%w`, `errors.Is`, `errors.As`), panic recovery middleware, interactive debugging workflows (Delve, Chrome Inspector), distributed tracing with Correlation IDs, anti-patterns, and senior engineering checklists.

---

⬅️ **[Back to System Design Root](../README.md)**
