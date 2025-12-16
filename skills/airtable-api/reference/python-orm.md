# Python pyairtable ORM Reference

Model-based patterns for working with Airtable using pyairtable's ORM layer.

## Overview

The ORM provides a Django-style interface for Airtable, mapping Python classes to tables and instances to records.

---

## Defining Models

### Basic Model

```python
import os
from pyairtable.orm import Model, fields

class Contact(Model):
    class Meta:
        base_id = "appXXXXXXXXXXXXXX"
        table_name = "Contacts"  # Can be table ID (tblXXX) or name
        api_key = os.environ['AIRTABLE_API_KEY']

    # Define fields
    name = fields.TextField("Name")
    email = fields.EmailField("Email")
    phone = fields.TextField("Phone")
    is_active = fields.CheckboxField("Active")
```

### Meta Options

```python
class Meta:
    base_id = "appXXX"  # Required
    table_name = "Table Name"  # Required - table ID or name
    api_key = os.environ['AIRTABLE_API_KEY']  # Required

    # Optional settings
    timeout = (5, 10)  # (connect, read) timeout in seconds
    typecast = True  # Auto-convert field types
    use_field_ids = False  # Use field IDs instead of names
    memoize = True  # Cache model instances
```

---

## Field Types

### Text Fields

```python
from pyairtable.orm import fields

# Basic text
name = fields.TextField("Name")
name = fields.RequiredTextField("Name")  # Cannot be None

# Multiline text (same storage, semantic difference)
description = fields.TextField("Description")
```

### Number Fields

```python
# Integer
age = fields.IntegerField("Age")
age = fields.RequiredIntegerField("Age")

# Decimal/Float
price = fields.FloatField("Price")
price = fields.RequiredFloatField("Price")

# Currency (stored as number)
amount = fields.FloatField("Amount")

# Rating (1-5 or custom)
rating = fields.IntegerField("Rating")
```

### Boolean Fields

```python
# Checkbox
is_active = fields.CheckboxField("Active")
# Returns True or False (never None)
```

### Date/Time Fields

```python
# Date only
due_date = fields.DateField("Due Date")

# DateTime
created_at = fields.DatetimeField("Created At")
```

### Selection Fields

```python
# Single select
status = fields.SelectField("Status")
# Returns string value: "Active", "Pending", etc.

# Multiple select
tags = fields.MultipleSelectField("Tags")
# Returns list: ["urgent", "feature"]
```

### Linked Record Fields

```python
# Multiple linked records
tasks = fields.LinkField("Tasks", Task)
# Returns list of Task model instances

# Single linked record
manager = fields.SingleLinkField("Manager", Employee)
# Returns single Employee instance or None
```

### Special Fields

```python
# Email (validated)
email = fields.EmailField("Email")

# URL
website = fields.UrlField("Website")

# Attachment (list of file objects)
files = fields.AttachmentsField("Files")

# Barcode
barcode = fields.BarcodeField("Barcode")
```

### Read-Only Fields

```python
# Auto-number (read-only)
id_number = fields.AutoNumberField("ID")

# Count (rollup count, read-only)
task_count = fields.CountField("Task Count")

# Created time
created = fields.CreatedTimeField("Created")

# Last modified time
modified = fields.LastModifiedTimeField("Modified")

# Created by
created_by = fields.CreatedByField("Created By")

# Last modified by
modified_by = fields.LastModifiedByField("Modified By")
```

---

## CRUD Operations

### Create

```python
# Create instance
contact = Contact(
    name="John Doe",
    email="john@example.com",
    is_active=True
)

# Save to Airtable
contact.save()

# Access the record ID
print(contact.id)  # recXXX...
```

### Read

```python
# Get all records
contacts = Contact.all()

# Get first matching record
contact = Contact.first(formula="Name='John Doe'")

# Filter with formulas
active_contacts = Contact.all(formula=Contact.is_active == True)

# Sort
contacts = Contact.all(sort=['name', '-created_at'])

# Use a view
contacts = Contact.all(view='Active Contacts')

# Combine filters
contacts = Contact.all(
    formula=Contact.is_active == True,
    sort=['name'],
    view='Main View',
    max_records=100
)
```

### Update

```python
# Fetch and update
contact = Contact.first(formula="Name='John Doe'")
contact.email = "newemail@example.com"
contact.save()  # Only saves changed fields

# Force save all fields
contact.save(force=True)
```

### Delete

```python
# Delete single record
contact = Contact.first(formula="Name='John Doe'")
contact.delete()

# Batch delete
contacts = Contact.all(formula="{Status}='Archived'")
Contact.batch_delete(contacts)
```

---

## Batch Operations

### Batch Save

```python
contacts = [
    Contact(name="Alice", email="alice@example.com"),
    Contact(name="Bob", email="bob@example.com"),
    Contact(name="Charlie", email="charlie@example.com")
]

# Save all at once (batched in groups of 10)
Contact.batch_save(contacts)
```

### Batch Delete

```python
# Delete multiple records
contacts = Contact.all(formula="{Status}='Inactive'")
Contact.batch_delete(contacts)
```

---

## Working with Linked Records

### Defining Relationships

```python
class Project(Model):
    class Meta:
        base_id = "appXXX"
        table_name = "Projects"
        api_key = os.environ['AIRTABLE_API_KEY']

    name = fields.TextField("Name")
    tasks = fields.LinkField("Tasks", "Task")  # Forward reference as string


class Task(Model):
    class Meta:
        base_id = "appXXX"
        table_name = "Tasks"
        api_key = os.environ['AIRTABLE_API_KEY']

    title = fields.TextField("Title")
    project = fields.SingleLinkField("Project", Project)
    assignee = fields.SingleLinkField("Assignee", "User")
```

### Accessing Linked Records

```python
# Get linked records (lazy loaded)
project = Project.first()
for task in project.tasks:
    print(task.title)
    print(task.assignee.name)  # Access nested link

# Set linked records
task.project = some_project
task.save()

# Link multiple records
project.tasks = [task1, task2, task3]
project.save()
```

---

## Formula Building with Models

### Using Field References

```python
# Field comparison
formula = Contact.is_active == True

# Multiple conditions
formula = AND(
    Contact.is_active == True,
    Contact.name != ""
)

# Numeric comparisons
formula = Task.priority > 3

# Combine in queries
contacts = Contact.all(formula=Contact.is_active == True)
```

---

## Validation and Errors

### Required Fields

```python
class Task(Model):
    # Required - will raise error if None on save
    title = fields.RequiredTextField("Title")

    # Optional - can be None
    description = fields.TextField("Description")

# This will raise an error
task = Task(description="No title")
task.save()  # MissingValueError: title is required
```

### Type Validation

```python
# With typecast enabled, strings are converted
class Meta:
    typecast = True

task.priority = "5"  # Converted to integer 5 on save
```

---

## Advanced Patterns

### Model Inheritance (Not Recommended)

Airtable ORM doesn't support inheritance well. Use composition instead:

```python
# Composition pattern
class BaseFields:
    @staticmethod
    def add_common_fields(model_class):
        model_class.created_at = fields.CreatedTimeField("Created")
        model_class.modified_at = fields.LastModifiedTimeField("Modified")
```

### Memoization

```python
class Contact(Model):
    class Meta:
        memoize = True  # Cache instances by ID

# Same record returns same instance
contact1 = Contact.first()
contact2 = Contact.first()
contact1 is contact2  # True if same record
```

### Dynamic Table Selection

```python
# For multi-tenant or dynamic tables
class DynamicModel(Model):
    class Meta:
        base_id = None  # Set dynamically
        table_name = None

    name = fields.TextField("Name")

# Set at runtime
DynamicModel.Meta.base_id = tenant_base_id
DynamicModel.Meta.table_name = tenant_table
```

---

## Example: Complete Task Management Model

```python
import os
from pyairtable.orm import Model, fields
from pyairtable.formulas import AND, OR

class User(Model):
    class Meta:
        base_id = os.environ['AIRTABLE_BASE_ID']
        table_name = "Users"
        api_key = os.environ['AIRTABLE_API_KEY']

    name = fields.RequiredTextField("Name")
    email = fields.EmailField("Email")
    role = fields.SelectField("Role")


class Task(Model):
    class Meta:
        base_id = os.environ['AIRTABLE_BASE_ID']
        table_name = "Tasks"
        api_key = os.environ['AIRTABLE_API_KEY']
        typecast = True

    title = fields.RequiredTextField("Title")
    description = fields.TextField("Description")
    status = fields.SelectField("Status")
    priority = fields.IntegerField("Priority")
    due_date = fields.DateField("Due Date")
    assignee = fields.SingleLinkField("Assignee", User)
    attachments = fields.AttachmentsField("Attachments")
    created = fields.CreatedTimeField("Created")

    @classmethod
    def get_active_tasks(cls, user=None):
        """Get all active tasks, optionally filtered by user."""
        formula = cls.status != "Completed"
        if user:
            formula = AND(formula, cls.assignee == user)
        return cls.all(formula=formula, sort=['-priority', 'due_date'])

    @classmethod
    def get_overdue_tasks(cls):
        """Get tasks past due date."""
        from datetime import date
        return cls.all(
            formula=AND(
                cls.status != "Completed",
                cls.due_date < date.today().isoformat()
            )
        )
```
