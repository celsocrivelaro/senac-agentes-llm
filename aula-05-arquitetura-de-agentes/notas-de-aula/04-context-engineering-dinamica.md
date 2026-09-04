# IA Aplicada com LLMs — Aula 05: Arquitetura de agentes — Context engineering dinâmica

## Introdução

A nota 02 da Aula 03 fechou com uma promessa explícita:

> **Estático × dinâmico**: aqui, o que você monta na chamada. Na aula de agentes, o que se faz com a janela **ao longo da trajetória** — compaction, summarization, tool clearing, isolamento por sub-agentes.

Esta nota paga essa promessa. E ela responde à terceira das três perguntas que abriram a aula: *por que o laço não está pronto para rodar sozinho?* — **porque ele não sabe esquecer**.

O laço da Aula 03 tem uma propriedade que passa despercebida em execuções curtas e domina tudo em execuções longas:

> **A cada volta, o histórico inteiro é reenviado.** Nada nunca sai.

Numa trajetória de três passos, isso é irrelevante. Numa de trinta, é o fator dominante do custo, da latência e — o que menos se espera — da **qualidade**.

> **Pré-requisitos:** as notas [02](02-o-agente-e-o-estado.md) e [03](03-confiabilidade.md) desta aula (o estado e o orçamento). Da Aula 03, a [nota 02](../../aula-03-prompt-engineering/notas-de-aula/02-context-engineering.md) inteira — context engineering estática. Da Aula 02, a [nota 01 §4](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) (janela, custo quadrático da atenção, *lost in the middle*).

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Calcular** o crescimento do contexto ao longo de uma trajetória, e mostrar por que ele é superlinear no acumulado.
- **Distinguir** curadoria estática (o que você monta na chamada) de dinâmica (o que você faz ao longo da execução).
- **Aplicar** *tool clearing*, compaction reativa e periódica, e sumarização de trajetória.
- **Decidir o que preservar por obrigação** num resumo de trajetória, e o que pode ser descartado.
- **Usar sub-agentes como isolamento de contexto**, sabendo exatamente o que se perde.
- **Ordenar** as táticas da mais barata e segura para a mais destrutiva.

---

## Desenvolvimento teórico

### 1. A aritmética que ninguém faz

Tome um agente com um system prompt e declarações de ferramentas somando ~1.500 tokens, e uma trajetória em que cada passo acrescenta ao histórico a mensagem do modelo com a chamada (~80 tokens) e o resultado da ferramenta (~400 tokens) — cerca de **480 tokens por passo**.

O contexto **enviado no passo n** é `1.500 + 480 × (n − 1)`, e o total consumido pela execução é a soma disso desde o primeiro passo:

| Passo | Contexto enviado neste passo | Total acumulado até aqui |
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

> *Os números acima são a aritmética do acumulado, não uma medição.* Rode o `06-compaction.py` com o seu modelo e refaça a tabela com os seus números — a **forma** da curva não muda, os valores sim.

Duas leituras deste quadro:

**O custo acumulado cresce com o quadrado do número de passos**, porque cada passo reenvia tudo o que veio antes. Dobrar o número de passos não dobra a conta: quadruplica. Um agente de 30 passos não custa três vezes um de 10 — custa quase sete.

**A latência acompanha.** O tempo até o primeiro token depende do tamanho da entrada (Aula 02, nota 04, §5). Um agente que começa respondendo em 400 ms termina levando vários segundos por passo, e o usuário sente a execução ficando lenta sem saber por quê.

E existe uma terceira leitura, a menos intuitiva e a mais importante:

**A qualidade cai antes de a janela acabar.** No passo 25, o objetivo original está cercado por 12.000 tokens de observações — exatamente a região do meio onde o modelo recupera pior (*lost in the middle*, Aula 02, nota 01 §4.3). O agente não fica pior por falta de espaço. Fica pior por **excesso de passado**.

> **A formulação que vale guardar:** num agente, a janela não é um limite que você atinge. É um recurso que se degrada continuamente enquanto você o consome.

---

### 2. Estático × dinâmico

A Aula 03 tratou da **curadoria estática**: você decide, antes da chamada, o que entra — system prompt, exemplos, schema, documentos recuperados. É uma decisão tomada uma vez, por você, no momento de escrever o código.

A curadoria **dinâmica** é outra coisa: é o que o programa faz com a janela **enquanto** a execução acontece, sem que você saiba de antemão quantos passos haverá nem o que cada ferramenta vai devolver.

```
   ESTÁTICO (aula 03)                DINÂMICO (esta nota)
   ┌───────────────────┐             ┌───────────────────┐
   │ system            │             │ system            │  preservado
   │ exemplos          │             │ objetivo          │  preservado
   │ schema            │   ──────>   │ ····· resumo ·····│  comprimido
   │ documentos        │             │ passos 24-30      │  íntegros
   │ pergunta          │             │ objetivo (de novo)│  reancorado
   └───────────────────┘             └───────────────────┘
   você decide uma vez               o programa decide a cada volta
```

E isso só é implementável por causa da [nota 02](02-o-agente-e-o-estado.md): reescrever o histórico exige que ele seja um **campo do estado**, e não a própria estrutura em que o agente vive. Um agente cujo estado é `mensagens[]` não pode comprimir o histórico sem se apagar.

---

### 3. As três coisas que enchem a janela

Antes de comprimir, saiba o que ocupa espaço. Numa trajetória típica, em ordem de volume:

| Fonte | Volume | Ainda é útil depois? |
|---|---|---|
| **Resultados de ferramenta** | o maior, de longe | quase sempre **não** — depois de consumido, virou decisão |
| **Raciocínio intermediário** | médio | raramente |
| **Declarações de ferramentas** | fixo, mas em *toda* volta | sim, enquanto a ferramenta estiver ativa |
| **System prompt e objetivo** | pequeno | **sempre** |

A tabela já sugere a ordem de ataque, e ela é contraintuitiva para quem pensa em "resumir a conversa": o alvo principal **não é o diálogo**, são os **resultados de ferramenta**. Um `consultar_politica` que devolveu 900 tokens de texto legal foi lido, foi usado para decidir, e continua ocupando 900 tokens em todas as voltas seguintes.

---

### 4. Tool clearing — a tática mais barata

Descarte o **corpo** de resultados antigos, preservando o registro de que a chamada aconteceu:

```python
MANTER_INTEGROS = 3          # os N passos mais recentes ficam completos

def limpar_resultados(estado: Estado) -> None:
    """Substitui o conteúdo de resultados antigos por um marcador."""
    for passo in estado.passos[:-MANTER_INTEGROS]:
        if passo.resultado and not passo.limpo:
            passo.resumo = resumir_curto(passo)     # ~30 tokens
            passo.resultado = None
            passo.limpo = True
```

E o histórico enviado ao modelo passa a conter, no lugar do resultado inteiro:

```json
{"role": "tool", "tool_call_id": "...",
 "content": "[resultado removido do contexto] consultar_politica(categoria=refeicao) -> teto R$ 120,00 por refeição; exige nota fiscal"}
```

Por que esta é a primeira tática a aplicar:

- **é seletiva** — mexe só no que já foi consumido, não toca no raciocínio nem no objetivo;
- **é reversível** — o resultado completo continua no `Passo`, no estado; só saiu do que se envia. Se você precisar dele para auditoria ou para o trace, ele está lá;
- **preserva a estrutura da trajetória** — o modelo continua vendo que chamou aquela ferramenta com aqueles argumentos, o que é justamente o que impede que ele a chame de novo.

Este último ponto merece destaque, porque é onde a compressão ingênua estraga tudo: **apagar a chamada inteira faz o agente repeti-la**. Ele não lembra de ter consultado, consulta de novo, e você criou um laço com a sua própria otimização.

---

### 5. Compaction: reativa × periódica

Quando *tool clearing* não basta, comprima o histórico inteiro num resumo. A decisão de projeto é **quando disparar**.

**Reativa** — dispara ao cruzar um limiar:

```python
LIMIAR = 0.6      # fração da janela do modelo

def talvez_compactar(estado: Estado) -> None:
    if estimar_tokens(estado.historico) > LIMIAR * JANELA_DO_MODELO:
        compactar(estado)
```

**Periódica** — dispara a cada N passos:

```python
COMPACTAR_A_CADA = 10

def talvez_compactar(estado: Estado) -> None:
    if estado.n_passos and estado.n_passos % COMPACTAR_A_CADA == 0:
        compactar(estado)
```

| | Reativa | Periódica |
|---|---|---|
| **Dispara** | só quando precisa | sempre, no ritmo fixo |
| **Custo** | menor — nenhuma compressão desnecessária | maior — comprime mesmo quando cabia |
| **Previsibilidade** | baixa: você não sabe em que passo vai acontecer | alta: você sabe |
| **Risco** | um único resultado gigante pode estourar antes do próximo teste | comprimir cedo demais e perder detalhe ainda em uso |

Na prática, a combinação costuma vencer: **periódica como piso, reativa como teto** — comprime a cada 10 passos e também sempre que cruzar o limiar, o que cobre o caso do resultado de 30.000 tokens que chega sozinho.

Voltando à aritmética da §1, com compaction periódica a cada 10 passos reduzindo o histórico a ~1.200 tokens:

| Passos | Sem compaction | Com compaction (a cada 10) |
|---:|---:|---:|
| 10 | 36.600 | 36.600 |
| 20 | 121.200 | 85.200 |
| 30 | 253.800 | 133.800 |

Aos 30 passos, quase **metade** dos tokens — e o efeito cresce com o tamanho da trajetória, porque o que a compaction elimina é justamente o termo quadrático.

---

### 6. Sumarização de trajetória: o que não se pode perder

Comprimir é escolher o que jogar fora, e há uma lista curta do que **nunca** pode sair:

| Preservar sempre | Por quê |
|---|---|
| **O objetivo** | é a âncora; sem ele o agente deriva (nota 03, §7). No nosso desenho ele nem passa pela compaction — é campo do estado |
| **Decisões já tomadas** | "concluí que a despesa viola o artigo 4" precisa sobreviver, ou o agente reconclui |
| **Ações de escrita executadas** | "já registrei o parecer P-1001" — perder isto é registrar duas vezes |
| **Erros já cometidos** | "o id F-88 não existe" — perder isto é repetir a tentativa |
| **Fatos que a decisão final vai citar** | valores, prazos, artigos da política |

E o que pode ir embora sem dor: raciocínio intermediário, resultados já consumidos, tentativas que não levaram a nada e o texto integral de documentos dos quais só um trecho importou.

```python
PROMPT_COMPACTACAO = """Resuma a trajetória abaixo para que outro agente
continue a tarefa do ponto em que ela parou.

PRESERVE OBRIGATORIAMENTE, em tópicos:
- decisões já tomadas e a razão de cada uma;
- ações de ESCRITA já executadas, com os identificadores devolvidos;
- erros já cometidos e o que se aprendeu com cada um;
- fatos numéricos (valores, prazos, identificadores) citados até aqui.

DESCARTE: raciocínio intermediário, resultados já usados e tentativas
que não levaram a nada.

Não conclua a tarefa. Não invente informação que não esteja na trajetória.
"""
```

Repare que este prompt segue tudo o que a Aula 03 ensinou: contrato de saída explícito, exemplos negativos ("não conclua", "não invente") e formato definido. **O prompt de compaction é um prompt de produção**, e merece o mesmo cuidado — inclusive versionamento e teste de regressão (Aula 03, nota 03). Um resumo que perde um identificador de escrita causa um dano que nenhum log vai explicar depois.

E o preço, dito sem eufemismo:

> **Compaction é perda de informação com aparência de continuidade.** O agente segue funcionando, com uma memória que alguém editou. Quando ele errar por causa disso, o erro não vai parecer um erro de memória — vai parecer burrice do modelo.

---

### 7. Sub-agentes como isolamento de contexto

A última tática é diferente das outras três: em vez de comprimir o contexto depois, ela **impede que ele se forme**.

```
   PRINCIPAL                            SUB-AGENTE
   ┌──────────────────┐                 ┌──────────────────┐
   │ objetivo         │  ──delega──>    │ subtarefa        │
   │ ...              │                 │ 14 passos        │
   │ [resumo 1,2k]    │  <──devolve──   │ 60k tokens       │
   │ ...              │                 │ (descartados)    │
   └──────────────────┘                 └──────────────────┘
     paga 1,2k                            gastou 60k, e
                                          some ao terminar
```

O sub-agente tem a própria janela, gasta o que precisar e devolve apenas um resumo. O contexto do principal cresce por um resumo, não por catorze pares de chamada e resultado.

É a tática certa quando a subtarefa é **verificável e delimitada**: "encontre em qual artigo da política esta categoria de despesa é tratada e devolva o texto do artigo". Entra pergunta, sai resposta curta.

O preço aqui é o mais alto das quatro táticas:

> **O que o sub-agente não resumiu, o principal nunca saberá.** Não é compressão: é uma parede. O principal não tem como perceber que faltou algo, porque nunca viu o que existia do outro lado.

Por isso: sub-agente para subtarefa **fechada**, com critério claro de sucesso. Sub-agente para tarefa exploratória — "investigue isso aí e me conte" — é uma forma cara de perder informação.

E aqui esta aula encosta em multi-agente e para. Coordenação entre agentes, protocolo de comunicação, agentes que negociam ou revisam uns aos outros: é aula própria. O que fica desta nota é o uso **arquitetural** do sub-agente, como técnica de contexto.

---

### 8. A ordem de aplicação

As quatro táticas, da mais segura para a mais destrutiva. Aplique nesta ordem e pare quando couber:

| # | Tática | Destrói? | Quando |
|---|---|---|---|
| 1 | **Tool clearing** | não — reversível, o dado fica no estado | sempre, a partir de ~5 passos |
| 2 | **Compaction periódica + reativa** | sim, de forma controlada | trajetórias longas (>10 passos) |
| 3 | **Sumarização agressiva** | sim | quando a trajetória é o custo dominante |
| 4 | **Sub-agentes** | sim, irreversivelmente | subtarefa fechada e verificável |

E a tática número zero, que vale mais que as quatro: **encurtar o retorno das ferramentas**. Uma ferramenta que devolve 900 tokens quando 60 bastariam é um problema de projeto de ferramenta (Aula 03, nota 04, §7), e resolvê-lo na origem é sempre melhor do que comprimir depois. Antes de escrever a primeira linha de compaction, olhe os retornos das suas ferramentas.

---

### 9. Fechando a lista

A Aula 03 terminou com seis pendências. Depois desta aula:

| Pendência (Aula 03, nota 04 §9) | Situação |
|---|---|
| ~~outros padrões de arquitetura~~ | **fechado** — [nota 01](01-padroes-de-arquitetura.md) |
| ~~estado~~ (dentro de uma execução) | **fechado** — [nota 02](02-o-agente-e-o-estado.md) |
| ~~múltiplas ferramentas e a escolha entre elas~~ | **fechado** — nota 02, §6 |
| ~~confiabilidade: retry, laço, orçamento, checkpoint~~ | **fechado** — [nota 03](03-confiabilidade.md) |
| ~~context engineering dinâmica~~ | **fechado** — esta nota |
| **memória entre execuções** | aula de memória |
| **MCP** | aula de MCP |
| **multi-agente** (coordenação) | aula de sistemas multi-agente |

E duas pendências novas, criadas por esta aula: o **trace** que você passou a gravar (nota 03, §10) é a matéria-prima das aulas de **observabilidade** e **evals**. Como sempre, fechar um assunto abre outro — a diferença é que agora as perguntas são melhores.

---

## Exemplos

### Exemplo 1 — O mesmo agente, com e sem tool clearing

Trajetória de 12 passos, ferramenta que devolve ~400 tokens por chamada.

```
SEM tool clearing
  passo  1   contexto enviado:  1.500
  passo  6   contexto enviado:  3.900
  passo 12   contexto enviado:  6.780
  total consumido: ~49.700 tokens

COM tool clearing (mantendo 3 passos íntegros)
  passo  1   contexto enviado:  1.500
  passo  6   contexto enviado:  2.610   (3 resultados inteiros + 2 marcadores)
  passo 12   contexto enviado:  2.850   (idem — estabiliza)
  total consumido: ~28.900 tokens
```

Repare no que acontece a partir do passo 6: o contexto **para de crescer**. Ele estabiliza em "os últimos três resultados inteiros + um marcador de ~30 tokens por passo antigo". A curva quadrática vira quase reta, e isso sem nenhuma chamada extra de LLM — *tool clearing* é código, não modelo.

### Exemplo 2 — Uma compaction que quebrou o agente

```python
# Resumo produzido por um prompt vago: "resuma a conversa acima".
resumo = """O agente analisou a despesa D-4471 do funcionário F-088,
consultou a política de refeições e concluiu que a despesa está dentro
do teto. O parecer foi elaborado."""
```

O que sumiu: **o identificador do parecer registrado**. "O parecer foi elaborado" não diz que `registrar_parecer` já foi executado e devolveu `P-1001`.

O que aconteceu no passo seguinte: o agente, sem ver nenhum registro de escrita concluída, chamou `registrar_parecer` de novo. Dois pareceres para a mesma despesa.

O que salvou: a **chave de idempotência** da [nota 03, §8](03-confiabilidade.md) — a segunda chamada devolveu `{"id": "P-1001", "ja_existia": true}` e o agente seguiu adiante.

A lição tem duas metades. A primeira é que o prompt de compaction precisa exigir a preservação de ações de escrita. A segunda, mais geral: **as salvaguardas se cobrem umas às outras**. Nenhuma delas é suficiente sozinha, e é por isso que a arquitetura tem camadas em vez de um remédio.

---

## Exercícios resolvidos

### 1. Qual tática para cada caso?

| Situação | Tática | Por quê |
|---|---|---|
| Agente de 8 passos, cada ferramenta devolve 200 tokens | **nenhuma** | ~3.100 tokens no pior passo. Comprimir aqui é otimização prematura que só adiciona risco |
| Agente de 40 passos lendo trechos de documentos longos | **tool clearing + compaction periódica** | o volume está nos resultados; comece pela tática reversível |
| Uma única ferramenta devolve 25.000 tokens | **encurtar a ferramenta** (tática zero) | é defeito de projeto da ferramenta; nenhuma compressão conserta a origem |
| Subtarefa de pesquisa com número imprevisível de consultas | **sub-agente** | fechada e verificável; devolve o achado, descarta o caminho |
| Trajetória longa em que o agente começa a ignorar o pedido | **reancoragem** (nota 03, §7) antes de comprimir | o problema é posição, não volume |

O último caso é o mais instrutivo: nem todo problema de contexto se resolve com compressão. Se o objetivo está soterrado, o remédio é **movê-lo**, não encolher o resto.

### 2. Consertar este prompt de compaction

```python
PROMPT = "Resuma a conversa acima em até 200 palavras."
```

Três defeitos:

1. **Não diz o que preservar.** O modelo vai resumir *narrativamente* — o que se leu, o que se conversou — e é justamente o que não importa. Precisa da lista obrigatória: decisões, escritas executadas com identificadores, erros cometidos, fatos numéricos.
2. **Não diz para quem o resumo serve.** "Resuma para que outro agente continue a tarefa do ponto em que parou" muda completamente a saída: o resumo passa a ser um *handoff*, não uma sinopse.
3. **Não tem exemplos negativos** (Aula 03, nota 01). Sem "não conclua a tarefa" e "não invente informação", o modelo às vezes entrega a conclusão dentro do resumo — e o agente continua a partir de uma conclusão que ninguém verificou.

O limite de 200 palavras, por sua vez, é a única coisa que se aproveita — mas em tokens, e calibrado pelo que sobra do orçamento, não por um número redondo.

---

## Síntese

- Num agente, o histórico inteiro é reenviado a cada volta: o **custo acumulado cresce com o quadrado** do número de passos.
- A qualidade cai **antes** de a janela acabar — o objetivo vai para o meio do contexto, que é onde o modelo lê pior.
- Curadoria estática é o que você monta na chamada; **dinâmica** é o que o programa faz com a janela durante a execução — e só é possível porque o histórico é campo do estado.
- O maior volume está nos **resultados de ferramenta**, não no diálogo. Ataque ali primeiro.
- **Tool clearing** é a primeira tática: barata, reversível e preserva a estrutura da trajetória. Apagar a chamada inteira faz o agente repeti-la.
- **Compaction** periódica dá previsibilidade, reativa dá economia; a combinação cobre os dois riscos.
- Um resumo de trajetória preserva obrigatoriamente: objetivo, decisões, **escritas executadas com identificadores**, erros cometidos e fatos numéricos.
- O prompt de compaction é um prompt de produção — versionado e testado como qualquer outro.
- **Sub-agente** isola contexto ao preço mais alto: o que ele não resumiu, ninguém recupera. Use em subtarefa fechada e verificável.
- A tática zero vale mais que as quatro: **encurte o retorno das suas ferramentas**.
- As salvaguardas se cobrem umas às outras — a idempotência salvou o erro da compaction.

---

## Fontes e leituras

- Aula 03, [nota 02](../../aula-03-prompt-engineering/notas-de-aula/02-context-engineering.md) — context engineering estática, cuja promessa esta nota cumpre; [nota 03](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) — versionamento, que vale para o prompt de compaction; [nota 04 §7](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — projeto de ferramenta, a origem da tática zero.
- Aula 02, [nota 01 §4](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — janela, custo quadrático da atenção e *lost in the middle*; [nota 04 §5](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md) — latência e o efeito do tamanho da entrada.
- **Effective context engineering for AI agents** — Anthropic (2025). Compaction, *tool clearing* e sub-agentes como táticas de contexto ao longo da trajetória.
- **Building Effective Agents** — Anthropic (2024). A seção sobre agentes autônomos e o custo da trajetória.
- **Lost in the Middle: How Language Models Use Long Contexts** — Liu et al. (2023). O viés em U que fundamenta a reancoragem.
