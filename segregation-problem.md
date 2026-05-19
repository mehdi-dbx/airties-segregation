# Multi-Tenant Data Segregation for Databricks Genie -- Problem Statement

## Context

AirTies operates a multi-tenant Databricks workspace where all Unity Catalog tables contain a `tenant_id` column. Tenants and their sub-tenants (potentially hundreds) query data through Databricks Genie via an application layer.

The requirement is to guarantee that every Genie response only returns data belonging to the requesting tenant, filtered by `tenant_id`.

## Constraints

- **Single service principal**: One application identity calls Genie on behalf of all tenants. Identity-based row-level security cannot distinguish between tenants.
- **High tenant volume**: Hundreds of sub-tenants make a one-service-principal-per-tenant approach unmanageable.
- **No per-tenant infrastructure**: Duplicating Genie rooms, views, or service principals per tenant does not scale.

## Ruled-Out Approaches

| Approach | Why It Fails |
|----------|-------------|
| One service principal per tenant | Hundreds of sub-tenants make this unmanageable |
| One Genie room per tenant with pre-filtered views | Does not scale with tenant volume |
| Unity Catalog row filters with `current_user()` | Single SP means all tenants share the same identity |
| Genie room instructions ("always filter by tenant_id") | Relies on LLM compliance -- not a security boundary |

## Workspace Reference

| Resource | Value |
|----------|-------|
| Workspace URL | https://dbc-0d6ca8a5-c7ce.cloud.databricks.com |
| CLI Profile | `airties-customer` |
| SQL Warehouse ID | `6435bcfcf68fd910` |
| Genie Space ID | `01f13e5b64ab1c68937e8d964e779116` |
| Key Table | `datalake_ireland_dev.ddm.dim_devices` |
| Tenant Column | `tenant_id` (aliased as `ext_tenant_id` in dim_devices) |
