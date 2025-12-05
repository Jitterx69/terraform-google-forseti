# Open Source Contribution Report: Modernizing Terraform Google Forseti

**Date:** December 5, 2025  
**Repository:** [terraform-google-forseti](https://github.com/forseti-security/terraform-google-forseti) (Forked)  
**Contributor:** Mohit Ranjan

## 1. Executive Summary

The primary objective of this contribution was to revitalize the archived `terraform-google-forseti` repository, making it compatible with modern development environments, specifically Apple Silicon (ARM64) architectures. The repository relied on outdated Terraform providers (most notably the `template` provider) that lacked pre-compiled binaries for `darwin_arm64`, causing `terraform init` failures.

We successfully modernized the codebase by upgrading provider dependencies, refactoring deprecated code patterns (replacing `template_file` data sources with the `templatefile()` function), and fixing deprecated resource arguments. The repository now successfully initializes and validates with modern Terraform CLI versions.

## 2. Problem Statement

Upon attempting to set up the repository on a macOS ARM64 machine, the following critical issues were identified:

1.  **Incompatible Provider Versions:** The `versions.tf` files locked providers to old versions (e.g., `template ~> 2.2`, `google ~> 3.52`) which do not support the `darwin_arm64` platform.
2.  **Deprecated `template` Provider:** The `hashicorp/template` provider is deprecated and archived. It was extensively used via `data "template_file"` for rendering configuration and scripts.
3.  **Deprecated GCS Arguments:** Google Cloud Storage bucket resources used `bucket_policy_only`, which has been deprecated in favor of `uniform_bucket_level_access`.
4.  **Broken Initialization:** `terraform init` failed completely, preventing any usage or testing of the module.

## 3. Technical Implementation Details

### 3.1. Provider Dependency Upgrades
We performed a comprehensive update of `versions.tf` files across the root, modules, and examples to support wider and newer version ranges compatible with ARM64.

**Changes:**
- **Removed:** `template` provider dependency.
- **Updated:**
    - `google`: `~> 3.52` → `>= 3.52, < 5.0`
    - `google-beta`: `~> 3.52` → `>= 3.52, < 5.0`
    - `local`: `~> 1.4` → `~> 2.0`
    - `null`: `~> 2.1` → `~> 3.0`
    - `random`: `~> 2.2` → `~> 3.0`
    - `tls`: `~> 2.1` → `~> 3.0`
    - `kubernetes`: `~> 1.13` → `~> 2.0`

### 3.2. Refactoring `template_file` to `templatefile()`
The most significant code change involved removing the `data "template_file"` data source. We refactored the code to use Terraform's built-in `templatefile()` function, which renders templates directly without requiring a separate provider.

**Modules Affected:**
- `modules/rules`
- `modules/client`
- `modules/server`
- `modules/server_config`
- `modules/client_config`
- `modules/real_time_enforcer`
- `test/fixtures/install_simple`

**Example Transformation:**

*Before:*
```hcl
data "template_file" "forseti_client_config" {
  template = local.client_conf
  vars = {
    forseti_server_ip = var.server_address
  }
}

resource "google_storage_bucket_object" "config" {
  content = data.template_file.forseti_client_config.rendered
  ...
}
```

*After:*
```hcl
resource "google_storage_bucket_object" "config" {
  content = templatefile("${path.module}/templates/configs/forseti_conf_client.yaml.tpl", {
    forseti_server_ip = var.server_address
  })
  ...
}
```

### 3.3. Fixing Deprecated Arguments
We identified and fixed the usage of the deprecated `bucket_policy_only` argument in Google Cloud Storage resources.

**Modules Affected:**
- `modules/server_gcs`
- `modules/client_gcs`
- `modules/real_time_enforcer`

**Change:**
- Replaced `bucket_policy_only = true` with `uniform_bucket_level_access = true`.

### 3.4. Output Adjustments
Since `data "template_file"` blocks were removed, output variables referencing them (e.g., `data.template_file.name.rendered`) were broken. These were updated to reference the `content` attribute of the generated `google_storage_bucket_object` resources.

## 4. Verification and Testing

To ensure the integrity of the changes, we performed the following validation steps:

1.  **Initialization:** Ran `terraform init` in `examples/install_simple`.
    *   **Result:** Success. All providers (Google v4.x, Local v2.x, etc.) were downloaded and installed correctly for `darwin_arm64`.
2.  **Validation:** Ran `terraform validate` in `examples/install_simple`.
    *   **Result:** Success. The configuration is syntactically valid and internally consistent.

## 5. Impact

*   **Cross-Platform Compatibility:** The repository can now be developed and deployed from Apple Silicon Macs.
*   **Modernization:** Removed reliance on archived/deprecated providers, ensuring future compatibility with newer Terraform CLI versions.
*   **Code Quality:** Simplified the codebase by removing unnecessary data sources and using built-in functions.

## 6. Future Roadmap

With the infrastructure code now functional, the following tasks are unlocked:
1.  **Implement CIS GCP 7.16:** Develop Config Validator (OPA) policies to check for "GKE Private Google Access" by correlating Cluster and Subnetwork resources.
2.  **Integration Testing:** Run `terraform apply` in a sandbox environment to verify the end-to-end deployment of Forseti Security.
