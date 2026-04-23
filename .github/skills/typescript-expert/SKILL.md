---
name: typescript-expert
description: >
  TypeScript, Jest, Express, and project-structure standards for Node.js fullstack development.
  Provides language-level coding standards (strict mode, explicit types, no `any`), Express
  typed handler patterns, Jest testing conventions (AAA pattern, mocking strategy, coverage
  focus), file and folder organisation, and import ordering rules. Activate this skill when
  reviewing, planning, or implementing any TypeScript code in this stack.
---

# TypeScript Expert — Shared Skill

You are an expert in **TypeScript strict-mode development** within a Node.js fullstack stack. This skill provides deep, opinionated standards for writing production-quality TypeScript code across the backend (Express), testing (Jest), and general project conventions.

---

## Language Standards

### Strict Mode

`strict: true` in `tsconfig.json` is **non-negotiable**. Every project in this stack uses TypeScript strict mode, which enables:

- `strictNullChecks`
- `noImplicitAny`
- `strictFunctionTypes`
- `strictPropertyInitialization`

Never disable strict mode or individual strict flags to work around type errors. Fix the types instead.

### Explicit Types

```typescript
// ✅ DO: Use explicit types
interface UserResponse {
  id: string;
  email: string;
  createdAt: Date;
}

async function getUser(id: string): Promise<UserResponse> {
  // ...
}

// ❌ DON'T: Use implicit any or loose types
async function getUser(id) {
  // ...
}
```

### No `any`

Do not use `any` unless there is a genuine technical reason (e.g. a third-party library with no type definitions). When you must:

```typescript
// eslint-disable-next-line @typescript-eslint/no-explicit-any
const raw = externalLibrary.getData() as any; // external lib has no types — tracked in issue #123
```

Always add an inline comment explaining why `any` is unavoidable.

### Interfaces and Type Definitions

- Define an `interface` or `type` for every data structure that crosses a function or module boundary.
- Use `interface` for objects that may be extended; use `type` for unions, intersections, and primitives.
- Place shared domain types in `src/types/` and co-locate component/service-specific types with their file.

```typescript
// ✅ Shared domain type (src/types/user.ts)
interface User {
  id: string;
  email: string;
  role: 'admin' | 'member';
  createdAt: Date;
}

// ✅ Request-scoped type (co-located with the route file)
interface CreateUserBody {
  email: string;
  name: string;
}
```

### JSDoc Comments

Add JSDoc to:

- All exported functions and classes
- Complex business logic
- Non-obvious decisions or workarounds

```typescript
/**
 * Retrieves a user by ID.
 * @throws {NotFoundError} if no user with the given ID exists.
 * @param id - The UUID of the user.
 */
async function getUser(id: string): Promise<User> { ... }
```

### ESLint

**AirBnb ESLint configuration is the standard.** All code must pass `eslint .` with zero warnings. Key rules to be aware of:

- No unused variables (`no-unused-vars`)
- Consistent return (`consistent-return`)
- No `console.log` in production code (`no-console`) — use a proper logger
- Prefer `const` over `let`; never use `var`

---

## Express Patterns

### Typed Request Handlers

Always use TypeScript generics on Express `Request` to type the body, params, and query:

```typescript
import { Request, Response, NextFunction } from 'express';

interface CreateUserBody {
  email: string;
  name: string;
}

const createUser = async (
  req: Request<{}, {}, CreateUserBody>,
  res: Response,
  next: NextFunction,
): Promise<void> => {
  try {
    const { email, name } = req.body;
    // validate, create, respond
    res.status(201).json({ user });
  } catch (err) {
    next(err); // Let error middleware handle it
  }
};
```

### Centralised Error Middleware

Never return error responses inline in route handlers. Always call `next(err)` and handle all errors in a single error-handling middleware:

```typescript
import { Request, Response, NextFunction } from 'express';

export function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction,
): void {
  const status = (err as any).statusCode ?? 500; // eslint-disable-next-line
  res.status(status).json({
    error: err.message ?? 'Internal Server Error',
  });
}
```

Register it **last**, after all routes:

```typescript
app.use(errorHandler);
```

---

## Testing Standards

### Jest Setup

Use Jest with the following packages:

- `@testing-library/react` for React component tests
- `supertest` for Express API tests
- `jest-mock-extended` or manual mocks for Prisma

### AAA Pattern

Every test follows **Arrange → Act → Assert**:

```typescript
// ✅ DO: Descriptive tests with AAA pattern
describe('UserService', () => {
  describe('getUser', () => {
    it('should return the user when found', async () => {
      // Arrange
      const mockUser = { id: '1', email: 'test@example.com' };
      prismaMock.user.findUnique.mockResolvedValue(mockUser);

      // Act
      const result = await userService.getUser('1');

      // Assert
      expect(result).toEqual(mockUser);
    });

    it('should throw NotFoundError when user does not exist', async () => {
      // Arrange
      prismaMock.user.findUnique.mockResolvedValue(null);

      // Act & Assert
      await expect(userService.getUser('999'))
        .rejects
        .toThrow(NotFoundError);
    });
  });
});
```

### Test Naming

Test names must read as a specification sentence:

```typescript
// ✅ DO
it('should return 401 when the request has no auth token')
it('should return the paginated list of users with a default limit of 20')

// ❌ DON'T
it('test auth')
it('users test')
```

### Mocking Strategy

- **Mock external dependencies**: databases (Prisma, Mongoose), third-party APIs, file system, timers.
- **Do not mock your own code**: test services, utilities, and business logic with real implementations.
- **For Express routes**: use `supertest` to make real HTTP requests against the app — do not mock the router.

```typescript
// ✅ supertest for API testing
import request from 'supertest';
import app from '../app';

it('should return 201 and the new user on POST /users', async () => {
  const res = await request(app)
    .post('/users')
    .send({ email: 'new@example.com', name: 'Alice' });

  expect(res.status).toBe(201);
  expect(res.body.user.email).toBe('new@example.com');
});
```

### Coverage Focus

Aim for **meaningful coverage**, not 100% line coverage. Prioritise:

- Happy paths
- Error and edge cases from the acceptance criteria
- Boundary conditions
- Input validation

---

## Project Conventions

### File and Folder Structure

```
src/
├── components/          # React components (PascalCase directories)
│   ├── UserCard/
│   │   ├── UserCard.tsx
│   │   ├── UserCard.test.tsx
│   │   └── index.ts
│   └── ...
├── pages/ or app/       # Next.js pages/app router
├── hooks/               # Custom React hooks (camelCase)
│   ├── useAuth.ts
│   └── useAuth.test.ts
├── services/            # Business logic (camelCase)
│   ├── userService.ts
│   └── userService.test.ts
├── routes/              # Express route handlers
│   ├── userRoutes.ts
│   └── userRoutes.test.ts
├── middleware/           # Express middleware
├── types/               # Shared TypeScript types/interfaces
├── utils/               # Utility functions
│   ├── validation.ts
│   └── validation.test.ts
└── config/              # Configuration and environment
```

**Rules:**

- React components live in `PascalCase` directories with an `index.ts` barrel export.
- Hooks, services, routes, and utilities use `camelCase` filenames.
- Test files are co-located with the source file they test (`*.test.ts` / `*.test.tsx`).
- Shared interfaces and domain types live in `src/types/`.

### Import Order

Always organise imports in this order, separated by blank lines:

```typescript
// 1. Node.js built-in modules
import path from 'path';

// 2. External packages
import express from 'express';
import { z } from 'zod';

// 3. Internal modules (absolute paths / aliases)
import { UserService } from '@/services/userService';
import { validateRequest } from '@/middleware/validation';

// 4. Relative imports
import { UserCard } from './UserCard';
import type { UserCardProps } from './types';
```

Configure this order in your ESLint config using `eslint-plugin-import` with `import/order`.
