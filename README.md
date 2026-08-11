# Intuit AI Pilot — Hosted MCP Server

Connect AI agents and LLM-powered apps to QuickBooks Online through the Model Context Protocol (MCP).

> **Status: PILOT - Closed to Applications **
>
> **Audience:** AI pilot partners under the [[Intuit App Partner Program](https://static.developer.intuit.com/resources/Intuit_App_Partner_Program_Guide.pdf)].
>

 **NOTE: This is a pilot program and is provided "as is" for testing purposes. Intuit reserves the right to change functionality or charge for any MCP server functionality in the future, although no charge is associated with participation in the pilot program.**

---

## Overview

The Intuit MCP Server exposes a curated set of QuickBooks Online capabilities as MCP tools that any MCP-compatible AI client (Claude Desktop, Cursor, custom agents built on the Anthropic / OpenAI SDKs, etc.) can discover and invoke at runtime.

Use this server when you are building an agentic experience — one where the LLM decides which action to take — rather than a traditional integration where your code orchestrates API calls directly. For deterministic, code-driven integrations, continue to use the QuickBooks Online REST API.

---

## Table of contents

- [MCP server onboarding](#mcp-server-onboarding)
- [MCP endpoint](#mcp-endpoint)
- [Protocol details](#protocol-details)
- [Authentication](#authentication)
- [MCP tools list](#mcp-tools-list)
- [Sample prompts](#sample-prompts)
- [Appendix](#appendix)

---

## MCP server onboarding

For the pilot program, pilot partners need to share the following details with your assigned Intuit Solution Engineer:

- Your AI app/agent's IP range to whitelist for access to the MCP server.
- Your primary agentic use case (e.g., Quote-to-Cash).
- Your app/agent's LLM model (e.g., ChatGPT).
- Your specific `AppID` from the Intuit Developer Portal workspace which needs to be onboarded with MCP scopes.

Once your app is whitelisted, you can connect to the hosted MCP server endpoint.

---

## MCP endpoint

| Field | Value |
|---|---|
| Production Base URL | `https://mcp.quickbooks.intuit.com/mcp` |
| User-Agent | Set `User-Agent` in request headers to `partner_app_[AppName]` |

---

## Protocol details

The integration uses the following protocol:

| Field | Value |
|---|---|
| Transport | MCP Streamable HTTP (JSON-RPC 2.0 over POST) |
| Protocol Version | `2025-11-25` |
| Stateless Mode | Enabled (no `mcp-session-id` requirement) |
| Auth | OAuth 2.0 (per MCP Authorization Spec) |

---

## Authentication

The MCP server uses the same OAuth 2.0 authorization flow as the QuickBooks Online REST API. Your client obtains an access token tied to a specific QuickBooks company (realm) and presents it on every MCP request. Use your existing Intuit app's `client_id` / `client_secret`.

| Field | Value |
|---|---|
| Flow | OAuth 2.0 Authorization Code |
| Scopes | `com.intuit.quickbooks.accounting` (required) **and** restricted MCP scopes — reach out to your assigned Intuit Solution Engineer for scope onboarding |
| Auth Model | OAuth 2.0 |
| Token refresh, expiry, and company/realm selection | Work identically to the REST API — see the [OAuth 2.0 Guide](https://developer.intuit.com/app/developer/qbo/docs/develop/authentication-and-authorization/oauth-2.0) |

As next steps, configure your MCP client. Once connected, the client calls `tools/list` automatically. You'll see tools like `qbo_sales_create_invoice` etc. See the list of tools and scopes available below. The LLM picks a tool based on the user's natural-language request and calls it through the MCP transport.

---

## MCP tools list

Available MCP tools for partners:

| MCP Tool Name | Scope |
|---|---|
| `qbo_contact_search_customer` | `contact.tools.customer.read` |
| `qbo_contact_create_customer` | `contact.tools.customer` |
| `qbo_catalog_search_products` | `catalog.tools.item.read` |
| `qbo_catalog_create_product` | `catalog.tools.item` |
| `qbo_sales_get_sales_orders` | `sales.tools.salesorder.read` |
| `qbo_sales_create_sales_order_workflow`, `qbo_sales_close_sales_order` | `sales.tools.salesorder` |
| `qbo_sales_create_invoice`, `qbo_sales_delete_invoice`, `qbo_sales_duplicate_invoice`, `qbo_sales_send_invoice`, `qbo_sales_update_invoice` | `sales.tools.invoice` |
| `qbo_sales_get_invoices` | `sales.tools.invoice.read` |
| `qbo_sales_get_settings` | `sales.tools.settings.read` |
| `qbo_sales_update_settings` | `sales.tools.settings` |
| `qbo_sales_get_estimates` | `sales.tools.estimate.read` |
| `qbo_sales_create_estimate`, `qbo_sales_delete_estimate`, `qbo_sales_duplicate_estimate`, `qbo_sales_send_estimate`, `qbo_sales_update_estimate` | `sales.tools.estimate` |
| `qbo_sales_get_payment_links` | `sales.tools.payment-link.read` |
| `qbo_sales_create_payment_link`, `qbo_sales_send_payment_link` | `sales.tools.payment-link` |

---

## Sample prompts

Here are samples you can use for testing.

**LLM model used for testing:** ChatGPT 5.5 Extended Thinking

| Use Case | User Prompt | Expected LLM Output | Expected QBO Changes |
|---|---|---|---|
| **Create Sales Order** (Known Customer) | The Smith Headquarters quote was just signed in Knowify for $50,000. Create the Sales Order in QuickBooks using their PO number 'PO-987'. Map revenue to 'Construction Services' account. | Confirms: Sales Order ORD-XXXX created for Smith Headquarters, $50,000, PO-987, mapped to Construction Services. Ask the user to confirm before saving. | New Sales Order created in QBO:<br>• Customer: Smith Headquarters<br>• Amount: $50,000<br>• PO#: PO-987<br>• Account: Construction Services<br>• Status: Open |
| **Create Sales Order** (New Customer) | I have a new customer, Acme Corp. Create a Sales Order for $5,000 for landscaping services. | Confirms new customer Acme Corp will be created, then Sales Order for $5,000. Requests confirmation before proceeding. | New Customer record created in QBO.<br>New Sales Order linked to Acme Corp:<br>• Amount: $5,000<br>• Service: Landscaping<br>• Status: Open |
| **Convert Sales Order → Invoice** (Full) | The Demolition phase is 100% complete in Knowify. Generate a full invoice from Sales Order SO-2045 and send it to the customer. | Confirms: Invoice INV-XXXX created from SO-2045 for full amount $50,000, due date calculated per terms. Asks to confirm send. | New Invoice created in QBO:<br>• Linked to SO-2045<br>• Amount: $50,000<br>• Status: Sent<br>• Due date: per payment terms |
| **Convert Sales Order → Invoice** (Progress / Partial) | The Demolition phase is marked 100% finished. Generate a progress invoice for $5,000 against Sales Order SO-2045. | Draft invoice presented for $5,000 against SO-2045. LLM shows a summary and asks the user to review before sending. | Progress Invoice created in QBO:<br>• Linked to SO-2045<br>• Amount: $5,000 (partial)<br>• Remaining balance tracked on SO<br>• Status: Draft (pending send) |
| **Create & Send Estimate** | Create an estimate for Steve for $1,000 for yard work and send it to him. | Estimate EST-XXXX created for Steve, $1,000 for yard work. Confirms sent to steve@email.com with approval link. | New Estimate in QBO:<br>• Customer: Steve<br>• Amount: $1,000<br>• Service: Yard Work<br>• Status: Sent |
| **Update Estimate** | Use my last estimate for Steve, add a line item for plants for $500 and send the updated version. | Retrieves last estimate for Steve. Adds plants $500. Updated total shown: $1,500. Asks to confirm send. | Existing Estimate updated in QBO:<br>• New line item: Plants $500<br>• Updated total: $1,500<br>• Status: Re-sent |
| **Send Invoice with Payment Link** | Send invoice INV-3098 to the customer with a Pay Now link. | Confirms invoice INV-3098 sent to customer email with QB Payments link. Delivery method: email. | Invoice status updated to 'Sent' in QBO.<br>QB Payments link generated and embedded in email.<br>Delivery log recorded. |
| **Create Payment Link and send it to customer** | Create a Payment Link for INV-3098 and send it to the customer with a Pay Now link. | Generates a payment link and sends it to the customer.
Delivery method: email. | Invoice status updated to 'Sent' in QBO. QB Payments link generated and embedded in email. Delivery log recorded. |
| **Get Payment Link for Invoice** | Get a Payment link for INV-3098 | Retrieves payment link for the Inv-3098 | QBO Payment link retrieved. |

---

## Partner feedback

Feedback, questions, and bug reports should be directed to your assigned Intuit Solution Engineer.
