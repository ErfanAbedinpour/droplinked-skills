# Developer Checklists

## Pre-Commit Checklist

Before committing code, verify all items:

### DTO & Validation
- [ ] All DTO input fields have `@Is...` decorators (`@IsString`, `@IsOptional`, etc.)
- [ ] No field is left without a decorator
- [ ] Controller `ID` parameters use `IsObjectIdPipe`

### UseCase Structure
- [ ] `doExecute` is broken into smaller steps (private methods)
- [ ] Numbered comments describe the flow (// 1. Validate, // 2. Calculate, etc.)
- [ ] `interface`s defined for data transfer between internal functions
- [ ] No `any` types - all data is typed

### Business Logic
- [ ] Business rules (inventory, ownership, etc.) validated before changes
- [ ] Semantic validation happens in `validate()` or early in `doExecute()`

### Data Access
- [ ] Complex DB queries moved to Repository with `select` optimization
- [ ] No direct Prisma calls in `doExecute` - use semantic private methods
- [ ] Cross-module data accessed via Service Facade, not direct DB query

### Testing
- [ ] At least one test for the success path
- [ ] At least one test for the failure path (invalid input, not found, etc.)

### Logging & Errors
- [ ] Errors include context (cartId, userId, etc.)
- [ ] Using NestJS exceptions (BadRequestException, NotFoundException, etc.)
- [ ] No swallowed errors (empty catch blocks)

### Side Effects
- [ ] Non-blocking side effects (email, analytics) use Events
- [ ] Core logic requiring consistency uses direct calls

### Documentation
- [ ] Public methods have JSDoc comments
- [ ] UseCase documented in module's `docs.md`

---

## Code Review Checklist

Focus on issues that can cause: production incidents, data corruption, security breaches, or architectural decay.

### Critical Issues to Flag

| Category | Examples |
|----------|----------|
| **Architectural Violations** | Business logic in Repository, DB access in UseCase, cross-module coupling |
| **Data Integrity** | Non-atomic multi-step operations, missing rollback, partial writes |
| **Error Handling** | Swallowed errors, returning errors instead of throwing |
| **Performance** | N+1 queries, queries in loops, missing indexes |
| **Security** | Missing validation, SQL/NoSQL injection risks, exposed secrets |

### Ignore
- Formatting preferences
- Naming style preferences
- Code length concerns
- Micro-optimizations

---

## New Module Checklist

When creating a new module:

### Folder Structure
```
src/modules/<module-name>/
├── controllers/
│   └── <module>.controller.ts
├── services/
│   └── <module>.service.ts
├── use-cases/
│   └── <operation>.use-case.ts
├── repositories/
│   └── <entity>.repository.ts
├── dtos/
│   ├── <operation>.dto.ts
│   └── <entity>-response.dto.ts
├── events/
│   └── <entity>-<action>.event.ts
├── listeners/
│   └── <entity>.listener.ts
├── <module>.module.ts
└── docs.md
```

### Module Registration
- [ ] Module registered in `app.module.ts`
- [ ] Exports declared for services used by other modules
- [ ] Imports include required dependencies

### Controller Setup
- [ ] Swagger decorators for API documentation
- [ ] Auth guards where required
- [ ] `IsObjectIdPipe` for ID parameters
- [ ] Proper HTTP methods and status codes

### Service Facade
- [ ] Injects UseCases via constructor or `ModuleRef`
- [ ] No business logic - only orchestration
- [ ] Methods match controller actions

### Documentation
- [ ] `docs.md` created with exposed services documented
- [ ] Each method has: Input, Output, Description, Example

---

## Quick Manual Test Checklist

Before pushing, manually verify:

1. **Negative Test**: Does the DTO reject invalid input?
   - Missing required fields
   - Wrong types
   - Invalid ObjectIds

2. **Positive Test**: Does the core logic work with valid data?
   - Happy path returns expected result
   - Data persisted correctly

3. **Business Error Test**: Do business errors provide appropriate messages?
   - Insufficient stock returns clear message
   - Unauthorized access returns 403
   - Not found returns 404

---

## Database Migration Checklist

When modifying `prisma/schema.prisma`:

- [ ] Run `npm run db:generate` after changes
- [ ] New fields have sensible defaults or are optional
- [ ] Indexes added for frequently queried fields
- [ ] No breaking changes to existing data
- [ ] Related TypeScript types updated
- [ ] Zod schemas regenerated (if using `prisma-zod-generator`)
