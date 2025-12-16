# Project Structure Reference

Complete Next.js project setup for mobile-first web apps with Airtable backend.

---

## Directory Structure

```
my-app/
├── .env.local                 # Environment variables (gitignored)
├── .gitignore
├── next.config.js             # Next.js configuration
├── tailwind.config.ts         # Tailwind configuration
├── tsconfig.json              # TypeScript configuration
├── package.json
├── public/
│   ├── manifest.json          # PWA manifest
│   ├── icon-192.png           # App icon
│   ├── icon-512.png           # App icon large
│   └── sw.js                  # Service worker (generated)
└── src/
    ├── app/
    │   ├── layout.tsx         # Root layout
    │   ├── page.tsx           # Home page
    │   ├── globals.css        # Global styles
    │   ├── loading.tsx        # Loading UI
    │   ├── error.tsx          # Error UI
    │   ├── not-found.tsx      # 404 page
    │   ├── api/               # API routes
    │   │   └── airtable/
    │   │       └── [table]/
    │   │           └── route.ts
    │   └── [feature]/         # Feature routes
    │       ├── page.tsx
    │       └── [id]/
    │           └── page.tsx
    ├── components/
    │   ├── ui/                # shadcn/ui components
    │   ├── layout/            # Layout components
    │   │   ├── header.tsx
    │   │   ├── bottom-nav.tsx
    │   │   └── sidebar.tsx
    │   ├── forms/             # Form components
    │   └── [feature]/         # Feature components
    ├── hooks/                 # Custom hooks
    │   ├── use-airtable.ts
    │   ├── use-media-query.ts
    │   └── use-online-status.ts
    ├── lib/                   # Utilities
    │   ├── airtable.ts        # Airtable helpers
    │   ├── utils.ts           # General utilities
    │   └── validations.ts     # Zod schemas
    ├── providers/             # React context providers
    │   ├── query-provider.tsx
    │   └── theme-provider.tsx
    └── types/                 # TypeScript types
        ├── airtable.ts
        └── [feature].ts
```

---

## Configuration Files

### package.json

```json
{
  "name": "my-app",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit"
  },
  "dependencies": {
    "next": "^14.0.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "@tanstack/react-query": "^5.0.0",
    "zod": "^3.22.0",
    "react-hook-form": "^7.48.0",
    "@hookform/resolvers": "^3.3.0",
    "lucide-react": "^0.294.0",
    "date-fns": "^2.30.0",
    "class-variance-authority": "^0.7.0",
    "clsx": "^2.0.0",
    "tailwind-merge": "^2.0.0",
    "@radix-ui/react-dialog": "^1.0.0",
    "@radix-ui/react-slot": "^1.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0",
    "@types/react": "^18.2.0",
    "@types/react-dom": "^18.2.0",
    "tailwindcss": "^3.4.0",
    "postcss": "^8.4.0",
    "autoprefixer": "^10.4.0",
    "eslint": "^8.0.0",
    "eslint-config-next": "^14.0.0"
  }
}
```

### next.config.js

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  // Strict mode for better debugging
  reactStrictMode: true,

  // Image optimization
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'dl.airtable.com', // Airtable attachments
      },
    ],
  },

  // Headers for PWA
  async headers() {
    return [
      {
        source: '/sw.js',
        headers: [
          {
            key: 'Cache-Control',
            value: 'no-cache, no-store, must-revalidate',
          },
          {
            key: 'Content-Type',
            value: 'application/javascript; charset=utf-8',
          },
        ],
      },
    ];
  },
};

module.exports = nextConfig;
```

### tailwind.config.ts

```typescript
import type { Config } from 'tailwindcss';

const config: Config = {
  darkMode: ['class'],
  content: [
    './src/pages/**/*.{js,ts,jsx,tsx,mdx}',
    './src/components/**/*.{js,ts,jsx,tsx,mdx}',
    './src/app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    container: {
      center: true,
      padding: '1rem',
      screens: {
        '2xl': '1400px',
      },
    },
    extend: {
      colors: {
        border: 'hsl(var(--border))',
        input: 'hsl(var(--input))',
        ring: 'hsl(var(--ring))',
        background: 'hsl(var(--background))',
        foreground: 'hsl(var(--foreground))',
        primary: {
          DEFAULT: 'hsl(var(--primary))',
          foreground: 'hsl(var(--primary-foreground))',
        },
        secondary: {
          DEFAULT: 'hsl(var(--secondary))',
          foreground: 'hsl(var(--secondary-foreground))',
        },
        destructive: {
          DEFAULT: 'hsl(var(--destructive))',
          foreground: 'hsl(var(--destructive-foreground))',
        },
        muted: {
          DEFAULT: 'hsl(var(--muted))',
          foreground: 'hsl(var(--muted-foreground))',
        },
        accent: {
          DEFAULT: 'hsl(var(--accent))',
          foreground: 'hsl(var(--accent-foreground))',
        },
        popover: {
          DEFAULT: 'hsl(var(--popover))',
          foreground: 'hsl(var(--popover-foreground))',
        },
        card: {
          DEFAULT: 'hsl(var(--card))',
          foreground: 'hsl(var(--card-foreground))',
        },
      },
      borderRadius: {
        lg: 'var(--radius)',
        md: 'calc(var(--radius) - 2px)',
        sm: 'calc(var(--radius) - 4px)',
      },
      keyframes: {
        'accordion-down': {
          from: { height: '0' },
          to: { height: 'var(--radix-accordion-content-height)' },
        },
        'accordion-up': {
          from: { height: 'var(--radix-accordion-content-height)' },
          to: { height: '0' },
        },
      },
      animation: {
        'accordion-down': 'accordion-down 0.2s ease-out',
        'accordion-up': 'accordion-up 0.2s ease-out',
      },
    },
  },
  plugins: [require('tailwindcss-animate')],
};

export default config;
```

### tsconfig.json

```json
{
  "compilerOptions": {
    "target": "es5",
    "lib": ["dom", "dom.iterable", "esnext"],
    "allowJs": true,
    "skipLibCheck": true,
    "strict": true,
    "noEmit": true,
    "esModuleInterop": true,
    "module": "esnext",
    "moduleResolution": "bundler",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "jsx": "preserve",
    "incremental": true,
    "plugins": [
      {
        "name": "next"
      }
    ],
    "paths": {
      "@/*": ["./src/*"]
    }
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts"],
  "exclude": ["node_modules"]
}
```

---

## Core Files

### src/app/layout.tsx

```tsx
import type { Metadata, Viewport } from 'next';
import { Inter } from 'next/font/google';
import './globals.css';
import { Providers } from '@/providers';
import { BottomNav } from '@/components/layout/bottom-nav';
import { Toaster } from '@/components/ui/toaster';

const inter = Inter({ subsets: ['latin'] });

export const metadata: Metadata = {
  title: 'My App',
  description: 'Mobile-friendly web app',
  manifest: '/manifest.json',
  appleWebApp: {
    capable: true,
    statusBarStyle: 'default',
    title: 'My App',
  },
};

export const viewport: Viewport = {
  width: 'device-width',
  initialScale: 1,
  maximumScale: 1,
  userScalable: false,
  themeColor: '#ffffff',
};

export default function RootLayout({
  children,
}: {
  children: React.ReactNode;
}) {
  return (
    <html lang="en" suppressHydrationWarning>
      <head>
        <link rel="apple-touch-icon" href="/icon-192.png" />
      </head>
      <body className={inter.className}>
        <Providers>
          <div className="min-h-screen bg-background">
            <main className="pb-16 md:pb-0">{children}</main>
            <BottomNav />
          </div>
          <Toaster />
        </Providers>
      </body>
    </html>
  );
}
```

### src/app/globals.css

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;
    --primary: 222.2 47.4% 11.2%;
    --primary-foreground: 210 40% 98%;
    --secondary: 210 40% 96.1%;
    --secondary-foreground: 222.2 47.4% 11.2%;
    --muted: 210 40% 96.1%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96.1%;
    --accent-foreground: 222.2 47.4% 11.2%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 222.2 84% 4.9%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;
    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;
    --primary: 210 40% 98%;
    --primary-foreground: 222.2 47.4% 11.2%;
    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 40% 98%;
    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 212.7 26.8% 83.9%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
    -webkit-tap-highlight-color: transparent;
  }
}

/* Safe area insets for mobile */
.safe-area-top {
  padding-top: env(safe-area-inset-top);
}

.safe-area-bottom {
  padding-bottom: env(safe-area-inset-bottom);
}

/* Prevent pull-to-refresh on mobile */
html {
  overscroll-behavior: none;
}

/* Smooth scrolling */
html {
  scroll-behavior: smooth;
}

/* Better touch targets */
button, a {
  touch-action: manipulation;
}

/* Hide scrollbar but keep functionality */
.hide-scrollbar {
  -ms-overflow-style: none;
  scrollbar-width: none;
}

.hide-scrollbar::-webkit-scrollbar {
  display: none;
}
```

### src/providers/index.tsx

```tsx
'use client';

import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useState } from 'react';
import { ThemeProvider } from './theme-provider';

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(
    () =>
      new QueryClient({
        defaultOptions: {
          queries: {
            staleTime: 60 * 1000, // 1 minute
            refetchOnWindowFocus: false,
          },
        },
      })
  );

  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider
        attribute="class"
        defaultTheme="system"
        enableSystem
        disableTransitionOnChange
      >
        {children}
      </ThemeProvider>
    </QueryClientProvider>
  );
}
```

### src/lib/utils.ts

```typescript
import { type ClassValue, clsx } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

export function formatDate(date: string | Date, format: string = 'short') {
  const d = new Date(date);
  if (format === 'short') {
    return d.toLocaleDateString();
  }
  return d.toLocaleDateString('en-US', {
    weekday: 'short',
    month: 'short',
    day: 'numeric',
  });
}

export function debounce<T extends (...args: any[]) => any>(
  func: T,
  wait: number
): (...args: Parameters<T>) => void {
  let timeout: NodeJS.Timeout;
  return (...args: Parameters<T>) => {
    clearTimeout(timeout);
    timeout = setTimeout(() => func(...args), wait);
  };
}
```

---

## API Route Template

### src/app/api/airtable/[table]/route.ts

```typescript
import { NextRequest, NextResponse } from 'next/server';

const API_KEY = process.env.AIRTABLE_API_KEY!;
const BASE_ID = process.env.AIRTABLE_BASE_ID!;
const BASE_URL = `https://api.airtable.com/v0/${BASE_ID}`;

async function airtableFetch(
  endpoint: string,
  options: RequestInit = {}
): Promise<Response> {
  return fetch(`${BASE_URL}${endpoint}`, {
    ...options,
    headers: {
      Authorization: `Bearer ${API_KEY}`,
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });
}

export async function GET(
  request: NextRequest,
  { params }: { params: { table: string } }
) {
  try {
    const { searchParams } = new URL(request.url);
    const url = new URL(`${BASE_URL}/${encodeURIComponent(params.table)}`);

    // Forward supported query params
    const supportedParams = ['view', 'filterByFormula', 'sort', 'maxRecords', 'offset'];
    supportedParams.forEach((param) => {
      const value = searchParams.get(param);
      if (value) url.searchParams.set(param, value);
    });

    // Handle fields[] array
    searchParams.getAll('fields[]').forEach((field) => {
      url.searchParams.append('fields[]', field);
    });

    const response = await fetch(url.toString(), {
      headers: { Authorization: `Bearer ${API_KEY}` },
      next: { revalidate: 60 }, // Cache for 60 seconds
    });

    const data = await response.json();
    return NextResponse.json(data, { status: response.status });
  } catch (error) {
    console.error('Airtable GET error:', error);
    return NextResponse.json(
      { error: 'Failed to fetch data' },
      { status: 500 }
    );
  }
}

export async function POST(
  request: NextRequest,
  { params }: { params: { table: string } }
) {
  try {
    const body = await request.json();

    const response = await airtableFetch(`/${encodeURIComponent(params.table)}`, {
      method: 'POST',
      body: JSON.stringify(body),
    });

    const data = await response.json();
    return NextResponse.json(data, { status: response.status });
  } catch (error) {
    console.error('Airtable POST error:', error);
    return NextResponse.json(
      { error: 'Failed to create record' },
      { status: 500 }
    );
  }
}

export async function PATCH(
  request: NextRequest,
  { params }: { params: { table: string } }
) {
  try {
    const body = await request.json();

    const response = await airtableFetch(`/${encodeURIComponent(params.table)}`, {
      method: 'PATCH',
      body: JSON.stringify(body),
    });

    const data = await response.json();
    return NextResponse.json(data, { status: response.status });
  } catch (error) {
    console.error('Airtable PATCH error:', error);
    return NextResponse.json(
      { error: 'Failed to update record' },
      { status: 500 }
    );
  }
}

export async function DELETE(request: NextRequest) {
  try {
    const { searchParams } = new URL(request.url);
    const recordIds = searchParams.getAll('records[]');

    if (recordIds.length === 0) {
      return NextResponse.json(
        { error: 'No record IDs provided' },
        { status: 400 }
      );
    }

    const deleteParams = recordIds.map((id) => `records[]=${id}`).join('&');
    const response = await airtableFetch(`/Tasks?${deleteParams}`, {
      method: 'DELETE',
    });

    const data = await response.json();
    return NextResponse.json(data, { status: response.status });
  } catch (error) {
    console.error('Airtable DELETE error:', error);
    return NextResponse.json(
      { error: 'Failed to delete record' },
      { status: 500 }
    );
  }
}
```

---

## PWA Manifest

### public/manifest.json

```json
{
  "name": "My App",
  "short_name": "MyApp",
  "description": "Mobile-friendly web application",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#000000",
  "orientation": "portrait-primary",
  "icons": [
    {
      "src": "/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icon-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "any maskable"
    }
  ]
}
```
