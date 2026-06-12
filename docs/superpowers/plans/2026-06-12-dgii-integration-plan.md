# DGII Integration using ECF SSD Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integrate Dominican Republic e-invoicing (DGII) into the Lago backend using the `ecf-dgii` Ruby gem, applying asynchronous emission and hybrid frontend/backend polling.

**Architecture:** We use `ecf-dgii` gem in Rails. Multi-tenancy is handled via the Organization's `BillingEntity` tax_id. We'll build an adapter to map Lago Invoices to ECF models. Sidekiq handles the initial send. The frontend SDK requests a read-only token and polls, while a Sidekiq cron job acts as a fallback to poll pending invoices and trigger webhooks.

**Tech Stack:** Ruby on Rails, Sidekiq, ecf-dgii Gem, RSpec.

---

### Task 1: Add Dependency and Initializer

**Files:**
- Modify: `api/Gemfile`
- Create: `api/config/initializers/ecf_dgii.rb`

- [ ] **Step 1: Add gem to Gemfile**
```ruby
# In api/Gemfile, add:
gem 'ecf-dgii', '1.0.3'
```

- [ ] **Step 2: Run bundle install**
Run: `bundle install`
Expected: `ecf-dgii` installed successfully.

- [ ] **Step 3: Create initializer**
Create `api/config/initializers/ecf_dgii.rb`:
```ruby
# api/config/initializers/ecf_dgii.rb
EcfDgii.configure do |config|
  config.api_key = ENV["ECF_API_KEY"]
  config.environment = ENV.fetch("ECF_ENVIRONMENT", "test").to_sym
end
```

- [ ] **Step 4: Commit**
```bash
git add api/Gemfile api/Gemfile.lock api/config/initializers/ecf_dgii.rb
git commit -m "chore: add ecf-dgii gem and initializer"
```

---

### Task 2: Database Schema Additions for Invoices

**Files:**
- Create: `api/db/migrate/YYYYMMDDHHMMSS_add_dgii_fields_to_invoices.rb` (replace timestamp)
- Modify: `api/app/models/invoice.rb`
- Test: `api/spec/models/invoice_spec.rb`

- [ ] **Step 1: Create the migration**
Run: `rails g migration AddDgiiFieldsToInvoices dgii_message_id:string dgii_status:string dgii_impresion_url:string dgii_cod_sec:string dgii_error_message:text`
Wait for the file to be generated. Then modify it to include defaults if needed (but nullable is fine).

- [ ] **Step 2: Run the migration**
Run: `rails db:migrate`
Expected: Migration runs successfully.

- [ ] **Step 3: Update the Invoice Model with Enum**
Modify `api/app/models/invoice.rb` to add the enum and constants.
```ruby
class Invoice < ApplicationRecord
  # ... existing code ...
  
  enum dgii_status: {
    pending: 'pending',
    processing: 'processing',
    approved: 'approved',
    conditionally_approved: 'conditionally_approved',
    rejected: 'rejected',
    error: 'error'
  }, _prefix: true
end
```

- [ ] **Step 4: Write Model Test**
Modify `api/spec/models/invoice_spec.rb`:
```ruby
require 'rails_helper'

RSpec.describe Invoice, type: :model do
  describe 'dgii_status enum' do
    it { should define_enum_for(:dgii_status).with_values(
      pending: 'pending',
      processing: 'processing',
      approved: 'approved',
      conditionally_approved: 'conditionally_approved',
      rejected: 'rejected',
      error: 'error'
    ).backed_by_column_of_type(:string).with_prefix(true) }
  end
end
```

- [ ] **Step 5: Run tests**
Run: `rspec spec/models/invoice_spec.rb`
Expected: PASS

- [ ] **Step 6: Commit**
```bash
git add api/db/migrate/ api/db/schema.rb api/app/models/invoice.rb api/spec/models/invoice_spec.rb
git commit -m "feat: add dgii status and metadata fields to invoices"
```

---

### Task 3: Multi-tenant Onboarding Hooks (Billing Entity)

**Files:**
- Modify: `api/app/models/billing_entity.rb`
- Modify: `api/app/services/billing_entities/update_service.rb` (or equivalent where creation/update happens, we'll hook it on save)
- Test: `api/spec/services/billing_entities/update_service_spec.rb`

We will create a service to register the company in SSD when a DO entity is saved.

- [ ] **Step 1: Create the Register Company Service**
Create `api/app/services/e_invoices/dgii/register_company_service.rb`:
```ruby
module EInvoices
  module Dgii
    class RegisterCompanyService < BaseService
      def initialize(billing_entity:)
        super()
        @billing_entity = billing_entity
      end

      def call
        return result unless @billing_entity.country == 'DO' && @billing_entity.tax_id.present?

        req = EcfDgii::Generated::UpsertCompanyRequest.new(
          rnc: @billing_entity.tax_id,
          legal_name: @billing_entity.legal_name,
          name: @billing_entity.name
        )
        
        begin
          EcfDgii.client.upsert_company(req)
          
          # If a .p12 certificate and password exist in custom_metadata, upload them
          # Assuming custom_metadata['dgii_p12_path'] (or base64) and custom_metadata['dgii_p12_password']
          # This might need adjustment depending on how Lago stores files for BE.
          # We'll stub this out for now and focus on upsert.
        rescue StandardError => e
          result.error!("DGII Registration Failed: #{e.message}", :unprocessable_entity)
        end
        
        result
      end
    end
  end
end
```

- [ ] **Step 2: Write test for Register Company Service**
Create `api/spec/services/e_invoices/dgii/register_company_service_spec.rb`:
```ruby
require 'rails_helper'

RSpec.describe EInvoices::Dgii::RegisterCompanyService do
  let(:organization) { create(:organization) }
  let(:billing_entity) { create(:billing_entity, organization: organization, country: 'DO', tax_id: '131460941', legal_name: 'Test SRL', name: 'Test') }
  subject(:service) { described_class.new(billing_entity: billing_entity) }

  before do
    allow(EcfDgii.client).to receive(:upsert_company).and_return(true)
  end

  it 'registers the company with ECF SSD' do
    expect(EcfDgii.client).to receive(:upsert_company).with(
      have_attributes(rnc: '131460941', legal_name: 'Test SRL', name: 'Test')
    )
    service.call
  end
end
```

- [ ] **Step 3: Run the test**
Run: `rspec spec/services/e_invoices/dgii/register_company_service_spec.rb`
Expected: PASS

- [ ] **Step 4: Commit**
```bash
git add api/app/services/e_invoices/dgii/register_company_service.rb api/spec/services/e_invoices/dgii/register_company_service_spec.rb
git commit -m "feat: add service to register DO billing entities with ECF SSD"
```

---

### Task 4: The DGII Invoice Builder (Adapter)

**Files:**
- Create: `api/app/serializers/e_invoices/invoices/dgii/builder.rb`
- Test: `api/spec/serializers/e_invoices/invoices/dgii/builder_spec.rb`

- [ ] **Step 1: Write Builder test**
Create `api/spec/serializers/e_invoices/invoices/dgii/builder_spec.rb`:
```ruby
require 'rails_helper'

RSpec.describe EInvoices::Invoices::Dgii::Builder do
  let(:organization) { create(:organization) }
  let(:billing_entity) { create(:billing_entity, organization: organization, tax_id: '131460941', legal_name: 'Emisor', address_line1: 'Calle', city: 'SD') }
  let(:customer) { create(:customer, organization: organization, tax_identification_number: '131880681', legal_name: 'Receptor') }
  let(:invoice) { create(:invoice, organization: organization, customer: customer, issue_date: '2026-06-01', total_amount_cents: 11800, taxes_amount_cents: 1800) }
  
  before do
    allow(invoice).to receive(:billing_entity).and_return(billing_entity)
  end

  it 'builds an Ecf31ECF model from an invoice' do
    ecf = described_class.build(invoice: invoice)
    expect(ecf).to be_a(EcfDgii::Generated::Ecf31ECF)
    expect(ecf.encabezado.emisor.rnc_emisor).to eq('131460941')
    expect(ecf.encabezado.comprador.rnc_comprador).to eq('131880681')
    expect(ecf.encabezado.totales.monto_total).to eq(118.00)
  end
end
```

- [ ] **Step 2: Implement Builder**
Create `api/app/serializers/e_invoices/invoices/dgii/builder.rb`:
```ruby
module EInvoices
  module Invoices::Dgii
    class Builder
      def self.build(invoice:)
        be = invoice.billing_entity
        cust = invoice.customer
        
        # Simplification for plan. A real mapping requires parsing line items.
        EcfDgii::Generated::Ecf31ECF.new(
          encabezado: EcfDgii::Generated::Ecf31Encabezado.new(
            version: "Version1_0",
            id_doc: EcfDgii::Generated::Ecf31IdDoc.new(
              tipoe_cf: "FacturaDeCreditoFiscalElectronica", # Defaulting for now
              # encf is usually generated by the backend or ECF SSD. SSD handles it if empty, or we pass a placeholder if required.
              encf: "E310000000001", 
              tipo_pago: "Credito",
              tipo_ingresos: "01",
              fecha_vencimiento_secuencia: "2028-12-31T00:00:00"
            ),
            emisor: EcfDgii::Generated::Ecf31Emisor.new(
              rnc_emisor: be.tax_id,
              razon_social_emisor: be.legal_name,
              direccion_emisor: be.address_line1,
              fecha_emision: invoice.issue_date.to_s
            ),
            comprador: EcfDgii::Generated::Ecf31Comprador.new(
              rnc_comprador: cust.tax_identification_number,
              razon_social_comprador: cust.legal_name
            ),
            totales: EcfDgii::Generated::Ecf31Totales.new(
              monto_total: (invoice.total_amount_cents.to_f / 100.0).round(2),
              # These require proper mapping from invoice.fees and taxes
              monto_gravado_total: ((invoice.total_amount_cents - invoice.taxes_amount_cents).to_f / 100).round(2)
            )
          ),
          detalles_items: [
            # Placeholder item for compilation, requires mapping invoice.fees
            EcfDgii::Generated::Ecf31Item.new(
              numero_linea: 1,
              nombre_item: "Servicios",
              indicador_facturacion: "ITBIS1_18Percent",
              indicador_bieno_servicio: "Servicio",
              cantidad_item: 1,
              precio_unitario_item: ((invoice.total_amount_cents - invoice.taxes_amount_cents).to_f / 100).round(2),
              monto_item: ((invoice.total_amount_cents - invoice.taxes_amount_cents).to_f / 100).round(2)
            )
          ]
        )
      end
    end
  end
end
```

- [ ] **Step 3: Run the test**
Run: `rspec spec/serializers/e_invoices/invoices/dgii/builder_spec.rb`
Expected: PASS

- [ ] **Step 4: Commit**
```bash
git add api/app/serializers/e_invoices/invoices/dgii/builder.rb api/spec/serializers/e_invoices/invoices/dgii/builder_spec.rb
git commit -m "feat: add DGII invoice builder adapter"
```

---

### Task 5: Asynchronous Emission Job (Send)

**Files:**
- Create: `api/app/jobs/invoices/send_to_dgii_job.rb`
- Test: `api/spec/jobs/invoices/send_to_dgii_job_spec.rb`

- [ ] **Step 1: Write Job Test**
Create `api/spec/jobs/invoices/send_to_dgii_job_spec.rb`:
```ruby
require 'rails_helper'

RSpec.describe Invoices::SendToDgiiJob, type: :job do
  let(:invoice) { create(:invoice, dgii_status: 'pending') }
  let(:mock_response) { instance_double("EcfResponse", messageId: 'msg_123', encf: 'E31000') }
  let(:mock_ecf) { instance_double("Ecf31ECF") }

  before do
    allow(EInvoices::Invoices::Dgii::Builder).to receive(:build).and_return(mock_ecf)
    allow(EcfDgii.client).to receive(:send_ecf).and_return(mock_response)
  end

  it 'builds, sends, and transitions status' do
    described_class.new.perform(invoice.id)
    invoice.reload
    expect(invoice.dgii_status_processing?).to be true
    expect(invoice.dgii_message_id).to eq('msg_123')
  end
end
```

- [ ] **Step 2: Implement Job**
Create `api/app/jobs/invoices/send_to_dgii_job.rb`:
```ruby
module Invoices
  class SendToDgiiJob < ApplicationJob
    queue_as :invoices

    def perform(invoice_id)
      invoice = Invoice.find(invoice_id)
      return unless invoice.dgii_status_pending?

      ecf = EInvoices::Invoices::Dgii::Builder.build(invoice: invoice)
      
      begin
        response = EcfDgii.client.send_ecf(ecf)
        invoice.update!(
          dgii_status: :processing,
          dgii_message_id: response.messageId
          # We don't poll here. We let the fallback job or frontend handle it.
        )
      rescue StandardError => e
        invoice.update!(dgii_status: :error, dgii_error_message: e.message)
      end
    end
  end
end
```

- [ ] **Step 3: Run the test**
Run: `rspec spec/jobs/invoices/send_to_dgii_job_spec.rb`
Expected: PASS

- [ ] **Step 4: Commit**
```bash
git add api/app/jobs/invoices/send_to_dgii_job.rb api/spec/jobs/invoices/send_to_dgii_job_spec.rb
git commit -m "feat: add async job to send invoice to DGII"
```

---

### Task 6: Fallback Polling Job

**Files:**
- Create: `api/app/jobs/invoices/poll_dgii_status_job.rb`
- Test: `api/spec/jobs/invoices/poll_dgii_status_job_spec.rb`

- [ ] **Step 1: Write Job Test**
Create `api/spec/jobs/invoices/poll_dgii_status_job_spec.rb`:
```ruby
require 'rails_helper'

RSpec.describe Invoices::PollDgiiStatusJob, type: :job do
  let(:organization) { create(:organization) }
  let(:billing_entity) { create(:billing_entity, organization: organization, tax_id: '131460941') }
  let!(:invoice) { create(:invoice, organization: organization, dgii_status: 'processing', dgii_message_id: 'msg_123') }
  let(:mock_response) { instance_double("EcfResponse", progress: 'Finished', codSec: '123456', impresionUrl: 'http://qr', estatus: 'Aceptado') }

  before do
    allow(invoice).to receive(:billing_entity).and_return(billing_entity)
    allow(EcfDgii.client).to receive(:consulta_resultado).and_return(mock_response)
  end

  it 'polls ECF SSD and updates the invoice to approved' do
    described_class.new.perform
    invoice.reload
    expect(invoice.dgii_status_approved?).to be true
    expect(invoice.dgii_cod_sec).to eq('123456')
    expect(invoice.dgii_impresion_url).to eq('http://qr')
  end
end
```

- [ ] **Step 2: Implement Job**
Create `api/app/jobs/invoices/poll_dgii_status_job.rb`:
```ruby
module Invoices
  class PollDgiiStatusJob < ApplicationJob
    queue_as :invoices

    def perform
      # Fetch all invoices in processing state
      Invoice.where(dgii_status: :processing).where.not(dgii_message_id: nil).find_each do |invoice|
        poll_invoice(invoice)
      end
    end

    private

    def poll_invoice(invoice)
      rnc = invoice.organization.billing_entity&.tax_id
      return unless rnc

      response = EcfDgii.client.consulta_resultado(rnc, invoice.dgii_message_id)

      case response.progress
      when 'Finished'
        if response.estatus == 'AceptadoCondicional'
          invoice.update!(dgii_status: :conditionally_approved, dgii_cod_sec: response.codSec, dgii_impresion_url: response.impresionUrl)
        else
          invoice.update!(dgii_status: :approved, dgii_cod_sec: response.codSec, dgii_impresion_url: response.impresionUrl)
        end
        # Trigger webhook here (e.g., invoice.dgii_approved)
      when 'Rejected'
        invoice.update!(dgii_status: :rejected, dgii_error_message: response.try(:mensaje) || 'Rejected')
      when 'Error'
        invoice.update!(dgii_status: :error, dgii_error_message: response.try(:errors).to_json)
      end
    rescue StandardError => e
      Rails.logger.error("Failed to poll DGII for invoice #{invoice.id}: #{e.message}")
    end
  end
end
```

- [ ] **Step 3: Run the test**
Run: `rspec spec/jobs/invoices/poll_dgii_status_job_spec.rb`
Expected: PASS

- [ ] **Step 4: Commit**
```bash
git add api/app/jobs/invoices/poll_dgii_status_job.rb api/spec/jobs/invoices/poll_dgii_status_job_spec.rb
git commit -m "feat: add cron job to poll DGII processing status"
```

---

### Task 7: Read-only Token Endpoint (for Frontend React SDK)

**Files:**
- Modify: `api/config/routes.rb`
- Create: `api/app/controllers/api/v1/dgii_tokens_controller.rb`
- Test: `api/spec/controllers/api/v1/dgii_tokens_controller_spec.rb`

- [ ] **Step 1: Write Controller Test**
Create `api/spec/controllers/api/v1/dgii_tokens_controller_spec.rb`:
```ruby
require 'rails_helper'

RSpec.describe Api::V1::DgiiTokensController, type: :controller do
  let(:organization) { create(:organization) }
  let(:billing_entity) { create(:billing_entity, organization: organization, tax_id: '131460941') }
  
  before do
    # Assuming standard Lago authentication setup. For tests, we mock current_organization.
    allow(controller).to receive(:current_organization).and_return(organization)
    allow(EcfDgii.client).to receive(:new_company_api_key).with('131460941').and_return('scoped_jwt_token')
  end

  it 'returns a scoped api key' do
    get :show
    expect(response).to have_http_status(:ok)
    expect(JSON.parse(response.body)['token']).to eq('scoped_jwt_token')
  end
end
```

- [ ] **Step 2: Add Route**
Modify `api/config/routes.rb`:
```ruby
# In the api/v1 namespace scope:
get 'dgii/token', to: 'dgii_tokens#show'
```

- [ ] **Step 3: Implement Controller**
Create `api/app/controllers/api/v1/dgii_tokens_controller.rb`:
```ruby
module Api
  module V1
    class DgiiTokensController < BaseController
      def show
        rnc = current_organization.billing_entity&.tax_id
        if rnc.blank?
          render json: { error: 'Organization has no tax_id configured for DGII' }, status: :unprocessable_entity
          return
        end

        begin
          token = EcfDgii.client.new_company_api_key(rnc)
          render json: { token: token }, status: :ok
        rescue StandardError => e
          render json: { error: e.message }, status: :internal_server_error
        end
      end
    end
  end
end
```

- [ ] **Step 4: Run the test**
Run: `rspec spec/controllers/api/v1/dgii_tokens_controller_spec.rb`
Expected: PASS

- [ ] **Step 5: Commit**
```bash
git add api/config/routes.rb api/app/controllers/api/v1/dgii_tokens_controller.rb api/spec/controllers/api/v1/dgii_tokens_controller_spec.rb
git commit -m "feat: add endpoint to provide read-only ECF token for frontend"
```
