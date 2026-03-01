# Exercício 3: Protegendo dados sensíveis em logs com OTTL

## Objetivo

Usar o transform processor com OTTL (OpenTelemetry Transformation Language) para interceptar e sanitizar dados sensíveis em logs antes que saiam do Collector. Em produção, isso evita que PII (dados pessoais) e credenciais cheguem ao backend de observabilidade.

| | Descrição |
|---|---|
| **Estado inicial** | Uma configuração com filelog receiver que lê logs contendo PII fictícia (CPFs, CNPJs, tokens JWT, senhas de banco, dados bancários). O transform processor está declarado mas com regras vazias — os dados sensíveis passam sem tratamento. |
| **Objetivo** | Preencher as regras OTTL no transform processor para: (1) substituir CPFs por `[REDACTED]`, (2) remover atributos sensíveis como `auth.token` e `db.password`, e (3) truncar atributos longos a 256 caracteres. |

## Contexto real

Em ambientes de produção, é comum encontrar vazamentos de dados sensíveis nos logs de aplicações. Os padrões mais frequentes incluem:

- **PII bancária** (`cpf`, `cnpj`, `agencia`, `conta`) logada por serviços de domínio durante operações como PIX, consulta de saldo, e pagamento de boletos
- **Tokens JWT** (`auth.token`) logados por serviços de autenticação e APIs internas
- **Senhas de banco de dados** (`db.password`) logadas acidentalmente em mensagens de erro de conexão

O Collector é a última barreira antes que esses dados cheguem ao backend — as regras OTTL no transform processor são a defesa mais eficiente.

> **Nota:** para sanitização de dados sensíveis em rastros (trechos), veja o [Exercício 6](../exercicio-06/README.md).

## Arquivos fornecidos

- `config.yaml` — configuração inicial com transform processor declarado mas com regras vazias
- `logs/app-com-pii.log` — arquivo de log contendo PII fictícia

## Passo 1: Verificar o problema

Rode o Collector com a configuração inicial para ver os dados sensíveis aparecendo na saída:

```bash
otelcol-contrib --config config.yaml
```

Observe no debug exporter que os logs contêm atributos como `user.cpf`, `auth.token` e `db.password` com valores em texto claro.

## Desafio: Adicionar regras OTTL

Abra `config.yaml` e encontre o bloco do transform processor. Ele está declarado mas as regras estão vazias. Sua tarefa é adicionar regras OTTL no contexto `log` para:

1. **Substituir CPFs por `[REDACTED]`** — use uma função OTTL que permita encontrar e substituir valores baseado em um padrão regex (formato CPF: `XXX.XXX.XXX-XX`)
2. **Remover atributos sensíveis** — elimine completamente os atributos `auth.token` e `db.password`
3. **Truncar atributos longos** — limite todos os atributos a 256 caracteres

**Dicas:**
- Consulte o [cheatsheet de OTTL](../cheatsheet-ottl.md) para ver as funções disponíveis
- As funções `replace_pattern`, `delete_key` e `truncate_all` são úteis para esse tipo de tarefa
- Documentação do [transform processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/transformprocessor)

## Verificação

Pare o Collector, reinicie com a config modificada, e confirme que:

- O valor de `user.cpf` agora mostra `[REDACTED]`
- Os atributos `auth.token` e `db.password` não aparecem mais
- Nenhum atributo tem mais que 256 caracteres