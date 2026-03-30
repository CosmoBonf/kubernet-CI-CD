# kubernet-CI-CD

## Objetivo

Exemplo de portfólio com:

- API simples (Go) com endpoints `/`, `/healthz`, `/readyz`
- Docker (build multi-stage, runtime distroless)
- Kubernetes com Kustomize (`k8s/base` + `k8s/overlays`)
- CI/CD com GitHub Actions (CI + build/push no GHCR e deploy opcional)
- Rancher (UI web) para gerenciar o cluster

## Passo a passo (Pop-OS / Ubuntu)

### 1) Rodar a API localmente

```bash
cd /home/cf/cfsupport/kubernet-CI-CD
go test ./...
go run ./cmd/server
```

Testes:

```bash
curl -i http://localhost:8080/
curl -i http://localhost:8080/healthz
curl -i http://localhost:8080/readyz
```

### 2) Subir o Rancher (interface web) via Docker

Se as portas `80/443` ou `8443` estiverem ocupadas, use portas alternativas (ex.: `18080` e `18443`).

```bash
sudo docker rm -f rancher 2>/dev/null || true

sudo docker run -d --restart=unless-stopped \
  --privileged \
  -p 18080:80 -p 18443:443 \
  --name rancher \
  rancher/rancher:latest
```

Acessar:

- https://127.0.0.1:18443

### 3) Login no Rancher (senha bootstrap)

A senha do primeiro login não é fixa; o Rancher gera uma senha bootstrap nos logs do container.

Para obter:

```bash
sudo docker logs rancher 2>&1 | grep -i "Bootstrap Password"
```

No Rancher:

- Usuário: `admin`
- Senha: use a `Bootstrap Password` encontrada acima

Depois do primeiro login, o Rancher pede para você criar uma senha nova.

### 4) Criar um cluster Kubernetes local (opção recomendada: microk8s)

Instalar o microk8s:

```bash
sudo snap install microk8s --classic
sudo usermod -a -G microk8s $USER
newgrp microk8s
microk8s status --wait-ready
```

Habilitar addons comuns:

```bash
microk8s enable dns ingress metrics-server
```

Testar:

```bash
microk8s kubectl get nodes
microk8s kubectl get pods -A
```

Se você quiser usar `kubectl` “puro”, instale via snap:

```bash
sudo snap install kubectl --classic
kubectl version --client
```

### 5) Deploy da API no Kubernetes

Com microk8s:

```bash
microk8s kubectl apply -k k8s/overlays/dev
microk8s kubectl -n kubernet-ci-cd get all
microk8s kubectl -n kubernet-ci-cd port-forward svc/kubernet-ci-cd 8080:80
```

Teste:

```bash
curl -i http://localhost:8080/
```

## CI/CD (GitHub Actions)

- CI: `.github/workflows/ci.yml` roda em push/PR (format, test, vet, build docker sem push).
- CD: `.github/workflows/cd.yml` roda em push na `main` e tags `v*` (build+push no GHCR).
- Deploy automático no cluster é opcional e só roda se existir o secret `KUBE_CONFIG_DATA` no repositório.
