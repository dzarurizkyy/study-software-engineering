# Study Software Engineering 🛠️

A comprehensive software engineering reference guide — covering career growth, design patterns, system design & architecture, and database design.

## List of Material 📚

- 🚀 **[Programmer's Career Guide](001-career-guide.md)**

  A practical career guide for aspiring and junior programmers — covering community, focus, mindset, job hunting, and salary negotiation.

  Depth beats breadth — pick one language and go all in:

  ```text
  Phase 1 — Explore (≈1 Month)
  ├── Try a few popular languages
  ├── Note which one excites you most
  └── Pick ONE based on genuine interest, not trends

  Phase 2 — Deep Dive (School Years)
  ├── Use all your free time on this one language
  ├── Go beyond syntax — understand internals
  ├── Build real projects, debug hard problems
  └── Become the person others ask for help

  Phase 3 — Career Ready
  └── You're already an expert before graduating
  ```

- 🏗️ **[Design Patterns Basics](002-design-pattern.md)**

  A comprehensive reference guide for design patterns, architectures, and best practices in software development.

  The Singleton pattern — one instance, shared everywhere:

  ```javascript
  class PrismaClient {
    constructor() {
      if (PrismaClient.instance) {
        return PrismaClient.instance;
      }
      this.connection = null;
      PrismaClient.instance = this;
    }

    async connect() {
      if (!this.connection) {
        this.connection = await db.connect();
      }
      return this.connection;
    }
  }

  export default new PrismaClient();
  ```

- 🏛️ **[System Design & Architecture Guide](003-system-design.md)**

  A comprehensive reference for backend engineers — covering authentication, system reliability, distributed tracing, and software architecture patterns.

  The Golden Rule of timeouts — always wait longer than the third party:

  ```text
  Your app timeout > Third-party server timeout

  Example:
    Bank processes in max 30s
    Your timeout: 35s

    → Your app always waits long enough for the bank to respond
    → Data stays consistent
  ```

- 🗄️ **[Database Design Case Studies](004-database-design.md)**

  Practical database design patterns for real-world features — covering multi-language support and notification systems.

  One translation table instead of a column per language:

  ```text
  categories                    categories_translation
  ┌─────────────────────┐       ┌──────────────────────────────────┐
  │ id (PK)             │──┐    │ category_id (FK, PK)             │
  │ position            │  └──► │ language    (PK)    ◄── composite │
  └─────────────────────┘       │ name                             │
                                │ description                      │
                                └──────────────────────────────────┘
  ```

## 📍 References

- [Programmer Zaman Now (YouTube)](https://www.youtube.com/@ProgrammerZamanNow)

## 👨‍💻 Contributors

- [Dzaru Rizky Fathan Fortuna](https://www.linkedin.com/in/dzarurizky)
