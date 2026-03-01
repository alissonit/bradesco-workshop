# Cheatsheet: OpenTelemetry Collector

Referência rápida dos componentes e padrões de configuração do Collector mais usados neste workshop.

## Estrutura de uma configuração

```yaml
receivers:     # Como dados entram no Collector
processors:    # Como dados são transformados
exporters:     # Para onde dados são enviados
connectors:    # Conectam um pipeline a outro
extensions:    # Funcionalidades auxiliares (health check, pprof, etc.)
service:
  extensions: [...]
  pipelines:   # Define o fluxo de dados
    traces:
      receivers: [...]
      processors: [...]
      exporters: [...]
```

## Receivers (como dados entram)

### otlp

Recebe dados via protocolo OTLP (gRPC e HTTP).

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318
```

### filelog

Coleta logs de arquivos locais.

```yaml
receivers:
  filelog:
    include:
      - /var/log/app/*.log
      - /var/log/app/app-*.log
    start_at: beginning    # ou "end" para só novas linhas
    operators:
      - type: json_parser  # se os logs forem JSON
```

### prometheus

Coleta métricas via scrape (como o Prometheus).

```yaml
receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: minha-app
          scrape_interval: 15s
          static_configs:
            - targets: ['localhost:8080']
```

## Processors (como dados são transformados)

### memory_limiter (obrigatório)

Controla o uso de memória do Collector. Deve ser o primeiro processor no pipeline.

```yaml
processors:
  memory_limiter:
    check_interval: 1s
    limit_mib: 800
    spike_limit_mib: 200
```

O `limit_mib` define o limite total. O `spike_limit_mib` define a margem para picos. Quando o uso atinge `limit_mib - spike_limit_mib`, o Collector começa a rejeitar dados.

### batch

Agrupa dados antes de exportar para reduzir chamadas de rede.

```yaml
processors:
  batch:
    send_batch_size: 512
    timeout: 5s
```

### filter

Descarta dados baseado em condições OTTL.

```yaml
processors:
  filter:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.route"] == "/health"'
```

### transform

Modifica dados usando OTTL. Suporta `log_statements`, `metric_statements` e `trace_statements`, cada um com seu próprio contexto.

```yaml
processors:
  transform:
    log_statements:
      - context: log
        statements:
          - delete_key(attributes, "auth.token")
    metric_statements:
      - context: resource
        statements:
          - delete_key(attributes, "process.command_line")
          - delete_key(attributes, "process.pid")
    trace_statements:
      - context: span
        statements:
          - delete_key(attributes, "db.password")
```

Note que `metric_statements` aceita os contextos `resource`, `metric` e `datapoint`. O contexto `resource` é útil para remover atributos de alta cardinalidade que causam explosão de séries temporais.

### tailsampling

Amostragem inteligente de rastros via tail-sampling. A decisão de manter ou descartar um rastro é tomada **depois** que todos os trechos (spans) chegam, permitindo inspecionar status de erro, latência total, e atributos.

```yaml
processors:
  tail_sampling:
    decision_wait: 10s      # tempo de espera por trechos adicionais
    num_traces: 100          # máximo de rastros em memória simultaneamente
    policies:
      - name: always_sample_errors
        type: status_code
        status_code:
          status_codes:
            - ERROR
      - name: always_sample_slow
        type: latency
        latency:
          threshold_ms: 2000
      - name: percentage_sample
        type: probabilistic
        probabilistic:
          sampling_percentage: 10
```

As políticas são avaliadas em ordem. Se qualquer política decide manter o rastro, ele é mantido. Políticas comuns: `status_code` (por status do trecho), `latency` (por duração total), `probabilistic` (porcentagem aleatória), `string_attribute` (por valor de atributo).

### resource

Adiciona, modifica ou remove atributos de recurso.

```yaml
processors:
  resource:
    attributes:
      - key: service.name
        value: minha-aplicacao
        action: upsert
      - key: deployment.environment
        value: producao
        action: upsert
```

### logdedup

Agrega logs repetidos em uma única entrada com contador.

```yaml
processors:
  logdedup:
    log_count_attribute: log_count
    interval: 10s
```

## Exporters (para onde dados são enviados)

### debug

Imprime dados no terminal. Útil para desenvolvimento e troubleshooting.

```yaml
exporters:
  debug:
    verbosity: detailed    # basic, normal, ou detailed
```

### file

Grava dados em arquivo JSON.

```yaml
exporters:
  file:
    path: ./output.json
```

### otlp

Envia dados para um backend via OTLP gRPC.

```yaml
exporters:
  otlp:
    endpoint: backend:4317
    tls:
      insecure: false
```

### otlphttp

Envia dados para um backend via OTLP HTTP.

```yaml
exporters:
  otlphttp:
    endpoint: https://backend:4318
```

## Ordem dos processors no pipeline

A ordem importa. A recomendação é:

```text
memory_limiter -> filter -> transform -> resource -> batch
```

1. `memory_limiter` primeiro para proteger o Collector
2. `filter` descarta dados antes de gastar CPU transformando
3. `transform` modifica o que sobrou
4. `resource` adiciona metadados
5. `batch` agrupa tudo antes de exportar

## Comandos úteis

```bash
# Rodar o Collector
otelcol-contrib --config config.yaml

# Validar uma configuração sem rodar
otelcol-contrib validate --config config.yaml

# Ver a versão
otelcol-contrib --version

# Ver os componentes disponíveis
otelcol-contrib components

# Enviar rastros de teste
telemetrygen traces --traces 5 --otlp-insecure

# Enviar métricas de teste
telemetrygen metrics --metrics 5 --otlp-insecure

# Enviar logs de teste
telemetrygen logs --logs 5 --otlp-insecure
```
