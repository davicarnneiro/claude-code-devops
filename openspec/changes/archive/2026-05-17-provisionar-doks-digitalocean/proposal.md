## Why

O kube-news roda hoje apenas localmente via Docker Compose, o que não permite demonstrar Kubernetes real em ambiente gerenciado. Precisamos provisionar um cluster DOKS (DigitalOcean Kubernetes Service) para uso em aulas de demonstração, expondo a aplicação publicamente e mostrando scheduling, Services e integração com recursos cloud reais.

## What Changes

- Provisionar ambiente `dev` com cluster DOKS na DigitalOcean (região NYC1, single control plane, 2 nodes `s-2vcpu-2gb`) via Terraform
- Criar VPC dedicada para o ambiente `dev` via Terraform
- Configurar node pool fixo de 2 nodes (sem autoscaler) via Terraform
- Ajustar o manifest `k8s/kube-news.yml` para remover o PVC do Postgres (banco sem persistência, comportamento didático intencional)
- Expor a aplicação via Service `type: LoadBalancer` já existente no manifest (sem alteração)

## Capabilities

### New Capabilities

- `doks-cluster`: Cluster Kubernetes gerenciado na DigitalOcean para o ambiente `dev`, com VPC dedicada, node pool fixo e control plane single (sem HA), provisionado via Terraform

### Modified Capabilities

- `app-deployment`: sem alteração de requisitos — manifest existente já atende (Service LoadBalancer, probes, Secret)
- `postgres-deployment`: remover o PVC do Postgres — o banco passará a rodar sem volume persistente (dados perdidos em restart, comportamento didático intencional)

## Impact

- **Infraestrutura cloud**: criação de recursos na DigitalOcean para o ambiente `dev` (DOKS, VPC, Load Balancer) via Terraform — custo estimado ~US$ 48/mês se ligado 24/7
- **Novo diretório `terraform/`**: código Terraform para provisionar o ambiente `dev` (cluster DOKS e VPC)
- **`k8s/kube-news.yml`**: remoção do PersistentVolumeClaim e do volumeMount do Postgres
- **Docker Hub**: sem alterações no registry (`davicarneiro/imersao-kube-news`)
- **Sem alterações no código-fonte** da aplicação (`src/`)
