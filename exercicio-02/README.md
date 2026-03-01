# Exercício 2: Coletando logs locais

## Objetivo

Usar o filelog receiver para capturar arquivos de log do disco e enviá-los pelo pipeline do Collector. Em produção, isso substitui agentes de log tradicionais como Fluentd ou Filebeat.

| | Descrição |
|---|---|
| **Estado inicial** | Uma configuração com filelog receiver que lê arquivos `logs/app-*.log` e exibe no terminal. Os logs aparecem sem identificação de origem (sem `service.name` ou `deployment.environment.name`). |
| **Objetivo** | Adicionar um **resource processor** para injetar atributos que identifiquem a origem dos logs (`service.name` e `deployment.environment.name`). |

## Contexto real

Em ambientes com milhares de microsserviços, métricas e logs frequentemente chegam ao backend sem identificação de origem. Quando o `service.name` está ausente, os dados aparecem como "unknown" — tornando impossível filtrar, correlacionar e criar alertas por serviço. O resource processor resolve isso injetando atributos de recurso diretamente no pipeline do Collector.

## Arquivos fornecidos

- `config.yaml` — configuração inicial com filelog receiver e debug exporter
- `logs/app-2026-02-24.log` e `logs/app-2026-02-25.log` — arquivos de log da aplicação

## Passo 1: Verificar os arquivos de log

Olhe o conteúdo do diretório `logs/` e abra um dos arquivos para ver o formato.

## Passo 2: Rodar o Collector

A configuração em `config.yaml` já tem o filelog receiver configurado com um glob pattern que captura todos os arquivos `.log` do diretório `logs/`.

```bash
otelcol-contrib --config config.yaml
```

Os logs dos arquivos devem aparecer na saída do debug exporter. Note que não há `service.name` nem `deployment.environment.name` nos dados.

## Passo 3: Entender a configuração

Observe no `config.yaml`:

- O `include` usa um glob pattern para capturar arquivos com qualquer data no nome
- O `start_at` define se o Collector lê desde o início do arquivo ou apenas novas linhas
- O `operators` pode ser usado para parsear o conteúdo das linhas

## Desafio: Adicionar resource attributes

Adicione um **resource processor** ao pipeline para injetar os atributos `service.name` e `deployment.environment.name` nos logs. Não esqueça de adicioná-lo também ao pipeline na seção `service.pipelines`.

**Dica:** consulte a documentação do [resource processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/resourceprocessor). A ação `upsert` permite inserir ou atualizar um atributo.

## Verificação

Após aplicar suas mudanças, reinicie o Collector e confirme que os logs agora contêm os atributos `service.name` e `deployment.environment.name` na saída do debug exporter.