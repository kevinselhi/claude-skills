---
name: mobile-web-app
description: Build mobile-first, responsive web applications using TypeScript, React/Next.js with Airtable as a backend. Use when creating PWA-capable web apps optimized for mobile devices with touch interactions, offline support, and native-like UX.
license: Complete terms in LICENSE.txt
---

# Mobile-First Web App Development Guide

## Overview

Build production-quality, mobile-friendly web applications using modern TypeScript patterns. This skill covers responsive design, touch interactions, PWA features, offline support, and Airtable backend integration.

---

# Tech Stack Recommendations

## Recommended Stack

| Layer | Technology | Why |
|-------|------------|-----|
| Framework | **Next.js 14+** | App router, server components, API routes |
| Language | **TypeScript** | Type safety, better tooling |
| Styling | **Tailwind CSS** | Mobile-first utilities, rapid development |
| UI Components | **shadcn/ui** | Accessible, customizable components |
| State | **React Query** | Server state, caching, offline support |
| Forms | **React Hook Form** + **Zod** | Validation, performance |
| Backend | **Airtable API** | Via Next.js API routes |

## Alternative Options

- **Vite + React** - Lighter weight, faster dev server
- **Remix** - Nested routing, progressive enhancement
- **Expo Web** - If also targeting native apps

---

# Process

## Phase 1: Project Setup

### Initialize Next.js Project

```bash
npx create-next-app@latest my-app --typescript --tailwind --eslint --app --src-dir
cd my-app
```

### Install Dependencies

```bash
# Core
npm install @tanstack/react-query zod react-hook-form @hookform/resolvers

# UI Components (shadcn)
npx shadcn-ui@latest init
npx shadcn-ui@latest add button card input label toast sheet dialog

# Icons
npm install lucide-react

# Date handling
npm install date-fns

# PWA (optional)
npm install next-pwa
```

### Environment Setup

```bash
# .env.local
AIRTABLE_API_KEY=pat_xxxxx
AIRTABLE_BASE_ID=appxxxxx
```

---

## Phase 2: Mobile-First Architecture

### Responsive Breakpoints

```typescript
// tailwind.config.ts - Default breakpoints
// sm: 640px   - Large phones landscape
// md: 768px   - Tablets
// lg: 1024px  - Laptops
// xl: 1280px  - Desktops

// Design mobile-first: no prefix = mobile
// Add responsive prefixes for larger screens
<div className="p-4 md:p-6 lg:p-8">
  <h1 className="text-xl md:text-2xl lg:text-3xl">Title</h1>
</div>
```

### App Structure

```
src/
├── app/
│   ├── layout.tsx          # Root layout with providers
│   ├── page.tsx            # Home page
│   ├── (auth)/             # Auth route group
│   │   ├── login/
│   │   └── register/
│   ├── api/                # API routes (Airtable proxy)
│   │   └── airtable/
│   │       └── [table]/
│   │           └── route.ts
│   └── [feature]/          # Feature pages
│       └── page.tsx
├── components/
│   ├── ui/                 # shadcn components
│   ├── layout/             # Header, Footer, Nav
│   └── [feature]/          # Feature-specific components
├── hooks/                  # Custom React hooks
├── lib/                    # Utilities
│   ├── airtable.ts        # Airtable client
│   ├── utils.ts           # Helpers
│   └── validations.ts     # Zod schemas
└── types/                  # TypeScript types
```

See [Project Structure Reference](./reference/project-structure.md) for complete setup.

---

## Phase 3: Mobile UI Patterns

### Touch-Friendly Components

See [Mobile UI Reference](./reference/mobile-ui.md) for complete patterns.

**Key Principles:**

1. **Touch targets**: Minimum 44x44px (48x48px recommended)
2. **Spacing**: Generous padding between interactive elements
3. **Gestures**: Support swipe, pull-to-refresh, long-press
4. **Feedback**: Immediate visual feedback on touch

```tsx
// Touch-friendly button
<Button className="h-12 px-6 text-base">
  Submit
</Button>

// Touch-friendly list item
<div className="flex items-center min-h-[56px] px-4 py-3 active:bg-muted">
  <span className="flex-1">Item</span>
  <ChevronRight className="h-5 w-5 text-muted-foreground" />
</div>
```

### Mobile Navigation

```tsx
// Bottom navigation (mobile)
<nav className="fixed bottom-0 left-0 right-0 h-16 bg-background border-t md:hidden">
  <div className="flex h-full">
    <NavItem icon={Home} label="Home" href="/" />
    <NavItem icon={List} label="Tasks" href="/tasks" />
    <NavItem icon={Plus} label="Add" href="/add" />
    <NavItem icon={User} label="Profile" href="/profile" />
  </div>
</nav>

// Side navigation (tablet+)
<aside className="hidden md:flex md:w-64 md:flex-col">
  {/* Desktop sidebar */}
</aside>
```

### Sheet/Drawer Pattern

```tsx
import { Sheet, SheetContent, SheetHeader, SheetTitle, SheetTrigger } from "@/components/ui/sheet";

<Sheet>
  <SheetTrigger asChild>
    <Button variant="outline" size="icon">
      <Menu className="h-5 w-5" />
    </Button>
  </SheetTrigger>
  <SheetContent side="left" className="w-[300px]">
    <SheetHeader>
      <SheetTitle>Menu</SheetTitle>
    </SheetHeader>
    {/* Menu items */}
  </SheetContent>
</Sheet>
```

---

## Phase 4: Airtable Integration

### API Route Pattern

```typescript
// app/api/airtable/[table]/route.ts
import { NextRequest, NextResponse } from 'next/server';

const API_KEY = process.env.AIRTABLE_API_KEY!;
const BASE_ID = process.env.AIRTABLE_BASE_ID!;

export async function GET(
  request: NextRequest,
  { params }: { params: { table: string } }
) {
  const { searchParams } = new URL(request.url);
  const view = searchParams.get('view');
  const formula = searchParams.get('filterByFormula');

  const url = new URL(`https://api.airtable.com/v0/${BASE_ID}/${params.table}`);
  if (view) url.searchParams.set('view', view);
  if (formula) url.searchParams.set('filterByFormula', formula);

  const response = await fetch(url, {
    headers: { 'Authorization': `Bearer ${API_KEY}` },
    next: { revalidate: 60 }
  });

  return NextResponse.json(await response.json());
}

export async function POST(
  request: NextRequest,
  { params }: { params: { table: string } }
) {
  const body = await request.json();

  const response = await fetch(
    `https://api.airtable.com/v0/${BASE_ID}/${params.table}`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${API_KEY}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify(body),
    }
  );

  return NextResponse.json(await response.json());
}
```

### React Query Hooks

```typescript
// hooks/use-airtable.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

interface AirtableRecord<T> {
  id: string;
  fields: T;
  createdTime: string;
}

export function useAirtableQuery<T>(
  table: string,
  options?: { view?: string; formula?: string }
) {
  return useQuery({
    queryKey: ['airtable', table, options],
    queryFn: async () => {
      const params = new URLSearchParams();
      if (options?.view) params.set('view', options.view);
      if (options?.formula) params.set('filterByFormula', options.formula);

      const res = await fetch(`/api/airtable/${table}?${params}`);
      const data = await res.json();
      return data.records as AirtableRecord<T>[];
    },
  });
}

export function useAirtableCreate<T>(table: string) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (fields: Partial<T>) => {
      const res = await fetch(`/api/airtable/${table}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ records: [{ fields }], typecast: true }),
      });
      return res.json();
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['airtable', table] });
    },
  });
}
```

---

## Phase 5: Forms and Validation

### Zod Schemas

```typescript
// lib/validations.ts
import { z } from 'zod';

export const taskSchema = z.object({
  name: z.string().min(1, 'Name is required').max(100),
  description: z.string().optional(),
  dueDate: z.string().optional(),
  priority: z.enum(['Low', 'Medium', 'High']).default('Medium'),
  status: z.enum(['Todo', 'In Progress', 'Done']).default('Todo'),
});

export type TaskInput = z.infer<typeof taskSchema>;
```

### Mobile-Optimized Form

```tsx
// components/task-form.tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';

export function TaskForm({ onSubmit }: { onSubmit: (data: TaskInput) => void }) {
  const form = useForm<TaskInput>({
    resolver: zodResolver(taskSchema),
    defaultValues: { priority: 'Medium', status: 'Todo' },
  });

  return (
    <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
      {/* Large touch targets */}
      <div className="space-y-2">
        <Label htmlFor="name">Task Name</Label>
        <Input
          id="name"
          className="h-12 text-base"  {/* Larger for mobile */}
          {...form.register('name')}
        />
        {form.formState.errors.name && (
          <p className="text-sm text-destructive">
            {form.formState.errors.name.message}
          </p>
        )}
      </div>

      {/* Native date picker on mobile */}
      <div className="space-y-2">
        <Label htmlFor="dueDate">Due Date</Label>
        <Input
          id="dueDate"
          type="date"
          className="h-12 text-base"
          {...form.register('dueDate')}
        />
      </div>

      {/* Large select for touch */}
      <div className="space-y-2">
        <Label>Priority</Label>
        <select
          className="w-full h-12 px-3 rounded-md border text-base"
          {...form.register('priority')}
        >
          <option value="Low">Low</option>
          <option value="Medium">Medium</option>
          <option value="High">High</option>
        </select>
      </div>

      <Button type="submit" className="w-full h-12 text-base">
        Save Task
      </Button>
    </form>
  );
}
```

---

## Phase 6: PWA Features

### next.config.js PWA Setup

```javascript
// next.config.js
const withPWA = require('next-pwa')({
  dest: 'public',
  register: true,
  skipWaiting: true,
  disable: process.env.NODE_ENV === 'development',
});

module.exports = withPWA({
  // Next.js config
});
```

### Manifest

```json
// public/manifest.json
{
  "name": "Home Maintenance App",
  "short_name": "HomeMaint",
  "description": "Track home maintenance tasks",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#000000",
  "icons": [
    { "src": "/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

### Offline Support

See [PWA Reference](./reference/pwa-offline.md) for complete offline patterns.

---

## Phase 7: Performance Optimization

### Loading States

```tsx
// Skeleton loading for mobile
function TaskListSkeleton() {
  return (
    <div className="space-y-3">
      {[...Array(5)].map((_, i) => (
        <div key={i} className="h-20 rounded-lg bg-muted animate-pulse" />
      ))}
    </div>
  );
}

// Usage with Suspense
<Suspense fallback={<TaskListSkeleton />}>
  <TaskList />
</Suspense>
```

### Image Optimization

```tsx
import Image from 'next/image';

<Image
  src="/photo.jpg"
  alt="Description"
  width={400}
  height={300}
  className="rounded-lg"
  placeholder="blur"
  blurDataURL="data:image/jpeg;base64,..."
  sizes="(max-width: 768px) 100vw, 50vw"
/>
```

### Code Splitting

```tsx
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('@/components/chart'), {
  loading: () => <ChartSkeleton />,
  ssr: false,
});
```

---

# Reference Files

## Documentation Library

- [Project Structure Reference](./reference/project-structure.md) - Complete project setup
- [Mobile UI Reference](./reference/mobile-ui.md) - Touch-friendly components
- [PWA Reference](./reference/pwa-offline.md) - Offline support patterns
- [Performance Reference](./reference/performance.md) - Optimization techniques

---

# Best Practices

## Mobile UX

1. **Thumb zone** - Place primary actions in easy reach
2. **One-handed use** - Design for single-hand operation
3. **Reduce typing** - Use selects, toggles, and defaults
4. **Fast feedback** - Optimistic updates, loading states
5. **Offline-first** - Assume intermittent connectivity

## Performance

1. **Lazy load** - Only load visible content
2. **Cache data** - React Query handles this well
3. **Optimize images** - Use Next.js Image component
4. **Minimize JS** - Tree shake, code split
5. **Prefetch** - Anticipate navigation

## Accessibility

1. **Focus visible** - Clear focus indicators
2. **Touch targets** - Minimum 44x44px
3. **Color contrast** - WCAG AA minimum
4. **Screen readers** - Semantic HTML, ARIA labels
5. **Motion reduced** - Respect prefers-reduced-motion
