# Exercício 4: Removendo atributos de alta cardinalidade em métricas

## Objetivo

Usar o transform processor para remover resource attributes de alta cardinalidade que causam explosão de séries temporais em backends de métricas como Prometheus ou Mimir.

| | Descrição |
|---|---|
| **Estado inicial** | Uma configuração que lê métricas de aplicações Java (JVM, HTTP) de arquivos JSON. Os resource attributes incluem atributos pesados como `process.command_line` (~2000 caracteres), `process.command_args`, `container.id`, PIDs e UUIDs que não agregam valor para observabilidade de métricas. |
| **Objetivo** | Adicionar statements no **transform processor** para remover os atributos desnecessários, mantendo apenas os que são úteis para identificar o serviço e o contexto de deploy. |

## Contexto real

Em ambientes de produção com centenas de serviços Java, cada resource de métrica carrega atributos como `process.command_line` — uma string com a linha de comando completa da JVM, incluindo classpath e flags. Em um ambiente real do Bradesco, esse atributo tem em média **2.361 caracteres** por ocorrência, repetido em **7.131 resources** por janela de coleta — são mais de **16 MB de desperdício** a cada scrape.

O problema não é só tamanho em bytes. Cada combinação única de resource attributes gera uma série temporal diferente no backend. Quando atributos como `process.pid`, `container.id` ou `service.instance.id` mudam a cada restart do pod, todas as séries temporais anteriores se tornam "órfãs" e novas são criadas. Isso causa:

- **Explosão de séries temporais** (cardinalidade) no Prometheus/Mimir
- **Aumento de custo** de armazenamento e ingestão
- **Degradação de performance** em queries e dashboards
- **Atingimento de limites** de ingestão do backend

Atributos que são úteis para métricas (e devem ser mantidos):
- `service.name`, `k8s.namespace.name`, `k8s.deployment.name` — identificam o serviço
- `telemetry.sdk.*` — informam a versão do SDK

Atributos que são ruído em métricas (e devem ser removidos):
- `process.command_line` — enorme, muda raramente, e não ajuda em queries de métricas
- `process.command_args` — mesma informação em formato de array
- `process.pid`, `process.parent_pid` — mudam a cada restart, alta cardinalidade
- `host.id` — UUID do host, alta cardinalidade em ambientes cloud

## Arquivos fornecidos

- `config.yaml` — configuração inicial com um transform processor vazio para completar
- `pagamentos-metrics.json` — métricas de serviços do domínio de pagamentos (`pagamentos-srv-processador`, `pagamentos-srv-liquidacao`, `pix-srv-dict`)
- `cartoes-metrics.json` — métricas de serviços do domínio de cartões (`cartoes-srv-antifraude`, `cartoes-srv-autorizador`, `cartoes-srv-fatura`)

Cada arquivo contém métricas JVM (memória, GC, threads, classes, CPU) e métricas HTTP (histogramas de duração de requests) com 2 réplicas por serviço.

## Passo 1: Observar o problema

Rode o Collector com a configuração inicial:

```bash
otelcol-contrib --config config.yaml
```

Observe na saída (debug exporter) os resource attributes de cada batch. Note a presença de:

- `process.command_line` com milhares de caracteres (classpath Java completo)
- `process.command_args` com dezenas de elementos
- `process.pid` e `process.parent_pid` com valores numéricos
- `container.id` com 64 caracteres hexadecimais
- `host.id` com um UUID

Imagine esses atributos em 500 serviços, cada um com 3 réplicas, coletando métricas a cada 15 segundos. A cardinalidade explode.

## Passo 2: Completar o transform processor

Abra `config.yaml` e complete as statements do transform processor. Você precisa usar o contexto `resource` com `metric_statements` para remover os atributos indesejados dos resources das métricas.

Atributos a remover:
- `process.command_line`
- `process.command_args`
- `process.pid`
- `process.parent_pid`
- `host.id`

Dica: a função OTTL para remover uma chave de um mapa é `delete_key(mapa, "chave")`.

## Passo 3: Verificar

Reinicie o Collector com a configuração modificada. Compare a saída do debug exporter:

- **Antes:** cada resource tem 15 atributos, incluindo strings enormes
- **Depois:** cada resource tem apenas os atributos úteis (`service.name`, `k8s.namespace.name`, `k8s.deployment.name`, `telemetry.sdk.*`, etc.)

Os dados das métricas (datapoints, histogramas) permanecem intactos — apenas os resource attributes desnecessários foram removidos.