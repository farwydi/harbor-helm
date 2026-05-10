# Migration: External Redis via existingSecret (ArgoCD-compatible)

## What changed

The chart no longer uses Helm `lookup` to read Redis credentials from an existing
Kubernetes Secret at render time. Instead, when `redis.external.existingSecret` is
set, each component reads its Redis URL directly from the named Secret via
`secretKeyRef` at **pod runtime**.

This means:
- `helm template` renders correctly without a live cluster
- ArgoCD / Flux / any GitOps tool works out of the box
- No more `NOAUTH Authentication required` crashes after ArgoCD sync

## Breaking change

If you previously used `redis.external.existingSecret` with a Secret that had
keys `REDIS_PASSWORD` and optionally `REDIS_USERNAME`, and relied on `helm install`
/ `helm upgrade` (which called `lookup` against the live cluster) -- **you must
update your Secret to the new key contract**.

The old `external.password` / `external.username` values are **ignored** when
`existingSecret` is set.

## Step-by-step migration

### 1. Prepare the Secret

Your Secret (created manually, via SealedSecrets, or ExternalSecrets Operator)
must contain these keys:

| Key | Example value | Required |
|-----|--------------|----------|
| `_REDIS_URL_CORE` | `redis://user:p%40ss@redis.example.com:6379/0?idle_timeout_seconds=30` | yes |
| `_REDIS_URL_JOBSERVICE` | `redis://user:p%40ss@redis.example.com:6379/1` | yes |
| `_REDIS_URL_REG` | `redis://user:p%40ss@redis.example.com:6379/2?idle_timeout_seconds=30` | yes |
| `_REDIS_URL_TRIVY` | `redis://user:p%40ss@redis.example.com:6379/5?idle_timeout_seconds=30` | if `trivy.enabled` |
| `_REDIS_URL_HARBOR` | `redis://user:p%40ss@redis.example.com:6379/6?idle_timeout_seconds=30` | if `harborDatabaseIndex` is set |
| `_REDIS_URL_CACHE_LAYER` | `redis://user:p%40ss@redis.example.com:6379/7?idle_timeout_seconds=30` | if `cacheLayerDatabaseIndex` is set |
| `REDIS_ADDR` | `redis.example.com:6379` | yes |
| `REDIS_USERNAME` | `default` | optional (ACL mode) |
| `REDIS_PASSWORD` | `p@ss` | yes |

**Important:** passwords inside `_REDIS_URL_*` values must be **URL-encoded**
(e.g. `@` -> `%40`, `:` -> `%3A`, `/` -> `%2F`, `|` -> `%7C`).
`REDIS_PASSWORD` must be the **raw** (unencoded) password.

### 2. ExternalSecrets Operator example

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: harbor-redis
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault          # or aws-secrets-manager, gcp-sm, etc.
    kind: SecretStore
  target:
    name: harbor-redis   # must match redis.external.existingSecret
  data:
    - secretKey: REDIS_PASSWORD
      remoteRef:
        key: harbor/redis
        property: password

    - secretKey: REDIS_ADDR
      remoteRef:
        key: harbor/redis
        property: addr       # e.g. "redis.example.com:6379"

    - secretKey: REDIS_USERNAME
      remoteRef:
        key: harbor/redis
        property: username   # optional

  # URL keys are built from the raw password using ESO templating
  target:
    template:
      engineVersion: v2
      data:
        REDIS_PASSWORD: "{{ .REDIS_PASSWORD }}"
        REDIS_ADDR: "{{ .REDIS_ADDR }}"
        REDIS_USERNAME: "{{ .REDIS_USERNAME }}"
        _REDIS_URL_CORE: "redis://{{ .REDIS_USERNAME | urlquery }}:{{ .REDIS_PASSWORD | urlquery }}@{{ .REDIS_ADDR }}/0?idle_timeout_seconds=30"
        _REDIS_URL_JOBSERVICE: "redis://{{ .REDIS_USERNAME | urlquery }}:{{ .REDIS_PASSWORD | urlquery }}@{{ .REDIS_ADDR }}/1"
        _REDIS_URL_REG: "redis://{{ .REDIS_USERNAME | urlquery }}:{{ .REDIS_PASSWORD | urlquery }}@{{ .REDIS_ADDR }}/2?idle_timeout_seconds=30"
        _REDIS_URL_TRIVY: "redis://{{ .REDIS_USERNAME | urlquery }}:{{ .REDIS_PASSWORD | urlquery }}@{{ .REDIS_ADDR }}/5?idle_timeout_seconds=30"
        # Uncomment if needed:
        # _REDIS_URL_HARBOR: "redis://{{ .REDIS_USERNAME | urlquery }}:{{ .REDIS_PASSWORD | urlquery }}@{{ .REDIS_ADDR }}/6?idle_timeout_seconds=30"
        # _REDIS_URL_CACHE_LAYER: "redis://{{ .REDIS_USERNAME | urlquery }}:{{ .REDIS_PASSWORD | urlquery }}@{{ .REDIS_ADDR }}/7?idle_timeout_seconds=30"
```

### 3. Plain Kubernetes Secret example (for testing)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: harbor-redis
type: Opaque
stringData:
  REDIS_ADDR: "redis.example.com:6379"
  REDIS_USERNAME: ""
  REDIS_PASSWORD: "mysecretpassword"
  _REDIS_URL_CORE: "redis://:mysecretpassword@redis.example.com:6379/0?idle_timeout_seconds=30"
  _REDIS_URL_JOBSERVICE: "redis://:mysecretpassword@redis.example.com:6379/1"
  _REDIS_URL_REG: "redis://:mysecretpassword@redis.example.com:6379/2?idle_timeout_seconds=30"
  _REDIS_URL_TRIVY: "redis://:mysecretpassword@redis.example.com:6379/5?idle_timeout_seconds=30"
```

### 4. Update values.yaml

```yaml
redis:
  type: external
  external:
    addr: "redis.example.com:6379"
    # username and password are IGNORED when existingSecret is set
    existingSecret: "harbor-redis"
```

### 5. Deploy

```bash
# Dry-run first (works correctly now!)
helm template harbor . -f values.yaml > /tmp/rendered.yaml

# Verify the rendered output
grep -c 'secretKeyRef' /tmp/rendered.yaml   # should show multiple entries
grep 'harbor-redis' /tmp/rendered.yaml       # should reference your secret

# Deploy
helm upgrade --install harbor . -f values.yaml
# or: argocd app sync harbor
```

### 6. Verify

```bash
# All pods should be Running/Ready
kubectl get pods -l app=harbor

# Check that jobservice is not crashing with NOAUTH
kubectl logs -l component=jobservice --tail=50

# Test push/pull
docker push harbor.example.com/library/test:latest
docker pull harbor.example.com/library/test:latest
```

## Sentinel support

`sentinelMasterSet` with `existingSecret` is **not yet supported**. The chart
will fail with a clear error message if you try this combination.

For sentinel, either:
- Use sentinel **without** `existingSecret` (password in values.yaml -- legacy path)
- Wait for a future release that adds sentinel URL scheme support
  (`redis+sentinel://...`) in the ExternalSecret

## Password rotation

When the external Secret is updated (e.g. ESO refreshes from Vault), pods will
**not** automatically restart. Use one of:

- [Reloader](https://github.com/stakater/Reloader) -- watches secrets and
  triggers rolling restarts
- `kubectl rollout restart deployment` after rotation
- ArgoCD sync with `argocd.argoproj.io/hook: PostSync` annotation
