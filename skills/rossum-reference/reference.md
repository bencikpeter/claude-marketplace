# Rossum.ai Complete Reference

## Platform Overview

Rossum.ai is a cloud-based document automation platform specializing in AI-powered data extraction from business documents (invoices, purchase orders, receipts, etc.). The platform provides:

- AI-powered data extraction from documents
- Cloud-based user interface for verification and correction
- Extension environment for custom logic (webhooks, serverless functions)
- Reporting database
- API for programmatic access

### Architecture Hierarchy

```
Organization
└── Workspace
    └── Queue (linked to a Schema)
        ├── Inbox (email import)
        ├── Hooks (extensions: webhooks, serverless functions, connectors)
        └── Documents
            └── Annotations (extracted data + lifecycle)
                └── Pages
```

### Key Concepts

- **Organization**: Top-level account containing users, workspaces, and billing
- **Workspace**: Groups queues for logical project separation
- **Queue**: Document processing pipeline with a linked schema; each queue processes documents according to its configured schema
- **Schema**: Defines the structure and fields to extract from documents (sections, datapoints, multivalues/tables)
- **Document**: An uploaded file (PDF, PNG, JPEG, TIFF, XLSX, XLS, DOCX, DOC, HTML)
- **Annotation**: Extracted data from a document, tracking the full processing lifecycle
- **Page**: Individual page within a document
- **Hook/Extension**: Webhook, serverless function, or connector that extends platform behavior
- **Inbox**: Email endpoint that auto-imports documents into a queue
- **Dedicated Engine**: Custom AI model trained for specific document types or use cases
- **Label**: Tags for organizing and filtering annotations

---

## Authentication

### Token-Based Auth

**Login**: `POST /v1/auth/login`
- Parameters: `username` (string, required), `password` (string, required), `max_token_lifetime_s` (integer, optional, default: 162 hours)
- Response: `{"key": "token_string", "domain": "domain_name"}`
- Usage: `Authorization: Bearer {token}` or `Authorization: Token {token}`

**Logout**: `POST /v1/auth/logout`

**Token Exchange**: `POST /v1/auth/token`
- Parameters: `scope` ("default" or "approval"), `max_token_lifetime_s` (max 583200s)
- Response: `{"key": "token", "domain": "domain", "scope": "default"}`

### JWT Authentication

Short-lived JWT tokens can be exchanged for access tokens. Supports EdDSA (Ed25519, Ed448) and RS512 signatures only, max token validity 60 seconds.

**JWT Header**: `alg` (required: "EdDSA" or "RS512"), `kid` (required, ends with `:{Rossum org ID}`), `typ` (optional)

**JWT Payload**: `ver` ("1.0"), `iss` (issuer name), `aud` (target domain URL), `sub` (user email), `exp` (UNIX timestamp, max 60s from now), `email`, `name`, `rossum_org` (org ID), `roles` (optional, for auto-provisioning)

### Single Sign-On (SSO)

OAuth2 OpenID Connect protocol. Redirect URI: `https://<domain>.rossum.app/api/v1/oauth/code`. Email claims use case-insensitive matching.

### Basic Auth

Supported for upload/export endpoints: `Authorization: Basic {base64(username:password)}`

---

## API Conventions

**Base URL**: `https://<domain>.rossum.app/api/v1`

**Pagination**: All list endpoints use `page_size` (default: 20, max: 100) and `page` (default: 1)

**Ordering**: `ordering` parameter, prefix with `-` for descending

**Date Format**: ISO 8601 in UTC (e.g., `2018-06-01T21:36:42.223415Z`)

**Rate Limits**: 10 requests/second (general), 10 requests/minute (translate endpoint)

**Metadata**: Most objects support custom `metadata` JSON (up to 4 KB per object)

**File Size Limit**: 40 MB per document, 50 MB for email imports

**Supported Import Formats**: PDF, PNG, JPEG, TIFF, XLSX, XLS, DOCX, DOC, HTML

**Export Formats**: CSV, XML, JSON, XLSX

### Common Response Codes

| Code | Meaning |
|------|---------|
| 200 | Success |
| 201 | Created |
| 400 | Bad request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not found |
| 409 | Conflict |
| 429 | Too many requests (check `Retry-After` header) |
| 500 | Server error |

---

## Organizations

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/organizations` | List organizations |
| POST | `/v1/organizations` | Create organization |
| GET | `/v1/organizations/{id}` | Retrieve organization |
| POST | `/v1/organizations/{id}/token` | Generate access token |
| GET | `/v1/organizations/{id}/limits` | Get usage limits |
| GET | `/v1/organizations/{id}/billing` | Get billing info |

---

## Workspaces

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/workspaces` | List workspaces |
| POST | `/v1/workspaces` | Create workspace |
| GET | `/v1/workspaces/{id}` | Retrieve workspace |
| PUT | `/v1/workspaces/{id}` | Update workspace |
| PATCH | `/v1/workspaces/{id}` | Partial update |
| DELETE | `/v1/workspaces/{id}` | Delete workspace |

**Create/Update fields**: `name` (required), `organization` (URL, required), `metadata` (optional, up to 4 KB)

**Filtering**: `organization` (integer)

---

## Queues

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/queues` | List queues |
| POST | `/v1/queues` | Create queue |
| GET | `/v1/queues/{id}` | Retrieve queue |
| PUT | `/v1/queues/{id}` | Update queue |
| PATCH | `/v1/queues/{id}` | Partial update |
| DELETE | `/v1/queues/{id}` | Delete queue |
| POST | `/v1/queues/{id}/duplicate` | Duplicate queue |
| POST | `/v1/queues/{id}/import` | Import document |
| GET | `/v1/queues/{id}/export` | Export annotations |
| GET | `/v1/queues/{id}/counts` | Get counts |

**Create/Update fields**: `name` (required), `workspace` (URL, required), `schema` (URL, required), `metadata` (optional)

**Export parameters**: `status` (filter), `format` (csv/xml/json/xlsx), `id` (specific annotation)

**Filtering**: `workspace` (integer)

---

## Schemas

Schemas define what data gets extracted from documents.

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/schemas` | List schemas |
| POST | `/v1/schemas` | Create schema |
| GET | `/v1/schemas/{id}` | Retrieve schema |
| PUT | `/v1/schemas/{id}` | Update schema |
| PATCH | `/v1/schemas/{id}` | Partial update |
| DELETE | `/v1/schemas/{id}` | Delete schema |
| POST | `/v1/schemas/validate` | Validate schema |

### Schema Content Structure

Schemas consist of **sections** containing **datapoints** (header fields) and **multivalues** (tables/line items).

**Common attributes** (all schema objects):
- `category`: "section", "datapoint", "multivalue", or "tuple"
- `id`: Unique identifier (max 50 chars)
- `label`: Display name
- `hidden`: Hide from UI (default: false)
- `disable_prediction`: Disable AI extraction (default: false)

### Datapoint (Field) Types

| Type | Description |
|------|-------------|
| `string` | Text. Supports length constraints and regex patterns |
| `number` | Numeric. `format` for display (e.g., `#,##0.#`), `aggregations` for column sums |
| `date` | Date. `format` for layout (e.g., `MM/DD/YYYY`) |
| `enum` | Dropdown. `options` array of `{value, label}`, `enum_value_type` |
| `button` | Interactive UI element. `popup_url`, `can_obtain_token` |
| `formula` | Calculated from other fields |
| `reasoning` | AI-generated from prompt and context |

### Datapoint Configuration

- `rir_field_names`: Sources for AI extraction (e.g., `document_id`, `date_issue`, `amount_total`)
- `default_value`: Fallback if extraction unavailable
- `constraints`: `length` (min/max), `regexp`, `required`
- `score_threshold`: AI confidence threshold for auto-validation
- `ui_configuration.type`: `captured`, `data`, `manual`, `formula`, `reasoning`
- `ui_configuration.edit`: `enabled`, `enabled_without_warning`, `disabled`

### Common `rir_field_names` (AI Extraction Sources)

**Identifiers**: `document_id`, `customer_id`, `order_id`, `account_num`, `iban`, `bic`, `bank_num`

**Dates**: `date_issue`, `date_due`, `date_delivery`, `date_performance`

**Parties**: `sender_name`, `sender_address`, `sender_ic`, `sender_dic`, `recipient_name`, `recipient_address`, `recipient_ic`, `recipient_dic`

**Amounts**: `amount_total`, `amount_due`, `amount_paid`, `amount_total_tax`, `amount_total_base`, `amount_rounding`

**Document attributes**: `currency`, `document_type`, `language`, `payment_method_type`

**Line item columns**: `item_description`, `item_quantity`, `item_amount_total`, `item_amount_base`, `item_amount_tax`, `item_tax_rate`, `item_uom`, `item_code`, `item_other`

**Tax details**: `tax_detail_rate`, `tax_detail_base`, `tax_detail_tax`, `tax_detail_total`, `tax_detail_code`

### Multivalue (Table Container)

- `children`: Nested datapoint or tuple
- `min_occurrences` / `max_occurrences`: Row count limits
- `grid.row_types`: Classify rows (header, data, footer)
- `grid.default_row_type`: Default classification
- `grid.row_types_to_extract`: Which rows to include in export

### Tuple (Table Row)

- `children`: Array of datapoints in the row
- `rir_field_names`: AI field sources for the row

### Schema Update Behavior

Data values are preserved when: adding/removing fields, reordering fields, moving fields between sections, converting single fields to multivalues, changing tuple membership, updating labels/formats/constraints/enum options.

---

## Documents

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/documents` | List documents |
| POST | `/v1/documents` | Create document |
| GET | `/v1/documents/{id}` | Retrieve document |
| PATCH | `/v1/documents/{id}` | Partial update |
| GET | `/v1/documents/{id}/content` | Get file content |
| DELETE | `/v1/documents/{id}` | Delete document |

**Attributes**: `id`, `url`, `s3_name`, `mime_type`, `arrived_at`, `original_file_name`, `content` (file URL), `metadata`, `annotations` (array of URLs)

**Supported formats**: PDF, PNG, JPEG, TIFF, XLSX, XLS, DOCX, DOC, HTML (max 40 MB)

---

## Annotations

Annotations represent extracted data from documents and track the full processing lifecycle.

### Annotation Lifecycle

```
                                ┌──────────┐
                         ┌─────│ importing │
                         │     └──────────┘
                         │           │
                         │     ┌─────▼──────┐
              ┌──────────┤     │ to_review   │◄─────────────────────┐
              │          │     └─────┬───────┘                      │
              │          │           │                               │
         ┌────▼─────┐   │     ┌─────▼──────┐    ┌────────────┐     │
         │ failed_   │   │     │ reviewing  │───►│ confirmed  │     │
         │ import    │   │     └────────────┘    └─────┬──────┘     │
         └──────────┘   │                              │            │
                         │     ┌────────────┐    ┌─────▼──────┐     │
                         │     │ rejected   │    │in_workflow  │     │
                         │     └────────────┘    └─────┬──────┘     │
                         │                              │            │
                         │                        ┌─────▼──────┐     │
                         │                        │ exporting  │─────┘
                         │                        └─────┬──────┘  (on failure)
                         │                              │
                         │                        ┌─────▼──────┐
                         │                        │ exported   │
                         │                        └────────────┘
                         │
                    ┌────▼─────┐    ┌──────────┐
                    │postponed │    │ deleted   │──► purged
                    └──────────┘    └──────────┘
```

**Status descriptions**:

| Status | Description |
|--------|-------------|
| `created` | Manually created, awaiting import |
| `importing` | AI engine actively extracting data |
| `failed_import` | Processing error (malformed file, etc.) |
| `split` | Divided into multiple documents |
| `to_review` | Extraction complete, awaiting validation |
| `reviewing` | User actively validating |
| `confirmed` | User validated and confirmed |
| `rejected` | User declined annotation |
| `in_workflow` | Processing through automated workflows (content locked) |
| `exporting` | Awaiting connector completion |
| `exported` | Successfully exported (terminal state) |
| `failed_export` | Connector returned error |
| `postponed` | User deferred processing |
| `deleted` | Marked for deletion |
| `purged` | Metadata-only retention (irreversible) |

### Annotation Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/annotations` | List annotations |
| POST | `/v1/annotations` | Create annotation |
| GET | `/v1/annotations/{id}` | Retrieve annotation |
| PUT | `/v1/annotations/{id}` | Update annotation |
| PATCH | `/v1/annotations/{id}` | Partial update |
| DELETE | `/v1/annotations/{id}` | Delete annotation |
| POST | `/v1/annotations/{id}/copy` | Copy annotation |
| POST | `/v1/annotations/{id}/start` | Start annotation |
| POST | `/v1/annotations/{id}/confirm` | Confirm annotation |
| POST | `/v1/annotations/{id}/cancel` | Cancel annotation |
| POST | `/v1/annotations/{id}/approve` | Approve annotation |
| POST | `/v1/annotations/{id}/reject` | Reject annotation |
| POST | `/v1/annotations/{id}/assign` | Assign to user |
| POST | `/v1/annotations/{id}/postpone` | Switch to postponed |
| POST | `/v1/annotations/{id}/switch_to_deleted` | Switch to deleted |
| POST | `/v1/annotations/{id}/rotate` | Rotate pages |
| POST | `/v1/annotations/{id}/edit` | Edit annotation |
| POST | `/v1/annotations/{id}/split` | Split annotation |
| POST | `/v1/annotations/{id}/validate` | Validate content |
| POST | `/v1/annotations/{id}/purge` | Purge deleted |
| GET | `/v1/annotations/{id}/time_spent` | Get time spent |
| GET | `/v1/annotations/{id}/page_data` | Get spatial data |
| POST | `/v1/annotations/{id}/page_data/translate` | Translate spatial data |
| POST | `/v1/annotations/search` | Search annotations |

### Annotation Content (Extracted Data)

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/annotations/{id}/content` | Get extracted data |
| PATCH | `/v1/annotations/{id}/content` | Update data |
| POST | `/v1/annotations/{id}/content/bulk_update` | Bulk update |
| POST | `/v1/annotations/{id}/content/replace_by_ocr` | Re-OCR |
| POST | `/v1/annotations/{id}/content/validate` | Validate against schema |

**Filtering annotations**: `status`, `queue`, `workspace`, date ranges

---

## Pages

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/pages` | List pages |
| GET | `/v1/pages/{id}` | Retrieve page |

**Attributes**: `id`, `url`, `annotation`, `page_number` (1-indexed), `image` (URL), `width`, `height`

---

## Uploads

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/v1/uploads` | Upload document |
| GET | `/v1/uploads/{id}` | Check upload status |

**Upload states**: `created` → `processing` → `succeeded` / `failed`

**Parameters**: `queue` (required), `content` (multipart file, required)

**Recommended**: A4 format, minimum 150 DPI for scans/photos

---

## Hooks (Extensions)

Hooks extend Rossum with custom logic. Three types: **webhooks**, **serverless functions**, and **connectors**.

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/hooks` | List hooks |
| POST | `/v1/hooks` | Create hook |
| GET | `/v1/hooks/{id}` | Retrieve hook |
| PUT | `/v1/hooks/{id}` | Update hook |
| PATCH | `/v1/hooks/{id}` | Partial update |
| DELETE | `/v1/hooks/{id}` | Delete hook |
| POST | `/v1/hooks/{id}/test` | Test hook |
| POST | `/v1/hooks/{id}/manual_trigger` | Manual trigger |
| GET | `/v1/hooks/{id}/logs` | List call logs |

### Webhook Extension

Webhooks send HTTP POST payloads to a configured URL when events occur.

**Payload validation**: HMAC-SHA256 signature in request headers.

### Serverless Function Extension

Custom code executed in response to events without maintaining infrastructure. Functions receive event payloads and can modify annotation data or trigger downstream processes.

### Connector Extension

Connectors push validated data to external systems via two endpoints:
- **Validate endpoint**: Called before export; can reject invalid data
- **Save endpoint**: Called after validation; HTTP 200 marks annotation as exported

### Webhook Events

| Event | Trigger |
|-------|---------|
| `upload.created` | Document uploaded |
| `annotation.started` | Annotation begins |
| `annotation.confirmed` | User confirms data |
| `annotation.exported` | Export succeeds |
| `annotation.rejected` | Annotation rejected |
| `annotation.failed_export` | Export failed |
| `email.received` | Email arrives at inbox |

---

## Connectors

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/connectors` | List connectors |
| POST | `/v1/connectors` | Create connector |
| GET | `/v1/connectors/{id}` | Retrieve connector |
| PUT | `/v1/connectors/{id}` | Update connector |
| PATCH | `/v1/connectors/{id}` | Partial update |
| DELETE | `/v1/connectors/{id}` | Delete connector |

---

## Inboxes

Email endpoints that auto-import documents into queues.

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/inboxes` | List inboxes |
| POST | `/v1/inboxes` | Create inbox |
| GET | `/v1/inboxes/{id}` | Retrieve inbox |
| PUT | `/v1/inboxes/{id}` | Update inbox |
| PATCH | `/v1/inboxes/{id}` | Partial update |
| DELETE | `/v1/inboxes/{id}` | Delete inbox |

**Email limits**: 50 MB (raw message). ZIP archives: 40 MB uncompressed, max 1000 files. Only root-level or first-level directory contents extracted.

---

## Emails

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/emails` | List emails |
| GET | `/v1/emails/{id}` | Retrieve email |
| PUT | `/v1/emails/{id}` | Update email |
| PATCH | `/v1/emails/{id}` | Partial update |
| POST | `/v1/emails/{id}/import` | Import email |
| POST | `/v1/emails/{id}/send` | Send email |
| GET | `/v1/emails/counts` | Get counts |

---

## Email Templates

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/email_templates` | List templates |
| POST | `/v1/email_templates` | Create template |
| GET | `/v1/email_templates/{id}` | Retrieve template |
| PUT | `/v1/email_templates/{id}` | Update template |
| PATCH | `/v1/email_templates/{id}` | Partial update |
| DELETE | `/v1/email_templates/{id}` | Delete template |
| POST | `/v1/email_templates/{id}/render` | Render with annotation data |

---

## Users

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/users` | List users |
| POST | `/v1/users` | Create user |
| GET | `/v1/users/{id}` | Retrieve user |
| GET | `/v1/users/me` | Current user |
| PUT | `/v1/users/{id}` | Update user |
| PATCH | `/v1/users/{id}` | Partial update |
| DELETE | `/v1/users/{id}` | Delete user |
| POST | `/v1/users/{id}/set_password` | Set password |

Users can be auto-provisioned through SSO. Roles determine permissions (annotation review, approval, admin functions).

---

## Rules and Triggers

### Rules

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/rules` | List rules |
| POST | `/v1/rules` | Create rule |
| GET | `/v1/rules/{id}` | Retrieve rule |
| PUT | `/v1/rules/{id}` | Update rule |
| PATCH | `/v1/rules/{id}` | Partial update |
| DELETE | `/v1/rules/{id}` | Delete rule |

**Rule actions**: Send email, update fields, change status, assign to user, add labels, trigger webhooks.

**Rule conditions**: Field value matches/contains, numerical comparisons, date ranges, AND/OR logic.

### Triggers

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/triggers` | List triggers |
| POST | `/v1/triggers` | Create trigger |
| GET | `/v1/triggers/{id}` | Retrieve trigger |
| PUT | `/v1/triggers/{id}` | Update trigger |
| PATCH | `/v1/triggers/{id}` | Partial update |
| DELETE | `/v1/triggers/{id}` | Delete trigger |

**Trigger events**: `annotation.started`, `annotation.confirmed`, `annotation.rejected`, `annotation.exported`, `field.changed`, `status.changed`

---

## Dedicated Engines

Custom AI models trained for specific document types or use cases.

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/v1/dedicated_engines` | Create engine |
| GET | `/v1/dedicated_engines` | List engines |
| GET | `/v1/dedicated_engines/{id}` | Retrieve engine |
| PUT | `/v1/dedicated_engines/{id}` | Update engine |
| PATCH | `/v1/dedicated_engines/{id}` | Partial update |
| DELETE | `/v1/dedicated_engines/{id}` | Delete engine |

### Dedicated Engine Schemas

| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/v1/dedicated_engine_schemas/validate` | Validate schema |
| POST | `/v1/dedicated_engine_schemas/predict` | Test extraction |
| GET | `/v1/dedicated_engine_schemas` | List schemas |
| POST | `/v1/dedicated_engine_schemas` | Create schema |
| GET | `/v1/dedicated_engine_schemas/{id}` | Retrieve schema |
| PUT | `/v1/dedicated_engine_schemas/{id}` | Update schema |
| DELETE | `/v1/dedicated_engine_schemas/{id}` | Delete schema |

### Generic Engines

Pre-built extraction engines for common document types.

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/generic_engines` | List engines |
| GET | `/v1/generic_engines/{id}` | Retrieve engine |
| GET | `/v1/generic_engine_schemas` | List schemas |
| GET | `/v1/generic_engine_schemas/{id}` | Retrieve schema |

---

## Labels

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/labels` | List labels |
| POST | `/v1/labels` | Create label |
| GET | `/v1/labels/{id}` | Retrieve label |
| PUT | `/v1/labels/{id}` | Update label |
| PATCH | `/v1/labels/{id}` | Partial update |
| DELETE | `/v1/labels/{id}` | Delete label |

Labels can be added/removed on annotations for tagging and filtering.

---

## Automation

### AI Confidence & Auto-validation

`score_threshold` on datapoints controls automatic validation. If AI confidence exceeds the threshold, the field is auto-validated. Falls back to queue's `default_score_threshold` if not set on the datapoint.

### Automation Blockers

Track reasons preventing full automation:

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/automation_blockers` | List blockers |
| GET | `/v1/automation_blockers/{id}` | Retrieve blocker |

---

## Audit Logs

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/audit_logs` | List audit logs |

Records include: user, action type (create/update/delete/export), timestamp, affected object, previous/updated values, IP address, session info.

**Filtering**: date range, user, action type, object type, queue, workspace.

---

## Hook Logs

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/v1/hook_logs` | List hook execution logs |

Records include: request sent, response received, timestamp, duration, success/failure, error messages.
