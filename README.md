# alps-skills

Claude Code skills for ALPS (Application-Level Profile Semantics) development.

## Overview

A collection of AI-powered skills for designing RESTful APIs using ALPS profiles. These skills help generate, validate, and convert ALPS profiles to other formats.

```
                    ALPS
                      │
    ┌─────────────────┼─────────────────┐
    ↓                 ↓                 ↓
API Spec           Data Layer        Validation
├─ openapi         ├─ jsonschema     └─ alps
├─ graphql         └─ sql
└─ (asyncapi)
```

## Available Skills

| Skill | Description |
|-------|-------------|
| alps | Generate, validate, and improve ALPS profiles |
| alps-to-openapi | Convert ALPS profiles to OpenAPI specifications |
| alps-to-graphql | Convert ALPS profiles to GraphQL schema |
| alps-to-jsonschema | Convert ALPS profiles to JSON Schema |
| alps-to-sql | Convert ALPS profiles to SQL DDL |

## Installation

### Claude Code Plugin (Recommended)

```bash
# 1. Add marketplace
/plugin marketplace add alps-asd/alps-skills

# 2. Install
/plugin install alps-skills
```

### Update

```bash
/plugin update alps-skills
```

### Remove

```bash
/plugin uninstall alps-skills
```

### Manual Installation (Alternative)

```bash
git clone https://github.com/alps-asd/alps-skills.git
cp -r alps-skills/skills/ /path/to/your/project/.claude/skills/
```

## Usage

Describe your task naturally - Claude will automatically select the appropriate skill.

Examples:
- "Create an ALPS profile for a blog application"
- "Validate my ALPS profile"
- "Convert this ALPS to OpenAPI"
- "Generate GraphQL schema from ALPS"
- "Create database tables from ALPS"

## Skills

### alps

Generate, validate, and improve ALPS profiles for RESTful API design.

**Features:**
- Generate ALPS from natural language descriptions
- Validate existing profiles with detailed error messages
- Suggest improvements for better API design

**Requirements:**
- [app-state-diagram](https://github.com/alps-asd/app-state-diagram) (`asd` command) for validation

### alps-to-openapi

Convert ALPS profiles to OpenAPI 3.1 specifications.

**Features:**
- Automatic HTTP method inference from ALPS types
- Schema generation from semantic descriptors
- Spectral validation of generated specs

### alps-to-graphql

Convert ALPS profiles to GraphQL schema.

**Features:**
- Types from ALPS states
- Queries from safe transitions
- Mutations from unsafe/idempotent transitions
- Input types from transition parameters

### alps-to-jsonschema

Convert ALPS profiles to JSON Schema.

**Features:**
- Response schemas from ALPS states
- Request schemas from transition parameters
- Type inference from schema.org definitions

### alps-to-sql

Convert ALPS profiles to SQL DDL.

**Features:**
- CREATE TABLE from ALPS states
- Column types from semantic descriptors
- Foreign keys from state relationships
- Index suggestions

**Supported Dialects:**
- PostgreSQL (default)
- MySQL/MariaDB
- SQLite

## Conversion Rules

| ALPS type | OpenAPI | GraphQL | SQL |
|-----------|---------|---------|-----|
| safe | GET | Query | SELECT |
| unsafe | POST | Mutation | INSERT |
| idempotent | PUT/DELETE | Mutation | UPDATE/DELETE |

## References

- [ALPS Specification](http://alps.io/spec/)
- [app-state-diagram](https://github.com/alps-asd/app-state-diagram)
- [Schema.org](https://schema.org/)

## License

MIT
