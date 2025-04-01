# GitOps for k3s.bigli.io

## Repo structure

* Each subdirectory is a namespace
* `_apps` is the meta directory for Argo CD apps

## Secrets

Secrets are encrypted using [SOPS](https://github.com/mozilla/sops) and [age](https://github.com/FiloSottile/age).
Argo CD uses [KSOPS](https://github.com/viaduct-ai/kustomize-sops) and [kustomize](https://github.com/kubernetes-sigs/kustomize/).

Install `sops` and `age` packages on Arch Linux.

Public key: `age1pjrv5skm6p380h8smk4x9sp2m0valkz7ldh4npqrrk55vrwn0veqsvv375`

The installation and configuration happens in a kustomize patch in `argocd/`.

A good helper to work with SOPS encrypted secrets is [vscode-sops](https://github.com/signageos/vscode-sops).

The `age` key needs to be stored at `$HOME/.config/sops/age/keys.txt`

### Usage

Create a normal secret with a `.sops.yaml` file ending. Encrypt it with:

```
sops --encrypt --in-place secret.sops.yaml
```

Create a kustomize configuration to generate the secret:

secret-generator.yaml
```yaml
apiVersion: viaduct.ai/v1
kind: ksops
metadata:
  name: secret-generator
files:
  - ./secret.sops.yaml
```

kustomization.yaml
```yaml
generators:
  - ./secret-generator.yaml
```

## Argo CD

Either

`sudo -E kubefwd svc -n argocd` and then https://argocd-server/

or

`kubectl port-forward svc/argocd-server -n argocd 8080:443` and
then https://localhost:8080/

## Maintenance

* K3s is kept up-to-date using [System Upgrade Controller (SUC)](https://github.com/rancher/system-upgrade-controller).
  See `system/system-upgrade-controller/plan.yaml`.
* The OS is kept up-to-date using unattended upgrades and [kured](https://github.com/kubereboot/kured).
  See `system/kube-system/unattended-upgrades.yaml`.

## Backup

### Kubernetes native - K8up

K8up has a global configuration in `system/apps/k8up.yaml`.
To access the storage destination which is only available in the tailnet, a Jspolicy policy injects Tailscale into the backup Pods.
See `system/k8up`.

### Host level

There is a full filesystem backup done on the host using BorgMatic.
See `/etc/borgmatic/config.yaml` for the configuration.

## Bootstrap GitOps

```
kubectl create ns argocd
kubectl -n argocd create secret generic sops-age --from-file=$HOME/.config/sops/age/keys.txt
kubectl -n argocd apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get pods -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
argocd login argocd-server

kubectl apply -f system/apps/appprojects.yaml
kubectl apply -f system/apps/_root.yaml
```

## Bootstrap K3s

```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC='--flannel-backend=none --disable-network-policy --disable local-storage --tls-san k3s.bigli.io --cluster-cidr=10.42.0.0/16,2001:cafe:42:0::/56 --service-cidr=10.43.0.0/16,2001:cafe:42:1::/112' sh -
```

Then install Cilium: https://docs.cilium.io/en/stable/installation/k3s/

```
cilium install --version 1.17.2 --set=ipam.operator.clusterPoolIPv4PodCIDRList="10.42.0.0/16" --helm-set ipv6.enabled=true
cilium status
```
