# Exercício 8 (bônus): Amostragem inteligente de rastros

## Objetivo

Usar o tail sampling processor para implementar amostragem inteligente de rastros: manter 100% dos rastros com erros e latência alta, e amostrar apenas uma porcentagem dos rastros saudáveis. Em produção, mesmo após filtrar ruído (exercício 7), o volume de rastros pode ser avassalador e caro para armazenar.

| | Descrição |
|---|---|
| **Estado inicial** | Uma configuração com otlpjsonfile receiver que lê rastros sintéticos de diversos serviços. Os rastros incluem erros (timeout de gateway, falha de banco, circuit breaker), operações lentas (>2s), operações normais saudáveis, e ruído de health checks. O tail sampling processor está declarado mas com `policies` vazio — todos os rastros passam sem amostragem. |
| **Objetivo** | Configurar políticas no tail sampling processor para: (1) manter sempre rastros com erro, (2) manter sempre rastros lentos (>2s), e (3) amostrar 10% dos rastros restantes. |

## Por que tail-sampling?

Existem duas abordagens para amostragem de rastros:

**Head-based sampling** (amostragem na origem): a decisão de coletar ou descartar um rastro é tomada no início, antes de qualquer trecho (span) ser gerado. É simples e eficiente, mas tem um problema crítico: como a decisão é tomada antes de saber o resultado da operação, você pode descartar rastros de erros ou de operações lentas que seriam essenciais para debugging.

**Tail-based sampling** (tail-sampling): a decisão é tomada no Collector, **depois** que todos os trechos de um rastro chegam. Isso permite inspecionar o rastro completo — status de erro, duração total, atributos — e tomar uma decisão informada. O custo é que o Collector precisa manter rastros em memória enquanto aguarda todos os trechos.

## Contexto real

Em um ambiente de produção com 50+ microsserviços:

- **~340.000 trechos/dia** de ruído (health checks, pings, assets estáticos) — já tratados no [Exercício 7](../exercicio-07/README.md)
- Mesmo após filtrar ruído, o volume de rastros de negócio pode ser de **milhões de trechos/dia**
- Manter 100% dos rastros custa caro em armazenamento e licenciamento do backend
- Mas descartar rastros aleatoriamente (head-based) pode eliminar exatamente os rastros que você precisa para debugar um incidente

A solução é **tail-sampling**: manter 100% dos rastros de erro e de operações lentas (que representam uma fração pequena do total), e amostrar uma porcentagem dos rastros saudáveis.

## Dados de rastros

O arquivo `traces.json` contém 25 rastros sintéticos representando um mix realístico de produção:

| Categoria | Quantidade | Exemplos |
|---|---|---|
| **Erros** (status ERROR) | 4 rastros | Gateway timeout no SPI do Banco Central, falha de conexão Oracle RAC, circuit breaker aberto, score de fraude alto |
| **Lentos** (duração >2s) | 4 rastros | INSERT lento no PostgreSQL, TED com consulta demorada, extrato com query pesada no Oracle, chamada lenta à API da B3 |
| **Normais saudáveis** | 11 rastros | PIX rápido, consulta de saldo, análise antifraude, validação mtoken, simulação de crédito, cotação de seguros, etc. |
| **Ruído** (health checks) | 6 rastros | GET /health/liveness, GET /health/readiness, GET /ready, Redis PING |

## Passo 1: Observar o problema

Rode o Collector com a configuração inicial:

```bash
otelcol-contrib --config config.yaml
```

Com `policies: []`, o tail sampling processor não tem regras definidas e vai descartar todos os rastros. Observe que nenhum rastro aparece na saída do debug exporter.

## Desafio: Configurar políticas de amostragem

Abra `config.yaml` e encontre o bloco do tail sampling processor. Sua tarefa é adicionar três políticas (`policies`) para:

1. **Manter sempre rastros com erro** — use uma política do tipo `status_code`
2. **Manter sempre rastros lentos** — use uma política do tipo `latency` com threshold de 2 segundos
3. **Amostrar o restante** — use uma política do tipo `probabilistic` para manter 10% dos rastros restantes

**Dicas:**
- Consulte o [cheatsheet do Collector](../cheatsheet-collector.md) para a sintaxe do tail sampling processor
- Cada política tem um `name`, `type`, e um bloco de configuração específico do tipo
- As políticas são avaliadas em ordem — se qualquer política decide manter, o rastro é mantido
- Documentação do [tail sampling processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor)

## Verificação

Reinicie o Collector e observe a saída:

- Rastros com **erro** sempre aparecem (4/4): `pagamentos-srv-processador` com gateway timeout, `conta-digital-service` com falha Oracle, `credito-analise-service` com circuit breaker, `antifraude-service` com score de fraude
- Rastros **lentos** sempre aparecem (4/4): operações com duração >2s
- Rastros **normais** aparecem em ~10% das execuções: PIX rápido, consultas de saldo, etc.
- Rastros de **health check** também são amostrados a 10%

Rode o Collector várias vezes para confirmar que rastros de erro e lentos **sempre** aparecem, enquanto rastros normais aparecem esporadicamente.

## Parâmetros importantes

| Parâmetro | Valor | Descrição |
|---|---|---|
| `decision_wait` | 10s | Tempo que o Collector aguarda trechos adicionais antes de decidir. Rastros com trechos que chegam depois desse tempo podem ser cortados. |
| `num_traces` | 100 | Número máximo de rastros mantidos em memória simultaneamente. Em produção, ajuste conforme a memória disponível. |