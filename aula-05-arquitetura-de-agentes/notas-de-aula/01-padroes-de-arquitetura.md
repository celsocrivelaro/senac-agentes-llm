# IA Aplicada com LLMs — Aula 05: Arquitetura de agentes — Padrões de arquitetura

## Introdução

Você tem um laço que funciona. Ele não está pronto para rodar sozinho, e são três motivos — que são as três notas seguintes desta aula:

1. **Não sabe parar.** `max_passos=6` é um teto arbitrário, não um orçamento.
2. **Não sabe falhar.** Uma ferramenta que estoura derruba o programa.
3. **Não sabe esquecer.** O histórico só cresce.

Antes dos três, uma quarta pergunta — a desta nota:

> **Ele precisava ser um laço?**

Na aula passada você leu uma dúzia de casos perguntando *"que padrão é este?"* e respondeu com nomes: roteador, orquestrador-trabalhador, avaliador-otimizador. Hoje os nomes viram código, com custo em chamadas e com a metade que o material de mercado nunca traz: **quando não usar**.

Você também saiu de lá com um case escolhido. A pergunta útil em cada padrão não é *"entendi?"*, é **"é este que o meu case precisa?"**.

> **Pré-requisitos:** Aula 04, [nota 01](../../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/01-o-que-e-um-agente.md) §2.1 e §4–§8 · Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) §6 e §9 · Aula 01, [nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) §1 · Aula 02, [nota 02](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md) §7 e [nota 04](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md).
>
> **Código:** [`aula05-agentes/`](https://github.com/celsocrivelaro/senac-llm-code/tree/main/aula05-agentes) — os esqueletos desta nota rodam lá em versão completa.

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Decidir** entre workflow e agente com três perguntas operacionais, em vez de intuição.
- **Implementar** os cinco padrões de workflow: *chaining* com portão, roteador, paralelização (*sectioning* e *voting*), orquestrador-trabalhador e avaliador-otimizador.
- **Dizer, para cada padrão, quando ele não serve.**
- **Estimar o custo em chamadas** antes de escrever a primeira linha.
- **Identificar workflow disfarçado de agente** em código existente — inclusive no seu.

---

## Desenvolvimento teórico

### 1. A linha divisória, agora operacional

A Aula 01 (nota 03, §1.1) definiu: **workflow** é o caminho no código; **agente** é o caminho decidido em execução. Definição serve para prova. Para decidir às três da tarde, com requisito na mesa, serve uma pergunta:

> **Você consegue desenhar o fluxograma antes de rodar?**

**Sim** → desenhe, e depois escreva o fluxograma em Python. Mais barato, mais rápido, testável e explicável quando der errado.
**Não**, porque o próximo passo depende do que o anterior descobrir → aí existe motivo para autonomia.

Três perguntas de apoio quando não for óbvio:

| Pergunta | Sim | Não |
|---|---|---|
| **(a)** O número de passos é conhecido antes de rodar? | workflow | agente |
| **(b)** A ordem depende de dados que só existem em execução? | agente | workflow |
| **(c)** O erro é caro ou irreversível? | **reduza a autonomia**, adicione confirmação | autonomia é tolerável |

A (c) não é sobre a tarefa, é sobre a **consequência**. Tarefa que tecnicamente pediria agente, mas cujo erro emite dinheiro, é tarefa onde você aceita um sistema pior para ter um sistema previsível.

#### 1.1 O preço de subir no espectro

O espectro de seis níveis da Aula 01 (nota 03, §1.4) não detalhou quanto custa subir:

| Nível | Chamadas | Previsível? | Testável? | Depurável ao falhar? |
|---|---|---|---|---|
| Prompt único | 1 | sim | trivialmente | sim |
| Chain | N fixo | sim | etapa por etapa | sim |
| Roteador | 1 + rota | sim | por rota | sim |
| Workflow com ferramentas | N fixo | sim | sim | sim |
| **Agente** | **desconhecido** | **não** | só por propriedade | **só com log de trajetória** |
| Multi-agente | desconhecido × agentes | não | difícil | muito difícil |

A última coluna é a que dói: num agente a trajetória muda a cada execução, e sem log estruturado você tem uma saída errada e nenhuma pista. É por isso que a [nota 03](03-confiabilidade.md) insiste em registrar o estado.

> **A regra da Aula 01, agora com fundamento:** use a menor autonomia que resolve. Não por conservadorismo — cada nível acima custa mais dinheiro, mais latência e mais tempo seu quando algo der errado.

---

### 2. Autópsia do exercício 03

O melhor exemplo de workflow disfarçado de agente foi escrito por você, semana passada. Classifique as cinco etapas daquele atendimento:

| Etapa | Workflow ou agente? |
|---|---|
| 1. Coleta (conversar até ter pedido e problema) | **workflow** — a parada é "tenho os dois campos", isso é um `if` |
| 2. `consultar_pedido(id)` | **workflow** — sempre acontece, sempre nessa posição |
| 3. Raciocínio (previsão × hoje, cliente × sistema) | **decisão do modelo** — mas de *uma* decisão, não de uma sequência |
| 4. `abrir_chamado(...)` com confirmação | **workflow condicional** |
| 5. `consultar_chamado(id)` | **workflow** — sempre, e sempre por último |

Quatro das cinco estavam no código, e a sequência estava numerada de 1 a 5 no enunciado. **Era um workflow com dois momentos de decisão**, escrito como agente.

Erro? Não — o objetivo era o *mecanismo*. Mas em produção a versão workflow ganharia em tudo: menos chamadas, sem risco de pular a consulta final ou de abrir dois chamados, testável etapa por etapa. Perderia só o atendimento fora do roteiro — e é aí que se decide.

> **O padrão a reconhecer:** quando o enunciado vem numerado, o código também deveria vir.

---

### 3. Os cinco padrões

Taxonomia de *Building Effective Agents* (Anthropic, 2024). Todos os cinco são **workflows** — o caminho está no seu código. O agente é assunto da [nota 02](02-o-agente-e-o-estado.md).

---

#### 3.1 Prompt chaining com portão

Chamadas em sequência fixa, com **validação entre elas**.

```
   entrada ──> [ LLM 1 ] ──> portão ──> [ LLM 2 ] ──> saída
                              │
                              └── falhou ──> tratamento (não segue adiante)
```

```python
def extrair_e_formatar(texto: str) -> dict:
    bruto = chamar(PROMPT_EXTRACAO, texto, schema=SCHEMA_EXTRACAO)

    if not bruto.get("valor") or bruto["valor"] <= 0:   # O PORTÃO:
        raise ValorInvalido(bruto)                      # código, não modelo
    if bruto["data"] > date.today():
        raise DataFutura(bruto)

    return chamar(PROMPT_PARECER, bruto, schema=SCHEMA_PARECER)
```

O *chaining* você já viu na Aula 03 como técnica de prompt; o que muda aqui é o **portão**. Regra expressável em Python vai em Python: mais barata, mais confiável, e não alucina.

| Usar quando | **Não** usar quando |
|---|---|
| etapas conhecidas, cada uma melhor sozinha: extrair → validar → formatar, traduzir → revisar | as etapas não são independentes — cada elo perde informação |
| a falha intermediária precisa parar o processo | o número de etapas depende da entrada (aí é agente) |

---

#### 3.2 Roteador

O modelo **classifica**; o seu código **despacha**.

```
                    ┌──> rota A: código puro (sem LLM)
   entrada ──> [ classificador ] ──> rota B: outro prompt / outro modelo
                    │              └──> rota C: humano
                    └──> nenhuma das anteriores ──> fila de revisão
```

Tecnicamente é **saída estruturada com um enum** — o mecanismo da Aula 02 (nota 02, §7) decidindo o fluxo do programa:

```python
ROTAS = ["dentro_da_politica", "ambiguo", "acima_da_alcada", "nenhuma"]

def processar(item):
    r = classificar(item, schema_com_enum(ROTAS))   # 1 chamada, barata
    if r["rota"] == "dentro_da_politica":
        return aprovar_por_regra(item)              # ZERO chamadas de LLM
    if r["rota"] == "ambiguo":
        return agente_analisa(item)                 # caro, e raro
    if r["rota"] == "acima_da_alcada":
        return encaminhar_humano(item, r["justificativa"])
    return fila_de_revisao(item)                    # "nenhuma": não invente
```

Dois pontos separam um roteador bom de um ruim:

- **A rota `nenhuma` é obrigatória.** Sem ela o modelo escolhe entre opções que não servem — e escolhe com confiança. A rota de escape troca um erro silencioso por uma fila de revisão.
- **A rota mais valiosa é a que não chama o modelo.** Se 70% dos itens caem em `aprovar_por_regra`, você cortou 70% do custo sem perder nada. É a otimização mais rentável desta aula.

| Usar quando | **Não** usar quando |
|---|---|
| entradas heterogêneas com tratamentos distintos; triagem | as rotas fazem quase a mesma coisa — junte num prompt só |
| escolher modelo pequeno para o fácil, grande para o difícil | a classificação é tão confiável em `if` quanto no modelo |

Roda em [`00-roteador.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/00-roteador.py), que imprime **quantos itens do lote nem chegaram a precisar de LLM**.

---

#### 3.3 Paralelização: duas coisas com o mesmo nome

**Sectioning** — subtarefas independentes, ao mesmo tempo. **Você** conhece as seções: elas estão no seu código.

```
                 ┌──> [ LLM: seção 1 ] ──┐
   entrada ──────┼──> [ LLM: seção 2 ] ──┼──> juntar ──> saída
                 └──> [ LLM: seção 3 ] ──┘
```

**Voting** — a mesma tarefa, N vezes, decisão por maioria. É a *self-consistency* da Aula 03 promovida de técnica de prompt a padrão de arquitetura: lá você passava `n=5` na chamada; aqui **você** orquestra N chamadas e pode variar prompt, modelo ou temperatura entre elas.

```
                 ┌──> [ LLM ] ──┐
   entrada ──────┼──> [ LLM ] ──┼──> voto majoritário ──> saída
                 └──> [ LLM ] ──┘
```

```python
def votar(item, n=3):
    vereditos = [classificar(item) for _ in range(n)]      # ou em paralelo
    rota, votos = Counter(v["rota"] for v in vereditos).most_common(1)[0]
    if votos < n // 2 + 1:
        return {"rota": "nenhuma", "motivo": "sem maioria"}   # divergência é sinal
    return {"rota": rota, "confianca": votos / n}
```

O detalhe que quase todo mundo joga fora: **a divergência é informação**. Três vereditos diferentes não querem dizer "escolha um" — querem dizer que o caso é difícil, e provavelmente é caso de humano.

| Usar quando | **Não** usar quando |
|---|---|
| *sectioning*: a tarefa se decompõe e as partes não dependem umas das outras; latência importa | *sectioning*: as partes dependem umas das outras — você concatena pedaços que se contradizem |
| *voting*: o erro custa mais que 3 chamadas, e a resposta é discreta (aprovar/reprovar, classificar) | *voting*: tarefa aberta ("escreva um resumo") — não há maioria entre três textos, e a conta triplicou à toa |

---

#### 3.4 Orquestrador-trabalhador

Um modelo **decide quais são as subtarefas**; trabalhadores executam; um sintetizador junta.

```
                    ┌──> [ trabalhador ] ──┐
   entrada ──> [ orquestrador ] ──> ... ───┼──> [ sintetizador ] ──> saída
                    └──> [ trabalhador ] ──┘
                     ↑
              decide quantas e quais
```

```python
def processar_lote(lote: list) -> dict:
    plano = chamar(PROMPT_ORQUESTRADOR, lote, schema=SCHEMA_PLANO)
    # plano["subtarefas"] só existe agora — não estava no código

    if len(plano["subtarefas"]) > MAX_SUBTAREFAS:      # orçamento, já aqui
        raise PlanoGrandeDemais(len(plano["subtarefas"]))

    resultados = [executar_subtarefa(s) for s in plano["subtarefas"]]
    return chamar(PROMPT_SINTESE, resultados, schema=SCHEMA_PARECER)
```

A diferença para o *sectioning* é uma só, e é toda a diferença: lá as seções estão no seu código; aqui são decididas em execução, e você não sabe se serão duas ou onze. Daí o `if` — primeiro padrão da nota com autonomia real, primeiro que precisa de teto. **Orquestrador sem teto é conta aberta assinada por um modelo.**

| Usar quando | **Não** usar quando |
|---|---|
| a decomposição depende do conteúdo: mudança que toca N arquivos imprevisíveis, pesquisa com N consultas | você consegue listar as subtarefas — isso é *sectioning*, mais barato e sem surpresa |

Roda em [`01-orquestrador-trabalhador.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/01-orquestrador-trabalhador.py).

---

#### 3.5 Avaliador-otimizador (reflexão)

Um laço curto de crítica: gerar → avaliar contra critério → revisar.

```
   entrada ──> [ gerador ] ──> candidato ──> [ avaliador ]
                    ↑                             │
                    └───── crítica ───────────────┤ reprovou
                                                  └─ aprovou ──> saída
```

```python
def gerar_com_revisao(entrada, max_rodadas=3):
    critica = None
    for rodada in range(max_rodadas):
        candidato = chamar(PROMPT_GERADOR, entrada, critica=critica)
        aval = chamar(PROMPT_AVALIADOR, candidato, schema=SCHEMA_AVALIACAO)
        if aval["aprovado"]:
            return candidato, "aprovado", rodada + 1
        critica = aval["o_que_corrigir"]
    return candidato, "teto_de_rodadas", max_rodadas    # melhor esforço, e AVISA
```

Duas armadilhas, e as duas aparecem sempre:

- **Sem critério escrito, o avaliador vira elogio.** O critério precisa ser lista verificável — *cita a regra? menciona o valor exato? conclui?* — e o schema deve forçar resposta item a item. `{"nota": 8}` não serve para nada; `{"cita_regra": false, "o_que_corrigir": "..."}` serve.
- **Sem teto, o par oscila.** O avaliador pede A, o gerador entrega A e perde B, o avaliador pede B. Três rodadas costumam bastar — o importante é o retorno **dizer** que saiu por teto. Melhor esforço apresentado como aprovado é pior que reprovação honesta.

A formalização é o **Reflexion** (Shinn et al., 2023): a crítica é guardada e realimenta as tentativas — o que o parâmetro `critica` faz acima.

| Usar quando | **Não** usar quando |
|---|---|
| existe critério claro; a revisão melhora mesmo; 2–3 chamadas extras se pagam | você não consegue escrever o critério — o avaliador certo é uma pessoa |
| texto com requisitos formais, código que passa em teste, tradução com glossário | tarefa subjetiva sem lastro verificável |

Roda em [`02-avaliador-otimizador.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/02-avaliador-otimizador.py), que executa também **sem** critério para mostrar o avaliador virando elogio.

---

### 4. A tabela de decisão

Leia de cima para baixo e **pare no primeiro que resolve**. A ordem vai do mais previsível para o menos.

| Padrão | Resolve | Custo em chamadas | Quando **não** usar |
|---|---|---|---|
| **Prompt único** | tarefa fechada | 1 | a saída precisa de validação que só outra chamada dá |
| **Chaining + portão** | etapas conhecidas, melhores separadas | N fixo | etapas interdependentes; nº de etapas variável |
| **Roteador** | entradas heterogêneas | 1 + rota | rotas quase iguais; classificação trivial em código |
| **Sectioning** | partes independentes conhecidas | 1 por seção (paralelo) | partes que dependem umas das outras |
| **Voting** | erro caro, resposta discreta | N (3–5) | tarefa aberta, sem maioria possível |
| **Orquestrador-trabalhador** | decomposição desconhecida | variável — **exige teto** | você consegue listar as subtarefas |
| **Avaliador-otimizador** | qualidade com critério explícito | 2 por rodada | sem critério escrito |
| **Agente** ([nota 02](02-o-agente-e-o-estado.md)) | sequência imprevisível + feedback do ambiente | desconhecido — **exige orçamento** | quando o fluxograma existe |

---

### 5. Os padrões se compõem

Nenhuma regra diz que você escolhe um. Sistemas reais encaixam vários, cada peça no seu nível:

```
   lote ──> [ ROTEADOR ] ──┬──> regra em código (a maioria dos itens)
                           ├──> [ AGENTE ]  (os ambíguos, poucos)
                           └──> humano      (os caros)
                                  │
                                  ▼
                        [ ORQUESTRADOR-TRABALHADOR ]  (parecer do lote)
                                  │
                                  ▼
                        [ AVALIADOR-OTIMIZADOR ]      (revisar o texto final)
```

Este é, deliberadamente, o desenho do exercício desta aula: a autonomia cara fica confinada ao caminho estreito onde é necessária, e o volume passa por código determinístico.

> **O princípio que resume a nota:** autonomia é recurso escasso, e arquitetura é gastá-la só onde ela compra alguma coisa.

---

## Exemplos

### Exemplo 1 — O mesmo problema, três arquiteturas, três contas

Classificar 1.000 despesas e justificar as que violam a política (15% violam):

| Arquitetura | Conta | Chamadas |
|---|---|---|
| **(a)** Agente para cada despesa | 1.000 × ~4 passos | **~4.000** |
| **(b)** Chain fixa: classificar + justificar se violar | 1.000 + 150 | **1.150** |
| **(c)** Roteador: regra em código resolve 70%, LLM nos 30%, justificativa nos que violam | 0 + 300 + 150 | **450** |

De (a) para (c), quase **nove vezes** menos chamadas — e (c) é *mais* confiável nos 70%, porque regra não alucina. O ganho não veio de prompt melhor nem de modelo melhor. Veio de decidir o que **não** mandar para o modelo.

### Exemplo 2 — O avaliador sem critério

```python
PROMPT_A = "Avalie se o parecer abaixo está bom. Responda aprovado ou reprovado."
# -> "Aprovado. O parecer está claro, bem estruturado e cobre os pontos principais."

PROMPT_B = """Verifique item a item e responda no schema:
- cita_regra: cita o artigo específico da política?
- cita_valor: menciona o valor exato da despesa?
- conclui: termina com APROVADO ou REPROVADO explícito?
- o_que_corrigir: se algum item é falso, o que exatamente falta."""
# -> {"cita_regra": false, "cita_valor": true, "conclui": true,
#     "o_que_corrigir": "não cita qual artigo da política foi violado"}
```

O primeiro aprovou um parecer que não cita a regra — não porque o modelo é ruim, mas porque **ninguém disse o que era estar bom**. A qualidade do avaliador é a qualidade do critério; o modelo só executa.

### Exemplo 3 — A rota de escape, ligada e desligada

Item: *"Uber R$ 340,00 — funcionário de outra filial, sem centro de custo."* Nenhuma das três rotas previstas serve.

```python
# Sem escape: enum = ["dentro_da_politica", "ambiguo", "acima_da_alcada"]
# -> {"rota": "ambiguo", "justificativa": "requer análise"}   # foi para o agente
#    o agente gastou 6 passos e concluiu "não foi possível determinar"

# Com escape: enum = [..., "nenhuma"]
# -> {"rota": "nenhuma", "justificativa": "centro de custo ausente"}
#    foi para a fila de revisão em 1 chamada
```

Mesma incerteza, dois destinos. Sem a rota de escape a incerteza vira **gasto**; com ela, vira **fila**. O modelo não errou nas duas vezes — na primeira, ele não tinha como acertar.

### Exemplo 4 — Sectioning ou orquestrador? O teste do `for`

```python
# SECTIONING — as partes existem antes de qualquer chamada
partes = dividir_por_capitulo(documento)              # código puro
resumos = [resumir(p) for p in partes]                # N conhecido

# ORQUESTRADOR — as partes só existem depois de uma chamada
plano = chamar(PROMPT_ORQUESTRADOR, documento)        # o modelo decide
for s in plano["subtarefas"]:                         # N desconhecido
    ...                                               # <- precisa de teto
```

A pergunta prática: **você consegue escrever o `for` sem chamar o modelo antes?** Se consegue, é *sectioning* — e é a opção preferida sempre que couber.

---

## Exercícios resolvidos

### 1. Workflow ou agente?

> Sistema que recebe currículos em PDF, extrai dados, compara com os requisitos de uma vaga e produz parecer de aderência. Todo currículo passa pelas mesmas três etapas.

**Workflow — *chaining* com portão.** A frase final entrega o caso: *todo currículo passa pelas mesmas três etapas*. O fluxograma está no enunciado e o número de chamadas é conhecido.

O portão entre a 1 e a 2 é onde mora o valor: se a extração não achou nem nome nem experiência, o PDF é imagem escaneada — e mandar isso adiante produz um parecer inventado sobre um currículo que ninguém leu.

Viraria agente se o parecer pudesse exigir buscas decididas em execução (verificar uma certificação, procurar o repositório citado). Aí o número de passos deixa de ser conhecido.

### 2. Sectioning ou orquestrador-trabalhador?

> Documentos jurídicos longos são divididos em trechos, cada trecho é resumido em paralelo, e os resumos são juntados.

***Sectioning*.** A divisão em trechos é feita pelo seu código — por páginas, seções ou tokens; nada disso é decisão do modelo.

Seria orquestrador-trabalhador se você pedisse ao modelo que lesse o documento e decidisse quais partes merecem análise separada — *"identifique as cláusulas de risco e trate cada uma como subtarefa"*. Aí o número e a natureza das subtarefas passam a existir só em execução, e você precisa de teto.

---

## Síntese

- Antes de melhorar o laço, pergunte se a tarefa precisava de um laço.
- A pergunta operacional: **você consegue desenhar o fluxograma antes de rodar?** Se sim, escreva o fluxograma em código.
- Subir no espectro custa dinheiro, previsibilidade, testabilidade e — o mais caro — capacidade de depurar.
- Cinco padrões cobrem quase tudo: *chaining* com portão, roteador, *sectioning*, *voting*, orquestrador-trabalhador e avaliador-otimizador.
- No roteador, a rota mais valiosa é a que **não chama o modelo**; a rota `nenhuma` é obrigatória.
- *Sectioning* × orquestrador se distinguem por **quem define as subtarefas**. O segundo exige teto.
- *Voting* transforma divergência em sinal de incerteza — não descarte esse sinal.
- Avaliador sem critério produz elogio; par gerador-avaliador sem teto oscila.
- Os padrões se compõem, e o bom projeto confina a autonomia ao caminho estreito onde ela paga.

---

## Fontes e leituras

- **Building Effective Agents** — Anthropic (2024). A taxonomia adotada aqui. Leitura obrigatória do tema.
- **A practical guide to building agents** — OpenAI (2025). Orquestração, *guardrails* e transferência entre agentes.
- **ReAct: Synergizing Reasoning and Acting in Language Models** — Yao et al. (2022). Origem do laço que você implementou.
- **Reflexion: Language Agents with Verbal Reinforcement Learning** — Shinn et al. (2023). Avaliador-otimizador com memória de crítica.
- Aula 01, [nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — definição de agente, espectro de autonomia, tabela de falhas.
- Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — tool calling e o laço ReAct em código.
