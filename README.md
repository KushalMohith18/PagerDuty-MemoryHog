# Kubernetes Memory Monitoring with Prometheus Operator, Grafana, and PagerDuty

## Overview

This Repo contains a working example for Kubernetes pod memory monitoring and PagerDuty alerting using:

- Prometheus Operator (kube-prometheus)
- Prometheus
- Grafana
- Alertmanager
- PagerDuty

The implementation in this repository is focused on:

- A stress pod in the `production` namespace (`memory-hog.yaml`)
- A Prometheus alert for high pod memory usage (`pod-memory-alert.yaml`)
- Alertmanager routing of `severity=critical` alerts to PagerDuty (`alertmanager.yaml`)

## Folder Contents

- `memory-hog.yaml`: test workload that consumes memory
- `pod-memory-alert.yaml`: `PrometheusRule` for `HighPodMemoryUsage`
- `alertmanager.yaml`: Alertmanager config template using `${PAGERDUTY_ROUTING_KEY}`
- `.env.example`: example environment file for local secret injection
- `.gitignore`: ignores local secret and rendered config files
- `kube-prometheus/`: local copy of kube-prometheus manifests and docs

Generated local-only files:

- `.env`: your local secret values
- `alertmanager.rendered.yaml`: rendered config with env values applied
- `alertmanager-encoded.txt`: optional base64 output for manual secret updates

## Architecture

```text
memory-hog pod (production namespace)
            |
            v
      Kubelet / cAdvisor
            |
            v
        Prometheus
   (scrape + rule eval)
         /       \
        v         v
   Grafana    Alertmanager
                  |
                  v
              PagerDuty
```

## Prerequisites

- A running Kubernetes cluster (Kind, Minikube, K3d, or cloud)
- `kubectl`
- Access to a PagerDuty service integration key

## Deploy Monitoring Stack

This workspace already includes `kube-prometheus/`, so cloning is optional.

```bash
cd kube-prometheus
kubectl apply --server-side -f manifests/setup
kubectl apply -f manifests/
kubectl get pods -n monitoring
```

## Deploy Test Workload

```bash
kubectl create namespace production
kubectl apply -f ../memory-hog.yaml
kubectl get pods -n production
```

`memory-hog.yaml` uses:

- image: `polinux/stress`
- memory request: `200Mi`
- memory limit: `500Mi`
- stress args: `--vm 1 --vm-bytes 400M --vm-hang 1`

## Apply Alert Rule

```bash
kubectl apply -f ../pod-memory-alert.yaml
kubectl get prometheusrules -n monitoring
```

The configured alert is:

- alert name: `HighPodMemoryUsage`
- group name: `pod-memory-alerts`
- condition: memory usage / memory limit > `0.8`
- duration: `for: 1m`
- severity label: `critical`

## Alertmanager and PagerDuty

The checked-in `alertmanager.yaml` already contains:

- route for `severity = critical` to receiver `Critical`
- receiver `Critical` with `pagerduty_configs`
- env placeholder for PagerDuty key: `${PAGERDUTY_ROUTING_KEY}`
- PagerDuty event description: `{{ .CommonAnnotations.summary }}`

Create your local env file from the example:

- Linux/macOS:

```bash
cp ../.env.example ../.env
```

- Windows PowerShell:

```powershell
Copy-Item ..\.env.example ..\.env
```

Set your real value in `.env`:

```env
PAGERDUTY_ROUTING_KEY=your_real_integration_key
```

Render a deployable Alertmanager config from env:

- Linux/macOS:

```bash
set -a
source ../.env
set +a
envsubst < ../alertmanager.yaml > ../alertmanager.rendered.yaml
```

- Windows PowerShell:

```powershell
$env:PAGERDUTY_ROUTING_KEY = "your_real_integration_key"
(Get-Content ..\alertmanager.yaml -Raw).Replace('${PAGERDUTY_ROUTING_KEY}', $env:PAGERDUTY_ROUTING_KEY) | Set-Content ..\alertmanager.rendered.yaml -NoNewline
```

Apply/update Alertmanager config via secret:

```bash
kubectl create secret generic alertmanager-main \
  -n monitoring \
  --from-file=alertmanager.yaml=../alertmanager.rendered.yaml \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl rollout restart statefulset alertmanager-main -n monitoring
```

`alertmanager.rendered.yaml` is a generated local file and is ignored by Git.

If you need base64 output for manual editing:

- Linux/macOS:

```bash
base64 -w0 ../alertmanager.rendered.yaml > ../alertmanager-encoded.txt
```

- Windows PowerShell:

```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("..\alertmanager.rendered.yaml")) | Out-File -Encoding ascii "..\alertmanager-encoded.txt"
```

## Grafana Query (Pod Memory in MB)

Use this query in Grafana:

```promql
sum by(pod)(
  container_memory_usage_bytes{namespace="production", container!="POD"}
) / 1024 / 1024
```

## Validate End-to-End Flow

1. Keep `memory-hog` running in `production`.
2. Wait for `HighPodMemoryUsage` to fire in Prometheus/Alertmanager.
3. Confirm incident creation in PagerDuty.

## Important Security Note

This repository is intentionally example-only and does not include real secrets:

- keep `.env` local and out of version control
- keep `alertmanager.rendered.yaml` local and out of version control
- commit only placeholders like `${PAGERDUTY_ROUTING_KEY}`
- prefer Kubernetes Secrets or external secret managers in real deployments

