# 🧭 Set Up a Local Kubernetes Cluster with `kubeadm` on macOS

> This guide shows how to create a real Kubernetes cluster (not Minikube or kind) using `kubeadm` inside Ubuntu VMs managed by [Multipass](https://multipass.run).

---

## ⚙️ 1. Install Multipass on macOS

```bash
brew install --cask multipass
```

Verify installation:

```bash
multipass version
```

---

## 🧱 2. Create a Control Plane VM

```bash
multipass launch --name k8s-master --cpus 2 --memory 4G --disk 20G
```

Enter the VM shell:

```bash
multipass shell k8s-master
```

### Enable IP forwarding

- `sudo sysctl -w net.ipv4.ip_forward=1` to enable it temporarily
- add `net.ipv4.ip_forward = 1` to your `/etc/sysctl.conf` file
- run `sudo sysctl -p` to make the change permanent.

---

## 🧩 3. Install Kubernetes Components (Inside the VM)

```bash
sudo apt update && sudo apt install -y apt-transport-https ca-certificates curl gpg
```

Add the Kubernetes repository:

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
```

Install the required components:

```bash
sudo apt install -y kubelet kubeadm kubectl containerd
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## ⚙️ 4. Configure Containerd

```bash
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

---

## 🚫 5. Disable Swap

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

---

## 🚀 6. Initialize the Control Plane

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

> 💡 Note: The `--pod-network-cidr` option is needed for most CNI plugins (e.g., Flannel, Calico).

When finished, note the `kubeadm join ...` command that is printed — you’ll use it later to add worker nodes.

```bash
kubeadm join 192.168.2.3:6443 --token 2fscn9.3vpvq1z7hds86xaa \
        --discovery-token-ca-cert-hash sha256:e7ebbb8216a1242c83d7b7c7547a26c9d36519bb10eef464a5f51a1b26c58ab6
```

---

## 🧑‍💻 7. Configure `kubectl` Access

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Test it:

```bash
kubectl get nodes
```

---

## 🌉 8. Install a Pod Network (CNI)

Example: **Flannel**

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

Alternative: **Calico**

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Wait until all system pods are running:

```bash
kubectl get pods -A
```

---

## 🧍‍♂️ 9. (Optional) Add a Worker Node

Create another VM:

```bash
multipass launch --name k8s-worker --cpus 2 --memory 4G --disk 20G
```

SSH into it:

```bash
multipass shell k8s-worker
```

Repeat **steps 3–5** inside the worker node (install kubeadm, disable swap, configure containerd).
Then, use the `kubeadm join` command from step 6 to connect it to the cluster.

---

## 🧰 10. Verify the Cluster

On your control-plane node:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl cluster-info
```

You should see something like:

```
NAME           STATUS   ROLES           AGE   VERSION
k8s-master     Ready    control-plane   5m    v1.30.0
k8s-worker     Ready    <none>          2m    v1.30.0
```

---

## 💡 11. Enable Scheduling on the Control Plane (Optional)

If you only have one node and want to schedule workloads on it:

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```

---

## 🧾 12. Access the Cluster from macOS

Copy your kubeconfig from the VM to your Mac:

```bash
multipass transfer k8s-master:/home/ubuntu/.kube/config ~/.kube/config
```

Now from macOS, you can run:

```bash
kubectl get nodes
```

✅ You’re now controlling your `kubeadm`-based cluster from macOS.

---

## 🧼 13. (Optional) Reset or Delete the Cluster

To tear down the cluster:

```bash
sudo kubeadm reset -f
sudo rm -rf ~/.kube
```

To delete the VMs completely (from macOS):

```bash
multipass delete --all
multipass purge
```

---

## 🧠 Summary

| Task                        | Command                                                                                                       |
|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| Create control plane VM     | `multipass launch --name k8s-master --cpus 2 --mem 4G --disk 20G`                                             |
| Initialize Kubernetes       | `sudo kubeadm init --pod-network-cidr=10.244.0.0/16`                                                          |
| Install network plugin      | `kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml` |
| Enable scheduling on master | `kubectl taint nodes --all node-role.kubernetes.io/control-plane-`                                            |
| Verify cluster              | `kubectl get nodes -A`                                                                                        |

---

