# IA Aplicada com LLMs — Aula 05: Arquitetura de agentes — Context engineering dinâmica

## Introdução

A nota 02 da Aula 03 fechou com uma promessa: *aqui, o que você monta na chamada; na aula de agentes, o que se faz com a janela **ao longo da trajetória***. Esta nota paga a promessa, e responde à terceira das perguntas que abriram a aula — *por que o laço não está pronto para rodar sozinho?* Porque **ele não sabe esquecer**.

> **A cada volta, o histórico inteiro é reenviado. Nada nunca sai.**

Numa trajetória de três passos isso é irrelevante. Numa de trinta, domina o custo, a latência e — o que menos se espera — a **qualidade**.

> **Pré-requisitos:** notas [02](02-o-agente-e-o-estado.md) e [03](03-confiabilidade.md) desta aula · Aula 03, [nota 02](../../aula-03-prompt-engineering/notas-de-aula/02-context-engineering.md) inteira · Aula 02, [nota 01 §4](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md).
>
> **Código:** [`06-compaction.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/06-compaction.py) — trajetória longa com e sem compaction, com os tokens por passo na tela.

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Calcular** o crescimento do contexto ao longo de uma trajetória, e mostrar por que é superlinear no acumulado.
- **Distinguir** curadoria estática (o que você monta na chamada) de dinâmica (o que o programa faz durante a execução).
- **Aplicar** *tool clearing*, compaction reativa e periódica, e sumarização de trajetória.
- **Decidir o que preservar por obrigação** num resumo, e o que pode ser descartado.
- **Usar sub-agentes como isolamento de contexto**, sabendo exatamente o que se perde.
- **Ordenar** as táticas da mais barata e segura para a mais destrutiva.

---

## Desenvolvimento teórico

### 1. A aritmética que ninguém faz

System prompt + declarações ≈ 1.500 tokens. Cada passo acrescenta a chamada do modelo (~80) e o resultado da ferramenta (~400) — **~480 tokens por passo**. O contexto enviado no passo *n* é `1.500 + 480 × (n − 1)`; o total consumido é a soma disso desde o primeiro.

| Passo | Contexto neste passo | Total acumulado |
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

> *A aritmética do acumulado, não uma medição.* Rode o `06-compaction.py` com o seu modelo e refaça a tabela — a **forma** da curva não muda, os valores sim.

Três leituras:

| Efeito | O que acontece |
|---|---|
| **Custo quadrático** | cada passo reenvia tudo o que veio antes. Dobrar os passos não dobra a conta: quadruplica. Um agente de 30 passos custa quase **sete** vezes um de 10 |
| **Latência acompanha** | o tempo até o primeiro token depende do tamanho da entrada (Aula 02 nota 04 §5). O agente começa em 400 ms e termina em vários segundos por passo |
| **A qualidade cai antes de a janela acabar** | no passo 25 o objetivo está cercado por 12.000 tokens de observações — a região do meio, onde o modelo recupera pior (*lost in the middle*) |

O terceiro é o menos intuitivo e o mais importante: o agente não fica pior por falta de espaço. Fica pior por **excesso de passado**.

> **A formulação que vale guardar:** num agente, a janela não é um limite que você atinge. É um recurso que se degrada continuamente enquanto você o consome.

---

### 2. Estático × dinâmico

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

A curadoria estática é uma decisão tomada uma vez, por você, ao escrever o código. A dinâmica é o que o programa faz com a janela **enquanto** a execução acontece, sem saber de antemão quantos passos haverá nem o que cada ferramenta vai devolver.

Só é implementável por causa da [nota 02](02-o-agente-e-o-estado.md): reescrever o histórico exige que ele seja **campo do estado**, e não a própria estrutura em que o agente vive. Um agente cujo estado é `mensagens[]` não pode comprimir o histórico sem se apagar.

---

### 3. As três coisas que enchem a janela

| Fonte | Volume | Ainda é útil depois? |
|---|---|---|
| **Resultados de ferramenta** | o maior, de longe | quase sempre **não** — depois de consumido, virou decisão |
| **Raciocínio intermediário** | médio | raramente |
| **Declarações de ferramentas** | fixo, mas em *toda* volta | sim, enquanto a ferramenta estiver ativa |
| **System prompt e objetivo** | pequeno | **sempre** |

A tabela sugere a ordem de ataque, e ela é contraintuitiva para quem pensa em "resumir a conversa": o alvo principal **não é o diálogo**, são os **resultados de ferramenta**. Um `consultar_politica` que devolveu 900 tokens de texto legal foi lido, foi usado para decidir, e continua ocupando 900 tokens em todas as voltas seguintes.

---

### 4. Tool clearing — a tática mais barata

Descarte o **corpo** de resultados antigos, preservando o registro de que a chamada aconteceu:

```python
MANTER_INTEGROS = 3          # os N passos mais recentes ficam completos

def limpar_resultados(estado: Estado) -> None:
    for passo in estado.passos[:-MANTER_INTEGROS]:
        if passo.resultado and not passo.limpo:
            passo.resumo = resumir_curto(passo)     # ~30 tokens
            passo.resultado = None
            passo.limpo = True
```

O histórico enviado passa a conter, no lugar do resultado inteiro:

```json
{"role": "tool", "tool_call_id": "...",
 "content": "[resultado removido] consultar_politica(categoria=refeicao) -> teto R$ 120,00 por refeição; exige nota fiscal"}
```

| Por que é a primeira tática | |
|---|---|
| **Seletiva** | mexe só no que já foi consumido; não toca no raciocínio nem no objetivo |
| **Reversível** | o resultado completo continua no `Passo`, no estado — só saiu do que se envia. Para auditoria e trace, ele está lá |
| **Preserva a estrutura** | o modelo continua vendo que chamou aquela ferramenta com aqueles argumentos |
| **É código, não modelo** | zero chamadas extras de LLM |

O terceiro ponto é onde a compressão ingênua estraga tudo: **apagar a chamada inteira faz o agente repeti-la**. Ele não lembra de ter consultado, consulta de novo, e você criou um laço com a sua própria otimização.

---

### 5. Compaction: reativa × periódica

Quando *tool clearing* não basta, comprima o histórico inteiro num resumo. A decisão de projeto é **quando disparar**.

```python
LIMIAR = 0.6                 # REATIVA: fração da janela do modelo
if estimar_tokens(estado.historico) > LIMIAR * JANELA_DO_MODELO:
    compactar(estado)

COMPACTAR_A_CADA = 10        # PERIÓDICA: ritmo fixo
if estado.n_passos and estado.n_passos % COMPACTAR_A_CADA == 0:
    compactar(estado)
```

| | Reativa | Periódica |
|---|---|---|
| **Dispara** | só quando precisa | sempre, no ritmo fixo |
| **Custo** | menor — nenhuma compressão desnecessária | maior — comprime mesmo quando cabia |
| **Previsibilidade** | baixa: não se sabe em que passo acontece | alta |
| **Risco** | um resultado gigante pode estourar antes do próximo teste | comprimir cedo e perder detalhe ainda em uso |

Na prática a combinação vence: **periódica como piso, reativa como teto**. Voltando à aritmética da §1, com compaction periódica a cada 10 passos reduzindo o histórico a ~1.200 tokens:

| Passos | Sem compaction | Com compaction |
|---:|---:|---:|
| 10 | 36.600 | 36.600 |
| 20 | 121.200 | 85.200 |
| 30 | 253.800 | **133.800** |

Aos 30 passos, quase **metade** dos tokens — e o efeito cresce com o tamanho da trajetória, porque o que a compaction elimina é justamente o termo quadrático.

---

### 6. Sumarização de trajetória: o que não se pode perder

| Preservar sempre | Por quê |
|---|---|
| **O objetivo** | é a âncora; sem ele o agente deriva (nota 03 §7). No nosso desenho ele nem passa pela compaction — é campo do estado |
| **Decisões já tomadas** | "concluí que a despesa viola o artigo 4" precisa sobreviver, ou o agente reconclui |
| **Ações de escrita executadas** | "já registrei o parecer P-1001" — perder isto é registrar duas vezes |
| **Erros já cometidos** | "o id F-88 não existe" — perder isto é repetir a tentativa |
| **Fatos que a decisão final vai citar** | valores, prazos, artigos da política |

Pode ir embora sem dor: raciocínio intermediário, resultados já consumidos, tentativas que não levaram a nada, e o texto integral de documentos dos quais só um trecho importou.

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

Este prompt segue tudo o que a Aula 03 ensinou: contrato de saída explícito, exemplos negativos e formato definido. **O prompt de compaction é um prompt de produção** — versionado e testado como qualquer outro (Aula 03, nota 03). Um resumo que perde um identificador de escrita causa um dano que nenhum log explica depois.

> **Compaction é perda de informação com aparência de continuidade.** O agente segue funcionando, com uma memória que alguém editou. Quando ele errar por causa disso, o erro não vai parecer erro de memória — vai parecer burrice do modelo.

---

### 7. Sub-agentes como isolamento de contexto

Esta tática é diferente das outras: em vez de comprimir o contexto depois, ela **impede que ele se forme**.

```
   PRINCIPAL                            SUB-AGENTE
   ┌──────────────────┐                 ┌──────────────────┐
   │ objetivo         │  ──delega──>    │ subtarefa        │
   │ ...              │                 │ 14 passos        │
   │ [resumo 1,2k]    │  <──devolve──   │ 60k tokens       │
   │ ...              │                 │ (descartados)    │
   └──────────────────┘                 └──────────────────┘
     paga 1,2k                            gastou 60k, e some
```

É a tática certa quando a subtarefa é **verificável e delimitada**: *"encontre em qual artigo da política esta categoria é tratada e devolva o texto do artigo"*. Entra pergunta, sai resposta curta.

O preço é o mais alto das quatro:

> **O que o sub-agente não resumiu, o principal nunca saberá.** Não é compressão: é uma parede. O principal não percebe que faltou algo, porque nunca viu o que existia do outro lado.

Sub-agente para subtarefa **fechada**, com critério claro de sucesso. Sub-agente para tarefa exploratória — *"investigue isso aí e me conte"* — é uma forma cara de perder informação.

Aqui esta aula encosta em multi-agente e para. Coordenação, protocolo, agentes que revisam uns aos outros: é aula própria. Desta nota fica o uso **arquitetural** do sub-agente, como técnica de contexto.

---

### 8. A ordem de aplicação

| # | Tática | Destrói? | Quando |
|---|---|---|---|
| **0** | **Encurtar o retorno das ferramentas** | não | **antes de tudo** |
| 1 | **Tool clearing** | não — reversível, o dado fica no estado | sempre, a partir de ~5 passos |
| 2 | **Compaction periódica + reativa** | sim, de forma controlada | trajetórias longas (>10 passos) |
| 3 | **Sumarização agressiva** | sim | quando a trajetória é o custo dominante |
| 4 | **Sub-agentes** | sim, irreversivelmente | subtarefa fechada e verificável |

A tática zero vale mais que as quatro. Uma ferramenta que devolve 900 tokens quando 60 bastariam é defeito de projeto (Aula 03, nota 04 §7), e resolvê-lo na origem é sempre melhor do que comprimir depois. **Antes de escrever a primeira linha de compaction, olhe os retornos das suas ferramentas.**

---

### 9. Fechando a lista

| Pendência (Aula 03, nota 04 §9) | Situação |
|---|---|
| ~~outros padrões de arquitetura~~ | **fechado** — [nota 01](01-padroes-de-arquitetura.md) |
| ~~estado~~ (dentro de uma execução) | **fechado** — [nota 02](02-o-agente-e-o-estado.md) |
| ~~múltiplas ferramentas e a escolha entre elas~~ | **fechado** — nota 02 §6 |
| ~~confiabilidade: retry, laço, orçamento, checkpoint~~ | **fechado** — [nota 03](03-confiabilidade.md) |
| ~~context engineering dinâmica~~ | **fechado** — esta nota |
| **memória entre execuções** | aula de memória |
| **MCP** | aula de MCP |
| **multi-agente** (coordenação) | aula de sistemas multi-agente |

E duas pendências novas, criadas por esta aula: o **trace** que você passou a gravar (nota 03 §10) é a matéria-prima das aulas de **observabilidade** e **evals**. Fechar um assunto abre outro — a diferença é que agora as perguntas são melhores.

---

## Exemplos

### Exemplo 1 — O mesmo agente, com e sem tool clearing

Trajetória de 12 passos, ferramenta que devolve ~400 tokens por chamada.

```
SEM tool clearing                    COM tool clearing (3 passos íntegros)
  passo  1   enviado:  1.500           passo  1   enviado: 1.500
  passo  6   enviado:  3.900           passo  6   enviado: 2.610
  passo 12   enviado:  6.780           passo 12   enviado: 2.850  <- estabiliza
  total: ~49.700 tokens                total: ~28.900 tokens
```

A partir do passo 6 o contexto **para de crescer**: estabiliza em "três resultados inteiros + ~30 tokens de marcador por passo antigo". A curva quadrática vira quase reta, sem nenhuma chamada extra de LLM.

### Exemplo 2 — A tática zero, medida

Mesma ferramenta, dois projetos de retorno:

```python
# 900 tokens: devolve o artigo inteiro da política
{"artigo": "Art. 4º — Das despesas com alimentação. §1º O reembolso de
 refeições em viagem a serviço fica limitado ... [38 linhas] ..."}

# 60 tokens: devolve o que a decisão precisa
{"categoria": "refeicao", "teto": 120.00, "exige_nf": true,
 "artigo": "4º §1º", "excecoes": ["viagem internacional: teto 260,00"]}
```

Numa trajetória de 12 passos com 4 consultas à política, a diferença é ~3.400 tokens **por volta** a partir da quarta consulta. Nenhuma compaction recupera isso tão barato quanto reescrever o `return` da ferramenta.

### Exemplo 3 — Uma compaction que quebrou o agente

```python
# Resumo produzido por um prompt vago: "resuma a conversa acima".
resumo = """O agente analisou a despesa D-4471 do funcionário F-088,
consultou a política de refeições e concluiu que a despesa está dentro
do teto. O parecer foi elaborado."""
```

Sumiu **o identificador do parecer registrado**: "o parecer foi elaborado" não diz que `registrar_parecer` já executou e devolveu `P-1001`. No passo seguinte o agente, sem ver registro de escrita concluída, chamou `registrar_parecer` de novo.

O que salvou foi a **chave de idempotência** da [nota 03 §8](03-confiabilidade.md): a segunda chamada devolveu `{"id": "P-1001", "ja_existia": true}` e o agente seguiu.

Duas lições. A primeira: o prompt de compaction precisa exigir a preservação de escritas. A segunda, mais geral: **as salvaguardas se cobrem umas às outras** — nenhuma é suficiente sozinha, e é por isso que a arquitetura tem camadas em vez de um remédio.

### Exemplo 4 — O que o sub-agente não contou

```
PRINCIPAL delega:  "qual o teto de reembolso para refeição?"
SUB-AGENTE (9 passos, 22k tokens) devolve:  "R$ 120,00 por refeição."

Passo seguinte do principal: aprova uma refeição de R$ 180,00 em Lisboa.
```

O sub-agente tinha lido a exceção — *viagem internacional: teto R$ 260,00* — e não a incluiu, porque a pergunta não pedia. O principal aprovou por R$ 120,00 de teto e reprovou uma despesa legítima, **sem nenhum sinal de que faltava informação**.

A parede é essa. Compaction perde detalhe e deixa rastro no resumo; sub-agente perde detalhe e não deixa rastro nenhum. Por isso a pergunta delegada precisa carregar o critério inteiro: *"qual o teto aplicável a esta despesa, considerando destino e exceções?"*

---

## Exercícios resolvidos

### 1. Qual tática para cada caso?

| Situação | Tática | Por quê |
|---|---|---|
| 8 passos, ferramentas devolvem 200 tokens | **nenhuma** | ~3.100 tokens no pior passo. Comprimir é otimização prematura que só adiciona risco |
| 40 passos lendo trechos de documentos longos | **tool clearing + compaction periódica** | o volume está nos resultados; comece pela tática reversível |
| Uma ferramenta devolve 25.000 tokens | **encurtar a ferramenta** (tática zero) | é defeito de projeto; nenhuma compressão conserta a origem |
| Subtarefa de pesquisa com nº imprevisível de consultas | **sub-agente** | fechada e verificável; devolve o achado, descarta o caminho |
| Trajetória longa e o agente começa a ignorar o pedido | **reancoragem** (nota 03 §7), antes de comprimir | o problema é posição, não volume |

O último é o mais instrutivo: nem todo problema de contexto se resolve com compressão. Se o objetivo está soterrado, o remédio é **movê-lo**, não encolher o resto.

### 2. Consertar este prompt de compaction

```python
PROMPT = "Resuma a conversa acima em até 200 palavras."
```

| Defeito | Consequência | Correção |
|---|---|---|
| Não diz **o que preservar** | o modelo resume narrativamente — o que se leu, o que se conversou — que é o que não importa | lista obrigatória: decisões, escritas com identificadores, erros, fatos numéricos |
| Não diz **para quem serve** | sai uma sinopse, não um *handoff* | "resuma para que outro agente continue do ponto em que parou" |
| Não tem **exemplos negativos** | o modelo às vezes entrega a conclusão dentro do resumo, e o agente continua de uma conclusão que ninguém verificou | "não conclua a tarefa", "não invente informação" |

O limite de 200 palavras é a única coisa aproveitável — mas em tokens, e calibrado pelo que sobra do orçamento, não por um número redondo.

---

## Síntese

- O histórico inteiro é reenviado a cada volta: o **custo acumulado cresce com o quadrado** do número de passos.
- A qualidade cai **antes** de a janela acabar — o objetivo vai para o meio, que é onde o modelo lê pior.
- Curadoria dinâmica é o que o programa faz com a janela durante a execução, e só é possível porque o histórico é campo do estado.
- O maior volume está nos **resultados de ferramenta**, não no diálogo. Ataque ali primeiro.
- **Tool clearing** é a primeira tática: barata, reversível, e preserva a estrutura da trajetória. Apagar a chamada inteira faz o agente repeti-la.
- **Compaction** periódica dá previsibilidade, reativa dá economia; a combinação cobre os dois riscos.
- Um resumo preserva obrigatoriamente: objetivo, decisões, **escritas executadas com identificadores**, erros e fatos numéricos.
- O prompt de compaction é um prompt de produção — versionado e testado.
- **Sub-agente** isola contexto ao preço mais alto: o que ele não resumiu, ninguém recupera, e não fica rastro.
- A tática zero vale mais que as quatro: **encurte o retorno das suas ferramentas**.
- As salvaguardas se cobrem umas às outras — a idempotência salvou o erro da compaction.

---

## Fontes e leituras

- Aula 03, [nota 02](../../aula-03-prompt-engineering/notas-de-aula/02-context-engineering.md) — context engineering estática, cuja promessa esta nota cumpre; [nota 03](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) — versionamento, que vale para o prompt de compaction; [nota 04 §7](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — projeto de ferramenta, a origem da tática zero.
- Aula 02, [nota 01 §4](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — janela, custo quadrático da atenção e *lost in the middle*; [nota 04 §5](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md) — latência e tamanho da entrada.
- **Effective context engineering for AI agents** — Anthropic (2025). Compaction, *tool clearing* e sub-agentes.
- **Building Effective Agents** — Anthropic (2024). Agentes autônomos e o custo da trajetória.
- **Lost in the Middle: How Language Models Use Long Contexts** — Liu et al. (2023). O viés em U que fundamenta a reancoragem.
