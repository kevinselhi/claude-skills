# TypeScript Airtable API Reference

Complete patterns for integrating Airtable with TypeScript/JavaScript applications.

## Options

1. **Direct REST API** - Fetch-based, full control
2. **airtable npm package** - Official SDK (Node.js)
3. **Next.js API Routes** - Server-side proxy pattern

---

## Direct REST API

### Basic Setup

```typescript
// lib/airtable.ts

const AIRTABLE_API_KEY = process.env.AIRTABLE_API_KEY!;
const AIRTABLE_BASE_ID = process.env.AIRTABLE_BASE_ID!;

const BASE_URL = `https://api.airtable.com/v0/${AIRTABLE_BASE_ID}`;

interface AirtableRecord<T> {
  id: string;
  fields: T;
  createdTime: string;
}

interface AirtableResponse<T> {
  records: AirtableRecord<T>[];
  offset?: string;
}

async function airtableFetch<T>(
  endpoint: string,
  options: RequestInit = {}
): Promise<T> {
  const response = await fetch(`${BASE_URL}${endpoint}`, {
    ...options,
    headers: {
      'Authorization': `Bearer ${AIRTABLE_API_KEY}`,
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });

  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.error?.message || 'Airtable API error');
  }

  return response.json();
}
```

### CRUD Operations

```typescript
// Types for your table
interface Task {
  Name: string;
  Status: string;
  Priority: number;
  'Due Date': string;
  Assignee: string[];  // Linked record IDs
}

// CREATE
async function createTask(fields: Partial<Task>): Promise<AirtableRecord<Task>> {
  const response = await airtableFetch<{ records: AirtableRecord<Task>[] }>(
    '/Tasks',
    {
      method: 'POST',
      body: JSON.stringify({
        records: [{ fields }],
        typecast: true
      }),
    }
  );
  return response.records[0];
}

// READ ALL
async function getTasks(
  options: {
    filterByFormula?: string;
    sort?: { field: string; direction: 'asc' | 'desc' }[];
    view?: string;
    maxRecords?: number;
    fields?: string[];
  } = {}
): Promise<AirtableRecord<Task>[]> {
  const params = new URLSearchParams();

  if (options.filterByFormula) {
    params.append('filterByFormula', options.filterByFormula);
  }
  if (options.view) {
    params.append('view', options.view);
  }
  if (options.maxRecords) {
    params.append('maxRecords', options.maxRecords.toString());
  }
  if (options.fields) {
    options.fields.forEach(f => params.append('fields[]', f));
  }
  if (options.sort) {
    options.sort.forEach((s, i) => {
      params.append(`sort[${i}][field]`, s.field);
      params.append(`sort[${i}][direction]`, s.direction);
    });
  }

  const queryString = params.toString();
  const endpoint = `/Tasks${queryString ? `?${queryString}` : ''}`;

  const response = await airtableFetch<AirtableResponse<Task>>(endpoint);
  return response.records;
}

// READ ONE
async function getTask(recordId: string): Promise<AirtableRecord<Task>> {
  return airtableFetch<AirtableRecord<Task>>(`/Tasks/${recordId}`);
}

// UPDATE
async function updateTask(
  recordId: string,
  fields: Partial<Task>
): Promise<AirtableRecord<Task>> {
  return airtableFetch<AirtableRecord<Task>>(`/Tasks/${recordId}`, {
    method: 'PATCH',
    body: JSON.stringify({ fields }),
  });
}

// DELETE
async function deleteTask(recordId: string): Promise<{ id: string; deleted: boolean }> {
  return airtableFetch(`/Tasks/${recordId}`, {
    method: 'DELETE',
  });
}
```

### Pagination

```typescript
async function getAllTasks(): Promise<AirtableRecord<Task>[]> {
  const allRecords: AirtableRecord<Task>[] = [];
  let offset: string | undefined;

  do {
    const params = new URLSearchParams();
    if (offset) params.append('offset', offset);

    const response = await airtableFetch<AirtableResponse<Task>>(
      `/Tasks?${params.toString()}`
    );

    allRecords.push(...response.records);
    offset = response.offset;
  } while (offset);

  return allRecords;
}
```

### Batch Operations

```typescript
// Batch create (max 10 per request)
async function batchCreateTasks(
  records: Partial<Task>[]
): Promise<AirtableRecord<Task>[]> {
  const batches: Partial<Task>[][] = [];
  for (let i = 0; i < records.length; i += 10) {
    batches.push(records.slice(i, i + 10));
  }

  const results: AirtableRecord<Task>[] = [];
  for (const batch of batches) {
    const response = await airtableFetch<{ records: AirtableRecord<Task>[] }>(
      '/Tasks',
      {
        method: 'POST',
        body: JSON.stringify({
          records: batch.map(fields => ({ fields })),
          typecast: true
        }),
      }
    );
    results.push(...response.records);
  }

  return results;
}

// Batch update
async function batchUpdateTasks(
  updates: { id: string; fields: Partial<Task> }[]
): Promise<AirtableRecord<Task>[]> {
  const batches = [];
  for (let i = 0; i < updates.length; i += 10) {
    batches.push(updates.slice(i, i + 10));
  }

  const results: AirtableRecord<Task>[] = [];
  for (const batch of batches) {
    const response = await airtableFetch<{ records: AirtableRecord<Task>[] }>(
      '/Tasks',
      {
        method: 'PATCH',
        body: JSON.stringify({ records: batch }),
      }
    );
    results.push(...response.records);
  }

  return results;
}

// Batch delete
async function batchDeleteTasks(recordIds: string[]): Promise<void> {
  const batches = [];
  for (let i = 0; i < recordIds.length; i += 10) {
    batches.push(recordIds.slice(i, i + 10));
  }

  for (const batch of batches) {
    const params = batch.map(id => `records[]=${id}`).join('&');
    await airtableFetch(`/Tasks?${params}`, { method: 'DELETE' });
  }
}
```

---

## Using airtable npm Package

### Installation

```bash
npm install airtable
```

### Setup

```typescript
import Airtable from 'airtable';

const base = new Airtable({
  apiKey: process.env.AIRTABLE_API_KEY
}).base(process.env.AIRTABLE_BASE_ID!);

const tasksTable = base('Tasks');
```

### CRUD Operations

```typescript
import { FieldSet, Records } from 'airtable';

// CREATE
const created = await tasksTable.create([
  { fields: { Name: 'Task 1', Status: 'Todo' } },
  { fields: { Name: 'Task 2', Status: 'Todo' } }
]);

// READ
const records = await tasksTable
  .select({
    filterByFormula: "{Status}='Active'",
    sort: [{ field: 'Priority', direction: 'desc' }],
    view: 'Grid view',
    maxRecords: 100
  })
  .all();

// UPDATE
const updated = await tasksTable.update([
  { id: 'recXXX', fields: { Status: 'Done' } }
]);

// DELETE
const deleted = await tasksTable.destroy(['recXXX', 'recYYY']);
```

---

## Next.js API Routes Pattern

### Setup API Route

```typescript
// app/api/airtable/[table]/route.ts

import { NextRequest, NextResponse } from 'next/server';

const AIRTABLE_API_KEY = process.env.AIRTABLE_API_KEY!;
const AIRTABLE_BASE_ID = process.env.AIRTABLE_BASE_ID!;

export async function GET(
  request: NextRequest,
  { params }: { params: { table: string } }
) {
  const { searchParams } = new URL(request.url);

  const queryParams = new URLSearchParams();

  // Forward query parameters
  const view = searchParams.get('view');
  const formula = searchParams.get('filterByFormula');

  if (view) queryParams.append('view', view);
  if (formula) queryParams.append('filterByFormula', formula);

  const response = await fetch(
    `https://api.airtable.com/v0/${AIRTABLE_BASE_ID}/${params.table}?${queryParams}`,
    {
      headers: {
        'Authorization': `Bearer ${AIRTABLE_API_KEY}`,
      },
      next: { revalidate: 60 } // Cache for 60 seconds
    }
  );

  const data = await response.json();
  return NextResponse.json(data);
}

export async function POST(
  request: NextRequest,
  { params }: { params: { table: string } }
) {
  const body = await request.json();

  const response = await fetch(
    `https://api.airtable.com/v0/${AIRTABLE_BASE_ID}/${params.table}`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${AIRTABLE_API_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(body),
    }
  );

  const data = await response.json();
  return NextResponse.json(data);
}
```

### Client-Side Usage

```typescript
// hooks/useAirtable.ts

export function useAirtable<T>(table: string) {
  async function getRecords(options?: {
    view?: string;
    filterByFormula?: string;
  }): Promise<AirtableRecord<T>[]> {
    const params = new URLSearchParams();
    if (options?.view) params.append('view', options.view);
    if (options?.filterByFormula) {
      params.append('filterByFormula', options.filterByFormula);
    }

    const response = await fetch(`/api/airtable/${table}?${params}`);
    const data = await response.json();
    return data.records;
  }

  async function createRecord(fields: Partial<T>): Promise<AirtableRecord<T>> {
    const response = await fetch(`/api/airtable/${table}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ records: [{ fields }] }),
    });
    const data = await response.json();
    return data.records[0];
  }

  return { getRecords, createRecord };
}
```

---

## React Query Integration

```typescript
// hooks/useTasks.ts

import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

interface Task {
  Name: string;
  Status: string;
  Priority: number;
}

export function useTasks() {
  return useQuery({
    queryKey: ['tasks'],
    queryFn: async () => {
      const res = await fetch('/api/airtable/Tasks');
      const data = await res.json();
      return data.records as AirtableRecord<Task>[];
    },
  });
}

export function useCreateTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (fields: Partial<Task>) => {
      const res = await fetch('/api/airtable/Tasks', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ records: [{ fields }] }),
      });
      return res.json();
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['tasks'] });
    },
  });
}
```

---

## Type-Safe Helpers

```typescript
// lib/airtable-types.ts

// Generate types from your Airtable schema
export interface TaskFields {
  Name: string;
  Status: 'Todo' | 'In Progress' | 'Done';
  Priority: 1 | 2 | 3 | 4 | 5;
  'Due Date'?: string;
  Assignee?: string[];
  Notes?: string;
  Attachments?: Array<{
    id: string;
    url: string;
    filename: string;
    type: string;
  }>;
}

export interface UserFields {
  Name: string;
  Email: string;
  Role: 'Admin' | 'Member' | 'Viewer';
}

// Type-safe table access
export const tables = {
  tasks: createTableHelper<TaskFields>('Tasks'),
  users: createTableHelper<UserFields>('Users'),
};

function createTableHelper<T>(tableName: string) {
  return {
    async getAll() {
      return getTasks() as Promise<AirtableRecord<T>[]>;
    },
    async get(id: string) {
      return getTask(id) as Promise<AirtableRecord<T>>;
    },
    async create(fields: Partial<T>) {
      return createTask(fields as any) as Promise<AirtableRecord<T>>;
    },
    async update(id: string, fields: Partial<T>) {
      return updateTask(id, fields as any) as Promise<AirtableRecord<T>>;
    },
  };
}
```

---

## Error Handling

```typescript
class AirtableError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public airtableError?: any
  ) {
    super(message);
    this.name = 'AirtableError';
  }
}

async function airtableFetchWithRetry<T>(
  endpoint: string,
  options: RequestInit = {},
  retries = 3
): Promise<T> {
  for (let attempt = 0; attempt <= retries; attempt++) {
    try {
      const response = await fetch(`${BASE_URL}${endpoint}`, {
        ...options,
        headers: {
          'Authorization': `Bearer ${AIRTABLE_API_KEY}`,
          'Content-Type': 'application/json',
          ...options.headers,
        },
      });

      if (response.status === 429 && attempt < retries) {
        // Rate limited - wait and retry
        const waitTime = Math.pow(2, attempt) * 1000;
        await new Promise(resolve => setTimeout(resolve, waitTime));
        continue;
      }

      if (!response.ok) {
        const error = await response.json();
        throw new AirtableError(
          error.error?.message || 'Airtable API error',
          response.status,
          error
        );
      }

      return response.json();
    } catch (error) {
      if (attempt === retries) throw error;
    }
  }

  throw new Error('Max retries exceeded');
}
```

---

## Mobile-Optimized Patterns

### Optimistic Updates

```typescript
// For instant UI feedback on mobile
export function useOptimisticTasks() {
  const queryClient = useQueryClient();

  const updateTask = useMutation({
    mutationFn: async ({ id, fields }: { id: string; fields: Partial<Task> }) => {
      const res = await fetch(`/api/airtable/Tasks/${id}`, {
        method: 'PATCH',
        body: JSON.stringify({ fields }),
      });
      return res.json();
    },
    onMutate: async ({ id, fields }) => {
      // Cancel outgoing refetches
      await queryClient.cancelQueries({ queryKey: ['tasks'] });

      // Snapshot previous value
      const previousTasks = queryClient.getQueryData(['tasks']);

      // Optimistically update
      queryClient.setQueryData(['tasks'], (old: any) =>
        old?.map((task: any) =>
          task.id === id ? { ...task, fields: { ...task.fields, ...fields } } : task
        )
      );

      return { previousTasks };
    },
    onError: (err, variables, context) => {
      // Rollback on error
      queryClient.setQueryData(['tasks'], context?.previousTasks);
    },
  });

  return updateTask;
}
```

### Offline Support

```typescript
// Store pending changes locally
const PENDING_CHANGES_KEY = 'airtable_pending_changes';

interface PendingChange {
  id: string;
  type: 'create' | 'update' | 'delete';
  table: string;
  data: any;
  timestamp: number;
}

export function savePendingChange(change: Omit<PendingChange, 'id' | 'timestamp'>) {
  const pending = JSON.parse(localStorage.getItem(PENDING_CHANGES_KEY) || '[]');
  pending.push({
    ...change,
    id: crypto.randomUUID(),
    timestamp: Date.now(),
  });
  localStorage.setItem(PENDING_CHANGES_KEY, JSON.stringify(pending));
}

export async function syncPendingChanges() {
  const pending: PendingChange[] = JSON.parse(
    localStorage.getItem(PENDING_CHANGES_KEY) || '[]'
  );

  for (const change of pending) {
    try {
      // Process change...
      // Remove from pending on success
    } catch (error) {
      console.error('Failed to sync change:', change.id);
    }
  }
}
```
