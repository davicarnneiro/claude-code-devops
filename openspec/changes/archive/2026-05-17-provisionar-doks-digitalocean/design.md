## Context

O kube-news é uma aplicação Node.js/Express com banco Postgres, usada como artefato didático para aulas de containers, Kubernetes e CI/CD. A execução hoje acontece via Docker Compose local. A proposta é provisionar um ambiente `dev` com cluster DOKS via Terraform para demonstrações reais de Kubernetes em cloud, destruindo o cluster após cada aula para controlar custo.

O manifest `k8s/kube-news.yml` já existe com todos os recursos Kubernetes necessários (Secret, Deployments, Services). O único ajuste necessário é remover o PVC do Postgres, alinhando o manifest à decisão de não usar persistência em ambiente de demo.

**Restrições fixas (definidas no design doc de arquitetura):**
- Provedor: DigitalOcean (premissa didática, não negociável)
- Região: NYC1
- Node type: `s-2vcpu-2gb` × 2 nodes (pool fixo)
- Registry: Docker Hub externo (`davicarneiro/imersao-kube-news`)
- Banco: Postgres dentro do cluster, sem volume persistente (decisão didática)
- Monitoramento DO e backup: desabilitados
- Ferramenta de IaC: Terraform

## Goals / Non-Goals

**Goals:**
- Provisionar ambiente `dev` com cluster DOKS mínimo viável para aulas de demonstração via Terraform
- Expor a aplicação publicamente via DigitalOcean Load Balancer
- Ajustar `k8s/kube-news.yml` para remover PVC e volumeMount do Postgres

**Non-Goals:**
- Alta disponibilidade (sem HA no control plane)
- Persistência de dados entre aulas (banco sem PVC)
- Autoscaling de nodes
- TLS/HTTPS ou domínio próprio
- Monitoramento ou alertas
- Pipeline automatizada de deploy (o deploy é manual, didático)
- Uso de DOCR (DigitalOcean Container Registry)

## Decisions

### 1. Terraform para provisionamento do cluster DOKS

**Decisão:** usar Terraform com o provider `digitalocean/digitalocean` para criar VPC e cluster DOKS.

**Rationale:** Terraform é a ferramenta padrão de IaC no contexto do evento. Permite recriar o ambiente de forma reproduzível a cada aula com `terraform apply`, e destruir com `terraform destroy`, sem estado manual.

**Alternativa considerada:** `doctl` (CLI da DigitalOcean). Descartado — não é a ferramenta definida para este projeto.

### 2. Service `type: LoadBalancer` direto (sem Ingress)

**Decisão:** manter o Service `type: LoadBalancer` já existente no manifest para expor a aplicação.

**Rationale:** o cluster hospeda uma única aplicação. Ingress Controller adicionaria complexidade didática sem benefício real para uma única app sem TLS.

**Alternativa considerada:** NGINX Ingress com 1 LB compartilhado. Descartado: desnecessário para aplicação única.

### 3. Remoção do PVC do Postgres no manifest existente

**Decisão:** remover o `PersistentVolumeClaim`, o `volumeMount` e o `volumes` do Deployment do Postgres em `k8s/kube-news.yml`.

**Rationale:** decisão didática — demonstrar que dados são perdidos ao restartar o pod é parte do aprendizado. O manifest atual tem PVC com `storageClassName: standard`, que não é compatível com DOKS sem configuração adicional. Remover o PVC simplifica o manifest e alinha ao design doc.

**Alternativa considerada:** manter PVC com StorageClass do DOKS (`do-block-storage`). Descartado — persistência não é objetivo da demo e adiciona custo (DO Block Storage).

### 4. VPC dedicada gerenciada pelo Terraform

**Decisão:** criar VPC dedicada via recurso `digitalocean_vpc` no Terraform, antes do cluster.

**Rationale:** isola os recursos do cluster de outros eventuais recursos da conta DigitalOcean. Custo zero. Gerenciar no Terraform garante que a VPC é destruída junto com o cluster no `terraform destroy`.

### 5. Estrutura Terraform com `modules/` e `envs/`

**Decisão:** organizar o código Terraform com módulo reutilizável em `terraform/modules/doks/` e ambiente em `terraform/envs/dev/`.

**Rationale:** separar o módulo do ambiente permite reutilizar a lógica de provisionamento do cluster (VPC + DOKS) para outros ambientes futuros sem duplicação. A estrutura `envs/<ambiente>/` com `versions.tf`, `terraform.tfvars`, `variables.tf` e `outputs.tf` padroniza o contrato de cada ambiente.

**Estrutura:**
```
terraform/
├── modules/
│   ├── vpc/
│   │   ├── main.tf        # recurso: digitalocean_vpc
│   │   ├── variables.tf   # inputs: nome, região, ip_range
│   │   └── outputs.tf     # outputs: vpc_id, vpc_urn
│   └── doks/
│       ├── main.tf        # recurso: digitalocean_kubernetes_cluster (recebe vpc_id como input)
│       ├── variables.tf   # inputs: região, nome, node_size, node_count, vpc_id
│       └── outputs.tf     # outputs: cluster_id, cluster_endpoint, kubeconfig
└── envs/
    └── dev/
        ├── main.tf           # chama module "vpc" e module "doks" (passa vpc_id do módulo vpc)
        ├── variables.tf      # declaração das variáveis do ambiente
        ├── outputs.tf        # re-exporta outputs dos módulos
        ├── terraform.tfvars  # valores concretos do dev (excluído do git)
        └── versions.tf       # required_providers digitalocean ~> 2.0
```

## Risks / Trade-offs

- **Rate limit Docker Hub (pull anônimo)** → aceito; mitigação futura: adicionar `imagePullSecrets` com credenciais Docker Hub se ocorrer `ImagePullBackOff` em demos com múltiplos restarts.
- **Sem HA no control plane (SLA 99.5%)** → aceito; ambiente de demonstração, indisponibilidade durante janela de manutenção DO é tolerável.
- **Pool fixo de 2 nodes** → se um node cair, cluster opera com 1 node de 2 GB; risco de `Pending` pods por falta de recursos. Aceito para demo.
- **Sem persistência no Postgres** → dados perdidos em restart de pod; comportamento esperado e explorado didaticamente.
- **Custo 24/7 (~US$ 48/mês)** → mitigação: `terraform destroy` fora dos períodos de uso.
- **Auto-upgrade automático do Kubernetes pela DO** → pode causar restart de nodes durante aula; mitigação futura: configurar janela de manutenção fora do horário das aulas.
- **Token DO no Terraform** → o token de API da DigitalOcean não deve ser commitado; usar variável de ambiente `DIGITALOCEAN_TOKEN` ou arquivo `.tfvars` fora do versionamento.

## Migration Plan

1. Criar módulo Terraform em `terraform/modules/doks/` (VPC + cluster DOKS) e ambiente em `terraform/envs/dev/`
2. Executar `terraform init` e `terraform apply` em `terraform/envs/dev/` para provisionar VPC + cluster
3. Configurar `kubectl` com as credenciais do cluster (output do Terraform)
4. Remover PVC e volumeMount do Postgres em `k8s/kube-news.yml`
5. Aplicar o manifest no cluster: `kubectl apply -f k8s/kube-news.yml`
6. Aguardar provisionamento do Load Balancer e obter IP externo via `kubectl get svc -n kube-news`
7. Ao fim da aula: `terraform destroy` (cluster, VPC e LB destruídos)

**Rollback:** `terraform destroy` desfaz toda a infraestrutura. O manifest `k8s/kube-news.yml` pode ser revertido via git se necessário.

## Open Questions

- Autenticação Docker Hub: não configurada agora; adicionar `imagePullSecrets` se rate limit for atingido em produção das aulas.
