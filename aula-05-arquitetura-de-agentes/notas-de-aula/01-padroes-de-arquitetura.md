# IA Aplicada com LLMs — Aula 05: Arquitetura de agentes — Padrões de arquitetura

## Introdução

A Aula 03 terminou com um laço de *tool calling* em funcionamento. Três limitações o impedem de operar sem supervisão, e cada uma corresponde a uma nota desta aula:

1. **Não sabe parar.** `max_passos=6` é um teto arbitrário, não um orçamento.
2. **Não sabe falhar.** Uma ferramenta que levanta exceção derruba o programa.
3. **Não sabe esquecer.** O histórico cresce a cada volta e nada é removido.

Antes das três, há uma questão anterior, que é o objeto desta nota:

> **A tarefa exigia um laço?**

A Aula 04 apresentou uma dúzia de casos de indústria sob a pergunta *"que padrão é este?"*, e a resposta possível naquele momento era um vocabulário de nomes: roteador, orquestrador-trabalhador, avaliador-otimizador. Esta nota converte cada nome em código, com custo em chamadas e com o critério que a literatura de mercado costuma omitir: **quando não usar**.

A taxonomia adotada é a de ANTHROPIC (2024), a formulação mais difundida do tema.

> **Pré-requisitos:** Aula 04, [nota 01](../../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/01-o-que-e-um-agente.md) §2.1 e §4–§8 · Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) §6 e §9 · Aula 01, [nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) §1 · Aula 02, [nota 02](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md) §7 e [nota 04](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md).
>
> **Código:** [`aula05-agentes/`](https://github.com/celsocrivelaro/senac-llm-code/tree/main/aula05-agentes) — os esqueletos desta nota estão disponíveis lá em versão executável.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Decidir** entre *workflow* e agente para uma tarefa concreta, com três perguntas operacionais.
- **Implementar** os cinco padrões de *workflow*: *prompt chaining* com portão, roteador, paralelização (*sectioning* e *voting*), orquestrador-trabalhador e avaliador-otimizador.
- **Enunciar, para cada padrão, as condições em que ele não se aplica.**
- **Estimar o custo em chamadas** de cada padrão antes da implementação.
- **Identificar** *workflow* disfarçado de agente em código existente.

---

## Desenvolvimento teórico

### 1. A linha divisória, em forma operacional

A Aula 01 (nota 03, §1.1) definiu: **workflow** é aquele cujo caminho está no código; **agente** é aquele cujo caminho é decidido em execução pelo modelo. A definição é adequada para exposição e insuficiente para decisão de projeto. A forma operacional é uma pergunta:

> **É possível desenhar o fluxograma do processo antes da execução?**

Em caso afirmativo, o fluxograma deve ser escrito em Python: a solução resultante é mais barata, mais rápida, testável e explicável em caso de falha. Em caso negativo — porque o próximo passo depende do que o anterior descobrir — há motivo para autonomia.

Três perguntas de apoio, quando a resposta não for imediata:

| Pergunta | Sim | Não |
|---|---|---|
| **(a)** O número de passos é conhecido antes da execução? | workflow | agente |
| **(b)** A ordem depende de dados que só existem em execução? | agente | workflow |
| **(c)** O erro é caro ou irreversível? | **reduzir a autonomia**, acrescentar confirmação | autonomia é tolerável |

A pergunta (c) não trata da natureza da tarefa, e sim da **consequência do erro**. Uma tarefa que tecnicamente admitiria um agente, mas cujo erro emite dinheiro, é uma tarefa em que se aceita um sistema pior em troca de previsibilidade.

#### 1.1 O preço de subir no espectro

O espectro de autonomia em seis níveis da Aula 01 (nota 03, §1.4) não detalhou o custo de cada nível:

| Nível | Chamadas | Previsível | Testável | Depurável ao falhar |
|---|---|---|---|---|
| Prompt único | 1 | sim | trivialmente | sim |
| Chain | N fixo | sim | etapa por etapa | sim |
| Roteador | 1 + rota | sim | por rota | sim |
| Workflow com ferramentas | N fixo | sim | sim | sim |
| **Agente** | **desconhecido** | **não** | apenas por propriedade | **apenas com log de trajetória** |
| Multiagente | desconhecido × agentes | não | difícil | muito difícil |

A última coluna é a mais onerosa na prática: num agente a trajetória difere a cada execução e, sem log estruturado, resta uma saída incorreta sem indício de causa. É a razão pela qual a [nota 03](03-confiabilidade.md) insiste no registro do estado.

> **A regra da Aula 01, agora com fundamento:** adotar a menor autonomia que resolve o problema. Não por conservadorismo — cada nível acima custa mais dinheiro, mais latência e mais tempo de diagnóstico quando algo falha.

---

### 2. Autópsia do exercício 03

O exemplo de *workflow* disfarçado de agente mais próximo foi produzido pela turma na aula anterior. As cinco etapas daquele atendimento classificam-se assim:

| Etapa | Classificação |
|---|---|
| 1. Coleta (conversar até obter pedido e problema) | **workflow** — a condição de parada é "os dois campos existem", o que é um `if` |
| 2. `consultar_pedido(id)` | **workflow** — ocorre sempre, e sempre nessa posição |
| 3. Raciocínio (previsão × data corrente, cliente × sistema) | **decisão do modelo** — de *uma* decisão, não de uma sequência |
| 4. `abrir_chamado(...)` com confirmação | **workflow condicional** |
| 5. `consultar_chamado(id)` | **workflow** — sempre, e sempre por último |

Quatro das cinco etapas estavam no código, e a sequência constava numerada de 1 a 5 no enunciado. Tratava-se de **um workflow com dois momentos de decisão**, implementado como agente.

Isso não constitui erro: o objetivo daquele exercício era o *mecanismo* de *tool calling*. Em produção, porém, a versão em *workflow* seria superior em todos os eixos relevantes — menos chamadas, sem risco de omitir a consulta final ou de abrir dois chamados, testável etapa por etapa. Perderia apenas a capacidade de tratar atendimentos fora do roteiro, e é nesse ponto que a decisão se coloca.

> **Padrão a reconhecer:** quando o enunciado do problema vem numerado, o código também deveria vir.

---

### 3. Os cinco padrões

Todos os cinco são **workflows**: o caminho está no código. O agente propriamente dito é objeto da [nota 02](02-o-agente-e-o-estado.md).

---

#### 3.1 Prompt chaining com portão

**Definição.** Sequência fixa de chamadas em que a saída de uma alimenta a seguinte, com **validação determinística entre elas** — o *portão*.

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

O *chaining* foi apresentado na Aula 03 como técnica de prompt; o que se acrescenta aqui é o portão. Regra expressável em Python é implementada em Python: sai mais barata, é mais confiável e não alucina.

| Usar quando | **Não** usar quando |
|---|---|
| as etapas são conhecidas e cada uma produz melhor resultado isolada: extrair → validar → formatar, traduzir → revisar | as etapas não são independentes — cada elo é uma oportunidade de perda de informação |
| a falha intermediária deve interromper o processo | o número de etapas depende da entrada (nesse caso, trata-se de agente) |

---

#### 3.2 Roteador

**Definição.** O modelo **classifica**; o código **despacha**.

```
                    ┌──> rota A: código puro (sem LLM)
   entrada ──> [ classificador ] ──> rota B: outro prompt / outro modelo
                    │              └──> rota C: humano
                    └──> nenhuma das anteriores ──> fila de revisão
```

Tecnicamente, um roteador é **saída estruturada com enumeração** — o mecanismo da Aula 02 (nota 02, §7) determinando o fluxo do programa:

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
    return fila_de_revisao(item)                    # "nenhuma": não inferir
```

Dois pontos de engenharia distinguem um roteador adequado de um inadequado:

- **A rota `nenhuma` é obrigatória.** Sem ela, o modelo é forçado a escolher entre opções que não se aplicam, e escolhe com alta confiança declarada. A rota de escape converte erro silencioso em fila de revisão.
- **A rota de maior valor é a que não chama o modelo.** Se 70% dos itens são resolvidos por `aprovar_por_regra`, o custo do sistema cai 70% sem perda de qualidade. É a otimização de maior retorno desta aula.

| Usar quando | **Não** usar quando |
|---|---|
| entradas heterogêneas exigem tratamentos distintos; triagem | as rotas executam quase a mesma operação — nesse caso, unificá-las num prompt |
| convém selecionar modelo pequeno para o caso fácil e grande para o difícil | a classificação é tão confiável em `if` quanto no modelo |

Implementação em [`00-roteador.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/00-roteador.py), que reporta quantos itens do lote foram resolvidos sem chamada ao modelo.

---

#### 3.3 Paralelização: duas formas com o mesmo nome

**Sectioning.** Subtarefas independentes executadas simultaneamente. As seções são conhecidas: constam do código.

```
                 ┌──> [ LLM: seção 1 ] ──┐
   entrada ──────┼──> [ LLM: seção 2 ] ──┼──> juntar ──> saída
                 └──> [ LLM: seção 3 ] ──┘
```

**Voting.** A mesma tarefa executada N vezes, com decisão por maioria. Corresponde à *self-consistency* (WANG et al., 2022), apresentada na Aula 03 como técnica de prompt e aqui promovida a padrão de arquitetura. A diferença está no local de controle: naquele caso, o parâmetro `n` era passado na chamada e o provedor gerava N amostras do mesmo prompt; aqui, as N chamadas são orquestradas pelo código, o que permite variar prompt, modelo ou temperatura entre elas.

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

O detalhe habitualmente descartado: **a divergência é informação**. Três vereditos distintos não indicam que se deva escolher um deles; indicam que o caso é difícil e, provavelmente, que cabe encaminhamento a humano.

| Usar quando | **Não** usar quando |
|---|---|
| *sectioning*: a tarefa se decompõe e as partes são independentes; latência é requisito | *sectioning*: as partes dependem umas das outras — a concatenação produz trechos contraditórios |
| *voting*: o custo do erro supera o de 3 chamadas e a resposta é discreta (aprovar/reprovar, classificar) | *voting*: tarefa aberta ("redigir um resumo") — não há maioria entre três textos distintos, e o custo triplica sem contrapartida |

---

#### 3.4 Orquestrador-trabalhador

**Definição.** Um modelo **determina quais são as subtarefas**; trabalhadores as executam; um sintetizador consolida os resultados.

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

A distinção em relação ao *sectioning* é única e decisiva: lá as seções constam do código; aqui são determinadas em execução, e sua quantidade é desconhecida de antemão. Daí a verificação explícita: este é o primeiro padrão da nota com autonomia real e, por consequência, o primeiro que exige teto. **Um orquestrador sem teto é uma conta aberta assinada por um modelo.**

| Usar quando | **Não** usar quando |
|---|---|
| a decomposição depende do conteúdo: alteração que afeta número imprevisível de arquivos, pesquisa com número imprevisível de consultas | as subtarefas são enumeráveis — nesse caso trata-se de *sectioning*, mais barato e sem surpresa |

Implementação em [`01-orquestrador-trabalhador.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/01-orquestrador-trabalhador.py).

---

#### 3.5 Avaliador-otimizador (reflexão)

**Definição.** Laço curto de crítica: gerar → avaliar contra critério explícito → revisar.

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
    return candidato, "teto_de_rodadas", max_rodadas    # melhor esforço, sinalizado
```

Duas armadilhas ocorrem sistematicamente:

- **Sem critério escrito, o avaliador produz elogio.** O critério precisa ser uma lista verificável — *cita a regra aplicada? menciona o valor exato? conclui?* — e o schema da avaliação deve forçar resposta item a item. Um retorno `{"nota": 8}` não é acionável; `{"cita_regra": false, "o_que_corrigir": "..."}` é.
- **Sem teto de rodadas, o par oscila.** O avaliador exige A, o gerador entrega A e perde B, o avaliador exige B. Três rodadas costumam bastar; o essencial é que o retorno **declare** a saída por teto. Melhor esforço apresentado como aprovação é pior que reprovação explícita.

A formalização do padrão é o Reflexion (SHINN et al., 2023), que acrescenta a persistência da crítica em memória para realimentar as tentativas seguintes — função do parâmetro `critica` no esqueleto acima.

| Usar quando | **Não** usar quando |
|---|---|
| existe critério de qualidade explícito; a revisão melhora o resultado; 2–3 chamadas adicionais se justificam | o critério não é escrevível — nesse caso o avaliador adequado é uma pessoa |
| texto sujeito a requisitos formais, código submetido a teste, tradução com glossário obrigatório | tarefa subjetiva sem lastro verificável |

Implementação em [`02-avaliador-otimizador.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/02-avaliador-otimizador.py), que executa também **sem** critério, para evidenciar a degeneração do avaliador em elogio.

---

### 4. A tabela de decisão

A tabela é lida de cima para baixo, interrompendo-se **no primeiro padrão que resolve**. A ordem vai do mais previsível ao menos previsível.

| Padrão | Resolve | Custo em chamadas | Quando **não** usar |
|---|---|---|---|
| **Prompt único** | tarefa fechada | 1 | a saída exige validação que só outra chamada fornece |
| **Chaining + portão** | etapas conhecidas, melhores isoladas | N fixo | etapas interdependentes; número de etapas variável |
| **Roteador** | entradas heterogêneas | 1 + rota | rotas quase idênticas; classificação trivial em código |
| **Sectioning** | partes independentes conhecidas | 1 por seção (paralelo) | partes mutuamente dependentes |
| **Voting** | erro caro, resposta discreta | N (3–5) | tarefa aberta, sem maioria possível |
| **Orquestrador-trabalhador** | decomposição desconhecida | variável — **exige teto** | as subtarefas são enumeráveis |
| **Avaliador-otimizador** | qualidade com critério explícito | 2 por rodada | sem critério escrito |
| **Agente** ([nota 02](02-o-agente-e-o-estado.md)) | sequência imprevisível com feedback do ambiente | desconhecido — **exige orçamento** | o fluxograma existe |

---

### 5. Composição de padrões

Os padrões não são mutuamente exclusivos. Sistemas reais os encaixam, cada peça no nível de autonomia adequado:

```
   lote ──> [ ROTEADOR ] ──┬──> regra em código (a maioria dos itens)
                           ├──> [ AGENTE ]  (os ambíguos, poucos)
                           └──> humano      (os caros)
                                  │
                                  ▼
                        [ ORQUESTRADOR-TRABALHADOR ]  (parecer do lote)
                                  │
                                  ▼
                        [ AVALIADOR-OTIMIZADOR ]      (revisão do texto final)
```

Este é, deliberadamente, o desenho do exercício desta aula: a autonomia cara fica confinada ao caminho estreito em que é necessária, e o volume trafega por código determinístico.

> **Princípio de projeto:** autonomia é recurso escasso, e o trabalho de arquitetura consiste em gastá-la apenas onde ela produz retorno.

---

## Exemplos

### Exemplo 1 — O mesmo problema, três arquiteturas, três contas

Classificar 1.000 despesas e justificar as que violam a política, com 15% de violações:

| Arquitetura | Conta | Chamadas |
|---|---|---|
| **(a)** Agente por despesa | 1.000 × ~4 passos | **~4.000** |
| **(b)** Chain fixa: classificar + justificar se violar | 1.000 + 150 | **1.150** |
| **(c)** Roteador: regra em código resolve 70%, LLM nos 30%, justificativa nas violações | 0 + 300 + 150 | **450** |

De (a) para (c), a redução é de quase nove vezes — e (c) é *mais* confiável nos 70% resolvidos por regra, uma vez que regra não alucina. O ganho não decorre de prompt melhor nem de modelo melhor, e sim da decisão sobre o que **não** enviar ao modelo.

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

O primeiro avaliador aprovou um parecer que não cita a regra. A causa não é deficiência do modelo, e sim ausência de especificação do que constitui qualidade. A qualidade de um avaliador é a qualidade do critério redigido; o modelo apenas o executa.

### Exemplo 3 — A rota de escape, presente e ausente

Item: *"Uber R\$ 340,00 — funcionário de outra filial, sem centro de custo."* Nenhuma das três rotas previstas se aplica.

```python
# Sem escape: enum = ["dentro_da_politica", "ambiguo", "acima_da_alcada"]
# -> {"rota": "ambiguo", "justificativa": "requer análise"}   # foi para o agente
#    o agente consumiu 6 passos e concluiu "não foi possível determinar"

# Com escape: enum = [..., "nenhuma"]
# -> {"rota": "nenhuma", "justificativa": "centro de custo ausente"}
#    foi para a fila de revisão em 1 chamada
```

A mesma incerteza produz destinos distintos. Sem a rota de escape, a incerteza converte-se em **gasto**; com ela, converte-se em **fila**. O modelo não errou nas duas execuções: na primeira, não dispunha de alternativa correta.

### Exemplo 4 — Sectioning ou orquestrador: o teste do `for`

```python
# SECTIONING — as partes existem antes de qualquer chamada
partes = dividir_por_capitulo(documento)              # código puro
resumos = [resumir(p) for p in partes]                # N conhecido

# ORQUESTRADOR — as partes só existem após uma chamada
plano = chamar(PROMPT_ORQUESTRADOR, documento)        # o modelo decide
for s in plano["subtarefas"]:                         # N desconhecido
    ...                                               # <- exige teto
```

O teste operacional: **é possível escrever o `for` sem chamar o modelo antes?** Em caso afirmativo, trata-se de *sectioning* — a opção preferida sempre que aplicável.

---

## Fontes e leituras

ANTHROPIC. **Building effective agents**. 2024. (Taxonomia adotada nesta nota: distinção *workflow* × agente e os cinco padrões.)

LIU, N. F. et al. **Lost in the middle**: how language models use long contexts. arXiv:2307.03172, 2023.

OPENAI. **A practical guide to building agents**. 2025. (Orquestração, *guardrails* e transferência entre agentes.)

SHINN, N. et al. **Reflexion**: language agents with verbal reinforcement learning. arXiv:2303.11366, 2023.

WANG, X. et al. **Self-consistency improves chain of thought reasoning in language models**. arXiv:2203.11171, 2022.

YAO, S. et al. **ReAct**: synergizing reasoning and acting in language models. arXiv:2210.03629, 2022.

**Material da disciplina.** Aula 01, [nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — definição de agente, espectro de autonomia, tabela de modos de falha. Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — *tool calling* e o laço ReAct em código.
