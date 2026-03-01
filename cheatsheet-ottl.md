# Cheatsheet: OTTL (OpenTelemetry Transformation Language)

Referência rápida das funções OTTL mais usadas no transform processor e no filter processor.

## Contextos

O OTTL opera em contextos diferentes dependendo do tipo de dado:

| Contexto | Usado em | Campos acessíveis |
| --- | --- | --- |
| `log` | Pipelines de logs | `body`, `attributes`, `resource.attributes`, `severity_text`, `severity_number` |
| `span` | Pipelines de rastros | `name`, `attributes`, `resource.attributes`, `status`, `kind` |
| `resource` | Qualquer pipeline (via `metric_statements`, `log_statements`, `trace_statements`) | `attributes` (os resource attributes diretamente) |
| `metric` | Pipelines de métricas | `name`, `description`, `unit`, `type` |
| `datapoint` | Pontos de dados de métricas | `attributes`, `value` |

**Nota sobre o contexto `resource`:** No contexto `resource`, `attributes` se refere diretamente aos resource attributes (não é necessário usar `resource.attributes`). Isso é útil para remover atributos de alta cardinalidade em métricas, por exemplo:

```yaml
processors:
  transform:
    metric_statements:
      - context: resource
        statements:
          - delete_key(attributes, "process.command_line")
          - delete_key(attributes, "process.pid")
```

## Acessando valores

```text
attributes["http.route"]              # atributo do trecho ou log
resource.attributes["service.name"]   # atributo do recurso
body                                  # corpo do log
name                                  # nome do trecho ou métrica
```

## Funções de transformação

Usadas no transform processor para modificar dados.

```yaml
# Definir um valor
set(attributes["environment"], "producao")

# Definir valor condicional
set(attributes["tier"], "critical") where attributes["service.name"] == "auth-service"

# Remover um atributo
delete_key(attributes, "auth.token")

# Remover múltiplos atributos
delete_matching_key(attributes, "internal\\..*")

# Substituir padrão com regex
replace_pattern(attributes["user.cpf"], "^\\d{3}\\.\\d{3}\\.\\d{3}-\\d{2}$", "[REDACTED]")

# Substituir match exato
replace_match(attributes["db.system"], "postgres*", "postgresql")

# Truncar todos os atributos para N caracteres
truncate_all(attributes, 256)

# Converter tipo
Int(attributes["http.status_code"])

# Concatenar strings
Concat([attributes["method"], " ", attributes["path"]], "")

# Extrair com regex
ExtractPatterns(body, "^(?P<timestamp>\\S+) (?P<level>\\S+) (?P<message>.*)$")
```

## Expressões condicionais (where)

```yaml
# Igualdade
set(attributes["tier"], "critical") where attributes["service.name"] == "auth-service"

# Negação
delete_key(attributes, "debug.info") where attributes["environment"] != "development"

# Regex match
set(attributes["pii"], true) where IsMatch(attributes["user.cpf"], "^\\d{3}\\.\\d{3}\\.\\d{3}-\\d{2}$")

# Múltiplas condições (AND implícito com múltiplos where)
set(attributes["alert"], true) where attributes["http.status_code"] >= 500

# Verificar existência
delete_key(attributes, "temp") where attributes["temp"] != nil
```

## Expressões de filtro

Usadas no filter processor para descartar dados. Se a expressão for verdadeira, o dado é descartado.

```yaml
processors:
  filter:
    error_mode: ignore
    traces:
      span:
        - 'attributes["http.route"] == "/health"'
        - 'attributes["http.route"] == "/ready"'
        - 'name == "internal-heartbeat"'
    logs:
      log_record:
        - 'severity_number < 9'
        - 'IsMatch(body, "^DEBUG.*")'
    metrics:
      metric:
        - 'name == "system.file.system.usage"'
        - 'IsMatch(name, "process\\.runtime\\..*")'
```

## error_mode

Define o comportamento quando uma expressão OTTL falha:

| Modo | Comportamento |
| --- | --- |
| `ignore` | Ignora o erro e continua processando (recomendado para produção) |
| `propagate` | Propaga o erro e rejeita o dado |
| `silent` | Ignora sem log |

## Dicas

- O memory_limiter deve vir antes do transform no pipeline
- Teste expressões OTTL com o debug exporter antes de aplicar em produção
- Use `error_mode: ignore` no filter processor para evitar que dados válidos sejam descartados por erro de expressão
- Expressões regex no OTTL seguem a sintaxe RE2 (Go)
