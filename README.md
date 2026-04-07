# Lakeflow Connect — Salesforce Ingestion Pipeline (Databricks)

A step-by-step guide to ingesting Salesforce data into Databricks Delta Lake using **Lakeflow Connect** and the **Databricks Python SDK**, entirely from a Databricks notebook.

---

## Prerequisites

| Requirement | Detail |
|---|---|
| Databricks workspace | Unity Catalog must be enabled |
| Serverless compute | Must be enabled on your workspace |
| Databricks SDK | v0.67.0+ (`databricks-sdk`) |
| Salesforce Connected App | Required for OAuth credentials |
| Databricks Secret Scope | To store credentials securely |

---

## Architecture Overview

```
Salesforce (Account Object)
        │
        ▼
Lakeflow Connect (UC Connection)
        │
        ▼
Databricks Ingestion Pipeline (DLT)
        │
        ▼
Unity Catalog Delta Table
(my_catalog.salesforce_raw.Account)
```

---

## Step 1 — Create a Salesforce Connected App

You need OAuth credentials from Salesforce. This is a one-time setup.

Navigate to Salesforce Setup (gear icon, top right), then go to App Manager and create a New Connected App. Give it a name like `DatabricksLakeflow`, enable OAuth Settings, set the Callback URL to the Salesforce login endpoint, and add the following OAuth scopes: Full access and Perform requests at any time (refresh token). After saving, wait 2–10 minutes for activation, then copy the **Consumer Key** (your `client_id`) and **Consumer Secret** (your `client_secret`).

---

## Step 2 — Get Your Refresh Token

Using the Connected App credentials from Step 1, along with your Salesforce username, password, and security token, make a password-grant OAuth request to the Salesforce token endpoint. This returns a `refresh_token` which you will store as a secret.

> ⚠️ Your password and security token must be **combined with no space** when making this request. Your security token can be reset from Salesforce Settings → Reset My Security Token.

---

## Step 3 — Store Secrets in Databricks (CLI)

Using the Databricks CLI, store the following three values in your secret scope (`salesforce-scope`):

- `sf-client-id` — your Salesforce Connected App Consumer Key
- `sf-client-secret` — your Salesforce Connected App Consumer Secret
- `sf-refresh-token` — the refresh token obtained in Step 2

> Your existing `sf-username` and `sf-password` secrets are no longer needed — Lakeflow Connect requires OAuth credentials, not username/password.

---

## Step 4 — Create Unity Catalog Connection (Notebook)

Run this **once** in a Databricks notebook. It reads your secrets from the secret scope and registers a Unity Catalog Connection of type `SALESFORCE`. This connection is reused by all future pipelines — no need to recreate it.

The connection requires four options: `client_id`, `client_secret`, `refresh_token`, and `is_sandbox` (set to `"false"` for production, `"true"` for sandbox).

---

## Step 5 — Create the Ingestion Pipeline (Notebook)

Using the Databricks SDK (`WorkspaceClient`), create a Lakeflow ingestion pipeline. The key components are:

- **Connection name** — references the UC Connection created in Step 4
- **Source** — `source_schema` is always `"objects"` for Salesforce; `source_table` is the Salesforce object name (e.g. `Account`)
- **Destination** — your Unity Catalog name and schema where the Delta table will be created
- **Event log location** — a catalog and schema to write pipeline event logs
- **Continuous mode** — set to `False` for batch/triggered runs, `True` for continuous streaming

> ⚠️ Use the `TableSpec` class (not `TableSpecificConfig`) for source/destination mapping. This is the correct class in SDK v0.67.0.

---

## Step 6 — Trigger & Monitor the Pipeline Run (Notebook)

Trigger the pipeline using `start_update` with `full_refresh=False` for incremental mode. Poll the pipeline status every 15 seconds until it reaches one of the terminal states: `COMPLETED`, `FAILED`, or `CANCELED`.

---

## Step 7 — Verify Ingested Data (Notebook)

Once the pipeline completes, read the destination Delta table using `spark.table()` and verify the record count and data looks correct.

---

## Adding More Salesforce Objects

To ingest additional objects such as Contact, Opportunity, or Lead, add more `IngestionConfig` entries to the `objects` list in Step 5. Each entry follows the same pattern — just change the `source_table` value to the Salesforce object name.

---

## Optional: Enable Incremental Formula Fields

By default, Salesforce formula fields are ingested using full snapshots on every run. To enable incremental ingestion for formula fields, add the flag `pipelines.enableSalesforceFormulaFieldsMVComputation: "true"` to the pipeline configuration block.

---

## Incremental Ingestion Behaviour

| Run | Behaviour |
|---|---|
| First run | Full snapshot — all data pulled from Salesforce |
| Subsequent runs | Incremental — only changed records are pulled |
| full_refresh = True | Forces a full re-ingestion from scratch |

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `Failed to retrieve connection` | Connection name is wrong or doesn't exist | Re-run Step 4, verify connection name |
| `must include client_id, client_secret, refresh_token` | Using username/password instead of OAuth | Follow Steps 1–3 to get OAuth credentials |
| `TableSpecificConfig unexpected keyword` | Wrong SDK class used | Use `TableSpec` not `TableSpecificConfig` |
| `InvalidParameterValue` on connection type | SDK version mismatch | Verify installed SDK version in notebook |

---

## SDK Version Reference

This guide is tested with `databricks-sdk == 0.67.0`. You can verify your version by printing `databricks.sdk.__version__` in a notebook cell.

---

## Summary Checklist

- [ ] Salesforce Connected App created (Step 1)
- [ ] Refresh token obtained (Step 2)
- [ ] Secrets stored in Databricks secret scope (Step 3)
- [ ] Unity Catalog connection created — **run once** (Step 4)
- [ ] Ingestion pipeline created (Step 5)
- [ ] Pipeline triggered and completed (Step 6)
- [ ] Data verified in Delta table (Step 7)
