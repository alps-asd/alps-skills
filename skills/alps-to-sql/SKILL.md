---
name: alps-to-sql
description: Generate SQL DDL from ALPS profile. Converts ALPS semantic descriptors to database table definitions with automatic type inference and index suggestions. Use when user says "ALPS to SQL", "alps2sql", "generate SQL from ALPS", "generate DDL", "create tables from ALPS", "ALPSからSQL", "SQLを生成", "テーブル定義を生成", or asks to create database schema from ALPS.
---

# ALPS to SQL DDL Converter

This skill generates SQL DDL (Data Definition Language) from ALPS profiles for database schema creation.

## When to Use This Skill

Use this skill when:
- Need to create database tables from ALPS states
- Want consistent schema from semantic descriptors
- Building backend implementation from API design

## What This Skill Generates

1. **CREATE TABLE statements** - From ALPS states
2. **Column definitions** - From semantic descriptors
3. **Primary keys** - From ID fields
4. **Indexes** - Based on field usage patterns
5. **Foreign keys** - From state relationships

## Supported Dialects

- PostgreSQL (default)
- MySQL/MariaDB
- SQLite

## Conversion Rules

### 1. ALPS States → Tables

Each ALPS state with data fields becomes a table:

```
ALPS:
{"id": "BlogPost", "title": "Blog Post", "descriptor": [
  {"href": "#postId"},
  {"href": "#title"},
  {"href": "#body"},
  {"href": "#createdAt"}
]}

↓ Converts to (PostgreSQL):

CREATE TABLE blog_post (
    post_id VARCHAR(64) PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    body TEXT NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 2. Semantic Fields → Column Types

| ALPS def (schema.org) | PostgreSQL | MySQL | SQLite |
|----------------------|------------|-------|--------|
| schema.org/identifier | VARCHAR(64) | VARCHAR(64) | TEXT |
| schema.org/Text | TEXT | TEXT | TEXT |
| schema.org/headline | VARCHAR(255) | VARCHAR(255) | TEXT |
| schema.org/Integer | INTEGER | INT | INTEGER |
| schema.org/Number | DECIMAL(10,2) | DECIMAL(10,2) | REAL |
| schema.org/Boolean | BOOLEAN | TINYINT(1) | INTEGER |
| schema.org/DateTime | TIMESTAMP | DATETIME | TEXT |
| schema.org/Date | DATE | DATE | TEXT |
| schema.org/Time | TIME | TIME | TEXT |
| schema.org/URL | VARCHAR(2048) | VARCHAR(2048) | TEXT |
| schema.org/Email | VARCHAR(255) | VARCHAR(255) | TEXT |

**Default Inference from Field Names:**
- `*Id`, `*ID` → VARCHAR(64) PRIMARY KEY
- `*Count`, `*Number`, `*Qty` → INTEGER
- `*Price`, `*Amount` → DECIMAL(10,2)
- `is*`, `has*`, `can*` → BOOLEAN
- `*At`, `created*`, `updated*` → TIMESTAMP
- `*Date` → DATE
- `*email*` → VARCHAR(255)
- `*url*` → VARCHAR(2048)
- `title`, `name` → VARCHAR(255)
- `*description`, `*body`, `*content` → TEXT
- Others → VARCHAR(255)

### 3. Naming Conventions

| ALPS | SQL |
|------|-----|
| BlogPost (state) | blog_post (table) |
| postId (field) | post_id (column) |
| createdAt (field) | created_at (column) |
| userId (FK reference) | user_id (column) |

**Rules:**
- PascalCase → snake_case
- camelCase → snake_case
- Singular table names (blog_post, not blog_posts)

### 4. Primary Keys

Fields with `*Id` or `*ID` at the start of state descriptors:

```
ALPS:
{"id": "BlogPost", "descriptor": [
  {"href": "#postId"},  ← First ID field = Primary Key
  ...
]}

↓ Converts to:

post_id VARCHAR(64) PRIMARY KEY
```

### 5. Foreign Keys

When a state references another state:

```
ALPS:
{"id": "BlogPost", "descriptor": [
  {"href": "#postId"},
  {"href": "#authorId"},  ← References Author
  {"href": "#Author"}     ← State reference
]}

↓ Converts to:

CREATE TABLE blog_post (
    post_id VARCHAR(64) PRIMARY KEY,
    author_id VARCHAR(64) NOT NULL,
    CONSTRAINT fk_blog_post_author
        FOREIGN KEY (author_id) REFERENCES author(author_id)
);
```

### 6. Indexes

**Automatic index generation for:**
- Foreign key columns
- Fields used in `safe` transitions (search/filter)
- Date/time fields (for sorting)
- Email, URL fields (for lookup)

```sql
CREATE INDEX idx_blog_post_author_id ON blog_post(author_id);
CREATE INDEX idx_blog_post_created_at ON blog_post(created_at);
```

### 7. NOT NULL vs NULLABLE

**NOT NULL by default for:**
- Primary key fields
- Required fields in transitions
- Core entity fields

**NULLABLE:**
- Fields marked optional in ALPS doc
- Fields not in create transition

### 8. Default Values

| Field Pattern | Default |
|---------------|---------|
| created_at | CURRENT_TIMESTAMP |
| updated_at | CURRENT_TIMESTAMP |
| is*, has* (boolean) | FALSE |
| *count, *number | 0 |

## Step-by-Step Implementation Process

### Step 1: Analyze ALPS Profile

1. Parse the ALPS JSON/XML
2. Identify:
   - States that represent entities (have ID fields)
   - Semantic fields with types
   - Relationships between states
3. Build entity-relationship map

### Step 2: Determine Table Structure

For each state:
1. Extract table name (PascalCase → snake_case)
2. Identify primary key (first *Id field)
3. Map fields to columns
4. Identify foreign keys (references to other states)

### Step 3: Generate DDL

```sql
-- Generated from ALPS: {alps.title}
-- Generated at: {timestamp}

-- Table: {table_name}
-- ALPS State: {state_id}
CREATE TABLE {table_name} (
    {columns}
    {constraints}
);

{indexes}
```

### Step 4: Output Order

1. Tables without foreign keys first
2. Tables with foreign keys (dependency order)
3. Indexes after all tables

## Output Format

- Standard SQL DDL
- Comments with ALPS metadata
- Dialect-specific syntax when needed

## Example

### Input (ALPS)

```json
{
  "alps": {
    "title": "Blog API",
    "descriptor": [
      {"id": "authorId", "title": "Author ID", "def": "https://schema.org/identifier"},
      {"id": "authorName", "title": "Author Name", "def": "https://schema.org/name"},
      {"id": "email", "title": "Email", "def": "https://schema.org/email"},

      {"id": "postId", "title": "Post ID", "def": "https://schema.org/identifier"},
      {"id": "title", "title": "Title", "def": "https://schema.org/headline"},
      {"id": "body", "title": "Body", "def": "https://schema.org/articleBody"},
      {"id": "createdAt", "title": "Created At", "def": "https://schema.org/dateCreated"},
      {"id": "published", "title": "Published", "def": "https://schema.org/Boolean"},

      {"id": "Author", "title": "Author", "descriptor": [
        {"href": "#authorId"},
        {"href": "#authorName"},
        {"href": "#email"}
      ]},

      {"id": "BlogPost", "title": "Blog Post", "descriptor": [
        {"href": "#postId"},
        {"href": "#title"},
        {"href": "#body"},
        {"href": "#authorId"},
        {"href": "#createdAt"},
        {"href": "#published"}
      ]}
    ]
  }
}
```

### Output (PostgreSQL)

```sql
-- ============================================
-- Generated from ALPS: Blog API
-- Dialect: PostgreSQL
-- Generated at: 2025-01-12T12:00:00Z
-- ============================================

-- Table: author
-- ALPS State: Author
CREATE TABLE author (
    author_id VARCHAR(64) PRIMARY KEY,
    author_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL
);

CREATE UNIQUE INDEX idx_author_email ON author(email);

-- Table: blog_post
-- ALPS State: BlogPost
CREATE TABLE blog_post (
    post_id VARCHAR(64) PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    body TEXT NOT NULL,
    author_id VARCHAR(64) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    published BOOLEAN NOT NULL DEFAULT FALSE,

    CONSTRAINT fk_blog_post_author
        FOREIGN KEY (author_id) REFERENCES author(author_id)
        ON DELETE CASCADE
);

CREATE INDEX idx_blog_post_author_id ON blog_post(author_id);
CREATE INDEX idx_blog_post_created_at ON blog_post(created_at);
CREATE INDEX idx_blog_post_published ON blog_post(published);
```

### Output (MySQL)

```sql
-- ============================================
-- Generated from ALPS: Blog API
-- Dialect: MySQL
-- Generated at: 2025-01-12T12:00:00Z
-- ============================================

-- Table: author
CREATE TABLE author (
    author_id VARCHAR(64) NOT NULL,
    author_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL,
    PRIMARY KEY (author_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE UNIQUE INDEX idx_author_email ON author(email);

-- Table: blog_post
CREATE TABLE blog_post (
    post_id VARCHAR(64) NOT NULL,
    title VARCHAR(255) NOT NULL,
    body TEXT NOT NULL,
    author_id VARCHAR(64) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    published TINYINT(1) NOT NULL DEFAULT 0,
    PRIMARY KEY (post_id),

    CONSTRAINT fk_blog_post_author
        FOREIGN KEY (author_id) REFERENCES author(author_id)
        ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE INDEX idx_blog_post_author_id ON blog_post(author_id);
CREATE INDEX idx_blog_post_created_at ON blog_post(created_at);
```

## Options

| Option | Description | Default |
|--------|-------------|---------|
| `--dialect` | SQL dialect (postgresql, mysql, sqlite) | postgresql |
| `--drop` | Include DROP TABLE IF EXISTS | false |
| `--no-fk` | Skip foreign key constraints | false |
| `--no-index` | Skip index generation | false |

## Important Notes

- **Dependency order**: Tables are ordered by foreign key dependencies
- **ON DELETE**: Default is CASCADE, can be changed to SET NULL or RESTRICT
- **Character set**: UTF-8 for MySQL/MariaDB
- **Column order**: Primary key first, then data fields, then foreign keys, then timestamps

## Troubleshooting

- If table relationships are unclear, check for state references in descriptors
- For many-to-many relationships, generate junction tables manually
- If column types are wrong, add explicit `def` to ALPS field
- For advanced constraints (CHECK, UNIQUE), add manually after generation
