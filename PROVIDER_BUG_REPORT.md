# Vault Provider Bug: `set_namespace_from_token = false` not respected in v5.2.0+

## Summary

Starting from `terraform-provider-vault` v5.2.0, the provider setting `set_namespace_from_token = false` is no longer honored during client initialization. This causes namespace duplication (`admin/admin`) for HCP Vault configurations that authenticate in the `admin` namespace and use resource-level `namespace` attributes.

The regression was introduced by PR [#2540](https://github.com/hashicorp/terraform-provider-vault/pull/2540) and affects all versions from v5.2.0 onwards (confirmed on v5.2.1 and v5.4.0).

An open fix exists: PR [#2799](https://github.com/hashicorp/terraform-provider-vault/pull/2799) — "fix: respect set_namespace_from_token=false in setClient" — but has not been merged as of 2026-05-29.

**The bug is auth-method-agnostic.** It occurs in the `setClient()` function which runs after any authentication (token, approle, AWS IAM, JWT, etc.). The reproduction below uses a simple token for clarity.

---

## Minimal Reproduction

### Prerequisites

- An HCP Vault Dedicated cluster (namespaces require Enterprise/HCP)
- The cluster's admin token (available from HCP portal → Vault cluster → Generate token)
- Terraform >= 1.3

### Step 1: Create the reproduction directory

```bash
mkdir vault-namespace-bug && cd vault-namespace-bug
```

### Step 2: Create `versions.tf`

```hcl
terraform {
  required_version = ">= 1.3"
  required_providers {
    vault = {
      source  = "hashicorp/vault"
      version = "5.1.0"  # Change to "5.2.0" or later to reproduce the bug
    }
  }
}
```

### Step 3: Create `main.tf`

```hcl
variable "vault_address" {
  type        = string
  description = "HCP Vault cluster address (e.g. https://my-cluster.hashicorp.cloud:8200)"
}

variable "vault_token" {
  type        = string
  sensitive   = true
  description = "HCP Vault admin token"
}

# Provider authenticates with a token that is scoped to the "admin" namespace.
# We explicitly set set_namespace_from_token = false to tell the provider
# NOT to derive its operating namespace from the token.
provider "vault" {
  address                  = var.vault_address
  token                    = var.vault_token
  set_namespace_from_token = false
}

# Create a child namespace under "admin".
# We specify namespace = "admin" because that's where we want to create it.
resource "vault_namespace" "test" {
  namespace = "admin"
  path      = "bug-repro"
}

# Create a policy inside the child namespace.
# This uses the full path returned by vault_namespace.
resource "vault_policy" "test" {
  namespace = vault_namespace.test.path_fq
  name      = "test-policy"
  policy    = <<-EOT
    path "secret/data/*" {
      capabilities = ["read"]
    }
  EOT
}

# Now try to read something back from the "admin" namespace.
# This is where the bug manifests: the provider prepends "admin" from the token
# to the resource's namespace = "admin", resulting in "admin/admin".
data "vault_policy_document" "lookup_test" {
  rule {
    path         = "secret/data/*"
    capabilities = ["read"]
  }
}

resource "vault_policy" "admin_level" {
  namespace = "admin"
  name      = "admin-level-test-policy"
  policy    = data.vault_policy_document.lookup_test.hcl
}
```

### Step 4: Apply with v5.1.0 (works — creates baseline state)

```bash
terraform init
terraform apply -var="vault_address=https://YOUR-CLUSTER.hashicorp.cloud:8200" \
                -var="vault_token=hvs.YOUR_TOKEN"
```

**Expected**: Apply succeeds, resources are created. This establishes the Terraform state with resources managed under the correct namespaces.

### Step 5: Upgrade to v5.2.0+ and apply (breaks)

Change `versions.tf`:
```hcl
version = "5.2.1"  # or any version >= 5.2.0
```

```bash
terraform init -upgrade
terraform apply -var="vault_address=https://YOUR-CLUSTER.hashicorp.cloud:8200" \
                -var="vault_token=hvs.YOUR_TOKEN"
```

**Note**: A `terraform plan` alone may appear to succeed (showing no changes or minor drift), because the namespace duplication only manifests when the provider actually makes API calls to read/write resources. The error occurs during `apply` when the provider attempts to refresh state or modify resources.

**Actual result**: Errors with namespace duplication:

```
│ Error: error deleting from Vault: Error making API request.
│
│ Namespace: admin/admin/bug-repro
│ URL: DELETE https://YOUR-CLUSTER.hashicorp.cloud:8200/v1/sys/policies/acl/test-policy
│ Code: 404. Errors:
│
│ * no handler for route "admin/bug-repro/sys/policies/acl/test-policy". route entry not found.
│
╵
╷
│ Error: error writing to Vault: Error making API request.
│
│ Namespace: admin/admin
│ URL: PUT https://YOUR-CLUSTER.hashicorp.cloud:8200/v1/sys/policies/acl/admin-level-test-policy
│ Code: 404. Errors:
│
│ * no handler for route "admin/sys/policies/acl/admin-level-test-policy". route entry not found.
│
│   with vault_policy.admin_level,
│   on main.tf line 50, in resource "vault_policy" "admin_level":
│   50: resource "vault_policy" "admin_level" {
│╵
```

Notice the namespace values in the errors:
- `admin/admin/bug-repro` — the provider prepended `admin` (from token) to the state's `admin/bug-repro`
- `admin/admin` — the provider prepended `admin` (from token) to the resource's `namespace = "admin"`

### Step 6: Cleanup

```bash
# Downgrade back to 5.1.0 to be able to manage the resources again
# Change versions.tf back to version = "5.1.0"
terraform init -upgrade
terraform destroy -var="vault_address=..." -var="vault_token=..."
```

---

## Why This Happens

### The correct behavior (v5.1.0)

1. Provider receives a token scoped to `admin` namespace
2. `set_namespace_from_token = false` → provider does NOT set its internal namespace from the token
3. Resource specifies `namespace = "admin"` → API call uses `X-Vault-Namespace: admin`
4. Request reaches the correct endpoint ✓

### The broken behavior (v5.2.0+)

1. Provider receives a token scoped to `admin` namespace
2. `set_namespace_from_token = false` is configured BUT...
3. PR #2540 added code in `setClient()` that **unconditionally** sets the provider namespace:
   ```go
   // internal/provider/meta.go, lines 373-375 (added by PR #2540):
   if err := d.Set(consts.FieldNamespace, namespace); err != nil {
       return err
   }
   ```
   This runs regardless of `set_namespace_from_token` setting.
4. Provider's internal namespace is now `admin` (from the token)
5. Resource specifies `namespace = "admin"` → gets appended → `admin/admin`
6. API call uses `X-Vault-Namespace: admin/admin` → 404 ✗

---

## Real-World Impact

In our production setup, we manage ~25 team environments on HCP Vault using Terraform Cloud. Our modules use absolute namespace paths (e.g., `admin/digital-tech/teamname`) in resource attributes. With the provider also prepending `admin`, all paths become invalid.

Symptoms include:
- `no handler for route "admin/identity/lookup/group"` (404)
- `no Identity Entity found` / `no Identity Group found`
- Provider plugin crashes (nil pointer dereference when receiving unexpected 404 responses)
- Complete inability to run `terraform plan` on any workspace

---

## Affected Versions

| Version | Status |
|---------|--------|
| 5.1.0   | ✅ Works correctly |
| 5.2.0   | ❌ Broken (PR #2540 merged) |
| 5.2.1   | ❌ Broken + plugin crashes |
| 5.3.0   | ❌ Broken |
| 5.4.0   | ❌ Broken |
| 5.5.0 – 5.9.0 | ❌ Presumed broken (same code path) |

---

## The Fix (PR #2799 — Open)

PR [#2799](https://github.com/hashicorp/terraform-provider-vault/pull/2799) by @bpaquet (opened Feb 27, 2026) adds a check for `set_namespace_from_token` before setting the namespace in `setClient()`:

```go
// Only set namespace from token if set_namespace_from_token is true (or not explicitly set to false)
if setNamespaceFromToken {
    if err := d.Set(consts.FieldNamespace, namespace); err != nil {
        return err
    }
}
```

This restores the pre-5.2.0 behavior when `set_namespace_from_token = false` is configured.

---

## Timeline

| Date | Event |
|------|-------|
| 2023-06-14 | Issue [#1903](https://github.com/hashicorp/terraform-provider-vault/issues/1903) opened — namespace duplication first reported (v3.16.0) |
| 2023-11-01 | `set_namespace_from_token` option added in v3.22.0 as a workaround |
| 2024-09-21 | We confirmed `set_namespace_from_token = false` fixes the issue, upgraded to v4.x |
| 2025-07-09 | v5.1.0 released — works correctly with our configuration |
| 2025-08-18 | PR #2540 merged — introduces the regression |
| 2025-08-18 | v5.2.0 released — `set_namespace_from_token = false` broken |
| 2025-08-19 | v5.2.1 released — same issue + additional panic fix attempts |
| 2026-02-27 | PR [#2799](https://github.com/hashicorp/terraform-provider-vault/pull/2799) opened — fix for this exact issue |
| 2026-05-29 | PR #2799 still not merged (3 months) |

---

## Request

Please prioritize the merge of PR [#2799](https://github.com/hashicorp/terraform-provider-vault/pull/2799) which correctly fixes this regression by respecting `set_namespace_from_token = false` in the `setClient()` function.

---

## References

- Issue [#1903](https://github.com/hashicorp/terraform-provider-vault/issues/1903) — Original namespace duplication report
- Issue [#2331](https://github.com/hashicorp/terraform-provider-vault/issues/2331) — Related upgrade blocker
- PR [#2540](https://github.com/hashicorp/terraform-provider-vault/pull/2540) — The change that introduced this regression
- PR [#2799](https://github.com/hashicorp/terraform-provider-vault/pull/2799) — The pending fix
- [Vault provider docs: set_namespace_from_token](https://registry.terraform.io/providers/hashicorp/vault/latest/docs#set_namespace_from_token)

