# Exercício 6: Sanitizando dados sensíveis em rastros com OTTL

## Objetivo

Usar o transform processor com OTTL (OpenTelemetry Transformation Language) para interceptar e sanitizar dados sensíveis em rastros antes que saiam do Collector. Em produção, instrumentação automática e drivers de banco frequentemente capturam credenciais e informações sensíveis como atributos de trechos (spans).

| | Descrição |
|---|---|
| **Estado inicial** | Uma configuração com otlpjsonfile receiver que lê rastros sintéticos contendo atributos sensíveis típicos de produção: senhas de truststore, credenciais MongoDB, connection strings com senhas, headers de autorização, SQLs longos, e linhas de comando JVM com credenciais. O transform processor está declarado mas com regras vazias — os dados sensíveis passam sem tratamento. |
| **Objetivo** | Preencher as regras OTTL no transform processor para: (1) remover atributos sensíveis dos trechos (`javax.net.ssl.trustStorePassword`, `password`, `userName`, `db.connection_string`, `Authorization`), (2) truncar atributos longos a 500 caracteres (captura `db.statement`), e (3) remover `process.command_line` dos resource attributes. |

## Contexto real

Em ambientes de produção com milhares de microsserviços Java, a instrumentação automática e os drivers de banco capturam dados sensíveis como atributos de trechos. Os padrões mais frequentes incluem:

- **Senhas de certificados** (`javax.net.ssl.trustStorePassword`) — frameworks de segurança Java expõem a senha do truststore JKS como atributo de trecho (752 serviços afetados)
- **Credenciais MongoDB** (`password`, `userName`) — o driver MongoDB para Java loga credenciais como atributos de trecho (824 serviços)
- **Connection strings** (`db.connection_string`) — strings de conexão com senhas embutidas capturadas pela instrumentação de banco (4.126 trechos)
- **Headers de autorização** (`Authorization`) — tokens Bearer capturados em trechos HTTP (76 serviços)
- **SQLs longos** (`db.statement`) — statements SQL com média de 322 caracteres e máximo de 4,2K (6.040 trechos)
- **Linha de comando JVM** (`process.command_line`) — comando de inicialização da JVM com todas as flags `-D`, incluindo senhas, média de 2.361 caracteres (7.131 ocorrências, ~16,8 MB)

O Collector é a última barreira antes que esses dados cheguem ao backend. Neste exercício, usamos `trace_statements` no transform processor — diferente do [Exercício 3](../exercicio-03/README.md) que trata logs.

## Arquivos fornecidos

- `config.yaml` — configuração inicial com transform processor declarado mas com regras vazias
- `traces.json` — rastros sintéticos contendo atributos sensíveis

## Passo 1: Verificar o problema

Rode o Collector com a configuração inicial para ver os dados sensíveis nos rastros:

```bash
otelcol-contrib --config config.yaml
```

Observe no debug exporter que os trechos contêm atributos como `javax.net.ssl.trustStorePassword`, `password`, `userName`, `db.connection_string` e `Authorization` com valores em texto claro. Note também que `process.command_line` nos resource attributes contém senhas de banco, Redis, e chaves de API.

## Desafio: Adicionar regras OTTL para sanitizar rastros

Abra `config.yaml` e encontre o bloco do transform processor. Sua tarefa é adicionar regras OTTL em dois contextos diferentes:

1. **Contexto `span`** — remova atributos sensíveis dos trechos e limite o tamanho de atributos longos (como `db.statement`)
2. **Contexto `resource`** — remova atributos sensíveis dos resource attributes (como `process.command_line`)

**Dicas:**
- Consulte o [cheatsheet de OTTL](../cheatsheet-ottl.md) para ver as funções disponíveis
- As funções `delete_key` e `truncate_all` são úteis aqui
- Note que rastros usam `trace_statements` (diferente de `log_statements` no Exercício 3)
- Documentação do [transform processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/transformprocessor)

## Verificação

Pare o Collector, reinicie com a config modificada, e confirme que:

- Os atributos `javax.net.ssl.trustStorePassword`, `password`, `userName`, `db.connection_string` e `Authorization` não aparecem mais nos trechos
- O atributo `db.statement` foi truncado a 500 caracteres
- O resource attribute `process.command_line` não aparece mais
- Atributos legítimos como `http.method`, `http.url`, `db.system` continuam presentes