# Design Spec: Dominican Republic (DGII) E-Invoicing Integration

## Overview
This document specifies the integration of the Dominican Republic (DGII) electronic invoicing (e-CF) providers into Lago. The user wants to support two different libraries/processes for `DO`:
1. `ecf_dgii` (represented as `ecf_ssd` in the database) with display name **"DGII ECF - SSD"**.
2. `dgii-ecf` (represented as `dgii_ecf` in the database) with display name **"DGII ECF"**.

---

## Requirements & Scope

### 1. Backend Verification
- Ensure the database migration adding `einvoicing_provider` to `billing_entities` is executed.
- Ensure the whitelisted countries for e-invoicing include `"DO"` (Dominican Republic).
- Confirm the `EINVOICING_PROVIDERS` mapping handles:
  - `ecf_ssd`
  - `dgii_ecf`

### 2. Frontend Configuration
- Update the country whitelist constant to include `"DO"`.
- Define provider options per country:
  - For `DO`:
    - `ecf_ssd` ➔ **"DGII ECF - SSD"**
    - `dgii_ecf` ➔ **"DGII ECF"**
  - For `FR`:
    - `factur_x` ➔ **"Factur-X"**
    - `ubl` ➔ **"UBL"**

### 3. Frontend UI Updates
- **GraphQL:** Update the `BillingEntityItem` fragment in `useCreateEditBillingEntity.ts` to request and send the `einvoicingProvider` field.
- **Form Component:**
  - Update `BillingEntityCreateEdit.tsx` to handle `einvoicingProvider` in the Formik state.
  - Implement a conditional selector (dropdown) underneath the `einvoicing` switch. It should only be visible when `einvoicing` is `true`.
  - Automatically pre-select the default provider when `einvoicing` is enabled (`ecf_ssd` for `DO` and `factur_x` for `FR`).
- **Details Section:**
  - Update `InformationBlock.tsx` to display the selected e-invoicing provider name in the general settings view.

---

## Verification Plan

### Database & GraphQL Schema
1. Run `rails db:migrate` in the backend container.
2. Regenerate TypeScript GraphQL types using `pnpm codegen` in the frontend directory.

### UI Validation
1. Create/edit a Billing Entity.
2. Select country **Dominican Republic (DO)**.
3. Toggle "Issue compliant documents" (E-invoicing).
4. Verify that the provider dropdown appears and defaults to **"DGII ECF - SSD"**.
5. Switch to **"DGII ECF"** and save.
6. Verify that the general settings view displaying the entity info correctly lists the chosen provider.
