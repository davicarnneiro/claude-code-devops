## Context

O kube-news é uma aplicação Node.js/Express com deploy atualmente manual (build local → push Docker Hub → kubectl apply). O processo leva ~1h por execução, ocorre 2x/dia, e é feito a partir da máquina do desenvolvedor com kubeconfig admin local. O cluster alvo é o DOKS descrito em `docs/arquitetura-cloud-digitalocean.md`. A aplicação não possui suite de testes — `npm test` é placeholder.

## Goals / Non-Goals

**Goals:**
- Automatizar build da imagem Docker e push para Docker Hub em cada `push` na `main`
- Automatizar deploy no cluster DOKS após CI passar
- Separar responsabilidades: CI (validar + construir) e CD (promover ao cluster)
- Garantir que CD só rode quando CI passou
- Servir como artefato didático em aulas de CI/CD

**Non-Goals:**
- Introduzir suite de testes automatizados (a aplicação não tem e isso é escopo da app, não da pipeline)
- Suportar múltiplos ambientes ou clusters
- Implementar GitOps (ArgoCD/Flux) — escopo futuro
- Gerenciar secrets rotation ou RBAC do cluster (documentado como pré-requisito externo)

## Decisions

### 1. Dois workflows separados (`ci.yml` + `cd.yml`) com `workflow_call`

CI invoca CD via `workflow_call` passando a tag da imagem como output. Isso garante: (a) CD só roda após CI completo; (b) separação didática clara entre "validar código" e "promover ao cluster"; (c) CD pode ser acionado manualmente via `workflow_dispatch` sem depender do CI.

Alternativa descartada: workflow único — mistura responsabilidades, menos didático, dificulta explicar separação em aula.

### 2. Tag de imagem: `github.run_number`

Número incremental monotônico gerado automaticamente pelo GitHub Actions. A 47ª execução gera `:47`. Simples, rastreável e didático — o número do run no Actions é o mesmo da tag no Docker Hub.

Alternativa descartada: SHA do commit (`github.sha`) — menos legível para fins didáticos; tags semânticas — exigiriam ensinar versionamento antes da pipeline.

### 3. `azure/k8s-set-context@v4` para autenticação no cluster

Ação mantida pela Microsoft/Azure, funciona com qualquer cluster Kubernetes (não apenas AKS), recebe o kubeconfig diretamente do secret via `method: kubeconfig`. Evita instalar e configurar `doctl` no runner.

Alternativas descartadas: `azure/aks-set-context` — exige `AZURE_CREDENTIALS` (service principal Azure), incompatível com DOKS; `doctl` action — adiciona dependência e configuração extra; instalar kubectl manualmente — mais verboso sem benefício.

### 4. `azure/k8s-deploy@v5` para apply de manifests + substituição de imagem

Action oficial Azure que em um único step executa: apply dos manifests (`k8s/`), substituição da tag da imagem via parâmetro `images:`, e aguarda o rollout. Elimina três scripts kubectl manuais e adiciona anotações de proveniência nos resources do cluster.

```yaml
- uses: azure/k8s-deploy@v5
  with:
    namespace: kube-news
    manifests: k8s/
    images: davicarneiro/evento-kube-news:${{ inputs.image-tag }}
```

Alternativa descartada: scripts manuais `kubectl apply` + `kubectl set image` + `kubectl rollout status` — válido didaticamente, mas exige manutenção manual e não adiciona rastreabilidade no cluster.

### 5. `concurrency: deploy-main` no job de deploy

Impede que dois deploys rodem simultaneamente, evitando race condition de `rollout status` em dois workflows concorrentes.

## Risks / Trade-offs

- **Vazamento do kubeconfig** → Mitigação: gerar kubeconfig de ServiceAccount restrito ao namespace `kube-news` (RBAC limitado), não copiar kubeconfig admin do `doctl`.
- **Rate limit Docker Hub no cluster** → Mitigação: documentado como ponto de aula; se ocorrer, configurar `imagePullSecret` no namespace.
- **PR de fork disparando CD** → Mitigação: invocação do CD condicionada explicitamente a `github.event_name == 'push' && github.ref == 'refs/heads/main'`.
- **`azure/k8s-deploy` substituindo imagem errada** → Mitigação: o parâmetro `images:` faz match pelo nome da imagem nos manifests — o Deployment em `k8s/` deve referenciar `davicarneiro/evento-kube-news` para que a substituição funcione.
- **Schema migration implícita do Sequelize** → Risco herdado da aplicação (`sequelize.sync({ alter: true })`); a pipeline não causa mas precipita. Endereçar no escopo da app.

## Migration Plan

1. Configurar secrets no repositório: `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`, `KUBE_CONFIG`
2. Garantir pré-requisitos: `src/Dockerfile`, pasta `k8s/` com Namespace `kube-news`, Deployment (container chamado `kube-news`) e Service
3. Garantir cluster DOKS provisionado e kubeconfig coletado
4. Criar `.github/workflows/ci.yml` e `.github/workflows/cd.yml`
5. Fazer `git push main` e verificar execução no Actions
6. Critério de sucesso: workflow verde, `rollout status` OK em <180s, pods `Running`, `curl` no LoadBalancer retorna 200

**Rollback:** acionar `cd.yml` via `workflow_dispatch` passando o `run_number` anterior como `image-tag`, ou manualmente: `kubectl -n kube-news set image deploy/kube-news kube-news=davicarneiro/evento-kube-news:<run_number_anterior>`

## Open Questions

- O kubeconfig do cluster DOKS já foi coletado? (pré-requisito para primeira execução do CD)
- Os manifests em `k8s/` já existem com o nome de container `kube-news`?
