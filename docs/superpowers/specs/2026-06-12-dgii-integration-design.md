# Integration Design: ECF SSD (DGII) in Lago

## Overview
This document outlines the architecture for integrating Dominican Republic e-invoicing (DGII) into the Lago billing platform using the `ecf-dgii` Ruby SDK and `@ssddo/ecf-react` frontend SDK. The integration leverages ECF SSD to offload XML signing and DGII communication, implementing a robust hybrid polling architecture to handle asynchronous invoice processing while maintaining strict multi-tenancy.

## Architecture & Data Flow

### 1. Multi-Tenant Configuration
- **Global Auth:** The Lago instance authenticates with ECF SSD using a single master `ECF_API_KEY` defined in environment variables.
- **Tenant Isolation:** ECF SSD differentiates tenants via the `tax_id` (RNC) configured in the Lago `BillingEntity`.
- **Onboarding:** When an organization configures its `BillingEntity` with country `DO`, a Lago Service or `after_commit` hook will:
  1. Call `EcfDgii.client.upsert_company(rnc: ...)` to register the tenant in ECF SSD.
  2. Upload the organization's `.p12` digital certificate via `EcfDgii.client.update_certificate_company`.

### 2. Invoice Generation & Emission (Backend)
- **Adapter/Builder:** A new class `EInvoices::Invoices::Dgii::Builder` will map Lago's `Invoice` model (fees, taxes, coupons) to the strictly typed `EcfDgii::Generated::Ecf31ECF` models.
- **Asynchronous Emission:** 
  - The `SendToDgiiJob` will construct the eCF and call `client.send_ecf(ecf)` (non-polling).
  - The resulting `messageId` and `encf` will be stored in the invoice's metadata or dedicated columns.
  - The invoice state will transition to an intermediate state (e.g., `dgii_processing`).

### 3. Hybrid Polling Architecture (The "Option C" Approach)

#### Frontend Real-time Polling
- The React frontend requests a scoped, read-only token via a new Lago endpoint (e.g., `GET /api/v1/dgii/token`) which internally calls `client.new_company_api_key(tenant_rnc)`.
- The frontend uses `@ssddo/ecf-react` to poll ECF SSD directly, providing real-time UX feedback.
- Upon detecting a terminal state (`Finished`, `Error`, `Rejected`), the frontend displays the result (including the QR code via `impresionUrl`) and optionally notifies the Lago backend to sync state immediately.

#### Backend Fallback Polling (Resilience)
- A scheduled Sidekiq Job (`PollDgiiStatusJob`) runs periodically (e.g., every 5 minutes).
- It fetches all invoices currently in `dgii_processing` state.
- It queries ECF SSD for their status.
- It updates the database with the terminal state and triggers appropriate webhooks (`invoice.dgii_approved`, `invoice.dgii_rejected`, etc.).

### 4. Comprehensive State Handling
The system must gracefully handle all possible ECF response states:
- **`Queued` / `Sending` / `Polling`:** Invoice remains in `dgii_processing` state. UI shows a spinner/progress indicator.
- **`Finished` (Aceptado):** Invoice is marked as `approved`. The `impresionUrl`, `codSec`, and `fechaFirma` are saved. A webhook is fired. The UI renders the final receipt with the QR code.
- **`Finished` (Aceptado Condicional):** Treated similarly to accepted, but with a warning flag stored in metadata.
- **`Rejected` / `Error`:** Invoice is marked as `failed` or `rejected`. The specific `Mensaje` or `Errors` from the `EcfResponse` are logged and displayed in the UI for user correction. Webhook is fired to notify the failure.

## Database Schema Additions
- `billing_entities`: Ensure `.p12` certificate storage (encrypted) and password storage (already conceptually present or needs addition).
- `invoices`: 
  - `dgii_message_id` (string)
  - `dgii_status` (enum/string: pending, processing, approved, conditionally_approved, rejected, error)
  - `dgii_impresion_url` (string)
  - `dgii_cod_sec` (string)
  - `dgii_error_message` (text)