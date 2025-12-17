# PWA and Offline Support Reference

Progressive Web App features and offline-first patterns for mobile web apps.

---

## PWA Setup with next-pwa

### Installation

```bash
npm install next-pwa
```

### Configuration

```javascript
// next.config.js
const withPWA = require('next-pwa')({
  dest: 'public',
  register: true,
  skipWaiting: true,
  disable: process.env.NODE_ENV === 'development',
  runtimeCaching: [
    {
      urlPattern: /^https:\/\/api\.airtable\.com\/.*/i,
      handler: 'NetworkFirst',
      options: {
        cacheName: 'airtable-cache',
        expiration: {
          maxEntries: 100,
          maxAgeSeconds: 60 * 60 * 24, // 24 hours
        },
        networkTimeoutSeconds: 10,
      },
    },
    {
      urlPattern: /^https:\/\/dl\.airtable\.com\/.*/i,
      handler: 'CacheFirst',
      options: {
        cacheName: 'airtable-attachments',
        expiration: {
          maxEntries: 50,
          maxAgeSeconds: 60 * 60 * 24 * 30, // 30 days
        },
      },
    },
    {
      urlPattern: /\.(?:png|jpg|jpeg|svg|gif|webp|ico)$/i,
      handler: 'CacheFirst',
      options: {
        cacheName: 'images',
        expiration: {
          maxEntries: 100,
          maxAgeSeconds: 60 * 60 * 24 * 30, // 30 days
        },
      },
    },
    {
      urlPattern: /\.(?:js|css)$/i,
      handler: 'StaleWhileRevalidate',
      options: {
        cacheName: 'static-resources',
      },
    },
  ],
});

module.exports = withPWA({
  // Next.js config
});
```

---

## Online Status Detection

### Hook Implementation

```typescript
// hooks/use-online-status.ts
'use client';

import { useState, useEffect } from 'react';

export function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(true);

  useEffect(() => {
    // Set initial state
    setIsOnline(navigator.onLine);

    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);

    window.addEventListener('online', handleOnline);
    window.addEventListener('offline', handleOffline);

    return () => {
      window.removeEventListener('online', handleOnline);
      window.removeEventListener('offline', handleOffline);
    };
  }, []);

  return isOnline;
}
```

### Offline Banner Component

```tsx
// components/offline-banner.tsx
'use client';

import { WifiOff } from 'lucide-react';
import { useOnlineStatus } from '@/hooks/use-online-status';

export function OfflineBanner() {
  const isOnline = useOnlineStatus();

  if (isOnline) return null;

  return (
    <div className="fixed top-0 left-0 right-0 z-50 bg-amber-500 text-amber-950 px-4 py-2 text-center text-sm font-medium safe-area-top">
      <WifiOff className="inline-block h-4 w-4 mr-2" />
      You're offline. Some features may be unavailable.
    </div>
  );
}
```

---

## Offline Data Persistence

### Local Storage Queue

```typescript
// lib/offline-queue.ts

interface QueuedOperation {
  id: string;
  type: 'create' | 'update' | 'delete';
  table: string;
  data: any;
  timestamp: number;
  retries: number;
}

const QUEUE_KEY = 'offline_operations_queue';

export const offlineQueue = {
  getAll(): QueuedOperation[] {
    if (typeof window === 'undefined') return [];
    const stored = localStorage.getItem(QUEUE_KEY);
    return stored ? JSON.parse(stored) : [];
  },

  add(operation: Omit<QueuedOperation, 'id' | 'timestamp' | 'retries'>): void {
    const queue = this.getAll();
    queue.push({
      ...operation,
      id: crypto.randomUUID(),
      timestamp: Date.now(),
      retries: 0,
    });
    localStorage.setItem(QUEUE_KEY, JSON.stringify(queue));
  },

  remove(id: string): void {
    const queue = this.getAll().filter((op) => op.id !== id);
    localStorage.setItem(QUEUE_KEY, JSON.stringify(queue));
  },

  incrementRetry(id: string): void {
    const queue = this.getAll().map((op) =>
      op.id === id ? { ...op, retries: op.retries + 1 } : op
    );
    localStorage.setItem(QUEUE_KEY, JSON.stringify(queue));
  },

  clear(): void {
    localStorage.removeItem(QUEUE_KEY);
  },

  count(): number {
    return this.getAll().length;
  },
};
```

### Sync Service

```typescript
// lib/sync-service.ts

import { offlineQueue } from './offline-queue';

const MAX_RETRIES = 3;

export async function syncOfflineChanges(): Promise<{
  synced: number;
  failed: number;
}> {
  const operations = offlineQueue.getAll();
  let synced = 0;
  let failed = 0;

  for (const operation of operations) {
    try {
      await processOperation(operation);
      offlineQueue.remove(operation.id);
      synced++;
    } catch (error) {
      console.error('Sync failed for operation:', operation.id, error);

      if (operation.retries >= MAX_RETRIES) {
        // Move to dead letter queue or notify user
        offlineQueue.remove(operation.id);
        failed++;
      } else {
        offlineQueue.incrementRetry(operation.id);
      }
    }
  }

  return { synced, failed };
}

async function processOperation(operation: QueuedOperation): Promise<void> {
  const { type, table, data } = operation;

  switch (type) {
    case 'create':
      await fetch(`/api/airtable/${table}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ records: [{ fields: data }], typecast: true }),
      });
      break;

    case 'update':
      await fetch(`/api/airtable/${table}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ records: [{ id: data.id, fields: data.fields }] }),
      });
      break;

    case 'delete':
      await fetch(`/api/airtable/${table}?records[]=${data.id}`, {
        method: 'DELETE',
      });
      break;
  }
}
```

### Auto-Sync Hook

```typescript
// hooks/use-auto-sync.ts
'use client';

import { useEffect } from 'react';
import { useOnlineStatus } from './use-online-status';
import { syncOfflineChanges } from '@/lib/sync-service';
import { offlineQueue } from '@/lib/offline-queue';
import { useToast } from '@/components/ui/use-toast';

export function useAutoSync() {
  const isOnline = useOnlineStatus();
  const { toast } = useToast();

  useEffect(() => {
    if (isOnline && offlineQueue.count() > 0) {
      syncOfflineChanges().then(({ synced, failed }) => {
        if (synced > 0) {
          toast({
            title: 'Synced!',
            description: `${synced} changes synced successfully.`,
          });
        }
        if (failed > 0) {
          toast({
            variant: 'destructive',
            title: 'Sync Error',
            description: `${failed} changes failed to sync.`,
          });
        }
      });
    }
  }, [isOnline, toast]);
}
```

---

## Optimistic Updates with Offline Support

### Custom Hook

```typescript
// hooks/use-offline-mutation.ts
'use client';

import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useOnlineStatus } from './use-online-status';
import { offlineQueue } from '@/lib/offline-queue';

interface UseOfflineMutationOptions<T> {
  table: string;
  queryKey: string[];
  onOptimisticUpdate: (data: T, previousData: any) => any;
}

export function useOfflineCreate<T>({
  table,
  queryKey,
  onOptimisticUpdate,
}: UseOfflineMutationOptions<T>) {
  const queryClient = useQueryClient();
  const isOnline = useOnlineStatus();

  return useMutation({
    mutationFn: async (data: T) => {
      if (!isOnline) {
        // Queue for later sync
        offlineQueue.add({
          type: 'create',
          table,
          data,
        });
        // Return fake record for optimistic update
        return {
          id: `temp_${Date.now()}`,
          fields: data,
          createdTime: new Date().toISOString(),
        };
      }

      const response = await fetch(`/api/airtable/${table}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ records: [{ fields: data }], typecast: true }),
      });
      const result = await response.json();
      return result.records[0];
    },
    onMutate: async (newData) => {
      await queryClient.cancelQueries({ queryKey });
      const previousData = queryClient.getQueryData(queryKey);
      queryClient.setQueryData(queryKey, (old: any) =>
        onOptimisticUpdate(newData, old)
      );
      return { previousData };
    },
    onError: (err, newData, context) => {
      queryClient.setQueryData(queryKey, context?.previousData);
    },
    onSettled: () => {
      if (isOnline) {
        queryClient.invalidateQueries({ queryKey });
      }
    },
  });
}
```

### Usage Example

```tsx
// components/task-form.tsx
import { useOfflineCreate } from '@/hooks/use-offline-mutation';

export function TaskForm() {
  const createTask = useOfflineCreate<TaskInput>({
    table: 'Tasks',
    queryKey: ['airtable', 'Tasks'],
    onOptimisticUpdate: (newTask, previousTasks) => {
      const tempRecord = {
        id: `temp_${Date.now()}`,
        fields: newTask,
        createdTime: new Date().toISOString(),
      };
      return [...(previousTasks || []), tempRecord];
    },
  });

  const onSubmit = (data: TaskInput) => {
    createTask.mutate(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* Form fields */}
    </form>
  );
}
```

---

## IndexedDB for Large Data

### Setup with Dexie

```bash
npm install dexie
```

### Database Schema

```typescript
// lib/db.ts
import Dexie, { Table } from 'dexie';

export interface CachedRecord {
  id: string;
  table: string;
  fields: Record<string, any>;
  createdTime: string;
  cachedAt: number;
}

export interface PendingSync {
  id: string;
  type: 'create' | 'update' | 'delete';
  table: string;
  data: any;
  timestamp: number;
  retries: number;
}

export class AppDatabase extends Dexie {
  records!: Table<CachedRecord>;
  pendingSync!: Table<PendingSync>;

  constructor() {
    super('MyAppDB');
    this.version(1).stores({
      records: 'id, table, cachedAt',
      pendingSync: 'id, table, timestamp',
    });
  }
}

export const db = new AppDatabase();
```

### IndexedDB Cache Layer

```typescript
// lib/cache-layer.ts
import { db, CachedRecord } from './db';

const CACHE_TTL = 60 * 60 * 1000; // 1 hour

export async function getCachedRecords(table: string): Promise<CachedRecord[]> {
  const cutoff = Date.now() - CACHE_TTL;
  return db.records
    .where('table')
    .equals(table)
    .and((record) => record.cachedAt > cutoff)
    .toArray();
}

export async function setCachedRecords(
  table: string,
  records: Array<{ id: string; fields: any; createdTime: string }>
): Promise<void> {
  const cachedAt = Date.now();
  const cachedRecords = records.map((record) => ({
    ...record,
    table,
    cachedAt,
  }));

  await db.transaction('rw', db.records, async () => {
    // Remove old records for this table
    await db.records.where('table').equals(table).delete();
    // Add new records
    await db.records.bulkAdd(cachedRecords);
  });
}

export async function invalidateCache(table: string): Promise<void> {
  await db.records.where('table').equals(table).delete();
}
```

### Hook with IndexedDB

```typescript
// hooks/use-cached-query.ts
'use client';

import { useQuery } from '@tanstack/react-query';
import { useOnlineStatus } from './use-online-status';
import { getCachedRecords, setCachedRecords } from '@/lib/cache-layer';

export function useCachedAirtableQuery<T>(table: string, options?: { view?: string }) {
  const isOnline = useOnlineStatus();

  return useQuery({
    queryKey: ['airtable', table, options],
    queryFn: async () => {
      // Try to fetch from network first
      if (isOnline) {
        try {
          const params = new URLSearchParams();
          if (options?.view) params.set('view', options.view);

          const response = await fetch(`/api/airtable/${table}?${params}`);
          const data = await response.json();

          // Cache the results
          await setCachedRecords(table, data.records);

          return data.records;
        } catch (error) {
          console.error('Network fetch failed, using cache:', error);
        }
      }

      // Fall back to cache
      const cached = await getCachedRecords(table);
      if (cached.length > 0) {
        return cached;
      }

      throw new Error('No data available offline');
    },
    staleTime: 5 * 60 * 1000, // 5 minutes
    gcTime: 24 * 60 * 60 * 1000, // 24 hours
  });
}
```

---

## Background Sync API

### Service Worker Registration

```typescript
// lib/register-sw.ts
export async function registerBackgroundSync(): Promise<void> {
  if ('serviceWorker' in navigator && 'SyncManager' in window) {
    try {
      const registration = await navigator.serviceWorker.ready;
      await registration.sync.register('sync-airtable');
      console.log('Background sync registered');
    } catch (error) {
      console.error('Background sync registration failed:', error);
    }
  }
}
```

### Service Worker Sync Handler

```javascript
// public/sw-custom.js (merged with next-pwa)
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-airtable') {
    event.waitUntil(syncOfflineData());
  }
});

async function syncOfflineData() {
  // Get pending operations from IndexedDB or localStorage
  // Process each operation
  // Remove successful operations from queue
}
```

---

## Pending Changes Indicator

```tsx
// components/pending-changes.tsx
'use client';

import { useState, useEffect } from 'react';
import { Cloud, CloudOff, RefreshCw } from 'lucide-react';
import { offlineQueue } from '@/lib/offline-queue';
import { useOnlineStatus } from '@/hooks/use-online-status';
import { syncOfflineChanges } from '@/lib/sync-service';
import { Button } from '@/components/ui/button';

export function PendingChangesIndicator() {
  const [pendingCount, setPendingCount] = useState(0);
  const [isSyncing, setIsSyncing] = useState(false);
  const isOnline = useOnlineStatus();

  useEffect(() => {
    const updateCount = () => setPendingCount(offlineQueue.count());
    updateCount();

    // Poll for changes (or use custom events)
    const interval = setInterval(updateCount, 5000);
    return () => clearInterval(interval);
  }, []);

  const handleSync = async () => {
    setIsSyncing(true);
    try {
      await syncOfflineChanges();
      setPendingCount(offlineQueue.count());
    } finally {
      setIsSyncing(false);
    }
  };

  if (pendingCount === 0) {
    return (
      <div className="flex items-center gap-2 text-sm text-muted-foreground">
        <Cloud className="h-4 w-4" />
        <span>All changes saved</span>
      </div>
    );
  }

  return (
    <div className="flex items-center gap-2">
      <div className="flex items-center gap-2 text-sm text-amber-600">
        <CloudOff className="h-4 w-4" />
        <span>{pendingCount} pending</span>
      </div>
      {isOnline && (
        <Button
          variant="ghost"
          size="sm"
          onClick={handleSync}
          disabled={isSyncing}
        >
          <RefreshCw className={cn('h-4 w-4', isSyncing && 'animate-spin')} />
        </Button>
      )}
    </div>
  );
}
```

---

## Testing Offline Behavior

### Chrome DevTools

1. Open DevTools (F12)
2. Go to "Network" tab
3. Check "Offline" checkbox
4. Test app behavior

### Programmatic Testing

```typescript
// tests/offline.test.ts
describe('Offline Support', () => {
  beforeEach(() => {
    // Mock offline
    Object.defineProperty(navigator, 'onLine', {
      value: false,
      writable: true,
    });
  });

  it('queues operations when offline', async () => {
    // Test your offline queue
  });

  it('syncs when back online', async () => {
    // Mock coming back online
    Object.defineProperty(navigator, 'onLine', {
      value: true,
      writable: true,
    });
    window.dispatchEvent(new Event('online'));

    // Test sync behavior
  });
});
```
