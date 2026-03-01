# Exercício 9: OpenTelemetry Operator

Este exercício demonstra o uso do OpenTelemetry Operator no Kubernetes, incluindo:

- Gerenciamento do Collector via CRDs (Custom Resource Definitions)
- Auto-instrumentação de aplicações
- Target Allocator para distribuição de scrape targets do Prometheus

## Preparação do ambiente

### Pré-requisitos

Antes de começar, certifique-se de ter as seguintes ferramentas instaladas:

- [k3d](https://k3d.io/) — para criar clusters Kubernetes locais
- [kubectl](https://kubernetes.io/docs/tasks/tools/) — para interagir com o cluster
- [Docker](https://docs.docker.com/get-docker/) — necessário pelo k3d

### Criar o cluster k3d

```bash
k3d cluster create bradesco-workshop
```

Após a criação, o contexto `k3d-bradesco-workshop` estará disponível automaticamente.

### Instalar o cert-manager

O cert-manager é um pré-requisito do OpenTelemetry Operator, pois ele gerencia os certificados TLS usados pelos webhooks.

```bash
kubectl --context k3d-bradesco-workshop apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl --context k3d-bradesco-workshop wait --for=condition=Available deployments/cert-manager -n cert-manager --timeout=120s
kubectl --context k3d-bradesco-workshop wait --for=condition=Available deployments/cert-manager-webhook -n cert-manager --timeout=120s
```

### Instalar o OpenTelemetry Operator

```bash
kubectl --context k3d-bradesco-workshop apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
kubectl --context k3d-bradesco-workshop wait --for=condition=Available deployments/opentelemetry-operator-controller-manager -n opentelemetry-operator-system --timeout=120s
```

### Criar os namespaces

Os manifests deste exercício usam os namespaces `observability` e `aplicacoes`. Crie-os antes de prosseguir:

```bash
kubectl --context k3d-bradesco-workshop create ns observability
kubectl --context k3d-bradesco-workshop create ns aplicacoes
```

---

## Parte 1: Collector via Operator

O arquivo `collector.yaml` define um recurso `OpenTelemetryCollector` que o Operator usa para criar e gerenciar os pods do Collector.

Pontos para observar no YAML:

- `spec.mode`: define o modo de operação (deployment, daemonset, statefulset, sidecar)
- `spec.config`: a configuração do Collector, idêntica ao que usamos nos exercícios anteriores
- `spec.resources`: limites de CPU e memória (relação direta com o memory_limiter)
- `spec.replicas`: número de réplicas (relevante para modos deployment e statefulset)

### Aplicar o manifesto

```bash
kubectl --context k3d-bradesco-workshop apply -f collector.yaml
```

### Verificar os recursos criados

```bash
kubectl --context k3d-bradesco-workshop get pods -n observability
kubectl --context k3d-bradesco-workshop get svc -n observability
```

O instrutor vai:

1. Aplicar o YAML no cluster
2. Mostrar os pods criados
3. Modificar a config e reaplicar para demonstrar rolling update

## Parte 2: Auto-instrumentação

O arquivo `instrumentation.yaml` define um recurso `Instrumentation` que configura a injeção automática de SDKs do OpenTelemetry em aplicações.

Pontos para observar no YAML:

- `spec.exporter.endpoint`: para onde a telemetria instrumentada é enviada
- `spec.propagators`: quais propagadores de contexto usar (tracecontext, baggage, b3)
- `spec.java`, `spec.python`, etc.: configuração específica por linguagem

Para ativar a auto-instrumentação em um pod, basta adicionar a anotação correspondente. Quando o recurso `Instrumentation` está em um namespace diferente do pod, use o formato `namespace/nome`:

```yaml
annotations:
  instrumentation.opentelemetry.io/inject-java: "observability/otel-instrumentation"
```

### Aplicar os manifestos

Primeiro, aplique o recurso `Instrumentation`:

```bash
kubectl --context k3d-bradesco-workshop apply -f instrumentation.yaml
```

Em seguida, aplique o Deployment da aplicação de exemplo. O arquivo `app-deployment.yaml` mostra um Deployment com a anotação de auto-instrumentação configurada:

```bash
kubectl --context k3d-bradesco-workshop apply -f app-deployment.yaml
```

### Verificar a auto-instrumentação

Confira se os pods da aplicação foram criados e se o init container do agente OpenTelemetry foi injetado:

```bash
kubectl --context k3d-bradesco-workshop get pods -n aplicacoes
kubectl --context k3d-bradesco-workshop describe pod -n aplicacoes -l app=conta-service
```

Para verificar os logs do agente OpenTelemetry injetado:

```bash
kubectl --context k3d-bradesco-workshop logs -n aplicacoes -l app=conta-service -c conta-service
```

## Parte 3: Target Allocator (avançado)

O arquivo `target-allocator.yaml` mostra como configurar o Target Allocator para distribuir scrape targets de Prometheus entre réplicas do Collector.

Útil quando há muitos targets e o Collector roda em modo statefulset com múltiplas réplicas. O Target Allocator distribui os targets uniformemente para evitar que uma única réplica colete tudo.

### Criar o RBAC para o Target Allocator

O Target Allocator precisa de permissões para descobrir ServiceMonitors, PodMonitors e endpoints no cluster. Aplique o ClusterRole e ClusterRoleBinding antes do manifesto:

```bash
kubectl --context k3d-bradesco-workshop apply -f target-allocator-rbac.yaml
```

### Aplicar o manifesto

```bash
kubectl --context k3d-bradesco-workshop apply -f target-allocator.yaml
```

### Verificar os recursos criados

```bash
kubectl --context k3d-bradesco-workshop get pods -n observability
kubectl --context k3d-bradesco-workshop get statefulset -n observability
```

---

## Verificação geral

Para verificar o estado geral de todos os recursos criados neste exercício:

```bash
kubectl --context k3d-bradesco-workshop get all -n observability
kubectl --context k3d-bradesco-workshop get all -n aplicacoes
```

Para acompanhar os logs do Collector:

```bash
kubectl --context k3d-bradesco-workshop logs -n observability -l app.kubernetes.io/name=otel-collector-collector -f
```

Para acessar o Collector localmente via port-forward (por exemplo, para enviar telemetria de teste):

```bash
kubectl --context k3d-bradesco-workshop port-forward -n observability svc/otel-collector-collector 4317:4317 4318:4318
```

---

## Limpeza

Para remover o cluster e todos os recursos criados:

```bash
k3d cluster delete bradesco-workshop
```
