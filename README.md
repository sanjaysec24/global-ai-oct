# Stock Quote Agent with MCP and Azure Logic Apps

> **Project status:** In progress  
> **Goal:** Build an AI stock-quote assistant that retrieves stock data, saves successful quotes to Azure Table Storage, and exposes workflow capabilities to an AI agent through the Model Context Protocol (MCP).

## Overview

This project explores an AI agent workflow using Microsoft Foundry, Azure Logic Apps, an external stock-data API, and Azure Table Storage.

The intended flow is:

1. The user asks for a stock quote (for example, `IBM`).
2. The stock-data API returns quote information.
3. The workflow saves the quote to Azure Table Storage.
4. The workflow returns a success or error response.
5. The AI agent explains the result without inventing prices.

**Important architecture note:** The current Azure Logic App workflow was created on the **Consumption (multi-tenant)** plan. Microsoft's documented **remote MCP server** feature for Logic Apps requires a **Standard** Logic App. Therefore, this repository documents the current Consumption workflow and marks the remote MCP connection as pending. Do not claim the Azure Logic App is already an MCP server until the MCP endpoint is configured and tested.

## Architecture

### Current working workflow

<img width="1774" height="887" alt="70cfbcf5-8f90-4bfe-8f8d-c3f40c809b88" src="https://github.com/user-attachments/assets/8706bdbc-2680-408c-a2e4-d56faed6eac2" />


### Intended end-to-end architecture
<img width="1774" height="887" alt="9d4c405a-34b8-41a2-ac43-fd5942b3254b" src="https://github.com/user-attachments/assets/eb4b7f3c-74d2-4f27-b643-bb5337c2c63d" />


The dashed connections are **planned**, not confirmed as configured. The current Consumption workflow is not directly eligible for the documented Logic Apps remote MCP server feature.

## Technology Stack

- **Microsoft Foundry** — agent and model configuration
- **Azure Logic Apps (Consumption)** — current HTTP-triggered workflow
- **Azure Table Storage** — stores successful quote records
- **Alpha Vantage** — stock quote data source (API access required)
- **Model Context Protocol (MCP)** — planned tool connection between the agent and workflow

## Current Implementation

The existing Consumption workflow is named `stock-quote-workflow` (confirm the resource name in your Azure portal before publishing). It contains these actions:

1. **When an HTTP request is received** — accepts a JSON payload.
2. **Insert Entity (V2)** — writes a record to the `StockQuotes` table.
3. **Response** — returns a JSON result to the caller.

The request schema currently used by the storage workflow includes:

```json
{
  "type": "object",
  "properties": {
    "symbol": {
      "type": "string"
    },
    "price": {
      "type": "string"
    }
  }
}
```

Example request body for testing the **storage workflow only**:

```json
{
  "symbol": "IBM",
  "price": "example-value"
}
```

Replace `example-value` with a real quote value from your stock-data API when testing the complete flow. This example does not fetch a quote by itself.

### Storage mapping

| Table field | Workflow value |
|---|---|
| `PartitionKey` | `symbol` from the request |
| `RowKey` | Timestamp/unique key expression configured in the workflow |
| `Symbol` | `symbol` from the request |
| `Price` | `price` from the request |

The exact `RowKey` expression should be verified in the Azure designer before relying on it in production.

## Screenshots

<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/2176896c-5aaf-40ca-a237-e854174ecef7" />
<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/abea759e-4ca9-4958-a5d2-af8299e0d8e6" />
<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/79048106-a1fa-4f0c-905f-622af83a817c" />

<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/d6f0710d-428e-45ce-9419-2db42109fd4b" />
<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/ebaf3e41-7b5b-4042-8812-7e1012330f6b" />
<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/22f06603-edb0-4178-90a7-a6527478062e" />
<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/f9752d57-10e4-42b5-ae9d-63054259358b" />

<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/7abcd39e-a9ac-48c9-8950-872fadba5055" />


<img width="1920" height="1200" alt="image" src="https://github.com/user-attachments/assets/c10d5305-4190-42d7-9eff-36b344c12d5d" />



## Setup and Configuration

### Prerequisites

- An Azure subscription
- Permission to create and manage Logic Apps and Storage resources
- An Alpha Vantage API key for live stock data
- A Microsoft Foundry project if you are building the AI agent
- Git and a GitHub repository (optional, for version control)

### Step 1 — Review the existing Logic App

In the Azure portal, open the existing Consumption Logic App and confirm that its workflow contains:

- `When an HTTP request is received`
- `Insert Entity (V2)` connected to the `StockQuotes` table
- `Response` with status code `200`

Do not recreate these actions if they already exist.

### Step 2 — Configure the request payload

The current workflow expects `symbol` and `price`. The storage workflow only stores the values supplied to it; it does not retrieve live market data on its own.

For a basic storage test, send a POST request to the generated HTTP trigger URL with this JSON body:

```json
{
  "symbol": "IBM",
  "price": "example-value"
}
```

Use the actual trigger URL from your Azure portal. Treat it as a secret because it can contain a signed access token. Do not commit it to GitHub.

### Step 3 — Verify storage

Open the `StockQuotes` table in Azure Storage and confirm that the submitted record appears. Check the Logic App run history to verify the trigger and insert action succeeded.

### Step 4 — Connect live stock data

Configure the stock quote API call (for example, Alpha Vantage `GLOBAL_QUOTE`) so the workflow receives a real symbol and obtains the returned quote before inserting the record. Keep API keys in secure connection settings or secret storage; never put them in source control or screenshots.

### Step 5 — Add MCP support (pending)

The Logic Apps remote MCP server feature documented by Microsoft is for **Logic Apps Standard**, not Consumption. The current Consumption workflow cannot be marked as a completed remote MCP server using that documented feature.

If zero additional charges is a strict requirement, do **not** create or clone to a Standard Logic App until the hosting cost and available credits are understood. Consider a separately hosted MCP server that calls the existing HTTP workflow, or use Standard only if the project owner explicitly approves the cost.

Official documentation: [Create a remote MCP server from a Standard workflow](https://learn.microsoft.com/en-us/azure/logic-apps/create-model-context-protocol-server-standard).

### Step 6 — Connect the agent

Once an MCP endpoint is actually configured and tested, connect the Microsoft Foundry agent to that endpoint using the supported authentication method. Test a complete request end to end:

1. Ask for a quote using a valid ticker symbol.
2. Confirm that a real quote is retrieved.
3. Confirm the record is saved in `StockQuotes`.
4. Confirm the workflow returns success.
5. Confirm the agent reports only values returned by the tools.

Do not describe this step as complete until the remote MCP tool is visible to the agent and an end-to-end test succeeds.

## Security Notes

- Never commit API keys, SAS tokens, signed trigger URLs, connection strings, or credentials.
- Use placeholders in examples and screenshots.
- Avoid exposing customer or subscription secrets in public repositories.
- Check the provider's API limits and terms before repeated requests.
- A successful storage test with a sample price does **not** verify live stock-data retrieval.

## Troubleshooting

| Issue | What it means / what to check |
|---|---|
| Clone to Standard shows “No resources found” | No eligible Standard destination is available in the selected scope. Do not create a potentially billable resource without approval. |
| Storage action succeeds but Response is skipped | The caller may have closed the request or not waited for the response. Test with a client that waits for the HTTP response. |
| No stock price appears in storage | The current storage workflow only stores the `price` value passed in the request; it does not fetch a quote by itself. |
| Foundry agent cannot use the MCP tool | Confirm that an MCP server endpoint is configured, reachable, authenticated, and supported by the selected model/tool configuration. |
| Workflow URL or API key appears in a screenshot | Remove or redact it before sharing or committing the screenshot. |

## Project Status

- [x] Create Consumption Logic App workflow
- [x] Add HTTP request trigger
- [x] Add Azure Table Storage insert action
- [x] Add HTTP response action
- [ ] Verify a complete live stock quote → storage → response test
- [ ] Configure a compatible MCP server endpoint
- [ ] Connect and test the Foundry agent through MCP
- [ ] Add final screenshots and a successful end-to-end run

## References

- [GitHub Markdown formatting](https://docs.github.com/en/get-started/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)
- [Microsoft Learn: Create a remote MCP server from a Standard workflow](https://learn.microsoft.com/en-us/azure/logic-apps/create-model-context-protocol-server-standard)

---

**Author:** Sanjay Kumar  
**Status:** Educational prototype / in progress
