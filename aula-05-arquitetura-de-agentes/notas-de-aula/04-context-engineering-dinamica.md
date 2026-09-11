# IA Aplicada com LLMs — Aula 05: Arquitetura de agentes — Context engineering dinâmica

## Introdução

A nota 02 da Aula 03 encerrou com uma promessa: naquela nota, o que se monta na chamada; na aula de agentes, o que se faz com a janela **ao longo da trajetória**. Esta nota cumpre a promessa e trata da terceira limitação enunciada na abertura da aula: o laço **não sabe esquecer**.

> **A cada volta, o histórico íntegro é reenviado. Nada é removido.**

Numa trajetória de três passos, a propriedade é irrelevante. Numa de trinta, ela domina o custo, a latência e — o efeito menos esperado — a **qualidade**.

> **O que esta nota faz, e o que ela não faz.** Aqui está o **problema** — a aritmética do crescimento, a distinção entre curadoria estática e dinâmica, e a única tática que se aplica antes de qualquer outra: encurtar o que as ferramentas devolvem.
>
> As **técnicas de compressão** — *tool clearing*, *compaction* reativa e periódica, e o conjunto mínimo que um resumo nunca pode perder — são tratadas na **Aula 08** (nota 02), quando ela for publicada. Não é adiamento por falta de tempo: lá elas ficam ao lado da política de escrita da memória, que responde à **mesma pergunta** — o que guardar, o que descartar, e o que não se pode perder em hipótese alguma.

> **Pré-requisitos:** notas [02](02-o-agente-e-o-estado.md) e [03](03-confiabilidade.md) desta aula · Aula 03, [nota 02](../../aula-03-prompt-engineering/notas-de-aula/02-context-engineering.md) integralmente · Aula 02, [nota 01](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) §4.
>
> **Código:** [`07-compaction.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/07-compaction.py) — trajetória longa com e sem *compaction*, com a contagem de tokens por passo.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Calcular** o crescimento do contexto ao longo de uma trajetória e demonstrar que é superlinear no acumulado.
- **Distinguir** curadoria estática — o que se monta na chamada — de curadoria dinâmica — o que o programa faz durante a execução.
- **Empregar sub-agentes como isolamento de contexto**, com conhecimento preciso da perda envolvida.
- **Aplicar a tática zero** — encurtar o retorno das ferramentas — antes de qualquer técnica de compressão.
- **Ordenar** as táticas da mais segura à mais destrutiva, sabendo qual delas a Aula 08 vai implementar.

---

## Desenvolvimento teórico

### 1. A aritmética do acumulado

Considere um agente cujo *system prompt* e declarações de ferramentas somam aproximadamente 1.500 tokens, e em que cada passo acrescenta ao histórico a mensagem do modelo (cerca de 80 tokens) e o resultado da ferramenta (cerca de 400 tokens) — aproximadamente **480 tokens por passo**. O contexto enviado no passo *n* é `1.500 + 480 × (n − 1)`; o total consumido é a soma desses valores desde o primeiro passo.

| Passo | Contexto neste passo | Total acumulado |
|---:|---:|---:|
| 1 | 1.500 | 1.500 |
| 5 | 3.420 | 12.300 |
| 10 | 5.820 | 36.600 |
| 20 | 10.620 | 121.200 |
| 30 | 15.420 | 253.800 |

```
  tokens
  acumulados
  260k ┤                                                        ╭─
       │                                                    ╭───╯
  200k ┤                                               ╭────╯
       │                                          ╭────╯
  140k ┤                                    ╭─────╯
       │                              ╭─────╯
   80k ┤                       ╭──────╯
       │               ╭───────╯
   20k ┤────────╴──────╯
       └────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬──
            3    6    9   12   15   18   21   24   27   30  passos
```

> Os valores acima resultam da aritmética do acumulado, não de medição. A execução do `07-compaction.py` com o modelo em uso permite refazer a tabela: a **forma** da curva não se altera, os valores sim.

Três leituras decorrem do quadro:

| Efeito | Descrição |
|---|---|
| **Custo quadrático** | cada passo reenvia todo o histórico anterior. Dobrar o número de passos quadruplica a conta: um agente de 30 passos custa aproximadamente **sete** vezes um de 10 |
| **Latência proporcional** | o tempo até o primeiro token depende do tamanho da entrada (Aula 02, nota 04, §5). A execução inicia em cerca de 400 ms por passo e encerra em vários segundos |
| **Degradação de qualidade anterior ao limite da janela** | no passo 25, o objetivo está cercado por cerca de 12.000 tokens de observações — a região central, onde a recuperação é pior (*lost in the middle*, LIU et al., 2023) |

O terceiro efeito é o menos intuitivo e o mais relevante: o desempenho do agente não se degrada por falta de espaço, e sim por **excesso de passado**.

> **Formulação:** num agente, a janela de contexto (*context window*) não é um limite que se atinge. É um recurso que se degrada continuamente à medida que é consumido.

---

### 2. Curadoria estática × dinâmica

```
   ESTÁTICO (aula 03)                DINÂMICO (esta nota)
   ┌───────────────────┐             ┌───────────────────┐
   │ system            │             │ system            │  preservado
   │ exemplos          │             │ objetivo          │  preservado
   │ schema            │   ──────>   │ ····· resumo ·····│  comprimido
   │ documentos        │             │ passos 24-30      │  íntegros
   │ pergunta          │             │ objetivo (de novo)│  reancorado
   └───────────────────┘             └───────────────────┘
   decisão única, no código          decisão a cada volta, no programa
```

A curadoria estática é uma decisão tomada uma vez, na redação do código. A dinâmica é o tratamento aplicado à janela **durante** a execução, sem conhecimento prévio do número de passos nem do conteúdo devolvido por cada ferramenta.

Sua implementação depende da [nota 02](02-o-agente-e-o-estado.md): reescrever o histórico exige que ele seja **campo do estado**, e não a estrutura em que o agente reside. Um agente cujo estado é `mensagens[]` não pode comprimir o histórico sem se apagar.

---

### 3. Sub-agentes como isolamento de contexto

Esta tática difere das anteriores: em vez de comprimir o contexto posteriormente, **impede sua formação**.

```
   PRINCIPAL                            SUB-AGENTE
   ┌──────────────────┐                 ┌──────────────────┐
   │ objetivo         │  ──delega──>    │ subtarefa        │
   │ ...              │                 │ 14 passos        │
   │ [resumo 1,2k]    │  <──devolve──   │ 60k tokens       │
   │ ...              │                 │ (descartados)    │
   └──────────────────┘                 └──────────────────┘
     consome 1,2k                         consumiu 60k, e encerra
```

É a tática adequada quando a subtarefa é **verificável e delimitada**: *"identifique em qual artigo da política esta categoria de despesa é tratada e devolva o texto do artigo"*. Entra uma pergunta, sai uma resposta curta.

O custo é o mais elevado das quatro táticas:

> **O que o sub-agente não incluir no resumo, o principal jamais conhecerá.** Não se trata de compressão, e sim de barreira. O principal não dispõe de meio para constatar a omissão, uma vez que nunca teve acesso ao conteúdo original.

Daí a restrição: sub-agente para subtarefa **fechada**, com critério de sucesso explícito. Sub-agente para tarefa exploratória — *"investigue e relate"* — constitui forma custosa de perder informação.

Este é o ponto em que a aula toca em sistemas multiagente e se detém. Coordenação, protocolo de comunicação e revisão mútua entre agentes são objeto de aula própria. Desta nota permanece o uso **arquitetural** do sub-agente, como técnica de gestão de contexto.

---

### 4. Ordem de aplicação

| # | Tática | Destrutiva | Aplicação | Onde |
|---|---|---|---|---|
| **0** | **Encurtar o retorno das ferramentas** | não | **antes de qualquer outra** | esta nota |
| 1 | *Tool clearing* | não — reversível, o dado permanece no estado | a partir de ~5 passos | Aula 08 |
| 2 | *Compaction* periódica + reativa | sim, de forma controlada | trajetórias longas (>10 passos) | Aula 08 |
| 3 | Sumarização agressiva | sim | quando a trajetória é o custo dominante | Aula 08 |
| 4 | Sub-agentes | sim, irreversivelmente | subtarefa fechada e verificável | §3 desta nota |

A tática zero tem retorno superior ao das quatro seguintes. Uma ferramenta que devolve 900 tokens onde 60 seriam suficientes constitui defeito de projeto da ferramenta (Aula 03, nota 04, §7), e corrigi-lo na origem é sempre preferível a comprimir posteriormente. **Antes da primeira linha de compaction, examinam-se os retornos das ferramentas.**

A coluna da direita é o mapa desta nota: **a linha 0 e a linha 4 se resolvem aqui**, porque uma é projeto de ferramenta e a outra é decisão de arquitetura. As três do meio são técnicas de compressão, e a Aula 08 as implementa junto da política de escrita da memória — que decide exatamente as mesmas coisas, sobre um material diferente.

---

### 5. Encerramento da lista de pendências

| Pendência (Aula 03, nota 04, §9) | Situação |
|---|---|
| ~~outros padrões de arquitetura~~ | **encerrada** — [nota 01](01-0-padroes-de-arquitetura.md) e a série [01-1](01-1-sequencial.md) a [01-6](01-6-qual-padrao-usar.md) |
| ~~estado, no interior de uma execução~~ | **encerrada** — [nota 02](02-o-agente-e-o-estado.md) |
| ~~múltiplas ferramentas e a escolha entre elas~~ | **encerrada** — nota 02, §6 |
| ~~confiabilidade: retentativa, laço, orçamento, checkpoint~~ | **encerrada** — [nota 03](03-confiabilidade.md) |
| ~~context engineering dinâmica~~ | **encerrada** — esta nota |
| **memória entre execuções** | aula de memória |
| **MCP** | aula de MCP |
| **multiagente** (coordenação) | aula de sistemas multiagente |

Uma pendência nova foi criada por esta aula: o **trace** registrado a partir da nota 03, §10, é a matéria-prima da aula de **evals** e da **observabilidade** que a aula de produção trata. O encerramento de um assunto abre outro, com a diferença de que as perguntas subsequentes são mais precisas.

---

## Exemplos

### Exemplo 1 — A tática zero, medida

A mesma ferramenta, sob dois projetos de retorno:

```python
# 900 tokens: devolve o artigo íntegro da política
{"artigo": "Art. 4º — Das despesas com alimentação. §1º O reembolso de
 refeições em viagem a serviço fica limitado ... [38 linhas] ..."}

# 60 tokens: devolve o que a decisão requer
{"categoria": "refeicao", "teto": 120.00, "exige_nf": true,
 "artigo": "4º §1º", "excecoes": ["viagem internacional: teto 260,00"]}
```

Numa trajetória de 12 passos com 4 consultas à política, a diferença é de aproximadamente 3.400 tokens **por volta** a partir da quarta consulta. Nenhum mecanismo de *compaction* recupera esse volume ao custo de reescrever o `return` da ferramenta.

### Exemplo 2 — A informação que o sub-agente omitiu

```
PRINCIPAL delega:  "qual o teto de reembolso para refeição?"
SUB-AGENTE (9 passos, 22k tokens) devolve:  "R$ 120,00 por refeição."

Passo seguinte do principal: aprova uma refeição de R$ 180,00 em Lisboa.
```

O sub-agente havia lido a exceção — *viagem internacional: teto R\$ 260,00* — e não a incluiu, porque a pergunta não a solicitava. O principal aplicou o teto de R\$ 120,00 e reprovou uma despesa legítima, **sem qualquer indício de informação faltante**.

A distinção é essa: *compaction* descarta detalhe e deixa vestígio no resumo; sub-agente descarta detalhe sem deixar vestígio algum. Daí a exigência de que a pergunta delegada carregue o critério íntegro: *"qual o teto aplicável a esta despesa, considerados destino e exceções?"*

---

## Fontes e leituras

ANTHROPIC. **Effective context engineering for AI agents**. 2025. (*Compaction*, *tool clearing* e sub-agentes como táticas de gestão de contexto ao longo da trajetória.)

ANTHROPIC. **Building effective agents**. 2024. (Seção sobre agentes autônomos e o custo da trajetória.)

LIU, N. F. et al. **Lost in the middle**: how language models use long contexts. arXiv:2307.03172, 2023. (Fundamenta a reancoragem do objetivo.)

**Material da disciplina.** Aula 03, [nota 02](../../aula-03-prompt-engineering/notas-de-aula/02-context-engineering.md) — context engineering estática, cuja promessa esta nota cumpre; [nota 03](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) — versionamento, aplicável ao prompt de *compaction*; [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) §7 — projeto de ferramenta, origem da tática zero. Aula 02, [nota 01](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) §4 — janela, custo quadrático da atenção e *lost in the middle*; [nota 04](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md) §5 — latência e tamanho da entrada.
