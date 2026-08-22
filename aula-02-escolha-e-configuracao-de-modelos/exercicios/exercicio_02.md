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

E um `decisoes.md` justificando **cada parâmetro escolhido**, com base na teoria das notas de aula.

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

Escolha os parâmetros desta etapa também — **e eles não devem ser os mesmos da etapa 1.**

### 3. `decisoes.md` — a parte que vale a nota

Para **cada etapa**, preencha e justifique. Uma linha por parâmetro:

| Parâmetro | Etapa 1 | Por quê | Etapa 2 | Por quê |
| --- | --- | --- | --- | --- |
| `temperature` | | | | |
| `max_tokens` | | | | |
| `response_format` | | | | |
| `stop` | | | | |
| `frequency_penalty` | | | | |
| `presence_penalty` | | | | |

Regras para as justificativas:

- **cite a seção da nota** que sustenta a escolha (ex.: "nota 03, §2.2 — `max_tokens` é guilhotina, não pedido de brevidade");
- **"deixei em 0" ou "não usei" também precisa de justificativa** — em várias linhas, essa é a resposta certa, e saber por que é o ponto;
- para o `max_tokens`, diga **como** chegou ao número. Chute não conta: meça o `usage.completion_tokens` de algumas execuções e dimensione o pior caso.

### 4. Robustez

Cerca de 16 chamadas por execução. O programa não pode quebrar no meio:

- checar `finish_reason` — `"length"` **não** é uma resposta válida, é truncamento;
- tratar `429` com **backoff exponencial** (1s, 2s, 4s…) e continuar de onde parou;
- se uma mensagem falhar, registrar qual e por quê, e seguir para a próxima — **sem engolir a exceção em silêncio**;
- ao final, imprimir quantas mensagens viraram chamado e quantas falharam.

### 5. Custo

Some `usage` das **duas** etapas e informe:

- o custo médio **por chamado** (as duas chamadas juntas);
- qual das duas etapas custa mais, **e por quê** — a resposta tem a ver com qual tarifa domina em cada uma (nota 04, §1);
- a projeção para **800 chamados/dia**.

Preços em <https://mistral.ai/pricing>, com a data da consulta anotada.

## Conferindo a etapa 1

Compare a saída com o gabarito e informe quantas das 8 acertaram categoria e pedido.

Se alguma errar, **leia a mensagem e o seu prompt antes de culpar o modelo**: as regras de desempate estão escritas lá? O `enum` está no schema? Erro de categoria e erro de extração de número têm causas diferentes.

## Desafios opcionais

- **O guardrail da descrição.** Escreva uma verificação que detecte se o modelo inventou algo — por exemplo, conferir se todo número citado na descrição aparece na mensagem original. É o problema de "formato válido, conteúdo errado" (nota 02, §7.4).
- **Duas vozes.** Gere a descrição em dois registros — um técnico, para o time de logística, e um cordial, para responder ao cliente — mudando **só** os parâmetros e o prompt da etapa 2, sem tocar na etapa 1.
- **Uma chamada em vez de duas.** Tente fazer tudo numa chamada só, pedindo um JSON que contenha também o texto do chamado. Compare custo, latência e qualidade com a versão de duas etapas — e diga qual você levaria para produção.

## Entrega

- `08-abertura-de-chamados.py`
- `decisoes.md` com a tabela de parâmetros justificada e a conta de custo
- Os 8 chamados gerados (cópia do terminal ou arquivo)
- Um print de erro tratado: um `429` com backoff, ou um `finish_reason="length"` detectado

## Dicas

- **Comece por uma mensagem só**, com as duas etapas, e só depois faça o laço.
- **A etapa 1 é onde o formato importa; a etapa 2 é onde o texto importa.** Se você se pegar usando a mesma configuração nas duas, releia a tabela de receitas da nota 02.
- O modelo vai querer escrever mais do que você pediu no chamado — despedida, oferta de ajuda, "espero ter ajudado". Isso tem solução direta na nota 03.
- Se a descrição sair com informação que o cliente nunca deu, o problema **não** se resolve baixando a temperatura. Releia a nota 02, §7.4.
- Guarde o `decisoes.md`: na aula de agentes, este programa vira uma **ferramenta** que um agente chama — e os parâmetros que você escolheu aqui continuam valendo.
