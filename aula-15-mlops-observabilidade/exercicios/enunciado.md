# Exercício 15 — O plano de observabilidade de um agente de trocas

## Contexto

Um **plano de observabilidade** é o documento que declara, antes de
instrumentar, o que se precisa saber sobre um sistema enquanto ele roda, que
falhas ele precisa tornar visíveis e que objetivos ele precisa cumprir — e,
daí, que sinais ele emite e quem age sobre eles (nota sobre o plano de
observabilidade).

Este exercício pede o plano de um agente pequeno, descrito abaixo. Não há
código a escrever: a entrega é o plano.

> **Questão a ser respondida ao final:** como saber, sem ler o código, se o
> agente de trocas está funcionando — e se está funcionando bem?

## O case: o agente de trocas de uma loja virtual

Uma loja virtual atende pedidos de troca por um agente, no chat do site.

- **O que ele faz.** O cliente informa o número do pedido e o motivo da
  troca. O agente consulta o pedido, consulta a política de trocas e decide:
  **aprova** a troca, **recusa** explicando o motivo, ou **encaminha** a um
  atendente humano quando não tem certeza.
- **As ferramentas.**

  | Ferramenta | O que faz |
  |---|---|
  | `consultar_pedido(numero)` | devolve data da compra, produto, categoria e valor |
  | `consultar_politica(categoria)` | devolve o prazo de troca da categoria (em geral, 30 dias) e as condições |
  | `registrar_troca(numero, motivo)` | **registra a troca no sistema da loja** — é a única ferramenta que muda alguma coisa |

- **O modelo.** Chamado por API, cobrado por token.
- **O orçamento.** No máximo 6 passos por conversa; ao esgotar, o agente
  encaminha ao atendente.
- **O volume.** Cerca de 500 conversas por dia.
- **A promessa.** A loja adotou o agente esperando que ele **resolva 60% das
  trocas sem atendente**, com menos de 1% de trocas aprovadas fora da
  política.

## Requisitos

Preencha o modelo [`modelo-plano-de-observabilidade.md`](modelo-plano-de-observabilidade.md),
nesta pasta. Cada seção abaixo traz um exemplo do auditor de despesas, o
agente da aula, para mostrar o formato.

### 1. As perguntas

As perguntas dizem **o que se precisa saber sobre o sistema enquanto ele
roda, e para quê**. Só se mede o que responde a uma pergunta; e só fica a
pergunta cuja resposta muda alguma decisão.

Escreva **quatro perguntas**, uma para cada interessado — o negócio, a
operação, quem paga e quem responde pelo sistema —, cada uma com a decisão
que a resposta muda.

> *Exemplo (auditor):* "quantas análises saem sem veredito?" — interessado:
> a operação — decisão: investigar a versão no ar.

### 2. O *trace* de uma conversa

Uma conversa com o agente é **uma execução**, e cada execução é um *trace*.
Desenhe a árvore de *spans* de uma conversa típica, nas convenções GenAI:
`invoke_agent` na raiz, um `chat` por chamada ao modelo, um `execute_tool`
por ferramenta.

Depois, liste **de dois a três atributos** que a aplicação precisa gravar na
raiz — informações que nem a convenção nem a plataforma fornecem, porque
dependem do domínio — e diga para que serve cada um.

> *Exemplo (auditor):* `agente.termino` (respondeu, orçamento esgotado,
> erro) — separa as execuções que terminaram bem das que pararam no meio.

### 3. Os modos de falha

Liste **quatro formas** de o agente falhar. Para cada uma: o componente, a
falha, se ela é **ruidosa** (gera erro) ou **silenciosa** (o agente responde,
mas responde pior), e o **sinal** que a torna visível. Ao menos **duas
silenciosas**.

> *Exemplo (auditor):* laço — esgota o orçamento de passos — silenciosa: a
> resposta sai, sem veredito — sinal: `agente.termino = orcamento_esgotado`.

### 4. Os objetivos

Defina **dois SLOs**, cada um com o indicador, o objetivo e a janela:

- um **operacional** (tempo de resposta, ou conversas que terminam bem);
- um ligado à **promessa** da loja.

> *Exemplo (auditor):* p95 da execução ≤ 30 s, em 28 dias.

### 5. Os alertas

Defina **dois alertas**: um de **custo** e um de **degradação silenciosa**.
Para cada um: a condição, o limiar (tirado de um SLO da seção 4), quem é
avisado e o que essa pessoa faz.

> *Exemplo (auditor):* execuções terminadas por orçamento acima de 10% na
> última hora — avisa o responsável pelo agente — que compara a versão no ar
> com a anterior.

## Entrega

O plano preenchido, em `aula15-plano-de-observabilidade.md`.

> **Onde isto reaparece:** a **Parte 3 do trabalho** pede a observabilidade
> do sistema do grupo. O método é o mesmo deste exercício, aplicado ao case
> do grupo.

## Dicas

- Para achar falhas silenciosas, percorra cada ferramenta e o modelo e
  pergunte: *como isto pode devolver um resultado pior sem dar erro?*
- A ferramenta `registrar_troca` é a única que muda o mundo: pense no que
  acontece se ela for chamada quando não devia.
