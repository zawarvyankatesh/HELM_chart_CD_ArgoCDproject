# Orbit Taskboard Helm chart

This repository contains the GitOps-ready Helm chart at [`charts/taskboard`](charts/taskboard). It deploys the web (Nginx), API (FastAPI), and Redis components from [the application repo](https://github.com/zawarvyankatesh/application_code_argocd_project). The frontend proxies `/api/` to the `taskboard-api` Service, and the API connects to the `taskboard-redis` Service. Those two Service names must stay as they are unless you rebuild the frontend image.

Default images are `taskboard-web:local`, `taskboard-api:local` and `redis:7-alpine`. Both application images use `imagePullPolicy: Never` because they are loaded into the kind nodes. Redis is downloaded if missing. Web and API each have a CPU HPA (1–3 replicas, 75% of CPU request); Redis always has one replica. Redis uses a fresh 1 GiB PVC and append-only persistence. The old test PVC and data are not migrated.

## Validate the chart

```bash
git clone https://github.com/zawarvyankatesh/HELM_chart_CD_ArgoCDproject.git
cd HELM_chart_CD_ArgoCDproject
helm lint charts/taskboard
helm template taskboard charts/taskboard --namespace taskboard > /tmp/taskboard-rendered.yaml
kubectl apply --dry-run=client --validate=false -f /tmp/taskboard-rendered.yaml
```

The last command checks the rendered resource syntax; it does not install anything. The chart does not create a Namespace; install with `--create-namespace` or use your existing `taskboard` namespace.

## First install in the existing kind cluster

Verify the context and that both local images are in the **correct kind cluster**:

```bash
kubectl config current-context
kind get clusters
kind load docker-image taskboard-web:local taskboard-api:local --name YOUR_KIND_CLUSTER_NAME
```

Your old `kubectl apply -f k8s/kind.yaml` resources have the same names as this chart. Helm cannot take over existing Kubernetes objects automatically. Because you do not need their test data, remove the old workloads, Services and PVC before the first Helm install. This **deletes the existing Redis data**:

```bash
kubectl -n taskboard delete deployment taskboard-web taskboard-api taskboard-redis --ignore-not-found
kubectl -n taskboard delete service taskboard-web taskboard-api taskboard-redis --ignore-not-found
kubectl -n taskboard delete pvc taskboard-redis-data --ignore-not-found

helm upgrade --install taskboard charts/taskboard \
  --namespace taskboard --create-namespace --wait --timeout 3m

kubectl -n taskboard get deployments,pods,svc,hpa,pvc
kubectl -n taskboard port-forward service/taskboard-web 8080:80
```

Open <http://localhost:8080> to create and update tasks. In another terminal, verify the API:

```bash
curl -i http://localhost:8080/healthz
curl -i http://localhost:8080/api/tasks
```

If an application Pod shows `ErrImageNeverPull`, its image is not loaded in the cluster that your current kubectl context targets; load it again into that cluster. The Redis Pod pulls `redis:7-alpine` unless the image is already available in kind.

## HPA prerequisites

CPU autoscaling requires a working Kubernetes Metrics API, typically supplied by Metrics Server. The CPU requests in `values.yaml` are necessary for CPU utilization calculations. Without metrics, the application still starts, but HPA will show `<unknown>` and cannot scale. Check:

```bash
kubectl top nodes
kubectl -n taskboard get hpa
kubectl -n taskboard describe hpa taskboard-api
```

HPA controls the web and API replica counts. Redis is deliberately a single Pod because this chart runs one writable Redis instance, not a Redis cluster. Adjust `web.autoscaling` and `api.autoscaling` in `values.yaml` for this lab. If you disable either HPA, that component uses its `replicas` value.

## Later: replace local images with ECR images

After publishing images, set `web.image.repository`, `web.image.tag`, `api.image.repository`, `api.image.tag`, and the corresponding Redis image values to the ECR URIs and tags. Set each application's `image.pullPolicy` to `IfNotPresent` and configure Kubernetes pull access for private ECR. Keep those values in this Git repository for Argo CD. Do not use mutable tags for repeatable GitOps releases.

For the next phase, Argo CD will read this Git repository and use `charts/taskboard` as its Application source. Do not run `helm upgrade` and enable Argo CD automated sync on the same release at the same time; let Argo CD be the deployment owner after the handoff.
