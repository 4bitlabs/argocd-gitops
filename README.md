# ArgoCD GitOps - SAGA Application

Este repositório contém as configurações GitOps para deploy da aplicação SAGA utilizando ArgoCD.

## Estrutura do Repositório

```
argocd-gitops/
├── infrastructure/
│   └── cloudnative-pg/
│       └── application.yaml      # Instalação do operador CloudNativePG
├── apps/
│   ├── dev/
│   │   └── application.yaml      # Aplicação de desenvolvimento
│   └── prod/
│       └── application.yaml      # Aplicação de produção
└── README.md
```

## Componentes

### 1. CloudNativePG Operator

O operador CloudNativePG é instalado para gerenciar clusters PostgreSQL no Kubernetes. O Application está localizado em `infrastructure/cloudnative-pg/application.yaml`.

- **Namespace**: `cnpg-system`
- **Chart**: `cloudnative-pg` do repositório oficial
- **Versão**: `v1.24.4`

### 2. Aplicação de Desenvolvimento (dev)

A instância de desenvolvimento da aplicação SAGA está configurada em `apps/dev/application.yaml`.

- **Namespace**: `saga-dev`
- **Branch**: `develop`
- **Helm Chart**: `helm/saga-chart`
- **Values File**: `values-dev.yaml`

### 3. Aplicação de Produção (prod)

A instância de produção da aplicação SAGA está configurada em `apps/prod/application.yaml`.

- **Namespace**: `saga-prod`
- **Branch**: `main`
- **Helm Chart**: `helm/saga-chart`
- **Values File**: `values-prod.yaml`

## Sync Waves

Os recursos são sincronizados em ondas (sync waves) para garantir a ordem correta de deploy:

- **Wave 0**: CloudNativePG Cluster (banco de dados)
- **Wave 1**: Migration Job (executa migrações do banco)
- **Wave 2**: Application Deployment (aplicação principal)

## Dependências

As aplicações `saga-dev` e `saga-prod` dependem do `cloudnative-pg-operator` estar instalado primeiro. Isso é garantido através de sync waves: o operador é instalado na wave "0" e as aplicações na wave "1".

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
   
   # Aplicar as aplicações
   kubectl apply -f apps/dev/application.yaml
   kubectl apply -f apps/prod/application.yaml
   ```

6. **Verificar o status via ArgoCD CLI**:
   ```bash
   argocd app list
   argocd app get saga-dev
   argocd app get saga-prod
   ```

   Mas, como você está com o servidor aberto, você também pode acessar a UI do argocd e acompanhar tudo por lá

## Referências

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [CloudNativePG Documentation](https://cloudnative-pg.io/)
- [Helm Documentation](https://helm.sh/docs/)

