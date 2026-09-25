# Exercício 8 — A memória do seu agente

## Contexto

O Exercício 6 decidiu de onde o agente do trabalho tira o que sabe, e o
Exercício 7 o fez responder citando a fonte. Ele consulta documentos, e não
aprende nada: **o agente não lembra de nada**. Cada execução começa do zero,
repete as mesmas consultas e comete o mesmo erro que cometeu ontem.

Este exercício não constrói peça nenhuma. **O grupo decide quais peças o
agente do trabalho precisa** — e a decisão precede a construção, porque
construir a peça errada custa a semana inteira.

São duas decisões, e as duas são de projeto:

| | A decisão | A pergunta que ela responde |
|---|---|---|
| **1** | como o agente **lembra** | o que é curto prazo, o que é longo prazo, e o que cada nível guarda no domínio do case |
| **2** | como o agente **esquece** | contradição, decaimento e remoção: quando cada uma dispara, e como se verifica que funcionou |

> **Não se escreve código aqui.** A entrega é um texto, e ele é o rascunho
> direto de `docs/memoria.md`, que a **Parte 2 do trabalho** cobra no §4.
>
> O par prático é o [08-complementar.md](exercicio_08-complementar.md): lá o grupo
> implementa, no repositório do case, as decisões que este documento obriga a
> tomar.

> **Questão a ser respondida ao final:** o que o seu agente **não** guarda —
> e o que ele perde, quando perde.

## Objetivo

1. **Declarar** a fronteira entre memória de curto prazo e de longo prazo no
   domínio do case.
2. **Especificar** as três causas de esquecimento, e demonstrar que a remoção
   alcança todas as estruturas.

Onde houver medição, ela vem do que os exercícios complementares desta aula
já produziram.

---

## Decisão 1 — Como o agente lembra

### 1.1 Os dois níveis, separados

A literatura divide a memória de um agente em dois níveis antes de qualquer
taxonomia mais fina, e confundi-los produz sistemas que serializam tudo e não
lembram de nada:

| | curto prazo | longo prazo |
|---|---|---|
| o que é | a janela de contexto da execução corrente | o que atravessa execuções |
| conteúdo | system prompt, objetivo, trajetória, trechos recuperados | episódica, semântica, procedural |
| persistido como | **checkpoint** — estado íntegro, um arquivo por execução | índice, tabela e texto |
| lido | uma vez, no início da retomada | por relevância, a cada volta do laço |
| acesso | por identificador de execução | por relevância ou por chave |
| ciclo de vida | morre com a execução | acumula |

Declare, **no domínio do case**, o que ocupa cada célula. A coluna da
esquerda costuma ser esquecida porque parece óbvia, e não é: o que exatamente
precisa estar no checkpoint para que a execução seja retomável sem repetir
efeito colateral?

### 1.2 O curto prazo: o orçamento da janela

A memória de longo prazo é **mais uma fonte** competindo pela janela, ao lado
do system prompt, do objetivo, da trajetória e dos trechos recuperados. São
cinco fontes.

Declare o **teto de cada fonte**, em tokens, e o que é descartado primeiro
quando o total estoura. Um sistema sem essa decisão a toma sozinha, e a toma
mal: trunca no fim, que é onde costuma estar o mais recente.

Declare também o que o checkpoint guarda, e a resposta precisa ser específica
o bastante para responder a três perguntas:

- a execução pode ser **retomada** de onde parou, sem reaplicar efeito
  colateral já aplicado?
- uma aprovação humana pode chegar **horas depois**, de outro processo?
- um estado defeituoso pode ser **carregado e reproduzido** para depuração?

### 1.3 O longo prazo: as três memórias no case

Uma linha por tipo, com conteúdo do seu domínio — não do exemplo da aula:

| Tipo | O que guarda no case | Estrutura | Como é recuperada |
|---|---|---|---|
| **episódica** | | índice por similaridade | |
| **semântica** | | chave-valor | |
| **procedural** | | texto no *system prompt* | |

A estrutura de cada uma decorre do padrão de acesso, e não da preferência:
recuperar um fato semântico por similaridade é mais caro e menos exato que
resolvê-lo por chave.

### 1.4 Quem escreve, e o que não entra

Declare a política de escrita — o agente decide, o código extrai por regra,
ou o humano corrige — com o **volume por execução**: quantos registros, de
que tamanho.

E declare **o que o sistema não guarda**, que é a parte que vale mais:

- dado sensível — identificador pessoal, credencial, valor que não precisa
  persistir;
- conteúdo que veio de fora sem verificação — memória é durável, e o que
  entra nela sai muitas vezes;
- o que é derivável — se pode ser recalculado, guardar é dívida.

---

## Decisão 2 — Como o agente esquece

Esta é a parte que quase nenhum projeto especifica, e é a que separa um
sistema operável de um que acumula até quebrar.

São **três causas distintas**, e elas diferem no momento em que a decisão
ocorre e no que acontece com o dado:

| Causa | O que aconteceu | O dado é apagado? | Quando decide |
|---|---|---|---|
| **contradição** | o fato mudou | não | na leitura |
| **decaimento** | o fato envelheceu | sim, ou é rebaixado | em rotina |
| **remoção** | o titular solicitou | sim, obrigatoriamente | sob demanda |

### 2.1 Contradição — o fato que mudou

Identifique no domínio **dois fatos verdadeiros em datas diferentes** que
respondam à mesma pergunta: uma política reajustada, uma regra revogada, um
limite alterado. Todo domínio real tem um.

Os dois são igualmente similares à pergunta, porque o vetor não representa
anterioridade — é a cegueira de TEMPO da
[nota 02 da aula 07](../../aula-07-rag-e-documentos/notas-de-aula/02-o-que-o-embedding-nao-ve.md),
§5.

Declare a regra de desempate, e ela é determinística: carimbo de tempo
obrigatório em todo fato, e `max()` sobre o carimbo. Delegar o desempate ao
modelo tem três desvantagens — passa a errar, consome uma chamada, e deixa de
ser testável.

Declare também que o **descarte é registrado**. Um agente que ignora
silenciosamente um fato contraditório é indistinguível, no log, de um que
nunca o recuperou.

### 2.2 Decaimento — o fato que envelheceu sem ser contradito

Declare o **corte**, em dias, e a justificativa dele no domínio. O valor
adequado depende da taxa de mudança dos fatos armazenados: cinco meses pode
ser curto ou longo conforme o que se guarda.

E declare o efeito: o fato antigo é **removido** ou apenas **rebaixado** na
ordenação? As duas são defensáveis, e têm consequências diferentes para a
terceira causa.

### 2.3 Remoção — o titular solicitou

O requisito é de **cobertura**: o dado precisa sair de todas as estruturas em
que foi gravado, e não apenas das que vieram à lembrança.

Liste as estruturas em que o identificador do titular pode ter caído. A lista
tem mais itens que a taxonomia sugere, e três deles escapam com frequência:

- o **checkpoint**, que não é uma das três memórias e guarda os argumentos de
  cada passo;
- o **log**, se ele registra os argumentos das chamadas de ferramenta;
- a memória **procedural**, no caso em que uma regra aprendida a partir do
  erro de um titular mencione esse titular.

Especifique a **verificação**, e ela precisa ser independente da remoção:
grave, remova, e então varra as estruturas procurando o identificador. Uma
remoção que se declara concluída sem varredura não cumpre a obrigação — ela
apenas afirma tê-la cumprido.

> **Se o esquecimento exige reconstruir o índice inteiro, declare isso.** Ele
> continua servindo para privacidade e para custo, e deixa de servir para
> recuperação de incidente: uma memória envenenada que só sai com
> reconstrução completa é, na prática, permanente até a próxima janela de
> manutenção.

### 2.4 O preço: o sistema deixou de ser reprodutível

A mesma entrada, no mesmo modelo, com os mesmos parâmetros, produz saída
diferente amanhã — porque a memória mudou.

Isto não é defeito a corrigir. É consequência de projeto, e foi escolhida
deliberadamente na Decisão 1. Registre-a, porque a aula 11 vai ter
de conviver com ela: um conjunto de avaliação que roda sobre um sistema com
memória mede duas coisas ao mesmo tempo, e separá-las é trabalho.

---

## O que entregar

Um documento, no repositório do trabalho, em `docs/memoria.md`. É o rascunho
direto do que a Parte 2 cobra no §4.

Quatro itens obrigatórios:

1. **A tabela dos dois níveis do §1.1**, preenchida com o domínio do case —
   incluindo o que exatamente vai para o checkpoint.
2. **O orçamento da janela do §1.2**, com o teto de cada uma das cinco fontes
   e o que é descartado primeiro quando o total estoura.
3. **A tabela das três memórias do §1.3**, com a estrutura e o padrão de
   acesso de cada uma, mais a lista do §1.4 do que **não** entra.
4. **A especificação das três causas de esquecimento** — contradição,
   decaimento e remoção —, com a lista de estruturas do §2.3 e o
   procedimento de verificação.

Extensão sugerida: três a cinco páginas. O que se avalia é a **justificativa
de cada decisão** — um documento que declara o que o sistema não guarda, com
a razão, vale mais que um que enumera tudo o que guarda.

## Dicas

- Comece pela Decisão 2. Decidir o que o sistema esquece torna a Decisão 1
  mais fácil, porque a lista de exclusão restringe a de inclusão.
- A contradição do §2.1 é fácil de encontrar em qualquer domínio real:
  procure uma regra que mudou de valor.
- A lista de estruturas do §2.3 é a parte que mais rende. Se ela tiver apenas
  as três memórias, provavelmente está incompleta — o checkpoint e o log
  guardam identificador e não costumam entrar na conta.
- Os exercícios complementares desta aula produzem as medições que este
  documento cita. Fazê-los antes reduz o trabalho aqui a escrever decisões,
  em vez de tomá-las sem dado.
