# IA Aplicada com LLMs — Aula 05: Arquitetura de agentes — Os padrões, e quando cada um

## Introdução

A Aula 03 terminou com um laço de *tool calling* em funcionamento. Três limitações o impedem de operar sem supervisão, e cada uma corresponde a uma nota desta aula:

1. **Não sabe parar.** `max_passos=6` é um teto arbitrário, não um orçamento.
2. **Não sabe falhar.** Uma ferramenta que levanta exceção derruba o programa.
3. **Não sabe esquecer.** O histórico cresce a cada volta e nada é removido.

Antes das três, há uma questão anterior, que é o objeto desta nota:

> **A tarefa exigia um laço?**

### O que é um padrão de arquitetura, aqui

A Aula 04 apresentou uma dúzia de casos de indústria sob a pergunta *"que padrão é este?"*, e a resposta possível naquele momento era **um vocabulário de nomes**: *Router*, orquestrador-trabalhador, avaliador-otimizador. Esta série converte cada nome em código.

Um padrão, nesta aula, não é um diagrama nem uma biblioteca. É a resposta a **três perguntas de projeto**, e é por elas que os cinco se distinguem:

| | A pergunta |
|---|---|
| **Quem decide o fluxo** | o seu código, ou o modelo em tempo de execução? |
| **Quantas chamadas custa** | você sabe o número antes de rodar, ou não? |
| **Quando *não* usar** | qual é a condição em que ele é a escolha errada? |

A terceira é a que a literatura de mercado costuma omitir, e é a que esta série trata como obrigatória: **um padrão apresentado sem a condição de não uso é propaganda, não engenharia.**

Todos os cinco são **workflows** — o caminho está no código. O agente propriamente dito, em que o caminho é decidido pelo modelo a cada passo, é objeto da [nota 02](02-o-agente-e-o-estado.md).

### O que vem nesta série

Sete notas, e a ordem não é arbitrária: **a decisão sobre o fluxo migra do código para o modelo, e o custo deixa de ser calculável no meio do caminho.**

| Nota | Padrão | Quem decide o fluxo | Custo |
|---|---|---|---|
| **01** (esta) | — | o critério anterior a todos: **precisa de autonomia?** | — |
| [**01-1**](01-1-sequencial.md) | **Sequencial** (*prompt chaining*) com portão | o código, em cada portão | N fixo |
| [**01-2**](01-2-router.md) | ***Router*** | o modelo classifica; o código despacha | 1 + rota |
| [**01-3**](01-3-paralelizacao.md) | Paralelização — *sectioning* e *voting* | o código, antes de rodar | N fixo |
| [**01-4**](01-4-orquestrador.md) | Orquestrador-trabalhador | **o modelo, em execução** | **variável — exige teto** |
| [**01-5**](01-5-avaliador-otimizador.md) | Avaliador-otimizador | **o avaliador, a cada rodada** | **variável — exige teto** |
| [**01-6**](01-6-qual-padrao-usar.md) | — | a tabela de decisão, a composição e o fecho | — |

A linha entre **01-3** e **01-4** é a fronteira que organiza a aula: até ali, o número de chamadas é conhecido antes da execução; a partir dali, **o modelo decide quantas vezes ele mesmo será chamado**, e aparece a necessidade de teto.

Cada nota de padrão segue o mesmo formato — o que é, esqueleto de código, quando usar, **quando não usar** — e aponta para o script correspondente no laboratório.

A taxonomia adotada é a de ANTHROPIC (2024), a formulação mais difundida do tema.

> **Pré-requisitos:** Aula 04, [nota 01](../../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/01-o-que-e-um-agente.md) §2.1 e §4–§8 · Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) §6 e §9 · Aula 01, [nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) §1.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Decidir** entre *workflow* e agente para uma tarefa concreta, com três perguntas operacionais.
- **Estimar o preço de cada nível de autonomia** em chamadas, previsibilidade e custo de diagnóstico.
- **Identificar** *workflow* disfarçado de agente em código existente — inclusive no próprio.

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
| *Router* | 1 + rota | sim | por rota | sim |
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

---

## Exemplos

### Exemplo 1 — O mesmo problema, três arquiteturas, três contas

Classificar 1.000 despesas e justificar as que violam a política, com 15% de violações:

| Arquitetura | Conta | Chamadas |
|---|---|---|
| **(a)** Agente por despesa | 1.000 × ~4 passos | **~4.000** |
| **(b)** Chain fixa: classificar + justificar se violar | 1.000 + 150 | **1.150** |
| **(c)** *Router*: regra em código resolve 70%, LLM nos 30%, justificativa nas violações | 0 + 300 + 150 | **450** |

De (a) para (c), a redução é de quase nove vezes — e (c) é *mais* confiável nos 70% resolvidos por regra, uma vez que regra não alucina. O ganho não decorre de prompt melhor nem de modelo melhor, e sim da decisão sobre o que **não** enviar ao modelo.

## Fontes e leituras

ANTHROPIC. **Building effective agents**. 2024. (Taxonomia adotada nesta nota: distinção *workflow* × agente e os cinco padrões.)

LIU, N. F. et al. **Lost in the middle**: how language models use long contexts. arXiv:2307.03172, 2023.

OPENAI. **A practical guide to building agents**. 2025. (Orquestração, *guardrails* e transferência entre agentes.)

SHINN, N. et al. **Reflexion**: language agents with verbal reinforcement learning. arXiv:2303.11366, 2023.

WANG, X. et al. **Self-consistency improves chain of thought reasoning in language models**. arXiv:2203.11171, 2022.

YAO, S. et al. **ReAct**: synergizing reasoning and acting in language models. arXiv:2210.03629, 2022.

**Material da disciplina.** Aula 01, [nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — definição de agente, espectro de autonomia, tabela de modos de falha. Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — *tool calling* e o laço ReAct em código.
