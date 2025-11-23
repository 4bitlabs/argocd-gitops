# ArgoCD GitOps - SAGA Application

Este repositório contém as configurações GitOps para deploy da aplicação SAGA utilizando ArgoCD.

## Estrutura do Repositório

```
argocd-gitops/
├── infrastructure/
│   ├── cloudnative-pg/
│   │   └── application.yaml      # Instalação do operador CloudNativePG
│   └── postgres/
│       ├── dev/
│       │   └── application.yaml  # Banco de dados de desenvolvimento
│       └── prod/
│           └── application.yaml  # Banco de dados de produção
├── apps/
│   ├── dev/
│   │   └── application.yaml      # Aplicação de desenvolvimento
│   └── prod/
│       └── application.yaml      # Aplicação de produção
├── kustomization.yaml            # Kustomize para aplicar todas as applications
└── README.md
```

## Componentes

### 1. CloudNativePG Operator

O operador CloudNativePG é instalado para gerenciar clusters PostgreSQL no Kubernetes. O Application está localizado em `infrastructure/cloudnative-pg/application.yaml`.

- **Namespace**: `cnpg-system`
- **Chart**: `cloudnative-pg` do repositório oficial
- **Versão**: `0.21.1`
- **Sync Wave**: `0`

### 2. Banco de Dados PostgreSQL (Infrastructure)

O banco de dados PostgreSQL é gerenciado como uma aplicação separada, garantindo maior independência e disponibilidade. Cada ambiente possui seu próprio cluster PostgreSQL.

#### 2.1. Banco de Dados - Desenvolvimento

- **Application**: `saga-postgres-dev`
- **Namespace**: `saga-dev`
- **Branch**: `develop`
- **Helm Chart**: `helm/postgres-chart`
- **Values File**: `values-dev.yaml`
- **Sync Wave**: `0`

#### 2.2. Banco de Dados - Produção

- **Application**: `saga-postgres-prod`
- **Namespace**: `saga-prod`
- **Branch**: `main`
- **Helm Chart**: `helm/postgres-chart`
- **Values File**: `values-prod.yaml`
- **Sync Wave**: `0`

### 3. Aplicação SAGA (Applications)

A aplicação principal SAGA, que consome o banco de dados PostgreSQL.

#### 3.1. Aplicação de Desenvolvimento (dev)

A instância de desenvolvimento da aplicação SAGA está configurada em `apps/dev/application.yaml`.

- **Application**: `saga-dev`
- **Namespace**: `saga-dev`
- **Branch**: `develop`
- **Helm Chart**: `helm/saga-chart`
- **Values File**: `values-dev.yaml`
- **Sync Wave**: `1`

#### 3.2. Aplicação de Produção (prod)

A instância de produção da aplicação SAGA está configurada em `apps/prod/application.yaml`.

- **Application**: `saga-prod`
- **Namespace**: `saga-prod`
- **Branch**: `main`
- **Helm Chart**: `helm/saga-chart`
- **Values File**: `values-prod.yaml`
- **Sync Wave**: `1`

## Sync Waves

Os recursos são sincronizados em ondas (sync waves) para garantir a ordem correta de deploy:

- **Wave 0**: 
  - CloudNativePG Operator (infrastructure)
  - PostgreSQL Cluster (banco de dados - infrastructure)
- **Wave 1**: 
  - Migration Job (executa migrações do banco)
  - Application Deployment (aplicação principal)

## Dependências

A ordem de deploy é garantida através de sync waves:

1. **Wave 0**: 
   - O operador CloudNativePG deve estar instalado primeiro
   - Os clusters PostgreSQL são criados logo em seguida
   
2. **Wave 1**: 
   - As aplicações `saga-dev` e `saga-prod` dependem dos clusters PostgreSQL estarem prontos
   - O job de migração executa antes do deployment da aplicação (usando PreSync hook)

Esta ordem garante que o banco de dados esteja disponível antes da aplicação tentar se conectar.

## Configuração Inicial

### Pré-requisitos

1. ArgoCD instalado no cluster Kubernetes
2. Acesso ao repositório GitHub contendo o Helm chart
3. Permissões para criar namespaces no cluster

### Passos para Deploy

1. **Verificar Instalação do ArgoCD:**
   ```bash
   # Verificar se o ArgoCD está rodando
   kubectl get pods -n argocd

   # Obter senha inicial do admin
   argocd admin initial-password -n argocd
   ```
2. **Fazer port-forward para o serviço do argocd:**
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```
3. **Fazer login no argocd:**
   ```bash
   argocd login localhost:8080 
   ```
4. **Adicionar na lista do argocd os repositórios utilizados nas aplicações:**

   ```bash
   argocd repo add https://cloudnative-pg.github.io/charts --type helm --name cloudnative-pg
   ```

   ```bash
   argocd repo add https://github.com/4bitlabs/saga.git --type git
   ```

   OBS: Se o repositório Git fosse privado, seria necessário configurar credenciais no ArgoCD ao adicionar o repositório do projeto:

   ```bash
   argocd repo add https://github.com/4bitlabs/saga.git --type git --username SEU_USUARIO --password SEU_TOKEN
   ```

   Você pode verificar os repositórios adicionados com:

   ```bash
   argocd repo list
   ```

5. **Aplicar as configurações no ArgoCD**:
   ### você pode aplicar todas de uma vez fazendo uso do arquivo kustomization.yaml
   ```bash
   # Aplicar as aplicações de uma vez pelo kustomization.yaml
   kubectl apply -k .
   ```
   ### ou, você aplicar cada aplicação uma por uma

   ```bash
   # Aplicar o operador CloudNativePG
   kubectl apply -f infrastructure/cloudnative-pg/application.yaml
   
   # Aguardar o operador estar pronto (opcional, mas recomendado)
   kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=cloudnative-pg -n cnpg-system --timeout=300s
   
   # Aplicar os bancos de dados PostgreSQL
   kubectl apply -f infrastructure/postgres/dev/application.yaml
   kubectl apply -f infrastructure/postgres/prod/application.yaml
   
   # Aplicar as aplicações
   kubectl apply -f apps/dev/application.yaml
   kubectl apply -f apps/prod/application.yaml
   ```

6. **Verificar o status via ArgoCD CLI**:
   ```bash
   argocd app list
   
   # Verificar infrastructure
   argocd app get cloudnative-pg-operator
   argocd app get saga-postgres-dev
   argocd app get saga-postgres-prod
   
   # Verificar applications
   argocd app get saga-dev
   argocd app get saga-prod
   ```

   Mas, como você está com o servidor aberto, você também pode acessar a UI do argocd e acompanhar tudo por lá

## Arquitetura

A arquitetura do projeto segue o princípio de separação de responsabilidades:

- **Infrastructure**: Componentes de infraestrutura (operador e banco de dados) que devem estar sempre disponíveis
- **Applications**: Aplicações que dependem da infraestrutura

O banco de dados PostgreSQL foi separado em um Helm chart próprio (`postgres-chart`) para:
- Garantir maior independência e disponibilidade
- Facilitar manutenção e atualizações do banco
- Permitir que o banco permaneça disponível mesmo durante atualizações da aplicação

## Referências

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [CloudNativePG Documentation](https://cloudnative-pg.io/)
- [Helm Documentation](https://helm.sh/docs/)

