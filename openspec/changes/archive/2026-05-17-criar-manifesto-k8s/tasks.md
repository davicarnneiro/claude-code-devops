## 0. Preparação

- [x] 0.1 Carregar a skill `create-k8s` via `/create-k8s` para guiar a criação e validação dos manifestos conforme as regras da skill

## 1. Estrutura do diretório

- [x] 1.1 Criar o diretório `k8s/` na raiz do projeto

## 2. Namespace e Secret

- [x] 2.1 Criar o recurso `Namespace` com nome `kube-news`
- [x] 2.2 Criar o `Secret` com as credenciais do PostgreSQL em base64 (`DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD`, `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD`)

## 3. PostgreSQL

- [x] 3.1 Criar o `PersistentVolumeClaim` com `storageClassName: standard` e capacidade adequada para o banco
- [x] 3.2 Criar o `Deployment` do PostgreSQL com imagem `postgres:14-alpine`, env via `secretKeyRef`, `volumeMount` com `subPath: pgdata` e env `PGDATA=/var/lib/postgresql/data/pgdata`
- [x] 3.3 Adicionar `readinessProbe` no Deployment do PostgreSQL usando `exec` com `pg_isready -U <usuario> -d <banco>`
- [x] 3.4 Criar o `Service` do PostgreSQL como `ClusterIP` na porta `5432` com nome `postgres`

## 4. Aplicação

- [x] 4.1 Criar o `Deployment` da aplicação com imagem `davicarneiro/imersao-kube-news:v1`, `replicas: 2` e env via `secretKeyRef`
- [x] 4.2 Adicionar `livenessProbe` no Deployment da app com `httpGet` em `/health` porta `8080`
- [x] 4.3 Adicionar `readinessProbe` no Deployment da app com `httpGet` em `/ready` porta `8080`
- [x] 4.4 Criar o `Service` da aplicação como `LoadBalancer` expondo porta `80` com `targetPort: 8080`

## 5. Consolidação do arquivo

- [x] 5.1 Garantir que todos os recursos estão no arquivo único `k8s/kube-news.yml` separados por `---` na ordem: Namespace → Secret → PVC → Deployment banco → Service banco → Deployment app → Service app
- [x] 5.2 Verificar que todos os recursos possuem `metadata.namespace: kube-news`
- [x] 5.3 Adicionar comentários no arquivo indicando onde substituir `storageClass` e o tipo do Service em clusters locais

## 6. Teste e validação com kind

- [x] 6.1 Criar cluster Kubernetes local com `kind create cluster --name kube-news`
- [x] 6.2 Fazer o build da imagem Docker com `docker build -t davicarneiro/imersao-kube-news:v1 .`
- [x] 6.3 Fazer o push da imagem para o Docker Hub com `docker push davicarneiro/imersao-kube-news:v1`
- [x] 6.4 Aplicar os manifestos no cluster com `kubectl apply -f k8s/kube-news.yml`
- [x] 6.5 Aguardar os pods ficarem prontos com `kubectl wait --namespace kube-news --for=condition=Ready pod --all --timeout=120s`
- [x] 6.6 Verificar que os 2 pods da aplicação e o pod do banco estão em estado `Running` com `kubectl get pods -n kube-news`
- [x] 6.7 Validar os endpoints de saúde fazendo port-forward com `kubectl port-forward -n kube-news svc/app 8080:80` e testando `/health` e `/ready`
- [x] 6.8 Deletar o cluster kind após a validação com `kind delete cluster --name kube-news`
