# Dynatrace_automatic-injection
Desabilitar a auto-injeção de OneAgent globalmente e ativar apenas em namespaces com o label  ```bash dynatrace.com/inject=true. ```

## Passo a passo: Injeção seletiva do OneAgent com DynaKube

## 1️⃣ Acesse o recurso DynaKube do seu cluster

No Rancher ou via  <strong> kubectl </strong>, edite o recurso atual:

```bash kubectl edit dynakube productionk8s -n dynatrace ```

## 2️⃣ Altere o campo de auto-injeção automática

No YAML, altere ou adicione a anotação:

```yaml

metadata:
  annotations:
    feature.dynatrace.com/automatic-injection: "false"
```

## 3️⃣ Defina um filtro por label de namespace

No bloco oneAgent.cloudNativeFullStack, troque:

```yaml
namespaceSelector: {}
```

Por:

```yaml

namespaceSelector:
  matchLabels:
    dynatrace.com/inject: "true"
```

## 4️⃣ YAML final da seção relevante (DynaKube)

```yaml
metadata:
  annotations:
    feature.dynatrace.com/automatic-injection: "false"

spec:
  oneAgent:
    cloudNativeFullStack:
      args:
        - '--set-host-group=productionk8s'
      namespaceSelector:
        matchLabels:
          dynatrace.com/inject: "true"
      oneAgentResources: {}
      tolerations:
        - effect: NoSchedule
          key: node-role.kubernetes.io/master
          operator: Exists
        - effect: NoSchedule
          key: node-role.kubernetes.io/control-plane
          operator: Exists
```

## 5️⃣ Salve e feche a edição

A alteração será aplicada automaticamente pelo Dynatrace Operator.

## 6️⃣ Aplique o label nos namespaces onde deseja a injeção

```bash
kubectl label namespace <nome-do-namespace> dynatrace.com/inject=true --overwrite
```
Exemplo:

```bash
kubectl label namespace app-prod dynatrace.com/inject=true --overwrite
```

## 7️⃣ Reinicie os pods no namespace para aplicar a injeção

A injeção do OneAgent só ocorre quando o pod é (re)criado.

```bash
kubectl delete pods --all -n app-prod
```
## 8️⃣ Verifique se a injeção ocorreu corretamente

Use este comando para verificar se o OneAgent foi injetado:

```bash
kubectl describe pod <pod-name> -n app-prod | grep -i oneagent
```
Ou veja diretamente no Dynatrace UI → Hosts ou Smartscape.

## 9️⃣ (Opcional) Confirme os namespaces com injeção habilitada

```bash
kubectl get namespaces --show-labels | grep dynatrace.com/inject
```

## 🧠 Observações importantes:

| Situação                                  | Comportamento                   |
| ----------------------------------------- | ------------------------------   
| Namespace **sem label**                   | ❌ Não recebe OneAgent          |
| Namespace com `dynatrace.com/inject=true` | ✅ Recebe OneAgent após restart |
| Pods existentes                           | ❌ Precisam ser reiniciados     |


# Validações 

## Verificar Configuração do Dynakube

```bash 
kubectl get dynakube -n dynatrace
```
## Para ver os detalhes:

```bash 
kubectl describe dynakube productionk8s -n dynatrace
```
Verifique se contém seções como:

* oneAgent:
* applicationMonitoring:
* namespaceSelector:

## Verificar Namespaces com Label de Dynatrace

```bash
kubectl get ns --show-labels | grep dynakube.internal.dynatrace.com/instance
```
Isso mostra se o label de autoingestão está aplicado corretamente.


## Verificar Injeção do OneAgent em Pods

Escolha um namespace que esteja com o label do Dynatrace, por exemplo <my-app>:

```bash
kubectl get pods -n my-app -o json | jq '.items[].spec.containers[].name'
```

## Verificar Logs do OneAgent Operator

Veja se o operador está funcionando corretamente:

```bash
kubectl get pods -n dynatrace
```
Procure um pod com nome tipo:
dynatrace-oneagent-operator-xxxxx

Depois: 

```bash
kubectl logs <nome-do-pod> -n dynatrace
```

Você deve ver mensagens como:

* instrumenting pod
* injecting Dynatrace OneAgent
* namespace matched by namespaceSelector


## Verificar Métricas no Dynatrace

No Dynatrace (interface web):

* Vá até “Hosts” ou “Workloads”
* Verifique se os pods e namespaces aparecem
* Veja se estão gerando métricas, logs ou traces
