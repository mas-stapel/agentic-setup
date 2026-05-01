---
name: react
description: >
  React and Next.js expertise. Provides deep knowledge of React 18+ patterns,
  Next.js App Router and Pages Router, component architecture, state management,
  performance optimization, testing with Testing Library, and accessibility.
  Activate this skill when working on React or Next.js frontend code, reviewing
  React components, planning frontend architecture, or writing component tests.
---

# React & Next.js — Shared Skill

Opinionated, production-tested patterns for React 18+ and Next.js — component architecture, hooks, state management, performance, testing, and accessibility.

---

## Core React Knowledge

### Component Patterns

**Always use functional components with TypeScript.**

```tsx
// ✅ Standard component with typed props
interface UserCardProps {
  user: User;
  onEdit: (userId: string) => void;
  className?: string;
}

const UserCard: React.FC<UserCardProps> = ({ user, onEdit, className }) => {
  return (
    <div className={className}>
      <h3>{user.name}</h3>
      <button onClick={() => onEdit(user.id)}>Edit</button>
    </div>
  );
};

export default UserCard;
```

**Component composition over prop drilling:**

```tsx
// ✅ DO: Compose with children and render props
interface LayoutProps {
  children: React.ReactNode;
  sidebar?: React.ReactNode;
}

const Layout: React.FC<LayoutProps> = ({ children, sidebar }) => (
  <div className="layout">
    {sidebar && <aside>{sidebar}</aside>}
    <main>{children}</main>
  </div>
);

// ❌ DON'T: Pass data through 5 levels of components
// Use Context or state management instead
```

**Controlled vs. uncontrolled components:**

```tsx
// ✅ Controlled: Use when you need to react to every change
const SearchInput: React.FC<{ onSearch: (q: string) => void }> = ({ onSearch }) => {
  const [query, setQuery] = useState('');

  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setQuery(e.target.value);
    onSearch(e.target.value);
  };

  return <input value={query} onChange={handleChange} />;
};

// ✅ Uncontrolled: Use for simple forms where you only need the value on submit
const LoginForm: React.FC = () => {
  const emailRef = useRef<HTMLInputElement>(null);

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    const email = emailRef.current?.value;
    // submit
  };

  return (
    <form onSubmit={handleSubmit}>
      <input ref={emailRef} type="email" />
      <button type="submit">Login</button>
    </form>
  );
};
```

### Hooks Best Practices

**useState** — Keep state minimal and derived values computed:

```tsx
// ✅ DO: Derive values instead of storing redundant state
const [items, setItems] = useState<Item[]>([]);
const totalPrice = items.reduce((sum, item) => sum + item.price, 0); // Derived
const isEmpty = items.length === 0; // Derived

// ❌ DON'T: Store what can be derived
const [items, setItems] = useState<Item[]>([]);
const [totalPrice, setTotalPrice] = useState(0); // Redundant!
const [isEmpty, setIsEmpty] = useState(true); // Redundant!
```

**useEffect** — Follow the rules strictly:

```tsx
// ✅ DO: Clean up subscriptions and timers
useEffect(() => {
  const controller = new AbortController();

  const fetchData = async () => {
    try {
      const res = await fetch('/api/users', { signal: controller.signal });
      const data = await res.json();
      setUsers(data);
    } catch (err) {
      if (err instanceof DOMException && err.name === 'AbortError') return;
      setError('Failed to fetch users');
    }
  };

  fetchData();
  return () => controller.abort();
}, []);

// ❌ DON'T: Forget cleanup or use useEffect for derived state
useEffect(() => {
  setFullName(`${firstName} ${lastName}`); // This should be a useMemo or just a const
}, [firstName, lastName]);
```

**useCallback and useMemo** — Use sparingly and intentionally:

```tsx
// ✅ DO: Memoize when passing callbacks to memoized children
const handleDelete = useCallback((id: string) => {
  setItems(prev => prev.filter(item => item.id !== id));
}, []);

// ✅ DO: Memoize expensive computations
const sortedItems = useMemo(
  () => [...items].sort((a, b) => a.name.localeCompare(b.name)),
  [items],
);

// ❌ DON'T: Memoize everything "just in case"
const name = useMemo(() => `${first} ${last}`, [first, last]); // Overkill
```

**Custom hooks** — Extract reusable logic:

```tsx
// ✅ DO: Create custom hooks for reusable stateful logic
function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Usage
const debouncedSearch = useDebounce(searchQuery, 300);
```

### State Management

**Use the right tool for the scope:**

| Scope | Solution |
|-------|----------|
| Component-local | `useState` |
| Shared between siblings | Lift state to parent |
| Cross-cutting (auth, theme) | React Context + `useReducer` |
| Complex client state | Zustand or Jotai (prefer over Redux for new projects) |
| Server state | React Query / TanStack Query |

```tsx
// ✅ Context for cross-cutting concerns
interface AuthContextType {
  user: User | null;
  login: (credentials: Credentials) => Promise<void>;
  logout: () => void;
  isLoading: boolean;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export function useAuth(): AuthContextType {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
}
```

### Error Boundaries

```tsx
// ✅ Always wrap route-level components with error boundaries
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }: FallbackProps) {
  return (
    <div role="alert">
      <h2>Something went wrong</h2>
      <pre>{error.message}</pre>
      <button onClick={resetErrorBoundary}>Try again</button>
    </div>
  );
}

// Usage in layout or page
<ErrorBoundary FallbackComponent={ErrorFallback}>
  <UserDashboard />
</ErrorBoundary>
```

---

## Next.js Patterns

### App Router (Preferred)

**Server Components are the default.** Only add `'use client'` when you need interactivity, hooks, or browser APIs.

```tsx
// ✅ Server Component (default) — fetches data on the server
// app/users/page.tsx
import { prisma } from '@/lib/prisma';

export default async function UsersPage() {
  const users = await prisma.user.findMany();

  return (
    <div>
      <h1>Users</h1>
      <UserList users={users} />
    </div>
  );
}

// ✅ Client Component — only when you need interactivity
// app/users/UserList.tsx
'use client';

import { useState } from 'react';

interface UserListProps {
  users: User[];
}

export default function UserList({ users }: UserListProps) {
  const [search, setSearch] = useState('');
  const filtered = users.filter(u => u.name.includes(search));

  return (
    <>
      <input value={search} onChange={e => setSearch(e.target.value)} />
      {filtered.map(user => <UserCard key={user.id} user={user} />)}
    </>
  );
}
```

**Route organization:**

```
app/
├── layout.tsx              # Root layout (wraps all pages)
├── page.tsx                # Home page (/)
├── loading.tsx             # Loading UI for this segment
├── error.tsx               # Error UI for this segment
├── not-found.tsx           # 404 UI
├── users/
│   ├── page.tsx            # /users
│   ├── [id]/
│   │   ├── page.tsx        # /users/:id
│   │   └── edit/
│   │       └── page.tsx    # /users/:id/edit
│   └── layout.tsx          # Shared layout for /users/*
└── api/
    └── users/
        ├── route.ts        # GET /api/users, POST /api/users
        └── [id]/
            └── route.ts    # GET/PUT/DELETE /api/users/:id
```

**Data fetching patterns:**

```tsx
// ✅ Server Component with fetch + caching
async function getUser(id: string): Promise<User> {
  const res = await fetch(`${process.env.API_URL}/users/${id}`, {
    next: { revalidate: 60 }, // ISR: revalidate every 60 seconds
  });

  if (!res.ok) throw new Error('Failed to fetch user');
  return res.json();
}

// ✅ Server Actions for mutations
'use server';

import { revalidatePath } from 'next/cache';

export async function updateUser(id: string, data: UpdateUserInput) {
  await prisma.user.update({ where: { id }, data });
  revalidatePath(`/users/${id}`);
}
```

**Loading and error states:**

```tsx
// app/users/loading.tsx — automatic loading UI
export default function Loading() {
  return <UserListSkeleton />;
}

// app/users/error.tsx — automatic error UI
'use client';

export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string };
  reset: () => void;
}) {
  return (
    <div>
      <h2>Something went wrong</h2>
      <button onClick={reset}>Try again</button>
    </div>
  );
}
```

### Pages Router (Legacy Support)

Use when the project already uses Pages Router or when specific features aren't yet available in App Router.

```tsx
// pages/users/[id].tsx
import { GetServerSideProps } from 'next';

interface UserPageProps {
  user: User;
}

export const getServerSideProps: GetServerSideProps<UserPageProps> = async ({ params }) => {
  const user = await prisma.user.findUnique({ where: { id: params?.id as string } });

  if (!user) return { notFound: true };
  return { props: { user } };
};

export default function UserPage({ user }: UserPageProps) {
  return <UserProfile user={user} />;
}
```

---

## Performance Optimization

### Rendering Performance

1. **Memoize expensive child components:**

```tsx
const MemoizedUserCard = React.memo(UserCard);

// Only re-renders when user or onEdit actually change
<MemoizedUserCard user={user} onEdit={handleEdit} />
```

2. **Virtualize long lists** (use `@tanstack/react-virtual` or `react-window`):

```tsx
// For lists > 100 items, always virtualize
import { useVirtualizer } from '@tanstack/react-virtual';
```

3. **Code-split with dynamic imports:**

```tsx
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('./HeavyChart'), {
  loading: () => <ChartSkeleton />,
  ssr: false, // Skip SSR for client-only components
});
```

4. **Optimize images:**

```tsx
import Image from 'next/image';

// ✅ Always use next/image for automatic optimization
<Image
  src={user.avatar}
  alt={`${user.name}'s avatar`}
  width={48}
  height={48}
  className="rounded-full"
/>
```

### Bundle Size

- Use **barrel exports** sparingly — they can prevent tree-shaking.
- Import specific functions: `import { debounce } from 'lodash-es'` not `import _ from 'lodash'`.
- Analyze bundle with `@next/bundle-analyzer` when performance is a concern.

---

## Testing React Components

Use `@testing-library/react` with Jest. **Test behavior, not implementation.**

```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('LoginForm', () => {
  it('should show validation error for invalid email', async () => {
    const user = userEvent.setup();
    render(<LoginForm onSubmit={jest.fn()} />);

    await user.type(screen.getByLabelText(/email/i), 'not-an-email');
    await user.click(screen.getByRole('button', { name: /submit/i }));

    expect(screen.getByText(/invalid email/i)).toBeInTheDocument();
  });

  it('should call onSubmit with form data on valid submission', async () => {
    const handleSubmit = jest.fn();
    const user = userEvent.setup();
    render(<LoginForm onSubmit={handleSubmit} />);

    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/password/i), 'securePass123');
    await user.click(screen.getByRole('button', { name: /submit/i }));

    await waitFor(() => {
      expect(handleSubmit).toHaveBeenCalledWith({
        email: 'test@example.com',
        password: 'securePass123',
      });
    });
  });

  it('should disable submit button while loading', () => {
    render(<LoginForm onSubmit={jest.fn()} isLoading />);

    expect(screen.getByRole('button', { name: /submit/i })).toBeDisabled();
  });
});
```

**Testing hooks:**

```tsx
import { renderHook, act } from '@testing-library/react';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('should increment the count', () => {
    const { result } = renderHook(() => useCounter());

    act(() => {
      result.current.increment();
    });

    expect(result.current.count).toBe(1);
  });
});
```

**Testing with providers (Context, Router, etc.):**

```tsx
// test-utils.tsx — shared wrapper for tests
import { render, RenderOptions } from '@testing-library/react';
import { AuthProvider } from '@/contexts/AuthContext';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

function AllProviders({ children }: { children: React.ReactNode }) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });

  return (
    <QueryClientProvider client={queryClient}>
      <AuthProvider>
        {children}
      </AuthProvider>
    </QueryClientProvider>
  );
}

export function renderWithProviders(
  ui: React.ReactElement,
  options?: Omit<RenderOptions, 'wrapper'>,
) {
  return render(ui, { wrapper: AllProviders, ...options });
}
```

---

## Accessibility Checklist

When reviewing or implementing React components, ensure:

- [ ] All interactive elements are focusable and have visible focus indicators
- [ ] Images have meaningful `alt` text (or `alt=""` for decorative)
- [ ] Forms have associated labels (`<label htmlFor>` or `aria-label`)
- [ ] Dynamic content changes are announced (`aria-live`, `role="alert"`)
- [ ] Modals trap focus and restore on close
- [ ] Color is not the only way to convey information
- [ ] Touch targets are at least 44x44px
- [ ] Page has proper heading hierarchy (`h1` → `h2` → `h3`)
- [ ] Keyboard navigation works (Tab, Enter, Escape, Arrow keys)

---

## Common Anti-Patterns to Avoid

| Anti-Pattern | Why It's Bad | Do This Instead |
|-------------|-------------|-----------------|
| `useEffect` for derived state | Causes extra renders | `useMemo` or compute inline |
| Putting everything in global state | Over-coupling, poor performance | Use local state when possible |
| `any` types for props | Defeats TypeScript's purpose | Define proper interfaces |
| Inline object/array literals as props | Creates new reference every render, breaks memoization | `useMemo` or define outside component |
| Fetching in `useEffect` without cleanup | Race conditions, memory leaks | Use AbortController or React Query |
| Index as key in dynamic lists | Causes rendering bugs on reorder/delete | Use unique, stable IDs |
| Direct DOM manipulation | Conflicts with React's reconciliation | Use refs and React APIs |
| Large monolithic components | Hard to test, hard to reuse | Extract into smaller components |
