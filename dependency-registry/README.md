# WSO2 Dependency Registry

This directory serves as the **central registry for all approved third-party dependencies** across WSO2. By enforcing a centralized registry, we ensure **supply chain security, prevent framework bloat, and maintain architectural consistency**.

WSO2 product repositories enforce these rules by pulling this registry during their **automated CI/CD checks**. Any pull request attempting to merge an unapproved dependency or version will be **[automatically blocked at the build level](#downstream-cicd-enforcement)**.

## How to Add or Update a Dependency

1. Open the `.yaml` file corresponding to the dependency's language ecosystem (e.g., `java.yaml`, `javascript.yaml`).
2. Add or update the dependency following the format. Refer to the [Configuration Examples](#configuration-examples) section below for guidance.
3. Reference to the [Supported Version Notations](#supported-version-notations) section to ensure the correct versioning format is used.
4. Open a Pull Request and complete the mandatory **Dependency Request PR Template**. The PR description must include:
   - A clear explanation of the dependency's purpose and core functionality.
   - A technical justification for why this dependency is necessary and cannot be replaced by existing solutions.
   - Confirmation that the dependency has been evaluated for active maintenance, license compliance, and security posture.

## Configuration Examples

Each registry file must declare its `language` at the top, followed by the `dependencies` array. Use `"*"` in `allowed_scopes` to approve a dependency globally, or restrict it to specific products (e.g., `identity-server`, `apim`).

### Go (`go.yaml`)
Requires Go module path.

```yaml
# yaml-language-server: $schema=./schema.yaml
language: go
dependencies:
  - module: github.com/google/uuid
    versions:
      - version: ">=v1.6.0"
        allowed_scopes:
          - "*"
```
### Java (`java.yaml`)
Requires Maven `group` and `artifact`.

```yaml
# yaml-language-server: $schema=./schema.yaml
language: java
dependencies:
  - group: com.google.guava
    artifact: guava
    versions:
      - version: "33.2.1-jre"
        allowed_scopes:
          - identity-server
          - apim

  - group: com.fasterxml.jackson.core
    artifact: jackson-databind
    versions:
      - version: "2.*" 
        allowed_scopes:
          - "*"
```

## Supported Version Notations

To maintain cross-language consistency (Java, Go, JS) and avoid ecosystem-specific confusion, please use the following standard Semantic Versioning (SemVer) notations:

### ✅ Valid Notations
* **Wildcard:**
  * `1.2.*` (Allows any patch update, e.g., `1.2.1`, `1.2.9`)
  * `1.*` (Allows any minor/patch update, e.g., `1.3.0`, `1.4.2`)
  * `*` (Allows any version)
* **Minimum Version:**
  * `>=2.17.0` (Allows `2.17.0` and anything newer)
* **Exact Version:**
  * `18.2.0` (Strictly and only this version)

### 🚫 Invalid Notations (Will be blocked by CI)
* **Ecosystem-Specific Operators:** `^1.2.3` or `~1.2.3` 
  * Use `1.2.*` instead.
* **Non-Deterministic Tags:** `latest`, `release`, or `master`
* **Complex Ecosystem Ranges:** `[1.2.3, 2.0.0)` or `1.2.3 - 2.3.4`

## Available Scopes

The `allowed_scopes` field determines which WSO2 products or components are permitted to use a specific dependency version. This prevents heavy, domain-specific libraries from accidentally bloating unrelated lightweight microservices.

* **Global Scope (`"*"`):** **Use with caution!** Approves the dependency for use across *all* WSO2 projects. Reserve this for universal utilities (e.g., `commons-lang3`, `google/uuid`, `axios`).
* **Product/Component Scopes:** Restricts the dependency to specific teams or repositories. As enforced by the schema, all scopes must follow strict `kebab-case` naming (lowercase alphanumeric characters and hyphens).

**Standard Scopes Include:**
* `agent-manager`
* `api-manager`
* `api-platform`
* `choreo`
* `identity-server`
* `integrator-mi`
* `moesif`
* `openchoreo`
* `thunder`

> **Note:** If a specific product or component is not yet represented, use your standard repository prefix in kebab-case (e.g., `api-manager`) when requesting a new scope in your PR.

## Downstream CI/CD Enforcement

This centralized registry is used to actively enforce approved third-party dependency usage across all WSO2 product repositories. Here is exactly what happens when a developer opens a Pull Request in a downstream repository (e.g., `wso2/api-manager` or `wso2/identity-server`):

1. **Detect Changes & Fetch Registry:** The repo's CI pipeline (GitHub Actions) detects any changes to the dependency manifest files (e.g., `pom.xml`, `go.mod`). It then automatically downloads the latest, language-specific registry file (e.g., `go.yaml`) directly from the `main` branch of this `engineering-governance` repository.
2. **Extract Local Dependencies:** The GitHub Action parses the updated manifest files to extract the exact list of newly requested third-party libraries and their resolved versions.
3. **Cross-Reference & Evaluate:** A validation step compares the dependency changes against the approved registry. It verifies that the library exists, the requested version satisfies the approved SemVer range, and the repository's scope is listed in the `allowed_scopes`.
4. **Automated Blocking:** If an unapproved library is detected, or a version falls outside the allowed range, **the product CI build will instantly fail** and indicate that approval is required. 

> **💡 What to do if your product build fails:** If the CI blocks your PR due to a dependency violation, you cannot bypass it. You must first open a PR in **this** governance repository to request approval for the new dependency or version bump. Once your governance PR is merged by an Architect, simply re-run the failed CI job in your product repository.