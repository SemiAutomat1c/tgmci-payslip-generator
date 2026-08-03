# Payslip Generator Design

## Purpose

Build a shared internal payslip generator for Ryan's OJT supervisors. Supervisors should be able to import payroll data from Excel/CSV, manually enter or correct payslip details, preview a one-page payslip based on the company's existing PDF/image template, print individual or batch payslips, and search persistent payslip history later.

## Recommended Stack

- Next.js for the web app UI.
- shadcn/ui and Tailwind CSS for the supervisor interface components.
- Convex for the shared backend, live data, users, employees, payroll batches, payslip history, and print events.
- Convex Auth for supervisor login.
- HTML/CSS print layout recreated from the company's existing PDF/image payslip template.
- Client-side Excel/CSV parsing for import preview before saving rows to Convex.

This should be a full but small web app, not Google Apps Script. The project needs a polished UI, multiple supervisors/devices, persistent history, and batch printing. Convex keeps the backend simpler than building a separate REST API and database. shadcn/ui should be used for app chrome, forms, dialogs, tables, filters, badges, and toasts, while the payslip paper itself should use custom HTML/CSS to match the company template.

## V1 Scope

V1 includes the workflow supervisors need to use the tool in daily payroll work:

- Supervisor login.
- Employee master list with employee ID, name, department, and position.
- Payroll batches organized by pay period.
- Excel/CSV import with preview before saving.
- Manual payslip entry and editing.
- Batch detail screen with validation issues and payslip statuses.
- One-page payslip preview matching the provided PDF/image template.
- Finalize payslip records before printing.
- Individual print.
- Batch print with one payslip per page.
- Automatic print tracking when the print dialog is triggered.
- Persistent searchable history.
- Basic validation for required fields, duplicates, negative net pay, and unusual values.

## V1.5 Scope

These features are valuable but should wait until V1 works end to end:

- Full audit log with edit reasons.
- Role permissions for admin versus supervisor.
- Template manager for switching between payslip layouts.
- Batch summary reports.
- Export history to CSV.
- Bulk finalize after review.
- Archive old batches while keeping them searchable.

## Core Workflow

1. Supervisor logs in.
2. Supervisor creates a payroll batch for a pay period.
3. Supervisor imports an Excel/CSV file or manually adds payslip rows.
4. The app previews imported rows and flags missing fields, duplicate employees in the same batch, negative net pay, and unusual amounts.
5. Supervisor confirms the import.
6. The app creates or updates employee master records when a new employee ID or name appears.
7. Supervisor reviews the batch table and edits any draft payslip.
8. Supervisor opens a one-page payslip preview.
9. Supervisor finalizes a payslip when the details are ready.
10. Supervisor prints one payslip or batch prints selected/all finalized payslips.
11. The app records print events automatically when the print dialog opens.
12. Supervisor can search history and reprint old payslips later.

## Screens

### Dashboard

Shows recent payroll batches, pending drafts, finalized but unprinted payslips, recently printed payslips, and quick actions for creating a batch or importing a file.

### Employees

Searchable employee master list. V1 should keep employee data simple: employee ID, name, department, position, and active/inactive status.

### Batches

List of payroll batches by pay period. Each batch shows pay period, status, total payslips, draft count, finalized count, printed count, and last updated time.

### Batch Detail

Main work screen for supervisors. It shows all payslips in a batch, validation warnings, status filters, import actions, manual add, edit, finalize, individual print, and batch print.

### Payslip Preview

Print-focused preview screen. It renders one payslip per page using an HTML/CSS recreation of the company's PDF/image template. It should include actions for edit, finalize, print, download PDF if supported, and reprint from history.

### History

Searchable history of generated payslips. Filters should include employee name, employee ID, pay period, batch, department, status, generated date, and print status.

## Data Model

### users

Stores app user profile data linked to Convex Auth identity.

Important fields:

- name
- email
- role
- createdAt

### employees

Stores reusable employee information.

Important fields:

- employeeId
- name
- department
- position
- status
- createdAt
- updatedAt

Indexes:

- by_employeeId
- by_name
- by_department

### payrollBatches

Stores one payroll period/import group.

Important fields:

- name
- payPeriodStart
- payPeriodEnd
- status
- createdBy
- createdAt
- updatedAt

Indexes:

- by_payPeriodStart
- by_status
- by_createdBy

### payslips

Stores generated payslip data for one employee in one batch.

Important fields:

- batchId
- employeeId
- employeeSnapshot
- earnings
- deductions
- grossPay
- totalDeductions
- netPay
- status
- generatedBy
- finalizedAt
- createdAt
- updatedAt

Indexes:

- by_batchId
- by_employeeId
- by_batchId_and_status
- by_employeeId_and_batchId

### printEvents

Stores automatic print tracking events.

Important fields:

- payslipId
- batchId
- printedBy
- printedAt
- mode
- printCountAfterEvent

Indexes:

- by_payslipId
- by_batchId
- by_printedBy
- by_printedAt

### importRows

Optional V1 table if detailed import troubleshooting is needed. Stores raw imported row data, mapping result, validation errors, and whether the row was accepted.

## Print Behavior

Printing automatically records a print event when the app opens the browser print dialog. The app cannot reliably confirm that a physical printer succeeded, so the backend should store this as a print-trigger event. The UI can label the payslip as Printed for supervisor convenience.

For batch printing, the app renders selected finalized payslips in a print-only layout with a page break after every payslip. Each payslip must fit on one page.

Reprinting creates another print event and increments the visible print count.

## Validation And Error Handling

The app should prevent or warn about:

- Missing employee name.
- Missing employee ID when the company requires IDs.
- Duplicate employee in the same batch.
- Missing gross pay, deductions, or net pay.
- Negative net pay.
- Deductions greater than gross pay.
- Invalid pay period dates.
- Import columns that cannot be mapped.

Import should use a preview-first flow. No imported row should be saved silently without giving the supervisor a chance to review errors and warnings.

## Testing Strategy

V1 should include focused tests for:

- Import parsing and column mapping.
- Validation logic.
- Batch creation and payslip creation.
- Finalize behavior.
- Print event creation.
- History filtering.
- One-page print layout smoke checks for representative payslip data.

Manual QA should include:

- Import a clean file.
- Import a file with missing fields.
- Manually create a payslip.
- Edit a draft.
- Finalize a payslip.
- Print one payslip.
- Batch print multiple payslips.
- Reprint from history.
- Search by employee and pay period.

## Open Inputs Needed

Before implementation, Ryan needs to provide or confirm:

- A sample payslip PDF/image template.
- A sample Excel/CSV payroll file.
- Required fields in the company payslip.
- Whether employee ID is always available.
- Whether downloading PDFs is required in V1 or printing is enough.
