# CleanerCMS — Frontend Agent

## Your role
You own everything in apps/web.
You DO NOT touch apps/api, packages/db, or any backend files.
You import types from packages/domain — never define your own API response shapes.
If a type you need doesn't exist in packages/domain, stop and ask before inventing one.

## Full domain context
Read /CLAUDE.md before every session. It contains the domain concepts
and entity relationships that determine how the UI should behave.

---

## Stack
- React 18 + TypeScript
- Vite (dev server + build)
- React Router v6 (file-based routing convention in src/pages/)
- Tailwind CSS (utility-first styling)
- TanStack Query (server state, caching, loading/error states)
- TanStack Table (data tables — contracts, timesheets, jobs etc.)
- React Hook Form + Zod (all forms)
- Zustand (client-only UI state — modals, sidebar, preferences)
- Recharts (P&L charts, budget visualisations)
- date-fns (all date formatting and manipulation)
- Lucide React (icons only — no custom SVG icons)

---

## Project structure inside apps/web


```
apps/web/src/
├── pages/                 # One file per route (React Router convention)
│   ├── dashboard/
│   ├── contracts/
│   ├── budgets/
│   ├── workplans/
│   ├── timesheets/
│   ├── employees/
│   ├── jobs/
│   ├── services/
│   ├── store/
│   ├── assets/
│   ├── finance/
│   │   ├── billing/
│   │   ├── invoices/
│   │   ├── credit-notes/
│   │   └── payroll/
│   └── settings/
├── components/
│   ├── ui/                # Primitive components (Button, Input, Modal, Badge etc.)
│   ├── layout/            # AppShell, Sidebar, TopNav, PageHeader
│   ├── budget/            # BudgetTree, BudgetMetricsRow, PLChart
│   ├── contracts/         # ContractCard, VersionBadge, ContractTimeline
│   ├── timesheets/        # TimesheetGrid, ShiftRow, HoursVarianceBadge
│   ├── jobs/              # JobCard, JobStatusBadge, CostingMethodToggle
│   └── shared/            # ApprovalActions, AuditLogList, HierarchyBreadcrumb
├── hooks/                 # useContracts, useBudgets, useCurrentUser etc.
├── lib/
│   ├── api.ts             # Typed fetch wrapper
│   ├── auth.ts            # Token storage, current user
│   └── query-client.ts    # TanStack Query config
├── stores/                # Zustand stores (ui.store.ts, preferences.store.ts)
└── types/                 # Re-exports from packages/domain (never duplicate)
```



---

## API communication rules

### Never hardcode response shapes
All API response types come from packages/domain:

```typescript
// CORRECT
import { BudgetResponse, ContractResponse } from '@cleanercms/domain'

// WRONG — never define your own API types
interface MyBudget { id: string; value: number }
```



### Typed API wrapper
All API calls go through src/lib/api.ts, never raw fetch:

```typescript
// src/lib/api.ts
export async function apiGet<T>(path: string): Promise<T>
export async function apiPost<T>(path: string, body: unknown): Promise<T>
export async function apiPut<T>(path: string, body: unknown): Promise<T>
export async function apiDelete(path: string): Promise<void>
```



### TanStack Query for all server state

```typescript
// hooks/useContracts.ts
export function useContracts(nodeId: string) {
  return useQuery({
    queryKey: ['contracts', nodeId],
    queryFn: () => apiGet<ContractResponse[]>(`/contracts?nodeId=${nodeId}`)
  })
}

// Mutation with optimistic update
export function useActivateContract() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: (id: string) => apiPut(`/contracts/${id}/activate`, {}),
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ['contracts'] })
  })
}
```



### Never use useEffect for data fetching
Always use TanStack Query. useEffect is only for non-server-state side effects
(scroll position, focus management, third-party library setup).

---

## Forms

All forms use React Hook Form + Zod. No exceptions.


```typescript
const schema = z.object({
  name: z.string().min(1, 'Name is required'),
  valuationMethod: z.enum(['fixed', 'percentage', 'calculated']),
  fixedValue: z.number().optional(),
})

type FormData = z.infer<typeof schema>

function BudgetForm() {
  const { register, handleSubmit, watch, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema)
  })
  // ...
}
```



Zod schemas for forms live in src/lib/schemas/ and mirror the API validation.

---

## Styling rules

Use Tailwind utility classes. No inline styles, no CSS modules, no styled-components.

Design system constraints:
- Font: system-ui (Tailwind default)
- Colours: use Tailwind's slate palette for neutrals, blue-600 for primary actions,
  green-600 for positive/approved, red-600 for negative/rejected/over-budget,
  amber-500 for warnings/draft state
- Spacing: 4px base unit (Tailwind p-1 = 4px)
- Border radius: rounded-md (6px) for inputs/buttons, rounded-lg (8px) for cards
- Shadows: shadow-sm for cards, no shadow on inputs

### Status badge colours (consistent across all entities)

```typescript
// Use this pattern everywhere
const STATUS_COLOURS = {
  draft:       'bg-amber-100 text-amber-800',
  live:        'bg-green-100 text-green-800',
  archived:    'bg-slate-100 text-slate-600',
  pending:     'bg-blue-100 text-blue-800',
  approved:    'bg-green-100 text-green-800',
  rejected:    'bg-red-100 text-red-800',
  in_progress: 'bg-blue-100 text-blue-800',
  completed:   'bg-green-100 text-green-800',
}
```



### Budget variance colours
- Under budget (actual < budgeted): text-green-600
- Over budget (actual > budgeted): text-red-600
- Within 10% of budget: text-amber-600

---

## Key UI patterns

### Budget tree
The hierarchical budget view is the most important UI in the system.
It must show the three metrics (budgeted / expected / actual) at every level,
collapse/expand child budgets, and highlight variance.


```
Contract Name
├── Labour & Staffing         £8,400  £8,400  £8,210   ✓
│   └── Workplan: Mon–Fri     £8,400  £8,400  £8,210
├── Consumables (5% of labour) £420    £420    £390    ✓
├── Capital Expenditure        £1,200  £1,200  £1,200  ✓ (amortised: £100/mo)
└── TOTAL (+ 15% margin)      £11,523 £11,523 £11,270
```



Use a recursive component. Each BudgetNode renders its children.
The tree should be collapsible. Leaf nodes show plan details on expand.

### P&L chart
Recharts bar chart. Three grouped bars per period: Budgeted, Expected, Actual.
Toggle between Accrual and Cash Flow view (calls different API endpoint, same component).

### Timesheet grid
Weekly grid: rows = employees, columns = Mon–Sun.
Each cell = hours worked that day. Editable inline.
Highlight cells where actual ≠ planned (amber background).
Submit and Approve buttons at the top with approval state badge.

### Approval actions
Reusable component used on Timesheets, Jobs, StoreItemOrders, CreditNotes, ContractVersions:

```
[Approve] [Reject] [Add comment]
```


Show current approval state as a badge. Show approver name + timestamp when approved.

### Draft/Live version switcher
On contract pages, show a version selector in the page header:

```
[v3 — Live ▼] [+ New Draft]
```


Dropdown lists all versions with status badges and dates.
Viewing a Draft version shows a yellow banner: "You are viewing a Draft version".

### Hierarchy breadcrumb
Every page scoped to a node shows the org hierarchy path:

```
UK North > Manchester > Site: Arndale Centre > Contract: Core Cleaning 2025
```


Each segment is clickable and navigates to that level.

---

## Auth and permissions in the UI

Current user's resolved claims come from the /auth/me endpoint.
Store them in Zustand after login. Use a hook to check claims in components:


```typescript
// hooks/useAuth.ts
export function useCan(claim: string): boolean {
  const claims = useAuthStore(s => s.claims)
  return claims.includes(claim)
}

// Usage in component
function ContractActions() {
  const canCreate = useCan('contracts:create')
  return canCreate ? <Button>New Contract</Button> : null
}
```



Never hide navigation items — show them disabled with a tooltip explaining
the user doesn't have access. Only hide action buttons (create, approve, delete).

---

## Loading and error states

Every data-fetching component must handle three states explicitly:

```typescript
const { data, isLoading, error } = useContracts(nodeId)

if (isLoading) return <Skeleton />      // show skeleton, not spinner
if (error) return <ErrorCard error={error} />
return <ContractList contracts={data} />
```



Use skeleton components (pulsing grey boxes matching the shape of the content)
not generic spinners. Each page has its own skeleton that matches its layout.

---

## Page structure

Every page follows this layout:

```tsx
<PageLayout>
  <PageHeader
    title="Contracts"
    breadcrumb={<HierarchyBreadcrumb nodeId={nodeId} />}
    actions={canCreate && <Button onClick={openCreateModal}>New Contract</Button>}
  />
  <PageContent>
    {/* main content */}
  </PageContent>
</PageLayout>
```



---

## Navigation structure (sidebar)


```
Dashboard
Contracts
  └── (per site, filtered by user's accessible nodes)
Timesheets
Jobs
Services
Store
Employees
Assets
Finance
  ├── Billing
  ├── Invoices
  ├── Credit Notes
  └── Payroll
Reports
Settings (admin only)
```


---

## Sprint build order

Sprint 1: App shell, routing, auth (login + token), org hierarchy nav,
          contract list + detail, budget tree component (all 3 valuation types),
          P&L chart (accrual/cashflow toggle)

Sprint 2: Workplan builder, timesheet weekly grid + approval flow,
          employee list + profile (holiday balance), role/payscale management

Sprint 3: Service schedule calendar, store item catalogue + site template,
          job creation form + approval queue,
          asset register

Sprint 4: Billing run dashboard, invoice review queue,
          credit note form, payroll reconciliation screen,
          full P&L reporting page with filters

---

## Before ending any session

1. Run `npm run build` — fix all TypeScript errors before stopping
2. All new components must be typed — no `any`
3. Leave a comment at the top of your last modified file:
   `// SESSION END: [what was completed] [what is next]`
