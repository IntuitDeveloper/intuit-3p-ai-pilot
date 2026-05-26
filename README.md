# Intuit AI Pilot — Hosted MCP Server

Connect AI agents and LLM-powered apps to QuickBooks Online through the Model Context Protocol (MCP).

> **Status: PILOT**
>
> **Audience:** AI pilot partners under the Intuit App Partner Program.
>
> **Note:** This is a pilot program and is provided "as is" for testing purposes. Intuit reserves the right to change functionality or charge for any MCP server functionality in the future, although no charge is associated with participation in the pilot program.

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
| Production Base URL | `https://ai-inc.quickbooks.intuit.com/v1/mcp` |
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
| **Process Payment** (QB Payments Not Enabled) | Send invoice INV-3098 with a Pay Now link. QB Payments are not set up. | LLM returns error: QB Payments is not enabled for this account. Suggests steps to enable QB Payments before retrying. | No QBO changes. Invoice remains in current state.<br>Error: `{status: error, code: PAYMENTS_NOT_ENABLED, message: 'QB Payments is not configured for this account'}` |
| **Process Payment** (Customer Pays via Payment Link) | Customer clicks Pay Now link and pays $1,080 via credit card. | LLM confirms: Payment of $1,080 received for INV-3098 via QB Payments. Invoice status: Paid. Payment recorded automatically. | Invoice INV-3098 → Status: Paid.<br>Payment record auto-created in QBO via QB Payments callback:<br>• Method: Credit Card<br>• Amount: $1,080<br>• Processing fee recorded |
| **Process Payment** (3PP Payment via External Processor) | The customer paid invoice INV-3098 via Stripe. Record the payment. | Confirms: $1,080 payment recorded for INV-3098. Payment method: External/Stripe. Invoice status: Paid. | Invoice INV-3098 → Status: Paid.<br>Payment record created:<br>• Method: Other/External<br>• Reference: Stripe<br>• Amount: $1,080 |
| **Process Payment** (Deposit Reconciliation – Multiple Invoices) | Consolidate payments for INV-3098 ($1,080) and INV-3099 ($500) into a single bank deposit of $1,580. | Confirms: Two payments totalling $1,580 consolidated into one deposit. Both invoices marked Paid. Deposit record created in QBO. | Two invoices → Status: Paid.<br>Single deposit record created in QBO:<br>• Total: $1,580<br>• Linked to INV-3098 and INV-3099<br>• Deposit account: company default |
| **Process Payment** (Refund / Credit Memo) | Issue a refund of $500 to customer Smith Headquarters against INV-3098. | Confirms: Refund of $500 issued to Smith Headquarters. Credit memo created and linked to INV-3098. LLM asks for confirmation before processing. | Credit Memo created in QBO:<br>• Customer: Smith Headquarters<br>• Amount: $500<br>• Linked to INV-3098<br>• Customer balance adjusted accordingly |
| **Process Payment** (Payment with Discount Applied) | Record a payment for INV-3098. The customer is taking a 2% early payment discount, so they are paying $1,058.40 instead of $1,080. | Confirms: $1,058.40 payment recorded. Early payment discount of $21.60 applied. Invoice marked Paid with discount noted. | Invoice INV-3098 → Status: Paid.<br>Payment: $1,058.40<br>Discount: $21.60<br>Discount posted to correct GL account (QBO determines account) |
| **Process Payment** (Scope / Auth Failure) | Record a $1,080 payment for invoice INV-3098. The OAuth token does not include payment write scope. | LLM returns auth error: Insufficient permissions to record payments. Required scope: `payments.write`. Guides user to re-authenticate with correct scope. | No QBO changes made.<br>HTTP 403 returned: `{error: insufficient_scope, scope: 'com.intuit.quickbooks.payment'}` |
| **Record Payment** | Record a $1,000 payment for invoice INV-3098. | Confirms payment of $1,000 applied to INV-3098. Invoice status: Paid. Payment date: today. | Payment recorded in QBO:<br>• Invoice INV-3098 → Paid<br>• Amount applied: $1,000<br>• Deposit account: per company default<br>• Payment date: today |
| **Record Payment** (Full – Happy Path) | Record a $1,080 payment for invoice INV-3098. | Confirms: $1,080 applied to INV-3098. Invoice status: Paid. Payment date: today. Deposit account resolved automatically by QBO. | Invoice INV-3098 → Status: Paid.<br>Payment record created:<br>• Amount: $1,080<br>• Date: today<br>• Deposit account: company default<br>• Linked txn: INV-3098 |
| **Record Payment** (Partial Payment) | Record a $500 partial payment against invoice INV-3098 totalling $1,080. | Confirms: $500 applied to INV-3098. Remaining balance: $580. Invoice status remains Open/Partial. LLM surfaces remaining balance clearly. | Invoice INV-3098 remains Open:<br>• Amount paid: $500<br>• Remaining balance: $580<br>• Status: Partial<br>• Payment record created and linked to invoice |
| **Record Payment** (Overpayment) | Record a $1,500 payment for invoice INV-3098 which is for $1,080. | LLM warns: Payment of $1,500 exceeds invoice total of $1,080 by $420. Asks user to confirm or adjust amount before proceeding. | If confirmed:<br>• Invoice INV-3098 → Paid<br>• Unapplied credit of $420 recorded against customer account<br>• Credit visible on customer balance in QBO |
| **Record Payment** (Invoice Not Found) | Record a $1,000 payment for invoice INV-9999. | LLM returns error: Invoice INV-9999 not found in QBO. Suggests user checks the invoice number or lists open invoices. | No QBO changes made.<br>Error returned: `{status: error, code: INVOICE_NOT_FOUND, message: 'Invoice INV-9999 could not be found'}` |
| **Record Payment** (Already Paid Invoice) | Record a $1,080 payment for invoice INV-3098 which is already paid. | LLM returns error: Invoice INV-3098 is already fully paid. No payment recorded. Suggests checking invoice status. | No QBO changes made.<br>Error: `{status: error, code: INVOICE_ALREADY_PAID, message: 'Invoice INV-3098 has already been paid in full'}` |
| **Record Payment** (Multiple Invoices – Bulk) | Record payments for INV-3098 ($1,080), INV-3099 ($500), and INV-3100 ($750). | LLM confirms each payment individually with summary:<br>INV-3098: Paid $1,080 ✓<br>INV-3099: Paid $500 ✓<br>INV-3100: Paid $750 ✓<br>Total recorded: $2,330. Asks to confirm before writing. | Three payment records created in QBO:<br>• INV-3098 → Paid<br>• INV-3099 → Paid<br>• INV-3100 → Paid<br>All linked to respective invoices |

---

## Appendix

### Agent-friendly system prompt

Here is an example of a system prompt / agent instructions for clients integrating with the Intuit MCP server:

```
# QuickBooks MCP Server

You have access to QuickBooks Online via MCP tools prefixed with qbo_.
Use these tools to read and write accounting data (invoices, customers, payments,
estimates, items, vendors, bills) for the currently authorized company.

## When to use these tools
- The user asks about their books, finances, customers, invoices, payments or any QuickBooks data.
- The user wants to create or update a QuickBooks record.

## When NOT to use these tools
- The user is asking a general accounting question with no reference to their own data → answer directly.
- The user mentions a different accounting product (Xero, NetSuite, etc.) → say you only support QuickBooks.

## Operating rules
1. **Always confirm before any write operation** (create_*, update_*, delete_*, void_*, send_*).
   Show the user the exact payload you will submit.
2. **Resolve entities by name first.** If the user says "send invoice to Acme",
   call a customer search tool with a name filter before calling any invoice tool — never guess an ID.
3. **One company at a time.** All tools operate on the authorized realm.
   If the user references a different company, tell them they must re-authorize.
4. **On error, read the error message.** Tool errors are written for you — they tell you what to fix.
   Retry once with the correction; if it fails again, surface the error to the user.
5. **Prefer specific tools over generic ones.** If a purpose-built tool exists
   (e.g., `qbo_sales_get_invoices` with a filter), use it instead of a generic query tool.

## Currency and dates
- All monetary amounts are in the company's home currency unless a `CurrencyRef` is present.
- Dates are ISO 8601 (`YYYY-MM-DD`). Convert relative dates ("last month") before calling tools.
```
---

## Partner feedback

Feedback, questions, and bug reports should be directed to your assigned Intuit Solution Engineer.
