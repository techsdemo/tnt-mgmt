# How This ALZ Terraform Repo Works — Complete Deep Dive

## The Big Picture: 5 Files You Care About, Everything Else Is Plumbing

```
YOUR REPO
├── platform-landing-zone.auto.tfvars  ← ★ THE file you edit (95% of your work)
├── terraform.tfvars.json              ← Subscription IDs + feature flags
├── lib/                               ← Override policies & MG hierarchy
│   ├── alz_library_metadata.json      ← Points to ALZ Library v2026.01.1
│   ├── architecture_definitions/      ← Your management group tree
│   └── archetype_definitions/         ← Add/remove policies per MG
├── main.*.tf                          ← Module calls (you rarely touch these)
├── variables.*.tf                     ← Variable schemas (never edit)
├── modules/config-templating/         ← Template engine (never edit)
└── terraform.tf                       ← Providers + backend (rarely edit)
```

---

## LAYER 1: How Config Flows — The Template Engine

When you write `$${starter_location_01}` in your tfvars, it's NOT standard Terraform. The repo has a **custom template engine** (`modules/config-templating`) that resolves these in **3 cascading passes**:

```
Pass 1: Built-in + Names
  $${starter_location_01}           → "canadacentral"
  $${subscription_id_connectivity}  → "7a944d4f-..."
  $${primary_hub_name}              → "vwan-hub-canadacentral"

Pass 2: Resource Group IDs (can reference Pass 1 results)
  $${primary_connectivity_resource_group_id}
    → "/subscriptions/7a944d4f-.../resourceGroups/rg-hub-canadacentral"

Pass 3: Resource IDs (can reference Pass 1+2 results)
  $${log_analytics_workspace_id}
    → "/subscriptions/42493ae1-.../resourcegroups/rg-management-canadacentral/providers/..."
```

**Why this matters:** You define a name ONCE in `custom_replacements.names`, then reference it everywhere with `$${}`. Change one name → everything updates. The engine serializes your entire tfvars to JSON, runs `templatestring()`, then deserializes back.

### Concrete End-to-End Example: Log Analytics Workspace ID

Here's how a single value flows through all 3 passes:

```
Pass 1 (names):
  management_resource_group_name = "rg-management-$${starter_location_01}"
                                    ↓ resolves to
                                    "rg-management-canadacentral"

Pass 2 (resource_group_identifiers):
  management_resource_group_id = "/subscriptions/$${subscription_id_management}/resourcegroups/$${management_resource_group_name}"
                                  ↓ resolves to
                                  "/subscriptions/42493ae1-.../resourcegroups/rg-management-canadacentral"

Pass 3 (resource_identifiers):
  log_analytics_workspace_id = "$${management_resource_group_id}/providers/.../workspaces/$${log_analytics_workspace_name}"
                                ↓ resolves to
                                "/subscriptions/42493ae1-.../resourcegroups/rg-management-canadacentral/
                                 providers/Microsoft.OperationalInsights/workspaces/law-management-canadacentral"
```

Each pass **stacks on top** of the previous one. By the time you say `$${log_analytics_workspace_id}` anywhere in your config, it already carries the full subscription ID + resource group path + resource name — all assembled from pieces you defined individually.

### How tfvars Feeds Into the Template Engine (Full Pipeline)

```
Step 1: YOUR TFVARS
─────────────────────────────────────────────────
platform-landing-zone.auto.tfvars (line 279):

  virtual_wan_settings = {
    virtual_wan = {
      name = "vwan-$${starter_location_01}"       ← still raw
    }
  }

Terraform auto-loads this because the filename ends in .auto.tfvars.
It maps to a variable with the SAME NAME...


Step 2: VARIABLE DEFINITION (the schema/contract)
─────────────────────────────────────────────────
variables.connectivity.virtual.wan.tf (line 923):

  variable "virtual_wan_settings" {
    type = object({
      virtual_wan = optional(object({
        name     = optional(string)
        location = optional(string)
        ...
      }))
    })
  }

Terraform validates your tfvars value matches this type.
Now var.virtual_wan_settings exists — but still has raw "$${}".


Step 3: FED INTO CONFIG MODULE
─────────────────────────────────────────────────
main.config.tf (line 18):

  module "config" {
    inputs = {
      virtual_wan_settings = var.virtual_wan_settings   ← raw $${}
    }
  }

Passed as part of the "inputs" map into the template engine.


Step 4: TEMPLATE ENGINE RESOLVES IT
─────────────────────────────────────────────────
modules/config-templating/locals.config.tf (line 65-68):

  jsonencode(var.inputs)              → JSON string with $${}
  templatestring(json, replacements)  → JSON string with real values
  jsondecode(result)                  → clean Terraform object

Output: module.config.outputs.virtual_wan_settings
  = { virtual_wan = { name = "vwan-canadacentral" } }   ← RESOLVED ✓


Step 5: CONSUMED BY THE REAL MODULE
─────────────────────────────────────────────────
locals.tf (line 24):
  virtual_wan_settings = merge(module.config.outputs.virtual_wan_settings, ...)

main.connectivity.virtual.wan.tf (line 7):
  module "virtual_wan" {
    virtual_wan_settings = local.virtual_wan_settings   ← clean value
  }

This module creates the actual Azure Virtual WAN resource
named "vwan-canadacentral".
```

The full path: `.auto.tfvars` → `variable` definition → `main.config.tf` → template engine → `locals.tf` → real module. The variable definition file is the **gatekeeper** that validates the shape of your input before it enters the pipeline.

---

## LAYER 2: The 5 Modules — What Each Deploys

The repo has exactly **5 module calls** in the `main.*.tf` files. Here's what each does:

### Module 1: `module "config"` → Template Engine

- **File:** `main.config.tf`
- **Source:** `./modules/config-templating` (local)
- **What it does:** Takes your raw tfvars with `$${}` placeholders → outputs fully resolved values. Every other module reads from `module.config.outputs.*`.

### Module 2: `module "resource_groups"` → Pre-creates Resource Groups

- **File:** `main.resource.groups.tf`
- **Source:** `Azure/avm-res-resources-resourcegroup/azurerm` v0.2.1
- **Provider:** `azurerm.connectivity` (connectivity subscription)
- **What it does:** Creates resource groups in the connectivity subscription BEFORE the connectivity module runs. Uses `for_each` over `connectivity_resource_groups` from your tfvars.

**Your config creates these RGs:**

| Key | Name | Enabled? |
|-----|------|----------|
| `ddos` | `rg-hub-ddos-canadacentral` | ❌ (ddos disabled) |
| `vwan` | `rg-hub-vwan-canadacentral` | ✅ |
| `vwan_hub_primary` | `rg-hub-canadacentral` | ✅ |
| `dns` | `rg-hub-dns-canadacentral` | ❌ (private DNS disabled) |

### Module 3: `module "management_resources"` → Log Analytics + Monitoring

- **File:** `main.management.resources.tf`
- **Source:** `Azure/avm-ptn-alz-management/azurerm` v0.9.0
- **Provider:** default `azurerm` (management subscription)
- **Conditional:** `var.management_resources_enabled` (= `true`)

**Creates:**

- Log Analytics Workspace (`law-management-canadacentral`)
- Resource Group (`rg-management-canadacentral`)
- Data Collection Rules: Change Tracking, Defender SQL, VM Insights
- User-Assigned Managed Identity for Azure Monitor Agent (`uami-management-ama-canadacentral`)
- Sentinel onboarding (if configured)

#### What Each Resource Does

- **Log Analytics Workspace (LAW)** — the central log database. ALL diagnostic policies point here via `$${log_analytics_workspace_id}`. Activity logs, resource diagnostic logs, and VM guest logs all flow into this single workspace.
- **UAMI (AMA)** — a managed identity that Azure Monitor Agent uses on VMs to authenticate when pushing data to LAW. Policies auto-assign this identity to VMs.
- **DCR: Change Tracking** — tells AMA to collect file changes, registry changes, software inventory, services/daemons for security auditing.
- **DCR: Defender SQL** — tells AMA to collect SQL Server security events and audit logs for Defender for SQL.
- **DCR: VM Insights** — tells AMA to collect CPU, memory, disk, network performance counters and process dependency maps.

#### Three Types of Logs That Flow Into the LAW

```
                    ┌──────────────────────────────┐
                    │     Log Analytics Workspace   │
                    │  (law-management-canadacentral)│
                    └───────┬──────┬──────┬────────┘
                            │      │      │
              ┌─────────────┘      │      └──────────────┐
              ↑                    ↑                     ↑
   ┌──────────┴──────────┐ ┌──────┴──────────┐ ┌────────┴──────────┐
   │  Activity Logs      │ │ Resource Diags  │ │  VM Guest Logs    │
   │  (control plane)    │ │ (data plane)    │ │  (inside OS)      │
   │  Who did what       │ │ FW, KV, SQL,    │ │  AMA + DCRs       │
   │                     │ │ NSG, etc.       │ │                   │
   │  Policy:            │ │ Policies:       │ │  Policies:        │
   │  Deploy-AzActivity  │ │ Deploy-Diag-*   │ │  Deploy-VM-*      │
   │  -Log               │ │ per resource    │ │  Deploy-VMSS-*    │
   └─────────────────────┘ └─────────────────┘ └───────────────────┘
```

#### Active Monitoring Policies (Your Config)

| Policy | Platform MG | Landing Zones MG | Status |
|--------|-----------|-----------------|--------|
| `Deploy-VM-ChangeTrack` (change tracking) | ✅ Active | ✅ Active | AMA installs + DCR |
| `Deploy-VM-Monitoring` (VM insights) | ✅ Active | ✅ Active | AMA installs + DCR |
| `Deploy-MDFC-DefSQL-AMA` (Defender SQL) | ✅ Active | ✅ Active | AMA installs + DCR |
| `Deploy-VMSS-ChangeTrack` (VMSS) | ✅ Active | ✅ Active | Same for scale sets |
| `Deploy-VMSS-Monitoring` (VMSS) | ✅ Active | ✅ Active | Same for scale sets |
| `Deploy-vmArc-ChangeTrack` (Arc VMs) | ❌ Removed | ❌ Removed | No Arc VM monitoring |
| `Deploy-vmHybr-Monitoring` (Hybrid VMs) | ❌ Removed | ❌ Removed | No hybrid VM monitoring |
| `Enable-DDoS-VNET` (DDoS on VNets) | ✅ Active | ❌ Removed | DDoS disabled in config |

#### Defender for Cloud Configuration

Modified at root (`alz`) MG via `Deploy-MDFC-Config-H224`:

| Defender Plan | Status |
|--------------|--------|
| Servers | ✅ Enabled |
| Server Vuln Assessments | ✅ Enabled |
| SQL | ✅ Enabled |
| Storage | ✅ Enabled |
| Key Vault | ✅ Enabled |
| SQL on VMs | ✅ Enabled |
| CSPM | ✅ Enabled |
| App Services | ❌ Disabled |
| Containers | ❌ Disabled |
| ARM | ❌ Disabled |
| Open Source DBs | ❌ Disabled |
| Cosmos DB | ❌ Disabled |

#### LAW Cost Model

| Model | Per GB Cost (approx.) | Best For |
|-------|----------------------|----------|
| Pay-As-You-Go | ~$2.76/GB | < 100 GB/day |
| 100 GB/day commitment | ~$2.28/GB (~17% savings) | 100-200 GB/day |
| 200 GB/day commitment | ~$2.09/GB (~24% savings) | 200-500 GB/day |
| 500 GB/day commitment | ~$1.91/GB (~31% savings) | 500+ GB/day |

- **Retention:** First 31 days free, then ~$0.12/GB/month (interactive) or ~$0.02/GB/month (archive)
- **Queries:** Free — no charge to search/read
- Your config uses all defaults: Pay-As-You-Go, 31-day retention

#### Single LAW: RBAC for App Owners

The LAW defaults to `resource-context` mode (`allow_resource_only_permissions = true`). This means:

```
Platform team:  Log Analytics Reader on LAW    → sees everything
SOC team:       Log Analytics Reader on LAW    → sees everything (for Sentinel)
Network team:   Reader on Connectivity sub     → sees only FW/VPN/NSG logs
App Owner A:    Reader on their subscription   → sees only their VMs, apps, etc.
App Owner B:    Reader on their subscription   → sees only their stuff
```

No extra LAW permissions needed for app owners. Azure auto-filters logs based on what resources the user has Reader access to. They go to their VM → Logs blade → see only their own data.

#### How Azure Policy Works in ALZ

##### Policy Types

| Type | What it does | Auto-fixes existing? |
|------|-------------|---------------------|
| `Deny` | **Blocks** non-compliant actions in real-time | N/A (prevention) |
| `DeployIfNotExists` (DINE) | **Creates** missing config on NEW resources | ❌ Needs remediation task |
| `Modify` | **Changes** resource properties | ❌ Needs remediation task |
| `Audit` / `AuditIfNotExists` | **Reports** non-compliance only | N/A (never modifies) |

##### Why DINE Policies Show Non-Compliant

DINE policies (like `Deploy-AzActivity-Log`) auto-trigger on **new resources only**. Resources that existed before the policy was assigned are flagged non-compliant but **not auto-fixed**. You must create a **remediation task**:

- **Portal:** Policy → Assignments → select policy → Create Remediation Task
- **CLI:** `az policy remediation create --name "fix-it" --policy-assignment "Deploy-AzActivity-Log" --management-group "alz"`
- **Alternative:** Moving a subscription out and back into the MG also triggers the DINE policy

##### What Deploy-AzActivity-Log Creates (via Remediation)

Creates a subscription-level diagnostic setting that sends all Activity Log categories to your LAW:
- Administrative (resource CRUD), Security (Defender alerts), ServiceHealth (Azure incidents), Alert (Monitor alerts), Recommendation (Advisor), Policy (compliance changes), Autoscale, ResourceHealth

Activity Log = control plane only (WHO did WHAT). It does NOT capture data plane (firewall traffic logs, blob reads, OS events — those come from resource diagnostics and AMA).

##### What Happens When a Policy Is Deleted

```
Delete a DINE policy:
  → Existing deployments: STAY (not cleaned up)
  → New resources: NO enforcement
  → Compliance visibility: GONE

Delete a Deny policy:
  → Previously blocked actions: NOW ALLOWED

Delete an Audit policy:
  → Compliance data disappears, nothing else changes
```

Policies enforce **intent**, not resource lifecycle. They create things but don't clean up when removed.

#### Defender for Cloud Continuous Export (ASC Export)

The `Deploy-MDFC-Config-H224` policy creates a resource group `rg-asc-export-canadacentral` in **each subscription** containing a `Microsoft.Security/automations` resource. This is the "pipe" that sends Defender data to the LAW:

```
Per subscription:
  rg-asc-export-canadacentral
  └── ExportToWorkspace (Microsoft.Security/automations)
      Exports: Security alerts, recommendations, secure scores,
               compliance assessments (real-time + weekly snapshots)
      Destination: law-management-canadacentral (same central LAW)
```

The RG is separate for RBAC isolation and to prevent accidental deletion. No data is stored in the RG — it only holds the export configuration. All data flows to the central LAW.

### Module 4: `module "virtual_wan"` → ALL Networking

- **File:** `main.connectivity.virtual.wan.tf`
- **Source:** `Azure/avm-ptn-alz-connectivity-virtual-wan/azurerm` v0.13.5
- **Provider:** `azurerm.connectivity` (connectivity subscription)
- **Conditional:** `connectivity_type == "virtual_wan"` (yours = ✅)

**Creates (per virtual hub):**

- Virtual WAN (`vwan-canadacentral`)
- Virtual Hub (`vwan-hub-canadacentral`, 10.0.0.0/22)
- Azure Firewall + Firewall Policy (`fw-hub-canadacentral`)
- ExpressRoute Gateway (`vgw-hub-er-canadacentral`)
- VPN Gateway (`vgw-hub-vpn-canadacentral`)
- Sidecar VNet (`vnet-sidecar-canadacentral`, 10.0.4.0/22)
- DDoS Protection Plan (disabled)
- Bastion (disabled)
- Private DNS Zones (disabled)
- Private DNS Resolver (disabled)

> **OR** `module "hub_and_spoke_vnet"` — mutually exclusive, controlled by `connectivity_type`.

#### How the Toggle Works

```hcl
connectivity_type = "virtual_wan"   ← in your tfvars

locals.tf:
  connectivity_virtual_wan_enabled        = true   → module.virtual_wan count = 1
  connectivity_hub_and_spoke_vnet_enabled = false  → module.hub_and_spoke_vnet count = 0
```

Only one networking module runs. They're mutually exclusive.

#### Two Config Sections Feed This Module

1. `virtual_wan_settings` — vWAN-level (shared across all hubs): vWAN name/location/RG, DDoS plan
2. `virtual_hubs.primary` — per-hub settings: firewall, gateways, bastion, DNS, sidecar, spoke connections, routing intents

#### Resource Creation Order

```
1. Virtual WAN (vwan-canadacentral)           ← parent container
2. Virtual Hub (vwan-hub-canadacentral)       ← the router (~30 min deploy)
3. Firewall Policy (fwp-hub-canadacentral)    ← rules engine
4. Azure Firewall (fw-hub-canadacentral)      ← inspection engine (~10 min)
5. ExpressRoute Gateway                       ← on-prem dedicated (~30 min)
6. VPN Gateway                                ← on-prem IPsec (~30 min)
7. Sidecar VNet (vnet-sidecar-canadacentral)  ← peered to hub
```

#### IP Address Layout

```
10.0.0.0/16 Regional Block
├── 10.0.0.0/22  → Virtual Hub (managed by Azure, no user subnets)
├── 10.0.4.0/22  → Sidecar VNet
│   ├── 10.0.4.0/26   → Bastion subnet (reserved, not deployed)
│   └── 10.0.4.64/28  → DNS Resolver subnet (reserved, not deployed)
└── 10.0.8.0/21+      → Available for spoke VNets
```

#### What the Module CAN vs CANNOT Do

| Feature | Supported? | Notes |
|---------|:----------:|-------|
| Virtual WAN + Hub | ✅ | Core infrastructure |
| Azure Firewall + Policy settings | ✅ | DNS proxy, IDS, TLS, threat intel |
| **Firewall Rules** (allow/deny/DNAT) | ❌ | **Not in module** — manage separately via portal, separate TF, or Azure Policy |
| Spoke VNet connections | ✅ | `virtual_network_connections` (empty in your config) |
| Routing intents | ✅ | `routing_intents` (not configured in TF — you did via portal) |
| ER/VPN Gateways | ✅ | Both deployed |
| VPN Sites + connections | ✅ | `vpn_sites`, `vpn_site_connections` |
| ER Circuit connections | ✅ | `express_route_circuit_connections` |
| P2S VPN | ✅ | `p2s_gateways` |
| Sidecar VNet | ✅ | For resources that can't live in the hub |
| Bastion / DNS / DDoS | ✅ | All disabled in your config |
| FW Diagnostic Settings | ✅ | Built into the firewall sub-module |

#### Terraform Drift Risk (Portal Changes Not in Code)

| Feature | In Terraform? | In Azure? | Risk |
|---------|:------------:|:---------:|------|
| Routing intents (all-traffic → FW) | ❌ | ✅ (portal) | `terraform apply` could **remove** it |
| Spoke VNet connections | ❌ | Unknown | If added via portal, same drift risk |
| Firewall rules | ❌ | Unknown | Rules added in portal persist (not managed by TF) |

### Module 5: `module "management_groups"` → Governance Layer

- **File:** `main.management.groups.tf`
- **Source:** `Azure/avm-ptn-alz/azurerm` v0.19.0
- **Provider:** default `azurerm`
- **Conditional:** `var.management_groups_enabled` (= `true`)

**Creates:**

- 11 Management Groups (the ALZ hierarchy)
- ~170 Azure Policy Definitions
- ~50 Policy Set Definitions (initiatives)
- 5 Custom Role Definitions
- All Policy Assignments per management group
- Policy Role Assignments (managed identity RBAC for DeployIfNotExists)
- Subscription placement (which sub goes in which MG)

**This is the hidden iceberg.** It uses the `alz` Terraform provider which loads the **Azure Landing Zones Library** from GitHub at plan time.

#### How It Works — Step by Step

**Step 1:** At `terraform plan` time, the ALZ provider loads the Azure-Landing-Zones-Library (ref `2026.01.1`) from GitHub, then your local `lib/` overrides on top.

**Step 2:** Your architecture definition (`alz_custom.yaml`) defines the MG tree — 11 management groups, each referencing an archetype (e.g., `root_custom`, `platform_custom`).

**Step 3:** Each archetype = base (from GitHub) + your overrides (from `lib/archetype_definitions/`). Example:
```yaml
# root_custom = everything from "root" base archetype, minus Deploy-ASC-Monitoring
base_archetype: root
policy_assignments_to_remove: [Deploy-ASC-Monitoring]
```

**Step 4:** Policy inheritance — a subscription gets ALL policies from its MG AND every parent MG above it. A sub in `corp` gets: `corp` + `landingzones` + `alz` policies.

**Step 5:** `policy_default_values` injects resource IDs (LAW, DCRs, UAMI) into all policies so they know where to send data.

**Step 6:** Subscription placement moves your 4 subs into their designated MGs.

**Step 7:** Auto-creates RBAC role assignments for policy managed identities (DeployIfNotExists policies need identity + roles to create resources).

#### Changing the MG Hierarchy

Edit `lib/architecture_definitions/alz_custom.alz_architecture_definition.yaml`:

**To rename a MG:** Change `display_name` only. Do NOT change `id` (that would destroy/recreate).

**To add a new MG** (e.g., "Regulated"):
1. Add entry to the architecture YAML:
```yaml
- archetypes: ["regulated_custom"]
  display_name: "Regulated"
  exists: false
  id: "regulated"
  parent_id: "landingzones"
```
2. Create `lib/archetype_definitions/regulated_custom.alz_archetype_override.yaml`:
```yaml
base_archetype: landing_zones
name: regulated_custom
policy_assignments_to_add: []
policy_assignments_to_remove: []
```

#### Corp vs Online — Why Two Landing Zone Types?

| MG | Purpose | Key Policy Difference |
|----|---------|----------------------|
| **Corp** | Private/internal workloads connected to on-prem | Gets `Deploy-Private-DNS-Zones` (auto-register private endpoints) |
| **Online** | Internet-facing workloads, no on-prem connectivity | Does NOT get private DNS policies |

Common additions organizations make: **Regulated** (financial/healthcare), **Confidential** (extra encryption), **SAP**, **DevTest** (relaxed policies).

---

## LAYER 3: The ALZ Provider + Library — The Hidden Engine

```
terraform.tf
├── provider "alz" {
│     library_overwrite_enabled = true
│     library_references = [{ custom_url = "${path.root}/lib" }]
│   }
│
lib/alz_library_metadata.json
├── dependencies: platform/alz ref 2026.01.1
│   └── GitHub: Azure/Azure-Landing-Zones-Library
│       ├── policy_definitions/      ← ~170 custom policies
│       ├── policy_set_definitions/  ← ~50 initiatives
│       ├── role_definitions/        ← 5 custom roles
│       └── archetype_definitions/   ← Base archetypes per MG
```

### How the Override System Works

1. The ALZ Library defines base archetypes (e.g., `root` archetype has specific policies + roles)
2. Your `lib/archetype_definitions/root_custom.alz_archetype_override.yaml` says:
   - `base_archetype: root` ← inherit everything from `root`
   - `policy_assignments_to_remove: [...]` ← subtract these
   - `policy_assignments_to_add: [...]` ← add these
3. Your architecture definition (`alz_custom`) references `root_custom` instead of `root`

### Your Current Overrides

| Override File | What's Changed |
|--------------|----------------|
| `root_custom` | Removed: `Deploy-ASC-Monitoring` |
| `platform_custom` | Removed: `Deploy-vmArc-ChangeTrack`, `Deploy-vmHybr-Monitoring` |
| `landing_zones_custom` | Removed: `Deploy-vmArc-ChangeTrack`, `Deploy-vmHybr-Monitoring`, `Enable-DDoS-VNET` |
| `connectivity_custom` | Removed: `Enable-DDoS-VNET` |
| `corp_custom` | Removed: `Deploy-Private-DNS-Zones` |
| All others | No changes |

---

## LAYER 4: The Dependency Chain — Execution Order

```
terraform plan/apply execution:

1. module.config (template engine)
   ↓ outputs resolved values
2. module.resource_groups (creates RGs first)
   ↓ implicit dependency via locals.tf
3. module.management_resources  ──┐
   module.virtual_wan            ──┤ (can run in parallel)
                                   ↓
4. module.management_groups (runs LAST)
   - Waits for management_resources + virtual_wan via
     local.management_group_dependencies
   - This ensures policy assignments that reference
     LAW/firewall/etc. IDs don't fail
```

The dependency chain in `locals.tf` (line 44-48) is critical — it makes management groups deploy **after** connectivity and management resources exist.

---

## LAYER 5: Provider Aliases — Who Deploys Where

| Provider | Alias | Subscription | Used By |
|----------|-------|-------------|---------|
| `azurerm` | (default) | Inherited from CLI login | Management Groups |
| `azurerm` | `management` | `42493ae1-...` | Available (not directly used by modules) |
| `azurerm` | `connectivity` | `7a944d4f-...` | Resource Groups, Virtual WAN |
| `azapi` | (default) | `42493ae1-...` (management) | Available for management |
| `azapi` | `connectivity` | `7a944d4f-...` | Hub-and-spoke only (not vWAN) |
| `alz` | (none) | N/A | Loads ALZ Library at plan time |

---

## The ALZ Subscription Model — Why 4 Subscriptions?

### What This Terraform Does With Each Subscription

| Subscription | Resources Deployed by This TF? | What This TF Does |
|-------------|:-----------------------------:|-------------------|
| Management (`42493ae1`) | ✅ Yes — LAW, DCRs, UAMI | Module 3 deploys monitoring resources |
| Connectivity (`7a944d4f`) | ✅ Yes — vWAN, FW, Gateways | Module 4 deploys networking |
| Identity (`d77a20c5`) | ❌ None | Only placed in `identity` MG for policy inheritance |
| Security (`71164c02`) | ❌ None | Only placed in `security` MG for policy inheritance |

Identity and Security subscriptions are **governance placeholders** — they receive policies from their MG hierarchy but have no resources deployed by this Terraform. You'd deploy workloads into them separately.

### What Should Go In Each (CAF Best Practice)

| Subscription | Intended Workloads |
|-------------|-------------------|
| **Management** | LAW, Sentinel, Automation, DCRs, monitoring agents |
| **Connectivity** | vWAN/Hub VNet, Firewalls, Gateways, DNS, DDoS |
| **Identity** | AD Domain Controllers, Entra Connect, AD FS, identity DNS |
| **Security** | Security tooling, vulnerability scanners, CSPM, ASC export, incident response automation |

### Why Separate Subscriptions?

- **RBAC isolation** — network team can't touch identity resources and vice versa
- **Blast radius** — if one sub is compromised, others aren't affected
- **Cost tracking** — each team's costs are separately visible
- **Quota isolation** — one team can't exhaust another's resource quotas
- **Compliance** — auditors can scope reviews to specific subs

---

## HOW TO CONFIGURE EACH MODULE — Practical Guide

### Changing Networking (Virtual WAN)

Edit `platform-landing-zone.auto.tfvars` → sections:

- `custom_replacements.names` → enable/disable resources, set names, set IP ranges
- `connectivity_resource_groups` → add/remove resource groups
- `virtual_wan_settings` → vWAN-level settings (DDoS, vWAN name)
- `virtual_hubs.primary` → hub-level settings (firewall, gateways, bastion, DNS, sidecar)

**Key toggle flags** (in `custom_replacements.names`):

```hcl
primary_firewall_enabled                              = true   # Azure Firewall
primary_virtual_network_gateway_express_route_enabled = true   # ER Gateway
primary_virtual_network_gateway_vpn_enabled           = true   # VPN Gateway
primary_private_dns_zones_enabled                     = false  # Private DNS
primary_private_dns_resolver_enabled                  = false  # DNS Resolver
primary_bastion_enabled                               = false  # Bastion
primary_sidecar_virtual_network_enabled               = true   # Sidecar VNet
ddos_protection_plan_enabled                          = false  # DDoS
```

### Adding a Second Hub (e.g., East US)

1. Add `"eastus"` to `starter_locations`
2. Add secondary names/IPs in `custom_replacements.names`
3. Add `secondary` key to `virtual_hubs` map
4. Add new RGs to `connectivity_resource_groups`

### Adding Spoke VNet Connections

```hcl
virtual_hubs = {
  primary = {
    virtual_network_connections = {
      spoke1 = {
        name                      = "spoke1-connection"
        remote_virtual_network_id = "/subscriptions/.../providers/Microsoft.Network/virtualNetworks/vnet-spoke1"
      }
    }
    # ...existing config...
  }
}
```

### Adding Routing Intents (Force Traffic Through Firewall)

```hcl
virtual_hubs = {
  primary = {
    routing_intents = {
      all-traffic = {
        name = "all-traffic"
        routing_policies = [
          {
            name                  = "private-traffic"
            destinations          = ["PrivateTraffic"]
            next_hop_firewall_key = "primary"
          },
          {
            name                  = "internet-traffic"
            destinations          = ["Internet"]
            next_hop_firewall_key = "primary"
          }
        ]
      }
    }
    # ...existing config...
  }
}
```

### Changing Management/Monitoring

Edit `management_resource_settings` in `platform-landing-zone.auto.tfvars`:

- Log Analytics workspace name, retention, SKU, quotas
- Data Collection Rules (change tracking, VM insights, Defender SQL)
- Sentinel onboarding
- User-assigned managed identity for AMA

### Changing Governance/Policy

Four options:

1. **Modify policy parameters** → `management_group_settings.policy_assignments_to_modify` in tfvars
2. **Add/remove policies per MG** → edit `lib/archetype_definitions/<mg>_custom.yaml`
3. **Change MG hierarchy** → edit `lib/architecture_definitions/alz_custom.yaml`
4. **Add RBAC role assignments** → uncomment `management_group_role_assignments` in tfvars

### Switching from vWAN to Hub-and-Spoke

```hcl
connectivity_type = "hub_and_spoke_vnet"  # instead of "virtual_wan"
```

Then populate `hub_and_spoke_networks_settings` and `hub_virtual_networks` in `terraform.tfvars.json` (currently empty `{}`).

---

## Your Management Group Hierarchy

```
Tenant Root Group (b288268f-...)
└── Azure Landing Zones (alz) ← root_custom archetype
    ├── Platform (platform) ← platform_custom
    │   ├── Management ← management_custom     [sub: 42493ae1]
    │   ├── Connectivity ← connectivity_custom  [sub: 7a944d4f]
    │   ├── Identity ← identity_custom          [sub: d77a20c5]
    │   └── Security ← security_custom          [sub: 71164c02]
    ├── Landing Zones (landingzones) ← landing_zones_custom
    │   ├── Corp ← corp_custom
    │   └── Online ← online_custom
    ├── Sandbox ← sandbox_custom
    └── Decommissioned ← decommissioned_custom
```

---

## Summary: What Happens When You Run `terraform apply`

1. **Config module** resolves all `$${}` tokens → produces clean values
2. **Resource groups** get created in the connectivity subscription
3. **Management resources** deploy: LAW, DCRs, UAMI, Sentinel → management subscription
4. **Virtual WAN** deploys: vWAN, hub, firewall, gateways, sidecar → connectivity subscription
5. **Management groups** deploy last: 11 MGs, ~170 policies, ~50 initiatives, 5 roles, all assignments, subscription placement → tenant scope

**All from editing one file: `platform-landing-zone.auto.tfvars`.**
