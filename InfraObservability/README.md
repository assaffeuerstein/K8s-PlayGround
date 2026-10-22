# InfraObservability

Alloy scraping on-prem infrastructure and remote-writing to Mimir. Currently: Pure Storage FlashArray.
Planned: Palo Alto, F5 BIG-IP, Juniper (add one `declare` module + instances per vendor in `alloy-values.yaml`).

```
on-prem node (hybrid EKS)                 cloud side
 Alloy (StatefulSet, WAL on PVC) --remote_write--> Mimir gateway --> S3
   |  scrape (https + Bearer token)
   v
 FlashArrays
```

## Files
- `alloy-values.yaml` – Helm values for the `grafana/alloy` chart + the Alloy config.
- `dashboards/` – Grafana dashboards as ConfigMaps (Grafana sidecar loads them into folder "Infrastructure").
- `../Apps/infra-alloy.yaml`, `../Apps/infra-dashboards.yaml` – Argo CD Applications.

## 0. Probe the array first (assumption check)
The Alloy config assumes the array serves OpenMetrics at `/metrics/<array|volumes|hosts|pods|directories>`
with `Authorization: Bearer <api token>` (Purity//FA 6.6.11+ native exporter). Confirm before deploying:

    curl -sk -H "Authorization: Bearer $TOKEN" https://<array>/metrics/array | head

Expect `purefa_*` lines. If the path or auth differs, adjust `__metrics_path__` / `authorization` in `alloy-values.yaml`.
Use a dedicated **read-only** array user for the token.

## 1. Edit the array entry
In `alloy-values.yaml` replace `name` and `address` in the `pure_flasharray "fa1"` block. One block per array,
each with its own token env var (`PURE_TOKEN_<NAME>`).

## 2. Create the token secret (never commit it)
    kubectl create namespace infra-observability
    kubectl -n infra-observability create secret generic pure-api-tokens --from-literal=PURE_TOKEN_FA1=<token>
In prod use External Secrets / your secret manager to create the same Secret.

## 3. Deploy
Push to git, then `kubectl apply -f Apps/infra-alloy.yaml -f Apps/infra-dashboards.yaml`.
If the secret was created after the pod started: `kubectl -n infra-observability rollout restart statefulset/alloy`.

## 4. Verify
- Alloy UI: `kubectl -n infra-observability port-forward svc/alloy 12345` -> http://localhost:12345 (all components healthy).
- Grafana Explore (Mimir): `up{job="pure-flasharray"}` should be 1 for each category; `purefa_info` lists the array.
- Dashboard: Grafana > Dashboards > Infrastructure > "Pure Storage FlashArray - Overview".

## Prod notes
- Pin Alloy to the on-prem node (`nodeSelector`/`tolerations` in the values).
- Point `MIMIR_PUSH_URL` at the cloud Mimir endpoint; set `MIMIR_TENANT` if multitenancy is on; add auth/TLS to `remote_write` as your ingress requires.
- Set `tls_insecure = false` and trust your CA once arrays have proper certificates.
- Dashboard source: grafana.com ID 18515 rev 5, datasource rewritten to uid `mimir`.
