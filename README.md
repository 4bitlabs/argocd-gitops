# ArgoCD GitOps - SAGA Application

Este repositório contém as configurações GitOps para deploy da aplicação SAGA utilizando ArgoCD.

## Estrutura do Repositório

```
argocd-gitops/
├── helm/
│   └── argocd-applications-chart/
│       ├── Chart.yaml            # Definição do Helm chart
│       ├── values.yaml           # Valores padrão do chart
│       └── templates/
│           ├── _helpers.tpl      # Helpers do Helm
│           ├── cloudnative-pg-application.yaml
│           ├── postgres-dev-application.yaml
│           ├── postgres-prod-application.yaml
│           ├── saga-dev-application.yaml
│           └── saga-prod-application.yaml
└── README.md
```

> **Nota**: Todas as Applications do ArgoCD são gerenciadas através do Helm chart `argocd-applications-chart`.

## Componentes

### 1. CloudNativePG Operator

O operador CloudNativePG é instalado para gerenciar clusters PostgreSQL no Kubernetes. O Application é gerenciado através do Helm chart em `helm/argocd-applications-chart/templates/cloudnative-pg-application.yaml`.

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

A instância de desenvolvimento da aplicação SAGA é gerenciada através do Helm chart em `helm/argocd-applications-chart/templates/saga-dev-application.yaml`.

- **Application**: `saga-dev`
- **Namespace**: `saga-dev`
- **Branch**: `develop`
- **Helm Chart**: `helm/saga-chart`
- **Values File**: `values-dev.yaml`
- **Sync Wave**: `1`

#### 3.2. Aplicação de Produção (prod)

A instância de produção da aplicação SAGA é gerenciada através do Helm chart em `helm/argocd-applications-chart/templates/saga-prod-application.yaml`.

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

5. **Aplicar as configurações no ArgoCD usando Helm**:
   
   ```bash
   # Aplicar todas as Applications de uma vez usando o Helm chart
   helm install argocd-applications ./helm/argocd-applications-chart
   
   # Aguardar o operador estar pronto (opcional, mas recomendado)
   kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=cloudnative-pg -n cnpg-system --timeout=300s
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

## Gerenciamento com Helm

Este projeto utiliza Helm charts para gerenciar todas as Applications do ArgoCD de forma centralizada. O chart principal está localizado em `helm/argocd-applications-chart/`.

### Comandos Úteis do Helm

```bash
# Instalar todas as Applications
helm install argocd-applications ./helm/argocd-applications-chart

# Atualizar as Applications
helm upgrade argocd-applications ./helm/argocd-applications-chart

# Verificar o status da instalação
helm status argocd-applications

# Ver os valores configurados
helm get values argocd-applications

# Desinstalar todas as Applications
helm uninstall argocd-applications

# Renderizar os templates sem aplicar (dry-run)
helm template argocd-applications ./helm/argocd-applications-chart
```

### Personalização de Valores

Você pode personalizar os valores do chart criando um arquivo `values-custom.yaml` e aplicando com:

```bash
helm install argocd-applications ./helm/argocd-applications-chart -f values-custom.yaml
```

Ou sobrescrever valores específicos na linha de comando:

```bash
helm install argocd-applications ./helm/argocd-applications-chart \
  --set cloudnativePg.source.targetRevision=0.22.0 \
  --set sagaDev.source.targetRevision=feature-branch
```

## Arquitetura

A arquitetura do projeto segue o princípio de separação de responsabilidades:

- **Infrastructure**: Componentes de infraestrutura (operador e banco de dados) que devem estar sempre disponíveis
- **Applications**: Aplicações que dependem da infraestrutura

O banco de dados PostgreSQL foi separado em um Helm chart próprio (`postgres-chart`) para:
- Garantir maior independência e disponibilidade
- Facilitar manutenção e atualizações do banco
- Permitir que o banco permaneça disponível mesmo durante atualizações da aplicação

As Applications do ArgoCD são gerenciadas através de um Helm chart centralizado (`argocd-applications-chart`) para:
- Facilitar o gerenciamento de todas as Applications em um único lugar
- Permitir versionamento e controle de mudanças
- Simplificar o deploy e atualizações

## Referências

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [CloudNativePG Documentation](https://cloudnative-pg.io/)
- [Helm Documentation](https://helm.sh/docs/)

