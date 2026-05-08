# CleanerCMS — Backend Agent

## Your role
You own everything in apps/api and packages/db.
You DO NOT touch apps/web.
When you create or change a TypeScript type that the frontend needs,
you update packages/domain. That is the contract between you and the frontend agent.

## Full domain context
Read /CLAUDE.md before every session. It contains the domain concepts,
entity names, financial rules and constraints that govern all decisions here.

---

## Stack
- NestJS (modules, controllers, services, guards)
- Prisma ORM — schema lives in packages/db/schema.prisma
- PostgreSQL
- BullMQ for async jobs (billing runs, weekly timesheet generation, period instantiation)
- Passport + JWT for auth
- class-validator + class-transformer for DTOs
- Jest for unit and integration tests

---

## Project structure inside apps/api



```
apps/api/src/
├── auth/                  # JWT strategy, claims guard, user context
├── modules/
│   ├── org/               # OrganisationalHierarchyNode + Type
│   ├── contracts/         # Contract, ContractVersion lifecycle
│   ├── budgets/           # Budget (all 3 valuation methods + metrics)
│   ├── workplans/         # Workplan, WorkplanShift, Role, Payscale
│   ├── timesheets/        # Timesheet, TimesheetShift, Shift
│   ├── employees/         # Employee, EmployeeContract, holiday
│   ├── services/          # ServicePlan, Service, ServiceType, ServiceProvider
│   ├── store/             # StoreItem, StoreItemPlan, StoreItemOrder, Supplier
│   ├── jobs/              # Job lifecycle
│   ├── assets/            # Asset register
│   ├── finance/           # BillingRun, Invoice, CreditNote, PayrollRun
│   └── audit/             # AuditLog (write-only)
├── queues/                # BullMQ job processors
├── common/
│   ├── guards/            # ClaimsGuard
│   ├── decorators/        # @RequireClaim(), @CurrentUser()
│   ├── interceptors/      # AuditInterceptor
│   └── pipes/             # ValidationPipe config
└── main.ts
```





---

## Auth — claims-based, NOT role-based

### How it works
A User has UserRoleAssignments. Each assignment links a User to a Role
scoped to a specific OrganisationalHierarchyNode.
Claims are resolved by: collecting all roles assigned to the user across
all nodes that are ancestors-of or equal-to the resource's node,
then flattening all RoleClaims from those roles.

### Guard usage


```typescript
@RequireClaim('contracts:create')
@Post()
createContract() {}
```





### ClaimsGuard must
1. Extract the target node from the request (from body, param or query)
2. Walk UP the org tree from that node to root, collecting nodeIds
3. Find all UserRoleAssignments where nodeId is in that ancestor set
4. Collect all claims from those roles
5. Check the required claim is present

### Never do this


```typescript
// WRONG — role name checks are forbidden
if (user.role === 'ContractManager') { ... }
```





### Standard claims list


```
org:read, org:create, org:update, org:delete
contracts:read, contracts:create, contracts:update, contracts:activate
budgets:read, budgets:read:limited, budgets:create, budgets:update
workplans:read, workplans:create, workplans:update
timesheets:read, timesheets:submit, timesheets:approve
employees:read, employees:create, employees:update
jobs:read, jobs:create, jobs:approve
store:read, store:order, store:approve
services:read, services:create, services:update
finance:read, finance:billing-run, finance:invoices:approve
finance:credit-notes:create, finance:credit-notes:approve
finance:payroll:export, finance:payroll:attribute
assets:read, assets:create, assets:update
admin:users, admin:roles, admin:templates
```





---

## Prisma schema rules

- All IDs are `String @id @default(cuid())`
- All timestamps are `DateTime` with UTC assumption
- Soft deletes via `deletedAt DateTime?` — never hard delete core entities
- Use explicit relation names when a model has multiple relations to the same target
- Every model that can change must have `updatedAt DateTime @updatedAt`

### Key schema patterns



```prisma
// Self-referential hierarchy
model OrganisationalHierarchyNode {
  id       String  @id @default(cuid())
  name     String
  typeId   String
  parentId String?
  parent   OrganisationalHierarchyNode?  @relation("NodeChildren", fields: [parentId], references: [id])
  children OrganisationalHierarchyNode[] @relation("NodeChildren")
  type     OrganisationalHierarchyType   @relation(fields: [typeId], references: [id])
}

// Budget self-referential hierarchy
model Budget {
  id                        String   @id @default(cuid())
  contractVersionId         String
  parentBudgetId            String?
  name                      String
  valuationMethod           BudgetValuationMethod
  fixedValue                Decimal?
  percentageSourceBudgetId  String?
  percentageValue           Decimal?
  profitMarginPercent       Decimal?
  amortisationMonths        Int?
  categoryId                String?
  parent                    Budget?   @relation("BudgetChildren", fields: [parentBudgetId], references: [id])
  children                  Budget[]  @relation("BudgetChildren")
  percentageSource          Budget?   @relation("PercentageSource", fields: [percentageSourceBudgetId], references: [id])
  percentageTargets         Budget[]  @relation("PercentageSource")
  contractVersion           ContractVersion @relation(fields: [contractVersionId], references: [id])
}

enum BudgetValuationMethod {
  FIXED
  PERCENTAGE
  CALCULATED
}

// ContractVersion status
enum ContractVersionStatus {
  DRAFT
  LIVE
  ARCHIVED
}
```





---

## Service layer rules

### Budget calculation service
This is the most complex service. It must handle:

1. **Expected cost** — recursive, bottom-up calculation
   - Leaf budgets: sum expected costs of their plans
   - Parent budgets: sum children's budgeted values + own plans' expected costs
   - Apply profitMarginPercent last on CALCULATED budgets

2. **Circular reference guard** — before saving a PERCENTAGE budget,
   verify the source budget is not itself a PERCENTAGE budget

3. **Amortisation** — budget service must accept a `view` param:
   `'accrual'` | `'cashflow'`
   Accrual: spread FIXED budget over amortisationMonths
   Cash flow: full value in period 1, zero thereafter

4. **Actuals rollup** — sum approved Timesheets + invoiced Services +
   fulfilled StoreItemOrders attributed to this budget

### ContractVersion service
- `activate(id)` — sets status to LIVE, sets startDate to now,
  sets previous LIVE version's endDate to yesterday.
  MUST check approval workflow before allowing activation.
- `createDraft(contractId)` — deep-copies the current LIVE version:
  copies all Budgets (tree), Workplans, WorkplanShifts,
  ServicePlans, StoreItemPlans. Sets status to DRAFT.
- NEVER update a LIVE ContractVersion's financial fields directly.
  Return a 409 if attempted.

### Period instantiation (BullMQ — runs every Monday 00:01)
For every LIVE ContractVersion:
1. Workplan → create Timesheet for the week with TimesheetShifts
   copied from WorkplanShifts
2. ServicePlan → create Service instances due this period
3. StoreItemPlan → create StoreItemOrders due this period

### Billing run (BullMQ — triggered manually by Finance user)
1. Find all contracts with BillingProfile due in the requested period
2. For each: collate approved, unbilled Timesheets + Services + StoreItemOrders
3. Generate draft Invoice per contract formatted per BillingProfile.detailLevel:
   - SUMMARY: one line item
   - BY_BUDGET: one line item per top-level budget
   - DETAILED: one line item per budget including sub-budgets
4. Handle billable Jobs: per BillingProfile.billableJobHandling setting
5. Set all included items status to 'pending_invoice'
6. On Invoice approval: set items to 'billed', dispatch to mock finance stub

---

## Approval workflow

Implement as a reusable ApprovableEntity pattern.
Each approvable entity (Timesheet, Job, StoreItemOrder, CreditNote, ContractVersion)
has: status, submittedAt, approvedAt, approvedBy, rejectedAt, rejectedBy, comments.

Hardcoded thresholds (Sprint 1–3). Make configurable in Sprint 4:
- Timesheet: auto-approve if actualHours within 10% of plannedHours
- Job: auto-approve if estimatedCost < 500
- StoreItemOrder: auto-approve if value < remainingBudget
- CreditNote: requires approval if value > 1000
- ContractVersion activation: requires approval if projectedMargin < 5%

---

## Audit logging

Every mutation (create/update/delete/approve) MUST write to AuditLog.
Implement as a NestJS interceptor that wraps all mutating endpoints.



```typescript
// AuditLog is write-only. No update or delete endpoints exist for it.
interface AuditLogEntry {
  userId: string
  entityType: string   // e.g. 'Budget', 'ContractVersion'
  entityId: string
  action: 'create' | 'update' | 'delete' | 'approve' | 'reject'
  oldValue: object | null
  newValue: object | null
  timestamp: Date      // UTC always
}
```





---

## API response shapes — publish these to packages/domain

Every response type must be exported from packages/domain/src/api-types.ts
so the frontend agent can import them without duplicating types.

Standard patterns:


```typescript
// List response
interface PaginatedResponse<T> {
  data: T[]
  total: number
  page: number
  pageSize: number
}

// Budget with computed metrics
interface BudgetResponse {
  id: string
  name: string
  valuationMethod: 'fixed' | 'percentage' | 'calculated'
  budgeted: number    // computed
  expected: number    // computed
  actual: number      // computed
  variance: number    // budgeted - actual
  children: BudgetResponse[]
}

// P&L view
interface PLResponse {
  view: 'accrual' | 'cashflow'
  period: string
  budgets: BudgetResponse[]
  totalBudgeted: number
  totalExpected: number
  totalActual: number
  margin: number
}
```





---

## Testing rules

- Every financial calculation service method needs a unit test
- Use specific numbers matching the spec examples, e.g.:
  "Workplan: 2 staff × 4 hours × £12/hr = £96/day expected"
- Integration tests use a test database (TEST_DATABASE_URL in .env.test)
- Mock external stubs (payroll, accounting, HR) are in src/stubs/
- Every BullMQ processor needs a unit test with mocked queue

---

## External system stubs (mock, not real integrations)

Create realistic stub services in src/stubs/:



```
src/stubs/
├── payroll.stub.ts       # Accepts PayrollRun, returns PayrollResult with NI/pension
├── accounting.stub.ts    # Accepts Invoice/CreditNote, returns confirmation
├── hr.stub.ts            # Webhook receiver for EmployeeMaster events
├── ta.stub.ts            # Returns TimeLogs for a site/period
└── edi.stub.ts           # Accepts PurchaseOrder, returns OrderConfirmation
```



Each stub logs what it received and returns a plausible response.
This decouples development from real third-party availability.

---

## Sprint build order

Sprint 1: Prisma schema (all entities), auth + claims, org hierarchy,
          Contract/ContractVersion lifecycle, Budget (3 valuation methods),
          seed script

Sprint 2: Role, Payscale, Workplan/WorkplanShift, Employee/EmployeeContract,
          Timesheet lifecycle + BullMQ weekly generator, Shift, holiday (4 methods)

Sprint 3: ServicePlan/Service, StoreItem/StoreItemPlan/StoreItemOrder,
          Job (both costing methods), Calendar, Asset register,
          wage rate types on shifts

Sprint 4: BillingProfile, BillingRun (async), Invoice, CreditNote,
          PayrollRun export + result import + attribution,
          amortisation dual-view, configurable approval thresholds

---

## Before ending any session

1. Run `npm run test` — fix all failures before stopping
2. Update packages/domain/src/api-types.ts with any new response shapes
3. Leave a comment at the top of your last modified file:
   `// SESSION END: [what was completed] [what is next]`
