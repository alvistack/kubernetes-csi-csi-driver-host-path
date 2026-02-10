# Project Instructions: CSI Driver Host Path Helm Chart Packaging

## Overview
This repository contains the Kubernetes CSI Driver Host Path implementation. To support Helm-based deployments and automated chart releases (via `gh-pages` and `helm/chart-releaser-action`), static deployment manifests under `deploy/` are converted into Helm Charts under `charts/csi-hostpathplugin/` using the script at `deploy/util/chart-releaser.sh`.

---

## Workflow for Stable Tag Releases & Chart Packaging

Before each stable tag release or when updating deployment manifests/versions, maintainers must perform the following steps:

1. **Verify & Update Static Deployment Manifests:**
   - Ensure image tags (e.g. `hostpathplugin`, sidecars) across static manifests under `deploy/*/hostpath/*.yaml` are synchronized to the target release version (e.g., `v1.19.0`).

2. **Verify & Update Chart Releaser Script:**
   - Since `chart-releaser.sh` is derived from `deploy-hostpath.sh`, verify and sync any corresponding deployment script updates or changes from `deploy-hostpath.sh` into `chart-releaser.sh`.

3. **Run the Chart Releaser Script:**
   - Execute the chart conversion script from any deployment directory (symlinked to `deploy/util/chart-releaser.sh`):
     ```bash
     ./deploy/kubernetes-latest/chart-releaser.sh
     ```
   - This script automatically:
     - Downloads and kustomizes RBAC rules.
     - Copies deployment manifests into `charts/csi-hostpathplugin/templates/` (excluding testing and snapshotclass manifests).
     - Automatically synchronizes `charts/csi-hostpathplugin/values.yaml` with image repositories, tags, replica counts, and hostPath settings from the manifests.
     - Automatically updates `charts/csi-hostpathplugin/Chart.yaml` (`version` and `appVersion`) with the current hostpath plugin version tag.
     - Converts static manifests into Helm template expressions (`{{ .Values... }}` and `{{ .Release.Namespace }}`).

4. **Verify Chart Integrity:**
   - Lint the Helm chart:
     ```bash
     helm lint charts/csi-hostpathplugin
     ```
   - Test template rendering:
     ```bash
     helm template csi-hostpathplugin charts/csi-hostpathplugin
     ```

5. **Commit Chart Updates:**
   - Force-add and commit the generated/updated chart artifacts:
     ```bash
     git add --all --force charts/* && git commit -m "Update Helm Chart for v1.19.0"
     ```

---

## Architectural & Script Design Principles (`chart-releaser.sh`)
- **No Deployment Execution:** Unlike `deploy-hostpath.sh`, `chart-releaser.sh` does not run `kubectl apply` or post-deploy health checks; instead, it outputs generated manifests directly as template files.
- **Exclusion of Testing/Non-Chart Resources:** Test pods/services (`csi-hostpath-testing.yaml`) and snapshot classes (`csi-hostpath-snapshotclass.yaml`, handled separately by Ansible roles) are excluded from the Helm chart export.
- **Automation of Values & Metadata:** `yq` is utilized to parse, template, and sync deployment parameters directly into `values.yaml` and `Chart.yaml`.
