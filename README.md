# Aranya take-home

3-node Kubernetes cluster built end-to-end on DigitalOcean droplets for the Aranya take-home assignment. Bootstrapped with [kubespray](https://github.com/kubernetes-sigs/kubespray), networked with Cilium, with [clusterdOS](https://gitlab.com/aranya-tech/public/clusterdos) gitapps reconciled by ArgoCD.

## Reproduce from scratch

[`runbook.md`](./runbook.md) is the source of truth — 13 numbered sections that take a fresh local control machine (macOS or Linux) plus 3 bare Ubuntu droplets to a fully running cluster (HA control plane, Cilium CNI, ArgoCD, clusterdOS, public `hello aranya` page). Section 8a documents the inter-node networking checks; the troubleshooting notes at the bottom explain the non-obvious snags (kube-vip ARP on DO's VPC, Cilium init-container ownership on Ubuntu 24.04, clusterdOS chart overrides).

## Live deliverables

| What | Where |
| --- | --- |
| `hello aranya` page | <http://138.68.244.183:30080> · <http://167.172.197.138:30080> · <http://165.227.57.127:30080> |
| Admin kubeconfig + ArgoCD initial admin password | Encrypted bundle attached to the email (decrypt with `gpg --decrypt aranya-secrets.tar.gz.gpg \| tar -xz`) |
| ArgoCD UI | Not publicly exposed. Reach via `kubectl -n argocd port-forward svc/argocd-server 8443:443` after using the bundled kubeconfig. |

## Repo layout

| Path | Purpose |
| --- | --- |
| `runbook.md` | End-to-end reproduction guide. |
| `inventory/inventory.ini` | Kubespray inventory — 3 nodes, stacked etcd, SSH over public IPs, K8s components on private VPC. |
| `inventory/group_vars/` | Kubespray overrides: Cilium CNI, `kube_owner=root`, localhost-LB HA, private-IP networking, public-IP SAN list. |
| `clusterdos/install.yaml` | clusterdOS parent ArgoCD `Application` (pinned to v0.3.21). Enables cert-manager, metrics-server, node-feature-discovery, kube-state-metrics. |
| `manifests/hello-aranya.yaml` | Demo workload — nginx Deployment + ConfigMap + NodePort Service on `:30080`. |

Kubespray itself, the Python venv, SSH keys, kubeconfigs, and the encrypted credentials bundle are all gitignored — the runbook covers how to materialize them on a fresh machine.

## Architecture at a glance

- Kubernetes 1.32; every node is control-plane + worker + etcd (stacked).
- **CNI**: Cilium (preferred over Calico per the assignment).
- **Inter-node traffic**: stays on the private DO VPC (`10.120.0.0/20` on `eth1`). External `kubectl` is the only path that uses public IPs.
- **HA control plane**: 3 stacked-etcd apiservers — etcd retains quorum with 2 of 3 nodes, and each kubelet talks to its own local apiserver, so any single node can fail without taking the cluster down. Originally planned around kube-vip, pivoted to the localhost-LB pattern after DO's VPC turned out to filter gratuitous ARP (commit `99f4561` has the full forensics).
- **clusterdOS gitapps enabled**: `certmanager`, `metricsserver`, `nfd`, plus `ksm` (kube-state-metrics) as the optional 4th. Cilium gitapp deliberately disabled — kubespray installs Cilium at bootstrap time.

The commit history is annotated — each step is its own commit with the reasoning in the message.
