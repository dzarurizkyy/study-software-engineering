# 🏗️ Design Patterns Basics

A comprehensive reference guide for design patterns, architectures, and best practices in software development.

---

## 📋 Table of Contents

- [Introduction](#-introduction)
- [What are Design Patterns](#-what-are-design-patterns)
- [Creational Patterns](#-creational-patterns)
  - [Singleton](#-singleton)
  - [Builder](#-builder)
  - [Factory Method](#-factory-method)
  - [Abstract Factory](#-abstract-factory)
  - [Prototype](#-prototype)
  - [Object Pool](#-object-pool)
- [Structural Patterns](#-structural-patterns)
  - [Adapter](#-adapter)
  - [Facade](#-facade)
  - [Repository](#-repository)
- [Behavioral Patterns](#-behavioral-patterns)
  - [Template Method](#-template-method)
  - [Composite](#-composite)
- [Quick Reference Table](#-quick-reference-table)
- [Best Practices](#-best-practices)

---

## 🎯 Introduction

**Who Is This For?**

- **Web Developers** — Build maintainable, scalable applications
- **System Designers** — Design flexible, extensible architectures
- **Team Leads** — Establish code standards and patterns across teams
- **Anyone writing production code** — Master proven solutions to recurring problems

**What This Material Covers**

- Theory and practical implementation of 11 major design patterns
- Real-world code examples in JavaScript and TypeScript
- Backend and frontend pattern applications
- Industry best practices and common pitfalls

**Why Learn Design Patterns?**

- **Proven solutions** — Tested by experts over decades
- **Communication** — Shared vocabulary with other developers
- **Maintainability** — Code becomes easier to understand and modify
- **Scalability** — Patterns support growing complexity
- **Career advancement** — Expected knowledge for senior roles

---

## 🧩 What are Design Patterns?

> **Analogy:** Architecture patterns are like recipe templates in a cookbook. Instead of inventing a new way to bake bread every time, you use established recipes that work. The recipe is a proven solution to a cooking problem — how to make bread rise properly, how to balance flour and water, how to get the right crust. Similarly, design patterns are proven solutions to common programming problems.

**Definition**

- **Design patterns** are reusable templates for solving recurring architectural and structural problems
- They describe **best practices** distilled from collective programming experience
- Different from algorithms — patterns are about **structure and relationships**, not step-by-step computation

**Core Benefits**

- **Avoid reinventing the wheel** — Use solutions that are battle-tested
- **Cleaner code** — Code becomes self-documenting when others recognize the pattern
- **Easier maintenance** — Patterns make it obvious what problem you were solving
- **Faster debugging** — Knowing the pattern tells you where to look for issues
- **Scaling development** — Teams can coordinate better when speaking the same pattern language

**Common Misconception**

❌ Not every problem needs a pattern — patterns are tools, not mandates  
✅ Use patterns when they solve a genuine pain point in your code

---

## 🏭 Creational Patterns

Patterns that focus on **object creation mechanisms**, hiding complexity and promoting code reuse.

---

### 🔒 Singleton

**Problem**

> **Analogy:** Your application needs a database connection, configuration object, or logging service. Creating a new instance every request is wasteful — a database connection pool can't handle 10,000 new connections per second. But multiple instances might be out of sync. You need exactly one instance shared across the entire application.

**What is Singleton?**

- A class that can have **only one instance** in memory
- Provides a **global access point** to that instance
- Lazy initialization — the instance is created only when first needed

**Backend Example: Database Connection**

```javascript
// ✅ Correct: Singleton pattern

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

// Usage across multiple files
import prisma from "./lib/prisma.js";
const users = await prisma.user.findMany();
```

**Frontend Example: API Client**

```javascript
// ✅ Singleton axios instance

const apiClient = axios.create({
  baseURL: "http://localhost:3000/api",
  timeout: 5000,
});

// Interceptor — runs on EVERY request
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

export default apiClient;

// Usage: same instance, interceptor runs automatically
import apiClient from "./lib/apiClient";
const response = await apiClient.get("/products"); // Token auto-added
```

**Advantages**

- **Resource efficiency** — One connection, one file handle, one logger instance
- **Consistency** — All parts of code use the exact same instance
- **Lazy loading** — Instance created only when first used
- **Global access** — No need to pass the instance through function parameters

**Disadvantages**

- **Testing difficulty** — Hard to mock a singleton for unit tests
- **Hidden dependencies** — Global state can make dependencies unclear
- **Thread safety** — In multi-threaded environments, must handle concurrent access
- **Overused** — Many situations don't actually need a singleton

---

### 🏗️ Builder

**Problem**

> **Analogy:** You're constructing a sandwich at a deli counter. You don't buy all 50 ingredients at once. You build it step-by-step: bread choice → meat → cheese → toppings → condiments. At the end, you have a complete sandwich. The Builder pattern lets you construct complex objects the same way.

**What is Builder?**

- Separates **object construction** from its representation
- Builds complex objects **step-by-step** using a fluent interface
- Each setter returns the builder itself, allowing method chaining

**Problem Without Builder**

```javascript
// ❌ Copy-paste nightmare
async function registerUser(email, password, name, phone, age, role) {
  const userData = {
    email,
    password: await bcrypt.hash(password, 10),
    name,
    phone: phone || null,
    age: age ? Number(age) : null,
    role: "user", // default role
  };
  return prisma.user.create({ data: userData });
}

// When PM asks to add a 'role' field, you must update EVERY place
async function createAdminUser(email, password, name, role) {
  // Copy-paste the SAME logic again...
  const userData = {
    email,
    password: await bcrypt.hash(password, 10),
    name,
    role, // different role
  };
}

// And again in seed.js, tests, migrations...
```

**Solution With Builder**

```javascript
// ✅ Builder pattern — one source of truth

class UserBuilder {
  constructor() {
    this.data = {
      email: "",
      password: "",
      name: "",
      phone: null,
      age: null,
      role: "user", // default in ONE place
      isVerified: false,
    };
  }

  setEmail(email) {
    this.data.email = email;
    return this; // return builder for chaining
  }

  setPassword(password) {
    this.data.password = password;
    return this;
  }

  setName(name) {
    this.data.name = name;
    return this;
  }

  setPhone(phone) {
    this.data.phone = phone;
    return this;
  }

  setAge(age) {
    this.data.age = age ? Number(age) : null;
    return this;
  }

  setRole(role) {
    this.data.role = role;
    return this;
  }

  build() {
    return { ...this.data }; // return copy
  }

  async buildAndHash() {
    return {
      ...this.data,
      password: await bcrypt.hash(this.data.password, 10),
    };
  }
}

// Usage in /register
const userData = new UserBuilder()
  .setEmail(req.body.email)
  .setPassword(req.body.password)
  .setName(req.body.name)
  .setPhone(req.body.phone)
  .setAge(req.body.age)
  // role defaults to 'user'
  .buildAndHash();

await prisma.user.create({ data: userData });

// Usage in /admin/create
const userData = new UserBuilder()
  .setEmail(req.body.email)
  .setPassword(req.body.password)
  .setName(req.body.name)
  .setRole(req.body.role) // override default
  .buildAndHash();

// Usage in /seed
const userData = new UserBuilder()
  .setEmail("test@test.com")
  .setPassword("password123")
  .setName("Test User")
  .buildAndHash();
```

**Advantages**

- **Single source of truth** — Defaults and field logic live in one place
- **Fluent API** — Method chaining reads like natural language
- **Flexible construction** — Set only the fields you need; rest use defaults
- **Validation centralization** — Constructor logic applies everywhere
- **Easy refactoring** — Add a new field once; all usages get it automatically

**Disadvantages**

- **More code initially** — Requires writing setters for all fields
- **Not needed for simple objects** — Overkill if object has only 2-3 fields

---

### 🏭 Factory Method

**Problem**

> **Analogy:** Your payment system needs to support multiple providers — Stripe, Midtrans, Dana. Each has a different API, authentication method, and response format. You don't want your order checkout code to know the details of each provider. You need a single interface that says "process payment" — and the factory decides which provider to use based on configuration.

**What is Factory Method?**

- Creates objects **without specifying exact classes**
- Hides the complexity of instantiation
- Allows switching implementations easily
- Object creation logic is centralized in one "factory" function

**Backend Example: Notification Channels**

```javascript
// ✅ Factory pattern — hides provider details

function createNotification(type, to, message) {
  if (type === "email") {
    return {
      channel: "email",
      to,
      message,
      send: async () => {
        // Twilio, SendGrid, or any provider here
        console.log(`Email to ${to}: ${message}`);
        // await sendgridClient.send({ to, subject: '...', text: message });
      },
    };
  }

  if (type === "sms") {
    return {
      channel: "sms",
      to,
      message,
      send: async () => {
        console.log(`SMS to ${to}: ${message}`);
        // await twilio.messages.create({ to, body: message });
      },
    };
  }

  if (type === "push") {
    return {
      channel: "push",
      to,
      message,
      send: async () => {
        console.log(`Push to ${to}: ${message}`);
        // await firebase.sendToDevice(to, { body: message });
      },
    };
  }

  throw new Error(`Unknown notification type: ${type}`);
}

// Usage — route doesn't care HOW notification is sent
router.post("/send", async (req, res) => {
  const { type, to, message } = req.body;
  const notification = createNotification(type, to, message);
  await notification.send();
  res.json({ success: true });
});
```

**Frontend Example: Payment Gateway**

```javascript
// ✅ Factory for payment providers

function createPaymentGateway(provider, amount) {
  if (provider === "midtrans") {
    return {
      provider: "midtrans",
      pay: async () => {
        // Midtrans has its own JS library and snap payment
        window.snap.pay("MIDTRANS_TOKEN_FROM_BACKEND");
      },
    };
  }

  if (provider === "stripe") {
    return {
      provider: "stripe",
      pay: async () => {
        // Stripe uses different payment flow
        window.location.href = `https://checkout.stripe.com?session=${STRIPE_SESSION}`;
      },
    };
  }

  throw new Error(`Unknown payment provider: ${provider}`);
}

// Component doesn't care which provider — just calls pay()
function CheckoutButton() {
  const handlePay = async () => {
    const provider = process.env.REACT_APP_PAYMENT_PROVIDER; // from config
    const gateway = createPaymentGateway(provider, totalAmount);
    await gateway.pay();
  };

  return <button onClick={handlePay}>Pay Now</button>;
}
```

**Advantages**

- **Loose coupling** — Client code doesn't depend on concrete classes
- **Easy provider switching** — Change one line in config, not everywhere
- **Centralized logic** — All provider-specific code in one place
- **Extensibility** — Add new providers without changing client code

**Disadvantages**

- **Factory function grows** — With many types, the if/else chain becomes long
- **Solution** — Use a strategy map: `const strategies = { email: EmailNotif, sms: SmsNotif }`

---

### 🎭 Abstract Factory

**Problem**

> **Analogy:** A furniture company manufactures both modern and classical styles. They can't make a modern sofa with classical legs — the whole line must be cohesive. They have two separate factories: one producing modern furniture (modern sofa, modern table, modern chair), another producing classical furniture. When you order "modern collection," you get everything from the modern factory — guaranteed to match.

**What is Abstract Factory?**

- Creates **families of related objects** that must be used together
- Ensures objects from the same family are consistent
- Higher abstraction than Factory Method — factory of factories

**Frontend Example: UI Themes (Dark/Light)**

```javascript
// ✅ Abstract factory for UI components

function createDarkTheme() {
  return {
    button: {
      backgroundColor: "#1a1a1a",
      color: "#ffffff",
      border: "1px solid #555",
    },
    input: {
      backgroundColor: "#2a2a2a",
      color: "#ffffff",
      border: "1px solid #555",
    },
    card: {
      backgroundColor: "#1a1a1a",
      color: "#ffffff",
      boxShadow: "0 0 10px rgba(255,255,255,0.1)",
    },
  };
}

function createLightTheme() {
  return {
    button: {
      backgroundColor: "#ffffff",
      color: "#1a1a1a",
      border: "1px solid #ccc",
    },
    input: {
      backgroundColor: "#ffffff",
      color: "#1a1a1a",
      border: "1px solid #ccc",
    },
    card: {
      backgroundColor: "#ffffff",
      color: "#1a1a1a",
      boxShadow: "0 0 10px rgba(0,0,0,0.1)",
    },
  };
}

// Factory selector
function createUIFactory(theme) {
  if (theme === "dark") return createDarkTheme();
  if (theme === "light") return createLightTheme();
  throw new Error(`Unknown theme: ${theme}`);
}

// Component ensures ALL parts use same theme
function LoginForm({ theme }) {
  const ui = createUIFactory(theme);

  return (
    <div>
      <input
        placeholder="Email"
        style={ui.input} // ← guaranteed to match button style
      />
      <button style={ui.button}>Login</button>
    </div>
  );
}
```

**Backend Example: Storage Factories (Testing vs Production)**

```javascript
// ✅ Different storage implementations, same interface

function createProductRepository(env) {
  if (env === "production") {
    return {
      findAll: async () => prisma.product.findMany(),
      findById: async (id) => prisma.product.findUnique({ where: { id } }),
      create: async (data) => prisma.product.create({ data }),
    };
  }

  if (env === "test") {
    // In-memory storage for testing (no database needed)
    const store = [];
    let nextId = 1;
    return {
      findAll: async () => store,
      findById: async (id) => store.find((p) => p.id === id),
      create: async (data) => {
        const product = { ...data, id: nextId++ };
        store.push(product);
        return product;
      },
    };
  }

  throw new Error(`Unknown environment: ${env}`);
}

// Tests run without any database
const productRepo = createProductRepository(process.env.NODE_ENV);
const product = await productRepo.create({ name: "Test", price: 100 });
```

**Advantages**

- **Family consistency** — Components from the same factory always match
- **Easy theme switching** — Theme changes in one place
- **Testability** — Can use mock factory in tests without real database
- **Flexible updates** — Update entire family without touching client code

**Disadvantages**

- **More code** — Requires defining multiple factory functions
- **Overkill for simple cases** — Use when families of related objects matter

---

### 🧬 Prototype

**Problem**

> **Analogy:** You're mass-mailing 100 similar letters to customers. Each letter has the same header, footer, and body template — only the name and date change. Instead of writing each letter from scratch, you create a prototype letter, then copy it 100 times and fill in the variable parts.

**What is Prototype?**

- Creates new objects by **copying an existing object** (prototype)
- Avoids the overhead of re-creating complex objects from scratch
- Useful when creating variations of similar objects

**Backend Example: Bulk Email**

```javascript
// ✅ Prototype pattern — copy with overrides

function createEmailPrototype(from, subject, signature) {
  const prototype = {
    from,
    subject,
    signature,
    to: "",
    body: "",
    sentAt: null,
  };

  const clone = (overrides = {}) => ({
    ...prototype, // copy all fields from prototype
    ...overrides, // override only what changed
    sentAt: new Date().toISOString(),
  });

  return { clone };
}

// Usage: send to 1000 users
const { clone } = createEmailPrototype(
  "support@company.com",
  "Your Order Confirmation",
  "Best regards,\nCompany Team",
);

const emails = users.map((user) =>
  clone({
    to: user.email,
    body: `Hello ${user.name},\n\nYour order #${ORDER_ID} has been placed.`,
  }),
);

// Send all 1000 emails (each with different to/body)
await Promise.all(emails.map((email) => transporter.sendMail(email)));
```

**Frontend Example: Duplicate Form Data**

```javascript
// ✅ Duplicate product with most fields same

function duplicateProduct(product) {
  // Deep copy for nested objects
  const cloned = {
    ...product,
    id: undefined, // remove id so backend generates new one
    name: `${product.name} (Copy)`,
    images: [...product.images], // copy array too
  };
  return cloned;
}

// Usage in product list component
const duplicateItem = (product) => {
  const newProduct = duplicateProduct(product);
  setProducts([...products, newProduct]);
};
```

**Important: Shallow vs Deep Copy**

```javascript
// ❌ DANGER: Shallow copy with nested objects
const person1 = { name: "Eko", address: { city: "Jakarta" } };
const person2 = { ...person1 }; // shallow copy
person1.address.city = "Surabaya";
console.log(person2.address.city); // 'Surabaya' — BUG!

// ✅ Deep copy for nested objects
const person2 = {
  ...person1,
  address: { ...person1.address }, // copy nested object too
};
person1.address.city = "Surabaya";
console.log(person2.address.city); // 'Jakarta' — correct!
```

**Advantages**

- **Efficiency** — Cloning faster than reconstructing from scratch
- **Variation creation** — Build variations of a base object easily
- **Reduced duplication** — Common data stored once in prototype

**Disadvantages**

- **Shallow copy pitfalls** — Nested objects share references; needs deep copy
- **Not always clearer** — Building from constructor sometimes more explicit

---

### 🎯 Object Pool

**Problem**

> **Analogy:** A parking lot doesn't build a new car for every visitor. Instead, it has 100 cars ready to use (pool). When you arrive, you borrow a car. When you leave, you return it. The next visitor borrows the same car. Creating and destroying cars constantly would be inefficient and expensive.

**What is Object Pool?**

- Pre-creates a pool of expensive objects
- Objects are reused instead of created/destroyed repeatedly
- When done using an object, it's returned to the pool for reuse

**Backend Example: Database Connection Pool**

```javascript
// pg library already implements Object Pool internally
import { Pool } from "pg";

const pool = new Pool({
  host: "localhost",
  database: "mydb",
  user: "postgres",
  password: "password",
  max: 10, // keep 10 connections in pool
  min: 2, // minimum 2 always ready
  idleTimeoutMillis: 30000, // close after 30 sec unused
});

// Usage — pool automatically borrows/returns connections
const result = await pool.query("SELECT * FROM users");

// Behind the scenes:
// 1. Borrow a connection from pool (or create if pool not full)
// 2. Execute query
// 3. Return connection to pool for reuse
// 4. Next request reuses the same connection
```

**Frontend Example: Web Workers Pool for CPU Tasks**

```javascript
// npm install workerpool

import workerpool from "workerpool";

// Create pool of workers
const workerPool = workerpool.pool("./workers/csvWorker.js", {
  minWorkers: 2,
  maxWorkers: 4,
});

// Usage: parse large CSV without freezing UI
async function handleFileUpload(file) {
  const csv = await file.text();

  // Borrow a worker from pool
  const result = await workerPool.exec("parseCSV", [csv]);

  // Worker is returned to pool automatically
  setData(result);
}
```

**Advantages**

- **Performance** — Reusing objects faster than creating new ones
- **Resource control** — Limits max concurrent connections/threads
- **Stability** — Prevents "too many connections" crashes

**Disadvantages**

- **Complexity** — Must handle borrowed/return lifecycle
- **Memory overhead** — Pool takes memory even when not used

---

## 🔗 Structural Patterns

Patterns focused on **relationships between objects**, making systems more flexible.

---

### 🔌 Adapter

**Problem**

> **Analogy:** Your laptop uses USB-C, but your old keyboard is USB-A. You need an adapter (USB-C to USB-A) so they can work together. The adapter translates between two incompatible interfaces without changing the laptop or keyboard.

**What is Adapter?**

- Makes incompatible interfaces compatible
- Converts calls from one interface to another
- Useful when integrating with third-party libraries or services

**Backend Example: Payment Gateway Adapter**

```javascript
// ✅ Adapter pattern for different payment providers

// Your internal interface (what your app expects)
function createPaymentAdapter({ charge, refund, getStatus }) {
  return { charge, refund, getStatus };
}

// Midtrans adapter — wraps Midtrans SDK
const midtransAdapter = createPaymentAdapter({
  charge: async (amount, orderId) => {
    // Midtrans expects 'gross_amount', you use 'amount'
    const transaction = await snap.createTransaction({
      transaction_details: {
        order_id: orderId,
        gross_amount: amount, // translate here
      },
    });
    // Translate response to your format
    return { transactionId: transaction.token, status: "pending" };
  },

  refund: async (transactionId) => {
    const result = await snap.transaction.refund(transactionId);
    return { status: result.transaction_status };
  },

  getStatus: async (transactionId) => {
    const result = await snap.transaction.status(transactionId);
    return { status: result.transaction_status };
  },
});

// Stripe adapter — wraps Stripe SDK
const stripeAdapter = createPaymentAdapter({
  charge: async (amount, orderId) => {
    // Stripe expects 'amount' in cents, you use dollars
    const intent = await stripe.paymentIntents.create({
      amount: amount * 100, // translate to cents
      currency: "idr",
      metadata: { orderId }, // translate orderId to metadata
    });
    return { transactionId: intent.id, status: intent.status };
  },

  refund: async (transactionId) => {
    const result = await stripe.refunds.create({
      payment_intent: transactionId,
    });
    return { status: result.status };
  },

  getStatus: async (transactionId) => {
    const intent = await stripe.paymentIntents.retrieve(transactionId);
    return { status: intent.status };
  },
});

// Route doesn't know provider details — just calls adapter
router.post("/charge", async (req, res) => {
  const gateway =
    process.env.PAYMENT_PROVIDER === "stripe" ? stripeAdapter : midtransAdapter;

  const result = await gateway.charge(req.body.amount, req.body.orderId);
  res.json(result);
});
```

**Frontend Example: Storage Adapter**

```javascript
// ✅ Storage adapter for different backends

function createStorageAdapter({ get, set, remove }) {
  return { get, set, remove };
}

// localStorage adapter
const localStorageAdapter = createStorageAdapter({
  get: (key) => {
    const item = localStorage.getItem(key);
    try {
      return JSON.parse(item);
    } catch {
      return item;
    }
  },
  set: (key, value) => {
    localStorage.setItem(key, JSON.stringify(value));
  },
  remove: (key) => {
    localStorage.removeItem(key);
  },
});

// sessionStorage adapter (for temporary data)
const sessionStorageAdapter = createStorageAdapter({
  get: (key) => {
    const item = sessionStorage.getItem(key);
    try {
      return JSON.parse(item);
    } catch {
      return item;
    }
  },
  set: (key, value) => {
    sessionStorage.setItem(key, JSON.stringify(value));
  },
  remove: (key) => {
    sessionStorage.removeItem(key);
  },
});

// In-memory adapter (for testing or SSR)
const store = {};
const inMemoryAdapter = createStorageAdapter({
  get: (key) => store[key] ?? null,
  set: (key, value) => {
    store[key] = value;
  },
  remove: (key) => {
    delete store[key];
  },
});

// Usage — switch adapters without changing app code
const storage =
  process.env.NODE_ENV === "test" ? inMemoryAdapter : localStorageAdapter;

function useAuth() {
  const login = (token) => {
    storage.set("token", token); // same interface, different backend
  };
}
```

**Advantages**

- **Provider independence** — Switch providers by changing one line
- **No vendor lock-in** — Migrate to different service easily
- **Translation logic centralized** — All conversion in one place
- **Decoupling** — App code doesn't know provider details

**Disadvantages**

- **Extra layer** — Adds a level of indirection
- **Overhead** — Translation on every call (usually negligible)

---

### 🎭 Facade

**Problem**

> **Analogy:** A bank customer doesn't directly interact with the vault, accountants, and loan officers. They visit the customer service desk (facade), which handles everything behind the scenes. The facade shields the customer from complex internal operations.

**What is Facade?**

- Provides a **single, simplified interface** to a complex subsystem
- Hides internal complexity from the client
- Useful for multi-step processes

**Backend Example: Order Checkout**

```javascript
// ❌ Without facade — route knows too many details
router.post('/checkout', async (req, res) => {
  // 1. Validate stock
  for (const item of req.body.items) {
    const product = await prisma.product.findUnique(...);
    if (product.stock < item.quantity) throw new Error('Stock too low');
  }

  // 2. Calculate total
  let total = 0;
  for (const item of req.body.items) {
    total += product.price * item.quantity;
  }

  // 3. Create order
  const order = await prisma.order.create({ data: {...} });

  // 4. Create order items
  for (const item of req.body.items) {
    await prisma.orderItem.create({ data: {...} });
  }

  // 5. Reduce stock
  for (const item of req.body.items) {
    await prisma.product.update({ data: { stock: { decrement: qty } }});
  }

  // 6. Notify customer
  await prisma.notification.create(...);

  res.json(order);
});
```

**✅ With Facade — route is one line**

```javascript
// Facade hides all the complexity
async function checkout(userId, items) {
  // Validate all stock
  for (const item of items) {
    const product = await productRepository.findById(item.productId);
    if (!product || product.stock < item.quantity) {
      throw new Error("Stock insufficient");
    }
  }

  // Calculate and prepare
  let total = 0;
  const orderItems = [];
  for (const item of items) {
    const product = await productRepository.findById(item.productId);
    const subtotal = product.price * item.quantity;
    total += subtotal;
    orderItems.push({
      productId: item.productId,
      quantity: item.quantity,
      price: product.price,
      subtotal,
    });
  }

  // Create order
  const order = await orderRepository.insert({
    userId,
    total,
    status: "pending",
    items: orderItems,
  });

  // Reduce stock
  for (const item of items) {
    await productRepository.decrementStock(item.productId, item.quantity);
  }

  // Notify
  await notificationRepository.insert({
    userId,
    message: `Order #${order.id} created. Total: Rp ${total}`,
  });

  return order;
}

// Route becomes trivial
router.post("/checkout", async (req, res) => {
  const order = await checkout(req.user.userId, req.body.items);
  res.json(order);
});
```

**Advantages**

- **Simplicity** — Complex operations become one function call
- **Maintainability** — Complexity is hidden; easier to modify
- **Consistency** — All checkout code in one place
- **Reusability** — Other routes can also use the checkout facade

**Disadvantages**

- **Hides detail** — Debugging requires looking at facade implementation
- **Less flexible** — Hard to customize individual steps from outside

---

### 📚 Repository

**Problem**

> **Analogy:** Instead of reading books directly from the library shelves yourself, you ask the librarian. The librarian knows how books are organized, where they're stored, and how to retrieve them. You don't need to know anything about the shelving system — just ask the librarian.

**What is Repository?**

- **Data access layer** — single source for database queries
- Hides implementation details of database/storage
- Services talk to repository, not directly to database

**Structure**

```
Application Layer
       ↓
  Repository Layer (findAll, findById, insert, update, delete)
       ↓
Database/Storage Layer (SQL queries, API calls, etc)
```

**Example: Layered Architecture**

```javascript
// Layer 1: Repository — only place that touches database
class ProductRepository {
  async findAll() {
    return prisma.product.findMany();
  }

  async findById(id) {
    return prisma.product.findUnique({ where: { id } });
  }

  async insert(data) {
    return prisma.product.create({ data });
  }

  async update(id, data) {
    return prisma.product.update({ where: { id }, data });
  }

  async delete(id) {
    return prisma.product.delete({ where: { id } });
  }
}

// Layer 2: Service — business logic, uses repository
class ProductService {
  constructor(repository) {
    this.repo = repository;
  }

  async getAllProducts() {
    return this.repo.findAll();
  }

  async createProduct(data) {
    // Business logic: validate
    if (data.price <= 0) throw new Error("Price must be positive");
    if (data.stock < 0) throw new Error("Stock cannot be negative");

    // Call repository
    return this.repo.insert(data);
  }

  async updateProduct(id, data) {
    // Check exists first
    const product = await this.repo.findById(id);
    if (!product) throw new Error("Product not found");

    return this.repo.update(id, data);
  }
}

// Layer 3: Route — HTTP handling, uses service
router.get("/products", async (req, res) => {
  const products = await productService.getAllProducts();
  res.json(products);
});

router.post("/products", async (req, res) => {
  try {
    const product = await productService.createProduct(req.body);
    res.status(201).json(product);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});
```

**Advantages**

- **Testability** — Can mock repository for testing
- **Separation of concerns** — Database logic separate from business logic
- **Easy migration** — Switch databases (PostgreSQL → MongoDB) by changing repository
- **Reusability** — Multiple services can use same repository
- **Consistency** — All queries go through same methods

**Disadvantages**

- **More files** — Adds repository layer to your structure
- **Indirection** — Need to trace through layers to understand flow

---

## 🎬 Behavioral Patterns

Patterns focused on **communication between objects** and **responsibility allocation**.

---

### 📋 Template Method

**Problem**

> **Analogy:** Every recipe for baking has the same steps: prep ingredients → mix → bake → cool. The algorithm is identical. What changes is the specific ingredients and baking time. A template method captures the common algorithm while letting subclasses provide the varying details.

**What is Template Method?**

- Defines the **skeleton** of an algorithm in base class
- Lets **subclasses override specific steps** without changing algorithm structure
- Use when multiple classes have similar workflows with minor variations

**Backend Example: Export Data (CSV, JSON, Excel)**

```javascript
// ✅ Template method — shared algorithm, different implementations

function createExportTemplate({ getData, formatData, sendResponse }) {
  // This is the ALGORITHM — same for all export types
  const execute = async (req, res) => {
    // Step 1: Validate permission (always same)
    if (!req.user?.isAdmin) {
      return res.status(403).json({ error: "Not authorized" });
    }

    // Step 2: Get data (implementation varies)
    const data = await getData();

    // Step 3: Format data (implementation varies)
    const formatted = formatData(data);

    // Step 4: Send response (implementation varies)
    sendResponse(res, formatted);
  };

  return { execute };
}

// CSV implementation
const csvExport = createExportTemplate({
  getData: async () => productRepository.findAll(),
  formatData: (products) => {
    const rows = [
      "id,name,price,stock",
      ...products.map((p) => `${p.id},${p.name},${p.price},${p.stock}`),
    ];
    return rows.join("\n");
  },
  sendResponse: (res, csv) => {
    res.setHeader("Content-Type", "text/csv");
    res.send(csv);
  },
});

// JSON implementation — algorithm same, format different
const jsonExport = createExportTemplate({
  getData: async () => productRepository.findAll(),
  formatData: (products) => {
    return products.map((p) => ({
      id: p.id,
      name: p.name,
      price: p.price,
    }));
  },
  sendResponse: (res, json) => {
    res.json(json);
  },
});

// Routes trivial
router.get("/export/csv", (req, res) => csvExport.execute(req, res));
router.get("/export/json", (req, res) => jsonExport.execute(req, res));

// Future: add Excel export by just filling in formatData/sendResponse
```

**Frontend Example: Fetch Data Hook**

```javascript
// ✅ Custom hook with template method

function useFetchTemplate({ fetchFn, onSuccess, onError }) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  // This is the ALGORITHM — same for all data fetches
  const execute = async () => {
    setLoading(true);
    setError(null);

    try {
      // Step 1: Start loading (always same)
      // Step 2: Fetch (implementation varies)
      const result = await fetchFn();

      // Step 3: Save data (always same)
      setData(result);

      // Step 4: Callback (implementation varies)
      onSuccess?.(result);
    } catch (err) {
      // Step 5: Handle error (implementation varies)
      setError(err.message);
      onError?.(err);
    } finally {
      // Step 6: Stop loading (always same)
      setLoading(false);
    }
  };

  return { data, loading, error, execute };
}

// Product page — implement only the varying parts
function Products() {
  const { data: products, loading, error, execute } = useFetchTemplate({
    fetchFn: async () => {
      const res = await api.get('/products');
      return res.data;
    },
    onSuccess: (data) => console.log(`Loaded ${data.length} products`),
  });

  useEffect(() => { execute(); }, []);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>{error}</p>;
  return <div>{products.map(p => ...)}</div>;
}

// Orders page — different fetchFn, same algorithm
function Orders() {
  const { data: orders, loading, error, execute } = useFetchTemplate({
    fetchFn: async () => {
      const res = await api.get('/orders/my-orders');
      return res.data;
    },
  });

  useEffect(() => { execute(); }, []);

  if (loading) return <p>Loading...</p>;
  if (error) return <p>{error}</p>;
  return <div>{orders.map(o => ...)}</div>;
}
```

**Advantages**

- **Code reuse** — Algorithm in one place, variations plugged in
- **Consistency** — All variations follow same process
- **Easy enhancement** — Change algorithm once, benefits everything
- **Reduced duplication** — No copy-paste of algorithm

**Disadvantages**

- **Abstraction level** — Might be harder to understand initially
- **Overkill for simple cases** — Use when algorithm truly complex

---

### 🌳 Composite

**Problem**

> **Analogy:** A file system has both files and folders. A folder can contain files AND other folders recursively. Whether you're opening a file or a folder, the operation is the same conceptually — you "open" it. The composite pattern lets you treat both files and folders with the same interface.

**What is Composite?**

- Treats **individual objects and compositions** of objects uniformly
- Builds **tree structures** of objects recursively
- Useful for hierarchical data (file systems, org charts, menus)

**Example: E-commerce Category Hierarchy**

```javascript
// ✅ Composite pattern — same interface for items and collections

class Category {
  constructor(name) {
    this.name = name;
  }

  getDescription() {
    return this.name; // leaf node — just name
  }
}

class CompositeCategory {
  constructor(name, subCategories = []) {
    this.name = name;
    this.subCategories = subCategories;
  }

  add(category) {
    this.subCategories.push(category);
  }

  getDescription() {
    // recursive — gets name and all sub-categories
    let description = this.name;
    for (const sub of this.subCategories) {
      description += "\n  " + sub.getDescription().split("\n").join("\n  ");
    }
    return description;
  }
}

// Build hierarchy
const fashion = new CompositeCategory("Fashion");
const mensFashion = new CompositeCategory("Mens Fashion");
const womensFashion = new CompositeCategory("Womens Fashion");

mensFashion.add(new Category("Shirts"));
mensFashion.add(new Category("Pants"));

womensFashion.add(new Category("Dresses"));
womensFashion.add(new Category("Skirts"));

fashion.add(mensFashion);
fashion.add(womensFashion);

// Print entire tree with one call
console.log(fashion.getDescription());
// Fashion
//   Mens Fashion
//     Shirts
//     Pants
//   Womens Fashion
//     Dresses
//     Skirts
```

**Advantages**

- **Uniform interface** — Treat leaf and composite same way
- **Flexible hierarchy** — Build any depth without special cases
- **Recursive operations** — Operations naturally cascade through tree
- **Easy to extend** — Add new node types without changing clients

**Disadvantages**

- **Complexity** — Recursive logic can be harder to debug
- **Type safety** — Can't distinguish between leaf and composite at compile time

---

## 🎯 Quick Reference Table

| Pattern              | Problem                               | Solution                                | When to Use                                   |
| -------------------- | ------------------------------------- | --------------------------------------- | --------------------------------------------- |
| **Singleton**        | Need exactly one instance globally    | Restrict instantiation to one object    | Database connections, loggers, config         |
| **Builder**          | Complex object creation with defaults | Step-by-step construction with defaults | User profiles, complex configs                |
| **Factory Method**   | Hide object creation details          | Centralize creation logic               | Multiple implementations (Stripe vs Midtrans) |
| **Abstract Factory** | Ensure related objects work together  | Provide factory for object families     | UI themes (dark/light), environments          |
| **Prototype**        | Expensive object creation             | Clone existing objects                  | Bulk operations, form duplication             |
| **Object Pool**      | Expensive create/destroy cycles       | Reuse objects from a pool               | DB connections, thread pools                  |
| **Adapter**          | Incompatible interfaces               | Translate between interfaces            | Third-party library integration               |
| **Facade**           | Complex subsystem operations          | Simplified unified interface            | Multi-step business processes                 |
| **Repository**       | Database access scattered everywhere  | Centralize data access layer            | Separation of concerns, testability           |
| **Template Method**  | Similar algorithms with variations    | Share algorithm skeleton, vary details  | Export formats, data processing               |
| **Composite**        | Mixed individuals and collections     | Treat both uniformly recursively        | File systems, category trees                  |

---

## 💡 Best Practices

**✅ Do This**

- **Start simple** — Use patterns only when they solve actual problems
- **Name descriptively** — Pattern names aren't required (e.g., `exportService` instead of `exportFacade`)
- **Keep interfaces small** — Don't force unrelated methods together
- **Test behavior, not implementation** — Tests should not break when internal implementation changes
- **Document the pattern** — Leave a comment: `// Adapter: translates Stripe API to internal interface`
- **Use dependency injection** — Pass dependencies in, don't hardcode them

**❌ Avoid This**

- **Over-engineering** — Don't use 10 patterns when 2 would do
- **Pattern-driven design** — Design for your actual problems, not theoretical ones
- **Forcing patterns** — If a pattern feels awkward, don't force it
- **Pattern cargo cult** — Using patterns just because they sound smart
- **Ignoring language features** — Some patterns less relevant in modern JavaScript

**Pattern Selection Guide**

| Scenario               | Recommended Patterns                 |
| ---------------------- | ------------------------------------ |
| New project            | Start simple, add patterns as needed |
| Switching providers    | Adapter + Factory                    |
| Complex workflows      | Facade                               |
| Multiple similar pages | Template Method                      |
| Hierarchical data      | Composite                            |
| Database layer         | Repository + Singleton               |
| UI customization       | Abstract Factory                     |
| Expensive resources    | Object Pool                          |

**Anti-Pattern: The Pattern Police**

> Using patterns correctly requires judgment. Some code doesn't need patterns. Some problems have multiple valid solutions. The goal is clean, maintainable code — not "correct" pattern usage.

---
