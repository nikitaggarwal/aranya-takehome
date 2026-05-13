# Aranya take-home runbook

This runbook walks you through reproducing the cluster from scratch on a fresh Mac with no Kubernetes tooling installed and 3 bare Ubuntu droplets in a DigitalOcean VPC. By the end you'll have:

- A 3-node Kubernetes 1.32 cluster (each node is control-plane + worker + etcd) installed by **kubespray**, with **Cilium** as the CNI.
- HA for in-cluster control-plane traffic via kubespray's localhost loadbalancer (one nginx per node fanning to all 3 apiservers).
- **ArgoCD** running inside the cluster, watching the clusterdOS git repo.
- **clusterdOS v0.3.21** with these gitapps enabled: cert-manager, metrics-server, node-feature-discovery, kube-state-metrics.
- An nginx `hello aranya` page reachable at `http://<any-node-public-ip>:30080`.
- An admin kubeconfig encrypted for Christian, Yoofi, and Sasi.

Reference repo: <https://github.com/nikitaggarwal/aranya-takehome>

If anything in this runbook is unclear, the **Troubleshooting notes** section at the bottom documents the two non-obvious snags I hit during the original build (kube-vip ARP on DO's VPC, Cilium init containers vs Ubuntu 24.04 file ownership) and why the config in this repo looks the way it does.

---

## 0. Inputs you'll need before starting

- The 3 node public IPs and the candidate SSH private key (delivered out of band).
- Christian's, Yoofi's, and Sasi's PGP public keys (delivered as email attachments).
- A GitHub account with permission to create a public repo, or any other public git host. The repo only needs to hold the inventory and manifests in this runbook — no secrets ever get pushed.

This runbook uses these placeholders — substitute your actual values:

| Placeholder | What it is |
| --- | --- |
| `NODE{1,2,3}_PUBLIC` | Public IPs of the 3 droplets. |
| `NODE{1,2,3}_PRIVATE` | Private VPC IPs (the addresses on `eth1`). |
| `~/.ssh/aranya_candidate_key` | The candidate-issued ed25519 private key. |

The values used in the original build (for reference, will be different on a fresh provision):

```
NODE1_PUBLIC=138.68.244.183   NODE1_PRIVATE=10.120.0.8
NODE2_PUBLIC=167.172.197.138  NODE2_PRIVATE=10.120.0.9
NODE3_PUBLIC=165.227.57.127   NODE3_PRIVATE=10.120.0.10
```

---

## 1. Set up local tooling on the Mac

```bash
# Homebrew (Mac package manager). Skip if `brew --version` already works.
curl -fsSL -o /tmp/brew.sh https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh
bash /tmp/brew.sh
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"

# CLI tools.
brew install kubectl gnupg helm jq gh python@3.12
```

Verify:

```bash
kubectl version --client
gpg --version | head -1
helm version --short
gh --version
python3.12 --version
```

---

## 2. Save the candidate SSH private key with the right perms

The key is the entire credential for SSH into the nodes. SSH refuses keys with loose perms (`0600` means only your user can read or write).

```bash
mkdir -p ~/.ssh && chmod 700 ~/.ssh

# Paste the key content from the email into ~/.ssh/aranya_candidate_key.
# Then:
chmod 600 ~/.ssh/aranya_candidate_key

# Sanity check: derive the public key from the private key.
ssh-keygen -y -f ~/.ssh/aranya_candidate_key
# Expected output starts with `ssh-ed25519 ... candidate-takehome`.
```

---

## 3. Verify SSH access and gather node facts

```bash
for ip in $NODE1_PUBLIC $NODE2_PUBLIC $NODE3_PUBLIC; do
  echo "===== $ip ====="
  ssh -i ~/.ssh/aranya_candidate_key \
      -o StrictHostKeyChecking=accept-new \
      -o ConnectTimeout=10 \
      root@$ip '
    echo "OS:        $(. /etc/os-release && echo "$PRETTY_NAME")"
    echo "Kernel:    $(uname -r)"
    echo "CPU:       $(nproc) cores"
    free -h | grep -E "Mem|Swap" | sed "s/^/  /"
    df -h / | tail -1 | awk "{print \"Disk free: \"\$4\" of \"\$2}"
    echo "Interfaces:"
    ip -4 -o addr show | awk "{print \"  \"\$2, \$4}"'
done
```

What to confirm:

- OS is Ubuntu 22.04 or 24.04 (kubespray supports both).
- `Swap: 0B 0B 0B` (K8s requires swap off).
- Each node has `eth0` (public IP) and `eth1` (private VPC `10.120.0.0/20`). The `eth1` address is what we'll use for inter-node K8s traffic.

Sanity check that the nodes can talk to each other on the private network:

```bash
ssh -i ~/.ssh/aranya_candidate_key root@$NODE1_PUBLIC \
    "ping -c 2 -W 2 $NODE2_PRIVATE && ping -c 2 -W 2 $NODE3_PRIVATE"
```

---

## 4. Add Christian's SSH keys to every node

Sasi's email requires Christian (`condaatje` on GitHub) to be able to SSH in as root. GitHub exposes any user's public SSH keys at `https://github.com/<user>.keys`.

```bash
curl -fsSL https://github.com/condaatje.keys -o /tmp/christian.keys

for ip in $NODE1_PUBLIC $NODE2_PUBLIC $NODE3_PUBLIC; do
  echo "===== $ip ====="
  cat /tmp/christian.keys | ssh -i ~/.ssh/aranya_candidate_key root@$ip '
    mkdir -p /root/.ssh && chmod 700 /root/.ssh
    touch /root/.ssh/authorized_keys && chmod 600 /root/.ssh/authorized_keys
    while IFS= read -r k; do
      [ -z "$k" ] && continue
      grep -qF "$k" /root/.ssh/authorized_keys || echo "$k" >> /root/.ssh/authorized_keys
    done
    echo "authorized_keys count: $(wc -l < /root/.ssh/authorized_keys)"
  '
done
rm /tmp/christian.keys
```

The `grep -qF ... || echo ...` pattern makes the script idempotent — running it twice doesn't duplicate keys.

---

## 5. Get the repo and Python dependencies

Pick a workspace directory:

```bash
mkdir -p ~/aranya-takehome && cd ~/aranya-takehome
```

Clone this repo to get the inventory and manifests:

```bash
git clone https://github.com/nikitaggarwal/aranya-takehome.git .
```

Clone kubespray (pinned to release-2.28; this repo's inventory is built against that):

```bash
git clone --depth 1 --branch release-2.28 https://github.com/kubernetes-sigs/kubespray.git
```

Create a Python venv and install kubespray's exact Ansible deps inside it:

```bash
/opt/homebrew/opt/python@3.12/libexec/bin/python -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r kubespray/requirements.txt
ansible --version | head -1   # should report ansible 9.13.0
```

The venv keeps Ansible's Python deps isolated from the system Python. Both `kubespray/` and `venv/` are in `.gitignore` — they're rebuilt locally per this section, not vendored into the repo.

---

## 6. Wire the inventory into kubespray

Open `inventory/inventory.ini` in this repo. Update the three lines under `[kube_control_plane]` if your node IPs differ from the originals:

```ini
node1 ansible_host=NODE1_PUBLIC  ip=NODE1_PRIVATE  access_ip=NODE1_PRIVATE  etcd_member_name=etcd1
node2 ansible_host=NODE2_PUBLIC  ip=NODE2_PRIVATE  access_ip=NODE2_PRIVATE  etcd_member_name=etcd2
node3 ansible_host=NODE3_PUBLIC  ip=NODE3_PRIVATE  access_ip=NODE3_PRIVATE  etcd_member_name=etcd3
```

- `ansible_host` is the address Ansible SSHes to from your Mac — public, because the Mac is off-VPC.
- `ip` and `access_ip` are the addresses K8s components on the nodes use to find each other — private, per the Aranya inter-node-on-private preference.

If the public IPs differ from the originals, also update the cert SAN list in `inventory/group_vars/k8s_cluster/k8s-cluster.yml`:

```yaml
supplementary_addresses_in_ssl_keys: [NODE1_PUBLIC, NODE2_PUBLIC, NODE3_PUBLIC]
```

Without those SANs, `kubectl` from your Mac (which hits the public IP) gets a TLS error because the apiserver's cert doesn't list the IP you're connecting to.

Sanity check that Ansible can reach all three nodes via this inventory:

```bash
ansible -i inventory/inventory.ini all -m ping
```

All three should return `"ping": "pong"`.

---

## 7. Run kubespray

```bash
cd kubespray
ansible-playbook -i ../inventory/inventory.ini --become --become-user=root cluster.yml | tee /tmp/kubespray.log
cd ..
```

Expect 20-30 minutes. The slow phases are package installs, container-image pulls (multi-GB across 3 nodes), and the Cilium daemonset rollout.

When it finishes, the play recap should report `failed=0` for every node. If you see `failed=1`, jump to the **Troubleshooting notes** at the bottom — there's a good chance it's one of the snags documented there.

---

## 8. Fetch the admin kubeconfig and verify the cluster

Kubespray writes the admin kubeconfig to `/etc/kubernetes/admin.conf` on every control-plane node. Copy it to your Mac and rewrite the server URL to point at a node we can reach:

```bash
mkdir -p ~/.kube

ssh -i ~/.ssh/aranya_candidate_key root@$NODE1_PUBLIC \
    "cat /etc/kubernetes/admin.conf" \
  | sed "s|server: https://127.0.0.1:6443|server: https://$NODE1_PUBLIC:6443|" \
  > ~/.kube/aranya-takehome.kubeconfig

chmod 600 ~/.kube/aranya-takehome.kubeconfig
export KUBECONFIG=~/.kube/aranya-takehome.kubeconfig

# Verify.
kubectl get nodes -o wide
kubectl -n kube-system get pods
```

All three nodes should be `Ready`. Every pod in `kube-system` should be `Running`.

---

## 9. Install ArgoCD

Apply the official manifests into a new `argocd` namespace. Use **server-side apply** — one of the ArgoCD CRDs (`applicationsets.argoproj.io`) is larger than the 256KB limit that client-side `kubectl apply` imposes (client-side apply tries to stash the full YAML in a metadata annotation).

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for the rollout (~1-2 min).
kubectl -n argocd rollout status deploy/argocd-server --timeout=5m

# Initial admin password (generated by the installer).
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d ; echo
```

To open the UI from your Mac (port-forward — the simplest path; we don't expose ArgoCD publicly):

```bash
kubectl -n argocd port-forward svc/argocd-server 8443:443
# Open https://localhost:8443 — accept the self-signed cert.
# Username: admin. Password: from the command above.
```

---

## 10. Apply clusterdOS

```bash
kubectl apply -f clusterdos/install.yaml
```

This creates a single ArgoCD `Application` named `clusterdos`. ArgoCD pulls `metadeployment/` from <https://gitlab.com/aranya-tech/public/clusterdOS.git> at tag `v0.3.21`, renders it as a Helm chart with the values in `clusterdos/install.yaml` (cluster name, enabled gitapps), and that chart expands into one child `Application` per enabled gitapp.

The enabled gitapps in `clusterdos/install.yaml` are:

| Gitapp | Purpose |
| --- | --- |
| `certmanager` | Issues TLS certs (e.g. via Let's Encrypt) for cluster services. |
| `metricsserver` | Collects realtime CPU/RAM metrics — required for `kubectl top`. |
| `nfd` | Node Feature Discovery — labels each node with detected hardware features. |
| `ksm` | Kube-state-metrics — exposes cluster-state metrics (pod phase, deployment health). My optional 4th. |

Wait for ArgoCD to reconcile everything:

```bash
kubectl -n argocd get applications -w
# Ctrl-C once every Application is SYNCED + HEALTHY.
```

You should end up with 6 Applications, all `Synced` + `Healthy`:

```
clusterdos                 Synced   Healthy
clusterdos-certmanager     Synced   Healthy
clusterdos-config          Synced   Healthy
clusterdos-ksm             Synced   Healthy
clusterdos-metricsserver   Synced   Healthy
clusterdos-nfd             Synced   Healthy
```

Sanity check metrics-server (proves it can scrape kubelets):

```bash
kubectl top nodes
# Should show CPU/MEM% for all three nodes.
```

If `clusterdos-certmanager` stays in `OutOfSync, Progressing` and you see `cert-manager-controller` pods in `CrashLoopBackOff`, see the **Troubleshooting notes** at the bottom — there's a documented edit to `clusterdos/install.yaml` for this. The version of `clusterdos/install.yaml` in this repo already has both overrides (cert-manager Gateway API gate + metrics-server kubelet TLS) baked in, so a fresh apply from this repo should not hit either issue. The notes are there for understanding *why* the YAML looks the way it does, and for debugging if the upstream clusterdOS chart changes in a future release.

---

## 11. Deploy the `hello aranya` page

```bash
kubectl apply -f manifests/hello-aranya.yaml
kubectl -n hello-aranya rollout status deploy/hello-aranya
```

The manifest creates:
- A `hello-aranya` namespace.
- A `ConfigMap` holding the HTML page (avoids needing a custom container image).
- A `Deployment` with 2 nginx replicas.
- A `Service` of type `NodePort` on port `30080`.

`NodePort` opens port 30080 on every node's IP and forwards to the pods. Verify from your laptop:

```bash
for ip in $NODE1_PUBLIC $NODE2_PUBLIC $NODE3_PUBLIC; do
  echo -n "http://$ip:30080 -> "
  curl -s -o /dev/null -w "HTTP %{http_code}\n" --max-time 5 http://$ip:30080/
done
```

All three should report `HTTP 200`. Visiting any of them in a browser shows the "hello aranya" page.

---

## 12. Build the encrypted credentials bundle for Christian, Yoofi, and Sasi

Import their public keys (delivered as email attachments):

```bash
gpg --import path/to/christian.asc.pub
gpg --import path/to/yoofi.asc.pub
gpg --import path/to/sasi.asc.pub
gpg --list-keys --keyid-format=long
```

Bundle the kubeconfig together with the ArgoCD admin password and a tiny README, then encrypt the tarball for all three recipients. The `--trust-model always` flag tells GPG to encrypt even though we haven't manually signed these keys as trusted — appropriate here because the keys came from the same email thread that gave us the assignment.

```bash
ARGOCD_PW=$(KUBECONFIG=~/.kube/aranya-takehome.kubeconfig \
  kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d)

WORKDIR=$(mktemp -d)
cp ~/.kube/aranya-takehome.kubeconfig $WORKDIR/admin.kubeconfig
echo "$ARGOCD_PW" > $WORKDIR/argocd-admin.txt
cat > $WORKDIR/README.txt <<EOF
admin.kubeconfig  — kubeadm cluster-admin kubeconfig
argocd-admin.txt  — ArgoCD initial admin password (port-forward to use)
Repo:  https://github.com/nikitaggarwal/aranya-takehome
EOF

tar -C $WORKDIR -czf /tmp/aranya-secrets.tar.gz .

gpg --trust-model always \
    --output ./aranya-secrets.tar.gz.gpg \
    --encrypt \
    --recipient christian.ondaatje@pm.me \
    --recipient ybquansah@gmail.com \
    --recipient sasivarnan619@gmail.com \
    /tmp/aranya-secrets.tar.gz

shred -u /tmp/aranya-secrets.tar.gz $WORKDIR/*  # clean up unencrypted copies
rmdir $WORKDIR
```

Verify the bundle was encrypted for all three (encryption goes to each recipient's `[E]` subkey, which is standard PGP — the keyids listed by `--list-packets` are subkey ids, not primary key ids):

```bash
gpg --list-packets ./aranya-secrets.tar.gz.gpg | grep "pubkey enc"
```

Each recipient decrypts with their own private key:

```bash
gpg --decrypt aranya-secrets.tar.gz.gpg | tar -xz
```

Attach `aranya-secrets.tar.gz.gpg` to your reply email along with `runbook.md`. The bundle is gitignored — do not commit it.

---

## 13. Push the code

The inventory, manifests, and this runbook should live in a public repo. The `.gitignore` in this repo excludes everything that should never be pushed: SSH keys, kubeconfigs, kubespray's working dir, the Python venv, and the `inventory/credentials/` directory kubespray writes secrets into.

```bash
git status                # confirm no secrets are staged
git push origin main
```

---

## Troubleshooting notes

These are the non-obvious problems I hit on the original build. All have been encoded into the inventory / `clusterdos/install.yaml` in this repo, so a fresh run of this runbook shouldn't re-trigger them — but they're worth knowing.

### kube-vip ARP doesn't work on DigitalOcean's VPC

The full story is in commit `99f4561` ("Pivot HA strategy: kube-vip ARP -> kubespray localhost LB"). In short:

Sasi's email lists kube-vip as a suggestion that "helps with" the no-SPOF preference. I initially enabled it in ARP mode (the simplest configuration, no BGP peer required). The leader node successfully claimed the VIP on its own `eth1` interface, but the gratuitous ARP announcement never reached other nodes — `ping VIP` from a non-leader returned "Destination Host Unreachable" and `ip neigh show` reported the VIP entry as `FAILED`. Direct pings to the leader's real eth1 IP worked fine.

Root cause: DigitalOcean's VPC is a software-defined routed network, not a true L2 broadcast domain. The DO SDN filters ARP for IPs that DO didn't assign. Same restriction affects kube-vip ARP on AWS and GCP without provider-specific integration.

Pivot in this repo: `kube_vip_enabled: false` in `inventory/group_vars/k8s_cluster/addons.yml`, and `loadbalancer_apiserver_localhost: true` with `loadbalancer_apiserver_type: nginx` in `inventory/group_vars/all/all.yml`. Kubespray installs a small nginx on every node listening at `127.0.0.1:6443` and fanning to all 3 real apiservers. Every kubelet, kube-proxy, and controller talks to its local nginx, so any single apiserver can die without disrupting in-cluster operations. External `kubectl` from your Mac points at one node's public IP and would need a kubeconfig update if that specific node died — an admin-access SPOF, not a cluster-availability SPOF.

A future version of this work could replace the localhost LB with kube-vip + DO Reserved IPs (kube-vip can move a real DO-assigned IP between droplets via the DO API), but that needs a DO API token we weren't given.

### Cilium init containers crash on Ubuntu 24.04 with default kubespray ownership

The full story is in commit `f72d30f` ("Fix Cilium init container crash on Ubuntu 24.04: kube_owner=root"). In short:

After the kube-vip pivot, kubespray got to the Cilium install phase and stalled with all 3 Cilium daemonset pods in `Init:CrashLoopBackOff`. The `mount-cgroup` init container, which copies `/usr/bin/cilium-mount` onto the host at `/opt/cni/bin/cilium-mount` via a hostPath mount, was failing with:

```
cp: cannot create regular file '/hostbin/cilium-mount': Permission denied
```

`ls -la /opt/cni/bin/` on the node showed it was owned by user `kube`. Cilium's mount-cgroup container has only `SYS_ADMIN`/`SYS_CHROOT`/`SYS_PTRACE` capabilities (not full privileged), and on Ubuntu 24.04 with AppArmor in enforce mode the kubelet profile blocks writes from this container to a non-root-owned host dir.

Fix in this repo: `kube_owner: root` in `inventory/group_vars/k8s_cluster/k8s-cluster.yml`. Kubespray's own CI tests for Cilium (`tests/files/ubuntu20-cilium-sep.yml`, `tests/files/rockylinux9-cilium.yml`) set this exact override. Default kubespray creates an unprivileged `kube` user and grants it ownership of every file/dir it lays down, which is fine for most CNIs but breaks Cilium-style "drop a binary onto the host via init container" patterns.

If you reset and re-run kubespray after flipping `kube_owner`, the in-place chowns are flaky — `ansible-playbook reset.yml -e reset_confirmation=yes` before re-running `cluster.yml` to be safe.

### clusterdOS chart defaults that break on a fresh cluster

The full story is in commit `cef5a9b` ("Step 7: clusterdOS installed with required gitapps + override broken defaults"). Two distinct chart-side issues, both fixed by `valuesObject` overrides already in `clusterdos/install.yaml`:

**cert-manager: `ExperimentalGatewayAPISupport=true` requires Gateway API CRDs.**
clusterdOS's chart sets `featureGates: "ExperimentalGatewayAPISupport=true"` and `config.enableGatewayAPI: true` by default. cert-manager-controller v1.20.2 then refuses to start unless Gateway API CRDs already exist in the cluster — they don't, since nothing in the take-home needs Gateway API. The override sets both back to false.

**metrics-server: kubelet TLS verification fails.**
metrics-server scrapes each kubelet over HTTPS. The kubelet TLS certs kubespray generates list hostnames in their SAN, not the kubelet's eth1 IP, so verification fails:
```
cannot validate certificate for 10.120.0.X because it doesn't contain any IP SANs
```
The override appends `--kubelet-insecure-tls` to the chart's args, which is the documented workaround. The cleaner fix would be enabling kubelet serving-cert rotation with IP SANs at the kubespray level, but that's orthogonal to this task.

### If ArgoCD gets stuck on the OLD valuesObject after you edit install.yaml

If `clusterdos-certmanager` stays `OutOfSync, Progressing` even after `install.yaml` is updated and re-applied, ArgoCD is likely stuck retrying an in-flight sync operation that was started with the old values. Unstick it:

```bash
# Terminate the in-flight sync.
kubectl -n argocd patch application clusterdos-certmanager \
  --type=json -p '[{"op":"remove","path":"/operation"}]'

# Delete the bad Deployment so ArgoCD has to recreate it.
kubectl -n clusterdos-cert-manager delete deploy clusterdos-certmanager-cert-manager

# Force ArgoCD to re-evaluate with the current valuesObject.
kubectl -n argocd annotate application clusterdos-certmanager \
  argocd.argoproj.io/refresh=hard --overwrite
```

ArgoCD will resync within ~30s with the new values; the cert-manager pods come up Healthy.
