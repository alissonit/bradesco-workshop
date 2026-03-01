# Exercício 1: Primeira pipeline

## Objetivo

Entender como uma pipeline do OpenTelemetry Collector funciona na prática: receivers recebem dados, processors transformam, e exporters enviam para algum destino.

| | Descrição |
|---|---|
| **Estado inicial** | Uma configuração pronta que lê métricas reais de arquivos JSON e exibe no terminal via debug exporter. |
| **Objetivo** | Adicionar um **file exporter** ao pipeline para gravar a telemetria também em disco (`output.json`). |

## Arquivos fornecidos

- `config.yaml` — configuração inicial do Collector com dois receivers e um debug exporter
- `35655312-metrics.json` e `35655313-metrics.json` — métricas reais de infraestrutura Kubernetes

## Passo 1: Rodar o Collector

```bash
otelcol-contrib --config config.yaml
```

O Collector vai ler os arquivos JSON automaticamente e exibir as métricas no terminal via debug exporter.

Observe na saída métricas de infraestrutura Kubernetes (`k8s.*`, `container.*`) e métricas internas do Collector (`otelcol_*`).

## Passo 2: Entender a configuração

Abra `config.yaml` e observe:

- O receiver `otlpjsonfile` é instanciado duas vezes (`otlpjsonfile/1` e `otlpjsonfile/2`) para ler arquivos diferentes
- Ambos alimentam o mesmo pipeline via `receivers: [otlpjsonfile/1, otlpjsonfile/2]`
- O `batch` processor agrupa os dados antes de exportar

## Desafio: Adicionar um file exporter

Modifique `config.yaml` para adicionar um segundo exporter que grava a telemetria em um arquivo `output.json`. Você vai precisar:

1. Declarar o novo exporter na seção `exporters`
2. Adicioná-lo ao pipeline na seção `service.pipelines`

Após modificar, pare o Collector (Ctrl+C), reinicie com a nova config, e verifique que o arquivo `output.json` foi criado com os dados.

**Dica:** consulte a documentação do [file exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/fileexporter).