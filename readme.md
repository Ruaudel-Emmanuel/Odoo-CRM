# Webhook to Odoo CRM via JSON‑RPC

This workflow turns n8n into a small HTTP API that receives lead data (e.g. from a browser extension), normalises it, and creates a **`crm.lead`** record in **Odoo CRM** using the **JSON‑RPC** API instead of the n8n Odoo node.[^1][^2]

The flow is:

```text
Client (extension / app)
   → Webhook (n8n)
   → Edit Fields
   → IF Valid Lead
      → Odoo Authenticate (JSON‑RPC)
      → Odoo Create Lead (JSON‑RPC)
   → Respond Success / Respond Error
```


## Prerequisites

- An n8n instance (local or remote).
- An Odoo instance with the CRM app enabled and the model **`crm.lead`** available.[^2]
- Network access from n8n to Odoo (e.g. Docker bridge or localhost).[^1]


## Odoo configuration

Make sure you know:

- `ODOO_URL` (example: `http://localhost:8069`)
- `ODOO_DATABASE` (Odoo DB name)
- `ODOO_LOGIN` (Odoo username / email)
- `ODOO_PASSWORD` (password for that user)[^3][^1]

Odoo’s JSON‑RPC endpoint is:

```text
POST {ODOO_URL}/jsonrpc
```

Authentication uses the `common` service and `authenticate` method to return a `uid`, which is then used with the `object` service and `execute_kw` to call methods on models like **`crm.lead`**.[^3][^1]

## Workflow JSON

The workflow JSON you provided defines:

- **Webhook** node: receives a `POST` at `/webhook/odoo-jsonrpc-lead`.
- **Edit Fields (Set)**: maps the incoming body into clean fields compatible with `crm.lead`, and also stores Odoo connection placeholders:
    - `name` (lead/opportunity title)
    - `contact_name` (first + last name)
    - `partner_name` (company name)
    - `email_from`
    - `phone`
    - `type` (set to `lead`)
    - `description` (contains job title and LinkedIn URL)
    - `odoo_url`, `odoo_db`, `odoo_login`, `odoo_password`[^2]
- **IF Valid Lead**: ensures `email_from` and `name` are not empty before calling Odoo.
- **Odoo Authenticate (HTTP Request)**:
    - `POST {{$json.odoo_url + '/jsonrpc'}}`
    - JSON body calls `common.authenticate` with `[db, login, password, {}]`.[^1][^3]
    - Returns `result = uid` on success.[^3][^1]
- **Odoo Create Lead (HTTP Request)**:
    - `POST {{$node['Edit Fields'].json.odoo_url + '/jsonrpc'}}`
    - JSON body calls `object.execute_kw` with:
        - `model`: `"crm.lead"`
        - `method`: `"create"`
        - args: one record dict with `name`, `contact_name`, `partner_name`, `email_from`, `phone`, `type`, `description`.[^2][^3]
- **Respond Success**: returns JSON like:

```json
{
  "success": true,
  "message": "Lead créé dans Odoo CRM via JSON-RPC",
  "lead_id": <Odoo ID>,
  "lead_name": "<name>",
  "email": "<email_from>"
}
```

- **Respond Error**: returns HTTP 400 when required fields are missing:

```json
{
  "success": false,
  "message": "Champs obligatoires manquants",
  "required": ["name", "email_from"]
}
```


## Importing the workflow

1. Open the n8n editor.
2. Click on **Import from file** or **Import from clipboard**.
3. Paste the workflow JSON you provided.
4. Save the workflow and ensure it is **inactive** while you configure it.

## Configuring placeholders

In the **Edit Fields** node, update the following fields:

- `odoo_url`: replace `__ODOO_URL__` with your Odoo URL, e.g. `http://localhost:8069`.[^1]
- `odoo_db`: replace `__ODOO_DATABASE__` with your database name.
- `odoo_login`: replace `__ODOO_LOGIN__` with your Odoo username/email.
- `odoo_password`: replace `__ODOO_PASSWORD__` with your Odoo password.[^1]

These values are injected into the JSON‑RPC calls for `authenticate` and `execute_kw`.[^3][^1]

## Expected request payload

The workflow expects a `POST` JSON body like:

```json
{
  "firstName": "Jean",
  "lastName": "Dupont",
  "email": "jean.dupont@example.com",
  "company": "Acme",
  "jobTitle": "Head of Procurement",
  "linkedinUrl": "https://www.linkedin.com/in/jean-dupont",
  "phone": "+33612345678",
  "leadTitle": "Inbound lead from extension"
}
```

Fields are mapped as follows:


| Incoming field | Odoo `crm.lead` field | Notes |
| :-- | :-- | :-- |
| `leadTitle` | `name` | Lead/opportunity title. [^2] |
| `firstName` + `lastName` | `contact_name` | Full contact name. [^2] |
| `company` | `partner_name` | Company name. [^2] |
| `email` | `email_from` | Email used for follow‑up. [^2] |
| `phone` | `phone` | Optional. [^2] |
| `jobTitle`, `linkedinUrl` | `description` | Stored as text notes. [^2] |
| constant | `type` | Set to `"lead"`. [^4] |

## Testing

1. In the Webhook node, click **Test → Listen for Test Event** to expose the test URL.
2. Send a `POST` request to the **test URL** (from curl, Postman, or your extension) with the payload above.
3. Check the workflow execution:
    - `IF Valid Lead` should follow the *true* branch for a valid payload.
    - `Odoo Authenticate` should return a `result` (user id).[^1][^3]
    - `Odoo Create Lead` should return `result = <new lead id>`.[^3]
4. The HTTP response should be a JSON success message with the new `lead_id`.

Example curl (replace `<TEST_URL>`):

```bash
curl -X POST "<TEST_URL>" \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Jean",
    "lastName": "Dupont",
    "email": "jean.dupont@example.com",
    "company": "Acme",
    "jobTitle": "Head of Procurement",
    "linkedinUrl": "https://www.linkedin.com/in/jean-dupont",
    "phone": "+33612345678",
    "leadTitle": "Inbound lead from extension"
  }'
```



