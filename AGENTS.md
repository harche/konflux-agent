# Konflux Agent Instructions

## Required Repositories

As a Konflux agent, you must work with two essential repositories:

### 1. Konflux Documentation
- **Repository**: https://github.com/konflux-ci/docs/
- **Purpose**: Official Konflux documentation and reference materials
- **Action**: Clone this repository locally if it doesn't already exist

### 2. Konflux GitOps Configuration
- **Repository**: https://gitlab.cee.redhat.com/releng/konflux-release-data
- **Purpose**: GitOps repository that drives all Konflux configuration in OpenShift
- **Action**: Clone this repository locally if it doesn't already exist

### 3. Konflux Tekton Configuration
- **Location**: Hosted in individual product repositories within `.tekton` folders
- **Purpose**: Product-specific Tekton pipeline configurations for Konflux
- **Examples**:
  - https://github.com/openshift/instaslice-operator (see `.tekton` folder)
  - https://github.com/openshift/kueue-operator (see `.tekton` folder)
- **Action**: Clone example repositories to understand Tekton configuration patterns

### 4. OLM Operator Konflux Sample
- **Repository**: https://github.com/konflux-ci/olm-operator-konflux-sample
- **Purpose**: Sample repository demonstrating how to build OLM Operators in Konflux
- **Contains**: Complete examples of Konflux configuration for operator development
- **Action**: Clone this repository to understand operator-specific Konflux patterns and configurations

### 5. File-Based Catalog (FBC) Repository
- **Repository**: https://github.com/openshift/instaslice-fbc
- **Purpose**: Builds File-Based Catalog (FBC) container images for OLM operator distribution across multiple OpenShift versions
- **Contains**:
  - FBC templates and generated catalogs for each OCP version (v4.16, v4.17, v4.18, v4.19)
  - Containerfiles for building catalog images
  - `.tekton` pipelines for automated FBC image builds in Konflux
- **Key Concept**: FBC is the modern way to distribute OLM operators - it replaces the older SQLite database format with a declarative file-based approach
- **Workflow**:
  1. Update `catalog-template.yaml` with new operator bundle image references
  2. Run `generate-fbc.sh` to regenerate catalog JSON using `opm` tool
  3. Konflux builds catalog container images via Tekton pipelines (separate pipeline per OCP version)
- **Action**: Clone this repository to understand how to build and maintain file-based catalogs for operators

## Critical Configuration Guidelines

### ⚠️ Configuration Changes Policy
- **IMPORTANT**: Direct cluster configuration changes are **NOT SUPPORTED**
- **IMPORTANT**: We only care about 4.19 release, don't bother with the config related to other releases.
- **ONLY METHOD**: All changes must be made through the GitOps repository (konflux-release-data)
- **Documentation Note**: While official docs may show direct cluster commands, ignore those sections in our setup

### Working with GitOps (Config as Code)
1. **Always** refer to the GitOps sections of the documentation
2. **Always** look for existing patterns in the GitOps repository for guidance
3. **Never** apply configurations directly to the cluster
4. **Follow** the established patterns and conventions found in the GitOps repo

## Operator Release Workflow

### Publishing Operator Updates to OperatorHub

To make the latest operator code available in OperatorHub, follow this 4-step workflow:

#### Step 1: Operator Bundle Build (Automatic)
- When you commit to the `next` branch in the operator repository (e.g., `openshift/instaslice-operator`)
- Konflux automatically builds the operator bundle via `.tekton/instaslice-operator-bundle-next-push.yaml`
- Output image: `quay.io/redhat-user-workloads/dynamicacceleratorsl-tenant/instaslice-operator-bundle-next:{{revision}}`
- The bundle contains all operator manifests (CRDs, CSV, RBAC, etc.)

#### Step 2: Update FBC Catalog Template (Manual)
Get the latest bundle image SHA using one of these methods:

**Option A - Konflux UI** (Easiest):
1. Navigate to tenant: `dynamicacceleratorsl-tenant`
2. Application: `dynamicacceleratorslicer-next`
3. Component: `instaslice-operator-bundle-next`
4. Copy the image SHA from the latest successful build

**Option B - Quay.io**:
- Visit: https://quay.io/repository/redhat-user-workloads/dynamicacceleratorsl-tenant/instaslice-operator-bundle-next?tab=tags
- Copy the SHA of the latest tag

**Option C - skopeo CLI**:
```bash
skopeo inspect docker://quay.io/redhat-user-workloads/dynamicacceleratorsl-tenant/instaslice-operator-bundle-next:COMMIT_SHA | jq -r '.Digest'
```

**Update the FBC repository**:
1. Edit catalog templates for each OCP version (e.g., `instaslice-fbc/v4.18/catalog-template.yaml`):
   ```yaml
   Schema: olm.semver
   GenerateMajorChannels: true
   GenerateMinorChannels: false
   Stable:
     Bundles:
     - Image: quay.io/redhat-user-workloads/dynamicacceleratorsl-tenant/instaslice-operator-bundle-next@sha256:NEW_SHA_HERE
   ```
2. Regenerate catalogs:
   ```bash
   cd instaslice-fbc
   ./generate-fbc.sh
   ```
3. Commit and push to `main` branch
4. Konflux builds new FBC catalog images via `.tekton` pipelines (one per OCP version)

#### Step 3: Release to Production (Automatic via GitOps)
- The GitOps repository contains ReleasePlan resources that automatically publish FBC images
- Location: `konflux-release-data/tenants-config/cluster/stone-prd-rh01/tenants/dynamicacceleratorsl-tenant/`
- Example: `instaslice-fbc-418-prod-release-plan.yaml`
- These ReleasePlans automatically push FBC images to `registry.redhat.io` when builds complete
- No manual GitOps changes needed (already configured)

#### Step 4: Verify in OperatorHub
- Once released, the updated catalog appears in OpenShift's OperatorHub
- The catalog is consumed via the configured CatalogSource in the cluster

**Summary**: You only need to manually update the FBC catalog templates with the new bundle SHA and regenerate the catalogs. Everything else is automated via Konflux pipelines and GitOps ReleasePlans.

## Best Practices
- Study existing configurations in the GitOps repo before making changes
- Follow established naming conventions and structure patterns
- Test changes in development environments when possible
- Document any new patterns or deviations from existing conventions
- **For operator releases**: Always verify the bundle SHA before updating FBC templates
- **For FBC updates**: Update all supported OCP versions (v4.16, v4.17, v4.18, v4.19) consistently 
