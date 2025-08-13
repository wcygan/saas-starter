# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Essential Commands

### Development
```bash
pnpm dev              # Start development server with Turbopack
pnpm build            # Production build
pnpm start            # Start production server
```

### Database Operations
```bash
pnpm db:setup         # Create .env file with database setup
pnpm db:migrate       # Run database migrations
pnpm db:seed          # Seed database with test user (test@test.com / admin123)
pnpm db:generate      # Generate new migrations from schema changes
pnpm db:studio        # Launch Drizzle Studio database GUI
```

### Stripe Development
```bash
stripe login          # Authenticate with Stripe CLI
stripe listen --forward-to localhost:3000/api/stripe/webhook  # Local webhook testing
```

## Architecture Overview

### Next.js App Router Structure
- `app/(dashboard)/` - Protected dashboard routes requiring authentication
- `app/(login)/` - Authentication flows (sign-in, sign-up)  
- `app/api/` - API routes for user/team management and Stripe webhooks
- `middleware.ts` - Global route protection and session management

### Multi-Tenant Team-Based Architecture
- **Teams** are the primary tenant boundary - all data is scoped by team
- **Users** belong to teams through `team_members` with roles (`owner`, `member`)
- **Team owners** can invite members, manage subscriptions, and control team settings
- **Activity logging** tracks all user actions within team context

### Key Directories
- `lib/auth/` - Authentication utilities and middleware patterns
- `lib/db/` - Database schema, queries, and Drizzle ORM configuration  
- `lib/payments/` - Stripe integration and subscription management
- `components/ui/` - shadcn/ui components with Tailwind CSS

## Database Architecture (Drizzle + PostgreSQL)

### Core Schema
- `users` - User accounts with soft delete support (`deletedAt`)
- `teams` - Multi-tenant organizations with Stripe billing fields
- `team_members` - User-to-team relationships with role-based access
- `activity_logs` - Comprehensive audit trail (10 activity types)
- `invitations` - Team invitation workflow management

### Data Patterns
- **Multi-tenancy**: All queries scoped by team membership
- **Soft deletes**: Users marked deleted, not removed (preserves audit trail)
- **Activity logging**: Required team context, tracks authentication and team actions
- **Foreign keys**: Proper relational constraints with "no action" policies

### Migration Commands
```bash
# After schema changes in lib/db/schema.ts
pnpm db:generate      # Create migration file
pnpm db:migrate       # Apply migrations to database
```

## Authentication System

### Session Management
- **JWT tokens** with 24-hour rolling expiration via secure HttpOnly cookies
- **bcryptjs** password hashing with 10 salt rounds
- **Automatic refresh** on authenticated GET requests
- **Global middleware** protects `/dashboard*` routes

### Server Action Patterns
Three middleware wrappers for server actions:
1. `validatedAction` - Basic Zod schema validation
2. `validatedActionWithUser` - Requires authentication, injects user
3. `withTeam` - Requires authentication + team membership, injects team data

### RBAC Implementation  
- **Team-level roles**: `owner` (full permissions) and `member` (limited)
- **Permission checks** in UI components and server actions
- **Team scoping** ensures users only access their team's data

## Stripe Payment Integration

### Subscription Features
- **Checkout sessions** for subscription purchases
- **Customer portal** for self-service billing management  
- **Webhook processing** for real-time subscription status updates
- **Trial support** with 14-day free trials

### Database Integration
Teams table includes Stripe fields:
- `stripeCustomerId` - Links to Stripe customer
- `stripeSubscriptionId` - Active subscription reference
- `subscriptionStatus` - Current status (active, trialing, canceled, etc.)
- `planName` and `stripeProductId` - Product configuration

### Payment Flow
1. User selects plan → API creates Stripe checkout session
2. Payment completed → Webhook updates team subscription data  
3. Team gains access to paid features based on subscription status

## Development Patterns

### TypeScript Configuration
- **Strict mode** enabled with comprehensive type checking
- **Path aliases** using `@/*` for root-level imports
- **Module resolution** set to `bundler` for Next.js compatibility

### Server Actions
```typescript
// Standard pattern for authenticated team actions
export const someAction = validatedActionWithUser(
  schema,
  async (data, formData, user) => {
    const userWithTeam = await getUserWithTeam(user.id);
    if (!userWithTeam?.teamId) {
      return { error: 'User is not part of a team' };
    }
    // Perform action...
    await logActivity(userWithTeam.teamId, user.id, ActivityType.ACTION);
    return { success: 'Action completed' };
  }
);
```

### UI Component Conventions
- **shadcn/ui** components with consistent styling
- **Form validation** using Zod schemas
- **Permission-based rendering** using user role checks
- **Loading states** and error handling for better UX

## Testing Architecture (Currently Missing)

### Recommended Setup
No tests currently exist. For comprehensive coverage, implement:

**Unit Testing (Vitest + Testing Library)**
```bash
pnpm add -D vitest @testing-library/react @testing-library/jest-dom jsdom
```
- Authentication utilities (`lib/auth/`)
- Database queries (`lib/db/queries.ts`)  
- UI components (`components/ui/`)

**Integration Testing (Vitest + TestContainers)**
```bash
pnpm add -D @testcontainers/postgresql
```
- API routes with database interactions
- Stripe webhook processing
- Authentication middleware

**E2E Testing (Playwright)**
```bash  
pnpm add -D @playwright/test
```
- Complete user flows (registration, subscription, dashboard)
- Cross-browser compatibility testing

### Priority Test Areas
1. Authentication and session management
2. Team-based data isolation  
3. Stripe payment flows and webhooks
4. RBAC permission enforcement

## Environment Variables

Required for development:
```bash
# Database
POSTGRES_URL="postgresql://..."

# Authentication  
AUTH_SECRET="random-32-char-string"

# Stripe
STRIPE_SECRET_KEY="sk_test_..."
STRIPE_WEBHOOK_SECRET="whsec_..."

# Application
BASE_URL="http://localhost:3000"
```

## Common Development Tasks

### Adding New Team Features
1. Update database schema in `lib/db/schema.ts`
2. Generate and run migrations: `pnpm db:generate && pnpm db:migrate`
3. Add server actions with `validatedActionWithUser` or `withTeam`
4. Implement UI components with proper permission checks
5. Add activity logging for audit trail

### Debugging Authentication Issues  
1. Check session cookie in browser dev tools
2. Verify JWT token in `lib/auth/session.ts:verifyToken()`
3. Confirm middleware routing in `middleware.ts`
4. Review activity logs for authentication events

### Testing Stripe Integration
1. Use Stripe CLI: `stripe listen --forward-to localhost:3000/api/stripe/webhook`
2. Test cards: `4242 4242 4242 4242` (any future date, any CVC)
3. Monitor webhook events in Stripe dashboard
4. Check team subscription status updates in database