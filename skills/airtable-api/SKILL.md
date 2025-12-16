---
name: airtable-api
description: Comprehensive guide for building applications that integrate with Airtable using the pyairtable Python library or direct REST API. Use when working with Airtable bases, tables, records, formulas, or building data-driven applications that use Airtable as a backend.
license: Complete terms in LICENSE.txt
---

# Airtable API Development Guide

## Overview

Build robust applications that integrate with Airtable as a backend database. This skill covers authentication, CRUD operations, formulas, filtering, sorting, and best practices for both Python (pyairtable) and TypeScript/JavaScript implementations.

---

# Process

## Phase 1: Authentication Setup

### Personal Access Tokens (PATs) - Recommended

Airtable deprecated API keys in February 2024. Use Personal Access Tokens:

1. Go to https://airtable.com/create/tokens
2. Create a token with appropriate scopes:
   - `data.records:read` - Read records
   - `data.records:write` - Create/update/delete records
   - `schema.bases:read` - Read base schema

### Secure Token Storage

**CRITICAL**: Never commit tokens to version control.

```python
# Python - Use environment variables
import os
api_key = os.environ['AIRTABLE_API_KEY']
```

```typescript
// TypeScript - Use environment variables
const apiKey = process.env.AIRTABLE_API_KEY;
```

**Configuration Options:**
- Store in `.env` file (add to `.gitignore`)
- Use `AIRTABLE_API_KEY_FILE` for file-based storage
- Use secrets management in production (AWS Secrets Manager, etc.)

---

## Phase 2: Understanding Airtable Structure

### Key Concepts

- **Base**: A database container (like a database)
- **Table**: A collection of records (like a table)
- **Record**: A row of data with fields
- **Field**: A column with a specific type
- **View**: A filtered/sorted subset of records

### IDs vs Names

- **Base ID**: Starts with `app` (e.g., `appABC123xyz`)
- **Table ID**: Starts with `tbl` (e.g., `tblXYZ789abc`)
- **Record ID**: Starts with `rec` (e.g., `recDEF456ghi`)
- **Field ID**: Starts with `fld` (e.g., `fldGHI123jkl`)

**Best Practice**: Use IDs in code for stability; table/field names can change.

---

## Phase 3: Python Implementation (pyairtable)

### Installation

```bash
pip install pyairtable
```

### Basic Setup

```python
from pyairtable import Api

# Initialize API client
api = Api(os.environ['AIRTABLE_API_KEY'])

# Get table reference
table = api.table('appBaseId', 'tblTableId')
# Or by name
table = api.table('appBaseId', 'Table Name')
```

### CRUD Operations

See [Python CRUD Reference](./reference/python-crud.md) for complete examples.

**Quick Reference:**

```python
# CREATE
record = table.create({'Name': 'John', 'Email': 'john@example.com'})

# READ
all_records = table.all()
single_record = table.get('recXXX')
first_match = table.first(formula="Name='John'")

# UPDATE
table.update('recXXX', {'Email': 'new@example.com'})

# DELETE
table.delete('recXXX')
```

### Batch Operations

```python
# Batch create (up to 10 per call, auto-batched)
records = table.batch_create([
    {'Name': 'Alice'},
    {'Name': 'Bob'}
])

# Batch update
table.batch_update([
    {'id': 'rec1', 'fields': {'Status': 'Done'}},
    {'id': 'rec2', 'fields': {'Status': 'Done'}}
])

# Batch delete
table.batch_delete(['rec1', 'rec2', 'rec3'])

# Upsert (create or update by key field)
table.batch_upsert(
    [{'fields': {'Email': 'alice@example.com', 'Name': 'Alice'}}],
    key_fields=['Email']
)
```

### ORM Pattern

See [Python ORM Reference](./reference/python-orm.md) for complete guide.

```python
from pyairtable.orm import Model, fields

class Task(Model):
    class Meta:
        base_id = "appXXX"
        table_name = "Tasks"
        api_key = os.environ['AIRTABLE_API_KEY']

    name = fields.TextField("Name")
    status = fields.SelectField("Status")
    due_date = fields.DateField("Due Date")
    assignee = fields.SingleLinkField("Assignee")
```

---

## Phase 4: TypeScript/JavaScript Implementation

### Direct REST API

For TypeScript applications, use the Airtable REST API directly or use `airtable` npm package.

See [TypeScript Reference](./reference/typescript-api.md) for complete patterns.

**Quick Reference:**

```typescript
// Using fetch
const response = await fetch(
  `https://api.airtable.com/v0/${baseId}/${tableId}`,
  {
    headers: {
      'Authorization': `Bearer ${apiKey}`,
      'Content-Type': 'application/json'
    }
  }
);
```

---

## Phase 5: Formulas and Filtering

### pyairtable Formula Builder

```python
from pyairtable.formulas import match, AND, OR, field

# Simple match
records = table.all(formula=match({'Status': 'Active'}))

# Complex formulas
formula = AND(
    field('Status') == 'Active',
    field('Priority') > 3
)
records = table.all(formula=formula)
```

### Common Formula Patterns

| Use Case | Formula |
|----------|---------|
| Text equals | `{Name}='John'` |
| Number greater than | `{Age}>30` |
| Contains text | `FIND('search', {Field})>0` |
| Not empty | `NOT({Field}='')` |
| Date after | `IS_AFTER({Date}, '2024-01-01')` |
| Multiple conditions | `AND({A}='x', {B}='y')` |

See [Formulas Reference](./reference/formulas.md) for complete guide.

---

## Phase 6: Sorting and Pagination

### Sorting

```python
# Single field ascending
table.all(sort=['Name'])

# Multiple fields, descending with minus prefix
table.all(sort=['Priority', '-CreatedDate'])

# Explicit direction
table.all(sort=[('Name', 'asc'), ('Date', 'desc')])
```

### Pagination

```python
# Limit results
records = table.all(max_records=100)

# Iterate with automatic pagination
for record in table.all():
    process(record)

# Use views for pre-filtered/sorted data
records = table.all(view='Active Tasks')
```

---

## Phase 7: Error Handling

### Rate Limiting

- **Limit**: 5 requests per second per base
- **Response**: 429 status code when exceeded
- **pyairtable**: Automatic retry (up to 5 times by default)

```python
# Custom retry strategy
from pyairtable.api.retrying import Retry

api = Api(
    api_key,
    retry_strategy=Retry(total=10, backoff_factor=0.5)
)
```

### Common Errors

| Status | Meaning | Solution |
|--------|---------|----------|
| 401 | Unauthorized | Check API token |
| 403 | Forbidden | Check token scopes |
| 404 | Not found | Verify base/table/record IDs |
| 422 | Invalid data | Check field types and names |
| 429 | Rate limited | Automatic retry or add delays |

```python
import requests

try:
    record = table.create({'Name': 'Test'})
except requests.exceptions.HTTPError as e:
    error_data = e.response.json()
    print(f"Error: {error_data}")
```

---

# Reference Files

## Documentation Library

Load these resources as needed during development:

- [Python CRUD Reference](./reference/python-crud.md) - Complete CRUD examples
- [Python ORM Reference](./reference/python-orm.md) - Model-based patterns
- [TypeScript Reference](./reference/typescript-api.md) - REST API patterns
- [Formulas Reference](./reference/formulas.md) - Formula syntax and examples

## External Documentation

- **pyairtable docs**: https://pyairtable.readthedocs.io/
- **Airtable API docs**: https://airtable.com/developers/web/api/introduction
- **Airtable Formulas**: https://support.airtable.com/docs/formula-field-reference

---

# Best Practices

## Data Modeling

1. **Use linked records** for relationships instead of duplicating data
2. **Create views** for common queries (filter/sort applied server-side)
3. **Use lookup/rollup fields** for computed data instead of client-side calculations
4. **Normalize data** to avoid redundancy

## Performance

1. **Use formulas** to filter server-side, not client-side
2. **Leverage views** for pre-filtered data sets
3. **Batch operations** for bulk changes (up to 10 per call)
4. **Cache responses** when data doesn't change frequently
5. **Use field filtering** to request only needed fields

```python
# Request specific fields only
records = table.all(fields=['Name', 'Email'])
```

## Security

1. **Never expose API tokens** in client-side code
2. **Use server-side proxy** for browser applications
3. **Apply minimal scopes** to tokens
4. **Rotate tokens** regularly

## Mobile/Web App Architecture

For mobile-friendly web apps using Airtable as backend:

1. **Use a backend proxy** (Next.js API routes, Express, etc.)
2. **Implement caching** for frequently accessed data
3. **Handle offline scenarios** with local storage
4. **Paginate data** to reduce initial load times
5. **Use optimistic updates** for better UX
