# Homelab Kubernetes Cluster

## Stack

- **CNI**: Cilium
- **Load Balancer**: MetalLB
- **GitOps**: ArgoCD

---

## Troubleshooting: Pods stuck in `ContainerCreating`

### Symptom

All pods outside `kube-system` stuck in `ContainerCreating` indefinitely. Events show:

```
Warning  FailedCreatePodSandBox  kubelet  Failed to create pod sandbox: plugin type="cilium-cni"
         failed (add): failed to find plugin "cilium-cni" in path [/usr/lib/cni]
```

### Root Cause

containerd is configured (in `/etc/containerd/config.toml`) to look for CNI binaries in `/usr/lib/cni`, but Cilium installs its binary to `/opt/cni/bin`. When containerd tries to set up pod networking, it can't find the `cilium-cni` binary and fails.

Cilium's own pods in `kube-system` are unaffected because they were already running before this mismatch was hit.

### Fix

On **every node** (master and workers):

```bash
sudo sed -i 's|bin_dir = "/usr/lib/cni"|bin_dir = "/opt/cni/bin"|' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

Then force-delete the stuck pods so they are recreated with clean sandboxes:

```bash
kubectl delete pods -n argocd --all --grace-period=0 --force
kubectl delete pods -n metallb-system --all --grace-period=0 --force
```

### Why `metallb-memberlist` secret is missing

MetalLB speaker pods mount a secret named `metallb-memberlist` at startup. This secret is created by `metallb-controller` on its first run. If `metallb-controller` was stuck due to the CNI issue, the secret never gets created and speakers will show `FailedMount`. It resolves automatically once `metallb-controller` starts successfully.

### Why pod deletion is slow

Pods stuck in `ContainerCreating` still go through the full 30-second termination grace period on deletion, even though no container ever started. Use `--grace-period=0 --force` to skip it.
