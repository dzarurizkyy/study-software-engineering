# 🗄️ Database Design Case Studies

Practical database design patterns for real-world features — covering multi-language support and notification systems.

---

## 📋 Table of Contents

- [Multi-Language Support](#-multi-language-support)
  - [The Problem with Adding Columns](#the-problem-with-adding-columns)
  - [The Solution: Translation Table](#the-solution-translation-table)
  - [SQL Implementation](#sql-implementation)
  - [Querying by Language](#querying-by-language)
- [Notification System](#-notification-system)
  - [Feature Requirements](#feature-requirements)
  - [Table Design Overview](#table-design-overview)
  - [User & Notification Tables](#user--notification-tables)
  - [Category Table](#category-table)
  - [Read / Unread Status](#read--unread-status)
  - [Notification Counter](#notification-counter)
- [Quick Reference](#-quick-reference)

---

## 🌐 Multi-Language Support

> **Key Insight:** Never add a new column per language. Instead, extract all translatable fields into a separate translation table — adding a new language then requires zero schema changes.

**The Problem with Adding Columns**

A naive approach adds language-specific columns directly to the main table:

```
❌ Anti-pattern: column-per-language

categories
┌────────────────────────────────────────────────────────┐
│ id │ name │ name_id │ name_jp │ desc │ desc_id │ desc_jp │
│    │ (EN) │  (ID)   │  (JP)   │ (EN) │  (ID)   │  (JP)   │
└────────────────────────────────────────────────────────┘

Problem: every new language = 2 more columns per translatable field
         10 translatable fields × 5 languages = 50 extra columns
```

**The Solution: Translation Table**

Split the table into a parent table (language-neutral data) and a translation table (all translatable fields).

```
✅ Recommended: one-to-many translation table

categories                    categories_translation
┌─────────────────────┐       ┌──────────────────────────────────┐
│ id (PK)             │──┐    │ category_id (FK, PK)             │
│ position            │  └──► │ language    (PK)    ◄── composite │
└─────────────────────┘       │ name                             │
                              │ description                      │
                              └──────────────────────────────────┘

Composite PK (category_id + language) guarantees no duplicate
translations for the same category in the same language.
```

**Schema Design**

| Table | Column | Type | Constraint |
|---|---|---|---|
| `categories` | id | VARCHAR(100) | PRIMARY KEY |
| `categories` | position | INT | NOT NULL |
| `categories_translation` | category_id | VARCHAR(100) | PK, FK → categories.id |
| `categories_translation` | language | VARCHAR(100) | PK |
| `categories_translation` | name | VARCHAR(100) | NOT NULL |
| `categories_translation` | description | TEXT | NULL allowed |

**SQL Implementation**

```sql
CREATE TABLE categories (
  id       VARCHAR(100) NOT NULL,
  position INT          NOT NULL,
  PRIMARY KEY (id)
) ENGINE = InnoDB;

CREATE TABLE categories_translation (
  category_id VARCHAR(100) NOT NULL,
  language    VARCHAR(100) NOT NULL,
  name        VARCHAR(100) NOT NULL,
  description TEXT         NULL,
  PRIMARY KEY (category_id, language),
  CONSTRAINT FK_categories_translation
    FOREIGN KEY (category_id) REFERENCES categories (id)
) ENGINE = InnoDB;
```

**Querying by Language**

```sql
-- Show all categories in Indonesian
SELECT *
FROM categories c
JOIN categories_translation ct ON c.id = ct.category_id
WHERE ct.language = 'in_ID'
ORDER BY c.position;

-- Switch to English — only the WHERE clause changes
WHERE ct.language = 'en_US'
```

**Adding a New Language**

```
Adding Japanese support:

❌ Old approach:  ALTER TABLE categories ADD COLUMN name_jp VARCHAR(100);
                  ALTER TABLE categories ADD COLUMN desc_jp TEXT;
                  ... repeat for every translatable field

✅ New approach:  INSERT INTO categories_translation
                  (category_id, language, name, description)
                  VALUES ('FOOD', 'ja_JP', '食べ物', '...');
                  -- Done. No schema change needed.
```

---

## 🔔 Notification System

> **Key Insight:** Separate concerns into distinct tables: one for notification content, one for read status. Never mix per-user state into a global broadcast row.

**Feature Requirements**

| Feature | Description |
|---|---|
| **Inbox** | List all notifications for the logged-in user |
| **Categories** | Filter by type — `promo` (global broadcast) or `info` (user-specific) |
| **Read / Unread** | Track which notifications each user has read |
| **Counter** | Show the number of unread notifications in the menu badge |

**Table Design Overview**

```
┌──────────┐        ┌──────────────────────┐        ┌──────────────────────┐
│   user   │──┐     │     notification     │        │ notification_read    │
│──────────│  │     │──────────────────────│        │──────────────────────│
│ id (PK)  │  └──►  │ id (PK, AUTO_INC)    │──────► │ id (PK, AUTO_INC)    │
│ name     │        │ title                │        │ notification_id (FK) │
└──────────┘        │ detail               │        │ user_id (FK)         │
                    │ created_at           │        │ is_read (BOOLEAN)    │
     ┌──────────┐   │ user_id (FK, NULL)   │        └──────────────────────┘
     │ category │   │ category_id (FK)     │
     │──────────│   └──────────────────────┘
     │ id (PK)  │◄──┘
     │ name     │
     └──────────┘

user_id = NULL  →  global broadcast (visible to all users)
user_id = 'eko' →  private notification (visible only to 'eko')
```

**User & Notification Tables**

```sql
CREATE TABLE user (
  id   VARCHAR(100) NOT NULL,
  name VARCHAR(100) NOT NULL,
  PRIMARY KEY (id)
) ENGINE = InnoDB;

CREATE TABLE notification (
  id          INT          NOT NULL AUTO_INCREMENT,
  title       VARCHAR(255) NOT NULL,
  detail      TEXT         NOT NULL,
  created_at  TIMESTAMP    NOT NULL,
  user_id     VARCHAR(100) NULL,       -- NULL = global broadcast
  category_id VARCHAR(100) NULL,
  PRIMARY KEY (id)
) ENGINE = InnoDB;
```

**Why `user_id` is Nullable**

```
Scenario: "11.11 Sale" promo must reach 1,000,000 users

❌ Insert 1 row per user:  INSERT ... (promo, user1)
                           INSERT ... (promo, user2)
                           ... × 1,000,000 rows

✅ Insert 1 row with NULL: INSERT ... (promo, NULL)

Query for user 'eko':
  WHERE (user_id = 'eko' OR user_id IS NULL)
  ORDER BY created_at DESC
```

**Category Table**

```sql
CREATE TABLE category (
  id   VARCHAR(100) NOT NULL,
  name VARCHAR(100) NOT NULL,
  PRIMARY KEY (id)
) ENGINE = InnoDB;

-- Sample data
INSERT INTO category (id, name) VALUES ('info', 'Info'), ('promo', 'Promo');
```

Querying with category filter:

```sql
SELECT n.*, c.name AS category_name
FROM notification n
JOIN category c ON n.category_id = c.id
WHERE (n.user_id = 'eko' OR n.user_id IS NULL)
  AND c.id = 'promo'           -- optional filter by category
ORDER BY n.created_at DESC;
```

**Read / Unread Status**

Storing `is_read` on the notification row itself breaks for global broadcasts — marking it read for one user would mark it read for everyone. A separate table solves this.

```sql
CREATE TABLE notification_read (
  id              INT          NOT NULL AUTO_INCREMENT,
  notification_id INT          NOT NULL,
  user_id         VARCHAR(100) NOT NULL,
  is_read         BOOLEAN      NOT NULL,
  PRIMARY KEY (id),
  CONSTRAINT FK_notification_read_notification
    FOREIGN KEY (notification_id) REFERENCES notification (id),
  CONSTRAINT FK_notification_read_user
    FOREIGN KEY (user_id) REFERENCES user (id)
) ENGINE = InnoDB;
```

Query using `LEFT JOIN` — rows absent from `notification_read` are treated as unread (NULL):

```sql
SELECT n.*, nr.is_read
FROM notification n
LEFT JOIN notification_read nr
       ON nr.notification_id = n.id
WHERE (n.user_id = 'eko' OR n.user_id IS NULL)
  AND (nr.user_id = 'eko' OR nr.user_id IS NULL)
ORDER BY n.created_at DESC;

-- is_read = true  → already read
-- is_read = null  → not yet read (no row in notification_read)
```

**Read / Unread Decision Flow**

```
New notification arrives
        │
        ▼
  user_id = NULL?
  ┌─────────┴──────────┐
 YES                   NO
(global)           (private)
  │                    │
  └──────────┬─────────┘
             ▼
  Store in notification table
  (do NOT pre-populate notification_read)
             │
             ▼
  User opens the notification
             │
             ▼
  INSERT INTO notification_read
  (notification_id, user_id, is_read)
  VALUES (?, ?, TRUE)
```

**Notification Counter**

No extra table needed — reuse the same LEFT JOIN query with a COUNT and filter for unread:

```sql
SELECT COUNT(*) AS unread_count
FROM notification n
LEFT JOIN notification_read nr
       ON nr.notification_id = n.id
      AND nr.user_id = 'eko'
WHERE (n.user_id = 'eko' OR n.user_id IS NULL)
  AND nr.is_read IS NULL;   -- absent row = unread
```

---

## 📌 Quick Reference

| Pattern | When to Use | Key Rule |
|---|---|---|
| **Translation table** | Any field that must display in multiple languages | Composite PK: `(entity_id, language)` |
| **Nullable user_id** | Notifications or content shared with all users | `NULL` = global; query with `OR user_id IS NULL` |
| **Separate read table** | Per-user read state on shared/global content | `LEFT JOIN` + treat NULL as unread |
| **COUNT unread** | Badge counter in UI | `WHERE is_read IS NULL` on the left-joined read table |
| **Composite PK** | Prevents duplicate entries without an extra ID column | Use when the combination of two FKs is naturally unique |
