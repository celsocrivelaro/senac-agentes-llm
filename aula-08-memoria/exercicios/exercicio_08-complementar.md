# Exercício 8 (complementar, de código) — A memória do case

> **Este é o exercício complementar da aula 08.** O que vale nota é o
> [enunciado.md](exercicio_08.md), que é de projeto e alimenta a Parte 2 do
> trabalho. Este implementa, no repositório do case, as decisões que aquele
> obriga a tomar.

## Contexto

O Exercício 7 entregou um sistema que consulta documentos. Ele não aprende
nada: a execução de amanhã começa exatamente onde a de hoje começou.

Este exercício acrescenta memória entre execuções — e com ela um problema
novo, que a Aula 11 vai ter de resolver: **o sistema deixa de ser
reprodutível**. A mesma entrada, no mesmo modelo, com os mesmos parâmetros,
produz saída diferente amanhã, porque a memória mudou.

> **Questão a ser respondida ao final:** o que o sistema **não** guarda —
> e por quê?

## Objetivo

1. implementar **as três memórias** nas estruturas adequadas;
2. escolher e defender uma **política de escrita**;
3. resolver a **contradição** entre fatos verdadeiros em datas diferentes;
4. implementar o **esquecimento seletivo**.

## Requisitos

### 1. A fronteira com o checkpoint

Antes de implementar, declare a diferença no case do trabalho:

| | checkpoint (Aula 05) | memória (Aula 08) |
|---|---|---|
| escopo | uma execução | todas |
| propósito | **retomar** | **lembrar** |
| forma | estado íntegro | fragmento selecionado |
| leitura | uma vez, no início | por relevância, a cada volta |

Serializar o estado de todas as execuções **não produz memória**: produz
arquivo morto. Demonstre isso com número — quantos tokens ocuparia o arquivo
morto contra quantos ocupa a memória recuperada por relevância.

### 2. As três memórias, nas estruturas certas

| Tipo | O que guarda | Estrutura |
|---|---|---|
| **episódica** | o que aconteceu em execuções passadas | índice por similaridade |
| **semântica** | fatos sobre entidades do domínio | chave-valor |
| **procedural** | como fazer algo | texto no *system prompt* |

Cada uma na estrutura adequada, e a justificativa no código. Buscar um fato
semântico por similaridade é caro e impreciso quando uma chave resolve.

### 3. A política de escrita

Três opções, e cada uma tem preço:

| Política | Preço |
|---|---|
| o **agente** decide o que guardar | guarda demais, e guarda errado |
| o **código** extrai por regra | perde o que a regra não previu |
| o **humano** corrige | não escala |

Escolha uma — ou uma combinação — e declare o **volume de escrita por
execução**: quantos registros, e de que tamanho. O que esse volume custa em
dinheiro e em armazenamento é assunto da Aula 13.

### 4. O que NÃO entra

A parte mais importante do exercício, e a que vale mais nota.

Liste explicitamente o que o sistema **não** guarda:

- **dado sensível** — identificador pessoal, credencial, valor que não
  precisa persistir;
- **conteúdo que veio de fora sem verificação** — e esta linha é a que a Aula
  14 vai cobrar: memória é durável, e o que entra nela **sai muitas vezes**;
- **o que é derivável** — se pode ser recalculado, guardar é dívida.

### 5. A escrita é escrita

Gravar memória tem efeito no mundo, e portanto exige **chave de idempotência
derivada do conteúdo**, com `ja_existia` no retorno. É a mesma regra da Aula
05, aplicada a um lugar em que quase ninguém a aplica.

`uuid4()` a cada chamada não é chave de idempotência.

### 6. A contradição

Introduza no domínio do case **dois fatos verdadeiros em datas diferentes** que
respondam à mesma pergunta — uma política que mudou, um valor reajustado, uma
regra revogada.

Verifique o que a busca por similaridade faz com eles: os dois são igualmente
similares à pergunta, porque **o vetor não tem noção de anterioridade** — a
cegueira de tempo, plantada na Aula 06.

E resolva **em código**: carimbo de tempo obrigatório em todo fato, e regra de
desempate determinística. Delegar isso ao modelo é o antipadrão.

### 7. O esquecimento seletivo

Implemente a operação de remoção de um registro específico, e demonstre que
ela funciona: grave, verifique que o comportamento mudou, remova, verifique
que voltou.

Três razões para ela existir, e a terceira só aparece na Aula 14:

- **privacidade** — o titular pede a remoção;
- **custo** — o crescimento é monotônico;
- **recuperação de incidente** — sem ela, um envenenamento de memória é
  permanente, e cada execução futura repete o comportamento injetado.

### 8. A não reprodutibilidade

Execute **a mesma entrada duas vezes, com memórias diferentes**, e reporte a
diferença.

Isso é o que a Aula 11 vai chamar de terceira fonte de não determinismo — e é
a pior das três, porque não é acidente: é **consequência de projeto**.

### 9. O carimbo

Esta aula acrescenta **o estado da memória**. Duas execuções com memórias
diferentes não são comparáveis, e tratá-las como comparáveis produz números
que variam sem que nada tenha sido alterado.

## O que deve sair na tela

```
FRONTEIRA
  arquivo morto (n execuções serializadas) ... <n> tokens
  memória recuperada por relevância .......... <n> tokens

AS TRÊS MEMÓRIAS
  episódica  <n> registros   consulta: <n> ms
  semântica  <n> chaves      consulta: <n> ms
  procedural <n> tokens no system prompt

POLÍTICA DE ESCRITA: <qual>
  volume por execução: <n> registros, <n> tokens
  o que NÃO entra: <lista>

IDEMPOTÊNCIA
  1ª gravação -> <id>  ja_existia=false
  2ª gravação -> <id>  ja_existia=true

CONTRADIÇÃO
  fato A [<data>]: <texto>   similaridade <n>
  fato B [<data>]: <texto>   similaridade <n>
  desempate por carimbo: <qual venceu>

ESQUECIMENTO
  antes da remoção: <comportamento>
  depois:           <comportamento>

NÃO REPRODUTIBILIDADE
  execução com memória vazia .... <resposta>
  execução com memória cheia .... <resposta>
```

## Desafios opcionais

**A.** Meça a **degradação com o crescimento**: encha a memória com centenas
de registros e verifique o que acontece com a precisão da recuperação e com a
latência da consulta. Proponha a política de retenção que decorre da
medição.

**B.** Faça a memória **procedural** ser escrita pelo próprio sistema a partir
de erros observados — e depois explique, em uma frase, por que essa
capacidade é exatamente o que a Aula 14 vai tratar como vetor de ataque.

## O que fica registrado

Este exercício não é avaliado isoladamente. O produto dele entra na Parte 2
do trabalho:

- as três memórias, a política de escrita, o desempate e o esquecimento, com
  **parâmetros e justificativas no código**;
- em `exercicios/aula-08-memoria-do-case.md`: a tabela da fronteira com o
  checkpoint, a política de escrita com o volume declarado, **a lista do que
  não entra**, e o registro da não reprodutibilidade;
- o carimbo com o estado da memória.

> **Onde isto é cobrado:** §4 da
> [Parte 2 do trabalho](../../../trabalho/02-segunda-entrega.md), que pede
> `docs/memoria.md`. O texto produzido aqui é o rascunho dele.

## Dicas

- Comece pelo requisito 4 — o que **não** guardar. É mais fácil decidir o que
  guardar depois de ter a lista de exclusão.
- A contradição do requisito 6 é fácil de encontrar em qualquer domínio real:
  procure uma regra que mudou de valor.
- Se o seu esquecimento seletivo exige reconstruir o índice inteiro, ele não
  serve para recuperação de incidente — e a Aula 14 vai cobrar isso.
