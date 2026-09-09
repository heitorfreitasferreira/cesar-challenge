# DECISIONS.md — log flat de decisoes (R5)

Formato: data + contexto + escolha + descarte. Sem cerimonia de ADR (1 decisao = 1 item).

- 2026-09-09 · k3d para o cluster. Contexto: padrao do trabalho e k3s; desafio permite
  local. Escolha: k3d (k3s em Docker, descartavel em 1 comando). Descarte: kind (nao e k3s),
  minikube (sujaria ~/.minikube), cloud (custo + conta pessoal).
- 2026-09-09 · 2 repos (opcao A). Contexto: separar CI (app) de CD (GitOps).
  Escolha: fork `todolist-app` (codigo + workflow) + `cesar-challenge` (manifests + Argo CD).
  Descarte: monorepo (misturaria CI com fonte de deploy).
- 2026-09-09 · Manifests em fonte unica. Contexto: council J2 apontou risco de drift.
  Escolha: `k8s/` SOMENTE em `cesar-challenge`; fork sem `k8s/`. CI faz bump de digest aqui.
- 2026-09-09 · Postgres simples junto no k3d. Contexto: validar SELECT/health sem HA.
  Escolha: Deployment + Service + PVC (sem replicas). Descarte: StatefulSet/HA, Postgres externo.
- 2026-09-09 · Argo CD fonte de verdade. Escolha: Application aponta para
  `k8s/overlays/desafio`; imagem por digest imutavel (`@sha256:`), nunca `:latest`.
- 2026-09-09 · `.opencode/` fora do entregavel. Contexto: IA encorajada mas avaliador
  grade README/DECISIONS/evidencias. Escolha: MCPs e comandos ficam no workspace local.
