# Exercício 14 — Os dez riscos, no case do grupo

## Contexto

A Parte 3 do trabalho cobra segurança com uma pergunta direta: *o que o
sistema faria se a entrada fosse hostil?*

O catálogo da OWASP para aplicações agênticas (ASI01 a ASI10) lista dez
riscos, e nem todos pesam o mesmo em todo sistema. Um agente sem memória não
sofre envenenamento de memória; um agente que só lê não move dinheiro. O
trabalho de segurança começa por **priorizar**: decidir quais riscos existem
no case do grupo, e qual o dano de cada um, antes de gastar esforço em defesa.

Um termo organiza o exercício:

- **Mitigação** é a medida que impede o risco de se materializar ou reduz o
  seu dano — menor privilégio por ferramenta, confirmação humana inescapável
  por prompt, credencial delegada, versão fixada, um dos seis padrões de
  defesa da nota 02.

> **Questão a ser respondida ao final:** quais são os riscos de maior impacto
> no case do grupo, e qual a mitigação de cada um?

## Objetivo

1. **Avaliar** os dez riscos no case do grupo: se existem, onde, com que
   impacto e com que mitigação;
2. **Declarar** ao menos um risco aceito.

## Requisitos

### 1. Os dez riscos, no case

Avalie cada risco no case do grupo e preencha um quadro por risco. Se o risco
**não existe**, escreva **não aplicável** na primeira linha e deixe as demais
vazias.

| Campo | O que se escreve |
|---|---|
| **Existe no case?** | sim, ou **não aplicável** — quando a capacidade correspondente não está no sistema (o agente não executa código, não tem memória, não usa sub-agente) |
| **Onde** | o componente do sistema — a ferramenta, o índice de RAG, a memória, o servidor MCP, o `Dockerfile` —, com arquivo e linha quando houver código |
| **Cenário de ataque** | concreto e do case: quem é o atacante, por qual canal o texto ou a ação dele entra, e o que o agente faria. "Um atacante poderia injetar instruções" não é cenário |
| **Impacto** | nota de 1 a 3, pelos critérios abaixo, e o critério que a justifica |
| **Mitigação** | a medida, nomeada pelo vocabulário da aula, e o ponto do sistema em que ela entra |

Os critérios de impacto:

| Nota | Impacto |
|---|---|
| **1** | dano reversível e restrito a uma execução; nenhum dado pessoal envolvido |
| **2** | dano reversível que alcança outros usuários, ou dado interno exposto |
| **3** | ação irreversível (pagamento, envio, exclusão, registro de parecer), dado pessoal sob a LGPD exposto, ou alcance sobre todos os usuários |

Os dez quadros, a preencher:

#### ASI01 — Sequestro de objetivo do agente

| Campo | Resposta |
|---|---|
| Existe no case? | |
| Onde | |
| Cenário de ataque | |
| Impacto | |
| Mitigação | |

#### ASI02 — Uso indevido e exploração de ferramenta

| Campo | Resposta |
|---|---|
| Existe no case? | |
| Onde | |
| Cenário de ataque | |
| Impacto | |
| Mitigação | |

#### ASI03 — Abuso de identidade e privilégio

| Campo | Resposta |
|---|---|
| Existe no case? | |
| Onde | |
| Cenário de ataque | |
| Impacto | |
| Mitigação | |

#### ASI04 — Cadeia de suprimentos agêntica

| Campo | Resposta |
|---|---|
| Existe no case? | |
| Onde | |
| Cenário de ataque | |
| Impacto | |
| Mitigação | |

#### ASI05 — Execução inesperada de código

| Campo | Resposta |
|---|---|
| Existe no case? | |
| Onde | |
| Cenário de ataque | |
| Impacto | |
| Mitigação | |

#### ASI06 — Envenenamento de memória e de contexto

| Campo | Resposta |
|---|---|
| Existe no case? | |
| Onde | |
| Cenário de ataque | |
| Impacto | |
| Mitigação | |

#### ASI07 — Comunicação insegura entre agentes

| Campo | Resposta |
|---|---|
| Existe no case? | |
| Onde | |
| Cenário de ataque | |
| Impacto | |
| Mitigação | |

#### ASI08 — Falhas em cascata

| Campo | Resposta |
|---|---|
| Existe no case? | |
| Onde | |
| Cenário de ataque | |
| Impacto | |
| Mitigação | |

#### ASI09 — Exploração da confiança humano-agente

| Campo | Resposta |
|---|---|
| Existe no case? | |
| Onde | |
| Cenário de ataque | |
| Impacto | |
| Mitigação | |

#### ASI10 — Agentes desgovernados

| Campo | Resposta |
|---|---|
| Existe no case? | |
| Onde | |
| Cenário de ataque | |
| Impacto | |
| Mitigação | |

ASI07 e ASI08 podem existir mesmo num sistema de agente único: o resumo que um
sub-agente devolve e a ferramenta que falha no meio da execução. Só são não
aplicáveis se o sistema não tiver nenhum dos dois.

### 2. Um risco aceito, declarado

Escolha ao menos um risco que o grupo **decidiu não mitigar** — entre os
dez — e declare:

- qual é o risco;
- **por que foi aceito** — normalmente, porque a mitigação inviabiliza um
  requisito do produto;
- **o que o limita hoje**;
- **que evento faria reavaliar**.

Aceitar risco declarado é engenharia. Não declarar é omissão — e uma análise
sem nenhum risco aceito quase sempre significa que a análise não foi feita.

## O que deve sair no relatório

```
OS DEZ RISCOS
  ASI01  sim            <componente> (<arquivo>:<linha>)
         cenário ...... <cenário em uma linha>
         impacto ...... 3 — <critério>
         mitigação .... <medida>, em <ponto do sistema>
  ASI05  não aplicável
  ...

RISCO ACEITO
  <id> ... <risco> ... aceito porque <...> ... limitado por <...> ... reavaliar se <...>
```

## Entrega

No repositório do trabalho, em `exercicios/aula14-respostas.md`: os dez
quadros de risco e o registro do risco aceito.

> **Onde isto reaparece:** a **Parte 3 do trabalho** pede `docs/seguranca.md`. O que se escreve aqui é o rascunho dele — na entrega, consolidado em `docs/`. E a aula de sistemas multiagente volta a ASI07 e ASI08 quando o sistema for dividido em vários agentes.

## Dicas

- Avalie pelo **case**, e não pelo catálogo. O mesmo risco tem impacto alto
  num agente que envia e-mail e baixo num que só responde.
- Liste os canais por onde conteúdo não confiável entra no sistema antes de
  preencher os quadros: documento recuperado, retorno de ferramenta, memória e
  agente subordinado. Cada canal costuma aparecer em mais de um risco.
- Uma mitigação escrita no prompt de sistema não é mitigação: o texto injetado
  está no mesmo canal (nota 02, §8).
