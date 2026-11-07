# k3s stack: data, observability and app

Manifests for a k3s cluster of 4 VMs: SQL Server, Redis, RabbitMQ, an Elasticsearch cluster with Kibana, APM Server and Filebeat, and an API with 3 workers behind Traefik.

## Layout

```
k8s/
  kustomization.yaml      all-in-one: file list, image tags, generated ConfigMap and Secrets
  namespaces.yaml         data, observability, app
  secrets/                *.env.example only; real *.env files are git-ignored
  data/                   sqlserver, redis, rabbitmq
  observability/          elasticsearch, kibana, apm-server, filebeat, ingress
  app/                    api, worker, shared-pvc, ingress
```

## Where things run

| Component | Placement |
|---|---|
| Redis, RabbitMQ | Control plane VM (`nodeSelector: node-role.kubernetes.io/control-plane`) |
| Elasticsearch | 4 pods, one per VM, control plane included (pod anti-affinity) |
| Filebeat | One pod on every VM (DaemonSet) |
| SQL Server | The VM labelled `role=data` |
| Kibana, APM Server, API, workers | Anywhere the scheduler picks |
| Traefik | Built into k3s, in `kube-system` |

The control plane VM runs workloads, so it needs enough memory: k3s server, Elasticsearch (2Gi), Redis and RabbitMQ together. Plan on 8 GB per VM. To lighten it, set the Elasticsearch `replicas` to 3 and trim the two node lists in its env.

## One-time setup

1. **Install k3s.** Server on VM 1 with `curl -sfL https://get.k3s.io | sh -`. Do not taint it, so pods can run there. Agents on the other VMs with `K3S_URL=https://SERVER_IP:6443 K3S_TOKEN=... ` and the same script.
2. **Get a kubeconfig on your laptop.** Copy `/etc/rancher/k3s/k3s.yaml` to `~/.kube/config` and replace `127.0.0.1` with the server IP. Check with `kubectl get nodes`.
3. **Check the control plane label.**
   ```bash
   kubectl get nodes --show-labels | grep control-plane
   ```
4. **Label the SQL Server VM.**
   ```bash
   kubectl label node VM2 role=data
   ```
5. **Install the NFS provisioner** for the shared volume (an NFS export must exist first):
   ```bash
   helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
   helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
     -n kube-system --set nfs.server=NFS_IP --set nfs.path=/exports/k8s --set storageClass.name=nfs-client
   ```
6. **Create the secrets.**
   ```bash
   cd k8s/secrets
   for f in *.env.example; do cp "$f" "${f%.example}"; done
   ```
   Edit the three `.env` files. They are git-ignored.
7. **Replace the placeholders:** `registry.example.com/api` and `/worker`, port 8080 with a `/health` probe, and the `*.example.local` hostnames in both `ingress.yaml` files. Point those names in DNS or `/etc/hosts` at any node IP.

## Deploy

```bash
kubectl kustomize k8s/        # render first, read the output
kubectl diff -k k8s/          # what would change
kubectl apply -k k8s/
kubectl rollout status deploy/api -n app
```

New version: change `newTag` under `images:` in `kustomization.yaml` and apply again. Use a unique tag per build (version or git SHA), never `latest`.

Bad release:

```bash
kubectl rollout undo deploy/api -n app
```

## Check Elasticsearch

```bash
kubectl exec -n observability elasticsearch-0 -- curl -s localhost:9200/_cluster/health
```

Expect `green` with 4 nodes. Yellow or red appears while a node restarts.

## Known gaps

- Not rendered or applied against a live cluster. Run `kubectl kustomize k8s/` before the first apply.
- Elastic images are pinned to 9.5.6. Confirm the tags exist for Elasticsearch, Kibana, APM Server and Filebeat before applying.
- SQL Server uses `2022-latest`. Pin a specific tag once you pick one.
- Standalone APM Server is the least-tested piece. If it crash-loops, check its logs first.
- Elasticsearch security is off and Redis has no password. Fine inside a private cluster for a first pass, not for real data.
- Redis, RabbitMQ and SQL Server are single instances on local-path volumes. A lost VM means lost data without backups.
- Elasticsearch needs an index lifecycle policy, or the volumes fill up.
