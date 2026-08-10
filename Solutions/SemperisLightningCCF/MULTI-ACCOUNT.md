# Multi-account setup (Semperis Lightning CCF)

**Status: reference only. Do not merge.**

One workspace. Many Semperis tenants. Each row must say which tenant it came from.

## The mechanism

CCF merges `addOnAttributes` into every polled event. The DCR then treats them as
normal input fields. Three things must line up or the value is silently dropped:

1. Poller sets `addOnAttributes`.
2. DCR `streamDeclarations` declares the column.
3. DCR `transformKql` projects the column.

Miss any one and the column lands empty. Nothing errors.

## Shipped precedent

| Connector | Discriminator | Source |
|---|---|---|
| Auth0 | `Auth0Domain` | `addOnAttributes` |
| Citrix DaaS | `CitrixCustomerId` | `addOnAttributes` |
| SailPoint IdentityNow | `Org`, `Pod`, `Stack` | API response |

Auth0 is the proof. `Solutions/Auth0/Data Connectors/Auth0_CCP/PollingConfig.json`
sets `"Auth0Domain": "[[parameters('domain')]"`. The DCR declares and projects it.
The Auth0 API never returns that field.

Note the Auth0 transform projects `TenantName` and `Auth0Domain` side by side.
`TenantName` comes from the API's `tenant_name`. `Auth0Domain` comes from
`addOnAttributes`. Same row, two different sources.

SailPoint does not need this. Its API returns `org`, `pod` and `stack` already.
Semperis has no such field, so Semperis must use the Auth0 route.

## What this solution does

Poller — `SemperisLightning_PollerConfig.json`:

```json
"addOnAttributes": {
  "SemperisInstanceName": "[[parameters('connectionName')]",
  "SemperisZone": "[[parameters('zone')[0]]",
  "SemperisDataStream": "Tier0Nodes"
}
```

DCR — `SemperisLightning_DCR.json`, every one of the 7 streams:

```
... SemperisInstanceName=tostring(SemperisInstanceName),
    SemperisZone=tostring(SemperisZone),
    SemperisDataStream=tostring(SemperisDataStream)
```

All 7 destination tables declare the 3 columns.

## The `[[` escape

Use `[[parameters('x')]`. Two opening brackets.

One bracket resolves at solution-install time. The value would be fixed for every
connection. Two brackets defer to connection-creation time, so each connection
gets its own value.

`tests/test_candidate.py::test_multi_account_attributes_reach_ingested_rows`
enforces all of this.

## Operational notes

The DCR is shared. CCF deploys one DCR per connector definition. Every connection
reuses it. Per-connection DCRs are not part of the model. This is why
`addOnAttributes` exists.

This solution runs 6 pollers per connection. So one connection creates 6
`dataConnectors` resources, and the grid shows 6 rows for it. The **Data Stream**
column separates them. `DeleteConnector` removes one row only, so removing a
tenant means deleting all 6.

## Querying

```kusto
LightningTier0Nodes_CL
| summarize Nodes = count() by SemperisInstanceName, SemperisZone
```

Scope any detection to one tenant:

```kusto
LightningAttackPaths_CL
| where SemperisInstanceName == "contoso-prod"
```

## Open

Auth is unresolved. See `docs/call-brief-2026-08-10.md`. The token endpoint takes a
single `apiKey` field. CCF `JwtToken` models a `userName`/`password` pair. The
current config pads with `ccfCompatibility`, which is invented and unverified.
Semperis must confirm the accepted shape before this is real.

`addOnAttributes` is also undocumented on Microsoft Learn. The behaviour above is
taken from shipped connectors, not from a spec.
