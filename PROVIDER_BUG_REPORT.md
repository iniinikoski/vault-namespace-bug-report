# Vault Provider Bug: `set_namespace_from_token = false` not respected in v5.2.0+

## Summary

Starting from `terraform-provider-vault` v5.2.0, the provider setting `set_namespace_from_token = false` is no longer honored during client initialization. This causes namespace duplication (`admin/admin`) for HCP Vault configurations that authenticate in the `admin` namespace and use resource-level `namespace` attributes.

The regression was introduced by PR [#2540](https://github.com/hashicorp/terraform-provider-vault/pull/2540) and affects all versions from v5.2.0 onwards (confirmed on v5.2.1 and v5.4.0).

An open fix exists: PR [#2799](https://github.com/hashicorp/terraform-provider-vault/pull/2799) — "fix: respect set_namespace_from_token=false in setClient" — however, **PR #2799 is incomplete** and does not fully resolve the issue (see "Why PR #2799 Is Insufficient" below).

**The bug is auth-method-agnostic.** It occurs in the `setClient()` function which runs after any authentication (token, approle, AWS IAM, JWT, etc.). The reproduction below uses a simple token for clarity.

**No state migration is required.** We have verified that the correct fix works seamlessly with existing Terraform state created by v5.1.0. The state format is unchanged; the bug is purely a runtime namespace prepending issue.

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
3. PR #2540 added code in `setClient()` that **unconditionally** sets the provider namespace in two places:
   ```go
   // internal/provider/meta.go — the second "if namespace" block added by PR #2540:
   if namespace != "" {
       // This writes to ResourceData unconditionally:
       if err := d.Set(consts.FieldNamespace, namespace); err != nil {
           return fmt.Errorf("failed to set namespace on provider: %w", err)
       }
       // This sets the namespace on the root client unconditionally:
       client.SetNamespace(namespace)
   }
   ```
   Both of these run regardless of the `set_namespace_from_token` setting.
4. Provider's internal namespace is now `admin` (from the token) — stored in both ResourceData AND the client's namespace header
5. When a resource specifies `namespace = "admin"`, `GetNSClient("admin")` is called
6. `GetNSClient` checks `p.resourceData.GetOk("namespace")` → finds `"admin"` → prepends it: `"admin/admin"`
7. API call uses `X-Vault-Namespace: admin/admin` → 404 ✗

---

## Why PR #2799 Is Insufficient

PR [#2799](https://github.com/hashicorp/terraform-provider-vault/pull/2799) by @bpaquet guards only the `d.Set(consts.FieldNamespace, namespace)` call in the second block. It tracks whether the namespace came from the token and skips the `d.Set` if so.

However, this fix is **incomplete** because:

1. **`client.SetNamespace(namespace)` is still called unconditionally.** The root client gets the `admin` namespace set on it. When `GetNSClient` clones this client, the clone inherits the namespace header. Even though ResourceData is clean, the cloned client already carries `admin` as its base namespace.

2. **`GetResourceDataBool` returns incorrect defaults during certain operations.** The function uses `d.GetRawConfig()` which returns null during state refresh and destroy operations. When null, it falls back to the default value (`true`), effectively ignoring the user's `set_namespace_from_token = false` configuration during those phases.

### The correct fix

The fix must **reset the `namespace` variable to `""`** when `set_namespace_from_token = false` and the namespace was derived from the token. This prevents the entire second block from executing — no `d.Set`, no `client.SetNamespace`:

```go
if namespace == "" && tokenNamespace != "" {
    log.Printf("[WARN] The provider namespace should be set whenever ...")
    namespace = tokenNamespace

    setTokenFromNamespace := GetResourceDataBool(d, consts.FieldSetNamespaceFromToken, "VAULT_SET_NAMESPACE_FROM_TOKEN", true)
    // Fallback: also check d.GetOk() directly, as GetResourceDataBool may
    // return the default (true) when RawConfig is null during refresh/destroy.
    if val, ok := d.GetOk(consts.FieldSetNamespaceFromToken); ok {
        setTokenFromNamespace = val.(bool)
    }

    if setTokenFromNamespace {
        if err := d.Set(consts.FieldNamespace, namespace); err != nil {
            return err
        }
    } else {
        // When set_namespace_from_token=false, do NOT propagate the token's
        // namespace to ResourceData or the client. Reset namespace so the
        // subsequent block is skipped entirely.
        namespace = ""
    }
}

if namespace != "" {
    // This block now only executes when the namespace was explicitly
    // configured on the provider (not derived from the token).
    if err := d.Set(consts.FieldNamespace, namespace); err != nil {
        return fmt.Errorf("failed to set namespace on provider: %w", err)
    }
    client.SetNamespace(namespace)
}
```

This approach:
- Preserves the fix from PR #2540 (VAULT_NAMESPACE env var is still honored when explicitly set)
- Correctly respects `set_namespace_from_token = false`
- Does not require any state migration — existing state from v5.1.0 works as-is
- Works for all auth methods (token, approle, AWS IAM, JWT, etc.)

We have **verified this fix locally** by building a patched provider from `main` and testing against an HCP Vault cluster with both fresh state and existing state from v5.1.0.

---

## State Migration: Not Required

We confirmed that the correct fix works with existing Terraform state created by v5.1.0 without any state migration. The state stores absolute namespace paths (e.g., `"admin"`, `"admin/bug-repro"`). With the fix applied, the provider has no internal namespace prefix, so these paths are used as-is by `GetNSClient` — exactly as they were in v5.1.0.

---

## Real-World Impact

In our production setup, we manage ~25 team environments on HCP Vault using Terraform Cloud. Our modules use absolute namespace paths (e.g., `admin/digital-tech/teamname`) in resource attributes. With the provider also prepending `admin`, all paths become invalid.

Symptoms include:
- `no handler for route "admin/identity/lookup/group"` (404)
- `no Identity Entity found` / `no Identity Group found`
- Provider plugin crashes (nil pointer dereference when receiving unexpected 404 responses)
- Complete inability to run `terraform plan` or `terraform apply` on any workspace

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
| 2026-02-27 | PR [#2799](https://github.com/hashicorp/terraform-provider-vault/pull/2799) opened — partial fix (insufficient) |
| 2026-06-01 | We verified a complete fix locally (see "The correct fix" above) |

---

## Request

1. **Do not merge PR #2799 as-is** — it is incomplete and will not fully resolve the issue for users who also have `client.SetNamespace` causing prepending via cloned clients.
2. **Apply the complete fix** as described in "The correct fix" section above — resetting `namespace = ""` when `set_namespace_from_token = false` prevents both the ResourceData and client namespace leakage.
3. **No state migration is needed** — the fix is runtime-only and works with existing state from any previous provider version.
4. **Prioritize this fix** — it has blocked upgrades past v5.1.0 for multiple customers for 10+ months.

---

## References

- Issue [#1903](https://github.com/hashicorp/terraform-provider-vault/issues/1903) — Original namespace duplication report
- Issue [#2331](https://github.com/hashicorp/terraform-provider-vault/issues/2331) — Related upgrade blocker
- PR [#2540](https://github.com/hashicorp/terraform-provider-vault/pull/2540) — The change that introduced this regression
- PR [#2799](https://github.com/hashicorp/terraform-provider-vault/pull/2799) — The pending partial fix (insufficient)
- [Vault provider docs: set_namespace_from_token](https://registry.terraform.io/providers/hashicorp/vault/latest/docs#set_namespace_from_token)
