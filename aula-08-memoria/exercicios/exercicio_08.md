# Exercício 8 — O projeto de recuperação e de memória

## Contexto

Os exercícios das aulas 06, 07 e 08 construíram peças: um índice, um pipeline
que cita e recusa, três memórias. Cada um resolveu um problema técnico
isolado.

Este exercício não constrói peça nenhuma. Ele decide **quais peças o case
precisa** — e a decisão precede a construção, porque construir a peça errada
custa a semana inteira.

São duas decisões, e as duas são de projeto:

1. **Como o sistema recupera conhecimento.** Há cinco formas de recuperar, e
   quatro delas não envolvem *embedding*. A aula 07 mediu o preço de escolher
   a errada.
2. **Como o agente lembra, e como esquece.** Memória é acúmulo, e acúmulo sem
   política de saída é dívida que cresce sozinha.

O produto é um documento de projeto, e ele é o rascunho direto de
`docs/rag.md` e `docs/memoria.md`, que a **Parte 2 do trabalho** cobra nos
§3 e §4.

> **Questão a ser respondida ao final:** quantas das perguntas do seu case
> realmente precisam de busca vetorial — e o que responde as outras?

## Objetivo

1. **Classificar** as perguntas do case pelo formato da resposta, e
   **selecionar** a forma de recuperação de cada classe.
2. **Declarar** a fronteira entre memória de curto prazo e de longo prazo no
   domínio do case.
3. **Especificar** as três causas de esquecimento, e demonstrar que a remoção
   alcança todas as estruturas.

Não há código obrigatório. Onde houver medição, ela vem do que os exercícios
complementares já produziram.

---

## Parte 1 — As extensões do RAG

### 1.1 Classificar as perguntas antes de escolher a ferramenta

Retome o conjunto de dez perguntas do Exercício 6 — o mesmo que mede
`recall@k`. Classifique **cada uma** pelo formato da resposta, na tabela da
[nota 01 da aula 07](../../aula-07-rag-e-documentos/notas-de-aula/01-as-formas-de-recuperar.md):

| A resposta é um… | Exemplo genérico | Recuperação adequada |
|---|---|---|
| **registro** | *"quem aprovou a D-4471?"* | consulta estruturada |
| **termo** | *"onde aparece 'força maior'?"* | busca textual |
| **assunto** | *"posso reembolsar almoço em viagem?"* | busca vetorial |
| **relação** | *"que artigos alteram o teto de refeição?"* | grafo |
| **agregação** | *"quais os temas recorrentes?"* | grafo, ou pré-cálculo |

Entregue a contagem: **quantas das dez caíram em cada linha.**

> Se as dez caíram em "assunto", o conjunto provavelmente foi escrito para o
> índice que já existia, e não para o que os usuários perguntam. Acrescente
> perguntas vindas de quem usa o sistema e refaça a contagem.

### 1.2 Decidir cada uma das quatro extensões

Para cada forma abaixo, a resposta é **usa** ou **não usa**, e as duas
precisam de justificativa. "Não usa" é resposta legítima e frequente — o que
não é legítimo é não ter decidido.

#### Consulta estruturada

O teste de um minuto: **se um `SELECT` responde a pergunta, não há RAG a
construir.** Declare quais perguntas do §1.1 caem aqui e de onde vêm os
parâmetros — expressão regular quando o campo tem formato fixo, ou modelo com
saída estruturada quando a redação varia.

Se usar, declare também a defesa contra *text-to-SQL*: a aplicação escreve a
consulta e o modelo preenche os parâmetros, nunca o contrário.

#### Busca textual

Ganha do vetor em identificador, nome próprio raro e citação exata. Declare
se o corpus do case tem esses elementos e se a busca textual já está
disponível na infraestrutura atual — `tsvector` mais índice GIN no
PostgreSQL, FTS5 no SQLite. Subir um serviço dedicado é decisão separada, e
mais cara.

Se usar junto com a vetorial, declare **como funde os dois resultados**.
Somar escores não funciona: o cosseno vive entre −1 e 1 e o BM25 não tem
teto.

#### Busca vetorial

Já existe, desde o Exercício 6. O que se pede aqui é o **recorte**: para
quais das dez perguntas ela é a ferramenta certa, e para quais foi usada por
inércia.

Declare também quais das quatro cegueiras da
[nota 02](../../aula-07-rag-e-documentos/notas-de-aula/02-o-que-o-embedding-nao-ve.md)
afetam o seu corpus, medidas com o par de controle. Um par por cegueira,
tirado do seu domínio, e o cosseno de cada um.

#### Grafo

A pergunta que identifica a necessidade: **existe no domínio uma pergunta
cuja resposta é um caminho?** Uma regra que excetua outra, um documento que
remete a um terceiro, uma aprovação que depende de uma cadeia de delegações.

Se existir, há duas saídas, e a escolha depende de uma única propriedade:

| A profundidade da cadeia é… | A solução |
|---|---|
| conhecida e rasa | um campo de referência e um `JOIN` |
| desconhecida de antemão | banco de grafo, com caminho de comprimento variável |

Se optar pelo grafo, declare o **modelo**: quais são os nós, quais são as
arestas e o que cada tipo de aresta significa. E declare o custo de
construção, que é onde o grafo se paga ou não — extrair as relações é o
trabalho caro, e mantê-las sincronizadas com o corpus é o trabalho
recorrente.

### 1.3 A escada, aplicada ao case

Feche a Parte 1 com a tabela de decisão, uma linha por forma:

| Forma | Usa? | Para quais perguntas | Custo declarado | Por quê |
|---|---|---|---|---|
| regex | | | | |
| consulta estruturada | | | | |
| busca textual | | | | |
| busca vetorial | | | | |
| grafo | | | | |

A regra que ordena a tabela é a cascata do roteador da aula 05: **começa-se
no degrau mais barato que resolve**, e só se sobe quando o de baixo tem
limite conhecido e demonstrado.

---

## Parte 2 — A memória do agente principal

### 2.1 Os dois níveis, separados

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

### 2.2 O curto prazo: o orçamento da janela

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

### 2.3 O longo prazo: as três memórias no case

Uma linha por tipo, com conteúdo do seu domínio — não do exemplo da aula:

| Tipo | O que guarda no case | Estrutura | Como é recuperada |
|---|---|---|---|
| **episódica** | | índice por similaridade | |
| **semântica** | | chave-valor | |
| **procedural** | | texto no *system prompt* | |

A estrutura de cada uma decorre do padrão de acesso, e não da preferência:
recuperar um fato semântico por similaridade é mais caro e menos exato que
resolvê-lo por chave.

### 2.4 Quem escreve, e o que não entra

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

## Parte 3 — Como o agente perde a memória

Esta é a parte que quase nenhum projeto especifica, e é a que separa um
sistema operável de um que acumula até quebrar.

São **três causas distintas**, e elas diferem no momento em que a decisão
ocorre e no que acontece com o dado:

| Causa | O que aconteceu | O dado é apagado? | Quando decide |
|---|---|---|---|
| **contradição** | o fato mudou | não | na leitura |
| **decaimento** | o fato envelheceu | sim, ou é rebaixado | em rotina |
| **remoção** | o titular solicitou | sim, obrigatoriamente | sob demanda |

### 3.1 Contradição — o fato que mudou

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

### 3.2 Decaimento — o fato que envelheceu sem ser contradito

Declare o **corte**, em dias, e a justificativa dele no domínio. O valor
adequado depende da taxa de mudança dos fatos armazenados: cinco meses pode
ser curto ou longo conforme o que se guarda.

E declare o efeito: o fato antigo é **removido** ou apenas **rebaixado** na
ordenação? As duas são defensáveis, e têm consequências diferentes para a
terceira causa.

### 3.3 Remoção — o titular solicitou

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

### 3.4 O preço: o sistema deixou de ser reprodutível

A mesma entrada, no mesmo modelo, com os mesmos parâmetros, produz saída
diferente amanhã — porque a memória mudou.

Isto não é defeito a corrigir. É consequência de projeto, e foi escolhida
deliberadamente ao construir a Parte 2. Registre-a, porque a aula 11 vai ter
de conviver com ela: um conjunto de avaliação que roda sobre um sistema com
memória mede duas coisas ao mesmo tempo, e separá-las é trabalho.

---

## O que entregar

Um documento, no repositório do trabalho, em
`docs/projeto-recuperacao-e-memoria.md`. Ele é o rascunho consolidado de
`docs/rag.md` e `docs/memoria.md`, que a Parte 2 cobra separados.

Cinco itens obrigatórios:

1. **A contagem do §1.1** — as dez perguntas classificadas, com o total por
   linha.
2. **A tabela de decisão do §1.3** — as cinco formas, com `usa`/`não usa` e a
   justificativa de cada uma.
3. **A tabela dos dois níveis do §2.1**, preenchida com o domínio do case.
4. **A tabela das três memórias do §2.3**, com a estrutura e o padrão de
   acesso de cada uma.
5. **A especificação das três causas de esquecimento**, com a lista de
   estruturas do §3.3 e o procedimento de verificação.

Extensão sugerida: três a cinco páginas. O que se avalia é a **justificativa
de cada decisão**, e não o número de tecnologias adotadas — um documento que
decide "não usa" em três das cinco formas, com razão declarada, vale mais que
um que adota todas sem medir.

## Critérios de avaliação

| Peso | Item |
|---|---|
| alto | a contagem do §1.1 é feita sobre perguntas reais, e não sobre o índice existente |
| alto | cada `não usa` da tabela do §1.3 tem justificativa, e não omissão |
| alto | a lista de estruturas do §3.3 inclui pelo menos uma fora das três memórias |
| médio | o orçamento da janela do §2.2 declara o que é descartado primeiro |
| médio | a estrutura de cada memória decorre do padrão de acesso declarado |
| médio | os dois fatos contraditórios do §3.1 vêm do domínio, e não do exemplo da aula |

## Dicas

- Comece pela Parte 3. Decidir o que o sistema esquece torna a Parte 2 mais
  fácil, porque a lista de exclusão restringe a de inclusão.
- A pergunta de caminho do §1.2 costuma existir e não ser percebida.
  Procure no domínio um "salvo o disposto em", um "exceto quando" ou um
  "conforme definido em" — cada um deles é uma aresta.
- Se o `não usa` do grafo for a resposta, ela precisa vir com a alternativa:
  qual pergunta de relação o case tem, e como ela é respondida sem grafo.
- Os exercícios complementares desta aula e da 07 produzem as medições que
  este documento cita. Fazê-los antes reduz o trabalho aqui a escrever
  decisões, em vez de tomá-las sem dado.
