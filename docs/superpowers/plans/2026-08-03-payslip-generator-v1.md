# Payslip Generator V1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the V1 shared payslip generator with supervisor login, payroll batches, Excel/CSV import preview, manual editing, one-page payslip preview, individual/batch printing, automatic print tracking, and searchable history.

**Architecture:** Use Next.js App Router for the supervisor interface and Convex as the only backend. Keep business rules in shared TypeScript modules where possible, persist canonical data in Convex tables, and use custom HTML/CSS for the printable payslip page while using shadcn/ui for the application UI.

**Tech Stack:** Next.js, TypeScript, Tailwind CSS, shadcn/ui, Convex, Convex Auth, Vitest, Playwright, SheetJS `xlsx`, date-fns.

## Global Constraints

- Use Next.js for the web app UI.
- Use shadcn/ui and Tailwind CSS for supervisor interface components.
- Use Convex for the shared backend, live data, users, employees, payroll batches, payslip history, and print events.
- Use Convex Auth for supervisor login.
- Recreate the company's PDF/image payslip template as custom HTML/CSS.
- Parse Excel/CSV on the client for import preview before saving rows to Convex.
- Print one payslip per page.
- Record automatic print tracking when the browser print dialog is triggered; do not claim physical printer success.
- Keep V1 focused on supervisor login, employees, payroll batches, import preview, manual editing, finalize, print, batch print, print events, history, and basic validation.
- Keep these V1.5 features outside this plan: full audit log, role permissions, template manager, summary reports, CSV history export, bulk finalize, and archive workflows.

---

## File Structure

- `app/layout.tsx`: Root shell, metadata, global providers.
- `app/page.tsx`: Dashboard route.
- `app/employees/page.tsx`: Employee list route.
- `app/batches/page.tsx`: Payroll batch list route.
- `app/batches/[batchId]/page.tsx`: Batch detail and import/manual-entry workflow.
- `app/payslips/[payslipId]/page.tsx`: One-page payslip preview and print actions.
- `app/history/page.tsx`: Searchable payslip history route.
- `app/sign-in/page.tsx`: Convex Auth sign-in route.
- `app/globals.css`: Tailwind theme, shadcn variables, print CSS.
- `components/app/app-shell.tsx`: Sidebar/topbar layout.
- `components/app/nav.tsx`: Main navigation links.
- `components/batches/batch-form.tsx`: Create batch form.
- `components/batches/batch-table.tsx`: Batch list table.
- `components/batches/import-preview.tsx`: Import parsing review UI.
- `components/batches/payslip-row-table.tsx`: Batch payslip rows and actions.
- `components/employees/employee-form.tsx`: Employee create/edit form.
- `components/employees/employee-table.tsx`: Employee list table.
- `components/history/history-table.tsx`: History search and result table.
- `components/payslip/payslip-page.tsx`: Printable payslip document.
- `components/payslip/print-actions.tsx`: Individual and batch print buttons.
- `components/providers/convex-client-provider.tsx`: Convex React provider and auth provider.
- `components/ui/*`: shadcn/ui components.
- `convex/schema.ts`: Convex schema and indexes.
- `convex/auth.ts`: Convex Auth setup.
- `convex/auth.config.ts`: Convex Auth JWT issuer config.
- `convex/http.ts`: Convex Auth HTTP routes.
- `convex/users.ts`: Current user profile query/mutation.
- `convex/employees.ts`: Employee queries and mutations.
- `convex/batches.ts`: Payroll batch queries and mutations.
- `convex/payslips.ts`: Payslip queries and mutations.
- `convex/printEvents.ts`: Print event mutation and print history queries.
- `lib/import/parse-payroll-file.ts`: Excel/CSV parser.
- `lib/payroll/validation.ts`: Shared validation rules.
- `lib/payroll/types.ts`: Frontend-facing payroll types.
- `lib/format.ts`: Currency/date/status formatting helpers.
- `tests/payroll/validation.test.ts`: Validation tests.
- `tests/import/parse-payroll-file.test.ts`: Import parser tests.
- `tests/e2e/print-layout.spec.ts`: One-page print layout smoke test.
- `tests/fixtures/payroll-sample.csv`: Sample import file.
- `tests/fixtures/payroll-missing-fields.csv`: Invalid import file.
- `playwright.config.ts`: Playwright config.
- `vitest.config.ts`: Vitest config.

---

### Task 1: Scaffold Next.js, shadcn/ui, Convex, And Test Tooling

**Files:**
- Create: `package.json`
- Create: `app/layout.tsx`
- Create: `app/page.tsx`
- Create: `app/globals.css`
- Create: `components/providers/convex-client-provider.tsx`
- Create: `components/app/app-shell.tsx`
- Create: `components/app/nav.tsx`
- Create: `convex/schema.ts`
- Create: `convex/auth.ts`
- Create: `convex/auth.config.ts`
- Create: `convex/http.ts`
- Create: `vitest.config.ts`
- Create: `playwright.config.ts`
- Create: `.env.local.example`

**Interfaces:**
- Consumes: Empty repository plus design spec.
- Produces: Runnable Next.js + shadcn/ui + Convex skeleton with `npm run dev`, `npm run typecheck`, `npm run test`, and `npm run test:e2e` scripts.

- [ ] **Step 1: Scaffold the app**

Run:

```bash
pnpm dlx shadcn@latest init -t next
```

Choose:

```text
Base color: Neutral
Use CSS variables: yes
React Server Components: yes
Import alias: @/*
```

Expected: Next.js project files are created in the current repository.

- [ ] **Step 2: Install required runtime and test dependencies**

Run:

```bash
pnpm add convex @convex-dev/auth xlsx date-fns lucide-react
pnpm add -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/jest-dom playwright
```

Expected: `package.json` includes Convex, auth, spreadsheet parsing, date formatting, icons, Vitest, and Playwright.

- [ ] **Step 3: Add shadcn components**

Run:

```bash
pnpm dlx shadcn@latest add button input label card table badge tabs dialog dropdown-menu select textarea toast sonner separator checkbox skeleton
```

Expected: `components/ui/*` contains the listed shadcn components.

- [ ] **Step 4: Configure package scripts**

Edit `package.json` so scripts include:

```json
{
  "scripts": {
    "dev": "next dev",
    "convex:dev": "convex dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:e2e": "playwright test"
  }
}
```

- [ ] **Step 5: Add Vitest config**

Create `vitest.config.ts`:

```ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  test: {
    environment: "jsdom",
    globals: true,
    include: ["tests/**/*.test.ts", "tests/**/*.test.tsx"],
  },
  resolve: {
    alias: {
      "@": new URL(".", import.meta.url).pathname,
    },
  },
});
```

- [ ] **Step 6: Add Playwright config**

Create `playwright.config.ts`:

```ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests/e2e",
  timeout: 30_000,
  retries: 0,
  use: {
    baseURL: "http://127.0.0.1:3000",
    trace: "on-first-retry",
  },
  webServer: {
    command: "pnpm dev",
    url: "http://127.0.0.1:3000",
    reuseExistingServer: true,
    timeout: 120_000,
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
  ],
});
```

- [ ] **Step 7: Add Convex Auth files**

Create `convex/auth.ts`:

```ts
import { convexAuth } from "@convex-dev/auth/server";
import Password from "@convex-dev/auth/providers/Password";

export const { auth, signIn, signOut, store, isAuthenticated } = convexAuth({
  providers: [Password],
});
```

Create `convex/auth.config.ts`:

```ts
export default {
  providers: [{ domain: process.env.CONVEX_SITE_URL, applicationID: "convex" }],
};
```

Create `convex/http.ts`:

```ts
import { httpRouter } from "convex/server";
import { auth } from "./auth";

const http = httpRouter();
auth.addHttpRoutes(http);

export default http;
```

- [ ] **Step 8: Add initial schema**

Create `convex/schema.ts`:

```ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";
import { authTables } from "@convex-dev/auth/server";

export default defineSchema({
  ...authTables,
  users: defineTable({
    name: v.string(),
    email: v.string(),
    role: v.union(v.literal("supervisor"), v.literal("admin")),
    tokenIdentifier: v.string(),
    createdAt: v.number(),
  }).index("by_tokenIdentifier", ["tokenIdentifier"]),
});
```

- [ ] **Step 9: Add root provider**

Create `components/providers/convex-client-provider.tsx`:

```tsx
"use client";

import { ConvexAuthNextjsProvider } from "@convex-dev/auth/nextjs";
import { ConvexReactClient } from "convex/react";
import { ReactNode } from "react";

const convex = new ConvexReactClient(process.env.NEXT_PUBLIC_CONVEX_URL!);

export function ConvexClientProvider({ children }: { children: ReactNode }) {
  return (
    <ConvexAuthNextjsProvider client={convex}>
      {children}
    </ConvexAuthNextjsProvider>
  );
}
```

- [ ] **Step 10: Wire layout**

Create `app/layout.tsx`:

```tsx
import type { Metadata } from "next";
import "./globals.css";
import { ConvexClientProvider } from "@/components/providers/convex-client-provider";

export const metadata: Metadata = {
  title: "TGMCI Payslip Generator",
  description: "Shared payroll batch and payslip printing tool.",
};

export default function RootLayout({ children }: Readonly<{ children: React.ReactNode }>) {
  return (
    <html lang="en">
      <body>
        <ConvexClientProvider>{children}</ConvexClientProvider>
      </body>
    </html>
  );
}
```

Create `app/page.tsx`:

```tsx
export default function DashboardPage() {
  return <main className="p-6">TGMCI Payslip Generator</main>;
}
```

- [ ] **Step 11: Add environment example**

Create `.env.local.example`:

```bash
NEXT_PUBLIC_CONVEX_URL=
CONVEX_DEPLOYMENT=
```

- [ ] **Step 12: Verify scaffold**

Run:

```bash
pnpm typecheck
pnpm test
```

Expected: both commands pass. If `pnpm test` reports no test files, add a temporary `tests/smoke.test.ts` with `expect(true).toBe(true)` and commit it with the scaffold.

- [ ] **Step 13: Commit**

Run:

```bash
git add .
git commit -m "chore: scaffold payslip app"
```

---

### Task 2: Add Payroll Types And Validation Rules

**Files:**
- Create: `lib/payroll/types.ts`
- Create: `lib/payroll/validation.ts`
- Create: `tests/payroll/validation.test.ts`

**Interfaces:**
- Consumes: TypeScript/Vitest setup from Task 1.
- Produces: `validatePayslipDraft(input: PayslipDraftInput): ValidationResult` and shared payroll types for import, UI, and Convex calls.

- [ ] **Step 1: Write failing validation tests**

Create `tests/payroll/validation.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { validatePayslipDraft } from "@/lib/payroll/validation";
import type { PayslipDraftInput } from "@/lib/payroll/types";

const baseInput: PayslipDraftInput = {
  employeeCode: "EMP-001",
  employeeName: "Maria Santos",
  department: "Accounting",
  position: "Staff",
  grossPay: 15000,
  totalDeductions: 2500,
  netPay: 12500,
  earnings: [{ label: "Basic Pay", amount: 15000 }],
  deductions: [{ label: "SSS", amount: 2500 }],
};

describe("validatePayslipDraft", () => {
  it("accepts a complete valid payslip", () => {
    expect(validatePayslipDraft(baseInput)).toEqual({ errors: [], warnings: [] });
  });

  it("requires employee name", () => {
    const result = validatePayslipDraft({ ...baseInput, employeeName: "" });
    expect(result.errors).toContain("Employee name is required.");
  });

  it("rejects negative net pay", () => {
    const result = validatePayslipDraft({ ...baseInput, netPay: -1 });
    expect(result.errors).toContain("Net pay cannot be negative.");
  });

  it("warns when deductions are greater than gross pay", () => {
    const result = validatePayslipDraft({ ...baseInput, totalDeductions: 16000 });
    expect(result.warnings).toContain("Total deductions are greater than gross pay.");
  });

  it("requires at least one earning line", () => {
    const result = validatePayslipDraft({ ...baseInput, earnings: [] });
    expect(result.errors).toContain("At least one earning line is required.");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run:

```bash
pnpm test tests/payroll/validation.test.ts
```

Expected: FAIL because `@/lib/payroll/validation` does not exist.

- [ ] **Step 3: Add payroll types**

Create `lib/payroll/types.ts`:

```ts
export type MoneyLine = {
  label: string;
  amount: number;
};

export type PayslipDraftInput = {
  employeeCode: string;
  employeeName: string;
  department: string;
  position: string;
  grossPay: number;
  totalDeductions: number;
  netPay: number;
  earnings: MoneyLine[];
  deductions: MoneyLine[];
};

export type ValidationResult = {
  errors: string[];
  warnings: string[];
};
```

- [ ] **Step 4: Add validation implementation**

Create `lib/payroll/validation.ts`:

```ts
import type { PayslipDraftInput, ValidationResult } from "@/lib/payroll/types";

export function validatePayslipDraft(input: PayslipDraftInput): ValidationResult {
  const errors: string[] = [];
  const warnings: string[] = [];

  if (!input.employeeName.trim()) errors.push("Employee name is required.");
  if (!Number.isFinite(input.grossPay)) errors.push("Gross pay must be a valid number.");
  if (!Number.isFinite(input.totalDeductions)) errors.push("Total deductions must be a valid number.");
  if (!Number.isFinite(input.netPay)) errors.push("Net pay must be a valid number.");
  if (input.netPay < 0) errors.push("Net pay cannot be negative.");
  if (input.earnings.length === 0) errors.push("At least one earning line is required.");
  if (input.totalDeductions > input.grossPay) warnings.push("Total deductions are greater than gross pay.");

  return { errors, warnings };
}
```

- [ ] **Step 5: Verify validation tests pass**

Run:

```bash
pnpm test tests/payroll/validation.test.ts
pnpm typecheck
```

Expected: PASS.

- [ ] **Step 6: Commit**

Run:

```bash
git add lib/payroll tests/payroll
git commit -m "test: add payroll validation rules"
```

---

### Task 3: Add Excel/CSV Import Parser

**Files:**
- Create: `lib/import/parse-payroll-file.ts`
- Create: `tests/import/parse-payroll-file.test.ts`
- Create: `tests/fixtures/payroll-sample.csv`
- Create: `tests/fixtures/payroll-missing-fields.csv`

**Interfaces:**
- Consumes: `PayslipDraftInput` and `validatePayslipDraft`.
- Produces: `parsePayrollRows(csvText: string): ParsedPayrollImport` for import preview.

- [ ] **Step 1: Add CSV fixtures**

Create `tests/fixtures/payroll-sample.csv`:

```csv
Employee ID,Name,Department,Position,Basic Pay,Overtime,Deductions,Net Pay
EMP-001,Maria Santos,Accounting,Staff,15000,500,2500,13000
EMP-002,Juan Dela Cruz,Operations,Driver,14000,0,1800,12200
```

Create `tests/fixtures/payroll-missing-fields.csv`:

```csv
Employee ID,Name,Department,Position,Basic Pay,Overtime,Deductions,Net Pay
EMP-003,,Operations,Staff,12000,0,1000,11000
```

- [ ] **Step 2: Write failing parser tests**

Create `tests/import/parse-payroll-file.test.ts`:

```ts
import { readFileSync } from "node:fs";
import { join } from "node:path";
import { describe, expect, it } from "vitest";
import { parsePayrollRows } from "@/lib/import/parse-payroll-file";

function fixture(name: string) {
  return readFileSync(join(process.cwd(), "tests/fixtures", name), "utf8");
}

describe("parsePayrollRows", () => {
  it("maps a valid CSV into payslip drafts", () => {
    const result = parsePayrollRows(fixture("payroll-sample.csv"));
    expect(result.rows).toHaveLength(2);
    expect(result.rows[0].draft.employeeName).toBe("Maria Santos");
    expect(result.rows[0].draft.grossPay).toBe(15500);
    expect(result.rows[0].draft.netPay).toBe(13000);
    expect(result.rows[0].errors).toEqual([]);
  });

  it("returns row-level validation errors", () => {
    const result = parsePayrollRows(fixture("payroll-missing-fields.csv"));
    expect(result.rows[0].errors).toContain("Employee name is required.");
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run:

```bash
pnpm test tests/import/parse-payroll-file.test.ts
```

Expected: FAIL because parser does not exist.

- [ ] **Step 4: Implement parser**

Create `lib/import/parse-payroll-file.ts`:

```ts
import { read, utils } from "xlsx";
import { validatePayslipDraft } from "@/lib/payroll/validation";
import type { PayslipDraftInput } from "@/lib/payroll/types";

export type ParsedPayrollRow = {
  rowNumber: number;
  draft: PayslipDraftInput;
  errors: string[];
  warnings: string[];
};

export type ParsedPayrollImport = {
  rows: ParsedPayrollRow[];
};

type RawPayrollRow = Record<string, string | number | undefined>;

function text(value: unknown): string {
  return String(value ?? "").trim();
}

function money(value: unknown): number {
  const normalized = String(value ?? "0").replace(/,/g, "").trim();
  const parsed = Number(normalized);
  return Number.isFinite(parsed) ? parsed : 0;
}

export function parsePayrollRows(csvText: string): ParsedPayrollImport {
  const workbook = read(csvText, { type: "string" });
  const sheet = workbook.Sheets[workbook.SheetNames[0]];
  const rawRows = utils.sheet_to_json<RawPayrollRow>(sheet, { defval: "" });

  const rows = rawRows.map((row, index) => {
    const basicPay = money(row["Basic Pay"]);
    const overtime = money(row["Overtime"]);
    const deductions = money(row["Deductions"]);
    const draft: PayslipDraftInput = {
      employeeCode: text(row["Employee ID"]),
      employeeName: text(row["Name"]),
      department: text(row["Department"]),
      position: text(row["Position"]),
      grossPay: basicPay + overtime,
      totalDeductions: deductions,
      netPay: money(row["Net Pay"]),
      earnings: [
        { label: "Basic Pay", amount: basicPay },
        { label: "Overtime", amount: overtime },
      ].filter((line) => line.amount !== 0),
      deductions: deductions === 0 ? [] : [{ label: "Deductions", amount: deductions }],
    };

    const validation = validatePayslipDraft(draft);
    return {
      rowNumber: index + 2,
      draft,
      errors: validation.errors,
      warnings: validation.warnings,
    };
  });

  return { rows };
}
```

- [ ] **Step 5: Verify parser tests pass**

Run:

```bash
pnpm test tests/import/parse-payroll-file.test.ts tests/payroll/validation.test.ts
pnpm typecheck
```

Expected: PASS.

- [ ] **Step 6: Commit**

Run:

```bash
git add lib/import tests/import tests/fixtures
git commit -m "feat: add payroll import parser"
```

---

### Task 4: Implement Convex Data Model And Mutations

**Files:**
- Modify: `convex/schema.ts`
- Create: `convex/users.ts`
- Create: `convex/employees.ts`
- Create: `convex/batches.ts`
- Create: `convex/payslips.ts`
- Create: `convex/printEvents.ts`

**Interfaces:**
- Consumes: Convex setup from Task 1.
- Produces: Public Convex APIs for employees, batches, payslips, and print events with validators and indexed queries.

- [ ] **Step 1: Expand schema**

Modify `convex/schema.ts`:

```ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";
import { authTables } from "@convex-dev/auth/server";

const moneyLine = v.object({
  label: v.string(),
  amount: v.number(),
});

const employeeSnapshot = v.object({
  employeeCode: v.string(),
  employeeName: v.string(),
  department: v.string(),
  position: v.string(),
});

export default defineSchema({
  ...authTables,
  users: defineTable({
    name: v.string(),
    email: v.string(),
    role: v.union(v.literal("supervisor"), v.literal("admin")),
    tokenIdentifier: v.string(),
    createdAt: v.number(),
  }).index("by_tokenIdentifier", ["tokenIdentifier"]),
  employees: defineTable({
    employeeCode: v.string(),
    name: v.string(),
    department: v.string(),
    position: v.string(),
    status: v.union(v.literal("active"), v.literal("inactive")),
    createdAt: v.number(),
    updatedAt: v.number(),
  })
    .index("by_employeeCode", ["employeeCode"])
    .index("by_name", ["name"])
    .index("by_department", ["department"]),
  payrollBatches: defineTable({
    name: v.string(),
    payPeriodStart: v.string(),
    payPeriodEnd: v.string(),
    status: v.union(v.literal("draft"), v.literal("ready"), v.literal("completed")),
    createdBy: v.string(),
    createdAt: v.number(),
    updatedAt: v.number(),
  })
    .index("by_payPeriodStart", ["payPeriodStart"])
    .index("by_status", ["status"])
    .index("by_createdBy", ["createdBy"]),
  payslips: defineTable({
    batchId: v.id("payrollBatches"),
    employeeId: v.id("employees"),
    employeeSnapshot,
    earnings: v.array(moneyLine),
    deductions: v.array(moneyLine),
    grossPay: v.number(),
    totalDeductions: v.number(),
    netPay: v.number(),
    status: v.union(v.literal("draft"), v.literal("finalized")),
    generatedBy: v.string(),
    finalizedAt: v.union(v.number(), v.null()),
    printCount: v.number(),
    lastPrintTriggeredAt: v.union(v.number(), v.null()),
    createdAt: v.number(),
    updatedAt: v.number(),
  })
    .index("by_batchId", ["batchId"])
    .index("by_employeeId", ["employeeId"])
    .index("by_batchId_and_status", ["batchId", "status"])
    .index("by_employeeId_and_batchId", ["employeeId", "batchId"]),
  printEvents: defineTable({
    payslipId: v.id("payslips"),
    batchId: v.id("payrollBatches"),
    printedBy: v.string(),
    printedAt: v.number(),
    mode: v.union(v.literal("individual"), v.literal("batch")),
    printCountAfterEvent: v.number(),
  })
    .index("by_payslipId", ["payslipId"])
    .index("by_batchId", ["batchId"])
    .index("by_printedBy", ["printedBy"])
    .index("by_printedAt", ["printedAt"]),
});
```

- [ ] **Step 2: Add authenticated user helper**

Create `convex/users.ts`:

```ts
import { v } from "convex/values";
import { mutation, query } from "./_generated/server";

export const current = query({
  args: {},
  returns: v.union(
    v.object({
      _id: v.id("users"),
      _creationTime: v.number(),
      name: v.string(),
      email: v.string(),
      role: v.union(v.literal("supervisor"), v.literal("admin")),
      tokenIdentifier: v.string(),
      createdAt: v.number(),
    }),
    v.null(),
  ),
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) return null;
    return await ctx.db
      .query("users")
      .withIndex("by_tokenIdentifier", (q) => q.eq("tokenIdentifier", identity.tokenIdentifier))
      .unique();
  },
});

export const ensureCurrent = mutation({
  args: { name: v.string(), email: v.string() },
  returns: v.id("users"),
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Authentication required.");
    const existing = await ctx.db
      .query("users")
      .withIndex("by_tokenIdentifier", (q) => q.eq("tokenIdentifier", identity.tokenIdentifier))
      .unique();
    if (existing) return existing._id;
    return await ctx.db.insert("users", {
      name: args.name,
      email: args.email,
      role: "supervisor",
      tokenIdentifier: identity.tokenIdentifier,
      createdAt: Date.now(),
    });
  },
});
```

- [ ] **Step 3: Add employee functions**

Create `convex/employees.ts` with these public functions:

```ts
export const list: RegisteredQuery<
  "public",
  { search?: string },
  Array<{
    _id: Id<"employees">;
    _creationTime: number;
    employeeCode: string;
    name: string;
    department: string;
    position: string;
    status: "active" | "inactive";
    createdAt: number;
    updatedAt: number;
  }>
>;

export const get: RegisteredQuery<
  "public",
  { employeeId: Id<"employees"> },
  Doc<"employees"> | null
>;

export const upsert: RegisteredMutation<
  "public",
  {
    employeeCode: string;
    name: string;
    department: string;
    position: string;
    status?: "active" | "inactive";
  },
  Id<"employees">
>;
```

Implementation rules:

- Use object-form Convex functions.
- Add `args` and `returns` validators on every function.
- Require `ctx.auth.getUserIdentity()` in `upsert`.
- `list` uses `by_name` when `search` is provided and otherwise uses `by_department` with `.take(100)` for a bounded V1 list.
- `upsert` matches by `employeeCode`; if no match exists, insert an active employee.

- [ ] **Step 4: Add batch functions**

Create `convex/batches.ts` with three public functions: `list`, `create`, and `get`. The `list` query must use `.withIndex("by_payPeriodStart")`, order descending, and `.take(50)`. The `create` mutation must require authentication, accept `name`, `payPeriodStart`, and `payPeriodEnd`, and insert `status: "draft"`. The `get` query must accept `batchId: v.id("payrollBatches")` and return the matching batch or `null`.

- [ ] **Step 5: Add payslip functions**

Create `convex/payslips.ts` with six public functions: `listByBatch`, `get`, `createDraft`, `updateDraft`, `finalize`, and `searchHistory`.

Rules:

- `listByBatch` uses `by_batchId`.
- `get` uses `ctx.db.get(args.payslipId)`.
- `createDraft` requires auth, upserts employee by employee code first, then inserts a draft payslip.
- `updateDraft` rejects edits when `status === "finalized"`.
- `finalize` sets `status: "finalized"` and `finalizedAt: Date.now()`.
- `searchHistory` starts with `.withIndex("by_employeeId")` when employee ID is provided; otherwise use a bounded `.query("payslips").order("desc").take(100)` for V1.

- [ ] **Step 6: Add print event mutation**

Create `convex/printEvents.ts` with two public functions: `recordPrintTriggered` and `listByPayslip`.

Rules:

- `recordPrintTriggered` requires auth.
- It only accepts finalized payslips.
- It inserts a `printEvents` row with mode `"individual"` or `"batch"`.
- It increments `payslips.printCount`.
- It sets `lastPrintTriggeredAt`.

- [ ] **Step 7: Verify Convex code**

Run:

```bash
pnpm typecheck
CONVEX_AGENT_MODE=anonymous pnpm convex:dev --once
```

Expected: TypeScript passes and Convex functions push to an anonymous dev deployment.

- [ ] **Step 8: Commit**

Run:

```bash
git add convex
git commit -m "feat: add convex payroll data model"
```

---

### Task 5: Build App Shell, Dashboard, Employees, And Batches

**Files:**
- Modify: `app/layout.tsx`
- Modify: `app/page.tsx`
- Create: `app/employees/page.tsx`
- Create: `app/batches/page.tsx`
- Create: `components/app/app-shell.tsx`
- Create: `components/app/nav.tsx`
- Create: `components/employees/employee-table.tsx`
- Create: `components/employees/employee-form.tsx`
- Create: `components/batches/batch-table.tsx`
- Create: `components/batches/batch-form.tsx`
- Create: `lib/format.ts`

**Interfaces:**
- Consumes: Convex `employees.*` and `batches.*` functions.
- Produces: Navigable supervisor UI for dashboard, employees, and batch creation.

- [ ] **Step 1: Add formatting helpers**

Create `lib/format.ts`:

```ts
import { format } from "date-fns";

export function formatCurrency(value: number) {
  return new Intl.NumberFormat("en-PH", {
    style: "currency",
    currency: "PHP",
  }).format(value);
}

export function formatDateLabel(value: number | string) {
  return format(new Date(value), "MMM d, yyyy");
}
```

- [ ] **Step 2: Build app shell**

Create `components/app/nav.tsx` with links to `/`, `/employees`, `/batches`, and `/history`. Use lucide icons and shadcn `Button` variants.

Create `components/app/app-shell.tsx` that renders a left sidebar on desktop, top navigation on mobile, and a constrained main content area.

- [ ] **Step 3: Wrap routes in app shell**

Modify `app/layout.tsx` so authenticated app routes render inside `AppShell`. Keep sign-in route visually simple.

- [ ] **Step 4: Build dashboard**

Modify `app/page.tsx` to show:

- Recent batches.
- Finalized but unprinted count.
- Draft count.
- Recently printed payslips.
- Primary action to create/import a batch.

Use shadcn `Card`, `Button`, `Badge`, and `Table`.

- [ ] **Step 5: Build employees page**

Create `app/employees/page.tsx`, `components/employees/employee-table.tsx`, and `components/employees/employee-form.tsx`. The page must support listing employees and creating/updating an employee through Convex.

- [ ] **Step 6: Build batches page**

Create `app/batches/page.tsx`, `components/batches/batch-table.tsx`, and `components/batches/batch-form.tsx`. The page must create payroll batches and list the latest 50 batches.

- [ ] **Step 7: Verify UI**

Run:

```bash
pnpm typecheck
pnpm test
pnpm dev
```

Open `http://localhost:3000`, `/employees`, and `/batches`. Expected: routes load without runtime overlay errors.

- [ ] **Step 8: Commit**

Run:

```bash
git add app components lib/format.ts
git commit -m "feat: add supervisor app shell"
```

---

### Task 6: Build Batch Detail, Import Preview, And Manual Editing

**Files:**
- Create: `app/batches/[batchId]/page.tsx`
- Create: `components/batches/import-preview.tsx`
- Create: `components/batches/payslip-row-table.tsx`
- Create: `components/batches/manual-payslip-dialog.tsx`
- Modify: `lib/import/parse-payroll-file.ts`
- Modify: `convex/payslips.ts`

**Interfaces:**
- Consumes: `parsePayrollRows`, `validatePayslipDraft`, Convex batch and payslip APIs.
- Produces: Batch workspace where supervisors can import rows, review errors, add/edit drafts, and finalize payslips.

- [ ] **Step 1: Build batch detail route**

Create `app/batches/[batchId]/page.tsx` that loads `api.batches.get` and `api.payslips.listByBatch`. Show loading, missing batch, and loaded states.

- [ ] **Step 2: Build import preview component**

Create `components/batches/import-preview.tsx`:

- Accepts `.csv`, `.xls`, and `.xlsx` files.
- Reads file text/array buffer in the browser.
- Calls parser.
- Shows row number, employee code, name, gross pay, deductions, net pay, errors, and warnings.
- Disables confirm import when any row has errors.

- [ ] **Step 3: Add XLSX parsing support**

Modify `lib/import/parse-payroll-file.ts` to export:

```ts
export function parsePayrollWorkbook(input: ArrayBuffer | string): ParsedPayrollImport
```

Keep `parsePayrollRows(csvText)` as a test-friendly wrapper.

- [ ] **Step 4: Build payslip row table**

Create `components/batches/payslip-row-table.tsx` with status badges for draft/finalized/printed, actions for preview, edit draft, finalize, print, and row selection for batch print.

- [ ] **Step 5: Build manual payslip dialog**

Create `components/batches/manual-payslip-dialog.tsx` with fields for employee code, employee name, department, position, earnings, deductions, gross pay, total deductions, and net pay. Use validation before calling Convex.

- [ ] **Step 6: Confirm import writes drafts**

Wire confirm import to call `api.payslips.createDraft` once per valid row. Show a success toast with count inserted.

- [ ] **Step 7: Verify batch workflow**

Run:

```bash
pnpm test tests/import/parse-payroll-file.test.ts tests/payroll/validation.test.ts
pnpm typecheck
```

Manual QA:

```text
Create a batch.
Open batch detail.
Import tests/fixtures/payroll-sample.csv.
Confirm rows are shown as drafts.
Import tests/fixtures/payroll-missing-fields.csv.
Confirm import cannot be saved until error is fixed.
Create one manual draft.
Finalize one draft.
```

- [ ] **Step 8: Commit**

Run:

```bash
git add app/batches components/batches lib/import convex/payslips.ts
git commit -m "feat: add batch import workflow"
```

---

### Task 7: Build Payslip Preview, Individual Print, And Batch Print

**Files:**
- Create: `app/payslips/[payslipId]/page.tsx`
- Create: `components/payslip/payslip-page.tsx`
- Create: `components/payslip/print-actions.tsx`
- Modify: `app/batches/[batchId]/page.tsx`
- Modify: `app/globals.css`
- Create: `tests/e2e/print-layout.spec.ts`

**Interfaces:**
- Consumes: finalized payslips and `api.printEvents.recordPrintTriggered`.
- Produces: one-page print layout, individual print, batch print, and print-trigger persistence.

- [ ] **Step 1: Build printable payslip component**

Create `components/payslip/payslip-page.tsx`:

```tsx
import { formatCurrency } from "@/lib/format";

type MoneyLine = { label: string; amount: number };

export type PrintablePayslip = {
  employeeSnapshot: {
    employeeCode: string;
    employeeName: string;
    department: string;
    position: string;
  };
  earnings: MoneyLine[];
  deductions: MoneyLine[];
  grossPay: number;
  totalDeductions: number;
  netPay: number;
};

export function PayslipPage({ payslip }: { payslip: PrintablePayslip }) {
  return (
    <article className="payslip-page">
      <header className="payslip-header">
        <div>
          <p className="payslip-company">TGMCI</p>
          <h1>Payslip</h1>
        </div>
      </header>
      <section className="payslip-employee-grid">
        <div><span>Employee ID</span><strong>{payslip.employeeSnapshot.employeeCode}</strong></div>
        <div><span>Name</span><strong>{payslip.employeeSnapshot.employeeName}</strong></div>
        <div><span>Department</span><strong>{payslip.employeeSnapshot.department}</strong></div>
        <div><span>Position</span><strong>{payslip.employeeSnapshot.position}</strong></div>
      </section>
      <section className="payslip-columns">
        <div>
          <h2>Earnings</h2>
          {payslip.earnings.map((line) => (
            <div className="payslip-line" key={line.label}>
              <span>{line.label}</span><span>{formatCurrency(line.amount)}</span>
            </div>
          ))}
        </div>
        <div>
          <h2>Deductions</h2>
          {payslip.deductions.map((line) => (
            <div className="payslip-line" key={line.label}>
              <span>{line.label}</span><span>{formatCurrency(line.amount)}</span>
            </div>
          ))}
        </div>
      </section>
      <footer className="payslip-total">
        <div><span>Gross Pay</span><strong>{formatCurrency(payslip.grossPay)}</strong></div>
        <div><span>Total Deductions</span><strong>{formatCurrency(payslip.totalDeductions)}</strong></div>
        <div><span>Net Pay</span><strong>{formatCurrency(payslip.netPay)}</strong></div>
      </footer>
    </article>
  );
}
```

- [ ] **Step 2: Add print CSS**

Modify `app/globals.css`:

```css
@media print {
  body * {
    visibility: hidden;
  }

  .print-surface,
  .print-surface * {
    visibility: visible;
  }

  .print-surface {
    position: absolute;
    inset: 0;
  }

  .payslip-page {
    break-after: page;
    page-break-after: always;
  }

  .payslip-page:last-child {
    break-after: auto;
    page-break-after: auto;
  }
}

@page {
  size: A4;
  margin: 12mm;
}

.payslip-page {
  width: 186mm;
  min-height: 273mm;
  background: white;
  color: black;
  border: 1px solid #111827;
  padding: 12mm;
  font-family: Arial, sans-serif;
}

.payslip-header,
.payslip-line,
.payslip-total > div {
  display: flex;
  justify-content: space-between;
  gap: 16px;
}

.payslip-company {
  font-size: 12px;
  text-transform: uppercase;
  letter-spacing: 0;
}

.payslip-employee-grid,
.payslip-columns,
.payslip-total {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin-top: 18px;
}

.payslip-total {
  border-top: 1px solid #111827;
  padding-top: 12px;
}
```

- [ ] **Step 3: Build payslip preview route**

Create `app/payslips/[payslipId]/page.tsx` that loads `api.payslips.get`, renders `PayslipPage`, and shows `PrintActions`.

- [ ] **Step 4: Build print actions**

Create `components/payslip/print-actions.tsx` with:

- Individual print button.
- Batch print button when passed multiple payslip IDs.
- `window.print()` after awaiting `api.printEvents.recordPrintTriggered`.
- Toast explaining that print status means the print dialog opened.

- [ ] **Step 5: Wire batch print from batch detail**

Modify `app/batches/[batchId]/page.tsx` to render selected finalized payslips in a hidden `.print-surface` and call `recordPrintTriggered` for each selected payslip before `window.print()`.

- [ ] **Step 6: Add print layout smoke test**

Create `tests/e2e/print-layout.spec.ts`:

```ts
import { expect, test } from "@playwright/test";

test("dashboard renders without horizontal overflow", async ({ page }) => {
  await page.goto("/");
  const overflow = await page.evaluate(() => document.documentElement.scrollWidth > document.documentElement.clientWidth);
  expect(overflow).toBe(false);
});
```

- [ ] **Step 7: Verify print workflow**

Run:

```bash
pnpm typecheck
pnpm test
pnpm test:e2e
```

Manual QA:

```text
Open a finalized payslip.
Click Print.
Confirm browser print dialog opens.
Cancel print.
Confirm print count increments.
Select multiple finalized payslips in a batch.
Click Batch Print.
Confirm each payslip has its own page in print preview.
```

- [ ] **Step 8: Commit**

Run:

```bash
git add app components/payslip app/globals.css tests/e2e
git commit -m "feat: add payslip printing"
```

---

### Task 8: Build Searchable History And Final Verification

**Files:**
- Create: `app/history/page.tsx`
- Create: `components/history/history-table.tsx`
- Modify: `convex/payslips.ts`
- Modify: `convex/printEvents.ts`
- Modify: `docs/superpowers/specs/2026-08-03-payslip-generator-design.md`

**Interfaces:**
- Consumes: payslip and print-event records.
- Produces: searchable history UI and final V1 verification notes.

- [ ] **Step 1: Build history table**

Create `components/history/history-table.tsx` with filters for employee name text, employee code text, pay period text, status, and print status. Use shadcn `Input`, `Select`, `Table`, and `Badge`.

- [ ] **Step 2: Build history page**

Create `app/history/page.tsx` that renders `HistoryTable`, loads `api.payslips.searchHistory`, and links each result to `/payslips/[payslipId]`.

- [ ] **Step 3: Tighten history query**

Modify `convex/payslips.ts` so `searchHistory` returns at most 100 records and maps only UI-needed fields:

```ts
{
  _id,
  employeeSnapshot,
  grossPay,
  totalDeductions,
  netPay,
  status,
  printCount,
  lastPrintTriggeredAt,
  createdAt,
  updatedAt
}
```

- [ ] **Step 4: Add final V1 checklist to spec**

Append a `## V1 Completion Checklist` section to `docs/superpowers/specs/2026-08-03-payslip-generator-design.md`:

```md
## V1 Completion Checklist

- Supervisor can sign in.
- Supervisor can create a payroll batch.
- Supervisor can import valid Excel/CSV payroll rows.
- Import preview blocks rows with validation errors.
- Supervisor can manually create a payslip draft.
- Supervisor can edit a draft.
- Supervisor can finalize a payslip.
- Supervisor can print one finalized payslip.
- Supervisor can batch print finalized payslips with one payslip per page.
- Print-trigger events are recorded automatically.
- Supervisor can search history and reprint.
```

- [ ] **Step 5: Run full verification**

Run:

```bash
pnpm typecheck
pnpm test
pnpm test:e2e
CONVEX_AGENT_MODE=anonymous pnpm convex:dev --once
pnpm build
git diff --check
```

Expected: all commands pass.

- [ ] **Step 6: Commit**

Run:

```bash
git add app/history components/history convex docs/superpowers/specs/2026-08-03-payslip-generator-design.md
git commit -m "feat: add payslip history"
```

- [ ] **Step 7: Push**

Run:

```bash
git push
```

Expected: branch `main` pushes to `origin/main`.

---

## Self-Review

Spec coverage:

- Supervisor login: Task 1 and Task 4.
- Employee master list: Task 4 and Task 5.
- Payroll batches by pay period: Task 4, Task 5, and Task 6.
- Excel/CSV import preview: Task 3 and Task 6.
- Manual payslip entry/editing: Task 6.
- Batch detail validation/statuses: Task 6.
- One-page payslip preview: Task 7.
- Finalize before printing: Task 4, Task 6, and Task 7.
- Individual print and batch print: Task 7.
- Automatic print tracking: Task 4 and Task 7.
- Persistent searchable history: Task 4 and Task 8.
- Basic validation: Task 2, Task 3, and Task 6.

Known implementation inputs still needed from Ryan before polishing the print page:

- Sample payslip PDF/image template.
- Sample company payroll Excel/CSV file.
- Required company payslip fields.
- Whether employee ID is always available.
- Whether V1 needs PDF download in addition to browser printing.
