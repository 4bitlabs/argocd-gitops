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
PREVIEW_HASH=$(kubectl get rollout saga-dev-saga-chart-app -n saga-dev -o jsonpath='{.status.blueGreen.previewSelector}')

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
# Defina spec.paused: true
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

# Ver peso atual do tráfego canary
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.canary.weights}'

# Ver se está pausado
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.spec.paused}'
```

#### Ajuste Manual de Tráfego (Canary)

Durante a apresentação, você pode ajustar o tráfego manualmente:

**Método 1: Usando kubectl patch (Recomendado)**

```bash
# Verificar status atual
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o yaml | grep -A 5 "canary:"

# Ajustar peso do tráfego para 30%
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"strategy":{"canary":{"steps":[{"setWeight":30}]}}}}'

# Continuar para o próximo passo (remover pause)
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"paused":false}}'

# Pausar o rollout manualmente
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"paused":true}}'
```

**Método 2: Usando kubectl edit**

```bash
# Editar o rollout
kubectl edit rollout saga-prod-saga-chart-app -n saga-prod

# No editor, você pode:
# - Modificar spec.strategy.canary.steps para ajustar os pesos
# - Alterar spec.paused para pausar/retomar
# - Modificar spec.replicas se necessário
# Salve e feche o editor
```

**Método 3: Promover para 100% (completar rollout)**

```bash
# Promover para 100% removendo todos os steps e pausas
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"strategy":{"canary":{"steps":[]}},"paused":false}}'
```

#### Exemplo de Demonstração

Para demonstrar o ajuste de tráfego durante a apresentação:

```bash
# 1. Iniciar um novo deployment
# (Atualize a tag da imagem no values-prod.yaml e faça sync no ArgoCD)

# 2. Verificar que o rollout está pausado em 10%
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.canary.weights}'

# 3. Ajustar manualmente para 20%
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"strategy":{"canary":{"steps":[{"setWeight":20}]}},"paused":false}}'

# 4. Aguardar alguns segundos e verificar
kubectl get rollout saga-prod-saga-chart-app -n saga-prod

# 5. Continuar para 50%
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"strategy":{"canary":{"steps":[{"setWeight":50}]}},"paused":false}}'

# 6. Promover para 100% (completar)
kubectl patch rollout saga-prod-saga-chart-app -n saga-prod --type merge -p '{"spec":{"strategy":{"canary":{"steps":[]}},"paused":false}}'
```

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
# Pausar rollout
kubectl patch rollout <nome-do-rollout> -n <namespace> --type merge -p '{"spec":{"paused":true}}'

# Retomar rollout
kubectl patch rollout <nome-do-rollout> -n <namespace> --type merge -p '{"spec":{"paused":false}}'

# Verificar se está pausado
kubectl get rollout <nome-do-rollout> -n <namespace> -o jsonpath='{.spec.paused}'
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

# Ver peso atual
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.status.canary.weights}'

# Ver se está pausado
kubectl get rollout saga-prod-saga-chart-app -n saga-prod -o jsonpath='{.spec.paused}'
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
