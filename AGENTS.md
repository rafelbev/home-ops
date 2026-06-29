# AGENTS.md - Guide for Adding Applications to This Repository

This document outlines the correct patterns and conventions for adding new applications to this Kubernetes GitOps repository using Flux CD and the bjw-s app-template Helm chart.

## Architecture Overview

This repository uses:
- **Flux CD** for GitOps automation (HelmRelease, OCIRepository, Kustomization)
- **bjw-s app-template** chart for deploying applications
- **SOPS** for secret encryption (age-based encryption)
- **OCIRepository** for chart sources (no traditional HelmRepository)

## Directory Structure

Each application follows this structure:

```
kubernetes/apps/<category>/<app-name>/
├── ks.yaml                    # Flux Kustomization (points to app path)
├── kustomization.yaml         # Kustomize config (lists resources)
├── namespace.yaml             # Namespace definition (if needed)
└── app/
    ├── helmrelease.yaml       # Flux HelmRelease
    ├── ocirepository.yaml     # OCIRepository for app-template
    ├── pvc.yaml               # PersistentVolumeClaims (if needed)
    └── kustomization.yaml     # App-level resources list
```

Top-level apps organization:
```
kubernetes/apps/
├── cert-manager/
├── cluster-secrets/           # Only for flux-system apps that need secrets
├── default/
├── flux-system/
├── kube-system/
├── media/                     # NEW: Create this for media apps
│   ├── kustomization.yaml
│   ├── radarr/
│   │   ├── ks.yaml
│   │   ├── kustomization.yaml
│   │   ├── namespace.yaml
│   │   └── app/
│   │       ├── helmrelease.yaml
│   │       ├── ocirepository.yaml
│   │       ├── pvc.yaml
│   │       └── kustomization.yaml
├── network/
└── ...
```

## Step-by-Step App Addition

### 1. Create Directory Structure

```bash
# Create the app directory structure
mkdir -p kubernetes/apps/<category>/<app-name>/app
```

### 2. Create Namespace (if needed)

```yaml
# kubernetes/apps/<category>/<app-name>/namespace.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: <app-name>
  annotations:
    kustomize.toolkit.fluxcd.io/prune: disabled
```

### 3. Create Flux Kustomization

```yaml
# kubernetes/apps/<category>/<app-name>/ks.yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: <app-name>
  namespace: flux-system  # REQUIRED for flux-local validation
spec:
  interval: 1h
  path: ./kubernetes/apps/<category>/<app-name>
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
    namespace: flux-system
  targetNamespace: <app-name>
  wait: false
```

**Key points:**
- `metadata.namespace` must be `flux-system`
- `path` must point to the app directory (not the app/ subdirectory)
- `targetNamespace` matches the app name

### 4. Create App-Level Kustomization

```yaml
# kubernetes/apps/<category>/<app-name>/kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - namespace.yaml
  - app/
```

### 5. Create Media/Category Kustomization

```yaml
# kubernetes/apps/<category>/kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

components:
  - ../../components/sops

resources:
  - ./<app-name>/ks.yaml
```

### 6. Create OCIRepository for app-template

```yaml
# kubernetes/apps/<category>/<app-name>/app/ocirepository.yaml
---
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: app-template
  namespace: <app-name>
spec:
  interval: 15m
  layerSelector:
    mediaType: application/vnd.cncf.helm.chart.content.v1.tar+gzip
    operation: copy
  ref:
    tag: 4.4.0
  url: oci://ghcr.io/bjw-s-labs/helm/app-template
```

**Key points:**
- API version: `source.toolkit.fluxcd.io/v1beta2`
- Name: `app-template` (not the app name)
- Include `namespace: <app-name>` in metadata
- Use `layerSelector` for OCI layer filtering

### 7. Create PVCs (if needed)

```yaml
# kubernetes/apps/<category>/<app-name>/app/pvc.yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: <app-name>-config
  namespace: <app-name>
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-path
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: media-pvc-v2
  namespace: <app-name>
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 4Ti
  storageClassName: truenas-nfs-static
```

**Key points:**
- Must include `namespace: <app-name>` in metadata
- Use appropriate storageClassName (local-path, truenas-nfs-static, etc.)
- Use correct accessMode (ReadWriteOnce for config, ReadWriteMany for media)

### 8. Create HelmRelease

```yaml
# kubernetes/apps/<category>/<app-name>/app/helmrelease.yaml
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: <app-name>
  namespace: <app-name>
spec:
  chartRef:
    kind: OCIRepository
    name: app-template  # Must match OCIRepository name
  interval: 1h
  values:
    controllers:
      main:  # Use 'main', not '<app-name>'
        annotations:
          reloader.stakater.com/auto: "true"
        containers:
          app:
            image:
              repository: lscr.io/linuxserver/<app-name>
              tag: <version>
            env:
              TZ: "Europe/Malta"  # Always quote timezone values
              PUID: "0"
              PGID: "0"
            probes:
              liveness:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /ping
                    port: <port>
                  initialDelaySeconds: 0
                  periodSeconds: 10
                  timeoutSeconds: 1
                  failureThreshold: 3
              readiness:
                enabled: true
                custom: true
                spec:
                  httpGet:
                    path: /ping
                    port: <port>
                  initialDelaySeconds: 0
                  periodSeconds: 10
                  timeoutSeconds: 1
                  failureThreshold: 3
            securityContext:
              allowPrivilegeEscalation: false
              readOnlyRootFilesystem: true
              capabilities: { drop: ["ALL"] }
            resources:
              requests:
                cpu: 10m
              limits:
                memory: 512Mi
    defaultPodOptions:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        # NO fsGroup or fsGroupChangePolicy - these are invalid
    service:
      app:
        ports:
          http:
            port: <port>
    persistence:
      config:
        type: persistentVolumeClaim  # NOT 'pvc'
        existingClaim: <app-name>-config  # NOT 'name'
        globalMounts:
          - path: /config
      media:
        type: persistentVolumeClaim
        existingClaim: media-pvc-v2
        globalMounts:
          - path: /media
```

**Critical HelmRelease schema rules:**
- API version: `helm.toolkit.fluxcd.io/v2` (not v2beta1)
- Use `chartRef` with `kind: OCIRepository` and `name: app-template`
- Use `controllers.main`, not `controllers.<app-name>`
- Use `type: persistentVolumeClaim` (not `pvc`)
- Use `existingClaim: <name>` (not `name`)
- Remove `enabled: true` from persistence sections
- Quote timezone values: `TZ: "Europe/Malta"`
- NO `fsGroup` or `fsGroupChangePolicy` in defaultPodOptions.securityContext
- NO `strategy: RollingUpdate` in defaultPodOptions
- NO `nodeSelector` in defaultPodOptions

### 9. Create App-Level Kustomization

```yaml
# kubernetes/apps/<category>/<app-name>/app/kustomization.yaml
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - pvc.yaml
  - helmrelease.yaml
  - ocirepository.yaml
```

## Common Pitfalls and Fixes

### 1. Duplicate Secret Registration
**Problem:** `may not add resource with an already registered id: Secret.v1.[noGrp]/cluster-secrets.[noNs]`

**Cause:** The sops component includes `cluster-secrets.sops.yaml` as a resource, and multiple apps include the sops component. This causes the Secret to be registered multiple times.

**Fix:**
- Remove `resources: - ./cluster-secrets.sops.yaml` from `kubernetes/components/sops/kustomization.yaml`
- Add `../../components/sops/cluster-secrets.sops.yaml` to the resources list in the flux-system kustomization
- Keep the component empty or with patches only

### 2. Persistence Schema Validation Failure
**Problem:** `at '/persistence/config': validation failed - missing properties 'accessMode', 'size'`

**Cause:** Using `type: pvc` with `name` field instead of `type: persistentVolumeClaim` with `existingClaim`.

**Fix:**
```yaml
persistence:
  config:
    type: persistentVolumeClaim  # NOT 'pvc'
    existingClaim: <pvc-name>    # NOT 'name'
    globalMounts:
      - path: /config
```

### 3. Empty Kustomization.yaml Error
**Problem:** `kustomization.yaml is empty`

**Cause:** Removing resources from the sops component without adding patches.

**Fix:** Add patches to the component kustomization.yaml:
```yaml
patches:
  - patch: |-
      - op: add
        path: /metadata/annotations/sops.dev~1encrypted
        value: "true"
    target:
      kind: Secret
```

### 4. Missing Namespace in Metadata
**Problem:** `Invalid <class 'flux_local.manifest.Kustomization'> missing metadata.namespace`

**Cause:** ks.yaml metadata missing `namespace: flux-system`.

**Fix:** Always include `namespace: flux-system` in ks.yaml metadata.

### 5. Wrong Controller Name
**Problem:** Chart template fails

**Cause:** Using `controllers.<app-name>` instead of `controllers.main`.

**Fix:** Always use `controllers.main` in app-template values.

### 6. YAML Anchor Issues
**Problem:** Helm template fails with anchor references

**Cause:** Using YAML anchors (`&`) and references (`*`) in helmrelease.yaml.

**Fix:** Use explicit values instead of anchors, especially for probes and ports.

## Validating Before Committing

### 1. Check YAML Syntax
```bash
python3 -c "import yaml; yaml.safe_load(open('kubernetes/apps/<category>/<app-name>/app/helmrelease.yaml'))"
```

### 2. Check Multiple YAML Documents
```bash
python3 -c "
import yaml
with open('kubernetes/apps/<category>/<app-name>/app/pvc.yaml') as f:
    docs = list(yaml.safe_load_all(f))
    print(f'Found {len(docs)} documents')
"
```

### 3. Verify Flux Local Checks
```bash
# Run flux-local test (requires Docker)
flux-local test --path kubernetes/flux/cluster -v

# Run flux-local diff
flux-local diff --path kubernetes/flux/cluster
```

### 4. Check PR Status
```bash
gh pr checks 1
```

## Commit Convention

Use conventional commits:
- `feat(<app>): migrate <App> to GitOps structure with Flux and bjw-s app-template`
- `fix(sops): <description>`
- `fix(<app>): <description>`

## Files That Should NOT Be Committed

- **Encrypted secrets directly in the app directory** - Use the sops component
- **Duplicate cluster-secrets** - Only include in flux-system kustomization
- **YAML anchors** - Use explicit values to avoid template issues

## Quick Checklist

Before pushing, verify:
- [ ] ks.yaml has `namespace: flux-system` in metadata
- [ ] ks.yaml path points to app directory (not app/)
- [ ] OCIRepository uses `source.toolkit.fluxcd.io/v1beta2`
- [ ] OCIRepository is named `app-template`
- [ ] HelmRelease uses `helm.toolkit.fluxcd.io/v2`
- [ ] HelmRelease chartRef points to `OCIRepository` named `app-template`
- [ ] Controllers section uses `main` not app name
- [ ] Persistence uses `type: persistentVolumeClaim` and `existingClaim`
- [ ] No `enabled: true` in persistence sections
- [ ] No `fsGroup` or `fsGroupChangePolicy` in defaultPodOptions
- [ ] No YAML anchors in helmrelease.yaml
- [ ] TZ value is quoted: `TZ: "Europe/Malta"`
- [ ] PVCs have namespace in metadata
- [ ] No duplicate cluster-secrets registration

## Example: Adding a New App

1. Create directory structure
2. Create namespace.yaml
3. Create ks.yaml with proper metadata
4. Create kustomization.yaml
5. Create app/kustomization.yaml
6. Create app/ocirepository.yaml
7. Create app/pvc.yaml (if needed)
8. Create app/helmrelease.yaml
9. Update parent kustomization.yaml
10. Validate with `gh pr checks 1`
11. Commit with conventional commit message
12. Push and verify all checks pass

## References

- [bjw-s app-template documentation](https://github.com/bjw-s/helm-charts)
- [Flux CD HelmRelease API](https://fluxcd.io/docs/components/helm/)
- [Flux CD OCIRepository](https://fluxcd.io/docs/source-controller/ocirepository/)
- [Kustomize documentation](https://kustomize.dev/)
