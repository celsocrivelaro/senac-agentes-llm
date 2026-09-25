# Exercício 7 (complementar) — A resposta que cita, e que recusa

> **Este é o exercício complementar da aula 07, e é o único dela.** Não há
> entrega avaliada nesta aula: o que ele constrói é cobrado no **§3 da Parte 2
> do trabalho**, e o exercício existe para que essa construção não comece do
> zero na semana da entrega.

## Contexto

O Exercício 6 entregou um buscador: dada uma pergunta, ele devolve trechos.
Nenhuma resposta ainda.

A Parte 2 do trabalho exige **RAG como memória consultável**, e o enunciado é
específico quanto ao que isso significa: *vocês definem o que entra, como é
indexado e **como a resposta cita a fonte***. Este exercício entrega a
geração — com as duas exigências que separam RAG de "colar o documento no
prompt": **citação verificável** e **recusa**.

> **Questão a ser respondida ao final:** o sistema recusa quando deve? E
> quantas vezes ele recusa quando **não** deveria?

## Objetivo

Sobre o índice do Exercício 6:

1. implementar o **pipeline completo** — recuperar, montar contexto, gerar;
2. produzir **citação verificável**, e não apenas plausível;
3. implementar o **portão** que faz o sistema recusar;
4. medir **`recall` e fidelidade separadamente**, e diagnosticar a falha.

## Requisitos

### 1. O pipeline

Recuperação → montagem de contexto → geração, com o `k` justificado do
exercício anterior. A ordem dos trechos no contexto importa: reporte o efeito
de reordená-los em ao menos uma pergunta.

### 2. A citação verificável

A resposta precisa citar a fonte, e a citação precisa ser **verificada em
código**:

```
o identificador citado está entre os trechos recuperados?
```

Uma citação que o modelo produziu mas que não está no contexto é alucinação
com aparência de rigor — e é o modo de falha mais perigoso do RAG, porque
parece o oposto.

Reporte a **taxa de citações verificáveis**, e não apenas a existência de
citação.

### 3. O portão, e a recusa

Implemente o limiar abaixo do qual o sistema **não responde**. Duas decisões:

- **onde fica o limiar**, e ele precisa ser medido — não escolhido no chute;
- **o que o sistema diz** quando recusa. "Não sei" é pior que "o regulamento
  não trata deste assunto; o mais próximo que encontrei foi X".

E meça os dois erros, que são diferentes:

| Erro | O que é | Custo |
|---|---|---|
| **falso positivo** | respondeu quando devia recusar | resposta errada com aparência de fundamentada |
| **falso negativo** | recusou quando podia responder | sistema inútil |

Um limiar que zera o primeiro e dispara o segundo produz um sistema que
ninguém usa. Declare a troca escolhida.

### 4. As duas métricas, separadas

`recall` mede a **busca**. Fidelidade mede a **geração**. Medi-las juntas
impede o diagnóstico.

Construa a tabela que separa os três casos:

| `recall` | fidelidade | Onde está o defeito |
|---|---|---|
| baixo | — | índice ou *chunking* |
| alto | baixa | prompt de resposta |
| alto | alta, e a resposta está errada | corpus |

A terceira linha é a que quase ninguém considera, e é a mais desconfortável:
o sistema fez tudo certo sobre um documento desatualizado.

### 5. Os quatro modos de falha

Reproduza os quatro no case do trabalho, e reporte o `trace` de cada um:

- **não recuperou** o que precisava;
- **recuperou e ignorou** — o trecho estava lá, no meio do contexto;
- **recuperou e contradisse** o contexto;
- **respondeu sem base** — deveria ter recusado.

### 6. O corpus desatualizado

Introduza no corpus **um documento revogado ou obsoleto** que responda a
uma das perguntas, e verifique o que acontece. O sistema tem como saber que
aquele documento não vale mais?

Se não tiver, isso é um achado — e a resposta não é técnica de recuperação, é
**curadoria de corpus**.

### 7. O carimbo

Esta aula acrescenta **o `k`** e **o prompt de resposta**, junto com a
estratégia de *chunking* da Aula 06.

## O que deve sair na tela

```
PIPELINE
  k=<n>   limiar=<n>   prompt de resposta v<x.y>

MÉTRICAS (<n> perguntas)
  recall@k ................. <n>%
  citações verificáveis .... <n>%
  fidelidade ............... <n>%
  recusas corretas ......... <n>/<n>
  recusas indevidas ........ <n>/<n>

DIAGNÓSTICO POR PERGUNTA
  <id>  recall <ok|falhou>  fidelidade <ok|falhou>  -> <índice|prompt|corpus>

MODOS DE FALHA REPRODUZIDOS
  não recuperou ......... <pergunta>
  recuperou e ignorou ... <pergunta>   (posição do trecho: <n> de <n>)
  contradisse ........... <pergunta>
  respondeu sem base .... <pergunta>

CORPUS DESATUALIZADO
  documento revogado: <id>   o sistema <detectou|não detectou>
```

## Desafios opcionais

**A.** Faça a recuperação virar **ferramenta do agente** — o agente decide
quando buscar, em vez de o pipeline sempre buscar. Compare as duas versões em
número de chamadas e em `recall` efetivo, e indique qual das duas deve ser
mantida.

**B.** Construa uma pergunta que exija **três trechos diferentes** para ser
respondida, e verifique se o seu `k` a cobre. Perguntas multi-salto são onde
o `recall` alto engana: o sistema recupera dois dos três e responde com
confiança.

## O que fica registrado

Este exercício não é avaliado isoladamente. O produto dele entra na Parte 2
do trabalho, e convém deixá-lo no repositório em forma utilizável:

- o pipeline, o portão e os medidores, com **parâmetros e justificativas no
  código**;
- em `exercicios/aula-07-resposta-que-cita.md`: a tabela de diagnóstico por
  pergunta, a justificativa do limiar com os dois erros medidos, e o achado
  do corpus desatualizado;
- o carimbo com `k` e a versão do prompt de resposta.

> **Onde isto é cobrado:** §3.4, §3.5 e §3.6 da
> [Parte 2 do trabalho](../../../trabalho/02-segunda-entrega.md), que pedem
> `docs/rag.md`. O texto produzido aqui é o rascunho dele.

## Dicas

- Meça o limiar antes de escolhê-lo: rode com vários valores e olhe a curva
  dos dois erros.
- A pergunta sem resposta no corpus, do Exercício 6, é a que valida o portão.
  Se ela não estiver lá, o portão não foi testado.
- Se a sua fidelidade está em 100%, verifique se o juiz — ou o critério —
  está medindo alguma coisa.
