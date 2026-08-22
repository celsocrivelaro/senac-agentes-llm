# Exercício 2 - Do e-mail do cliente ao chamado aberto

## Contexto

Na transportadora do exercício anterior, as mensagens dos clientes chegam cruas — WhatsApp, e-mail, formulário do site — e alguém do atendimento lê cada uma, decide de que se trata, procura o número do pedido no meio do texto e **escreve o chamado à mão** no sistema. São centenas por dia, e o chamado sai diferente dependendo de quem escreveu.

Você vai automatizar isso. O programa lê a mensagem, **processa** (classifica e extrai) e **gera o texto do chamado**, pronto para abrir.

Repare que são **duas tarefas de natureza oposta** dentro do mesmo programa:

| Etapa | O que faz | Variação na saída é... |
| --- | --- | --- |
| **1. Ler e processar** | classificar e extrair dados do texto | **defeito** — o mesmo e-mail tem que dar sempre o mesmo resultado |
| **2. Redigir o chamado** | escrever um texto para um humano ler | **aceitável** — fluência importa |

E é daí que sai o trabalho de verdade deste exercício: **elas não podem usar os mesmos parâmetros.** Decidir quais usar em cada uma, e saber justificar, é o que se avalia aqui.

## Objetivo

Criar um programa `08-abertura-de-chamados.py` que, para cada mensagem de cliente:

1. **Extrai** categoria, número do pedido e urgência, em formato estruturado
2. **Gera** o texto do chamado a partir desses dados
3. Imprime o chamado e o custo daquela mensagem

Os parâmetros das duas etapas ficam **no topo do script**, cada um com um comentário dizendo por que aquele valor — com base na teoria das notas de aula. Não há relatório à parte.

```
 mensagem crua
       │
       ▼
 ┌─────────────────┐        ┌──────────────────────┐
 │ ETAPA 1         │ dados  │ ETAPA 2              │  texto do
 │ ler e processar │───────>│ redigir o chamado    │──> chamado
 └─────────────────┘        └──────────────────────┘
   parâmetros A                parâmetros B
        (você escolhe, e justifica)
```

## As mensagens

Oito mensagens, processadas **uma vez cada**. O gabarito é para você conferir a etapa 1.

```python
# (mensagem, categoria correta, pedido correto)
MENSAGENS = [
    ("Meu pedido 48219 era pra chegar terça e até hoje nada. Já são 5 dias!",
     "entrega_atrasada", "48219"),

    ("bom dia, o entregador deixou na rua de tras, numero 45. o meu é 145",
     "endereco_errado", None),

    ("A caixa do pedido 77310 chegou toda amassada e a tampa do liquidificador trincou",
     "produto_avariado", "77310"),

    ("vocês entregam no sábado?",
     "duvida", None),

    ("Só pra dizer que o rapaz da entrega foi super educado, obrigada!",
     "elogio", None),

    ("Pedido 90021 cancelado e recomprado como 90455, o 90455 não chegou",
     "entrega_atrasada", "90455"),

    ("PEDIDO 12 MIL 340 ENTREGUE NO CEP ERRADO, EU MORO NO 04567890 E FOI PRO 04567089",
     "endereco_errado", "12340"),

    ("Recebi o pedido n 55.102 com a tela rachada. Quero trocar.",
     "produto_avariado", "55102"),
]

CATEGORIAS = ["entrega_atrasada", "endereco_errado", "produto_avariado", "duvida", "elogio"]
```

> **Leia as oito antes de programar.** Metade é armadilha: números que **não** são pedido (número de casa, CEP), número **por extenso** ("12 mil 340"), número **com ponto** ("55.102"), **dois** pedidos na mesma mensagem, e uma que elogia o produto e reclama do prazo.
>
> Regras de desempate, que você precisa colocar no prompt: com mais de um pedido, vale aquele de que o cliente reclama; com mais de um problema, vale a **reclamação**, não o elogio.

## Requisitos

### 1. Etapa 1 — ler e processar

A saída desta etapa é consumida pelo **seu código**, não por uma pessoa. Ela precisa ter contrato:

```json
{ "categoria": "entrega_atrasada", "pedido": "48219", "urgencia": "alta" }
```

- `categoria`: um dos 5 valores de `CATEGORIAS`
- `pedido`: string só com dígitos, ou `null`
- `urgencia`: `"baixa"`, `"media"` ou `"alta"` — decidida pelo modelo a partir do tom e do problema

Escolha os parâmetros desta etapa e **anote por quê** (requisito 3).

### 2. Etapa 2 — gerar o chamado

A partir dos dados da etapa 1 **e** da mensagem original, o modelo escreve o chamado. Formato:

```
=== CHAMADO #2026-0842 ===
Categoria:  entrega_atrasada
Urgência:   alta
Pedido:     48219

Título: Atraso de 5 dias na entrega do pedido 48219

Descrição:
O cliente relata que o pedido 48219 tinha previsão de entrega para terça-feira
e, cinco dias depois do prazo, ainda não foi recebido. Não há informação sobre
tentativa de entrega. Cliente demonstra insatisfação com a demora.

Ação sugerida: rastrear o pedido e retornar ao cliente com nova previsão.
```

Regras:

- o **número do chamado** é gerado pelo seu código (sequencial ou data + contador), **não** pelo modelo — ele inventaria;
- os campos `Categoria`, `Urgência` e `Pedido` vêm da etapa 1, escritos pelo **seu código**;
- o **Título**, a **Descrição** e a **Ação sugerida** são gerados pelo modelo;
- a descrição é factual: **não pode inventar** informação que não está na mensagem do cliente (nome, endereço, datas que ninguém citou);
- quando não houver número de pedido, o campo tem que dizer isso, e não sumir nem vir `None`.

#### A ação sugerida é obrigatória

**Todos os 8 chamados têm que sair com uma `Ação sugerida`** — sem exceção, e sem campo vazio.

Isso é fácil nas reclamações e **desconfortável** nas outras duas categorias, que é exatamente o ponto:

| Categoria | A pergunta que o modelo tem que responder |
| --- | --- |
| `entrega_atrasada` | rastrear? acionar a transportadora? reenviar? |
| `endereco_errado` | recoletar? redespachar? confirmar o endereço com o cliente? |
| `produto_avariado` | trocar? estornar? pedir foto? |
| **`duvida`** | quem responde, e o que precisa ser respondido? |
| **`elogio`** | um elogio também gera trabalho — repassar a quem? agradecer? |

Duas exigências sobre o conteúdo:

- **específica da mensagem, não genérica.** "Verificar com a equipe responsável" serve para qualquer chamado e portanto não serve para nenhum. A ação tem que citar o que aquele cliente precisa;
- **acionável por uma pessoa**: um verbo no início e um destinatário claro. Quem ler o chamado tem que saber o que fazer a seguir sem reler a mensagem do cliente.

> Antes de acusar o modelo de gerar ações vagas, confira se o **seu prompt** disse que a ação é obrigatória, específica e acionável — e se deu um exemplo do que você considera boa. Isso é conteúdo da aula 03, mas o efeito você já vê aqui.

Escolha os parâmetros desta etapa também — **e eles não devem ser os mesmos da etapa 1.**

### 3. Os parâmetros ficam no código — e explicados lá

Não há relatório à parte. **As duas configurações vivem no topo do script**, cada
uma num bloco, com um comentário curto dizendo **por que** aquele valor. É assim
que se documenta decisão de configuração em código de verdade: junto do código.

```python
# --- ETAPA 1: ler e processar -------------------------------------------
# temperature 0     -> extração: variação aqui é DEFEITO (nota 02, §9)
# max_tokens 120    -> pior caso do schema; medi 48 na mediana, dobrei
# response_format   -> JSON Schema com enum na categoria: o modelo não
#                      consegue inventar rótulo fora dos 5 (nota 02, §7.3)
# penalidades 0     -> penalizariam as chaves repetidas do JSON (nota 03, §5)
EXTRACAO = {
    "temperature": 0,
    "max_tokens": 120,
    ...
}

# --- ETAPA 2: redigir o chamado -----------------------------------------
# temperature ...   -> (você decide, e diz por quê)
# ...
REDACAO = {
    ...
}
```

Regras:

- **cite a seção da nota** que sustenta a escolha, como nos exemplos acima;
- **"0" e "não usei" também precisam de justificativa** — em várias linhas essa
  é a resposta certa, e saber *por que* é o ponto;
- para o `max_tokens`, diga **como** chegou ao número. Chute não conta: meça o
  `usage.completion_tokens` de algumas execuções e dimensione o pior caso;
- se as duas etapas saírem com a mesma configuração, algo está errado — releia
  a tabela de receitas da nota 02.

### 4. Robustez

Cerca de 16 chamadas por execução. O programa não pode quebrar no meio:

- checar `finish_reason` — `"length"` **não** é uma resposta válida, é truncamento;
- tratar `429` com **backoff exponencial** (1s, 2s, 4s…) e continuar de onde parou;
- se uma mensagem falhar, registrar qual e por quê, e seguir para a próxima — **sem engolir a exceção em silêncio**;
- ao final, imprimir quantas mensagens viraram chamado e quantas falharam.

### 5. Custo

Some `usage` das **duas** etapas e faça o programa **imprimir**, ao final:

- o custo médio **por chamado** (as duas chamadas juntas);
- o custo de cada etapa separado — e, num comentário no código, **qual delas
  custa mais e por quê** (a resposta tem a ver com qual tarifa domina em cada
  uma: nota 04, §1);
- a projeção para **800 chamados/dia**.

Os preços ficam num dicionário no topo do script, com a data da consulta a
<https://mistral.ai/pricing> num comentário.

## Conferindo o resultado

**Etapa 1 — contra o gabarito.** Informe quantas das 8 acertaram categoria e pedido.

Se alguma errar, **leia a mensagem e o seu prompt antes de culpar o modelo**: as regras de desempate estão escritas lá? O `enum` está no schema? Erro de categoria e erro de extração de número têm causas diferentes.

**Etapa 2 — contra o formato.** Aqui não há gabarito, mas há verificação automática. Antes de imprimir, confira que o chamado tem **todos** os campos preenchidos:

```python
CAMPOS = ["Categoria:", "Urgência:", "Pedido:", "Título:", "Descrição:", "Ação sugerida:"]

faltando = [c for c in CAMPOS if c not in chamado]
if faltando:
    raise ValueError(f"chamado incompleto, faltam: {faltando}")
```

E leia as 8 ações sugeridas de uma vez, em sequência. **Se duas forem intercambiáveis entre chamados diferentes, elas são genéricas demais** — o problema está no seu prompt, não no modelo.

## Desafios opcionais

- **O guardrail da descrição.** Escreva uma verificação que detecte se o modelo inventou algo — por exemplo, conferir se todo número citado na descrição aparece na mensagem original. É o problema de "formato válido, conteúdo errado" (nota 02, §7.4).
- **Duas vozes.** Gere a descrição em dois registros — um técnico, para o time de logística, e um cordial, para responder ao cliente — mudando **só** os parâmetros e o prompt da etapa 2, sem tocar na etapa 1.
- **Uma chamada em vez de duas.** Tente fazer tudo numa chamada só, pedindo um JSON que contenha também o texto do chamado. Compare custo, latência e qualidade com a versão de duas etapas — e diga qual você levaria para produção.

## Entrega

- `08-abertura-de-chamados.py`
- **Os 8 chamados gerados**, completos — cada um com a sua `Ação sugerida` preenchida
- Um print de erro tratado: um `429` com backoff, ou um `finish_reason="length"` detectado

## Dicas

- **Comece por uma mensagem só**, com as duas etapas, e só depois faça o laço.
- **A etapa 1 é onde o formato importa; a etapa 2 é onde o texto importa.** Se você se pegar usando a mesma configuração nas duas, releia a tabela de receitas da nota 02.
- O modelo vai querer escrever mais do que você pediu no chamado — despedida, oferta de ajuda, "espero ter ajudado". Isso tem solução direta na nota 03.
- Se a descrição sair com informação que o cliente nunca deu, o problema **não** se resolve baixando a temperatura. Releia a nota 02, §7.4.
- Guarde este script: na aula de agentes, ele vira uma **ferramenta** que um agente chama — e os parâmetros que você escolheu (e comentou) aqui continuam valendo.
