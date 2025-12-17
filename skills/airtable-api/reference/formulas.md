# Airtable Formulas Reference

Complete guide to Airtable formula syntax for filtering records.

---

## Basic Syntax

### Field References

```
{Field Name}           // Reference a field by name
{Field Name With Spaces}  // Spaces are fine
```

### Strings

```
"text"                 // Double quotes
'text'                 // Single quotes
```

### Numbers

```
123                    // Integer
123.45                 // Decimal
```

---

## Comparison Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `=` | Equals | `{Status}='Active'` |
| `!=` | Not equals | `{Status}!='Deleted'` |
| `>` | Greater than | `{Age}>30` |
| `<` | Less than | `{Priority}<5` |
| `>=` | Greater or equal | `{Count}>=10` |
| `<=` | Less or equal | `{Score}<=100` |

---

## Logical Operators

### AND

All conditions must be true:

```
AND({Status}='Active', {Priority}>3)
AND({A}='x', {B}='y', {C}='z')  // Multiple conditions
```

### OR

At least one condition must be true:

```
OR({Status}='Active', {Status}='Pending')
OR({A}='x', {B}='y')
```

### NOT

Negates a condition:

```
NOT({Status}='Deleted')
NOT(AND({A}='x', {B}='y'))
```

### Combining

```
AND(
  OR({Status}='Active', {Status}='Pending'),
  {Priority}>3,
  NOT({Archived})
)
```

---

## Text Functions

### FIND

Find substring position (case-sensitive):

```
FIND('search', {Field})>0       // Contains 'search'
FIND('john', LOWER({Email}))>0  // Case-insensitive search
```

### SEARCH

Similar to FIND but case-insensitive:

```
SEARCH('text', {Field})>0
```

### LOWER / UPPER

```
LOWER({Name})='john doe'
UPPER({Code})='ABC'
```

### LEN

String length:

```
LEN({Description})>100
LEN({Code})=6
```

### LEFT / RIGHT / MID

```
LEFT({Phone}, 3)='555'
RIGHT({Code}, 2)='US'
MID({Serial}, 4, 3)='ABC'
```

### TRIM

Remove whitespace:

```
TRIM({Input})!=""
```

### SUBSTITUTE

Replace text:

```
SUBSTITUTE({Text}, 'old', 'new')
```

### CONCATENATE / &

Join strings:

```
CONCATENATE({First}, ' ', {Last})
{First}&' '&{Last}
```

### REGEX_MATCH

Regular expression matching:

```
REGEX_MATCH({Email}, '.+@.+\\..+')
REGEX_MATCH({Phone}, '^\\d{3}-\\d{3}-\\d{4}$')
```

---

## Numeric Functions

### Basic Math

```
{Total}+{Tax}
{Price}*{Quantity}
{Total}/{Count}
{Value}-{Discount}
```

### ABS

Absolute value:

```
ABS({Difference})>10
```

### CEILING / FLOOR / ROUND

```
CEILING({Value})
FLOOR({Value})
ROUND({Value}, 2)  // 2 decimal places
```

### MIN / MAX

```
MAX({A}, {B}, {C})
MIN({Score1}, {Score2})
```

### SUM / AVERAGE

Only in formula fields, not filter formulas.

### MOD

Modulo (remainder):

```
MOD({Number}, 2)=0  // Even numbers
```

### POWER

```
POWER({Base}, 2)  // Square
```

---

## Date Functions

### TODAY / NOW

```
{Due Date}=TODAY()
{Created}<NOW()
```

### Date Comparisons

```
IS_AFTER({Due Date}, TODAY())
IS_BEFORE({Created}, '2024-01-01')
IS_SAME({Date}, TODAY(), 'day')
```

### DATEADD

```
DATEADD({Start Date}, 7, 'days')
DATEADD(TODAY(), -30, 'days')  // 30 days ago
```

### DATETIME_DIFF

```
DATETIME_DIFF({End}, {Start}, 'days')>7
DATETIME_DIFF(NOW(), {Created}, 'hours')<24
```

### Date Parts

```
YEAR({Date})=2024
MONTH({Date})=12
DAY({Date})=25
WEEKDAY({Date})=1  // Sunday=0, Monday=1, etc.
HOUR({DateTime})>=9
```

### DATETIME_FORMAT

```
DATETIME_FORMAT({Date}, 'YYYY-MM-DD')
```

### DATETIME_PARSE

```
DATETIME_PARSE('2024-01-15', 'YYYY-MM-DD')
```

---

## Empty/Blank Checks

### Check if Empty

```
{Field}=''              // Empty text
{Field}=BLANK()         // Any empty field
NOT({Field})            // Falsy (empty, 0, false)
LEN({Field})=0          // Empty text
```

### Check if Not Empty

```
{Field}!=''
NOT({Field}='')
{Field}!=BLANK()
LEN({Field})>0
```

### For Linked Records

```
{Linked Field}!=''      // Has any linked records
LEN({Linked Field})=0   // No linked records
```

---

## Checkbox/Boolean

```
{Active}                 // Is checked
{Active}=TRUE()
{Active}=1

NOT({Archived})          // Is not checked
{Archived}=FALSE()
{Archived}=0
```

---

## Linked Records

### Has Any Links

```
{Linked Records}!=''
{Assignee}!=BLANK()
```

### Specific Link Count (via Rollup/Count field)

```
{Task Count}>0
{Related Items Count}>=5
```

---

## Select Fields

### Single Select

```
{Status}='Active'
OR({Status}='Active', {Status}='Pending')
{Category}!=''          // Has a selection
```

### Multiple Select

Use FIND to check for values:

```
FIND('Tag1', {Tags}&'')>0
FIND('urgent', LOWER({Tags}&''))>0
```

---

## Attachments

```
{Attachments}!=''       // Has attachments
{Attachments}=''        // No attachments
```

---

## IF Function

```
IF({Priority}>3, 'High', 'Normal')
IF({Status}='Active', TRUE(), FALSE())
IF(
  {Score}>=90, 'A',
  IF({Score}>=80, 'B',
    IF({Score}>=70, 'C', 'F')
  )
)
```

---

## SWITCH Function

```
SWITCH(
  {Status},
  'Active', 1,
  'Pending', 2,
  'Completed', 3,
  0  // Default
)
```

---

## Common Filter Patterns

### Active Records

```python
# pyairtable
table.all(formula="{Status}='Active'")

# Python formula builder
from pyairtable.formulas import match
table.all(formula=match({'Status': 'Active'}))
```

### Records from Last 7 Days

```python
table.all(formula="IS_AFTER({Created}, DATEADD(TODAY(), -7, 'days'))")
```

### Overdue Tasks

```python
table.all(formula="AND({Status}!='Completed', IS_BEFORE({Due Date}, TODAY()))")
```

### Search by Name

```python
search_term = "john"
formula = f"FIND('{search_term.lower()}', LOWER({{Name}}))>0"
table.all(formula=formula)
```

### My Assigned Tasks (with user email)

```python
user_email = "john@example.com"
formula = f"FIND('{user_email}', {{Assignee Email}})>0"
table.all(formula=formula)
```

### High Priority Incomplete

```python
formula = "AND({Priority}>=4, {Status}!='Done')"
table.all(formula=formula)
```

### Records with Attachments

```python
table.all(formula="{Attachments}!=''")
```

### This Week's Due

```python
formula = """
AND(
  IS_AFTER({Due Date}, DATEADD(TODAY(), -WEEKDAY(TODAY()), 'days')),
  IS_BEFORE({Due Date}, DATEADD(TODAY(), 7-WEEKDAY(TODAY()), 'days'))
)
"""
table.all(formula=formula)
```

---

## pyairtable Formula Builder

### Field Class

```python
from pyairtable.formulas import field, AND, OR, NOT

# Build formulas programmatically
f = field('Status')

# Comparisons
f == 'Active'           # {Status}='Active'
f != 'Deleted'          # {Status}!='Deleted'
f > 3                   # {Status}>3
f >= 10                 # {Status}>=10
f < 5                   # {Status}<5
f <= 100                # {Status}<=100
```

### Combining Conditions

```python
from pyairtable.formulas import AND, OR, NOT, field

formula = AND(
    field('Status') == 'Active',
    field('Priority') > 3,
    OR(
        field('Category') == 'Bug',
        field('Category') == 'Feature'
    )
)
# Result: AND({Status}='Active',{Priority}>3,OR({Category}='Bug',{Category}='Feature'))
```

### Match Helper

```python
from pyairtable.formulas import match

# Simple equality match
formula = match({'Status': 'Active', 'Priority': 5})
# Result: AND({Status}='Active',{Priority}=5)

# With match_any=True (OR instead of AND)
formula = match({'Status': 'Active', 'Status': 'Pending'}, match_any=True)
# Result: OR({Status}='Active',{Status}='Pending')
```

---

## TypeScript Formula Helpers

```typescript
// lib/airtable-formulas.ts

export function eq(field: string, value: string | number | boolean): string {
  if (typeof value === 'string') {
    return `{${field}}='${value.replace(/'/g, "\\'")}'`;
  }
  if (typeof value === 'boolean') {
    return `{${field}}=${value ? 'TRUE()' : 'FALSE()'}`;
  }
  return `{${field}}=${value}`;
}

export function neq(field: string, value: string | number): string {
  if (typeof value === 'string') {
    return `{${field}}!='${value.replace(/'/g, "\\'")}'`;
  }
  return `{${field}}!=${value}`;
}

export function gt(field: string, value: number): string {
  return `{${field}}>${value}`;
}

export function gte(field: string, value: number): string {
  return `{${field}}>=${value}`;
}

export function lt(field: string, value: number): string {
  return `{${field}}<${value}`;
}

export function lte(field: string, value: number): string {
  return `{${field}}<=${value}`;
}

export function and(...conditions: string[]): string {
  return `AND(${conditions.join(',')})`;
}

export function or(...conditions: string[]): string {
  return `OR(${conditions.join(',')})`;
}

export function not(condition: string): string {
  return `NOT(${condition})`;
}

export function isEmpty(field: string): string {
  return `{${field}}=''`;
}

export function isNotEmpty(field: string): string {
  return `{${field}}!=''`;
}

export function contains(field: string, value: string): string {
  return `FIND('${value.toLowerCase()}', LOWER({${field}}))>0`;
}

// Usage
const formula = and(
  eq('Status', 'Active'),
  gt('Priority', 3),
  or(
    eq('Category', 'Bug'),
    eq('Category', 'Feature')
  )
);
```
