# IA Aplicada com LLMs — Aula 05: Padrões de arquitetura — Qual padrão usar

> **Esta nota fecha a série.** As cinco anteriores abriram um padrão cada; esta responde à pergunta que só faz sentido com os cinco na mesa: **dado um problema concreto, qual deles?**
>
> Série completa: [01](01-0-padroes-de-arquitetura.md) a decisão de fundo · [01-1](01-1-sequencial.md) *chaining* · [01-2](01-2-router.md) *Router* · [01-3](01-3-paralelizacao.md) paralelização · [01-4](01-4-orquestrador.md) orquestrador · [01-5](01-5-avaliador-otimizador.md) avaliador.

---

## 1. O eixo que ordena tudo

Antes da tabela, o critério que a organiza. Os padrões se distinguem por **quem decide o fluxo**, e a resposta migra do código para o modelo:

```
   o CÓDIGO decide ─────────────────────────────> o MODELO decide
   │                                                            │
   prompt único  sequencial  Router   sectioning/voting   orquestrador   avaliador   agente
   └──────────── custo calculável ─────────────┘ └──── custo desconhecido: EXIGE TETO ────┘
```

A fronteira não está onde a intuição sugere. **No *Router* o modelo já decide** — mas decide entre rotas que você escreveu, e por isso o custo continua sendo `1 + rota`. A conta só deixa de fechar quando o modelo passa a decidir **quantas vezes ele mesmo será chamado**: do orquestrador em diante.

---

## 2. A tabela de decisão

Ela é lida de cima para baixo, interrompendo-se **no primeiro padrão que resolve**. A ordem vai do mais previsível ao menos previsível.

| Padrão | Quem decide o fluxo | Resolve | Custo em chamadas | Quando **não** usar |
|---|---|---|---|---|
| **Prompt único** | ninguém | tarefa fechada | 1 | a saída exige validação que só outra chamada fornece |
| **Sequencial + portão** | o código | etapas conhecidas, melhores isoladas | N fixo | etapas interdependentes; número de etapas variável |
| ***Router*** | o modelo **classifica**; o código despacha | entradas heterogêneas | 1 + rota | rotas quase idênticas; classificação trivial em código |
| **Sectioning** | o código, antes de rodar | partes independentes conhecidas | 1 por seção (paralelo) | partes mutuamente dependentes |
| **Voting** | o código; a maioria decide | erro caro, resposta discreta | N (3–5) | tarefa aberta, sem maioria possível |
| **Orquestrador-trabalhador** | **o modelo, em execução** | decomposição desconhecida | variável — **exige teto** | as subtarefas são enumeráveis |
| **Avaliador-otimizador** | **o avaliador**, a cada rodada | qualidade com critério explícito | 2 por rodada | sem critério escrito |
| **Agente** ([nota 02](02-o-agente-e-o-estado.md)) | **o modelo, a cada passo** | sequência imprevisível com feedback do ambiente | desconhecido — **exige orçamento** | o fluxograma existe |

A segunda coluna é a que ordena a tabela — e é o eixo desenhado na §1. A linha do **orquestrador** é onde a conta deixa de fechar de antemão.

---

## 3. Os quatro papéis, em uma linha cada

Para reconhecer o padrão antes de escolhê-lo:

| Padrão | Papel | Analogia |
|---|---|---|
| ***Router*** | triagem e direcionamento | **recepção** |
| **Paralelização** | executar o independente ao mesmo tempo | **equipe em paralelo** |
| **Orquestrador** | planejar, dividir, coordenar, sintetizar | **gerência de projeto** |
| **Avaliador** | verificar e refinar por *feedback* | **controle de qualidade** |

A analogia serve para lembrar o papel. **Ela não serve para escolher** — para isso existe a tabela da §2, que tem custo e condição de não uso.

---

## 4. Três perguntas que resolvem a maioria dos casos

**O fluxograma existe?** Se sim, escreva-o em Python. Nenhum padrão desta série é necessário — e a [nota 01](01-0-padroes-de-arquitetura.md) trata disso.

**Você consegue escrever `for parte in partes` sem chamar o modelo antes?** Se sim, é *sectioning*. Se não, é orquestrador, e ele precisa de teto.

**O critério de qualidade é executável?** Se sim, o avaliador é um teste — não um LLM, e não uma pessoa.

---

## 5. Composição: eles não são excludentes

Os padrões não são mutuamente exclusivos. Sistemas reais os encaixam, cada peça no nível de autonomia adequado:

```
   lote ──> [ ROUTER ]   ──┬──> regra em código (a maioria dos itens)
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

---


---

## 6. O erro mais caro da série

Não é escolher o padrão errado — é **escolher um padrão quando nenhum era necessário**.

A tabela da §2 começa em "prompt único, 1 chamada" de propósito. A maior parte dos problemas que chegam como *"preciso de um agente"* se resolve com uma chamada e três `if`, e a diferença entre as duas soluções aparece na conta do fim do mês, no tempo de diagnóstico quando algo falha, e na capacidade de explicar o sistema a quem não o escreveu.

> **Autonomia é recurso escasso**, e o trabalho de arquitetura consiste em gastá-la apenas onde ela produz retorno.
