# Kubernetes Pod Networking — Step-by-step Notes
# Big-picture summary

* ✅ **A Pod is the network boundary in Kubernetes.**
* ✅ All containers inside a **single pod** share the **same network namespace** and thus the **same IP address** and `localhost`.
* 🧭 The CNI plugin creates a Linux network namespace (often named `cni-xxx`) for each pod on the node and sets up a veth pair to connect the pod to the node network.

---

# Diagram / Sketch explained

(From your paper)

* **Pod1**: contains `C1`, `C2`, `C3`.
  * `C1` & `C2` are `sleep` debug containers.
  * `C3` is `nginx` (server).
* **Pod2**: contains `C4` which is `nginx`.
* **Important**:
  * `C1`, `C2`, `C3` → **share pod IP** and `localhost`.
  * `Pod1` and `Pod2` → **different network namespaces** and different IPs.

---

# Key terms & symbols

* **Pod** — smallest deployable unit in Kubernetes (one network namespace).
* **Network Namespace (netns)** — Linux kernel isolation of network stack.
* **CNI (Container Network Interface)** — plugin that creates netns and attaches interfaces.
* **veth pair** — virtual ethernet link: one end in pod netns (`eth0`), other on node (bridge).
* **Pod IP** — IP assigned to the pod netns; shared by all containers in that pod.

Symbols used in notes:

* ✅ — correct / success / share
* ⚠️ — caution / possible issue
* ⛓️ — network link / veth pair
* 🧭 — where to run (node vs pod)
* 🔎 — inspect / identify
* 🛠️ — troubleshooting action

---

# Commands — where to run & what they show (step-by-step)

> **Note:** Commands are grouped by where to run: **Node** (ssh into the worker node) vs **Pod** (use `kubectl exec`).

## A. On your laptop / control plane (kubectl)

1. **Get pods and IPs**
```bash
kubectl get pods -o wide
```
* 🔎 Shows pod names, node, and Pod IP in the *IP* column.

2. **Get only a pod's IP**
```bash
kubectl get pod <pod-name> -o jsonpath='{.status.podIP}{"\n"}'
```
* ✅ Useful for scripting.

## B. Inside a Pod (use kubectl exec)

3. **Show network interfaces inside a container**
```bash
kubectl exec <pod-name> -c <container-name> -- ip addr
```
* 🧭 Run this to see `eth0` and the pod IP. For multi-container pods, any container shows the same `ip addr`.

4. **Curl localhost (hit service inside same pod)**
```bash
kubectl exec <pod-name> -c <container-name> -- curl -sS localhost:80
```
* ✅ Because containers share the netns, `localhost` reaches any process listening on 0.0.0.0 in the pod.

5. **Test connectivity to another pod IP**
```bash
POD2_IP=$(kubectl get pod pod2 -o jsonpath='{.status.podIP}')
kubectl exec pod1 -c container1 -- curl -v http://$POD2_IP:80
```
* 🔎 Tests an actual TCP/HTTP connection from pod1 → pod2.

## C. On the Node (inspect CNI netns)

6. **List network namespaces**
```bash
ip netns
```
* 🔎 Shows netns names like `cni-xxxxxxxx`.

7. **Show interfaces inside a specific cni netns**
```bash
sudo ip netns exec cni-12345678 ip addr
```
* ✅ Shows `eth0` and IP assigned by CNI for that pod.

8. **Find a container/Pod PID and inspect namespace**
```bash
crictl ps | grep <pod-name>
crictl inspect <container-id> | jq '.info.pid'
sudo nsenter -t <pid> -n ip addr
```
* 🔎 Map runtime PID to its network namespace (alternate approach).

---

# Map CNI netns (`cni-xxx`) to a Pod — step-by-step

1. `kubectl get pods -o wide` → note Pod name and node.
2. On that **node**: use runtime tool to find container ID and PID:
```bash
crictl ps --name <pod-name>
crictl inspect <container-id> | jq '.info.pid'
```
3. Use `readlink` to inspect netns:
```bash
readlink /proc/<pid>/ns/net
```
4. If CNI created `/var/run/netns/cni-xxx`, you can run:
```bash
sudo ip netns exec cni-xxx ip addr
```

---

# Pod-to-Pod connectivity test (step-by-step)

1. Get Pod2 IP:
```bash
kubectl get pod pod2 -o jsonpath='{.status.podIP}{"\n"}'
```
2. From Pod1, run curl to Pod2 IP:
```bash
kubectl exec pod1 -c container1 -- curl -v http://<POD2_IP>:80
```
3. Expected results:
* ✅ Success: HTTP response from Pod2 (nginx), connectivity OK.
* ⚠️ Failure: connection refused / timed out — see Troubleshooting below.

---

# Troubleshooting checklist (step-by-step)

1. 🔎 **Confirm IP is correct**
```bash
kubectl get pod pod2 -o wide
```
2. 🔎 **Is something listening in Pod2?**
```bash
kubectl exec pod2 -c <container> -- ss -ltnp
```
3. ⚠️ **NetworkPolicy**
```bash
kubectl get networkpolicy --all-namespaces
```
4. ⛓️ **Node / CNI issues**
```bash
ip netns
sudo ip netns exec cni-xxx ip addr
```
5. 🔎 **Try from node**
```bash
ping <pod-ip>
```
6. 🛠️ **Check kube-proxy & iptables** (if using Services).

---

# Quick reference / cheat sheet

### Get pods and IPs
```bash
kubectl get pods -o wide
kubectl get pod <pod> -o jsonpath='{.status.podIP}{"\n"}'
```

### Exec into container and check network
```bash
kubectl exec <pod> -c <container> -- ip addr
kubectl exec <pod> -c <container> -- curl -sS localhost:80
kubectl exec pod1 -c container1 -- curl -v http://<pod2IP>:80
```

### On node: inspect network namespaces
```bash
ip netns
sudo ip netns exec cni-12345678 ip addr
readlink /proc/<pid>/ns/net
sudo nsenter -t <pid> -n ip addr
```

### Map container → PID (containerd)
```bash
crictl ps --name <pod-name>
crictl inspect <container-id> | jq '.info.pid'
```

---

