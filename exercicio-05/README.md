# Exercício 5: Reduzindo volume de logs

## Objetivo

Usar o filter processor e o logdedup processor para eliminar logs de baixo valor e agregar logs repetidos antes que sejam enviados ao backend. Em produção, isso pode reduzir o volume de logs em mais de 80%, diminuindo custos de ingestão sem perder informação relevante.

| | Descrição |
|---|---|
| **Estado inicial** | Uma configuração com filelog receiver que lê um arquivo de log ruidoso. Os logs incluem chatter de clientes Kafka (desconexão/reconexão em loop), logs internos do Spring Boot, health checks do Actuator em nível DEBUG, erros 504 repetidos, e eventos de infraestrutura K8s — tudo misturado com logs de negócio genuínos. |
| **Objetivo** | Adicionar um **filter processor** para descartar logs sem valor (Spring Boot internals, health checks) e um **logdedup processor** para agregar logs repetidos (Kafka disconnects, erros em loop), preservando os logs de negócio. |

## Contexto real

Em ambientes de produção, a maioria dos logs são repetições ou ruído. Os padrões mais comuns incluem:

- **Kafka client chatter** (99.3% duplicação): clientes Kafka com consumer groups ociosos geram milhares de logs de desconexão/reconexão por minuto
- **Spring Boot internals** (99.9% duplicação): logs como "Returning cached instance of singleton bean" não têm valor em produção
- **Health checks** (97.3% duplicação): cada probe de liveness/readiness gera 4 linhas de log em nível DEBUG
- **Erros em loop** (96% duplicação): quando um serviço upstream está fora, o mesmo erro se repete milhares de vezes

## Arquivos fornecidos

- `config.yaml` — configuração inicial com filelog receiver e debug exporter
- `logs/app-ruidosa.log` — arquivo de log com mistura de ruído e logs de negócio genuínos

## Passo 1: Observar o problema

Rode o Collector com a configuração inicial:

```bash
otelcol-contrib --config config.yaml
```

Conte quantas linhas aparecem na saída. Observe que a maioria dos logs são repetições do mesmo conteúdo — informação com valor real (transferências PIX, renovação de token, timeout de banco) está soterrada sob o ruído.

## Desafio: Filtrar ruído e agregar repetições

Sua tarefa é modificar `config.yaml` para reduzir drasticamente o volume de logs, usando dois processors:

1. **filter processor** — descarte logs que não têm valor em produção (Spring Boot internals, health checks do Actuator). Use condições OTTL com `IsMatch` para identificar padrões no corpo dos logs.

2. **logdedup processor** — agrupe logs repetidos (Kafka disconnects, erros em loop) em uma única entrada com um contador, usando uma janela de tempo.

Lembre-se de adicionar ambos os processors ao pipeline na ordem correta: primeiro filtre, depois agregue.

**Dicas:**
- Consulte o [cheatsheet de OTTL](../cheatsheet-ottl.md) para a sintaxe de condições
- Documentação do [filter processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/filterprocessor)
- Documentação do [logdedup processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/logdedupprocessor)
- Olhe o conteúdo de `logs/app-ruidosa.log` para identificar quais padrões são ruído

## Verificação

Reinicie o Collector e compare a saída com o Passo 1:

- Os logs do Spring Boot e Actuator sumiram completamente (descartados pelo filter)
- Os logs de Kafka disconnect, erros 504, e eventos K8s aparecem uma única vez cada, com um atributo `log_count` indicando quantas repetições foram agregadas
- Os logs de negócio (PIX, auth, timeout de banco) passam normalmente

> **Próximo:** O [Exercício 6](../exercicio-06/README.md) aborda a proteção de dados sensíveis em rastros, e o [Exercício 7](../exercicio-07/README.md) aplica a mesma ideia de filtragem de ruído deste exercício mas para rastros (health checks, pings Redis, assets estáticos).
