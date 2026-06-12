# Dominican Republic (DGII) E-Invoicing UI Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integrate the frontend interface to allow selection between the two DGII electronic invoicing providers ("DGII ECF - SSD" and "DGII ECF") and French providers, ensuring full sync with backend GraphQL.

**Architecture:** Use Formik for form state management, `ComboBoxField` for dropdown selection, and Apollo GraphQL queries/mutations to save/receive settings.

**Tech Stack:** React, TypeScript, Formik, Apollo Client, Ruby on Rails (backend).

---

### Task 1: Execute Rails Migrations
Execute pending migrations on the backend to expose the `einvoicing_provider` column.

**Files:**
- Modify: None (run migration command)

- [ ] **Step 1: Run rails migration**
  Run: `docker compose exec api bundle exec rails db:migrate` or equivalent environment command.
  Expected: Successful migration execution adding `einvoicing_provider`.

---

### Task 2: Update Constants and Mappings
Update frontend constants to whitelist Dominican Republic (`DO`) for e-invoicing and define available providers and labels.

**Files:**
- Modify: `front/src/pages/settings/BillingEntity/const.ts`

- [ ] **Step 1: Add country and provider configuration**
  Replace contents of `front/src/pages/settings/BillingEntity/const.ts` with the whitelisted countries and country-to-provider mappings:
  ```typescript
  import { BasicComboBoxData } from '~/components/form/ComboBox/types'

  export const MANDATORY_EINVOICING_COUNTRIES = ['FR', 'DO']

  export const EINVOICING_PROVIDERS = {
    factur_x: 'factur_x',
    ubl: 'ubl',
    ecf_ssd: 'ecf_ssd',
    dgii_ecf: 'dgii_ecf',
  } as const

  export const EINVOICING_PROVIDERS_BY_COUNTRY: Record<string, BasicComboBoxData[]> = {
    FR: [
      { value: 'factur_x', label: 'Factur-X' },
      { value: 'ubl', label: 'UBL' },
    ],
    DO: [
      { value: 'ecf_ssd', label: 'DGII ECF - SSD' },
      { value: 'dgii_ecf', label: 'DGII ECF' },
    ],
  }
  ```

- [ ] **Step 2: Commit constants**
  Run: `git add front/src/pages/settings/BillingEntity/const.ts` and commit.

---

### Task 3: Update GraphQL Fragment & Regenerate Types
Expose `einvoicingProvider` in frontend GraphQL fragment so it gets requested and submitted.

**Files:**
- Modify: `front/src/hooks/useCreateEditBillingEntity.ts:25-57`

- [ ] **Step 1: Add einvoicingProvider to BillingEntityItem fragment**
  Edit `front/src/hooks/useCreateEditBillingEntity.ts` to include `einvoicingProvider` in the `BillingEntityItem` fragment:
  ```graphql
  fragment BillingEntityItem on BillingEntity {
    ...
    euTaxManagement
    selectedInvoiceCustomSections {
      id
      name
    }
    appliedDunningCampaign {
      id
      name
      code
    }
    einvoicing
    einvoicingProvider
  }
  ```

- [ ] **Step 2: Regenerate GraphQL types**
  Run command in `front` directory: `pnpm codegen`
  Expected: Successful build and update to `front/src/generated/graphql.tsx` containing `einvoicingProvider` in types.

- [ ] **Step 3: Commit GraphQL updates**
  Run: `git commit -am "graphql: add einvoicingProvider field and run codegen"`

---

### Task 4: Integrate Provider Selector in Form
Update billing entity creation/editing form to support selecting a provider.

**Files:**
- Modify: `front/src/pages/settings/BillingEntity/sections/BillingEntityCreateEdit.tsx`

- [ ] **Step 1: Update Formik Initial Values and Effects**
  - Add `einvoicingProvider` to form initial values: `einvoicingProvider: billingEntity?.einvoicingProvider || undefined`.
  - Add `einvoicingProvider` to the GraphQL inputs (`CreateBillingEntityInput` & `UpdateBillingEntityInput`) if they do not automatically absorb it.
  - Update country-change side-effects: If country is not in whitelist, set `einvoicingProvider` to `undefined`. If country changes and is in whitelist, default `einvoicingProvider` to the default one (`ecf_ssd` for `DO`, `factur_x` for `FR`).
  
- [ ] **Step 2: Add ComboBox dropdown for provider**
  Import `EINVOICING_PROVIDERS_BY_COUNTRY` from `../const`.
  In `BillingEntityCreateEdit.tsx`, render `ComboBoxField` when `einvoicing` is `true`:
  ```tsx
  {formikProps.values.einvoicing && (
    <ComboBoxField
      data={EINVOICING_PROVIDERS_BY_COUNTRY[formikProps.values.country || ''] || []}
      name="einvoicingProvider"
      label="E-invoicing Provider"
      formikProps={formikProps}
      disableClearable
    />
  )}
  ```

- [ ] **Step 3: Commit form changes**
  Run: `git commit -am "feat: add conditional einvoicingProvider field to BillingEntity form"`

---

### Task 5: Display Selected Provider in General Settings
Expose the chosen provider name in the general settings view of the Billing Entity.

**Files:**
- Modify: `front/src/pages/settings/BillingEntity/sections/general/InformationBlock.tsx`

- [ ] **Step 1: Update fields array to display einvoicing provider**
  - Extract `einvoicingProvider` from `billingEntity`.
  - Map provider value to its display label (e.g., `'ecf_ssd'` ➔ `'DGII ECF - SSD'`).
  - Add a field inside the `fields` array when `einvoicing` is active:
    ```typescript
    {
      label: 'E-invoicing Provider',
      value: einvoicingProvider ? getProviderLabel(einvoicingProvider) : null,
      fieldKey: 'einvoicingProvider',
    }
    ```

- [ ] **Step 2: Commit display updates**
  Run: `git commit -am "feat: display selected e-invoicing provider in general details view"`
