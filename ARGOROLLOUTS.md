# Argo Rollouts - Guia de Uso

Este documento descreve como usar o Argo Rollouts no projeto SAGA, incluindo as estratégias BlueGreen (dev) e Canary (prod).

## Instalação

⚠️ **Importante**: O Argo Rollouts deve ser instalado **manualmente** no cluster Kubernetes antes de usar os Rollouts nas aplicações. Ele não é gerenciado pelo ArgoCD.

### Instalação Manual

Execute os seguintes comandos para instalar o Argo Rollouts:

```bash
# Criar namespace
kubectl create namespace argo-rollouts

# Instalar Argo Rollouts v1.7.0
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/download/v1.7.0/install.yaml

# Verificar instalação
kubectl get pods -n argo-rollouts
```

### Resolver Erro CreateContainerConfigError

Após a instalação, o pod do Argo Rollouts pode apresentar o erro `CreateContainerConfigError`. Isso ocorre devido a um conflito entre a configuração `runAsNonRoot: true` no Deployment e a imagem do Argo Rollouts que roda como root.

**Solução:**

```bash
# Aplicar patch para corrigir o problema
kubectl patch deployment argo-rollouts -n argo-rollouts --type json -p '[{"op": "remove", "path": "/spec/template/spec/securityContext/runAsNonRoot"}]'

# Aguardar alguns segundos e verificar
kubectl get pods -n argo-rollouts
```

O pod deve estar com status `Running` e `1/1 READY`.

### Verificar Instalação

```bash
# Verificar pods do Argo Rollouts
kubectl get pods -n argo-rollouts

# Verificar CRDs instalados
kubectl get crd | grep rollouts

# Verificar o controller está funcionando
kubectl get deployment argo-rollouts -n argo-rollouts
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

#### Verificar Status do Rollout

```bash
# Ver status básico
kubectl get rollout saga-dev-saga-chart-app -n saga-dev

# Ver detalhes completos
kubectl describe rollout saga-dev-saga-chart-app -n saga-dev

# Ver em formato YAML
kubectl get rollout saga-dev-saga-chart-app -n saga-dev -o yaml
```

#### Promover a Versão Preview (BlueGreen)

Para promover manualmente a versão preview para produção, você precisa atualizar o status do Rollout:

**Método 1: Usando kubectl patch (Recomendado)**

```bash
# 1. Obter o hash da versão preview (verde)
kubectl get rollout saga-dev-saga-chart-app -n saga-dev -o jsonpath='{.status.blueGreen.activeSelector}'

# 2. Obter o hash da versão preview
kubectl get rollout saga-dev-saga-chart-app -n saga-dev -o jsonpath='{.status.blueGreen.previewSelector}'

# 3. Promover trocando o activeSelector pelo previewSelector
# Primeiro, obtenha o previewSelector
$PREVIEW_HASH=$(kubectl get rollout saga-dev-saga-chart-app -n saga-dev -o jsonpath='{.status.blueGreen.previewSelector}')

# 4. Promover atualizando o status
kubectl patch rollout saga-dev-saga-chart-app -n saga-dev --type merge -p "{\"status\":{\"currentPodHash\":\"$PREVIEW_HASH\"}}"
```

**Método 2: Usando kubectl edit**

```bash
# Editar o rollout
kubectl edit rollout saga-dev-saga-chart-app -n saga-dev

# No editor, localize a seção status.blueGreen e altere:
# - activeSelector: deve receber o valor do previewSelector
# - previewSelector: pode ser removido ou deixado vazio
# Salve e feche o editor
```

**Método 3: Via ArgoCD UI**

1. Acesse o ArgoCD
2. Navegue até a aplicação `saga-dev`
3. No recurso `Rollout`, você verá opções para promover ou abortar
4. Clique em "Promote" para promover a versão preview

#### Abortar um Deployment

Se encontrar problemas na versão preview e quiser abortar:

```bash
# Abortar o rollout atual
kubectl patch rollout saga-dev-saga-chart-app -n saga-dev --type merge -p '{"spec":{"paused":true}}'

# Ou editar diretamente
kubectl edit rollout saga-dev-saga-chart-app -n saga-dev

# No editor, adicione dentro de spec:
# spec:
#   paused: true  # ← Adicione esta linha (não existe por padrão)
```

Para reverter completamente, você pode fazer rollback:

```bash
# Ver histórico de revisões
kubectl rollout history rollout saga-dev-saga-chart-app -n saga-dev

# Fazer rollback para a revisão anterior
kubectl rollout undo rollout saga-dev-saga-chart-app -n saga-dev
```

### Canary (Ambiente Prod)

O ambiente de produção utiliza a estratégia **Canary** com incremento gradual de tráfego.

#### Como funciona:

1. **Deployment inicial**: A versão atual está recebendo 100% do tráfego
2. **Canary inicial**: Uma nova versão é criada e recebe 10% do tráfego
3. **Incremento gradual**: O tráfego é aumentado gradualmente (10% → 25% → 50% → 75% → 100%)
4. **Pausas automáticas**: O rollout pausa em cada etapa para análise
5. **Promoção ou rollback**: Baseado em métricas ou aprovação manual

#### Verificar Status do Rollout

```bash
# Ver status básico
kubectl get rollout saga-prod-saga-chart-app -n saga-prod

# Ver detalhes completos
kubectl describe rollout saga-prod-saga-chart-app -n saga-prod

# Ver step atual do canary (índice do passo atual, começa em 0)
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.currentStepIndex}'

# Ver se está pausado (retorna true/false)
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.controllerPause}'

# Ver fase do rollout (Paused, Progressing, Degraded, Healthy)
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.phase}'

# Ver condições de pausa (razão da pausa)
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.pauseConditions[*].reason}'

# Ver peso atual do canary (verificar o setWeight do step atual)
# PowerShell:
$stepIndex = kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.currentStepIndex}'
Write-Host "Step atual: $stepIndex"
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath="{.spec.strategy.canary.steps[$stepIndex].setWeight}"

# Bash/Linux:
# STEP_INDEX=$(kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.currentStepIndex}')
# echo "Step atual: $STEP_INDEX"
# kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath="{.spec.strategy.canary.steps[$STEP_INDEX].setWeight}"

# Ver todos os steps configurados (mostra todos os pesos definidos)
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.spec.strategy.canary.steps[*].setWeight}'

# Ver informações resumidas do status
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o custom-columns=STEP:.status.currentStepIndex,PHASE:.status.phase,PAUSED:.status.controllerPause
```

#### Ajuste Manual de Tráfego (Canary)

Durante a apresentação, você pode ajustar o tráfego manualmente. 

⚠️ **Importante**: 
- Os steps do Canary são uma lista sequencial: `[setWeight: 10, pause, setWeight: 25, pause, ...]`
- Quando você faz `patch` com novos steps, você **substitui TODA a lista**
- Para ajustar apenas o step atual, use `kubectl edit` e modifique apenas o step específico
- O `currentStepIndex` no status indica qual step está ativo (0 = primeiro step)

#### Como Continuar o Rollout Quando Está Pausado

Quando o rollout está pausado em um step de pause, o campo `spec.paused` **não existe** por padrão. Para continuar:

**Opção 1: Usando kubectl patch (Rápido)**
```bash
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"paused":false}}'
```

**Opção 2: Usando kubectl edit (Mais controle)**
```bash
kubectl edit rollout saga-prod-saga-chart-app -n saga-prod
```

No editor, adicione `paused: false` dentro da seção `spec:`:

```yaml
spec:
  replicas: 10
  paused: false  # ← Adicione esta linha (não existe por padrão)
  selector:
    matchLabels:
      ...
  strategy:
    canary:
      steps:
        ...
```

**Como funciona:**
- Quando você adiciona `spec.paused: false`, o controller do Argo Rollouts:
  1. Remove automaticamente as `pauseConditions` do status
  2. Continua para o próximo step na sequência
  3. Aplica o próximo `setWeight` ou `pause` conforme definido nos steps

**Método 1: Usando kubectl patch (Recomendado)**

```bash
# 1. Verificar status atual
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.currentStepIndex}'
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.controllerPause}'

# 2. Ajustar peso do tráfego para um valor específico (exemplo: 30%)
# IMPORTANTE: Isso substitui TODOS os steps. Você precisa incluir os steps seguintes.
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"strategy":{"canary":{"steps":[{"setWeight":30},{"pause":{}},{"setWeight":50},{"pause":{}},{"setWeight":75},{"pause":{}},{"setWeight":100}]}},"paused":false}}'

# 3. Continuar para o próximo passo (remover pause)
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"paused":false}}'

# 4. Pausar o rollout manualmente (força pausa mesmo que não esteja em um step de pausa)
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"paused":true}}'
```

**Método 2: Usando kubectl edit (Mais flexível)**

```bash
# Editar o rollout
kubectl edit rollout saga-prod-saga-chart-app -n saga-prod
```

**Para continuar para o próximo step quando o rollout está pausado:**

1. No editor, localize a seção `spec:` (não `status:`)
2. Adicione a linha `paused: false` dentro de `spec:`:

```yaml
spec:
  replicas: 10
  paused: false  # ← Adicione esta linha para continuar
  selector:
    matchLabels:
      ...
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {}
        ...
```

3. Salve e feche o editor

**Nota importante**: 
- O campo `spec.paused` não existe por padrão no Rollout
- Quando você adiciona `spec.paused: false`, o controller do Argo Rollouts remove automaticamente as `pauseConditions` do status e continua para o próximo step
- Para pausar manualmente, adicione `spec.paused: true`

**Outras modificações possíveis no editor:**

```yaml
spec:
  # Modificar número de réplicas
  replicas: 15
  
  # Pausar/retomar rollout
  paused: false  # ou true para pausar
  
  # Modificar steps do canary
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: {}
        - setWeight: 40
        - pause: {}
        - setWeight: 60
        - pause: {}
        - setWeight: 80
        - pause: {}
        - setWeight: 100
```

**Método 3: Promover para 100% (completar rollout)**

```bash
# Promover para 100% removendo todos os steps e pausas
# Isso faz o rollout completar imediatamente
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"strategy":{"canary":{"steps":[]}},"paused":false}}'
```

**Método 4: Ajustar apenas o step atual (Mais preciso)**

Para ajustar apenas o step atual sem substituir todos os steps, você precisa editar manualmente:

```bash
# 1. Obter o índice do step atual
$stepIndex = kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.currentStepIndex}'
Write-Host "Step atual: $stepIndex"

# 2. Editar o rollout e modificar apenas o step no índice atual
kubectl edit rollout saga-prod-saga-chart-app -n saga-prod

# No editor:
# - Localize spec.strategy.canary.steps[$stepIndex] e ajuste o setWeight
# - Mantenha os outros steps inalterados
# - Se estiver pausado, adicione spec.paused: false para continuar
```

#### Exemplo de Demonstração

Para demonstrar o ajuste de tráfego durante a apresentação:

```bash
# 1. Iniciar um novo deployment
# (Atualize a tag da imagem no values-prod.yaml e faça sync no ArgoCD)

# 2. Verificar status inicial
kubectl get rollout saga-prod-saga-chart-app -n saga-prod
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='Step: {.status.currentStepIndex}, Phase: {.status.phase}, Paused: {.status.controllerPause}'

# 3. Verificar o peso atual do step
$stepIndex = kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.currentStepIndex}'
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath="{.spec.strategy.canary.steps[$stepIndex].setWeight}"

# 4. Ajustar manualmente para 20% (substituindo os steps seguintes)
# IMPORTANTE: Inclua os steps seguintes para manter a progressão
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"strategy":{"canary":{"steps":[{"setWeight":20},{"pause":{}},{"setWeight":40},{"pause":{}},{"setWeight":60},{"pause":{}},{"setWeight":80},{"pause":{}},{"setWeight":100}]}},"paused":false}}'

# 5. Aguardar alguns segundos e verificar
kubectl get rollout saga-prod-saga-chart-app -n saga-prod

# 6. Se quiser pausar novamente para análise
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"paused":true}}'

# 7. Continuar (retomar)
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"paused":false}}'

# 8. Promover para 100% imediatamente (completar rollout)
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"strategy":{"canary":{"steps":[]}},"paused":false}}'
```

**Nota**: Quando você usa `kubectl patch` para ajustar os steps, você está substituindo TODA a lista de steps. Por isso é importante incluir os steps seguintes na sequência desejada. Para ajustes mais precisos, use `kubectl edit` para modificar apenas o step atual.

## Comandos Úteis

### Verificar Status

```bash
# Status básico
kubectl get rollout <nome-do-rollout> -n <namespace>

# Status detalhado
kubectl describe rollout <nome-do-rollout> -n <namespace>

# Status em formato YAML
kubectl get rollout <nome-do-rollout> -n <namespace> -o yaml

# Status em formato JSON (útil para scripts)
kubectl get rollout <nome-do-rollout> -n <namespace> -o json

# Ver pods relacionados ao rollout
kubectl get pods -n <namespace> -l app.kubernetes.io/name=saga-api

# Ver serviços relacionados
kubectl get svc -n <namespace> | grep saga
```

### Rollback

```bash
# Ver histórico de revisões
kubectl rollout history rollout <nome-do-rollout> -n <namespace>

# Rollback para a revisão anterior
kubectl rollout undo rollout <nome-do-rollout> -n <namespace>

# Rollback para uma revisão específica
kubectl rollout undo rollout <nome-do-rollout> -n <namespace> --to-revision=<numero>
```

### Pausar/Retomar Rollout

```bash
# Pausar rollout manualmente
kubectl patch rollout <nome-do-rollout> -n <namespace> --type merge -p '{"spec":{"paused":true}}'

# Retomar rollout (continuar para próximo step)
kubectl patch rollout <nome-do-rollout> -n <namespace> --type merge -p '{"spec":{"paused":false}}'

# OU usando kubectl edit:
# kubectl edit rollout <nome-do-rollout> -n <namespace>
# No editor, adicione dentro de spec:
# spec:
#   paused: false  # ← Adicione esta linha (não existe por padrão)

# Verificar se está pausado (retorna true/false)
kubectl get rollout <nome-do-rollout> -n <namespace> -o jsonpath='{.status.controllerPause}'

# Ver fase do rollout (Paused, Progressing, Degraded, Healthy)
kubectl get rollout <nome-do-rollout> -n <namespace> -o jsonpath='{.status.phase}'

# Ver condições de pausa (razão da pausa)
kubectl get rollout <nome-do-rollout> -n <namespace> -o jsonpath='{.status.pauseConditions[*].reason}'
```

### Restart

```bash
# Reiniciar o rollout (força um novo deployment)
kubectl rollout restart rollout <nome-do-rollout> -n <namespace>
```

## Integração com ArgoCD

O ArgoCD gerencia os Rollouts da mesma forma que gerencia Deployments:

- **Sincronização automática**: O ArgoCD sincroniza mudanças no Git
- **Visualização**: O status do Rollout aparece no dashboard do ArgoCD
- **Promoção via UI**: É possível promover/abortar via interface do ArgoCD
- **Sync Waves**: Os Rollouts são deployados após as migrações (sync-wave: "2")

**Nota**: Embora o ArgoCD gerencie os Rollouts das aplicações, o próprio controlador do Argo Rollouts deve estar instalado manualmente no cluster.

### ⚠️ Problema: Promoção Automática com BlueGreen

**Sintoma**: Mesmo com `autoPromotionEnabled: false`, o ArgoCD promove automaticamente a versão preview.

**Causa**: O `Replace=true` no `syncOptions` do ArgoCD faz com que o recurso seja substituído completamente durante o sync, fazendo o Argo Rollouts perder o estado do BlueGreen e promover automaticamente.

**Solução**: Foi adicionada a anotação `argocd.argoproj.io/sync-options: Replace=false,ServerSideApply=true` diretamente no recurso Rollout. Isso permite que apenas o Rollout seja sincronizado com essas opções específicas (preservando o status), enquanto os outros recursos (Service, ConfigMap, etc.) continuam usando `Replace=true` para garantir atualizações completas.

**Como funciona**:

- **Anotação no Rollout**: `argocd.argoproj.io/sync-options: Replace=false,ServerSideApply=true` controla apenas o Rollout
  - `Replace=false`: Evita substituir completamente o recurso, preservando o status
  - `ServerSideApply=true`: Usa Server-Side Apply do Kubernetes para melhor preservação de campos gerenciados e status
- **SyncOptions da aplicação**: Mantém apenas `Replace=true` para outros recursos (Service, ConfigMap, Job, etc.)
  - `ServerSideApply=true` foi removido do syncOptions da aplicação para evitar possíveis conflitos
  - Apenas o Rollout usa ServerSideApply via anotação específica
- **Resultado**: O Rollout preserva seu status (activeSelector/previewSelector) usando Server-Side Apply sem Replace, enquanto outros recursos são atualizados normalmente com Replace

**Verificar configuração**:

```bash
# Verificar a anotação no Rollout
kubectl get rollout saga-dev-saga-chart-app -n saga-dev -o jsonpath='{.metadata.annotations.argocd\.argoproj\.io/sync-options}'

# Deve retornar: Replace=false,ServerSideApply=true
```

**Nota**: Esta configuração já está aplicada no template do Rollout. Após o próximo deploy, o comportamento correto será aplicado automaticamente.

## Troubleshooting

### Rollout não está progredindo

```bash
# Verificar eventos
kubectl describe rollout <nome-do-rollout> -n <namespace>

# Verificar pods
kubectl get pods -n <namespace> -l app.kubernetes.io/name=saga-api

# Verificar serviços
kubectl get svc -n <namespace>

# Verificar logs do controller do Argo Rollouts
kubectl logs -n argo-rollouts deployment/argo-rollouts
```

### Problemas com serviços BlueGreen

Certifique-se de que os serviços `-active` e `-preview` foram criados:

```bash
# Verificar serviços
kubectl get svc -n saga-dev | grep saga-dev

# Verificar qual serviço está apontando para qual versão
kubectl get svc saga-dev-active -n saga-dev -o yaml
kubectl get svc saga-dev-preview -n saga-dev -o yaml
```

### Problemas com tráfego Canary

Verifique se o Rollout está pausado e qual o peso atual:

```bash
# Ver status completo
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o yaml

# Ver step atual do canary
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.currentStepIndex}'

# Ver se está pausado (controllerPause)
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.controllerPause}'

# Ver fase do rollout
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.phase}'

# Ver condições de pausa
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.pauseConditions[*].reason}'

# Ver peso atual do step (PowerShell)
$stepIndex = kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.currentStepIndex}'
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath="{.spec.strategy.canary.steps[$stepIndex].setWeight}"

# Ver todos os steps configurados
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.spec.strategy.canary.steps[*].setWeight}'
```

### Controller do Argo Rollouts não está funcionando

```bash
# Verificar se o controller está rodando
kubectl get pods -n argo-rollouts

# Ver logs do controller
kubectl logs -n argo-rollouts deployment/argo-rollouts

# Verificar eventos do namespace
kubectl get events -n argo-rollouts --sort-by='.lastTimestamp'
```

## Referências

- [Documentação Oficial do Argo Rollouts](https://argoproj.github.io/argo-rollouts/)
- [BlueGreen Strategy](https://argoproj.github.io/argo-rollouts/features/bluegreen/)
- [Canary Strategy](https://argoproj.github.io/argo-rollouts/features/canary/)
