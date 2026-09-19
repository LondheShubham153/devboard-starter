# Setting up Kind

`kind-config.yaml` here pins the cluster to Kubernetes **v1.36.1** — see
`../docs/kubernetes.md` for why that matters. You need four tools before
running `../k8s/kind-up.sh`: Docker, `kind`, `kubectl`, and `helm`.

## Docker

All platforms: install [Docker Desktop](https://www.docker.com/products/docker-desktop/)
(Mac/Windows) or Docker Engine (Ubuntu), then confirm it's running:

```bash
docker version
```

## Ubuntu

```bash
# Docker Engine
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER"   # log out/in after this

# kind
[ "$(uname -m)" = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.32.0/kind-linux-amd64
[ "$(uname -m)" = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.32.0/kind-linux-arm64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -Ls https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x ./kubectl && sudo mv ./kubectl /usr/local/bin/kubectl

# helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

## macOS

```bash
brew install docker kind kubectl helm
open -a Docker   # start Docker Desktop, wait for it to be ready
```

(No Homebrew? Install it first from https://brew.sh.)

## Windows

Run in PowerShell. Install Docker Desktop manually first (WSL2 backend), then:

```powershell
choco install kind kubernetes-cli kubernetes-helm -y
```

(No Chocolatey? Install it first from https://chocolatey.org/install, or use
`winget install Kubernetes.kind Kubernetes.kubectl Helm.Helm` instead.)

If you're on WSL2, you can skip Chocolatey and just follow the **Ubuntu**
steps above inside your WSL distro.

## Verify

```bash
docker version
kind version      # v0.32.0
kubectl version --client
helm version
```

## Try it: run a plain nginx Pod

Once your cluster is up (`kind create cluster --config kind/kind-config.yaml`
or `../k8s/kind-up.sh`), a quick way to check `kubectl` can talk to it:

```bash
kubectl run nginx --image=nginx:alpine   # create the Pod
kubectl get pods                         # wait for STATUS: Running
kubectl port-forward pod/nginx 8081:80   # forward a local port to it
curl http://localhost:8081               # nginx welcome page, in another terminal
kubectl delete pod nginx                 # clean up
```
