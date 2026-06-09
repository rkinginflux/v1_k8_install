# InfluxDB Enterprise v1 — Kubernetes Manifest

Single-file deploy for InfluxDB Enterprise v1.12.3 on Kubernetes.

## What's deployed

| Component | Count | Details |
|-----------|-------|---------|
| Meta Nodes | 3 | v1.12.3-meta, Raft consensus |
| Data Nodes | 2 | v1.12.3-data, TSI index (tsi1) |
| TLS | cert-manager | Self-signed Issuer + Certificates |
| Storage | local-path | 8Gi per PVC |
| Auth | bootstrap Job | admin / changeme, ALL PRIVILEGES |
| License | license-path | Air-gapped ready via Secret |

## Prerequisites

1. **local-path-provisioner**
   ```
   kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml
   ```

2. **cert-manager**
   ```
   kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
   ```

3. Control-plane taint removed (single-node clusters):
   ```
   kubectl taint nodes <node> node-role.kubernetes.io/control-plane:NoSchedule-
   ```

4. **License JSON in a Kubernetes Secret** (see below)

## License: Air-Gapped vs Connected

### Air-Gapped (license-path — default in this manifest)

For clusters that cannot reach `portal.influxdata.com`. Contact `sales@influxdb.com` to obtain the permanent JSON license file.

**Step 1:** Create the Secret with your license JSON before deploying:
```bash
export LICENSE_JSON='{"signature":"eyJ0eX...","license":{"key":"684a9932-...","state":"active",...}}'
kubectl create ns influxdb-enterprise
kubectl -n influxdb-enterprise create secret generic influxdb-license \
  --from-literal=license.json="${LICENSE_JSON}"
```

**Step 2:** Deploy the manifest (ConfigMaps already use `license-path`):
```bash
kubectl apply -f influxdb-enterprise-v1-deploy.yaml
```

The StatefulSets mount the Secret at `/var/lib/influxdb/license.json` using `subPath` (prevents overwriting the data directory). Both meta and data ConfigMaps reference `license-path = "/var/lib/influxdb/license.json"`.

### Connected (license-key)

For clusters that can reach `portal.influxdata.com:80/443`. The meta node phones home, gets a temporary JSON license, and caches it locally.

Switch both ConfigMaps in the manifest back to `license-key`:
```toml
[enterprise]
  license-key = "YOUR-KEY-HERE"
  # license-path = "/var/lib/influxdb/license.json"  # mutually exclusive
```

You can also remove the `license` volume and volumeMount from both StatefulSets since they're not needed.

### How it works

| Mechanism | Where | Requirement |
|-----------|-------|-------------|
| `license-key` | ConfigMap | Meta node calls portal.influxdata.com on port 80/443 |
| `license-path` | ConfigMap + Secret | License JSON file present on every node at the specified path |

**Important:** `license-key` and `license-path` are mutually exclusive — one must be empty. Use the same method on all nodes. Restart all nodes after changing license configuration.

## Deploy

```bash
kubectl apply -f influxdb-enterprise-v1-deploy.yaml
```

## Post-deploy

**Check cluster health:**
```bash
kubectl -n influxdb-enterprise exec influxdb-enterprise-meta-0 -- influxd-ctl -bind-tls -k show
```

**Connect with CLI:**
```bash
kubectl -n influxdb-enterprise exec influxdb-enterprise-data-0 -- influx -ssl -unsafeSsl -username admin -password changeme
```

**Verify license is mounted:**
```bash
kubectl -n influxdb-enterprise exec influxdb-enterprise-data-0 -- cat /var/lib/influxdb/license.json
kubectl -n influxdb-enterprise exec influxdb-enterprise-meta-0 -- cat /var/lib/influxdb/license.json
```

## Configuration

- Namespace: `influxdb-enterprise`
- Auth: `[http] auth-enabled = true` with bootstrap job creating admin user
- Flux: enabled
- Index: TSI (`index-version = "tsi1"`) for high-cardinality workloads
- TLS: HTTPS everywhere, self-signed certs via cert-manager

## Architecture

```
  3 Meta Nodes       2 Data Nodes
  ┌─────────┐        ┌─────────┐
  │ meta-0  │ leader  │ data-0  │
  │ meta-1  │        │ data-1  │
  │ meta-2  │        └─────────┘
  └─────────┘
```

## Troubleshooting

### All pods stuck in ContainerCreating with "secret not found"

**Symptoms:**
```
Warning  FailedMount  ... MountVolume.SetUp failed for volume "license" :
                      secret "influxdb-license" not found
```

Every pod (meta + data) is deadlocked and never reaches Running. The bootstrap Job also fails with `no such host` because data pods aren't up yet.

**Cause:** You applied the manifest before creating the `influxdb-license` Secret. The StatefulSets reference it in their volume definitions, and K8s blocks pod startup until the volume can be mounted.

**Fix:**
```bash
# 1. Create the missing Secret
export LICENSE_JSON='{"signature":"eyJ0...","license":{...}}'
kubectl -n influxdb-enterprise create secret generic influxdb-license \
  --from-literal=license.json="${LICENSE_JSON}"

# 2. Force-delete the stuck pods (K8s doesn't retry dead volume mounts)
kubectl -n influxdb-enterprise delete pod -l influxdb.influxdata.com/component=meta --force --grace-period=0
kubectl -n influxdb-enterprise delete pod -l influxdb.influxdata.com/component=data --force --grace-period=0

# 3. Wait for pods to come up, then recreate the admin user
kubectl -n influxdb-enterprise exec influxdb-enterprise-data-0 -- \
  influx -ssl -unsafeSsl -execute 'CREATE USER admin WITH PASSWORD '\''changeme'\'' WITH ALL PRIVILEGES'
```

### Bootstrap Job keeps failing

The bootstrap Job's `auth` init container needs the data service to be reachable. If data pods aren't Running yet, the Job can't connect.

```bash
# Check what's blocking the data pods
kubectl -n influxdb-enterprise describe pod -l influxdb.influxdata.com/component=data | grep -A2 FailedMount

# Delete and recreate the Job once data pods are healthy
kubectl -n influxdb-enterprise delete job influxdb-enterprise-bootstrap --ignore-not-found
```

## Updating the License

When your license expires, update the Secret and restart:

```bash
kubectl -n influxdb-enterprise create secret generic influxdb-license \
  --from-literal=license.json="${NEW_LICENSE_JSON}" \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n influxdb-enterprise rollout restart sts influxdb-enterprise-meta
kubectl -n influxdb-enterprise rollout restart sts influxdb-enterprise-data
```
