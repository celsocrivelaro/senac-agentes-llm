# IA Aplicada com LLMs — Aula 15: Produção — Observabilidade

---

## Objetivos de aprendizagem

Ao final desta nota, espera-se que o aluno seja capaz de:

- **Instrumentar** um agente segundo as convenções semânticas GenAI do OpenTelemetry.
- **Justificar** o uso de convenção aberta em lugar de formato próprio ou de fornecedor.
- **Definir** as quatro métricas de operação e **lê-las** no painel da plataforma que recebe os *spans*.
- **Decompor** as métricas por ferramenta, por versão e por agente.
- **Definir** um alerta com limiar justificado, e reconhecer o sintoma de degradação silenciosa.
- **Distinguir** o que o painel mede — latência, custo, término — do que ele não mede: se a resposta está certa; e **medir o acerto** por avaliação online, com o juiz validado na aula de *evals*.

---

## Desenvolvimento teórico

### 1. O que já existe

Esta nota implementa o plano de observabilidade do auditor de despesas, derivado na nota anterior (Exemplos 1 e 2): os sinais, os atributos e os cortes que ali resultaram das perguntas, dos modos de falha e dos objetivos.

O *trace* modelado em arquitetura de agentes registra, por execução: cada passo com ferramenta e argumentos, tokens consumidos, custo estimado e o **motivo de término**. Não é preciso instrumentar do zero — é preciso traduzir.

### 2. Por que uma convenção, e não o formato próprio

As **convenções semânticas GenAI do OpenTelemetry** padronizam nomes de *span* e atributos para sistemas com LLM:

```
invoke_agent  "auditor-de-despesas"                    <- raiz
├── chat  "mistral-small-latest"
│     gen_ai.request.model        = "mistral-small-latest"
│     gen_ai.usage.input_tokens   = 1842
│     gen_ai.usage.output_tokens  = 96
├── execute_tool  "consultar_politica"
│     gen_ai.tool.name            = "consultar_politica"
├── execute_tool  "buscar_regulamento"
└── chat  "mistral-small-latest"
```

O ganho não é estético. É **poder trocar de ferramenta de observabilidade sem reinstrumentar** — e o custo de não poder trocar apareceu duas vezes nas aulas anteriores: quando a API do framework se moveu (na aula de *frameworks*) e quando o protocolo foi revisado cinco vezes em vinte meses (na aula de MCP).

Há um segundo ganho, menos evidente e mais relevante em equipe: a convenção **obriga a nomear**. Um campo chamado `gen_ai.usage.input_tokens` significa a mesma coisa em todo lugar; um campo chamado `tokens` significa o que o autor quis, e ninguém mais sabe se inclui a saída.

### 3. Exportar

Os *spans* saem do processo pelo **OTLP** (*OpenTelemetry Protocol*), sobre HTTP, para a plataforma de observabilidade — na aula, o Grafana Cloud. O envio é em lote e em segundo plano: o agente não espera a plataforma responder. Junto dos *spans* vai o *resource*, que identifica quem os emitiu: o nome do serviço e a versão implantada.

O destino e a credencial são variáveis de ambiente (`OTEL_EXPORTER_OTLP_ENDPOINT`, `OTEL_EXPORTER_OTLP_HEADERS`), lidas pelo exportador. O código não cita a plataforma: trocá-la é trocar a configuração, e é a convenção da §2 que garante que a plataforma nova entende os mesmos atributos.

### 4. As quatro perguntas que o painel precisa responder

Estas são as métricas a que o plano do auditor chega: a latência e o custo vêm dos SLOs; a distribuição dos motivos de término e a taxa de erro por ferramenta, dos modos de falha. Não constituem lista universal — outra arquitetura, com outros modos de falha, chega a outra lista (nota sobre o plano de observabilidade, Exemplo 3). Nenhuma delas é nova: todas saem do *trace* que já existia.

**Latência p50 e p95 por execução.** A média é inútil aqui: distribuições de latência de agente têm cauda longa, porque o número de passos varia. O p95 é o que o usuário insatisfeito sentiu.

**Custo por execução.** A unidade que a aula de escolha e configuração de modelos estabeleceu como a compreendida pela gestão. É também o insumo da conferência da promessa, na nota seguinte.

**Distribuição dos motivos de término.** Esta é a métrica que **quase nenhum painel comercial tem**, e a razão é simples: quase ninguém modela término como dado. Aqui ele foi modelado como dado em arquitetura de agentes, e por isso pode ser medido.

**Taxa de erro por ferramenta.** Erros de domínio não derrubam a execução — voltam como observação e o agente contorna. Por isso não aparecem como falha, e a única forma de vê-los é contá-los.

### 5. A dimensão que falta em quase todo painel

As quatro métricas **agregadas** dizem que algo piorou. Não dizem **onde**. Um **corte** é a decomposição de uma métrica pelos valores de um atributo do *span*; três cortes resolvem a pergunta, e os três atributos já existem como dado:

| Corte | Responde |
|---|---|
| **por ferramenta** | qual integração está falhando |
| **por versão** | a piora começou com qual mudança |
| **por agente** | qual participante errou |

O corte **por versão** é obrigatório: um painel que mistura execuções de versões diferentes num mesmo número está somando sistemas distintos. É o carimbo da aula de *prompt engineering*, aplicado à operação. A versão implantada vem de `VERSAO_SISTEMA`, que o agente lê e grava em todo *trace*; quando o agente roda numa imagem Docker, o natural é que esse número seja o mesmo da *tag* da imagem (nota [De *script* a imagem](../../aula-13-mlops-empacotamento/notas-de-aula/03-de-script-a-imagem.md), §5.1, na aula de MLOps e empacotamento).

O corte **por agente** é, por ora, trivial: há um agente só, e o atributo tem um valor. Ele é registrado desde já como preparação para a aula de sistemas multiagente. Num sistema multiagente, `invoke_agent` deixa de ser um *span* raiz único e passa a ser **um por participante**, e é essa decomposição que responde *qual agente errou* em minutos, em vez de dias. Instrumentar o atributo agora custa uma linha; acrescentá-lo depois exige reinstrumentar.

### 6. O alerta

Painel se olha quando alguém decide olhar. Alerta acorda. A pergunta, portanto, não é "o que é interessante", e sim **o que justifica interromper alguém**.

**Custo por hora acima do teto.** Direto, e o mais fácil de justificar: um laço não detectado ou um pico de tráfego aparecem aqui antes de aparecerem na fatura.

**Proporção de `orcamento_esgotado` em alta.** Este é o alerta que a maioria dos painéis não oferece, porque depende de término modelado como dado, e é o mais informativo dos dois. A razão:

> Execuções que terminam por orçamento **não são erro**. Elas devolvem `200`, com uma resposta pior. A taxa de erro não se mexe. A satisfação cai sem que nada acuse.

É o sintoma de **degradação silenciosa** — o corpus cresceu, o prompt ficou mais longo, uma ferramenta passou a devolver mais dado, e o agente deixou de caber no orçamento. Sem essa métrica, o defeito só aparece na reclamação.

### 7. O que o painel não diz, e a avaliação online

As quatro métricas medem **como** o serviço respondeu: em quanto tempo, a que custo, por qual motivo terminou, com quantas falhas de ferramenta. Nenhuma delas mede **se a resposta está certa**. Uma execução que termina em `respondeu`, em 4 s e por R$ 0,03, com a conclusão errada sobre a despesa, entra no painel como execução saudável.

A lacuna tem uma consequência operacional imediata. O *prompt registry* visto em *prompt engineering* permite mudar o prompt **sem *deploy***: a imagem é a mesma, a *tag* é a mesma, e o corte por versão não separa o antes do depois. Se a mudança piorou o acerto sem mexer em latência, custo ou término, o painel não registra diferença nenhuma.

Responder à pergunta *o serviço está acertando?* exige casos com resposta conhecida, um critério de correção e um juiz validado — a suíte construída na aula de *evals*. Ela respondeu à pergunta **antes** do *deploy*, sobre o *golden dataset*. A **avaliação online** a responde **depois**, sobre o tráfego real: uma fração das execuções é amostrada, o mesmo juiz validado é executado sobre a amostra, e a **taxa de acerto amostrada** entra no painel ao lado das quatro métricas de operação, decomposta pelos mesmos cortes.

A avaliação online pega o que o *dataset* não previu, e cada falha encontrada volta a ele como caso permanente — é o ciclo que a aula de *evals* descreveu, agora fechado sobre tráfego real. Uma cautela: o juiz continua valendo só enquanto a concordância medida naquela aula valer — mudou o domínio do tráfego, refaz-se a validação.

---

## Exemplos

### Exemplo 1 — O *trace* de arquitetura de agentes, traduzido

```python
# ANTES — formato próprio, legível só pela equipe que o definiu
{"passo": 3, "ferramenta": "consultar_politica",
 "argumentos": {"categoria": "refeicao"}, "tokens": 1938}

# DEPOIS — convenção
with tracer.start_as_current_span("execute_tool") as span:
    span.set_attribute("gen_ai.tool.name", "consultar_politica")
    span.set_attribute("gen_ai.operation.name", "execute_tool")
    resultado = ferramenta(**argumentos)
```

O dado é o mesmo; a diferença está em quais ferramentas conseguem interpretá-lo sem documentação adicional.

### Exemplo 2 — As quatro métricas, e o que cada uma acusa

```
  latência p50 ............ 4,2 s
  latência p95 ........... 38,1 s      <- cauda longa: passos variam
  custo por execução ...... R$ 0,041
  taxa de erro por ferramenta:
      buscar_regulamento ... 22%        <- integração degradando
      consultar_politica ....  1%
  motivos de término:
      respondeu ............ 78%
      orcamento_esgotado ... 19%        <- degradação SILENCIOSA
      laco_detectado ........  3%
```

Dezenove por cento das execuções devolveram `200` com resposta pior, e nenhum monitor de erro registraria a degradação.

### Exemplo 3 — Decomposição por versão

```
                        v1.4      v1.5     delta
  custo/execução      R$0,031   R$0,047    +52%
  p95                    22 s      38 s     +73%
  orcamento_esgotado       6%       19%     +13 p.p.
```

A v1.5 acrescentou uma ferramenta. O painel agregado mostraria uma média entre as duas versões, e o efeito ficaria diluído — que é exatamente o que o carimbo existe para impedir. Com a *tag* da imagem como versão, cada linha da tabela corresponde a uma imagem implantada.

---

## Fontes e leituras

OPENTELEMETRY. **Semantic conventions for generative AI systems**.

OPENTELEMETRY. **AI agent observability**: evolving standards and best practices. 2025.

GRAFANA LABS. **Grafana Cloud documentation**: send OpenTelemetry data via OTLP.

**Material da disciplina.** Nesta aula, [O plano de observabilidade](02-plano-de-observabilidade.md) — o plano do auditor, que esta nota implementa. [Custo, latência e a decisão](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md) — custo e latência como unidades de gestão. [O prompt como código](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) — o *prompt registry* e o carimbo. [Confiabilidade](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md), em arquitetura de agentes — o *trace* e a taxonomia de término. [O juiz, e a suíte](../../aula-11-evals/notas-de-aula/03-o-juiz-e-a-suite.md), em *evals* — a suíte e o juiz, que respondem à pergunta que o painel não responde. [Como isso falha](../../aula-17-sistemas-multiagente/notas-de-aula/03-como-isso-falha.md), em sistemas multiagente — o *trace* por agente num sistema multiagente.
