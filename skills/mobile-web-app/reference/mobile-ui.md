# Mobile UI Patterns Reference

Touch-friendly, mobile-optimized UI components and patterns for React/Next.js.

---

## Touch Target Sizes

| Element | Minimum | Recommended |
|---------|---------|-------------|
| Buttons | 44x44px | 48x48px |
| Icons (tappable) | 44x44px | 48x48px |
| List items | 44px height | 56px height |
| Form inputs | 44px height | 48px height |
| Spacing between targets | 8px | 12px |

---

## Button Components

### Primary Button

```tsx
import { Button } from "@/components/ui/button";

// Mobile-optimized primary button
<Button className="w-full h-12 text-base font-medium">
  Save Changes
</Button>

// With loading state
<Button className="w-full h-12" disabled={isLoading}>
  {isLoading ? (
    <>
      <Loader2 className="mr-2 h-4 w-4 animate-spin" />
      Saving...
    </>
  ) : (
    'Save Changes'
  )}
</Button>
```

### Icon Button

```tsx
// Minimum 44x44px touch target
<Button variant="ghost" size="icon" className="h-11 w-11">
  <Plus className="h-5 w-5" />
  <span className="sr-only">Add item</span>
</Button>
```

### Floating Action Button (FAB)

```tsx
function FloatingActionButton() {
  return (
    <Button
      className="fixed bottom-20 right-4 h-14 w-14 rounded-full shadow-lg md:bottom-8"
      size="icon"
    >
      <Plus className="h-6 w-6" />
      <span className="sr-only">Add new</span>
    </Button>
  );
}
```

---

## Navigation Patterns

### Bottom Navigation

```tsx
'use client';

import Link from 'next/link';
import { usePathname } from 'next/navigation';
import { Home, List, Plus, User } from 'lucide-react';
import { cn } from '@/lib/utils';

interface NavItemProps {
  href: string;
  icon: React.ComponentType<{ className?: string }>;
  label: string;
}

function NavItem({ href, icon: Icon, label }: NavItemProps) {
  const pathname = usePathname();
  const isActive = pathname === href;

  return (
    <Link
      href={href}
      className={cn(
        'flex flex-1 flex-col items-center justify-center gap-1 py-2',
        'transition-colors active:bg-muted',
        isActive ? 'text-primary' : 'text-muted-foreground'
      )}
    >
      <Icon className="h-5 w-5" />
      <span className="text-xs">{label}</span>
    </Link>
  );
}

export function BottomNav() {
  return (
    <nav className="fixed bottom-0 left-0 right-0 z-50 bg-background border-t safe-area-bottom md:hidden">
      <div className="flex h-16">
        <NavItem href="/" icon={Home} label="Home" />
        <NavItem href="/tasks" icon={List} label="Tasks" />
        <NavItem href="/add" icon={Plus} label="Add" />
        <NavItem href="/profile" icon={User} label="Profile" />
      </div>
    </nav>
  );
}
```

### Safe Area Handling

```css
/* globals.css */
.safe-area-bottom {
  padding-bottom: env(safe-area-inset-bottom);
}

.safe-area-top {
  padding-top: env(safe-area-inset-top);
}

/* Content padding for bottom nav */
.content-with-bottom-nav {
  padding-bottom: calc(4rem + env(safe-area-inset-bottom));
}
```

### Header with Back Button

```tsx
'use client';

import { useRouter } from 'next/navigation';
import { ChevronLeft, MoreVertical } from 'lucide-react';
import { Button } from '@/components/ui/button';

interface MobileHeaderProps {
  title: string;
  showBack?: boolean;
  actions?: React.ReactNode;
}

export function MobileHeader({ title, showBack = true, actions }: MobileHeaderProps) {
  const router = useRouter();

  return (
    <header className="sticky top-0 z-40 flex h-14 items-center gap-2 border-b bg-background px-2 safe-area-top">
      {showBack && (
        <Button
          variant="ghost"
          size="icon"
          className="h-11 w-11"
          onClick={() => router.back()}
        >
          <ChevronLeft className="h-6 w-6" />
          <span className="sr-only">Go back</span>
        </Button>
      )}
      <h1 className="flex-1 truncate text-lg font-semibold">{title}</h1>
      {actions}
    </header>
  );
}
```

---

## List Components

### Touchable List Item

```tsx
import { ChevronRight } from 'lucide-react';
import Link from 'next/link';
import { cn } from '@/lib/utils';

interface ListItemProps {
  href?: string;
  onClick?: () => void;
  children: React.ReactNode;
  showArrow?: boolean;
  className?: string;
}

export function ListItem({
  href,
  onClick,
  children,
  showArrow = true,
  className,
}: ListItemProps) {
  const content = (
    <>
      <div className="flex-1 min-w-0">{children}</div>
      {showArrow && (
        <ChevronRight className="h-5 w-5 flex-shrink-0 text-muted-foreground" />
      )}
    </>
  );

  const baseStyles = cn(
    'flex items-center min-h-[56px] px-4 py-3',
    'transition-colors active:bg-muted',
    className
  );

  if (href) {
    return (
      <Link href={href} className={baseStyles}>
        {content}
      </Link>
    );
  }

  return (
    <button onClick={onClick} className={cn(baseStyles, 'w-full text-left')}>
      {content}
    </button>
  );
}
```

### Swipeable List Item

```tsx
'use client';

import { useState, useRef } from 'react';
import { Trash2, Edit } from 'lucide-react';

interface SwipeableItemProps {
  children: React.ReactNode;
  onDelete?: () => void;
  onEdit?: () => void;
}

export function SwipeableItem({ children, onDelete, onEdit }: SwipeableItemProps) {
  const [translateX, setTranslateX] = useState(0);
  const startX = useRef(0);
  const currentX = useRef(0);

  const handleTouchStart = (e: React.TouchEvent) => {
    startX.current = e.touches[0].clientX;
  };

  const handleTouchMove = (e: React.TouchEvent) => {
    currentX.current = e.touches[0].clientX;
    const diff = currentX.current - startX.current;
    // Only allow swiping left
    if (diff < 0) {
      setTranslateX(Math.max(diff, -160)); // Max reveal 160px
    }
  };

  const handleTouchEnd = () => {
    // Snap to revealed or hidden
    if (translateX < -80) {
      setTranslateX(-160);
    } else {
      setTranslateX(0);
    }
  };

  return (
    <div className="relative overflow-hidden">
      {/* Action buttons (revealed on swipe) */}
      <div className="absolute right-0 top-0 bottom-0 flex">
        {onEdit && (
          <button
            onClick={onEdit}
            className="flex w-20 items-center justify-center bg-blue-500 text-white"
          >
            <Edit className="h-5 w-5" />
          </button>
        )}
        {onDelete && (
          <button
            onClick={onDelete}
            className="flex w-20 items-center justify-center bg-red-500 text-white"
          >
            <Trash2 className="h-5 w-5" />
          </button>
        )}
      </div>

      {/* Main content */}
      <div
        className="relative bg-background transition-transform"
        style={{ transform: `translateX(${translateX}px)` }}
        onTouchStart={handleTouchStart}
        onTouchMove={handleTouchMove}
        onTouchEnd={handleTouchEnd}
      >
        {children}
      </div>
    </div>
  );
}
```

---

## Pull to Refresh

```tsx
'use client';

import { useState, useRef } from 'react';
import { RefreshCw } from 'lucide-react';

interface PullToRefreshProps {
  onRefresh: () => Promise<void>;
  children: React.ReactNode;
}

export function PullToRefresh({ onRefresh, children }: PullToRefreshProps) {
  const [pullDistance, setPullDistance] = useState(0);
  const [isRefreshing, setIsRefreshing] = useState(false);
  const startY = useRef(0);
  const containerRef = useRef<HTMLDivElement>(null);

  const THRESHOLD = 80;

  const handleTouchStart = (e: React.TouchEvent) => {
    if (containerRef.current?.scrollTop === 0) {
      startY.current = e.touches[0].clientY;
    }
  };

  const handleTouchMove = (e: React.TouchEvent) => {
    if (startY.current === 0 || isRefreshing) return;

    const currentY = e.touches[0].clientY;
    const diff = currentY - startY.current;

    if (diff > 0 && containerRef.current?.scrollTop === 0) {
      // Resistance effect
      setPullDistance(Math.min(diff * 0.4, THRESHOLD * 1.5));
    }
  };

  const handleTouchEnd = async () => {
    if (pullDistance >= THRESHOLD && !isRefreshing) {
      setIsRefreshing(true);
      setPullDistance(THRESHOLD);

      try {
        await onRefresh();
      } finally {
        setIsRefreshing(false);
        setPullDistance(0);
      }
    } else {
      setPullDistance(0);
    }

    startY.current = 0;
  };

  return (
    <div
      ref={containerRef}
      className="h-full overflow-y-auto"
      onTouchStart={handleTouchStart}
      onTouchMove={handleTouchMove}
      onTouchEnd={handleTouchEnd}
    >
      {/* Pull indicator */}
      <div
        className="flex items-center justify-center overflow-hidden transition-all"
        style={{ height: pullDistance }}
      >
        <RefreshCw
          className={cn(
            'h-6 w-6 text-muted-foreground transition-transform',
            isRefreshing && 'animate-spin',
            pullDistance >= THRESHOLD && 'text-primary'
          )}
          style={{
            transform: `rotate(${(pullDistance / THRESHOLD) * 180}deg)`,
          }}
        />
      </div>

      {children}
    </div>
  );
}
```

---

## Form Components

### Mobile Input

```tsx
import { forwardRef } from 'react';
import { cn } from '@/lib/utils';

interface MobileInputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label?: string;
  error?: string;
}

export const MobileInput = forwardRef<HTMLInputElement, MobileInputProps>(
  ({ label, error, className, id, ...props }, ref) => {
    return (
      <div className="space-y-2">
        {label && (
          <label htmlFor={id} className="text-sm font-medium">
            {label}
          </label>
        )}
        <input
          ref={ref}
          id={id}
          className={cn(
            'flex h-12 w-full rounded-md border border-input bg-background px-4 py-3',
            'text-base ring-offset-background',
            'placeholder:text-muted-foreground',
            'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring',
            'disabled:cursor-not-allowed disabled:opacity-50',
            error && 'border-destructive focus-visible:ring-destructive',
            className
          )}
          {...props}
        />
        {error && (
          <p className="text-sm text-destructive">{error}</p>
        )}
      </div>
    );
  }
);

MobileInput.displayName = 'MobileInput';
```

### Mobile Select

```tsx
import { forwardRef } from 'react';
import { ChevronDown } from 'lucide-react';
import { cn } from '@/lib/utils';

interface Option {
  value: string;
  label: string;
}

interface MobileSelectProps extends React.SelectHTMLAttributes<HTMLSelectElement> {
  label?: string;
  options: Option[];
  error?: string;
}

export const MobileSelect = forwardRef<HTMLSelectElement, MobileSelectProps>(
  ({ label, options, error, className, id, ...props }, ref) => {
    return (
      <div className="space-y-2">
        {label && (
          <label htmlFor={id} className="text-sm font-medium">
            {label}
          </label>
        )}
        <div className="relative">
          <select
            ref={ref}
            id={id}
            className={cn(
              'flex h-12 w-full appearance-none rounded-md border border-input',
              'bg-background px-4 py-3 pr-10 text-base',
              'focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring',
              'disabled:cursor-not-allowed disabled:opacity-50',
              error && 'border-destructive',
              className
            )}
            {...props}
          >
            {options.map((option) => (
              <option key={option.value} value={option.value}>
                {option.label}
              </option>
            ))}
          </select>
          <ChevronDown className="pointer-events-none absolute right-3 top-1/2 h-5 w-5 -translate-y-1/2 text-muted-foreground" />
        </div>
        {error && (
          <p className="text-sm text-destructive">{error}</p>
        )}
      </div>
    );
  }
);

MobileSelect.displayName = 'MobileSelect';
```

---

## Loading States

### Skeleton Components

```tsx
import { cn } from '@/lib/utils';

function Skeleton({ className, ...props }: React.HTMLAttributes<HTMLDivElement>) {
  return (
    <div
      className={cn('animate-pulse rounded-md bg-muted', className)}
      {...props}
    />
  );
}

// List skeleton
export function ListSkeleton({ count = 5 }: { count?: number }) {
  return (
    <div className="space-y-3">
      {Array.from({ length: count }).map((_, i) => (
        <div key={i} className="flex items-center gap-4 p-4">
          <Skeleton className="h-10 w-10 rounded-full" />
          <div className="flex-1 space-y-2">
            <Skeleton className="h-4 w-3/4" />
            <Skeleton className="h-3 w-1/2" />
          </div>
        </div>
      ))}
    </div>
  );
}

// Card skeleton
export function CardSkeleton() {
  return (
    <div className="rounded-lg border p-4 space-y-3">
      <Skeleton className="h-5 w-1/2" />
      <Skeleton className="h-4 w-full" />
      <Skeleton className="h-4 w-3/4" />
    </div>
  );
}
```

### Loading Overlay

```tsx
import { Loader2 } from 'lucide-react';

export function LoadingOverlay() {
  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center bg-background/80 backdrop-blur-sm">
      <Loader2 className="h-8 w-8 animate-spin text-primary" />
    </div>
  );
}
```

---

## Empty States

```tsx
import { FileQuestion } from 'lucide-react';
import { Button } from '@/components/ui/button';

interface EmptyStateProps {
  icon?: React.ComponentType<{ className?: string }>;
  title: string;
  description?: string;
  action?: {
    label: string;
    onClick: () => void;
  };
}

export function EmptyState({
  icon: Icon = FileQuestion,
  title,
  description,
  action,
}: EmptyStateProps) {
  return (
    <div className="flex flex-col items-center justify-center py-12 px-4 text-center">
      <div className="rounded-full bg-muted p-4 mb-4">
        <Icon className="h-8 w-8 text-muted-foreground" />
      </div>
      <h3 className="text-lg font-semibold">{title}</h3>
      {description && (
        <p className="mt-1 text-sm text-muted-foreground max-w-xs">
          {description}
        </p>
      )}
      {action && (
        <Button onClick={action.onClick} className="mt-4">
          {action.label}
        </Button>
      )}
    </div>
  );
}
```

---

## Toast Notifications

```tsx
'use client';

import { Toaster } from '@/components/ui/toaster';
import { useToast } from '@/components/ui/use-toast';

// In layout
<Toaster />

// Usage
const { toast } = useToast();

// Success
toast({
  title: 'Saved!',
  description: 'Your changes have been saved.',
});

// Error
toast({
  variant: 'destructive',
  title: 'Error',
  description: 'Something went wrong. Please try again.',
});

// With action
toast({
  title: 'Task deleted',
  description: 'The task has been removed.',
  action: (
    <ToastAction altText="Undo" onClick={handleUndo}>
      Undo
    </ToastAction>
  ),
});
```

---

## Modal/Sheet Patterns

### Action Sheet (iOS-style)

```tsx
import {
  Sheet,
  SheetContent,
  SheetHeader,
  SheetTitle,
} from '@/components/ui/sheet';

interface ActionSheetProps {
  open: boolean;
  onOpenChange: (open: boolean) => void;
  title: string;
  actions: Array<{
    label: string;
    onClick: () => void;
    variant?: 'default' | 'destructive';
  }>;
}

export function ActionSheet({ open, onOpenChange, title, actions }: ActionSheetProps) {
  return (
    <Sheet open={open} onOpenChange={onOpenChange}>
      <SheetContent side="bottom" className="rounded-t-xl">
        <SheetHeader className="text-center">
          <SheetTitle>{title}</SheetTitle>
        </SheetHeader>
        <div className="mt-4 space-y-2">
          {actions.map((action, i) => (
            <button
              key={i}
              onClick={() => {
                action.onClick();
                onOpenChange(false);
              }}
              className={cn(
                'w-full py-4 text-base font-medium rounded-lg',
                'transition-colors active:bg-muted',
                action.variant === 'destructive'
                  ? 'text-destructive'
                  : 'text-foreground'
              )}
            >
              {action.label}
            </button>
          ))}
        </div>
        <button
          onClick={() => onOpenChange(false)}
          className="w-full mt-2 py-4 text-base font-medium text-muted-foreground"
        >
          Cancel
        </button>
      </SheetContent>
    </Sheet>
  );
}
```
