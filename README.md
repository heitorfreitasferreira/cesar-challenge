# cesar-challenge — repo GitOps (ENTREGAVEL)

Fonte de verdade do deploy da todolist-app no cluster `desafio` (k3d local).
Padrao espelhado de `infra-k3s`: manifests finais aqui, CI no repo da app,
Argo CD reconcilia Git -> cluster. Digest imutavel versionado aqui; o repo
da app nao e segunda fonte de verdade.

## O que esta onde

| Caminho | Papel (requisito) |
|---|---|
| `clusters/desafio/k3d-config.yaml` | R1: cluster criado por codigo |
| `clusters/desafio/argocd/` | R1: install + projects + applications do Argo CD |
| `bootstrap/` | R1: ingress (traefik) + postgres simples (Deployment/Service/PVC, sem HA) |
| `k8s/base + k8s/overlays/desafio/` | R2/R4: manifests da app (fonte unica — nao duplicar no fork) |
| `../todolist-app/.github/workflows/` | R2: CI (build -> GHCR -> bump de digest aqui) |
| `evidencias/` | R5: logs, screenshots |

## Reproducao

```bash
export KUBECONFIG=~/.kube/desafio-k3d.kubeconfig
k3d cluster create --config clusters/desafio/k3d-config.yaml
# + bootstrap argocd/ingress/postgres (ver clusters/desafio/argocd/)
```

App acessivel em `http://localhost:8081` (R3).
Detalhes de operacao: `DECISIONS.md`.
