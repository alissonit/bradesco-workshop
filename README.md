# Workshop: OpenTelemetry Collector e Operator na prática

Workshop presencial de meio período sobre OpenTelemetry Collector e Operator, com exercícios práticos hands-on.

- **Horário**: 09:00 - 12:00
- **Formato**: Presencial, hands-on com exercícios guiados
- **Instrutor**: Juraci Paixão Kröhling

## Pré-requisitos

Antes do workshop, cada participante precisa:

1. Baixar o binário do OpenTelemetry Collector Contrib para o sistema operacional do seu laptop
2. Baixar o material deste repositório (clone via Git ou download do .zip)
3. Ter um editor de texto instalado (VS Code recomendado, mas qualquer um serve)
4. Verificar que o Collector roda executando `otelcol-contrib --version` no terminal
5. (Apenas para o exercício 09) Ter [Docker](https://docs.docker.com/get-docker/) e [k3d](https://k3d.io/) instalados para criar um cluster Kubernetes local

### Download do Collector

Baixe a versão 0.143.1 correspondente ao seu sistema operacional:

| Sistema operacional | Download |
| --- | --- |
| Linux (amd64) | [otelcol-contrib_0.143.1_linux_amd64.tar.gz](https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.143.1/otelcol-contrib_0.143.1_linux_amd64.tar.gz) |
| macOS (arm64, Apple Silicon) | [otelcol-contrib_0.143.1_darwin_arm64.tar.gz](https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.143.1/otelcol-contrib_0.143.1_darwin_arm64.tar.gz) |
| macOS (amd64, Intel) | [otelcol-contrib_0.143.1_darwin_amd64.tar.gz](https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.143.1/otelcol-contrib_0.143.1_darwin_amd64.tar.gz) |
| Windows (amd64) | [otelcol-contrib_0.143.1_windows_amd64.tar.gz](https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v0.143.1/otelcol-contrib_0.143.1_windows_amd64.tar.gz) |

Após baixar, extraia o arquivo e mova o binário `otelcol-contrib` para um diretório no seu PATH, ou para a raiz deste repositório.

## Programação

### 09:00 - 09:15 | Abertura (15 min)

Apresentação dos objetivos, verificação de ambiente e visão geral dos três blocos do workshop.

### 09:15 - 10:15 | Bloco 1: Collector na prática (60 min)

Configuração, pipelines e troubleshooting. Neste bloco vamos entender a anatomia de uma configuração do Collector: receivers, processors, exporters, extensions e connectors, e como o pipeline conecta tudo através de `service.pipelines`. Vamos também discutir padrões de deployment (agent vs. gateway) e os trade-offs de cada abordagem.

A parte prática inclui dois exercícios:

**Exercício 1: Primeira pipeline** (diretório `exercicio-01/`). Montar e rodar uma pipeline mínima que lê métricas reais de arquivos de payload, processa com batch, e exporta para o debug exporter. Depois, adicionar um file exporter para gravar a telemetria em disco.

**Exercício 2: Coletando logs locais** (diretório `exercicio-02/`). Configurar o filelog receiver para capturar arquivos de log com datas no nome. O exercício inclui dados fictícios prontos para teste.

### 10:15 - 10:30 | Intervalo (15 min)

### 10:30 - 11:15 | Bloco 2: Segurança, resiliência e otimização (45 min)

Problemas reais e como resolver no Collector. Vamos abordar temas como: controle de memória com o memory_limiter (por que é obrigatório e como configurar), segurança na telemetria usando o transform processor com OTTL para redaction de dados sensíveis, redução de ruído com o filter processor e o logdedup processor, e amostragem inteligente de rastros com o tail sampling processor.

A parte prática inclui cinco exercícios (mais um bônus para fazer em casa):

**Exercício 3: Protegendo dados sensíveis em logs com OTTL** (diretório `exercicio-03/`). Criar regras OTTL no transform processor para substituir CPFs por `[REDACTED]`, remover atributos sensíveis em logs como tokens JWT (`auth.token`) e senhas de banco (`db.password`), e truncar atributos longos.

**Exercício 4: Removendo atributos de alta cardinalidade em métricas** (diretório `exercicio-04/`). Usar o transform processor com `metric_statements` no contexto `resource` para remover resource attributes de alta cardinalidade (`process.command_line`, `process.command_args`, `process.pid`, `host.id`, etc.) que causam explosão de séries temporais em backends como Prometheus e Mimir.

**Exercício 5: Reduzindo volume de logs** (diretório `exercicio-05/`). Usar o filter processor para descartar logs sem valor (Spring Boot internals, health checks do Actuator) e o logdedup processor para agregar logs repetidos (Kafka disconnects, erros em loop), com potencial de redução de mais de 80% do volume.

**Exercício 6: Sanitizando dados sensíveis em rastros com OTTL** (diretório `exercicio-06/`). Criar regras OTTL no transform processor com `trace_statements` para remover credenciais de trechos (senhas de truststore, credenciais MongoDB, connection strings, headers de autorização), truncar SQLs longos, e remover `process.command_line` dos resource attributes.

**Exercício 7: Filtrando ruído em rastros** (diretório `exercicio-07/`). Usar o filter processor para descartar trechos sem valor — health checks do Kubernetes (`GET /health/**`), pings de conexão Redis (`PING`), e assets estáticos (`.well-known`, `swagger-ui`). Baseado em dados reais que mostram ~340.000 trechos/dia de ruído.

**Exercício 8 (bônus): Amostragem inteligente de rastros** (diretório `exercicio-08/`). Usar o tail sampling processor para implementar tail-sampling: manter 100% dos rastros com erros e latência alta (>2s), e amostrar apenas 10% dos rastros saudáveis. Exercício para fazer em casa ou em tempo livre.

### 11:15 - 11:50 | Bloco 3: OpenTelemetry Operator (35 min)

Gerenciamento de Collectors e auto-instrumentação no Kubernetes. Este bloco é uma demonstração ao vivo pelo instrutor; os participantes acompanham os YAMLs no laptop (diretório `exercicio-09/`). Participantes com Docker e k3d instalados podem acompanhar criando seu próprio cluster local.

**Parte 1: Operator e CRD do Collector.** O que é o OpenTelemetry Operator, deploy declarativo de Collectors, modos de operação (Deployment, DaemonSet, StatefulSet, Sidecar), e como atualizar configuração sem downtime.

**Parte 2: Auto-instrumentação.** O CRD Instrumentation para injeção automática de SDKs em aplicações Java, .NET, Python, Node.js e Go. Demonstração com uma aplicação Java.

**Parte 3: Target Allocator.** Distribuição de scrape targets de Prometheus entre réplicas do Collector, com integração com ServiceMonitor e PodMonitor.

### 11:50 - 12:00 | Encerramento (10 min)

Recapitulação, ações concretas e próximos passos.

## Estrutura do repositório

```text
bradesco-workshop/
├── README.md                    # Este arquivo
├── exercicio-01/
│   ├── README.md                # Instruções do exercício
│   ├── config.yaml              # Pipeline mínima: otlpjsonfile -> batch -> debug
│   └── *.json                   # Payloads OTLP reais (métricas anonimizadas)
├── exercicio-02/
│   ├── README.md                # Instruções do exercício
│   ├── config.yaml              # Filelog receiver com glob patterns
│   └── logs/                    # Arquivos de log fictícios
├── exercicio-03/
│   ├── README.md                # Instruções do exercício
│   ├── config.yaml              # Transform processor com OTTL (incompleto)
│   └── logs/                    # Logs com PII fictícia
├── exercicio-04/
│   ├── README.md                # Instruções do exercício
│   ├── config.yaml              # Transform processor para métricas (incompleto)
│   └── *.json                   # Payloads OTLP reais (métricas de serviços Java)
├── exercicio-05/
│   ├── README.md                # Instruções do exercício
│   ├── config.yaml              # Filter + logdedup processors (incompleto)
│   └── logs/                    # Logs ruidosos simulados
├── exercicio-06/
│   ├── README.md                # Instruções do exercício
│   ├── config.yaml              # Transform processor para rastros (incompleto)
│   └── traces.json              # Rastros sintéticos com dados sensíveis
├── exercicio-07/
│   ├── README.md                # Instruções do exercício
│   ├── config.yaml              # Filter processor para rastros (incompleto)
│   └── traces.json              # Rastros sintéticos com ruído (health checks, PINGs)
├── exercicio-08/
│   ├── README.md                # Instruções do exercício
│   ├── config.yaml              # Tail sampling processor (incompleto)
│   └── traces.json              # Rastros sintéticos (erros, lentos, normais, health checks)
├── exercicio-09/
│   ├── README.md                # Instruções do exercício (K8s Operator demo)
│   ├── collector.yaml           # CRD OpenTelemetryCollector
│   ├── instrumentation.yaml     # CRD Instrumentation
│   ├── target-allocator.yaml    # Exemplo de Target Allocator
│   └── app-deployment.yaml      # Deployment de exemplo
├── cheatsheet-ottl.md           # Referência rápida de funções OTTL
└── cheatsheet-collector.md      # Referência rápida de componentes do Collector
```

## Dúvidas

Se tiver problemas com a instalação do Collector ou com qualquer exercício, consulte o `README.md` dentro de cada diretório de exercício. Cada um contém instruções detalhadas.
