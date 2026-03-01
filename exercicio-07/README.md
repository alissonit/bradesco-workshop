# Exercício 7: Filtrando ruído em rastros

## Objetivo

Usar o filter processor para descartar trechos (spans) sem valor antes que sejam enviados ao backend. Em produção com milhares de microsserviços, health checks, pings de conexão e requests de assets estáticos geram milhares de trechos por minuto que não contribuem para a observabilidade — só aumentam custos de ingestão.

| | Descrição |
|---|---|
| **Estado inicial** | Uma configuração com otlpjsonfile receiver que lê rastros sintéticos de 5 serviços. Os rastros incluem health checks do Kubernetes (`GET /health/**`) com trechos internos do Spring Boot (`OperationHandler.handle`), pings de conexão Redis (`PING`), requests de assets estáticos (`/.well-known/jwks.json`), e operações de negócio reais — tudo misturado. O filter processor está declarado mas com regras vazias. |
| **Objetivo** | Adicionar condições OTTL ao filter processor para descartar trechos de health check, pings Redis, e assets estáticos, preservando apenas trechos de operações de negócio. |

## Contexto real

Em uma amostra de 10 minutos de um ambiente de produção com 50+ serviços, foram observados:

- **Health checks** (`GET /health/**`): 1.113 trechos distribuídos em 10 serviços — cada probe de liveness/readiness do Kubernetes gera um trecho de servidor e um trecho interno `OperationHandler.handle` do Spring Boot
- **Redis PING**: 139 trechos — o driver Lettuce gera um trecho para cada PING de health check de conexão com o Redis
- **Assets estáticos** (`/.well-known/jwks.json`, `/swagger-ui*`): 12 trechos sem valor para observabilidade
- **Total de ruído**: ~2.400 trechos em 10 minutos = ~340.000 trechos/dia de telemetria sem valor

Este exercício é o equivalente do [Exercício 5](../exercicio-05/README.md) (filtragem de logs) mas aplicado a rastros.

## Arquivos fornecidos

- `config.yaml` — configuração inicial com filter processor declarado mas com regras vazias
- `traces.json` — rastros sintéticos com mistura de ruído e operações de negócio

## Passo 1: Observar o problema

Rode o Collector com a configuração inicial:

```bash
otelcol-contrib --config config.yaml
```

Conte quantos trechos aparecem na saída. Observe que a maioria são health checks (`GET /health/**`, `OperationHandler.handle`), pings Redis (`PING`), e assets estáticos — as operações de negócio reais (criação de proposta, análise antifraude, ativação de mtoken, emissão de token OAuth) ficam soterradas.

## Desafio: Filtrar trechos de ruído

Abra `config.yaml` e encontre o bloco do filter processor. Sua tarefa é adicionar condições OTTL na seção `traces.span` para descartar:

1. **Health checks do Kubernetes** — trechos cujo nome corresponde ao padrão de probes de liveness/readiness, e os trechos internos do Spring Boot que os acompanham
2. **Pings de conexão Redis** — trechos de PING, mas apenas os que são do Redis (verifique `db.system` para não afetar outros trechos)
3. **Assets estáticos** — requests a endpoints como `/.well-known/*` e `/swagger-ui*`

**Dicas:**
- Consulte o [cheatsheet de OTTL](../cheatsheet-ottl.md) para a sintaxe de condições
- `IsMatch` permite comparar com regex, e condições podem ser combinadas com `and`
- Examine o arquivo `traces.json` para ver os nomes exatos dos trechos que você precisa filtrar
- Documentação do [filter processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/filterprocessor)

## Verificação

Reinicie o Collector e compare a saída com o Passo 1:

- Os trechos de `GET /health/**` e `OperationHandler.handle` sumiram
- Os trechos de `PING` do Redis sumiram
- O trecho de `GET /.well-known/jwks.json` sumiu
- Trechos de negócio continuam presentes: `POST /api/v1/propostas`, `POST /api/v1/analise/transacao`, `POST /api/v1/mtoken/activate`, `POST /api/v1/oauth/token`, e seus trechos filhos (queries de banco, HGETALL Redis)