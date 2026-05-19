# Multi-Tenant Data Segregation for Databricks Genie -- Solution Design

## Research Sources

This analysis draws from:
- Databricks official documentation (AWS, Azure, GCP)
- Josh Rosenberg, "Embedding Genie API for a Multi-Tenant Application" (Medium, March 2026)
- Genie Ready FAQ (internal Databricks document)
- GenierAILS open-source project (github.com/databricks-solutions/genierails)
- Databricks blog: "Access Genie Everywhere" -- OBO/U2M/M2M OAuth patterns
- Unity Catalog ABAC and row filter documentation
- Databricks Apps OBO user authorization documentation
- Databricks community: multi-tenant architecture articles and session variable discussions

---

## Approaches Evaluated

### Approach A: SP-per-Tenant + ABAC Row Filters (Databricks Recommended)

This is the pattern recommended by Databricks engineering (Josh Rosenberg article, March 2026).

**How it works:**
- Each tenant gets a dedicated service principal
- Genie API calls use that SP's OAuth M2M credentials
- Unity Catalog ABAC row filters evaluate `current_user()` at query time
- Since each SP is a distinct identity, ABAC automatically restricts data to that tenant's rows
- No application-level filtering needed -- Databricks handles isolation at the SQL execution layer

**Pros:**
- Databricks-recommended architecture
- Hard security boundary at the platform level
- Genie works fully: NL answers, SQL results, text summaries -- all tenant-scoped
- ABAC policies inherit down the object hierarchy (catalog -> schema -> table)
- GenierAILS provides this as code (ACLs, instructions, benchmarks, CI/CD)

**Cons:**
- Requires one SP per tenant
- With hundreds of sub-tenants, SP management overhead increases
- Needs automation for SP lifecycle (creation, rotation, cleanup)

**Assessment:** This is the gold standard. The concern about "hundreds of SPs" should be re-evaluated -- SP creation/management can be fully automated via Databricks Account API. Hundreds is well within operational limits. **If this is feasible, stop here.**

---

### Approach B: OBO OAuth + Row Filters + User Provisioning

**How it works:**
- Provision each tenant/sub-tenant as a Databricks **user** (not SP) via SCIM or AIM
- Create a mapping table: `user_email` -> `tenant_id`
- Create row filter functions using `session_user()` that look up the mapping table
- The application uses On-Behalf-Of (OBO) OAuth to call Genie API with per-user tokens
- Genie executes queries with the end user's data credentials
- Row filters fire automatically -- every result is tenant-scoped

**Row filter example:**

```sql
-- Mapping table
CREATE TABLE governance.security.user_tenant_map (
  user_email STRING,
  tenant_id  STRING
);

-- Row filter function (runs with definer's rights on mapping table,
-- but session_user() returns the invoker's identity)
CREATE FUNCTION governance.security.filter_by_tenant(tid STRING)
RETURN tid IN (
  SELECT tenant_id
  FROM governance.security.user_tenant_map
  WHERE user_email = session_user()
);

-- Apply to table
ALTER TABLE datalake_ireland_dev.ddm.dim_devices
SET ROW FILTER governance.security.filter_by_tenant ON (ext_tenant_id);
```

**OBO OAuth flow:**

```
1. Tenant user authenticates to your app (your IdP)
2. Your app registers as OAuth app in Databricks Account Console
3. User grants your app permission -> Databricks returns auth code
4. Your backend exchanges auth code for user-specific access + refresh tokens
5. Backend creates a scoped WorkspaceClient with user's token:

   w = WorkspaceClient(host="https://...", token=user_access_token)
   message = w.genie.create_message_and_wait(
       space_id="<space-id>",
       conversation_id=conv.conversation_id,
       content="user question"
   )

6. Genie executes with user's data credentials -> row filters apply
7. Return results to tenant
```

**Pros:**
- Hard security boundary (UC row filters + per-user identity)
- Genie works fully (NL answers + SQL + results all tenant-scoped)
- Users scale better than SPs -- SCIM/AIM handles thousands
- Mapping table makes tenant assignment dynamic (update rows, not infra)
- Row filter hybrid model: definer's rights on mapping table, invoker's identity for `session_user()`

**Cons:**
- Requires tenant users to exist in Databricks (SCIM/AIM provisioning)
- Requires tenant users to exist in your IdP (for OBO flow)
- OBO OAuth adds implementation complexity (token management, refresh)
- Each user needs CAN VIEW on Genie space + SELECT on UC data objects

**Assessment:** Best option if tenants can be provisioned as Databricks users. More scalable than SP-per-tenant for hundreds of sub-tenants.

---

### Approach B2: Databricks App with Native OBO Token Forwarding + Row Filters

A variant of Approach B that eliminates custom OAuth plumbing by using Databricks Apps' built-in user authorization.

**How it works:**
- Deploy the tenant-facing application as a **Databricks App**
- Enable **user authorization** on the app -- Databricks automatically forwards the logged-in user's access token via HTTP headers
- The app extracts the token and creates a user-scoped WorkspaceClient to call Genie
- Row filters on tables enforce tenant isolation via `session_user()`
- No custom OAuth registration, no token exchange, no refresh token management

**Token forwarding example:**

```python
# In a Databricks App (e.g., Gradio, Streamlit, Flask)
from databricks.sdk import WorkspaceClient

def handle_query(request):
    # Databricks forwards the user's token automatically
    user_token = request.headers.get("x-forwarded-access-token")

    # Create user-scoped client
    w = WorkspaceClient(
        host="https://dbc-0d6ca8a5-c7ce.cloud.databricks.com",
        token=user_token
    )

    # Genie call runs with user's identity -> row filters apply
    message = w.genie.create_message_and_wait(
        space_id="<space-id>",
        conversation_id=conv.conversation_id,
        content=user_question
    )
    return message
```

**Pros:**
- All the benefits of Approach B
- No custom OAuth app registration or token exchange -- Databricks handles it
- Simpler implementation (just read the forwarded token header)
- App permissions control who can access the app, UC controls who sees what data

**Cons:**
- App must be deployed as a Databricks App (not an external standalone app)
- Tenant users still need Databricks identities (SCIM/AIM)
- Requires workspace with user authorization enabled (auto-enabled for compliance security profile)

**Assessment:** Simplest implementation path if the app can live inside Databricks Apps. Combines Approach B's security model with native platform integration. Eliminates the main complexity of Approach B (OAuth plumbing).

---

### Approach C: Session Variables + Row Filters (Single SP)

**How it works:**
- Single SP authenticates all requests
- Before each query, the app runs `SET VAR tenant_id = '<value>'`
- Row filter functions read the session variable
- Filter applies at query time

```sql
DECLARE VARIABLE IF NOT EXISTS tenant_id STRING DEFAULT 'NONE';
SET VAR tenant_id = 'acme-corp';

-- Row filter reads session variable
CREATE FUNCTION filter_by_session_tenant(tid STRING)
RETURN (tid = session.tenant_id);
```

**Critical problem: Genie does not support session variable injection.** The Genie API has no mechanism to run `SET VAR` before the generated query executes. This approach works with the Statement Execution API but **NOT with Genie**.

**Additional risk:** Session variables are mutable. Any SQL in the session can overwrite the variable. This is not a security boundary -- it is an application convention. Databricks lacks a tamper-proof session context (unlike Oracle's `SYS_CONTEXT` or SQL Server's `SESSION_CONTEXT`).

**Assessment:** Not viable for Genie. Could supplement other approaches for direct SQL execution paths.

---

### Approach D: Genie `additional_context` Parameter

The Genie API accepts an `additional_context` field per message. You could inject: `"Only return data where tenant_id = 'X'"`.

**Problem:** This is an LLM instruction, not a security boundary. Genie may or may not honor it. Explicitly documented: "Genie Spaces do not enforce a hard data boundary."

**Assessment:** Useful as a UX hint (guides SQL generation). Not a security mechanism.

---

## Recommendation

### Primary: Approach A (SP-per-Tenant + ABAC)

If the SP count can be managed (and hundreds is manageable with automation):
- This is the Databricks-recommended architecture
- GenierAILS provides it as infrastructure-as-code
- Hardest security boundary with least custom code

### Alternative: Approach B/B2 (OBO + Row Filters + User Provisioning)

If tenant identities can be provisioned via SCIM/AIM:
- More scalable for large sub-tenant counts
- B2 (Databricks App) is simplest -- no custom OAuth, just token forwarding
- B (custom app with OBO) gives more flexibility if the app must live outside Databricks
- Row filters with mapping table provide dynamic, maintainable isolation

### Supplementary: Approach D (additional_context)

Regardless of the primary approach, use `additional_context` in Genie API calls to guide SQL generation toward tenant-relevant queries. This improves result quality but must never be relied on for security.

---

## Decision Matrix

| Criteria                          | A: SP+ABAC      | B: OBO+Filters | B2: DBX App+OBO | C: Session Vars | D: Views+Rooms | E: SQL Intercept | F: Context Param |
|-----------------------------------|------------------|-----------------|------------------|-----------------|-----------------|------------------|------------------|
| Hard security boundary            | Yes              | Yes             | Yes              | No              | Yes             | Partial          | No               |
| Genie NL answers tenant-scoped   | Yes              | Yes             | Yes              | N/A             | Yes             | No               | No               |
| Scales to hundreds of tenants    | With automation  | Yes             | Yes              | N/A             | No              | Yes              | Yes              |
| Custom code required             | Minimal          | Moderate        | Minimal          | Moderate        | High            | High             | Minimal          |
| Databricks-recommended           | Yes              | Partially       | Yes              | No              | No              | No               | No               |
| Works with Genie                 | Yes              | Yes             | Yes              | No              | Yes             | Partially        | Partially        |
| App can live outside Databricks  | Yes              | Yes             | No               | Yes             | N/A             | Yes              | Yes              |

---

## Next Steps

1. **Validate SP scalability** -- Confirm whether AirTies can automate SP lifecycle for their tenant count. If yes, go with Approach A.
2. **If not** -- Evaluate whether tenant users can be provisioned via SCIM/AIM for Approach B or B2.
3. **Determine app hosting** -- If the app can be a Databricks App, B2 is the simplest path. If it must be external, Approach B with custom OBO OAuth.
4. **Read the Josh Rosenberg article** in full: "Embedding Genie API for a Multi-Tenant Application" (Medium, March 2026).
5. **Evaluate GenierAILS** (github.com/databricks-solutions/genierails) for infrastructure-as-code governance.
6. **Prototype** the chosen approach against the AirTies workspace with a test tenant.

---

## Key References

- Josh Rosenberg, "Embedding Genie API for a Multi-Tenant Application" (Medium, March 2026)
- Databricks Blog, "Access Genie Everywhere" -- OBO/U2M/M2M OAuth patterns
- Databricks Docs, "Row filters and column masks" -- UC row filter mechanics
- Databricks Docs, "ABAC in Unity Catalog" -- attribute-based access control
- Databricks Docs, "Use the Genie Spaces API" -- conversation API reference
- Databricks Docs, "Configure authorization in a Databricks app" -- OBO token forwarding
- Databricks Community, "Implement fine-grained permissions for Databricks Apps with OBO authorization"
- Genie Ready FAQ (internal) -- governance and data access section
- GenierAILS (github.com/databricks-solutions/genierails) -- ABAC governance as code
- Databricks Community, "Building MultiTenant Architecture on Databricks Platform"
- Data+AI Summit, "Multi-Tenant Architecture at Scale: Fewer Workspaces, Less Admin, No Silos"
