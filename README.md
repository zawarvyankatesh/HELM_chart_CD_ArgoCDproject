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

## Move the release to Argo CD

Argo CD reads this public Git repository, renders `charts/taskboard` using Helm, and applies the resulting Kubernetes objects. The bootstrap Application manifest is [`argocd/taskboard-application.yaml`](argocd/taskboard-application.yaml). It is kept outside the chart directory so it is not rendered as part of the Taskboard application.

Install Argo CD into the current kind cluster using the [upstream instructions](https://argo-cd.readthedocs.io/en/stable/getting_started/):

```bash
kubectl config current-context
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get pods
kubectl -n argocd rollout status deployment/argocd-server --timeout=300s
kubectl -n argocd rollout status deployment/argocd-repo-server --timeout=300s
```

**Handoff:** You installed Taskboard using `helm upgrade --install` above. Stop the Helm release before Argo CD takes over. The next command deletes its managed PVC and current tasks, which is acceptable for this lab; keep the namespace and your kind cluster:

```bash
helm uninstall taskboard --namespace taskboard
kubectl -n taskboard get deployment,service,pvc
kubectl apply -f argocd/taskboard-application.yaml
kubectl -n argocd get applications
kubectl -n taskboard get deployments,pods,svc,hpa,pvc
```

The Application watches `main` and automatically syncs chart changes. `selfHeal` restores changes made manually in the cluster. Automatic pruning is disabled so deleting a chart resource from Git does not automatically delete its PVC or other live resources. The Application manifest itself is applied once with `kubectl`; edits to that manifest need another `kubectl apply`, as it is outside the chart's watched path.

See the Argo CD UI locally (leave the port-forward running; use another port if 8080 is occupied by the web port-forward):

```bash
kubectl -n argocd port-forward service/argocd-server 8081:443
```

Open <https://localhost:8081>, accept the local certificate warning, and log in as `admin`. Get the initial password in another terminal without adding it to Git:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

The Argo CD UI should show Taskboard as `Synced` and `Healthy`. Verify the app itself with `kubectl -n taskboard port-forward service/taskboard-web 8080:80` and <http://localhost:8080>. For an exercise, change `web.autoscaling.maxReplicas` in `charts/taskboard/values.yaml`, commit and push to `main`, and watch Argo CD update `kubectl -n taskboard get hpa taskboard-web`. Then restore the value in Git. The local `:local` images still need to be present in every kind node that can schedule the Pods; pushing a new image to ECR comes later.
