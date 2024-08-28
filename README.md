# ArgoCD KSOPS test

This repository allows you to run a k3s cluster using K3D which runs a basic ArgoCD with in cluster decryption using
kustomize and sops.

## Prerequisites

- Install Docker

```bash
sudo apt install docker.io
sudo groupadd docker
sudo usermod -aG docker $USER
```

- Install K3D

```bash
sudo curl -s https://raw.githubusercontent.com/rancher/k3d/main/install.sh | bash
```

- Install Kubectl

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```

## Start k3d cluster

Using 2 worker nodes and 1 control/master node

```bash
sudo k3d cluster create argocd-ksops --api-port 6550 --agents 2 --servers 1
```

## Get the kubeconfig file

```bash
sudo k3d kubeconfig get argocd-ksops > ~/.kube/config
```

## Setup GPG key and setup SOPS

```bash
gpg --batch --gen-key gen-key.conf
```

List GPG keys:

```bash
gpg --list-secret-keys argocd-ksops.cluster
export GPG_ID=<Your SEC output>
#export GPG_ID=0xA94C431BB3CB41DBCC6F3BCCA64B1010017CB27E
```

Export public and private key:

```bash
gpg --export-secret-keys --armor "${GPG_ID}" > gpg.key
gpg --armor --export "${GPG_ID}" > gpg.pub
gpg --delete-secret-keys "${GPG_ID}" #optional
```

## Install ArgoCD

Apply the secret for KSOPS decryption to the cluster:

```bash
kubectl create namespace argocd
kubectl create secret generic sops-gpg \
--namespace=argocd \
--from-file=gpg.key
```

## Get SOPS
```bash
curl -LO https://github.com/getsops/sops/releases/download/v3.9.0/sops-v3.9.0.linux.amd64
chmod +x sops-v3.9.0.linux.amd64
sudo mv sops-v3.9.0.linux.amd64 /usr/local/bin/sops
```

Encrypt your secret with sops:

```bash
sops --encrypt argocd/overlays/local/dummy.Secret.yaml > argocd/overlays/local/dummy.Secret.enc.yaml
sops -e --gpg "${GPG_ID}" argocd/overlays/local/dummy.Secret.yaml > argocd/overlays/local/dummy.Secret.enc.yaml
```

Apply with Kustomize:

```bash
kubectl apply -k argocd
```

### Get/Update password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
# start this in a separate terminal
kubectl port-forward svc/argocd-server -n argocd 8080:443
# then go back to login to argocd
argocd login 127.0.0.1:8080
# Username: admin
# Password: <from kubectl get secret response, without %>
argocd account update-password
```
