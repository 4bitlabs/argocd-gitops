# Argo Rollouts - Guia de Uso

Este documento descreve como usar o Argo Rollouts no projeto SAGA, incluindo as estratégias BlueGreen (dev) e Canary (prod).

## Instalação

O Argo Rollouts é instalado automaticamente via ArgoCD através do Application `argorollouts` localizado em `argocd-gitops/infrastructure/argorollouts/application.yaml`.

Para instalar manualmente, execute:

```bash
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

## Estratégias de Deployment

### BlueGreen (Ambiente Dev)

O ambiente de desenvolvimento utiliza a estratégia **BlueGreen** com **auto-promoção desabilitada**, permitindo testes manuais antes de promover a nova versão.

#### Como funciona:

1. **Deployment inicial**: A versão atual (azul) está rodando e recebendo tráfego
2. **Novo deployment**: Uma nova versão (verde) é criada em paralelo
3. **Teste manual**: A equipe pode testar a versão verde através do serviço `preview`
4. **Promoção manual**: Quando aprovada, a versão verde é promovida e recebe todo o tráfego
5. **Limpeza**: A versão antiga (azul) é removida após um delay configurado

#### Serviços criados:

- `saga-dev-active`: Serviço que aponta para a versão ativa (recebendo tráfego)
- `saga-dev-preview`: Serviço que aponta para a versão preview (em teste)

#### Promoção Manual

Para promover manualmente a versão preview para produção:

**Opção 1: Usando kubectl argo rollouts (Recomendado)**

⚠️ **Importante**: Antes de usar os comandos `kubectl argo rollouts`, você precisa instalar a CLI do Argo Rollouts. Veja a seção [Instalação da CLI](#instalação-da-cli) abaixo.

```bash
# Verificar o status do rollout
kubectl argo rollouts get rollout saga-dev-app -n saga-dev

# Promover a versão preview
kubectl argo rollouts promote saga-dev-app -n saga-dev
```

**Opção 2: Usando kubectl patch (Alternativa sem CLI)**

Se você não tiver a CLI instalada, pode usar `kubectl patch` diretamente:

```bash
# Verificar o rollout
kubectl get rollout saga-dev-app -n saga-dev -o yaml

# Promover manualmente editando o rollout
kubectl patch rollout saga-dev-app -n saga-dev --type merge -p '{"status":{"currentPodHash":"<hash-da-versao-preview>"}}'
```

## Instalação da CLI

A CLI do Argo Rollouts é necessária para usar comandos como `kubectl argo rollouts`. Siga as instruções abaixo para sua plataforma:

### Windows (PowerShell)

**Método 1: Download Manual (Recomendado se houver problemas de conexão)**

1. Acesse: https://github.com/argoproj/argo-rollouts/releases/latest
2. Baixe o arquivo `kubectl-argo-rollouts-windows-amd64.exe`
3. Renomeie para `kubectl-argo-rollouts.exe`
4. Mova para um diretório no seu PATH (ex: `C:\Windows\System32` ou `$env:USERPROFILE\AppData\Local\Microsoft\WindowsApps\`)

**Método 2: PowerShell (Download Automático)**

```powershell
# Baixar a CLI
$ProgressPreference = 'SilentlyContinue'
Invoke-WebRequest -Uri "https://github.com/argoproj/argo-rollouts/releases/download/v1.7.0/kubectl-argo-rollouts-windows-amd64.exe" -OutFile "kubectl-argo-rollouts.exe"

# Mover para PATH (opcional, mas recomendado)
# Opção A: Para o usuário atual
Move-Item kubectl-argo-rollouts.exe $env:USERPROFILE\AppData\Local\Microsoft\WindowsApps\kubectl-argo-rollouts.exe

# Opção B: Para o sistema (requer permissões de administrador)
# Move-Item kubectl-argo-rollouts.exe C:\Windows\System32\kubectl-argo-rollouts.exe
```

**Verificar Instalação:**

```powershell
# Verificar se está no PATH
kubectl argo rollouts version

# Se não estiver no PATH, use o caminho completo
.\kubectl-argo-rollouts.exe version
```

### Linux/Mac

```bash
# Linux
curl -LO https://github.com/argoproj/argo-rollouts/releases/download/v1.7.0/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

# Mac
curl -LO https://github.com/argoproj/argo-rollouts/releases/download/v1.7.0/kubectl-argo-rollouts-darwin-amd64
chmod +x kubectl-argo-rollouts-darwin-amd64
sudo mv kubectl-argo-rollouts-darwin-amd64 /usr/local/bin/kubectl-argo-rollouts
```

**Verificar Instalação:**

```bash
kubectl argo rollouts version
```

**Opção 3: Via ArgoCD UI**

1. Acesse o ArgoCD
2. Navegue até a aplicação `saga-dev`
3. No recurso `Rollout`, você verá opções para promover ou abortar

#### Abortar um Deployment

Se encontrar problemas na versão preview e quiser abortar:

```bash
kubectl argo rollouts abort saga-dev-app -n saga-dev
```

### Canary (Ambiente Prod)

O ambiente de produção utiliza a estratégia **Canary** com incremento gradual de tráfego.

#### Como funciona:

1. **Deployment inicial**: A versão atual está recebendo 100% do tráfego
2. **Canary inicial**: Uma nova versão é criada e recebe 10% do tráfego
3. **Incremento gradual**: O tráfego é aumentado gradualmente (10% → 25% → 50% → 75% → 100%)
4. **Pausas automáticas**: O rollout pausa em cada etapa para análise
5. **Promoção ou rollback**: Baseado em métricas ou aprovação manual

#### Ajuste Manual de Tráfego

Durante a apresentação, você pode ajustar o tráfego manualmente:

**Opção 1: Usando kubectl argo rollouts**

```bash
# Verificar status atual
kubectl argo rollouts get rollout saga-prod-app -n saga-prod

# Ajustar peso do tráfego (exemplo: 30%)
kubectl argo rollouts set weight saga-prod-app -n saga-prod 30

# Continuar para o próximo passo
kubectl argo rollouts promote saga-prod-app -n saga-prod

# Pausar o rollout
kubectl argo rollouts pause saga-prod-app -n saga-prod

# Retomar o rollout
kubectl argo rollouts resume saga-prod-app -n saga-prod
```

**Opção 2: Editar o Rollout diretamente**

```bash
# Editar o rollout
kubectl edit rollout saga-prod-app -n saga-prod

# Modificar o campo spec.strategy.canary.steps para ajustar os pesos
```

**Opção 3: Usando kubectl patch**

```bash
# Ajustar para 30% de tráfego
kubectl patch rollout saga-prod-app -n saga-prod --type merge -p '{"spec":{"strategy":{"canary":{"steps":[{"setWeight":30}]}}}}'
```

#### Exemplo de Demonstração

Para demonstrar o ajuste de tráfego durante a apresentação:

```bash
# 1. Iniciar um novo deployment
# (Atualize a tag da imagem no values-prod.yaml e faça sync no ArgoCD)

# 2. Verificar que o rollout está pausado em 10%
kubectl argo rollouts get rollout saga-prod-app -n saga-prod

# 3. Ajustar manualmente para 20%
kubectl argo rollouts set weight saga-prod-app -n saga-prod 20

# 4. Continuar para 50%
kubectl argo rollouts set weight saga-prod-app -n saga-prod 50

# 5. Promover para 100%
kubectl argo rollouts promote saga-prod-app -n saga-prod
```

## Comandos Úteis

### Verificar Status

```bash
# Status detalhado
kubectl argo rollouts get rollout <nome-do-rollout> -n <namespace>

# Status em formato YAML
kubectl get rollout <nome-do-rollout> -n <namespace> -o yaml

# Histórico de revisões
kubectl argo rollouts history <nome-do-rollout> -n <namespace>
```

### Rollback

```bash
# Rollback para uma revisão anterior
kubectl argo rollouts undo <nome-do-rollout> -n <namespace>

# Rollback para uma revisão específica
kubectl argo rollouts undo <nome-do-rollout> -n <namespace> --to-revision=<numero>
```

### Abortar Deployment

```bash
# Abortar o deployment atual
kubectl argo rollouts abort <nome-do-rollout> -n <namespace>
```

### Restart

```bash
# Reiniciar o rollout
kubectl argo rollouts restart <nome-do-rollout> -n <namespace>
```

## Integração com ArgoCD

O ArgoCD gerencia os Rollouts da mesma forma que gerencia Deployments:

- **Sincronização automática**: O ArgoCD sincroniza mudanças no Git
- **Visualização**: O status do Rollout aparece no dashboard do ArgoCD
- **Promoção via UI**: É possível promover/abortar via interface do ArgoCD
- **Sync Waves**: Os Rollouts são deployados após as migrações (sync-wave: "2")

## Troubleshooting

### Rollout não está progredindo

```bash
# Verificar eventos
kubectl describe rollout <nome-do-rollout> -n <namespace>

# Verificar pods
kubectl get pods -n <namespace> -l app.kubernetes.io/name=saga-api

# Verificar serviços
kubectl get svc -n <namespace>
```

### Problemas com serviços BlueGreen

Certifique-se de que os serviços `-active` e `-preview` foram criados:

```bash
kubectl get svc -n saga-dev | grep saga-dev
```

### Problemas com tráfego Canary

Verifique se o Rollout está pausado e qual o peso atual:

```bash
kubectl argo rollouts get rollout saga-prod-app -n saga-prod
```

## Referências

- [Documentação Oficial do Argo Rollouts](https://argoproj.github.io/argo-rollouts/)
- [BlueGreen Strategy](https://argoproj.github.io/argo-rollouts/features/bluegreen/)
- [Canary Strategy](https://argoproj.github.io/argo-rollouts/features/canary/)

