# IA Aplicada com LLMs — Aula 15: Produção — Monitoração, telemetria e OpenTelemetry

---

## Objetivos de aprendizagem

Ao final desta nota, espera-se que o aluno seja capaz de:

- **Definir** monitoração e enumerar seus propósitos: detectar, diagnosticar, alertar, planejar capacidade e conferir objetivos de nível de serviço.
- **Distinguir** monitoração de observabilidade, e sinais de caixa-preta de sinais de caixa-branca.
- **Caracterizar** os sinais de telemetria — métricas, *logs* e *traces*, com eventos e perfis — pelo que cada um responde e pelo que custa.
- **Escolher** o que medir num serviço com os quatro sinais de ouro, o método RED e o método USE.
- **Justificar** o uso de percentis em lugar da média e **estimar** o custo de cardinalidade de uma métrica.
- **Descrever** o que o OpenTelemetry padroniza — API, SDK, OTLP, convenções semânticas e *Collector* — e o que ele não é.
- **Explicar** *trace*, *span*, contexto e propagação, e diferenciar instrumentação manual de automática.

---

## Desenvolvimento teórico

### 1. Monitoração

**Monitoração** (*monitoring*) é a coleta, o processamento, a agregação e a exibição de dados quantitativos, em tempo real, sobre um sistema — contagem e tipo de requisições, contagem e tipo de erros, tempos de processamento, tempo de vida de processos (BEYER et al., 2016, cap. 6). O dado é coletado continuamente, não sob demanda: quando alguém precisa dele, a série histórica já existe.

Os propósitos da monitoração são cinco:

| Propósito | O que a monitoração entrega |
|---|---|
| **Detectar** | o fato de que algo está fora do normal, idealmente antes do usuário |
| **Diagnosticar** | onde e desde quando, por comparação entre componentes, versões e intervalos |
| **Alertar** | a notificação de uma pessoa quando o desvio exige ação humana |
| **Planejar capacidade** | a tendência de uso de recursos, que diz quando o sistema deixará de caber na infraestrutura |
| **Conferir objetivos de serviço** | a medida contínua do que foi prometido ao usuário |

O último propósito exige duas definições. Um **SLI** (*service level indicator*) é uma medida quantitativa de um aspecto do serviço — por exemplo, a fração de requisições respondidas em menos de 300 ms. Um **SLO** (*service level objective*) é o valor-alvo de um SLI num intervalo — por exemplo, 99,5% das requisições em menos de 300 ms, medidos em 28 dias (BEYER et al., 2016, cap. 4). A diferença entre 100% e o SLO é o **orçamento de erro** (*error budget*): a quantidade de falha que o serviço pode consumir sem violar o objetivo.

Um alerta notifica uma pessoa, e por isso tem custo. A recomendação de Beyer et al. (2016, cap. 6) é alertar por **sintoma** — o que o usuário sente: lentidão, erro, indisponibilidade — e não por **causa** — CPU alta, disco cheio. Uma causa sem sintoma não exige ação imediata; um sintoma sem causa conhecida exige, e é exatamente o caso que um alerta por causa deixa passar.

### 2. Caixa-preta e caixa-branca

**Monitoração de caixa-preta** (*black-box*) observa o comportamento externo do sistema, como um cliente o observaria: uma sonda que faz uma requisição a cada minuto e registra o código de resposta e o tempo. **Monitoração de caixa-branca** (*white-box*) observa o estado interno, exposto pelo próprio sistema: contadores, histogramas, *logs*, *traces*, filas internas (BEYER et al., 2016, cap. 6).

As duas são complementares. A caixa-preta mede sintoma, e só sintoma presente: detecta a indisponibilidade, mas não a antecipa nem a explica. A caixa-branca mede também causa, e permite ver a degradação antes de ela virar falha — uma fila crescendo, um *pool* de conexões perto do limite. A caixa-preta cobre o caso em que a caixa-branca cala: um processo que caiu não emite métrica nenhuma, e só uma sonda externa registra a ausência.

### 3. Monitoração e observabilidade

**Observabilidade** (*observability*) é a propriedade de um sistema de permitir que se entenda qualquer estado interno dele a partir das saídas que ele emite, sem implantar código novo para isso (MAJORS; FONG-JONES; MIRANDA, 2022). O termo vem da teoria de controle, onde designa a possibilidade de reconstruir o estado de um sistema a partir de suas saídas.

A diferença para a monitoração está no tipo de pergunta:

| | Monitoração | Observabilidade |
|---|---|---|
| Pergunta | conhecida de antemão | formulada durante a investigação |
| Forma | painel e alerta definidos antes | consulta exploratória sobre dados ricos |
| Exemplo | *a taxa de erro passou de 1%?* | *quais requisições lentas têm em comum, que as rápidas não têm?* |
| Classe de falha | *known unknowns* | *unknown unknowns* |

Monitoração responde bem às falhas que já foram previstas, porque alguém decidiu medi-las. Em sistemas distribuídos, uma parte relevante das falhas é inédita — uma combinação de versão, cliente e região que ninguém previu —, e para essas não há painel pronto. Observabilidade é a capacidade de investigá-las com o dado que já foi emitido. As duas não se excluem: a monitoração detecta e alerta; a observabilidade diagnostica. O que as separa é o dado: perguntas novas exigem eventos com muitos atributos, que possam ser fatiados por qualquer dimensão, e não apenas séries agregadas.

### 4. Telemetria e seus sinais

**Telemetria** é o dado que um sistema emite sobre o próprio funcionamento, para ser coletado, transmitido e analisado fora dele. Cada tipo de dado é um **sinal** (*signal*). Os três sinais principais são:

- **Métrica** (*metric*): uma medida numérica agregada no tempo, identificada por um nome e por um conjunto de atributos (*labels*). Os tipos usuais são o **contador** (*counter*), monotônico, como o total de requisições; o **medidor** (*gauge*), que sobe e desce, como o tamanho de uma fila; e o **histograma** (*histogram*), que conta observações por faixa de valor (*bucket*), como a duração das requisições. A métrica é barata para armazenar e consultar, porque agrega antes de guardar, e por isso perde o caso individual.
- **Log**: um registro de um acontecimento discreto, com instante e conteúdo. O *log* **estruturado** — pares chave-valor, em geral JSON — é consultável; o *log* em texto livre exige *parsing*. O *log* preserva o caso individual e custa proporcionalmente ao volume.
- **Trace**: o registro do caminho de **uma** requisição pelo sistema, composto de *spans* — cada *span* é uma operação com início, duração e atributos, ligada à operação que a originou (SIGELMAN et al., 2010). O *trace* responde onde o tempo foi gasto e qual componente falhou numa requisição específica.

Dois sinais complementares aparecem na literatura e no OpenTelemetry: o **evento** (*event*), um registro estruturado e nomeado de um acontecimento — no OpenTelemetry, representado como um *log* com nome de evento —, e o **perfil** (*profile*), amostras de pilha de execução que dizem em que função o processo gasta CPU ou memória.

Os sinais se complementam quando podem ser correlacionados. O mecanismo é o identificador de *trace* (`trace_id`): gravado nos *logs* emitidos durante a requisição e anexado a amostras de métrica — os **exemplares** (*exemplars*) —, ele permite ir de um pico no gráfico de latência a um *trace* lento concreto, e dele aos *logs* daquela requisição.

### 5. O que medir: sinais de ouro, RED e USE

A escolha das métricas segue três métodos complementares.

**Os quatro sinais de ouro** (*four golden signals*), propostos por Beyer et al. (2016, cap. 6) para sistemas voltados ao usuário:

| Sinal | Definição |
|---|---|
| **Latência** | tempo para atender uma requisição — separando a latência das bem-sucedidas da das que falharam, já que um erro rápido distorce a média |
| **Tráfego** | demanda sobre o sistema, na unidade dele: requisições HTTP por segundo, transações por segundo |
| **Erros** | taxa de requisições que falham — explicitamente (HTTP 5xx), implicitamente (HTTP 200 com conteúdo errado) ou por política (resposta acima de um limite de tempo) |
| **Saturação** | quão "cheio" está o recurso mais restrito: fração de memória, de CPU, de conexões, profundidade de fila |

**O método RED** (WILKIE, 2018) restringe os sinais de ouro a serviços orientados a requisição — microsserviços — e os mede **por serviço**: *Rate* (requisições por segundo), *Errors* (requisições com falha por segundo) e *Duration* (distribuição do tempo de resposta). A saturação fica de fora, porque é propriedade do recurso, não do serviço.

**O método USE** (GREGG, 2013) mede **por recurso** — CPU, memória, disco, rede, *pool* de conexões, fila: *Utilization* (fração do tempo em que o recurso esteve ocupado), *Saturation* (trabalho extra que o recurso não consegue atender e enfileira) e *Errors* (contagem de eventos de erro do recurso).

Os métodos não competem: RED descreve o serviço como o cliente o vê; USE descreve os recursos sobre os quais o serviço roda. Uma piora em *Duration* (RED) diagnosticada por saturação do *pool* de conexões ao banco (USE) é o caminho típico de sintoma a causa.

### 6. Distribuições: percentis, e por que a média engana

O **percentil** $p$ de uma amostra de latências $x_1 \le x_2 \le \dots \le x_n$ é o menor valor $x_k$ tal que ao menos $p\%$ das observações são menores ou iguais a ele — $k = \lceil p \cdot n / 100 \rceil$. O p50 é a mediana; o p95 e o p99 descrevem a cauda.

A latência de serviços reais não é simétrica: tem cauda longa, causada por coleta de lixo, contenção, *retries*. A média soma a cauda ao corpo e produz um número que não corresponde a requisição nenhuma (ver Exemplo 2). O percentil descreve uma fração da população: o p99 é o tempo que 1% das requisições excede.

A cauda importa mais do que a proporção sugere, por dois motivos. Primeiro, o usuário que mais usa o serviço é o que mais vezes cai na cauda. Segundo, a cauda é **amplificada por *fan-out***: se uma requisição depende de $n$ chamadas independentes, e cada uma excede o próprio p99 com probabilidade $0{,}01$, a probabilidade de a requisição esperar ao menos uma chamada lenta é

$$P = 1 - 0{,}99^{\,n}$$

Com $n = 100$, $P \approx 0{,}63$: o que era o caso de 1% de cada componente vira o caso comum do serviço (DEAN; BARROSO, 2013).

Percentis têm uma limitação operacional: **não se agregam**. A média dos p99 de dez instâncias não é o p99 do conjunto. Por isso a latência é registrada como histograma: os contadores por *bucket* se somam entre instâncias e intervalos, e o percentil é estimado depois, sobre o histograma somado, com erro limitado pela largura do *bucket*.

### 7. Cardinalidade

A **cardinalidade** de uma métrica é o número de séries temporais distintas que ela gera, dado pelas combinações de valores dos atributos. Com atributos $a_1, \dots, a_k$ e $|a_i|$ valores distintos cada, o limite superior é

$$S \le \prod_{i=1}^{k} |a_i|$$

e, para um histograma com $b$ *buckets*, multiplica-se ainda por $b$. Cada série ocupa memória no coletor e armazenamento e índice no *backend*; o custo de uma plataforma de métricas cresce com $S$, não com o número de requisições. Um atributo de alta cardinalidade — identificador de usuário, de pedido, URL com parâmetro — multiplica $S$ pelo número de valores dele e é a causa usual de explosão de custo em sistemas de métricas como o Prometheus (PROMETHEUS, s.d.).

A cardinalidade tem sentidos opostos conforme o sinal. Em **métricas**, é custo, e se controla limitando atributos a domínios pequenos — método, rota, classe de *status*. Em ***traces* e eventos**, é o que permite a observabilidade da §3: o identificador do cliente num *span* não gera série nova, e é ele que permite perguntar *quais clientes estão lentos* (MAJORS; FONG-JONES; MIRANDA, 2022). A regra prática decorre daí: atributos de alta cardinalidade vão no *span*; a métrica guarda só dimensões de domínio pequeno.

### 8. OpenTelemetry: origem e escopo

O **OpenTelemetry** (OTel) é um projeto de código aberto da CNCF (*Cloud Native Computing Foundation*) que padroniza a geração, a coleta e a exportação de telemetria — *traces*, métricas e *logs* — de forma independente de fornecedor (OPENTELEMETRY, s.d.). Nasceu em 2019 da fusão de dois projetos concorrentes: o **OpenTracing**, da CNCF, que padronizava uma API de *tracing* sem implementação, e o **OpenCensus**, originado no Google, que oferecia bibliotecas de *tracing* e métricas com implementação própria. A coexistência obrigava autores de bibliotecas a escolher um dos dois; a fusão produziu um padrão único, e os dois projetos de origem foram descontinuados.

O que o OpenTelemetry padroniza:

| Componente | O que é |
|---|---|
| **API** | interfaces para criar *spans*, registrar métricas e emitir *logs*; sem implementação, uma chamada à API é uma operação nula |
| **SDK** | a implementação da API por linguagem: amostragem, processamento em lote, agregação de métricas, exportação |
| **OTLP** (*OpenTelemetry Protocol*) | o protocolo de transporte da telemetria, sobre gRPC ou HTTP, com codificação Protobuf ou JSON |
| **Convenções semânticas** | os nomes e significados padronizados de *spans*, métricas e atributos por domínio — HTTP, banco de dados, mensageria, *runtime* |
| ***Collector*** | um processo independente que recebe, processa e reexporta telemetria |

A separação entre API e SDK tem um propósito: uma biblioteca de terceiros se instrumenta contra a API, sem impor SDK nem destino à aplicação que a usa; a aplicação escolhe o SDK e o destino.

O que o OpenTelemetry **não é**: não é *backend*, não armazena telemetria, não oferece painel nem consulta nem alerta. Ele termina no envio. Armazenar, consultar e exibir são papel das plataformas da §11.

### 9. *Trace*, *span*, contexto e propagação

Um **trace** é o conjunto de *spans* de uma requisição, organizado em árvore pela relação de pai e filho e identificado por um `trace_id` de 16 *bytes*. Um **span** é uma operação dessa requisição e contém: nome, `span_id` de 8 *bytes*, `span_id` do pai (ausente na raiz), instantes de início e fim, atributos, eventos com instante, *links* para outros *spans*, um *status* (`Unset`, `Ok` ou `Error`) e um **tipo** (*span kind*) — `SERVER` e `CLIENT` para as duas pontas de uma chamada síncrona, `PRODUCER` e `CONSUMER` para as de uma mensagem assíncrona, `INTERNAL` para operação local.

O **contexto** (*span context*) é o subconjunto do *span* necessário para criar filhos dele: `trace_id`, `span_id` e as opções de amostragem. Dentro de um processo, o contexto corrente é mantido pela biblioteca (por *thread* ou por tarefa assíncrona), e um *span* criado sem pai explícito se torna filho do corrente.

Entre processos, o contexto precisa viajar com a requisição. **Propagação** é a injeção do contexto no meio de transporte pelo emissor e a extração pelo receptor. Em HTTP, o formato padrão é o cabeçalho `traceparent` da especificação **W3C Trace Context** (W3C, 2021):

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             ^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ ^^^^^^^^^^^^^^^^ ^^
          versão          trace_id                 span_id do pai  flags (01 = amostrado)
```

O receptor lê o cabeçalho e cria o seu *span* `SERVER` como filho do *span* `CLIENT` do emissor. Sem propagação, cada serviço produz um *trace* isolado, e a requisição deixa de ser reconstruível ponta a ponta. A mesma especificação define o cabeçalho `tracestate`, para dados de fornecedor; o OpenTelemetry acrescenta o ***baggage***, pares chave-valor de aplicação propagados junto com o contexto.

### 10. *Resource*, exportadores, *Collector* e amostragem

O ***resource*** é o conjunto de atributos que identifica a entidade que emitiu a telemetria, e não a operação: `service.name`, `service.version`, o ambiente de implantação, o *host*, o *container*. É declarado uma vez, na inicialização do SDK, e anexado a tudo o que o processo emite.

O **exportador** (*exporter*) é o componente do SDK que serializa a telemetria e a envia a um destino — por OTLP, para o console, ou num formato de fornecedor. O envio usual é em lote e em segundo plano: o processador de lote (*batch processor*) acumula *spans* e os despacha periodicamente, sem bloquear a aplicação. O destino é configurado por variáveis de ambiente padronizadas (`OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_HEADERS`, `OTEL_SERVICE_NAME`), o que mantém o código independente da plataforma.

O ***Collector*** é opcional e fica entre a aplicação e o *backend*, organizado em *pipelines* de três estágios:

```
  aplicações ──OTLP──► [ receivers ] ─► [ processors ] ─► [ exporters ] ──► backends
                        OTLP, Prometheus,  lote, filtro,     OTLP, Prometheus,
                        logs de arquivo    remoção de dados   formato de fornecedor
                                           sensíveis, amostragem
```

Ele permite trocar ou duplicar o destino sem tocar nas aplicações, remover atributos sensíveis antes de a telemetria sair da rede e concentrar credenciais num ponto só.

**Amostragem** (*sampling*) reduz o volume de *traces* guardados. Na **amostragem na cabeça** (*head sampling*), a decisão é tomada no início do *trace*, em geral por probabilidade sobre o `trace_id`, e propagada nas *flags*: é barata, mas cega ao desfecho. Na **amostragem na cauda** (*tail sampling*), feita no *Collector*, a decisão espera o *trace* terminar e pode guardar todos os *traces* com erro ou lentos e só uma fração dos normais, ao custo de manter *spans* em memória até a decisão.

### 11. O ecossistema de *backends*

Como o OpenTelemetry termina no envio, o armazenamento e a consulta ficam com plataformas que recebem OTLP, diretamente ou via *Collector*. Exemplos de código aberto:

| Plataforma | Sinal | Observação |
|---|---|---|
| **Prometheus** | métricas | projeto da CNCF; modelo de coleta por *pull* e linguagem de consulta PromQL |
| **Jaeger** | *traces* | projeto da CNCF, originado na Uber |
| **Grafana Tempo** | *traces* | armazenamento de *traces* da Grafana Labs |
| **Grafana Loki** e **Mimir** | *logs* e métricas | da mesma família, consultados no Grafana |
| **Grafana** | visualização | painel e alerta sobre várias fontes; não armazena |

Há também plataformas comerciais hospedadas que aceitam OTLP, entre elas o Grafana Cloud, que reúne Tempo, Loki e Mimir num serviço. A escolha entre elas é decisão de operação, não de instrumentação: o mesmo processo instrumentado exporta para qualquer uma, trocando o endereço do exportador.

### 12. Instrumentação manual e automática, e as convenções semânticas

**Instrumentação** é o código que gera telemetria. Há duas formas:

- **Automática** (*zero-code*): um agente ou *wrapper* modifica, em tempo de carga, as bibliotecas conhecidas — *frameworks* HTTP, clientes de banco, filas — para emitir *spans* e métricas. Em Python, o comando `opentelemetry-instrument` executa a aplicação com essas bibliotecas instrumentadas; em Java, um *agent* da JVM faz o mesmo. Cobre a fronteira do processo — o que entra e o que sai — sem alterar o código da aplicação, e não conhece o domínio.
- **Manual** (*code-based*): a aplicação chama a API para criar *spans* e registrar atributos nas operações de domínio — *autorizar pagamento*, *consultar antifraude*. Conhece o domínio e exige código.

As duas convivem: a automática dá o esqueleto (a requisição HTTP e a consulta ao banco), e a manual acrescenta os *spans* e atributos que só a aplicação conhece, como filhos dos automáticos, porque o contexto corrente é o mesmo.

As **convenções semânticas** fixam o nome e o significado dos atributos que as duas formas emitem. Para HTTP, o método é `http.request.method`, o código de resposta é `http.response.status_code`, a rota é `http.route`, e a duração no servidor é o histograma `http.server.request.duration`, em segundos (OPENTELEMETRY, s.d.). Uma plataforma que conhece a convenção monta o painel RED de qualquer serviço HTTP sem configuração por serviço. As convenções existem por domínio — HTTP, RPC, banco de dados, mensageria, *runtime* de linguagem, sistemas de IA generativa —, e cada domínio fixa os *spans*, os atributos e as métricas próprios dele.

---

## Exemplos

Os números desta seção são **ilustrativos**, construídos para o exemplo; não vêm de medição.

### Exemplo 1 — Uma API de pagamentos, pelos três métodos

Serviço `api-pagamentos`, com o endpoint `POST /pagamentos` que consulta um serviço de antifraude e grava num banco relacional.

```
  RED — por serviço (POST /pagamentos, última hora)
      rate ........... 420 req/s
      errors ......... 0,8 %
      duration ....... p50 85 ms   p95 210 ms   p99 1,9 s

  USE — por recurso
      CPU ................ utilização 45 %     saturação: fila de execução ~0
      pool de conexões ... utilização 96 %     saturação: 35 espera por conexão/s
      disco do banco ..... utilização 30 %     erros 0
```

O p99 de 1,9 s contra um p95 de 210 ms indica uma cauda separada do corpo. O RED localiza o sintoma no serviço; o USE aponta o *pool* de conexões a 96%, com requisições esperando conexão — a causa provável. Um alerta por sintoma dispararia sobre o p99 em relação ao SLO; o *pool* entraria no diagnóstico, não no alerta.

### Exemplo 2 — A média e os percentis

Mil requisições: 980 levam 80 ms e 20 levam 3 s.

```
  média = (980 × 80 + 20 × 3000) / 1000 = 138,4 ms
  p50   =  80 ms
  p95   =  80 ms
  p99   = 3000 ms
```

A média de 138 ms não descreve nenhuma requisição: as 980 rápidas levaram menos, as 20 lentas levaram vinte vezes mais. Um SLO formulado sobre a média ficaria satisfeito com 2% das requisições a 3 s; formulado sobre o p99, acusa o problema.

### Exemplo 3 — Um *trace* de pagamento

```
trace_id 4bf92f3577b34da6a3ce929d0e0e4736                         duração
POST /pagamentos                     SERVER  api-pagamentos         1 870 ms
├── autorizar_pagamento              INTERNAL                       1 862 ms
│   ├── POST /antifraude/avaliar     CLIENT                            62 ms
│   │   └── POST /antifraude/avaliar SERVER  antifraude                55 ms
│   ├── aguardar conexão do pool     INTERNAL                       1 740 ms   <- 93% do tempo
│   └── INSERT pagamentos            CLIENT  postgresql                41 ms
```

A requisição está no p99 do Exemplo 1, e o *trace* mostra onde o tempo foi gasto: na espera por conexão, não no antifraude nem na consulta. O *span* `SERVER` do antifraude é filho do `CLIENT` de `api-pagamentos` porque o `traceparent` foi propagado na chamada HTTP; sem a propagação, ele estaria num *trace* separado.

### Exemplo 4 — Cardinalidade de um histograma

Métrica `http.server.request.duration`, histograma com 15 *buckets*.

```
  atributos                                    séries (limite superior)
  método (5) × rota (40) × status (10)         5 × 40 × 10 × 15  =        30 000
  + cliente_id (200 000 clientes)              30 000 × 200 000  = 6 000 000 000
```

O atributo `cliente_id` multiplica o número de séries por duzentos mil. No *span*, o mesmo atributo não cria série nenhuma e permite filtrar os *traces* lentos de um cliente.

### Exemplo 5 — Instrumentação manual em Python

```python
from opentelemetry import trace, metrics
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.trace import Status, StatusCode

# resource: quem emite; exportador: destino lido de OTEL_EXPORTER_OTLP_ENDPOINT
resource = Resource.create({"service.name": "api-pagamentos", "service.version": "2.3.0"})
provider = TracerProvider(resource=resource)
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer("pagamentos")
duracao = metrics.get_meter("pagamentos").create_histogram("pagamento.autorizacao.duracao", unit="s")

def autorizar(pedido):
    with tracer.start_as_current_span("autorizar_pagamento") as span:
        span.set_attribute("pagamento.metodo", pedido.metodo)       # domínio pequeno
        span.set_attribute("pagamento.cliente_id", pedido.cliente)  # alta cardinalidade: só no span
        try:
            resultado = processar(pedido)
        except Exception as erro:
            span.record_exception(erro)
            span.set_status(Status(StatusCode.ERROR))
            raise
        duracao.record(resultado.segundos, {"pagamento.metodo": pedido.metodo})
        return resultado
```

O código não menciona a plataforma: o endereço e a credencial vêm de variáveis de ambiente. Com instrumentação automática do *framework* HTTP, o *span* `autorizar_pagamento` nasce como filho do *span* `SERVER` da requisição, como no Exemplo 3. A métrica recebe só o método de pagamento; o identificador do cliente fica no *span*, pela razão do Exemplo 4. O provedor de métricas do SDK é configurado de forma análoga ao de *traces* e foi omitido.

---

## Fontes e leituras

BEYER, B.; JONES, C.; PETOFF, J.; MURPHY, N. R. (org.). **Site reliability engineering**: how Google runs production systems. Sebastopol: O'Reilly Media, 2016. Cap. 4: Service level objectives; cap. 6: Monitoring distributed systems.

DEAN, J.; BARROSO, L. A. The tail at scale. **Communications of the ACM**, v. 56, n. 2, 2013.

GREGG, B. Thinking methodically about performance. **Communications of the ACM**, v. 56, n. 2, 2013. Ver também: GREGG, B. **The USE method**. Disponível em: https://www.brendangregg.com/usemethod.html.

MAJORS, C.; FONG-JONES, L.; MIRANDA, G. **Observability engineering**: achieving production excellence. Sebastopol: O'Reilly Media, 2022.

OPENTELEMETRY. **OpenTelemetry documentation**. Disponível em: https://opentelemetry.io/docs/. — Conceitos (sinais, contexto e propagação, *resource*, amostragem), *Collector* e instrumentação *zero-code*.

OPENTELEMETRY. **OpenTelemetry semantic conventions**. Disponível em: https://opentelemetry.io/docs/specs/semconv/. — Convenções para HTTP e demais domínios.

PROMETHEUS. **Prometheus documentation**: instrumentation; metric and label naming. Disponível em: https://prometheus.io/docs/. — Recomendações sobre cardinalidade de *labels*.

SIGELMAN, B. H. et al. **Dapper, a large-scale distributed systems tracing infrastructure**. Google Technical Report dapper-2010-1, 2010.

W3C. **Trace Context**. W3C Recommendation, 2021. Disponível em: https://www.w3.org/TR/trace-context/.

WILKIE, T. **The RED method**: how to instrument your services. Grafana Labs Blog, 2018.

**Material da disciplina.** [O plano de observabilidade](02-plano-de-observabilidade.md) — o método que deriva perguntas, modos de falha, objetivos e sinais da arquitetura do sistema; [Observabilidade](03-observabilidade.md) — as convenções GenAI aplicadas ao agente, a exportação ao Grafana Cloud e as métricas de operação de um agente.
