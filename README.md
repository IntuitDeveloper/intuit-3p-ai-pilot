# Intuit MCP Server — Developer & Agent Documentation

> **Status: DRAFT.** This document contains placeholder values (sandbox URL, tool names, error codes, rate limits) that have not yet been verified against the production Intuit MCP server. Do not treat as authoritative until reviewed by the MCP pilot team.

This repository hosts dual-audience documentation for the Intuit MCP Server:

- **Developer-friendly docs** — for human developers integrating their AI app with QuickBooks Online via MCP.
- **Agent-friendly docs** — concise, decision-oriented descriptions written for LLMs at inference time.

---

## Table of contents

- [Part 1 — Server-Level Documentation](#part-1--server-level-documentation)
  - [1A. Developer-Friendly Version](#1a-developer-friendly-version)
  - [1B. Agent-Friendly Version](#1b-agent-friendly-version)
- [Part 2 — Tool-Level Documentation (Worked Example: `get_invoice`)](#part-2--tool-level-documentation-worked-example-get_invoice)
  - [2A. Developer-Friendly Version](#2a-developer-friendly-version)
  - [2B. Agent-Friendly Version](#2b-agent-friendly-version)
- [Notes on the two styles](#notes-on-the-two-styles)

---

# Part 1 — Server-Level Documentation

## 1A. Developer-Friendly Version

*(Destined for developer.intuit.com)*

---

# Intuit MCP Server

Connect AI agents and LLM-powered apps to QuickBooks Online through the Model Context Protocol (MCP).

## Overview

The Intuit MCP Server exposes a curated set of QuickBooks Online capabilities as **MCP tools** that any MCP-compatible AI client (Claude Desktop, Cursor, custom agents built on the Anthropic / OpenAI SDKs, etc.) can discover and invoke at runtime.

Use this server when you are building an **agentic experience** — one where the LLM decides which action to take — rather than a traditional integration where your code orchestrates API calls directly. For deterministic, code-driven integrations, continue to use the [QuickBooks Online REST API](https://developer.intuit.com/app/developer/qbo/docs/api/accounting).

## Endpoint

| | |
|---|---|
| **Base URL** | `https://mcp.quickbooks.intuit.com/mcp` |
| **Protocol** | Model Context Protocol (MCP) over HTTPS |
| **Transport** | Streamable HTTP (JSON-RPC 2.0) |
| **Auth** | OAuth 2.0 (Intuit authorization flow) |

## Authentication

The MCP server uses the same OAuth 2.0 authorization flow as the QuickBooks Online REST API. Your client obtains an access token tied to a specific QuickBooks **company (realm)** and presents it on every MCP request.

```
Authorization: Bearer <access_token>
```

- Use your existing Intuit app's `client_id` / `client_secret`
- Required scope: `com.intuit.quickbooks.accounting`
- Token refresh, expiry, and realm selection work identically to the REST API — see [OAuth 2.0 Guide](https://developer.intuit.com/app/developer/qbo/docs/develop/authentication-and-authorization/oauth-2.0)

## Quick start

### 1. Configure your MCP client

**Claude Desktop / Claude Code** — add to your MCP config:

```json
{
  "mcpServers": {
    "quickbooks": {
      "url": "https://mcp.quickbooks.intuit.com/mcp",
      "headers": {
        "Authorization": "Bearer ${QBO_ACCESS_TOKEN}"
      }
    }
  }
}
```

**Python (Anthropic SDK)** — pass the MCP server in the `mcp_servers` parameter on a Messages call. See the [Anthropic MCP connector docs](https://docs.anthropic.com/en/docs/agents-and-tools/mcp-connector) for the exact shape.

### 2. Discover available tools

Once connected, the client calls `tools/list` automatically. You'll see tools like `get_invoice`, `list_customers`, `create_estimate`, etc.

### 3. Invoke a tool

The LLM picks a tool based on the user's natural-language request and calls it through the MCP transport. Your code does not need to map intents to endpoints — the model does that.

## Sandbox vs. Production

| Environment | URL |
|---|---|
| Sandbox | `https://sandbox-mcp.quickbooks.intuit.com/mcp` *(placeholder — confirm with pilot team)* |
| Production | `https://mcp.quickbooks.intuit.com/mcp` |

Use the same sandbox company you use for REST API testing.

## Rate limits

Per-realm limits match the QBO REST API (500 requests/minute, 10 concurrent). MCP tool invocations count against the same bucket.

## What's exposed

The server exposes a **curated** slice of QBO capabilities — not a 1:1 wrapper of every REST endpoint. Tools are designed around partner workflows (e.g. "get unpaid invoices for a customer") rather than raw CRUD.

## When to use MCP vs. REST

| Use the MCP server when... | Use the REST API when... |
|---|---|
| You're building an AI agent or chatbot | You're building a traditional UI / sync engine |
| The user's intent drives the action | Your code drives the action |
| You want the LLM to compose multi-step workflows | You need deterministic, scripted flows |
| You're prototyping fast | You need fine-grained control over every field |

---

## 1B. Agent-Friendly Version

*(System prompt / agent instructions — given to the model, not the developer)*

```markdown
# QuickBooks MCP Server

You have access to QuickBooks Online via MCP tools prefixed with `quickbooks_`.
Use these tools to read and write accounting data (invoices, customers, payments, estimates, items, vendors, bills) for the currently authorized company.

## When to use these tools
- The user asks about their books, finances, customers, invoices, payments, expenses, or any QuickBooks data.
- The user wants to create or update a QuickBooks record.

## When NOT to use these tools
- The user is asking a general accounting question with no reference to their own data → answer directly.
- The user mentions a different accounting product (Xero, NetSuite, etc.) → say you only support QuickBooks.

## Operating rules
1. **Always confirm before any write operation** (create_*, update_*, delete_*, void_*, send_*). Show the user the exact payload you will submit.
2. **Resolve entities by name first.** If the user says "send invoice to Acme", call `quickbooks_list_customers` with a name filter before calling any invoice tool — never guess an ID.
3. **One company at a time.** All tools operate on the authorized realm. If the user references a different company, tell them they must re-authorize.
4. **On error, read the error message.** Tool errors are written for you — they tell you what to fix. Retry once with the correction; if it fails again, surface the error to the user.
5. **Prefer specific tools over generic ones.** If `get_unpaid_invoices` exists, use it instead of `query_invoices` with a filter.

## Currency and dates
- All monetary amounts are in the company's home currency unless a `CurrencyRef` is present.
- Dates are ISO 8601 (`YYYY-MM-DD`). Convert relative dates ("last month") before calling tools.
```

---

# Part 2 — Tool-Level Documentation (Worked Example: `get_invoice`)

## 2A. Developer-Friendly Version

---

### `get_invoice`

Retrieve a single invoice by ID from the authorized QuickBooks company.

**Signature**

```
get_invoice(invoice_id: string) → Invoice
```

**Parameters**

| Name | Type | Required | Description |
|---|---|---|---|
| `invoice_id` | string | yes | The QuickBooks invoice `Id` (not `DocNumber`). Numeric string, e.g. `"130"`. |

**Returns** — `Invoice` object with the same shape as the QBO REST API's [Invoice entity](https://developer.intuit.com/app/developer/qbo/docs/api/accounting/most-commonly-used/invoice), including `Id`, `DocNumber`, `CustomerRef`, `TxnDate`, `DueDate`, `TotalAmt`, `Balance`, `Line[]`, and `EmailStatus`.

**Example request (raw MCP JSON-RPC)**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_invoice",
    "arguments": { "invoice_id": "130" }
  }
}
```

**Example response**

```json
{
  "Id": "130",
  "DocNumber": "1037",
  "CustomerRef": { "value": "24", "name": "Acme Corp" },
  "TxnDate": "2026-05-01",
  "DueDate": "2026-05-31",
  "TotalAmt": 1250.00,
  "Balance": 1250.00,
  "Line": [ /* ... */ ],
  "EmailStatus": "NotSet"
}
```

**Errors**

| Code | Meaning | Resolution |
|---|---|---|
| `INVOICE_NOT_FOUND` | No invoice with that ID in this realm | Verify the ID; consider `list_invoices` to search |
| `UNAUTHORIZED` | Token expired or scope missing | Refresh OAuth token |
| `RATE_LIMITED` | Realm rate limit hit | Back off and retry |

**Equivalent REST call** — `GET /v3/company/{realmId}/invoice/{invoice_id}`

---

## 2B. Agent-Friendly Version

*(The tool's `description` field, sent to the LLM via `tools/list`)*

```json
{
  "name": "get_invoice",
  "description": "Retrieve full details of a single QuickBooks invoice when you already know its ID. Use this after list_invoices or get_unpaid_invoices has returned an invoice the user wants to inspect, or when the user references a specific invoice number you have previously resolved to an ID. Do NOT use this to search — use list_invoices for that. Returns all invoice fields including line items, customer, amounts, due date, and email status.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "invoice_id": {
        "type": "string",
        "description": "QuickBooks internal invoice ID (the 'Id' field, e.g. '130'). This is NOT the invoice number the customer sees (DocNumber). If the user gave you a DocNumber like 'INV-1037', call list_invoices first to resolve it to an Id."
      }
    },
    "required": ["invoice_id"]
  }
}
```

**Example error message (written for the model, not the developer):**

```
Error: No invoice found with Id "130" in this company.
The user may have given you a DocNumber instead of an Id, or the invoice may belong to a different company.
Suggested next step: call list_invoices with a doc_number filter to find the right Id.
```

---

# Notes on the two styles

- **Developer doc** = reference. Exhaustive, has cURL/JSON-RPC, links to REST equivalents, error tables.
- **Agent doc** = prompt. Tight description, tells the model *when* and *when not* to use the tool, error messages are self-correcting hints.

Same tool, two surfaces, different writing.

---

## Open items before publishing

1. Confirm the actual tool inventory exposed by the production MCP server.
2. Confirm the sandbox URL (`sandbox-mcp.quickbooks.intuit.com/mcp` is a placeholder).
3. Confirm OAuth scope name and any MCP-specific scope additions.
4. Confirm rate-limit behavior — does MCP share the REST bucket, or have its own?
5. Confirm partner gating — pilot-only, or open to all IAPP partners?
