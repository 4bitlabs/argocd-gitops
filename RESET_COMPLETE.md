# Reset Completo - Remover Tudo e Começar do Zero

Este guia mostra como remover completamente todas as aplicações e recursos para começar do zero.

## Passo 1: Remover as Applications do ArgoCD

```bash
# Remover as Applications (se tiver finalizers travados, remova primeiro)
kubectl patch application cloudnative-pg-operator -n argocd -p '{\"metadata\":{\"finalizers\":[]}}' --type merge
kubectl patch application saga-dev -n argocd -p '{\"metadata\":{\"finalizers\":[]}}' --type merge
kubectl patch application saga-prod -n argocd -p '{\"metadata\":{\"finalizers\":[]}}' --type merge

# Deletar as Applications
kubectl delete application cloudnative-pg-operator -n argocd
kubectl delete application saga-dev -n argocd
kubectl delete application saga-prod -n argocd

# Ou usar kustomize
kubectl delete -k ./argocd-gitops
```

## Passo 2: Remover Todos os Recursos Criados

```bash
# Deletar os namespaces (isso remove TUDO dentro deles)
kubectl delete namespace saga-dev --ignore-not-found=true
kubectl delete namespace saga-prod --ignore-not-found=true
kubectl delete namespace cnpg-system --ignore-not-found=true

# Aguardar os namespaces serem completamente removidos
kubectl wait --for=delete namespace/saga-dev --timeout=60s || true
kubectl wait --for=delete namespace/saga-prod --timeout=60s || true
kubectl wait --for=delete namespace/cnpg-system --timeout=60s || true
```

## Passo 3: Verificar Limpeza

```bash
# Verificar Applications restantes
kubectl get applications -n argocd

# Verificar namespaces
kubectl get namespaces | grep -E "saga-|cnpg"

# Verificar recursos restantes (não deveria haver nada)
kubectl get all --all-namespaces | grep -E "saga-|cnpg"
```

## Passo 4: Reaplicar Tudo do Zero

```bash
# 1. Verificar se os repositórios estão configurados
argocd repo list

# 2. Se necessário, adicionar repositórios
argocd repo add https://cloudnative-pg.github.io/charts --type helm --name cloudnative-pg
argocd repo add https://github.com/4bitlabs/saga.git --type git

# 3. Aplicar as Applications
kubectl apply -k ./argocd-gitops

# 4. Verificar status
argocd app list
```

## Script Completo (Automático)

Execute este script para fazer tudo automaticamente:

```bash
# Reset Completo
echo "=== Removendo Applications ==="
kubectl patch application cloudnative-pg-operator -n argocd -p '{"metadata":{"finalizers":[]}}' --type merge 2>/dev/null || true
kubectl patch application saga-dev -n argocd -p '{"metadata":{"finalizers":[]}}' --type merge 2>/dev/null || true
kubectl patch application saga-prod -n argocd -p '{"metadata":{"finalizers":[]}}' --type merge 2>/dev/null || true

kubectl delete application cloudnative-pg-operator -n argocd 2>/dev/null || true
kubectl delete application saga-dev -n argocd 2>/dev/null || true
kubectl delete application saga-prod -n argocd 2>/dev/null || true

echo "=== Removendo Namespaces ==="
kubectl delete namespace saga-dev --ignore-not-found=true
kubectl delete namespace saga-prod --ignore-not-found=true
kubectl delete namespace cnpg-system --ignore-not-found=true

echo "=== Aguardando limpeza... ==="
sleep 10

echo "=== Verificando limpeza ==="
kubectl get applications -n argocd | grep -E "saga-|cloudnative" || echo "Nenhuma Application encontrada"
kubectl get namespaces | grep -E "saga-|cnpg" || echo "Nenhum namespace encontrado"

echo "=== Reset completo! Agora você pode reaplicar com: kubectl apply -k ./argocd-gitops ==="
```

