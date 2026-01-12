---
name: alps-to-jsonschema
description: Generate JSON Schema from ALPS profile. Converts ALPS semantic descriptors to JSON Schema for request/response validation with automatic type inference.
---

# ALPS to JSON Schema Converter

This skill generates JSON Schema from ALPS profiles for API request/response validation.

## When to Use This Skill

Use this skill when:
- Need validation schemas for API endpoints
- Want to generate request/response schemas from ALPS states
- Building OpenAPI specs that need component schemas

## What This Skill Generates

1. **Response Schemas** - From ALPS states
2. **Request Schemas** - From transition parameters
3. **Shared Definitions** - Reusable schema components

## Conversion Rules

### 1. ALPS States → JSON Schema Objects

Each ALPS state becomes a JSON Schema:

```
ALPS:
{"id": "BlogPost", "title": "Blog Post", "descriptor": [
  {"href": "#postId"},
  {"href": "#title"},
  {"href": "#body"}
]}

↓ Converts to:

JSON Schema:
{
  "$id": "BlogPost.json",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Blog Post",
  "type": "object",
  "properties": {
    "postId": { "type": "string" },
    "title": { "type": "string" },
    "body": { "type": "string" }
  },
  "required": ["postId", "title", "body"]
}
```

### 2. Semantic Fields → JSON Schema Properties

| ALPS def (schema.org) | JSON Schema |
|----------------------|-------------|
| schema.org/identifier | `{"type": "string"}` |
| schema.org/Text | `{"type": "string"}` |
| schema.org/Integer | `{"type": "integer"}` |
| schema.org/Number | `{"type": "number"}` |
| schema.org/Boolean | `{"type": "boolean"}` |
| schema.org/DateTime | `{"type": "string", "format": "date-time"}` |
| schema.org/Date | `{"type": "string", "format": "date"}` |
| schema.org/Time | `{"type": "string", "format": "time"}` |
| schema.org/URL | `{"type": "string", "format": "uri"}` |
| schema.org/Email | `{"type": "string", "format": "email"}` |

**Default Inference from Field Names:**
- `*Id`, `*ID` → `{"type": "string"}`
- `*Count`, `*Number`, `*Qty` → `{"type": "integer"}`
- `*Price`, `*Amount` → `{"type": "number"}`
- `is*`, `has*`, `can*` → `{"type": "boolean"}`
- `*At`, `*Date` → `{"type": "string", "format": "date-time"}`
- `*email*` → `{"type": "string", "format": "email"}`
- `*url*`, `*Uri*` → `{"type": "string", "format": "uri"}`
- Others → `{"type": "string"}`

### 3. ALPS Title/Doc → JSON Schema Metadata

```
ALPS:
{"id": "title", "title": "Post Title", "doc": {"value": "The title of the blog post"}}

↓ Converts to:

JSON Schema:
"title": {
  "type": "string",
  "title": "Post Title",
  "description": "The title of the blog post"
}
```

### 4. List States → Array Schemas

States ending with `List` or containing list descriptors:

```
ALPS:
{"id": "BlogPostList", "descriptor": [
  {"href": "#BlogPost"}
]}

↓ Converts to:

JSON Schema:
{
  "$id": "BlogPostList.json",
  "type": "object",
  "properties": {
    "items": {
      "type": "array",
      "items": { "$ref": "BlogPost.json" }
    }
  }
}
```

### 5. Transition Parameters → Request Schemas

For each transition, generate request schema:

```
ALPS:
{"id": "doCreateBlogPost", "type": "unsafe", "descriptor": [
  {"href": "#title"},
  {"href": "#body"}
]}

↓ Converts to:

{
  "$id": "CreateBlogPostRequest.json",
  "title": "Create Blog Post Request",
  "type": "object",
  "properties": {
    "title": { "type": "string" },
    "body": { "type": "string" }
  },
  "required": ["title", "body"],
  "additionalProperties": false
}
```

### 6. Required Fields

**Rules for determining required fields:**
- All fields in transition parameters are required by default
- ID fields in response schemas are required
- Fields with `(optional)` in doc are not required
- List/array fields may be optional

### 7. Nested States → $ref

When a state contains another state:

```
ALPS:
{"id": "BlogPost", "descriptor": [
  {"href": "#postId"},
  {"href": "#Author"}  ← References Author state
]}

↓ Converts to:

{
  "properties": {
    "postId": { "type": "string" },
    "author": { "$ref": "Author.json" }
  }
}
```

## Step-by-Step Implementation Process

### Step 1: Analyze ALPS Profile

1. Parse the ALPS JSON/XML
2. Categorize descriptors:
   - Semantic fields (leaf nodes)
   - States (composite descriptors)
   - Transitions (with type attribute)
3. Build dependency graph for $ref resolution

### Step 2: Generate Field Schemas

For each semantic field:

```json
"{fieldId}": {
  "type": "{inferred_type}",
  "title": "{alps.title}",
  "description": "{alps.doc.value}"
}
```

### Step 3: Generate State Schemas (Response)

For each ALPS state:

```json
{
  "$id": "{StateName}.json",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "{alps.title}",
  "description": "{alps.doc.value}",
  "type": "object",
  "properties": {
    // Generated from nested descriptors
  },
  "required": [/* ID and key fields */]
}
```

### Step 4: Generate Request Schemas

For each transition with parameters:

| Transition Type | Schema Name Pattern |
|-----------------|---------------------|
| doCreateXxx | CreateXxxRequest.json |
| doUpdateXxx | UpdateXxxRequest.json |
| doDeleteXxx | DeleteXxxRequest.json |
| goXxx (with params) | XxxQuery.json |

### Step 5: Output Structure

```
schemas/
├── response/
│   ├── BlogPost.json
│   ├── BlogPostList.json
│   └── Author.json
├── request/
│   ├── CreateBlogPostRequest.json
│   ├── UpdateBlogPostRequest.json
│   └── DeleteBlogPostRequest.json
└── definitions/
    └── common.json  (shared definitions)
```

## Output Format

- JSON Schema draft 2020-12 (default)
- Can also generate draft-07 if specified
- Pretty-printed JSON with 2-space indentation

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
      {"id": "published", "title": "Published", "def": "https://schema.org/Boolean"},

      {"id": "BlogPost", "title": "Blog Post", "descriptor": [
        {"href": "#postId"},
        {"href": "#title"},
        {"href": "#body"},
        {"href": "#createdAt"},
        {"href": "#published"}
      ]},

      {"id": "doCreateBlogPost", "type": "unsafe", "rt": "#BlogPost", "descriptor": [
        {"href": "#title"},
        {"href": "#body"}
      ]},
      {"id": "doUpdateBlogPost", "type": "idempotent", "rt": "#BlogPost", "descriptor": [
        {"href": "#postId"},
        {"href": "#title"},
        {"href": "#body"},
        {"href": "#published"}
      ]}
    ]
  }
}
```

### Output (JSON Schema)

**response/BlogPost.json:**
```json
{
  "$id": "BlogPost.json",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Blog Post",
  "type": "object",
  "properties": {
    "postId": {
      "type": "string",
      "title": "Post ID"
    },
    "title": {
      "type": "string",
      "title": "Title"
    },
    "body": {
      "type": "string",
      "title": "Body"
    },
    "createdAt": {
      "type": "string",
      "format": "date-time",
      "title": "Created At"
    },
    "published": {
      "type": "boolean",
      "title": "Published"
    }
  },
  "required": ["postId", "title", "body", "createdAt", "published"]
}
```

**request/CreateBlogPostRequest.json:**
```json
{
  "$id": "CreateBlogPostRequest.json",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Create Blog Post Request",
  "type": "object",
  "properties": {
    "title": {
      "type": "string",
      "title": "Title"
    },
    "body": {
      "type": "string",
      "title": "Body"
    }
  },
  "required": ["title", "body"],
  "additionalProperties": false
}
```

**request/UpdateBlogPostRequest.json:**
```json
{
  "$id": "UpdateBlogPostRequest.json",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "Update Blog Post Request",
  "type": "object",
  "properties": {
    "postId": {
      "type": "string",
      "title": "Post ID"
    },
    "title": {
      "type": "string",
      "title": "Title"
    },
    "body": {
      "type": "string",
      "title": "Body"
    },
    "published": {
      "type": "boolean",
      "title": "Published"
    }
  },
  "required": ["postId"],
  "additionalProperties": false
}
```

## Validation

After generating JSON Schema, validate:

```bash
# Using ajv-cli
npx ajv validate -s schema.json -d data.json

# Check schema syntax
npx ajv compile -s schema.json
```

## Important Notes

- **additionalProperties**: Set to `false` for request schemas (strict validation)
- **$ref**: Use relative paths for local references
- **Format keywords**: Only added when schema.org def is present or field name matches pattern
- **Required arrays**: Update based on create vs update operations

## Troubleshooting

- If type inference is wrong, add explicit `def` to ALPS field
- For complex validation rules (min/max, pattern), add manually after generation
- If nested objects are flattened incorrectly, check state references
- Use `--draft 07` for compatibility with older validators
