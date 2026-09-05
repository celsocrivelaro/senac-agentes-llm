# IA Aplicada com LLMs — Aula 05: Arquitetura de agentes — Context engineering dinâmica

## Introdução

A nota 02 da Aula 03 encerrou com uma promessa: naquela nota, o que se monta na chamada; na aula de agentes, o que se faz com a janela **ao longo da trajetória**. Esta nota cumpre a promessa e trata da terceira limitação enunciada na abertura da aula: o laço **não sabe esquecer**.

> **A cada volta, o histórico íntegro é reenviado. Nada é removido.**

Numa trajetória de três passos, a propriedade é irrelevante. Numa de trinta, ela domina o custo, a latência e — o efeito menos esperado — a **qualidade**.

> **Pré-requisitos:** notas [02](02-o-agente-e-o-estado.md) e [03](03-confiabilidade.md) desta aula · Aula 03, [nota 02](../../aula-03-prompt-engineering/notas-de-aula/02-context-engineering.md) integralmente · Aula 02, [nota 01](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) §4.
>
> **Código:** [`06-compaction.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/06-compaction.py) — trajetória longa com e sem *compaction*, com a contagem de tokens por passo.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Calcular** o crescimento do contexto ao longo de uma trajetória e demonstrar que é superlinear no acumulado.
- **Distinguir** curadoria estática — o que se monta na chamada — de curadoria dinâmica — o que o programa faz durante a execução.
- **Aplicar** *tool clearing*, *compaction* reativa e periódica, e sumarização de trajetória.
- **Determinar o que preservar obrigatoriamente** num resumo e o que pode ser descartado.
- **Empregar sub-agentes como isolamento de contexto**, com conhecimento preciso da perda envolvida.
- **Ordenar** as táticas da mais segura à mais destrutiva.

---

## Desenvolvimento teórico

### 1. A aritmética do acumulado

Considere um agente cujo *system prompt* e declarações de ferramentas somam aproximadamente 1.500 tokens, e em que cada passo acrescenta ao histórico a mensagem do modelo (cerca de 80 tokens) e o resultado da ferramenta (cerca de 400 tokens) — aproximadamente **480 tokens por passo**. O contexto enviado no passo *n* é `1.500 + 480 × (n − 1)`; o total consumido é a soma desses valores desde o primeiro passo.

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

> Os valores acima resultam da aritmética do acumulado, não de medição. A execução do `06-compaction.py` com o modelo em uso permite refazer a tabela: a **forma** da curva não se altera, os valores sim.

Três leituras decorrem do quadro:

| Efeito | Descrição |
|---|---|
| **Custo quadrático** | cada passo reenvia todo o histórico anterior. Dobrar o número de passos quadruplica a conta: um agente de 30 passos custa aproximadamente **sete** vezes um de 10 |
| **Latência proporcional** | o tempo até o primeiro token depende do tamanho da entrada (Aula 02, nota 04, §5). A execução inicia em cerca de 400 ms por passo e encerra em vários segundos |
| **Degradação de qualidade anterior ao limite da janela** | no passo 25, o objetivo está cercado por cerca de 12.000 tokens de observações — a região central, onde a recuperação é pior (*lost in the middle*, LIU et al., 2023) |

O terceiro efeito é o menos intuitivo e o mais relevante: o desempenho do agente não se degrada por falta de espaço, e sim por **excesso de passado**.

> **Formulação:** num agente, a janela de contexto (*context window*) não é um limite que se atinge. É um recurso que se degrada continuamente à medida que é consumido.

---

### 2. Curadoria estática × dinâmica

```
   ESTÁTICO (aula 03)                DINÂMICO (esta nota)
   ┌───────────────────┐             ┌───────────────────┐
   │ system            │             │ system            │  preservado
   │ exemplos          │             │ objetivo          │  preservado
   │ schema            │   ──────>   │ ····· resumo ·····│  comprimido
   │ documentos        │             │ passos 24-30      │  íntegros
   │ pergunta          │             │ objetivo (de novo)│  reancorado
   └───────────────────┘             └───────────────────┘
   decisão única, no código          decisão a cada volta, no programa
```

A curadoria estática é uma decisão tomada uma vez, na redação do código. A dinâmica é o tratamento aplicado à janela **durante** a execução, sem conhecimento prévio do número de passos nem do conteúdo devolvido por cada ferramenta.

Sua implementação depende da [nota 02](02-o-agente-e-o-estado.md): reescrever o histórico exige que ele seja **campo do estado**, e não a estrutura em que o agente reside. Um agente cujo estado é `mensagens[]` não pode comprimir o histórico sem se apagar.

---

### 3. A composição da janela

| Origem | Volume | Utilidade posterior |
|---|---|---|
| **Resultados de ferramenta** | o maior, com folga | quase sempre **nula** — uma vez consumido, o resultado converteu-se em decisão |
| **Raciocínio intermediário** | médio | rara |
| **Declarações de ferramentas** | fixo, em *toda* volta | permanente, enquanto a ferramenta estiver ativa |
| **System prompt e objetivo** | pequeno | **permanente** |

A tabela indica a ordem de intervenção, e ela é contraintuitiva para quem pensa em "resumir a conversa": o alvo principal **não é o diálogo**, são os **resultados de ferramenta**. Uma chamada a `consultar_politica` que devolveu 900 tokens de texto normativo foi lida, foi utilizada na decisão e continua ocupando 900 tokens em todas as voltas subsequentes.

---

### 4. Tool clearing

A tática consiste em descartar o **corpo** de resultados antigos, preservando o registro de que a chamada ocorreu:

```python
MANTER_INTEGROS = 3          # os N passos mais recentes permanecem completos

def limpar_resultados(estado: Estado) -> None:
    for passo in estado.passos[:-MANTER_INTEGROS]:
        if passo.resultado and not passo.limpo:
            passo.resumo = resumir_curto(passo)     # ~30 tokens
            passo.resultado = None
            passo.limpo = True
```

O histórico enviado passa a conter, no lugar do resultado íntegro:

```json
{"role": "tool", "tool_call_id": "...",
 "content": "[resultado removido] consultar_politica(categoria=refeicao) -> teto R$ 120,00 por refeição; exige nota fiscal"}
```

| Propriedade | Descrição |
|---|---|
| **Seletiva** | atua apenas sobre o que já foi consumido; não afeta raciocínio nem objetivo |
| **Reversível** | o resultado íntegro permanece no `Passo`, no estado — foi removido apenas do que se envia. Permanece disponível para auditoria e para o *trace* |
| **Preserva a estrutura** | o modelo continua registrando que a ferramenta foi invocada com aqueles argumentos |
| **Determinística** | é código, não modelo: nenhuma chamada adicional |

A terceira propriedade é o ponto em que a compressão ingênua falha: **remover a chamada íntegra induz o agente a repeti-la**. Sem registro da consulta anterior, ele consulta novamente — e a otimização produziu um laço.

---

### 5. Compaction: reativa × periódica

Quando o *tool clearing* é insuficiente, comprime-se o histórico íntegro num resumo. A decisão de projeto está no critério de disparo.

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
| **Disparo** | ao cruzar o limiar | a cada N passos |
| **Custo** | menor — nenhuma compressão desnecessária | maior — comprime mesmo quando desnecessário |
| **Previsibilidade** | baixa: o passo de disparo é desconhecido | alta |
| **Risco** | um único resultado extenso pode exceder a janela antes da verificação seguinte | comprimir prematuramente e descartar detalhe ainda em uso |

Na prática, a combinação prevalece: **periódica como piso, reativa como teto**. Retomando a aritmética da §1, com *compaction* periódica a cada 10 passos reduzindo o histórico a cerca de 1.200 tokens:

| Passos | Sem compaction | Com compaction |
|---:|---:|---:|
| 10 | 36.600 | 36.600 |
| 20 | 121.200 | 85.200 |
| 30 | 253.800 | **133.800** |

Aos 30 passos, a redução aproxima-se de metade dos tokens — e o efeito cresce com o comprimento da trajetória, uma vez que o termo eliminado é justamente o quadrático.

---

### 6. Sumarização de trajetória: o conjunto mínimo

| Preservar obrigatoriamente | Justificativa |
|---|---|
| **O objetivo** | é a âncora; sem ele o agente deriva (nota 03, §7). No desenho adotado, ele não passa pela *compaction* — é campo do estado |
| **Decisões tomadas** | "a despesa viola o artigo 4" precisa sobreviver, sob pena de reconclusão |
| **Ações de escrita executadas** | "o parecer P-1001 foi registrado" — a perda dessa informação produz registro duplicado |
| **Erros cometidos** | "o identificador F-88 não existe" — a perda induz repetição da tentativa |
| **Fatos numéricos citados** | valores, prazos, artigos da política que a decisão final referenciará |

Podem ser descartados sem prejuízo: raciocínio intermediário, resultados já consumidos, tentativas infrutíferas e o texto íntegro de documentos dos quais apenas um trecho foi relevante.

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

O prompt observa os requisitos estabelecidos na Aula 03: contrato de saída explícito, exemplos negativos e formato definido. **O prompt de compaction é um prompt de produção** — versionado e submetido a suíte de regressão como qualquer outro (Aula 03, nota 03). Um resumo que omite um identificador de escrita produz um dano que nenhum registro posterior explica.

> **Compaction é perda de informação com aparência de continuidade.** O agente prossegue em operação, com uma memória que foi editada. O erro decorrente não se apresenta como falha de memória: apresenta-se como incapacidade do modelo.

---

### 7. Sub-agentes como isolamento de contexto

Esta tática difere das anteriores: em vez de comprimir o contexto posteriormente, **impede sua formação**.

```
   PRINCIPAL                            SUB-AGENTE
   ┌──────────────────┐                 ┌──────────────────┐
   │ objetivo         │  ──delega──>    │ subtarefa        │
   │ ...              │                 │ 14 passos        │
   │ [resumo 1,2k]    │  <──devolve──   │ 60k tokens       │
   │ ...              │                 │ (descartados)    │
   └──────────────────┘                 └──────────────────┘
     consome 1,2k                         consumiu 60k, e encerra
```

É a tática adequada quando a subtarefa é **verificável e delimitada**: *"identifique em qual artigo da política esta categoria de despesa é tratada e devolva o texto do artigo"*. Entra uma pergunta, sai uma resposta curta.

O custo é o mais elevado das quatro táticas:

> **O que o sub-agente não incluir no resumo, o principal jamais conhecerá.** Não se trata de compressão, e sim de barreira. O principal não dispõe de meio para constatar a omissão, uma vez que nunca teve acesso ao conteúdo original.

Daí a restrição: sub-agente para subtarefa **fechada**, com critério de sucesso explícito. Sub-agente para tarefa exploratória — *"investigue e relate"* — constitui forma custosa de perder informação.

Este é o ponto em que a aula toca em sistemas multiagente e se detém. Coordenação, protocolo de comunicação e revisão mútua entre agentes são objeto de aula própria. Desta nota permanece o uso **arquitetural** do sub-agente, como técnica de gestão de contexto.

---

### 8. Ordem de aplicação

| # | Tática | Destrutiva | Aplicação |
|---|---|---|---|
| **0** | **Encurtar o retorno das ferramentas** | não | **antes de qualquer outra** |
| 1 | *Tool clearing* | não — reversível, o dado permanece no estado | sempre, a partir de aproximadamente 5 passos |
| 2 | *Compaction* periódica + reativa | sim, de forma controlada | trajetórias longas (mais de 10 passos) |
| 3 | Sumarização agressiva | sim | quando a trajetória é o custo dominante |
| 4 | Sub-agentes | sim, irreversivelmente | subtarefa fechada e verificável |

A tática zero tem retorno superior ao das quatro seguintes. Uma ferramenta que devolve 900 tokens onde 60 seriam suficientes constitui defeito de projeto da ferramenta (Aula 03, nota 04, §7), e corrigi-lo na origem é sempre preferível a comprimir posteriormente. **Antes da primeira linha de compaction, examinam-se os retornos das ferramentas.**

---

### 9. Encerramento da lista de pendências

| Pendência (Aula 03, nota 04, §9) | Situação |
|---|---|
| ~~outros padrões de arquitetura~~ | **encerrada** — [nota 01](01-padroes-de-arquitetura.md) |
| ~~estado, no interior de uma execução~~ | **encerrada** — [nota 02](02-o-agente-e-o-estado.md) |
| ~~múltiplas ferramentas e a escolha entre elas~~ | **encerrada** — nota 02, §6 |
| ~~confiabilidade: retentativa, laço, orçamento, checkpoint~~ | **encerrada** — [nota 03](03-confiabilidade.md) |
| ~~context engineering dinâmica~~ | **encerrada** — esta nota |
| **memória entre execuções** | aula de memória |
| **MCP** | aula de MCP |
| **multiagente** (coordenação) | aula de sistemas multiagente |

Duas pendências novas foram criadas por esta aula: o **trace** registrado a partir da nota 03, §10, é a matéria-prima das aulas de **observabilidade** e de **evals**. O encerramento de um assunto abre outro, com a diferença de que as perguntas subsequentes são mais precisas.

---

## Exemplos

### Exemplo 1 — O mesmo agente, com e sem tool clearing

Trajetória de 12 passos, com ferramenta que devolve aproximadamente 400 tokens por chamada.

```
SEM tool clearing                    COM tool clearing (3 passos íntegros)
  passo  1   enviado:  1.500           passo  1   enviado: 1.500
  passo  6   enviado:  3.900           passo  6   enviado: 2.610
  passo 12   enviado:  6.780           passo 12   enviado: 2.850  <- estabiliza
  total: ~49.700 tokens                total: ~28.900 tokens
```

A partir do passo 6 o contexto **cessa de crescer**: estabiliza em três resultados íntegros mais aproximadamente 30 tokens de marcador por passo antigo. A curva quadrática converte-se em curva quase linear, sem nenhuma chamada adicional ao modelo.

### Exemplo 2 — A tática zero, medida

A mesma ferramenta, sob dois projetos de retorno:

```python
# 900 tokens: devolve o artigo íntegro da política
{"artigo": "Art. 4º — Das despesas com alimentação. §1º O reembolso de
 refeições em viagem a serviço fica limitado ... [38 linhas] ..."}

# 60 tokens: devolve o que a decisão requer
{"categoria": "refeicao", "teto": 120.00, "exige_nf": true,
 "artigo": "4º §1º", "excecoes": ["viagem internacional: teto 260,00"]}
```

Numa trajetória de 12 passos com 4 consultas à política, a diferença é de aproximadamente 3.400 tokens **por volta** a partir da quarta consulta. Nenhum mecanismo de *compaction* recupera esse volume ao custo de reescrever o `return` da ferramenta.

### Exemplo 3 — Uma compaction que comprometeu a execução

```python
# Resumo produzido por um prompt genérico: "resuma a conversa acima".
resumo = """O agente analisou a despesa D-4471 do funcionário F-088,
consultou a política de refeições e concluiu que a despesa está dentro
do teto. O parecer foi elaborado."""
```

O elemento omitido é **o identificador do parecer registrado**: "o parecer foi elaborado" não informa que `registrar_parecer` foi executado e devolveu `P-1001`. No passo seguinte, o agente — sem registro de escrita concluída — invocou `registrar_parecer` novamente.

A execução foi preservada pela **chave de idempotência** ([nota 03](03-confiabilidade.md), §8): a segunda chamada devolveu `{"id": "P-1001", "ja_existia": true}` e o agente prosseguiu.

Duas conclusões. A primeira: o prompt de *compaction* precisa exigir a preservação das ações de escrita. A segunda, de alcance mais geral: **as salvaguardas se cobrem mutuamente**. Nenhuma é suficiente isoladamente, e é por essa razão que a arquitetura é composta por camadas em vez de um mecanismo único.

### Exemplo 4 — A informação que o sub-agente omitiu

```
PRINCIPAL delega:  "qual o teto de reembolso para refeição?"
SUB-AGENTE (9 passos, 22k tokens) devolve:  "R$ 120,00 por refeição."

Passo seguinte do principal: aprova uma refeição de R$ 180,00 em Lisboa.
```

O sub-agente havia lido a exceção — *viagem internacional: teto R\$ 260,00* — e não a incluiu, porque a pergunta não a solicitava. O principal aplicou o teto de R\$ 120,00 e reprovou uma despesa legítima, **sem qualquer indício de informação faltante**.

A distinção é essa: *compaction* descarta detalhe e deixa vestígio no resumo; sub-agente descarta detalhe sem deixar vestígio algum. Daí a exigência de que a pergunta delegada carregue o critério íntegro: *"qual o teto aplicável a esta despesa, considerados destino e exceções?"*

---

## Fontes e leituras

ANTHROPIC. **Effective context engineering for AI agents**. 2025. (*Compaction*, *tool clearing* e sub-agentes como táticas de gestão de contexto ao longo da trajetória.)

ANTHROPIC. **Building effective agents**. 2024. (Seção sobre agentes autônomos e o custo da trajetória.)

LIU, N. F. et al. **Lost in the middle**: how language models use long contexts. arXiv:2307.03172, 2023. (Fundamenta a reancoragem do objetivo.)

**Material da disciplina.** Aula 03, [nota 02](../../aula-03-prompt-engineering/notas-de-aula/02-context-engineering.md) — context engineering estática, cuja promessa esta nota cumpre; [nota 03](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) — versionamento, aplicável ao prompt de *compaction*; [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) §7 — projeto de ferramenta, origem da tática zero. Aula 02, [nota 01](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) §4 — janela, custo quadrático da atenção e *lost in the middle*; [nota 04](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md) §5 — latência e tamanho da entrada.
