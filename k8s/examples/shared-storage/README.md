# Shared storage: hostPath versus NFS

Two demos with the same three pods, so the only thing that changes is where the folder lives.

| Pod | Runs on | What it does |
|---|---|---|
| `writer` | control plane VM | copies the mock files in, then appends a line to `log.txt` every 10 s |
| `reader-same-node` | control plane VM | lists the folder and prints the last lines of `log.txt` every 20 s |
| `reader-other-node` | a worker VM | does the same, from a different VM |

`mock-data/` holds the fake files (`hello.txt`, `report-001.csv`, `settings.json`). Kustomize turns them into a ConfigMap, and an init container in `writer` copies them into the shared folder.

Each demo folder has its own `kustomization.yaml`, so you apply a folder, like `docker build .`:

```
kubectl apply -k <folder>
```

## Demo 1: hostPath

The folder is `/srv/shared-demo` on the VM's own disk. It is created on whichever VM a pod lands on.

```bash
kubectl apply -k k8s/examples/shared-storage/hostpath
kubectl get pods -n storage-demo -o wide
kubectl logs -n storage-demo deploy/reader-same-node --tail=8
kubectl logs -n storage-demo deploy/reader-other-node --tail=8
```

What you should see:
- `reader-same-node` lists `hello.txt`, `report-001.csv`, `settings.json` and `log.txt`, and the log lines keep growing.
- `reader-other-node` shows an empty listing and `tail: can't open '/shared/log.txt'`. It mounted a folder with the same path, but on a different VM's disk.

So hostPath only shares data between pods on the same VM.

Clean up:

```bash
kubectl delete -k k8s/examples/shared-storage/hostpath
# on every VM that ran a pod:
sudo rm -rf /srv/shared-demo
```

## Demo 2: NFS

The folder lives on one NFS server and every VM mounts it over the network. Here the server is the control plane VM.

**On the control plane VM (the NFS server):**

```bash
sudo apt install -y nfs-kernel-server
sudo mkdir -p /srv/nfs/shared-demo
sudo chmod 777 /srv/nfs/shared-demo
echo '/srv/nfs/shared-demo 10.0.0.0/24(rw,sync,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -ra
```

Replace `10.0.0.0/24` with the network your VMs share. The chmod keeps the demo simple, because the containers write as root and NFS maps root to an unprivileged user by default.

**On every VM (the NFS clients):**

```bash
sudo apt install -y nfs-common
showmount -e NFS_SERVER_IP
```

**On your laptop:** put the control plane VM's real IP in `nfs/demo.yaml` (`nfs.server`, not 127.0.0.1 or localhost), then:

```bash
kubectl apply -k k8s/examples/shared-storage/nfs
kubectl get pods -n storage-demo -o wide
kubectl logs -n storage-demo deploy/reader-same-node --tail=8
kubectl logs -n storage-demo deploy/reader-other-node --tail=8
```

What you should see: both readers list the same four files and the same growing `log.txt`, even though one runs on a different VM. You can also read the files on the server in `/srv/nfs/shared-demo`.

If a pod sticks in `ContainerCreating`, run `kubectl describe pod` and read the Events. A mount error usually means `nfs-common` is missing on that VM, the export does not allow the VM's IP, or the server IP is wrong.

Clean up:

```bash
kubectl delete -k k8s/examples/shared-storage/nfs
sudo rm -rf /srv/nfs/shared-demo/*   # on the server, if you want the files gone
```

The PersistentVolume uses `Retain`, so deleting the claim never deletes the files on the server.

## Which to use

| | hostPath | local-path volume | NFS |
|---|---|---|---|
| Where the data lives | a folder you name on one VM | a folder k3s creates on one VM | one NFS server |
| Shared between VMs | no | no | yes |
| Pod must stay on one VM | yes | yes | no |
| Survives its VM dying | no | no | only if the NFS server is alive |
| Use for | demos, node-level agents like Filebeat | single-instance databases | files shared by pods on several VMs |

- hostPath gives the pod access to the VM's real filesystem, which is a security risk. Avoid it for apps. The local-path storage class is the managed version of the same idea.
- If everything that shares the folder is pinned to the control plane VM, hostPath or a local-path volume works. You lose the freedom to move pods and spread them over VMs.
- Putting the NFS server on the control plane VM makes that VM a single point of failure for the shared files. A dedicated NAS or file server, or object storage, avoids that.
