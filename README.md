# yaml-configs

Configs I copy into projects and then forget how I tuned them. Keeping them in one place means the comments survive.

Nothing here is generic. It's all one fictional-but-realistic service called `web-api` so the labels and selectors line up across files, which is the part people actually get wrong.

## What's in here

```
k8s/deployment.yml          3 replicas, probes, non-root, read-only rootfs
k8s/service.yml             ClusterIP :80 -> container port named "http"
k8s/ingress.yml             nginx + cert-manager, TLS, /metrics blocked
prometheus/prometheus.yml   scrape configs, k8s pod SD, blackbox probes
docker-compose/full.yml     web + postgres + redis + prometheus + grafana
ansible/playbook.yml        Debian/Ubuntu deploy, systemd, ufw
helm/values.yaml            prod-ish overrides for the chart
ci/gitlab-ci.yml            lint / test / build / deploy
```

## The k8s bit

`deployment.yml`, `service.yml` and `ingress.yml` all agree on:

- `app.kubernetes.io/name: web-api`
- `app.kubernetes.io/instance: web-api`
- container port **named** `http` (8080)

The Service targets `targetPort: http` and the Ingress points at `port.name: http`. Rename the port in one place and the other two follow by name, not by number. I switched to named ports after a deploy where I changed 8080 to 8000 and only remembered two of the three files.

```bash
kubectl apply -f k8s/
kubectl -n apps get endpoints web-api      # should list 3 IPs, not 0
```

If you get a 503 from the ingress and `kubectl get endpoints web-api` shows `<none>`, your Service selector doesn't match the pod template labels. That's almost always it.

A few decisions worth knowing about:

- **`maxUnavailable: 0`** with 3 replicas. A bad rollout stalls instead of dropping capacity. Slower, and I've never regretted it.
- **startupProbe** at 2s x 30. Covers cold starts, so the liveness probe doesn't need a paranoid `initialDelaySeconds`.
- **`preStop: sleep 5`** plus `terminationGracePeriodSeconds: 30`. Gives the Service time to stop routing before the process dies.
- **`readOnlyRootFilesystem: true`** plus an `emptyDir` on `/tmp`. Python's `tempfile` will find out immediately if you forget the volume.
- **`/metrics` is denied at the ingress.** Prometheus scrapes pods directly, so there's no reason to expose it publicly.

## Prometheus

Three kinds of job in there:

1. **static_configs** for `prometheus` itself, `node-exporter`, and the blackbox targets.
2. **`kubernetes-pods`** which discovers pods by the `prometheus.io/scrape` annotation. That's why `deployment.yml` sets those annotations.
3. **blackbox** probes for HTTP and raw TCP. The `__param_target` relabeling looks like a mistake the first time you read it. It isn't.

Reload without a restart:

```bash
curl -X POST http://localhost:9090/-/reload
```

That needs `--web.enable-lifecycle`, which is in the compose file.

**Gotcha:** if a pod never appears in `/targets`, check the annotation names character by character. `prometheus.io/scrape` is easy to typo as `prometheus.io/scrape_`. Nothing warns you.

## Compose

```bash
cp .env.example .env          # not in this repo, you write it
docker compose -f docker-compose/full.yml up -d
```

`.env` needs `POSTGRES_PASSWORD` and `GRAFANA_PASSWORD`. Both are referenced with `${VAR:?message}` so compose refuses to start rather than silently creating a database with an empty password. That syntax is the reason those look noisy.

Ports for prometheus and grafana are bound to `127.0.0.1` only. Everything else talks over the `backend` network by service name.

## Ansible

```bash
ansible-galaxy collection install community.general
ansible-playbook -i inventory.ini ansible/playbook.yml --check
```

The playbook wants Debian or Ubuntu and asserts on that up front. It creates a `deploy` user, checks out a tagged release into `releases/<version>`, flips a `current` symlink, installs a systemd unit, and finishes by hitting `/healthz` with retries so a failed deploy fails the run.

You need to supply `templates/web-api.service.j2` and `templates/web-api.env.j2`. They're project-specific and I'd only be guessing at your exec path.

## Helm

`helm/values.yaml` is the override file, not a chart. The chart lives elsewhere.

```bash
helm upgrade --install web-api ./charts/web-api -n apps -f helm/values.yaml
```

## GitLab CI

Assumes a Docker executor with DinD, plus these CI/CD variables: `KUBE_CONTEXT_STAGING`, `KUBE_CONTEXT_PROD`. The registry ones (`CI_REGISTRY_*`) are provided by GitLab.

Production is a `when: manual` job on tags. Staging auto-deploys from the default branch.

## Notes

Every file here parses with PyYAML 6.x and I've run the k8s manifests through `kubectl apply --dry-run=client`. The service names and hostnames are made up (`api.example.com`, `10.0.20.11`). Swap them.

MIT.
