---
name: alps-to-graphql
description: Generate GraphQL schema from ALPS profile. Converts ALPS semantic descriptors to GraphQL types, queries, and mutations with automatic type inference.
---

# ALPS to GraphQL Converter

This skill generates a complete GraphQL schema from ALPS profiles following GraphQL best practices.

## When to Use This Skill

Use this skill when:
- Converting an ALPS profile to GraphQL API
- Need to generate GraphQL types from semantic descriptors
- Want consistent schema structure from ALPS states and transitions

## What This Skill Generates

1. **Types** - GraphQL object types from ALPS states
2. **Queries** - From `safe` transitions
3. **Mutations** - From `unsafe` and `idempotent` transitions
4. **Input Types** - From transition parameters
5. **Enums** - From constrained values (if defined)

## Conversion Rules

### 1. ALPS States → GraphQL Types

Each ALPS state (semantic descriptor with nested descriptors) becomes a GraphQL type:

```
ALPS:
{"id": "BlogPost", "title": "Blog Post", "descriptor": [
  {"href": "#postId"},
  {"href": "#title"},
  {"href": "#body"},
  {"href": "#author"}
]}

↓ Converts to:

GraphQL:
type BlogPost {
  postId: ID!
  title: String!
  body: String!
  author: Author
}
```

### 2. ALPS Transitions → Queries/Mutations

| ALPS type | GraphQL |
|-----------|---------|
| `safe` | Query |
| `unsafe` | Mutation (create) |
| `idempotent` | Mutation (update/delete) |

**Query Example:**
```
ALPS:
{"id": "goBlogPost", "type": "safe", "rt": "#BlogPost", "descriptor": [
  {"href": "#postId"}
]}

↓ Converts to:

GraphQL:
type Query {
  blogPost(postId: ID!): BlogPost
}
```

**Mutation Example:**
```
ALPS:
{"id": "doCreateBlogPost", "type": "unsafe", "rt": "#BlogPost", "descriptor": [
  {"href": "#title"},
  {"href": "#body"}
]}

↓ Converts to:

GraphQL:
type Mutation {
  createBlogPost(input: CreateBlogPostInput!): BlogPost
}

input CreateBlogPostInput {
  title: String!
  body: String!
}
```

### 3. Semantic Fields → GraphQL Fields

| ALPS def (schema.org) | GraphQL Type |
|----------------------|--------------|
| schema.org/identifier | ID! |
| schema.org/Text | String |
| schema.org/Integer | Int |
| schema.org/Number | Float |
| schema.org/Boolean | Boolean |
| schema.org/DateTime | DateTime (custom scalar) |
| schema.org/Date | Date (custom scalar) |
| schema.org/URL | String |
| schema.org/Email | String |

**Default:** If no `def` is specified, infer from field name:
- `*Id`, `*ID` → `ID!`
- `*Count`, `*Number`, `*Qty` → `Int`
- `*Price`, `*Amount` → `Float`
- `is*`, `has*`, `can*` → `Boolean`
- `*At`, `*Date`, `*Time` → `DateTime`
- Others → `String`

### 4. List Transitions → List Queries

Transitions ending with `List` return arrays:

```
ALPS:
{"id": "goBlogPostList", "type": "safe", "rt": "#BlogPostList"}

↓ Converts to:

GraphQL:
type Query {
  blogPosts: [BlogPost!]!
}
```

### 5. Naming Conventions

| ALPS Pattern | GraphQL Convention |
|--------------|-------------------|
| `goXxx` | `xxx` (query) |
| `goXxxList` | `xxxs` (plural query) |
| `doCreateXxx` | `createXxx` (mutation) |
| `doUpdateXxx` | `updateXxx` (mutation) |
| `doDeleteXxx` | `deleteXxx` (mutation) |
| `XxxList` state | `[Xxx!]!` return type |

### 6. Nested States → Relationships

If a state references another state, create a relationship:

```
ALPS:
{"id": "BlogPost", "descriptor": [
  {"href": "#postId"},
  {"href": "#Author"}  ← References Author state
]}

↓ Converts to:

GraphQL:
type BlogPost {
  postId: ID!
  author: Author  # Relationship
}
```

### 7. Input Types for Mutations

Create input types for mutations with parameters:

```graphql
# For doCreateBlogPost
input CreateBlogPostInput {
  title: String!
  body: String!
}

# For doUpdateBlogPost
input UpdateBlogPostInput {
  postId: ID!
  title: String
  body: String
}
```

**Rules:**
- Create operations: all fields required (except ID)
- Update operations: ID required, other fields optional

### 8. Delete Mutations

Delete mutations return Boolean or deleted object:

```graphql
type Mutation {
  deleteBlogPost(postId: ID!): Boolean!
}
```

## Step-by-Step Implementation Process

### Step 1: Analyze ALPS Profile

1. Parse the ALPS JSON/XML
2. Identify:
   - Semantic fields (ontology layer)
   - States (taxonomy layer)
   - Transitions (choreography layer)
3. Build relationship map between states

### Step 2: Generate Custom Scalars

If DateTime fields exist:

```graphql
scalar DateTime
scalar Date
```

### Step 3: Generate Types

For each ALPS state, create a GraphQL type:

```graphql
"""
{alps.title or alps.doc}
"""
type {StateName} {
  {fields from nested descriptors}
}
```

### Step 4: Generate Queries

For each `safe` transition:

```graphql
type Query {
  """
  {transition.title or transition.doc}
  """
  {queryName}({parameters}): {ReturnType}
}
```

### Step 5: Generate Mutations

For each `unsafe` or `idempotent` transition:

```graphql
type Mutation {
  """
  {transition.title or transition.doc}
  """
  {mutationName}(input: {InputType}!): {ReturnType}
}

input {InputType} {
  {fields from transition descriptors}
}
```

### Step 6: Generate Complete Schema

Combine all parts:

```graphql
# Custom Scalars
scalar DateTime

# Types
type BlogPost { ... }
type Author { ... }

# Input Types
input CreateBlogPostInput { ... }
input UpdateBlogPostInput { ... }

# Queries
type Query {
  blogPost(postId: ID!): BlogPost
  blogPosts: [BlogPost!]!
  author(authorId: ID!): Author
}

# Mutations
type Mutation {
  createBlogPost(input: CreateBlogPostInput!): BlogPost
  updateBlogPost(input: UpdateBlogPostInput!): BlogPost
  deleteBlogPost(postId: ID!): Boolean!
}
```

## Output Format

- GraphQL SDL format (.graphql)
- Include descriptions from ALPS `title` and `doc`
- Use standard GraphQL conventions (camelCase for fields, PascalCase for types)

## Example

### Input (ALPS)

```json
{
  "alps": {
    "title": "Blog API",
    "descriptor": [
      {"id": "postId", "title": "Post ID", "def": "https://schema.org/identifier"},
      {"id": "title", "title": "Title", "def": "https://schema.org/headline"},
      {"id": "body", "title": "Body", "def": "https://schema.org/articleBody"},
      {"id": "createdAt", "title": "Created At", "def": "https://schema.org/dateCreated"},

      {"id": "BlogPost", "title": "Blog Post", "descriptor": [
        {"href": "#postId"},
        {"href": "#title"},
        {"href": "#body"},
        {"href": "#createdAt"}
      ]},

      {"id": "goBlogPost", "type": "safe", "rt": "#BlogPost", "descriptor": [
        {"href": "#postId"}
      ]},
      {"id": "goBlogPostList", "type": "safe", "rt": "#BlogPostList"},
      {"id": "doCreateBlogPost", "type": "unsafe", "rt": "#BlogPost", "descriptor": [
        {"href": "#title"},
        {"href": "#body"}
      ]},
      {"id": "doUpdateBlogPost", "type": "idempotent", "rt": "#BlogPost", "descriptor": [
        {"href": "#postId"},
        {"href": "#title"},
        {"href": "#body"}
      ]},
      {"id": "doDeleteBlogPost", "type": "idempotent", "rt": "#BlogPostList", "descriptor": [
        {"href": "#postId"}
      ]}
    ]
  }
}
```

### Output (GraphQL)

```graphql
"""
Blog API
"""

scalar DateTime

"""
Blog Post
"""
type BlogPost {
  "Post ID"
  postId: ID!
  "Title"
  title: String!
  "Body"
  body: String!
  "Created At"
  createdAt: DateTime!
}

input CreateBlogPostInput {
  title: String!
  body: String!
}

input UpdateBlogPostInput {
  postId: ID!
  title: String
  body: String
}

type Query {
  "Get a blog post by ID"
  blogPost(postId: ID!): BlogPost

  "Get all blog posts"
  blogPosts: [BlogPost!]!
}

type Mutation {
  "Create a new blog post"
  createBlogPost(input: CreateBlogPostInput!): BlogPost

  "Update an existing blog post"
  updateBlogPost(input: UpdateBlogPostInput!): BlogPost

  "Delete a blog post"
  deleteBlogPost(postId: ID!): Boolean!
}
```

## Validation

After generating GraphQL schema, validate:

```bash
# Using graphql-js
npx graphql-inspector validate schema.graphql

# Or check syntax
npx graphql schema.graphql
```

## Important Notes

- **Nullability**: Fields are non-null by default, add `?` suffix in ALPS or nullable in doc to make optional
- **Connections**: For pagination, consider generating Connection types (Relay style) for list queries
- **Subscriptions**: Not generated by default; add manually if needed
- **Directives**: Custom directives can be added manually after generation

## Troubleshooting

- If type relationships are unclear, ask user for clarification
- If list vs single item is ambiguous, check transition naming pattern
- For complex nested structures, generate intermediate types
- Always validate generated schema before use
