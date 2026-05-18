## 1. Pré-requisitos

- [x] 1.1 Confirmar que `src/Dockerfile` existe na raiz do repositório
- [x] 1.2 Confirmar que a pasta `k8s/` existe com Namespace `kube-news`, Deployment (container chamado `kube-news`) e Service
- [ ] 1.3 Configurar secret `DOCKERHUB_USERNAME` em `Settings → Secrets and variables → Actions`
- [ ] 1.4 Configurar secret `DOCKERHUB_TOKEN` (Personal Access Token com escopo Read & Write) em `Settings → Secrets and variables → Actions`
- [ ] 1.5 Configurar secret `KUBE_CONFIG` (kubeconfig de ServiceAccount restrito ao namespace `kube-news`) em `Settings → Secrets and variables → Actions`

## 2. Workflow CI (`ci.yml`)

- [x] 2.1 Criar `.github/workflows/ci.yml` com gatilhos `push` (todas as branches) e `pull_request`
- [x] 2.2 Adicionar job `validate`: `actions/checkout` → `actions/setup-node@v4` com cache npm → `npm ci` em `src/`
- [x] 2.3 Adicionar job `build-and-push` (`needs: validate`, condicional a `push` na `main`): checkout → `docker/setup-buildx-action` → `docker/login-action` (Docker Hub) → `docker/build-push-action` com contexto `./src`, tag `davicarneiro/evento-kube-news:${{ github.run_number }}` e cache `type=gha`
- [x] 2.4 Expor output `image-tag: ${{ github.run_number }}` no job `build-and-push`
- [x] 2.5 Adicionar job `call-cd` (`needs: build-and-push`, condicional a `push` na `main`): `uses: ./.github/workflows/cd.yml` com `secrets: inherit` e `with: image-tag: ${{ needs.build-and-push.outputs.image-tag }}`

## 3. Workflow CD (`cd.yml`)

- [x] 3.1 Criar `.github/workflows/cd.yml` com gatilho `workflow_call` (input `image-tag` obrigatório) e `workflow_dispatch` (input `image-tag` manual)
- [x] 3.2 Adicionar job `deploy` com `environment: production` e `concurrency: deploy-main`
- [x] 3.3 Adicionar step `azure/k8s-set-context@v4` com `method: kubeconfig` e secret `KUBE_CONFIG`
- [x] 3.4 Adicionar step `azure/k8s-deploy@v5` com `namespace: kube-news`, `manifests: k8s/` e `images: davicarneiro/evento-kube-news:${{ inputs.image-tag }}`

## 4. Validação

- [ ] 4.1 Fazer `git push main` e verificar que workflow `CI` executa e fica verde
- [ ] 4.2 Verificar que a imagem `davicarneiro/evento-kube-news:<run_number>` aparece no Docker Hub
- [ ] 4.3 Verificar que o workflow `CD` executou após o CI com a tag correta
- [ ] 4.4 Verificar `kubectl -n kube-news rollout status deploy/kube-news` retorna sucesso
- [ ] 4.5 Verificar `kubectl -n kube-news get pods` mostra pods `Running` com a imagem na tag esperada
- [ ] 4.6 Verificar `curl` no endpoint público do LoadBalancer retorna 200 na rota `/`
