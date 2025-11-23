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
│           ├── argorollouts-application.yaml
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

### 2. Argo Rollouts

O Argo Rollouts é instalado para fornecer estratégias avançadas de deployment (BlueGreen e Canary). O Application é gerenciado através do Helm chart em `helm/argocd-applications-chart/templates/argorollouts-application.yaml`.

- **Namespace**: `argo-rollouts`
- **Repositório**: `https://github.com/argoproj/argo-rollouts.git`
- **Versão**: `v1.7.0`
- **Sync Wave**: `0`

#### 2.1. Instalação Automática via ArgoCD

O Argo Rollouts será instalado automaticamente quando o ArgoCD sincronizar a aplicação `argorollouts`.

#### 2.2. Instalação Manual

Se preferir instalar manualmente:

```bash
# Criar namespace
kubectl create namespace argo-rollouts

# Instalar Argo Rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Verificar instalação
kubectl get pods -n argo-rollouts
```

#### 2.3. Instalar CLI do Argo Rollouts

**Windows (PowerShell):**
```powershell
# Baixar a CLI
Invoke-WebRequest -Uri https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-windows-amd64.exe -OutFile kubectl-argo-rollouts.exe

# Mover para PATH (opcional)
Move-Item kubectl-argo-rollouts.exe $env:USERPROFILE\AppData\Local\Microsoft\WindowsApps\kubectl-argo-rollouts.exe
```

**Linux/Mac:**
```bash
# Linux
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

# Mac
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-darwin-amd64
chmod +x kubectl-argo-rollouts-darwin-amd64
sudo mv kubectl-argo-rollouts-darwin-amd64 /usr/local/bin/kubectl-argo-rollouts
```

#### 2.4. Verificar Instalação do Argo Rollouts

```bash
# Verificar pods do Argo Rollouts
kubectl get pods -n argo-rollouts

# Verificar CRDs instalados
kubectl get crd | grep rollouts

# Testar CLI
kubectl argo rollouts version
```

#### 2.5. Resolver Erro CreateContainerConfigError

**Problema:**

Após a instalação, o pod do Argo Rollouts pode apresentar o erro `CreateContainerConfigError`. Isso ocorre devido a um conflito entre a configuração `runAsNonRoot: true` no Deployment e a imagem do Argo Rollouts que roda como root.

**Verificar o Erro:**

```bash
# Verificar o status dos pods
kubectl get pods -n argo-rollouts

# Se o pod estiver com status CreateContainerConfigError, verificar detalhes
kubectl describe pod <nome-do-pod> -n argo-rollouts
```

**Solução:**

Execute o seguinte comando para remover a configuração `runAsNonRoot` do Deployment:

```bash
kubectl patch deployment argo-rollouts -n argo-rollouts --type json -p '[{"op": "remove", "path": "/spec/template/spec/securityContext/runAsNonRoot"}]'
```

Após executar o comando:

1. O Deployment será atualizado automaticamente
2. Um novo ReplicaSet será criado
3. O pod antigo será substituído por um novo pod funcionando
4. Aguarde alguns segundos e verifique novamente:

```bash
kubectl get pods -n argo-rollouts
```

O pod deve estar com status `Running` e `1/1 READY`.

⚠️ **Atenção**: Se o ArgoCD sincronizar novamente a aplicação `argorollouts`, este patch pode ser sobrescrito. Se isso acontecer e o erro retornar, execute o comando de patch novamente.

### 3. Banco de Dados PostgreSQL (Infrastructure)

O banco de dados PostgreSQL é gerenciado como uma aplicação separada, garantindo maior independência e disponibilidade. Cada ambiente possui seu próprio cluster PostgreSQL.

#### 3.1. Banco de Dados - Desenvolvimento

- **Application**: `saga-postgres-dev`
- **Namespace**: `saga-dev`
- **Branch**: `develop`
- **Helm Chart**: `helm/postgres-chart`
- **Values File**: `values-dev.yaml`
- **Sync Wave**: `0`

#### 3.2. Banco de Dados - Produção

- **Application**: `saga-postgres-prod`
- **Namespace**: `saga-prod`
- **Branch**: `main`
- **Helm Chart**: `helm/postgres-chart`
- **Values File**: `values-prod.yaml`
- **Sync Wave**: `0`

### 4. Aplicação SAGA (Applications)

A aplicação principal SAGA, que consome o banco de dados PostgreSQL.

#### 4.1. Aplicação de Desenvolvimento (dev)

A instância de desenvolvimento da aplicação SAGA é gerenciada através do Helm chart em `helm/argocd-applications-chart/templates/saga-dev-application.yaml`.

- **Application**: `saga-dev`
- **Namespace**: `saga-dev`
- **Branch**: `develop`
- **Helm Chart**: `helm/saga-chart`
- **Values File**: `values-dev.yaml`
- **Sync Wave**: `1`

#### 4.2. Aplicação de Produção (prod)

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
  - Argo Rollouts (infrastructure)
  - PostgreSQL Cluster (banco de dados - infrastructure)
- **Wave 1**: 
  - Migration Job (executa migrações do banco)
  - Application Deployment (aplicação principal)

## Dependências

A ordem de deploy é garantida através de sync waves:

1. **Wave 0**: 
   - O operador CloudNativePG deve estar instalado primeiro
   - O Argo Rollouts é instalado para suportar estratégias de deployment avançadas
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
   argocd app get argorollouts
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
- [Argo Rollouts Documentation](https://argoproj.github.io/argo-rollouts/)
- [CloudNativePG Documentation](https://cloudnative-pg.io/)
- [Helm Documentation](https://helm.sh/docs/)

