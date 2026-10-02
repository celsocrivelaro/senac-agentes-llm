# IA Aplicada com LLMs — Aula 09: Frameworks e orquestração

## Introdução

O curso inteiro se apoia na tese de que framework vem **depois** de tudo feito à mão. Quando LangGraph chega, você já escreveu o laço, o objeto de estado, o orçamento, o detector de laço, o índice e a memória. O framework, nessas condições, não é aprendido: é **reconhecido**.

Antes de escrever qualquer linha, porém, é preciso desfazer uma confusão que impede a leitura do resto da aula. **LangChain, LangGraph e LangSmith são três produtos distintos**, com problemas distintos, e quase todo mundo chega tratando os três como um nome só.

O resto da nota acompanha o laboratório na ordem em que ele roda: as duas peças sobre as quais tudo é montado, os padrões da Aula 05 reescritos como grafo, o estado, as duas memórias do runtime, o humano no meio da execução — e, no fim, a lista do que o framework **não** entrega.

> **Pré-requisitos:** Aula 05, [nota 01](../../aula-08-memoria/notas-de-aula/01-o-agente-e-o-estado.md) — o estado e o laço · [nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — orçamento, término e classificação de erro · Aula 08, as três memórias.
>
> **Código:** [`aula09-framework/`](https://github.com/celsocrivelaro/senac-llm-code/tree/main/aula09-framework) — catorze arquivos, numerados na ordem desta nota.

---

## Objetivos de aprendizagem

Ao final desta nota, você deve ser capaz de:

- **Distinguir** biblioteca de abstrações, runtime de orquestração e plataforma de observabilidade, e dizer que problema cada uma resolve.
- **Situar** os principais frameworks de agente em um eixo único, sem confundir escopo.
- **Reescrever como grafo** os padrões de arquitetura da Aula 05, e reconhecer cada peça manual na construção equivalente.
- **Declarar um estado** com reducers, e explicar o que acontece quando falta um.
- **Distinguir checkpoint de store** como duas classes diferentes, e não como dois nomes para memória.
- **Interromper a execução** para uma decisão humana, e explicar por que isso depende do checkpoint.
- **Listar o que o framework não fornece**, e reconhecer o que ele cobra sem avisar.

---

## Desenvolvimento teórico

### 1. Três produtos, um nome

| Camada | Produto | O problema que resolve |
|---|---|---|
| Biblioteca | **LangChain** | abstrações e integrações sobre modelo, ferramenta e laço de agente |
| Runtime | **LangGraph** | execução durável, persistência, *streaming*, interrupção para humano |
| Plataforma | **LangSmith** | *trace*, avaliação, registro de prompt, implantação |

A separação importa por uma razão prática: **elas se adotam separadamente**. É possível usar LangGraph sem LangChain, e é possível instrumentar um agente escrito à mão com LangSmith. Tratar as três como pacote único produz a decisão de adoção mais cara possível — a de tudo ou nada.

A camada que mais frequentemente justifica a adoção sozinha é a do **runtime**: reimplementar execução durável corretamente é caro, e é requisito de produção, não conveniência.

### 2. O mapa, sobre um eixo só

O ecossistema é grande e muda depressa. Para o propósito desta aula, um eixo basta: **quem fornece o laço e quem fornece o runtime**.

| Framework | Laço | Runtime durável | Modelo mental |
|---|---|---|---|
| **LangGraph** | sim | sim | grafo de estado com arestas condicionais |
| **CrewAI** | sim | parcial | equipe de papéis com tarefas |
| **OpenAI Agents SDK** | sim | não | agentes com *handoff* entre si |
| **Google ADK** | sim | sim | agentes componíveis, com avaliação integrada |
| **AutoGen** | sim | não | conversa entre agentes |

Nenhuma dessas linhas é elogio ou crítica. São recortes diferentes, e a diferença aparece na pergunta que cada um responde bem: CrewAI responde *"como divido isto entre papéis"*; AutoGen, *"como faço agentes conversarem"*; LangGraph, *"como declaro este fluxo como grafo e o torno retomável"*.

### 3. Por que LangGraph, aqui

Dois critérios, ambos declarados para que você não confunda escolha com adesão.

**O primeiro é externo:** a Parte 2 do trabalho exige o grafo em LangChain e LangGraph. A disciplina não escolheu o framework; o enunciado do trabalho escolheu, e a aula prepara para ele.

**O segundo é pedagógico, e é o que importa mais.** O modelo mental de LangGraph — um grafo com estado compartilhado, nós que o transformam e arestas condicionais que decidem o próximo nó — é literalmente o que você construiu à mão na Aula 05. O `EstadoAgente` é o estado; cada bloco do laço é um nó; o `if` que decide se há `tool_calls` é uma aresta condicional.

Isso torna cada peça **reconhecível**: onde há correspondente, você o identifica; onde não há, a ausência é informação — e a §9 é feita dela.

### 4. O modelo aumentado: as duas peças de base

Antes de qualquer grafo, duas operações sobre o modelo. Nenhuma delas é um agente; **todos os padrões adiante são combinações delas**.

| Operação | O que ela faz |
|---|---|
| `with_structured_output(Schema)` | a saída obedece a um schema, validado contra um tipo |
| `bind_tools([...])` | a saída pode ser um **pedido** de chamada: nome e argumentos |

A segunda é a que mais confunde na primeira leitura. **O modelo não executa nada.** Ele devolve o nome da ferramenta e os argumentos; quem executa é o seu código. É essa separação que o laço fecha, e é por isso que ela aparece sozinha no laboratório antes de qualquer nó.

E há um limite que precisa ser dito junto, porque ele reaparece em toda a disciplina: **o schema garante a forma, não o conteúdo.** Um `Literal["conto", "piada", "poema"]` garante que a resposta seja uma das três; ele não garante que seja a certa.

### 5. Os padrões da Aula 05, agora como grafo

A Aula 05 deu cinco padrões de workflow mais o agente, cada um com a condição de não uso. Nenhum deles muda aqui — o que muda é que deixam de ser escritos e passam a ser **declarados**.

| Padrão | Como vira grafo |
|---|---|
| sequencial com portão | nós em sequência, e uma **aresta condicional** cuja função é código puro |
| *router* | aresta condicional cujo destino vem de uma saída estruturada do modelo |
| paralelização (*sectioning*) | várias arestas saindo do mesmo ponto; o nó agregador só roda quando todas terminam |
| orquestrador-trabalhador | `Send`, que cria **um trabalhador por item** em tempo de execução |
| avaliador-otimizador | aresta de volta ao gerador, com um contador no estado |
| agente | aresta condicional que devolve ao nó do modelo enquanto houver `tool_calls` |

Três observações que o laboratório torna visíveis:

**Quem decide a bifurcação é o assunto, não a sintaxe.** No sequencial com portão, quem decide é um `if` — barato, determinístico, auditável. No *router*, quem decide é o modelo — caro, e **erra**. A cascata da Aula 06 continua valendo: regra, depois embedding, só então modelo.

**O orquestrador-trabalhador é o único em que o grafo não sabe quantos nós vai executar.** O grafo tem um nó de trabalhador; quantas cópias dele rodam sai do planejamento, em tempo de execução.

**O avaliador-otimizador precisa de um limite que o framework não dá.** Sem contador no estado, um avaliador exigente deixa o grafo girando e gastando até o limite de recursão — que não é a mesma coisa que um orçamento.

### 6. O estado, e os reducers

Na Aula 05 a tese era que **a lista de mensagens não é o estado**. Aqui ela deixa de ser argumento e vira declaração de tipo: um `TypedDict` com os campos que a execução carrega.

A novidade é o **reducer**: a função que diz o que fazer quando dois nós escrevem no mesmo campo. `add_messages` acumula mensagens; `operator.add` concatena listas; **sem reducer, o campo é substituído**.

Essa última frase é o defeito mais silencioso do padrão. Num grafo com nós paralelos, um campo sem reducer é sobrescrito pelo último que termina, **e nada reclama**: não há exceção, não há log, não há erro. O trabalho dos outros ramos simplesmente desaparece, e o sintoma é um resultado incompleto que parece correto.

### 7. As duas memórias do runtime: checkpoint e store

A Aula 08 separou *retomar* de *lembrar*. O runtime materializa essa fronteira em duas classes diferentes, e confundi-las é o erro mais comum de quem chega pela documentação.

| | **checkpointer** | **store** |
|---|---|---|
| guarda | o estado de **uma** execução | fatos **fora** da execução |
| endereçado por | `thread_id` | *namespace* (tupla) |
| serve para | **retomar** — a conversa continua de onde parou | **lembrar** — vale em qualquer thread |
| escopo | uma conversa | todas |

A prova está no laboratório: uma terceira chamada abre uma **thread nova** — o checkpoint está vazio, não há histórico nenhum — e o agente ainda assim sabe o nome do usuário, porque leu do store.

E uma consequência prática que decide implementação: **um checkpointer em memória morre com o processo.** "Fecha o terminal, reabre, a conversa continua" exige checkpointer em arquivo ou banco. Em memória, a promessa não se cumpre.

### 8. O humano no meio do grafo

Até aqui a execução é uma linha reta: começa, roda, termina. `interrupt()` quebra isso: ele **suspende a execução dentro de um nó**, devolve um valor a quem chamou, e espera.

A retomada é uma nova invocação com `Command(resume=...)`, e o valor passado vira o retorno do `interrupt()` lá dentro — a função continua da linha seguinte, como se nunca tivesse parado.

**Isso só funciona porque existe o checkpointer.** A pausa é um checkpoint gravado, endereçado pelo `thread_id`. É o encaixe mais bonito da aula: duas peças independentes que, combinadas, produzem uma terceira capacidade.

Três decisões que o framework não toma por você, e que a Parte 2 do trabalho cobra:

- **onde parar** — a ação irreversível, o custo acima de um teto, a confiança baixa;
- **o que mostrar** — a ação exata com os argumentos, e a consequência. Um *"confirma?"* transfere a responsabilidade sem transferir a informação;
- **o que fazer se ninguém responder** — e seguir em frente silenciosamente não é opção.

E a aprovação não dispensa **idempotência**: aprovar duas vezes não pode executar duas vezes.

### 9. O que o framework não dá

A lista abaixo é o produto mais útil da aula, e cada item precisa ser reimplantado por você, dentro do grafo:

| Peça | Por que ela não vem |
|---|---|
| **motivo de término** | o nó terminal diz que acabou; não diz **por quê** |
| **detector de laço** | `recursion_limit` limita o dano; **não detecta** progresso nulo |
| **orçamento** | dos quatro tetos da Aula 05 — passos, tokens, tempo, dinheiro —, o framework cobre o primeiro |
| **erro recuperável × fatal** | a classificação continua sendo decisão sua |

Um agente preso na primeira volta gasta o teto inteiro e devolve o diagnóstico errado. Limite de recursão e detecção de laço parecem a mesma coisa e não são.

### 10. O que ele cobra sem avisar

Duas cobranças que não aparecem em nenhum tutorial.

**A decisão que vira parâmetro com valor padrão.** Um limiar que você justificou com medição passa a ser um argumento com valor default; um `k` escolhido por `recall@k` vira um número na assinatura. O valor continua lá, mas a **justificativa** sai do código — e o padrão pode mudar numa atualização de versão, sem que nada no seu repositório mude. Declare o parâmetro explicitamente, mesmo quando o valor coincidir com o padrão.

**O paralelismo que não acontece.** Despachar nós em paralelo no grafo não produz paralelismo se o provedor do modelo atende uma requisição por vez. O grafo faz a parte dele; o tempo total continua sendo a soma. Medir é a única forma de saber qual dos dois você tem — e o Exemplo 3 mostra a medição.

---

## Exemplos

### Exemplo 1 — A mesma decisão, nas três camadas

```
PROBLEMA                                   CAMADA QUE RESOLVE
"preciso trocar de provedor de modelo"     biblioteca  (LangChain)
"o processo caiu no passo 9 de 14"         runtime     (LangGraph)
"o prompt mudou e ninguém sabe se piorou"  plataforma  (LangSmith)
```

As três podem ser resolvidas sem framework nenhum. A pergunta da aula é quanto custa cada alternativa.

### Exemplo 2 — O grafo que você já escreveu

```
        AULA 05, à mão                    LANGGRAPH
   ┌────────────────────────┐      ┌────────────────────────┐
   │ estado = EstadoAgente()│      │ State (TypedDict)      │
   │ while True:            │      │                        │
   │   msg = modelo(...)  ──┼──────┼─▶ nó "agente"          │
   │   if not msg.tool_calls│      │      │                 │
   │       break          ──┼──────┼──────┴─▶ aresta        │
   │   for c in tool_calls: │      │           condicional  │
   │       obs = exec(c)  ──┼──────┼─▶ nó "ferramentas"     │
   └────────────────────────┘      └────────────────────────┘
```

O `while` vira topologia. Não é uma ideia nova sendo aprendida — é a mesma ideia, declarada em vez de escrita.

### Exemplo 3 — O paralelismo que não aconteceu

Três nós despachados ao mesmo tempo, cada um medindo o próprio tempo desde o início da execução:

```
  [nó] gerar_conto   56.0 s
  [nó] gerar_piada   98.0 s
  [nó] gerar_poema  175.1 s
  total             175.2 s
```

Se os três tivessem rodado de fato em paralelo, os três terminariam por volta dos mesmos 175 s. Terminaram em 56, 98 e 175: **um depois do outro**. O grafo despachou corretamente; quem serializou foi o servidor do modelo, que atendia uma requisição por vez.

O total é a **soma**, não o **maior**. Contra uma API que atende as três ao mesmo tempo, o resultado seria o oposto — e é por isso que a afirmação "paralelizei, logo ficou mais rápido" precisa de medição, e não de diagrama.

---

## Fontes e leituras

LANGCHAIN. **LangChain 1.0 and LangGraph 1.0**: general availability. 2025.

LANGCHAIN. **LangGraph**: *graph API*, *persistence*, *interrupts*, *streaming*. Documentação do runtime.

ANTHROPIC. **Building effective agents**. 2024. — os padrões de workflow que a Aula 05 adotou e esta reescreve como grafo.

**Material da disciplina.** Aula 05, [nota 01](../../aula-08-memoria/notas-de-aula/01-o-agente-e-o-estado.md) — o estado explícito e o laço; [nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — orçamento, término e classificação de erro, que a §9 mostra não virem prontos. Aula 04, [nota 01](../../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/01-o-que-e-um-agente.md) — os padrões de arquitetura, que o framework não escolhe.
