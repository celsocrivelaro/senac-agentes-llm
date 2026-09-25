# IA Aplicada com LLMs — Aula 08: Memória — Escrever e ler

## Introdução

A [nota anterior](02-checkpoint-nao-e-memoria.md) estabeleceu o que a memória é e quais estruturas a compõem. Esta trata das duas operações que a movimentam, e a primeira delas é a decisão mais difícil da aula.

**Quem decide o que entra na memória?** Há três respostas defensáveis, com custos e modos de falha distintos, e a escolha entre elas determina se o sistema acumula conhecimento ou acumula ruído.

A segunda operação é a leitura, e ela traz um problema que a Aula 05 já formulava: memória é mais uma fonte disputando espaço numa janela que já tinha dono.

Uma advertência atravessa a nota inteira, e ela não é nova: **escrever na memória é escrita**. Valem a fronteira leitura/escrita da Aula 01 (nota 03, §3) e a chave de idempotência da Aula 05 ([nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md), §8), com todas as obrigações que elas impõem.

> **Pré-requisitos:** [nota 02](02-checkpoint-nao-e-memoria.md) desta aula · Aula 01, [nota 03 §3](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) · Aula 05, [nota 02 §1 e §8](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) e [nota 03 §1](../../aula-05-arquitetura-de-agentes/notas-de-aula/03-context-engineering-dinamica.md) — a aritmética do acumulado, cujo tratamento esta nota conclui.
>
> **Código:** [`03-quem-escreve.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula08-memoria/03-quem-escreve.py).

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Comparar** as três políticas de escrita em custo, cobertura e modo de falha, e defender uma escolha.
- **Aplicar** chave de idempotência a uma operação de escrita em memória.
- **Redigir** um prompt de extração como prompt de produção.
- **Orçar** a leitura da memória, tratando-a como fonte que disputa espaço na janela.
- **Aplicar** *tool clearing* e *compaction* — reativa e periódica — sobre a trajetória corrente.
- **Determinar o que preservar obrigatoriamente** num resumo, e reconhecer que essa lista é quase a mesma da política de escrita da memória.

---

## Desenvolvimento teórico

### 1. Três políticas de escrita

As três respondem à mesma pergunta: **o que, da execução que terminou, é promovido de curto para longo prazo**. É a etapa que a literatura chama de *extraction* ([nota 02](02-checkpoint-nao-e-memoria.md), §2.1), e a escolha entre as políticas é quem decide o que atravessa a fronteira.

| Política | Custo | Cobertura | Modo de falha |
|---|---|---|---|
| **O agente decide** — ferramenta `lembrar` | uma decisão por passo | captura o imprevisto | armazena ruído; a memória degrada |
| **O código extrai** — um passo sobre o resultado | uma chamada por execução | apenas o previsto no prompt | perde o que a extração não antecipou |
| **O humano corrige** | trabalho humano | a mais confiável | não escala |

#### 1.1 O agente decide

O agente recebe uma ferramenta `lembrar(conteudo, tipo)` e a invoca quando julgar pertinente. A vantagem é real: ele tem acesso ao raciocínio que produziu a informação, e pode registrar algo que nenhuma extração posterior identificaria.

O modo de falha também é real, e decorre da mesma propriedade que torna a Aula 05 necessária: **um agente sem restrição explícita tende a exercer as capacidades disponíveis**. Numa execução de quatro passos, a chamada a `lembrar` ocorre tipicamente duas a quatro vezes, e boa parte do que se grava é o ruído que a [nota 02](02-checkpoint-nao-e-memoria.md), §4, enumera — latência, contagem de passos, tentativas descartadas.

A restrição adequada é arquitetural, não instrucional. É a lição da Aula 05 ([nota 01](../../aula-08-memoria/notas-de-aula/01-o-agente-e-o-estado.md), §6): declarar `lembrar` apenas na fase final da execução limita quando ela pode ser chamada, o que instrução no *system prompt* não faz.

#### 1.2 O código extrai

Um passo determinístico sobre o resultado da execução:

```python
def extrair_pelo_codigo(execucao: str) -> list[dict]:
    bruto = _chamar(PROMPT_EXTRACAO.format(execucao=execucao),
                    schema=SCHEMA_EXTRACAO, nome="extracao")
    return json.loads(bruto)["fatos"]
```

Custo previsível — uma chamada por execução, independentemente do número de passos — e comportamento uniforme. É a política adequada como padrão.

O que ela perde é o que o prompt não previu, o que faz do prompt o componente crítico:

```python
PROMPT_EXTRACAO = """A execução abaixo terminou. Classifique cada informação
que ela produziu para decidir o que sobrevive a ela.

TIPOS:
- episodica: o que aconteceu e quando, com identificadores
- semantica: fato estável sobre uma entidade (funcionário, categoria)
- procedural: regra de como agir, aplicável a execuções futuras
- nao_guardar: ruído de execução — latência, contagem de tokens, número de
  passos, tentativas descartadas

PRESERVE OBRIGATORIAMENTE como `episodica` toda AÇÃO DE ESCRITA executada,
com o identificador que ela devolveu (ex: "registrou o parecer P-1188 para
a despesa D-4612"). Perder essa informação faz o agente repetir a escrita
numa execução futura.

A maior parte do que uma execução produz é `nao_guardar`. Não force
classificação: memória cheia de ruído é pior que memória vazia, porque o
ruído compete por espaço na janela.
"""
```

Duas cláusulas fazem trabalho distinto. A **primeira** transporta para cá uma
lição da §6 desta nota: o resumo que perde o identificador de
uma escrita já executada leva o agente a executá-la novamente. Ali o dano
ocorria dentro de uma execução; aqui, entre execuções — e a chave de
idempotência da §2 é a segunda camada que o contém.

O segundo parágrafo é o que impede o comportamento indesejado. Um modelo solicitado a classificar informações tende a distribuí-las entre as categorias oferecidas; declarar explicitamente que **a maioria pertence a uma delas** corrige o viés. É a mesma técnica que a Aula 05 ([nota 01](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-0-padroes-de-arquitetura.md), §3.2) aplica à rota `nenhuma` de um roteador.

#### 1.3 O humano corrige

A mais confiável e a menos escalável. É também a primeira que se abandona quando o volume cresce.

O desenho que se sustenta na prática é misto: **o código extrai por padrão, e o humano revisa a memória procedural**. A assimetria se justifica pelo que a [nota 02](02-checkpoint-nao-e-memoria.md), §3.3, estabeleceu — episódios e fatos incorretos são filtrados pela relevância na recuperação, mas uma regra procedural incorreta aplica-se a todas as execuções seguintes, sem filtro.

---

### 2. Escrever na memória é escrita

A operação tem todas as obrigações de qualquer escrita, e a mais importante é a idempotência:

```python
def gravar(self, entidade: str, chave: str, valor, data: str) -> dict:
    """Chave de idempotência derivada do CONTEÚDO."""
    id_fato = f"{entidade}:{chave}"
    anterior = self.fatos.get(id_fato)
    if anterior and anterior["valor"] == valor:
        return {**anterior, "ja_existia": True}
    ...
```

A chave é `entidade:chave` — derivada do **conteúdo**, não da tentativa. É exatamente a exigência da Aula 05 ([nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md), §8), e pelo mesmo motivo: um identificador gerado por `uuid4()` a cada chamada não é chave de idempotência, é identificador novo por tentativa.

O cenário que a torna necessária é concreto. Uma execução reprocessada — por retentativa, por retomada de *checkpoint* ou por reexecução manual — passa duas vezes pela extração. Sem a chave, a memória registra o mesmo fato duas vezes; e **memória duplicada é memória contraditória** assim que uma das cópias for atualizada e a outra não.

O retorno inclui `ja_existia: True`, e o campo tem destinatário: quem chamou. É a mesma decisão que a Aula 05 tomou em `registrar_parecer` — uma escrita idempotente que devolve resposta idêntica oculta do chamador o fato de que houve repetição.

O registro do campo `substituiu` completa o mecanismo:

```python
registro = {"entidade": entidade, "chave": chave, "valor": valor,
            "data": data, "substituiu": anterior["valor"] if anterior else None}
```

Um fato que substitui outro guarda o valor anterior. É o que permite auditar uma decisão tomada com base num valor que já mudou — e é o embrião do tratamento de contradição da [nota 04](04-esquecer-e-proteger.md).

---

### 3. A memória procedural exige idempotência de outra natureza

```python
def gravar(self, regra: str) -> bool:
    atuais = self.regras()
    if regra in atuais:
        return False
    ...
```

A comparação é por igualdade textual, o que a torna frágil: duas formulações da mesma regra são registradas como regras distintas.

A fragilidade é reconhecida e aceita, com uma justificativa. Deduplicar regras por similaridade semântica exigiria embutir cada regra e comparar por cosseno — e a Aula 07 ([nota 02](../../aula-07-rag-e-documentos/notas-de-aula/02-o-que-o-embedding-nao-ve.md), §2) demonstrou que o vetor não distingue uma regra de sua **negação**. Uma deduplicação por similaridade fundiria *"devolver antes de analisar"* e *"não devolver antes de analisar"*, que é um erro pior que a duplicação.

O tratamento adequado é a revisão humana da §1.3, e a lista curta de regras que ela viabiliza.

---

### 4. A leitura: memória compete por janela

A §5.1 desta nota enumera o que ocupa a janela de um agente: *system prompt*, objetivo, trajetória e declarações de ferramentas. A Aula 07 acrescentou os trechos recuperados. A memória é a **quinta** fonte — e a única que cresce indefinidamente entre execuções.

```python
@dataclass
class OrcamentoDeContexto:
    max_tokens_memoria: int = 800
    max_episodios: int = 3

    def montar(self, episodica: list[dict], semantica: dict,
               procedural: str) -> dict:
        ...
        return {"texto": texto, "tokens": tokens}
```

Três decisões de projeto, e nenhuma é nova:

**O teto é parâmetro, não constante.** Mesmo argumento do orçamento da Aula 05 ([nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md), §1): tarefas diferentes admitem quantidades diferentes de memória, e uma constante no meio do arquivo não é ajustada por ninguém.

**O método devolve o custo junto com o conteúdo.** Sem o número de tokens, não há como afirmar que a memória compensou — e a [nota 04](04-esquecer-e-proteger.md) fecha a aula justamente com essa conta.

**A ordem de montagem não é arbitrária.** O bloco reúne procedural, semântica e episódica nessa sequência, e a episódica — a mais volumosa e a menos confiável — fica adjacente à trajetória corrente. É a aplicação do viés de posição da Aula 02 (nota 01, §4.3): o que é mais importante ocupa as extremidades do contexto.

#### 4.1 Quando recuperar

Duas opções, e a escolha é a mesma que a Aula 07 ([nota 03](../../aula-07-rag-e-documentos/notas-de-aula/03-medir-o-rag-e-a-recuperacao-como-ferramenta.md), §5) enfrentou num objeto diferente:

| | Recuperar no início | Recuperar sob demanda, como ferramenta |
|---|---|---|
| Custo | 1 consulta por execução | variável, exige teto |
| Risco | traz o irrelevante | o agente não busca quando deveria |
| Previsibilidade | total | nenhuma |

E vale a mesma conclusão: a decisão é por tarefa, não por sistema. Para uma triagem de despesa simples, recuperar no início é suficiente e previsível; para uma investigação, a ferramenta se justifica.

> **Memória incorreta é pior que memória ausente.** Um fato irrelevante no contexto desvia a decisão, e o log não registra a causa — o agente simplesmente decidiu diferente, e nada indica que a memória o influenciou. É o modo de falha mais difícil de diagnosticar desta aula, e o argumento mais forte a favor de guardar pouco.

---

### 5. A trajetória também compete: comprimir o histórico

A §4 tratou de quanto a **memória** pode ocupar. Falta a fonte que cresce mais rápido que todas: a **trajetória da execução corrente**. Tudo o que esta seção trata é gestão de **memória de curto prazo**: nada do que ela descarta sobrevive à execução, e é por isso que descartar é admissível. A cada volta do laço, o histórico íntegro é reenviado, e nada sai dele.

A Aula 05 ([nota 03](../../aula-05-arquitetura-de-agentes/notas-de-aula/03-context-engineering-dinamica.md), §1) fez a aritmética: o consumo acumulado cresce com o **quadrado** do número de passos, e a qualidade cai antes de a janela acabar. O que ela deixou para cá foram as técnicas — e as deixou por uma razão que a §6 torna evidente: **comprimir trajetória e escrever memória são a mesma decisão**, tomada sobre materiais diferentes.

#### 5.1 A composição da janela

| Origem | Volume | Utilidade posterior |
|---|---|---|
| **Resultados de ferramenta** | o maior, com folga | quase sempre **nula** — uma vez consumido, o resultado converteu-se em decisão |
| **Raciocínio intermediário** | médio | rara |
| **Declarações de ferramentas** | fixo, em *toda* volta | permanente, enquanto a ferramenta estiver ativa |
| **System prompt e objetivo** | pequeno | **permanente** |

A tabela indica a ordem de intervenção, e ela é contraintuitiva para quem pensa em "resumir a conversa": o alvo principal **não é o diálogo**, são os **resultados de ferramenta**. Uma chamada a `consultar_politica` que devolveu 900 tokens de texto normativo foi lida, foi utilizada na decisão e continua ocupando 900 tokens em todas as voltas subsequentes.

---

#### 5.2 Tool clearing

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

#### 5.3 Compaction: reativa × periódica

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

Na prática, a combinação prevalece: **periódica como piso, reativa como teto**. Retomando a aritmética da Aula 05 (nota 03, §1), com *compaction* periódica a cada 10 passos reduzindo o histórico a cerca de 1.200 tokens:

| Passos | Sem compaction | Com compaction |
|---:|---:|---:|
| 10 | 36.600 | 36.600 |
| 20 | 121.200 | 85.200 |
| 30 | 253.800 | **133.800** |

Aos 30 passos, a redução aproxima-se de metade dos tokens — e o efeito cresce com o comprimento da trajetória, uma vez que o termo eliminado é justamente o quadrático.

---

### 6. O conjunto mínimo: o que um resumo nunca pode perder

| Preservar obrigatoriamente | Justificativa |
|---|---|
| **O objetivo** | é a âncora; sem ele o agente deriva (Aula 05, nota 02, §7). No desenho adotado, ele não passa pela *compaction* — é campo do estado |
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


> **E aqui as duas metades da aula se encontram.** A lista acima é o que um resumo de trajetória precisa preservar. Releia a política de extração da §1.2: ela decide o que a **memória** precisa preservar. As duas listas são quase a mesma — objetivo, decisões com a razão, escritas executadas com seus identificadores, erros cometidos, fatos numéricos —, e a coincidência não é acidente. Comprimir e memorizar respondem à mesma pergunta: *o que, desta execução, ainda vai importar?*
>
> A diferença está no horizonte. A compressão pergunta o que importa **até o fim desta execução**; a memória pergunta o que importa **daqui em diante**. É por isso que a memória pode descartar o que a compressão preserva — e o exemplo 2 mostra onde as duas listas divergem.

---

## Exemplos

### Exemplo 1 — A mesma execução, sob a política de extração

Execução analisada:

```
Execução exec-f2a0, 06/10/2026. Objetivo: analisar a despesa D-4612 do
funcionário F-088 (refeição de R$ 245,00 por pessoa, Lisboa, com nota).

passo 0  consultar_historico(F-88)  -> erro: funcionário inexistente
passo 1  consultar_historico(F-088) -> duas viagens a Lisboa em 2026
passo 2  consultar_politica(refeicao) -> Art. 4º §1º teto R$ 120,00;
         §2º viagem internacional teto R$ 260,00
passo 3  registrar_parecer(D-4612, aprovado) -> P-1188

Consumo: 4 passos, 2.410 tokens, 1.180 ms, R$ 0,0061.
```

Classificação obtida, em uma chamada:

```
  [episodica] 2
      D-4612 do F-088 aprovada em 06/10/2026 pelo Art. 4º §2º (R$ 245,00
      dentro do teto de R$ 260,00 para viagem internacional)
      registrou o parecer P-1188 para a despesa D-4612
  [semantica] 1
      F-088 realizou duas viagens a Lisboa em 2026
  [procedural] 1
      Ids de funcionário têm o formato F seguido de três dígitos
  [nao_guardar] 3
      4 passos · 2.410 tokens · 1.180 ms
```

*(Saída ilustrativa.)*

Três das sete informações foram descartadas. A pergunta a fazer diante dessa lista não é se a classificação está correta — é **o que a extração perdeu**, porque ela só captura o que o prompt solicitou.

A segunda entrada episódica está ali por uma cláusula explícita do prompt. Sem ela, o registro de que `registrar_parecer` devolveu `P-1188` desapareceria — e o exemplo 5 desta nota documenta o que acontece então: o agente registra o parecer de novo.

### Exemplo 2 — As duas listas de "preservar obrigatoriamente"

O prompt de extração desta aula e o prompt de *compaction* da §6 resolvem problemas diferentes com listas quase idênticas:

| Compaction (§6 desta nota) | Extração (§1.2) |
|---|---|
| decisões tomadas e suas razões | episódica |
| **ações de escrita executadas, com identificadores** | episódica, por cláusula explícita |
| erros cometidos e o que se aprendeu | procedural |
| fatos numéricos citados | semântica |
| *(descartar: raciocínio intermediário, resultados já usados)* | `nao_guardar` |

A correspondência é linha a linha, e a segunda foi transportada deliberadamente: o prompt de extração desta aula tem a cláusula de escrita porque a Aula 05 (nota 03, §1) demonstrou o custo de não tê-la.

A convergência das duas listas não é coincidência: **compaction e extração de memória são a mesma operação em escalas diferentes**. Uma comprime dentro de uma execução; a outra, entre execuções. Ambas são perda de informação com aparência de continuidade.

### Exemplo 3 — A gravação repetida

```python
primeira = semantica.gravar("F-088", "destinos_frequentes", ["Lisboa"], "2026-10-06")
segunda  = semantica.gravar("F-088", "destinos_frequentes", ["Lisboa"], "2026-10-06")

# primeira -> {'entidade': 'F-088', 'chave': 'destinos_frequentes',
#              'valor': ['Lisboa'], 'data': '2026-10-06', 'substituiu': None}
# segunda  -> {..., 'ja_existia': True}
```

A segunda chamada não cria registro e informa a repetição. Sem esse comportamento, uma execução reprocessada duplicaria o fato — e a duplicação só se manifestaria como problema semanas depois, quando uma das cópias fosse atualizada.

É o mesmo argumento da Aula 05: as salvaguardas que se acrescentam a um sistema **aumentam** a probabilidade de repetição, e a idempotência deixa de ser prudência para tornar-se requisito.

---

### Exemplo 4 — O mesmo agente, com e sem tool clearing

Trajetória de 12 passos, com ferramenta que devolve aproximadamente 400 tokens por chamada.

```
SEM tool clearing                    COM tool clearing (3 passos íntegros)
  passo  1   enviado:  1.500           passo  1   enviado: 1.500
  passo  6   enviado:  3.900           passo  6   enviado: 2.610
  passo 12   enviado:  6.780           passo 12   enviado: 2.850  <- estabiliza
  total: ~49.700 tokens                total: ~28.900 tokens
```

A partir do passo 6 o contexto **cessa de crescer**: estabiliza em três resultados íntegros mais aproximadamente 30 tokens de marcador por passo antigo. A curva quadrática converte-se em curva quase linear, sem nenhuma chamada adicional ao modelo.

---

### Exemplo 5 — Uma compaction que comprometeu a execução

```python
# Resumo produzido por um prompt genérico: "resuma a conversa acima".
resumo = """O agente analisou a despesa D-4471 do funcionário F-088,
consultou a política de refeições e concluiu que a despesa está dentro
do teto. O parecer foi elaborado."""
```

O elemento omitido é **o identificador do parecer registrado**: "o parecer foi elaborado" não informa que `registrar_parecer` foi executado e devolveu `P-1001`. No passo seguinte, o agente — sem registro de escrita concluída — invocou `registrar_parecer` novamente.

A execução foi preservada pela **chave de idempotência** (Aula 05, [nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md), §8): a segunda chamada devolveu `{"id": "P-1001", "ja_existia": true}` e o agente prosseguiu.

Duas conclusões. A primeira: o prompt de *compaction* precisa exigir a preservação das ações de escrita. A segunda, de alcance mais geral: **as salvaguardas se cobrem mutuamente**. Nenhuma é suficiente isoladamente, e é por essa razão que a arquitetura é composta por camadas em vez de um mecanismo único.

## Fontes e leituras

PACKER, C. et al. **MemGPT**: towards LLMs as operating systems. arXiv:2310.08560, 2023. (A gestão da janela por um agente que decide o que promover e o que arquivar.)

PARK, J. S. et al. **Generative agents**: interactive simulacra of human behavior. arXiv:2304.03442, 2023. (A política de escrita adotada ali corresponde à §1.1, com uma etapa de reflexão periódica.)

ANTHROPIC. **Effective context engineering for AI agents**. 2025.

**Material da disciplina.** Aula 01, [nota 03 §3](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — a fronteira leitura/escrita. Aula 05, [nota 01 §6](../../aula-08-memoria/notas-de-aula/01-o-agente-e-o-estado.md) — restrição por arquitetura; [nota 02 §1 e §8](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — orçamento e idempotência; [nota 03 §1](../../aula-05-arquitetura-de-agentes/notas-de-aula/03-context-engineering-dinamica.md) — a aritmética do acumulado, que motiva a §5. Aula 07, [nota 02 §2](../../aula-07-rag-e-documentos/notas-de-aula/02-o-que-o-embedding-nao-ve.md) — a cegueira que impede deduplicar regras por similaridade.
