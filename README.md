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

---

## Troubleshooting: MetalLB speaker in `CrashLoopBackOff`

### Symptom

MetalLB speaker pods crash-loop. Logs show the speaker creating ARP responders successfully then exiting cleanly after ~15 seconds with no error — until you catch the full log:

```
unable to start manager: ... dial tcp 10.96.0.1:443: i/o timeout
```

Liveness probe also fails because port 7472 never comes up:

```
Warning  Unhealthy  Liveness probe failed: Get "http://<node-ip>:7472/metrics": connection refused
```

### Root Cause

Cilium was configured with `kube-proxy-replacement: false` but kube-proxy was not running. This leaves ClusterIP routing completely unhandled — nothing translates `10.96.0.1` (the `kubernetes` service ClusterIP) to the actual API server. The MetalLB speaker uses `hostNetwork: true` and tries to reach the API server via ClusterIP on startup, times out, and exits.

### Fix

Enable kube-proxy replacement in Cilium so it handles ClusterIP routing via its eBPF datapath:

```bash
kubectl patch configmap -n kube-system cilium-config --type merge \
  -p '{"data":{"kube-proxy-replacement":"true"}}' && \
kubectl rollout restart daemonset -n kube-system cilium && \
kubectl rollout status daemonset -n kube-system cilium
```

Then restart MetalLB speaker:

```bash
kubectl rollout restart daemonset -n metallb-system metallb-speaker && \
kubectl rollout status daemonset -n metallb-system metallb-speaker
```

Since kube-proxy was already absent, there are no conflicting iptables rules — Cilium takes over cleanly.
