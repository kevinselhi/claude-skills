# Python pyairtable CRUD Reference

Complete examples for Create, Read, Update, and Delete operations using pyairtable.

## Setup

```python
import os
from pyairtable import Api

# Initialize
api = Api(os.environ['AIRTABLE_API_KEY'])
table = api.table('appBaseId', 'tblTableId')
```

---

## CREATE Operations

### Create Single Record

```python
# Basic create
record = table.create({
    'Name': 'John Doe',
    'Email': 'john@example.com',
    'Age': 30,
    'Active': True
})

print(record['id'])  # rec123...
print(record['fields'])  # {'Name': 'John Doe', ...}
print(record['createdTime'])  # '2024-01-15T...'
```

### Create with Typecast

Automatically convert string values to appropriate types:

```python
# String '30' converted to integer
record = table.create({'Age': '30'}, typecast=True)
```

### Batch Create

Create multiple records efficiently (auto-batched in groups of 10):

```python
records_to_create = [
    {'Name': 'Alice', 'Email': 'alice@example.com'},
    {'Name': 'Bob', 'Email': 'bob@example.com'},
    {'Name': 'Charlie', 'Email': 'charlie@example.com'}
]

created_records = table.batch_create(records_to_create)

for record in created_records:
    print(f"Created: {record['id']}")
```

### Create with Linked Records

```python
# Link to existing records by ID
record = table.create({
    'Name': 'Task 1',
    'Assignee': ['recUSER123'],  # Single link
    'Tags': ['recTAG1', 'recTAG2']  # Multiple links
})
```

---

## READ Operations

### Get All Records

```python
# Get all records (handles pagination automatically)
all_records = table.all()

for record in all_records:
    print(record['fields']['Name'])
```

### Get Single Record by ID

```python
record = table.get('recXXX123')
print(record['fields'])
```

### Get First Matching Record

```python
# Get first record (useful for single-result queries)
first = table.first()

# With formula filter
first = table.first(formula="{Email}='john@example.com'")
```

### Filter with Formula

```python
# Simple text match
records = table.all(formula="{Status}='Active'")

# Numeric comparison
records = table.all(formula="{Age}>30")

# Multiple conditions
records = table.all(formula="AND({Status}='Active', {Age}>30)")

# Not empty
records = table.all(formula="NOT({Email}='')")

# Contains text
records = table.all(formula="FIND('john', LOWER({Email}))>0")
```

### Sort Results

```python
# Sort ascending
records = table.all(sort=['Name'])

# Sort descending (minus prefix)
records = table.all(sort=['-CreatedDate'])

# Multiple sort fields
records = table.all(sort=['Priority', '-Name'])

# Explicit direction
records = table.all(sort=[('Priority', 'desc'), ('Name', 'asc')])
```

### Filter by View

```python
# Use a view's filters and sort order
records = table.all(view='Active Tasks')

# Combine with formula
records = table.all(view='Active Tasks', formula="{Priority}>3")
```

### Limit and Paginate

```python
# Limit total records
records = table.all(max_records=50)

# Limit per page (for large datasets)
records = table.all(page_size=100)

# Get specific fields only
records = table.all(fields=['Name', 'Email', 'Status'])
```

### Cell Format

```python
# Get formula results as strings (default)
records = table.all()  # cell_format='string'

# Get raw formula results
records = table.all(cell_format='json')
```

### Timezone

```python
# Specify timezone for date fields
records = table.all(
    timezone='America/New_York',
    user_locale='en-us'
)
```

---

## UPDATE Operations

### Update Single Record

```python
# Update specific fields (merge with existing)
updated = table.update('recXXX123', {
    'Status': 'Completed',
    'CompletedDate': '2024-01-15'
})
```

### Replace Entire Record

```python
# Replace all fields (destructive)
updated = table.update('recXXX123', {
    'Name': 'New Name',
    'Status': 'Active'
}, replace=True)
```

### Batch Update

```python
updates = [
    {'id': 'rec1', 'fields': {'Status': 'Done'}},
    {'id': 'rec2', 'fields': {'Status': 'Done'}},
    {'id': 'rec3', 'fields': {'Status': 'Done'}}
]

updated_records = table.batch_update(updates)
```

### Upsert (Create or Update)

```python
# Create if not exists, update if exists (match by key fields)
results = table.batch_upsert(
    [
        {'fields': {'Email': 'alice@example.com', 'Name': 'Alice Updated'}},
        {'fields': {'Email': 'newuser@example.com', 'Name': 'New User'}}
    ],
    key_fields=['Email']  # Match on Email field
)

for record in results:
    was_created = record.get('createdTime') is not None
    print(f"{record['id']}: {'created' if was_created else 'updated'}")
```

### Update Linked Records

```python
# Set linked records
table.update('recXXX', {
    'Assignee': ['recUSER456'],  # Replace links
    'Tags': ['recTAG1', 'recTAG3']  # Update multiple links
})

# Clear linked records
table.update('recXXX', {'Tags': []})
```

---

## DELETE Operations

### Delete Single Record

```python
# Returns deleted record info
deleted = table.delete('recXXX123')
print(f"Deleted: {deleted['id']}")
```

### Batch Delete

```python
record_ids = ['rec1', 'rec2', 'rec3']
deleted_records = table.batch_delete(record_ids)

for deleted in deleted_records:
    print(f"Deleted: {deleted['id']}")
```

### Delete with Confirmation

```python
# Find and delete records matching criteria
records = table.all(formula="{Status}='Archived'")
ids_to_delete = [r['id'] for r in records]

if ids_to_delete:
    print(f"Deleting {len(ids_to_delete)} records...")
    table.batch_delete(ids_to_delete)
```

---

## Record Structure

All operations return records in this structure:

```python
{
    'id': 'recXXX123',  # Unique record ID
    'fields': {
        'Name': 'John Doe',
        'Email': 'john@example.com',
        'Age': 30,
        'Tags': ['recTAG1', 'recTAG2'],  # Linked record IDs
        'Created': '2024-01-15T10:30:00.000Z'
    },
    'createdTime': '2024-01-15T10:30:00.000Z'
}
```

---

## Field Type Handling

| Airtable Type | Python Type | Notes |
|---------------|-------------|-------|
| Single line text | `str` | |
| Long text | `str` | May include markdown |
| Number | `int` or `float` | |
| Checkbox | `bool` | |
| Single select | `str` | Option value |
| Multiple select | `list[str]` | List of options |
| Date | `str` | ISO format |
| Link to record | `list[str]` | List of record IDs |
| Attachment | `list[dict]` | Contains url, filename, etc. |
| Email | `str` | |
| URL | `str` | |
| Phone | `str` | |

---

## Error Handling

```python
import requests

try:
    record = table.create({'Name': 'Test'})
except requests.exceptions.HTTPError as e:
    if e.response.status_code == 422:
        # Invalid field or type
        print(f"Validation error: {e.response.json()}")
    elif e.response.status_code == 404:
        # Record/table not found
        print("Not found")
    else:
        raise
```
