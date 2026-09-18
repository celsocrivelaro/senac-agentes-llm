# Exercício 6 (complementar) — O buscador do regulamento

> **Este é o exercício complementar da aula.** O que vale nota é o
> [exercicio_06-trabalho.md](exercicio_06-trabalho.md), em que você decide a base de conhecimento
> do seu próprio case. Aqui você implementa sobre um domínio já dado, para
> sentir no código o que aquelas decisões cobram.

## Contexto

A Parte 2 do trabalho exige **RAG como memória consultável** — não um chatbot
sobre PDFs, mas o conhecimento de domínio que os agentes precisam para
decidir. A aula seguinte constrói a geração; esta constrói o que vem antes, e
sem o que a geração não tem o que citar: **a busca**.

O laboratório mostrou que quatro coisas escapam ao vetor — negação, número,
entidade e tempo — e que a estratégia de corte muda o `recall` mais do que a
escolha do modelo de embedding.

> **Questão a ser respondida ao final:** qual pergunta o seu buscador erra, e
> o erro é do vetor, do *chunking* ou da pergunta?

## Objetivo

Sobre o corpus do **próprio case** — ou, se ele ainda não tiver corpus, sobre
o regulamento de despesas do laboratório:

1. construir o **conjunto de perguntas com resposta conhecida**;
2. implementar **três estratégias de *chunking*** e medi-las com `recall@k`;
3. escolher uma, **com número**;
4. encontrar e diagnosticar **a pergunta que falha**.

## Requisitos

### 1. O conjunto de perguntas, antes de qualquer medida

No mínimo **dez perguntas**, cada uma com o **trecho que a responde**
identificado. Construir isso é o trabalho do exercício; medir é fácil depois.

Exigências de composição:

- ao menos **duas perguntas cuja resposta depende de uma exceção** —
  parágrafo, ressalva, caso especial;
- ao menos **uma pergunta cuja resposta não está no corpus**. Um buscador que
  sempre devolve algo não está sendo avaliado no comportamento mais
  perigoso.

Este conjunto é o **primeiro *dataset* de avaliação do curso**, e a Aula 11 o
retoma sob esse nome. Escreva-o como se fosse durar o semestre — porque vai.

### 2. As três estratégias de *chunking*

Implemente e compare:

- por **contagem de caracteres**;
- por contagem **com sobreposição**;
- por **estrutura** do documento — artigo, seção, parágrafo, o que o seu
  corpus tiver.

Meça `recall@k` das três sobre o mesmo conjunto, com o mesmo `k`.

### 3. O `k`, justificado

`k` é decisão de projeto, não constante. `k` alto esconde defeito de índice e
enche a janela da aula seguinte; `k` baixo perde a resposta.

Declare o `k` escolhido e a razão — e reporte o `recall` em ao menos **dois
valores de `k`**, para mostrar a sensibilidade.

### 4. O RAG mínimo, e o limiar do portão

Monte o pipeline: `pergunta → busca → portão → contexto + LLM → resposta`,
com o índice num banco de vetores.

E **calibre o portão**. O valor de 0,6 de distância do laboratório não barra
nada — a nota 04 mostra por quê. Acrescente ao seu conjunto ao menos **três
perguntas sem resposta no corpus** e escolha o limiar que barra essas três
sem barrar as que têm resposta.

Reporte o limiar escolhido, quantas perguntas ele barra de cada lado, e o que
acontece com uma pergunta sem resposta quando o portão está desligado.

### 5. A pergunta que falha

Toda configuração tem uma. Ache a sua e diagnostique:

- é **chunking** — o trecho certo foi cortado ao meio, ou diluído num bloco
  grande?
- é a **pergunta que está mal feita**?
- ou é o **vetor**, que não representa o que a pergunta exige — número,
  identificador, negação ou data?

Metade da nota está aqui. O `recall` médio esconde exatamente isto.

### 6. O carimbo

Esta aula acrescenta **a estratégia de *chunking***. Trocar o corte invalida a
comparação de `recall` com as execuções anteriores, exatamente como trocar o
prompt invalidava a suíte da Aula 03.

O **modelo de embedding** também entra: reindexar com outro modelo muda o
`recall` sem que ninguém tenha mudado o código.

## O que deve sair na tela

```
CORPUS: <n> documentos, <n> tokens

CONJUNTO: <n> perguntas
  com exceção: <n>   sem resposta no corpus: <n>

RECALL@k
  estratégia            k=3      k=5
  caracteres           <n>%     <n>%
  caracteres+overlap   <n>%     <n>%
  estrutura            <n>%     <n>%
  ESCOLHIDA: <qual>   k=<n>   porque <razão>

PORTAO
  limiar escolhido: <n> de distancia
  barra <n>/<n> das perguntas sem resposta
  barra <n>/<n> das perguntas com resposta   <- tem que ser 0

A PERGUNTA QUE FALHA
  <pergunta>
  esperado: <trecho>   recuperado: <trechos>
  DIAGNÓSTICO: <chunking | pergunta mal feita | limitacao do vetor>

CARIMBO
  modelo de embedding ... <qual>
  chunking .............. <qual>
```

## Desafios opcionais

**A.** Compare `mistral-embed` com um **segundo modelo de embedding** no mesmo
conjunto de perguntas. É a Aula 02 — escolha de modelo — aplicada a uma
modalidade nova. Reporte se a diferença de `recall` justifica a troca.

**B.** Construa uma pergunta em que o trecho **mais similar** não seja o que
**responde**, e explique por quê. É a distinção entre similaridade e
relevância, no seu domínio.

## Entrega

No repositório do trabalho:

- o indexador, as três estratégias e o medidor, com **parâmetros e
  justificativas no código**;
- em `docs/`: o conjunto de perguntas com o trecho-resposta de cada uma, a
  tabela de `recall@k`, o limiar do portão calibrado e o diagnóstico da
  pergunta que falha;
- o carimbo com o modelo de embedding e a estratégia de corte.

## Dicas

- Escreva as dez perguntas **antes** de indexar qualquer coisa. Quem indexa
  primeiro escreve perguntas que o índice já responde.
- A pergunta sem resposta no corpus é a mais informativa das dez.
- Se as três estratégias derem o mesmo `recall`, o seu conjunto de perguntas
  é fácil demais.
