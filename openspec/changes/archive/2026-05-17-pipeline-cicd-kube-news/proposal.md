## Why

O processo de deploy do kube-news é inteiramente manual — `docker build`, `docker push` e `kubectl apply` rodando na máquina do desenvolvedor — consumindo ~1h por execução com 2 execuções/dia, totalizando 2h diárias de exposição a erro humano em um processo crítico. A automação via GitHub Actions elimina esse gargalo e serve como artefato didático em aulas de CI/CD.

## What Changes

- Adicionar `.github/workflows/ci.yml` — workflow de validação (`npm ci`) e build+push da imagem Docker para o Docker Hub
- Adicionar `.github/workflows/cd.yml` — workflow de deploy no cluster DOKS via `kubectl apply` + `set image` + `rollout status`
- CI invoca CD via `workflow_call` passando a tag da imagem como output, garantindo que CD só rode após CI passar
- Tag da imagem: `github.run_number` (número incremental monotônico — `:1`, `:2`, `:3`, ...)
- Desenvolvedores param de rodar build/push/apply localmente; apenas `git push` na main é necessário

## Capabilities

### New Capabilities

- `ci-validate-build`: Validação de dependências Node.js e build+push da imagem Docker com tag baseada em `github.run_number`
- `cd-deploy`: Deploy automatizado no cluster DOKS com `kubectl apply`, `set image` e verificação via `rollout status`

### Modified Capabilities

<!-- Nenhuma capability existente tem requisitos alterados por esta mudança -->

## Impact

- **Repo:** dois arquivos novos em `.github/workflows/`. Nenhum arquivo existente é modificado
- **Docker Hub** (`davicarneiro/evento-kube-news`): passa a receber pushes automáticos com tags numéricas crescentes
- **Cluster DOKS, namespace `kube-news`:** passa a receber `apply` + `set image` + `rollout status` automáticos via GitHub Actions
- **Secrets do repositório:** `DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN` e `KUBE_CONFIG` precisam ser configurados em `Settings → Secrets and variables → Actions`
- **Fluxo do desenvolvedor:** reduz de ~1h de trabalho manual para um simples `git push main`
