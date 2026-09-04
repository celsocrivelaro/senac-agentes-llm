# IA Aplicada com LLMs — Aula 04: Arquitetura de agentes — Padrões de arquitetura

## Introdução

A última nota da Aula 03 terminou com uma lista. Depois de montar o laço de tool calling, ela dizia: *você tem uma volta funcionando; falta o que transforma isso em sistema* — e enumerava estado e memória, múltiplas ferramentas, outros padrões de arquitetura, MCP, confiabilidade e context engineering dinâmica.

Esta aula fecha três desses itens. Esta nota fecha o primeiro: **os padrões**.

Mas antes de qualquer padrão, uma pergunta desconfortável. Você tem um laço que funciona. Por que ele não está pronto para rodar sozinho?

Três respostas curtas, e cada uma é uma nota desta aula:

1. **Ele não sabe parar.** O `max_passos=6` é um teto arbitrário, não um orçamento.
2. **Ele não sabe falhar.** Uma ferramenta que estoura derruba o programa inteiro.
3. **Ele não sabe esquecer.** A cada volta o histórico cresce, e nada nunca sai.

E existe uma quarta pergunta, anterior a todas — a que esta nota responde:

> **Ele precisava ser um laço?**

Boa parte dos sistemas que as pessoas chamam de "agente" resolveria melhor o problema com o caminho escrito no código. Reconhecer isso não é modéstia: é a diferença entre um sistema que você consegue testar e um que você só consegue torcer para dar certo.

> **Pré-requisitos:** da Aula 03, a [nota 04 §6 e §9](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) (o laço ReAct e o que ficou pendente). Da Aula 01, a [nota 03 §1](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) (definição de agente e o espectro de autonomia). Da Aula 02, a [nota 02 §7](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md) (saída estruturada) e a [nota 04](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md) (a conta por chamada).

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Decidir** entre workflow e agente para uma tarefa concreta, usando três perguntas operacionais em vez de intuição.
- **Reconhecer e implementar** os cinco padrões de workflow: *prompt chaining* com portão, roteador, paralelização (*sectioning* e *voting*), orquestrador-trabalhador e avaliador-otimizador.
- **Dizer, para cada padrão, quando ele não serve** — que é a metade que quase nenhum material traz.
- **Estimar o custo em chamadas** de cada padrão antes de escrever a primeira linha.
- **Identificar workflow disfarçado de agente** em código existente — inclusive no seu.

---

## Desenvolvimento teórico

### 1. O que a Aula 03 entregou

Este é o laço com que você saiu da aula passada:

```python
def rodar(pergunta: str, max_passos: int = 6) -> str:
    mensagens = [
        {"role": "system", "content": SYSTEM},
        {"role": "user", "content": pergunta},
    ]

    for passo in range(max_passos):
        r = client.chat.completions.create(
            model=MODELO, messages=mensagens,
            tools=DECLARACOES, temperature=0,
        )
        msg = r.choices[0].message

        if not msg.tool_calls:
            return msg.content

        mensagens.append(msg)
        for chamada in msg.tool_calls:
            resultado = executar(chamada)
            mensagens.append({
                "role": "tool", "tool_call_id": chamada.id,
                "name": chamada.function.name,
                "content": json.dumps(resultado, ensure_ascii=False),
            })

    raise RuntimeError(f"não concluiu em {max_passos} passos")
```

O código está **correto**. Ele é o padrão ReAct, implementa os quatro tempos e roda. O problema não é o que ele faz — é o que ele assume: que a tarefa **precisa** de um laço.

Repare no que esse `for` significa em termos de projeto. Você abriu mão de saber quantas chamadas o programa vai fazer, em que ordem, e com que argumentos. Em troca, ganhou a capacidade de resolver tarefas cuja sequência não dá para prever. **Se a sequência dava para prever, você pagou o preço sem levar a mercadoria.**

---

### 2. A linha divisória, agora como pergunta operacional

A Aula 01 (nota 03, §1.1) deu a definição:

> **Workflow**: o caminho está no código. **Agente**: o caminho é decidido em tempo de execução pelo modelo.

Definição é boa para prova e ruim para decidir às três da tarde com um requisito na mesa. A versão operacional é uma pergunta só:

> **Você consegue desenhar o fluxograma do processo antes de rodar?**

Se consegue — desenhe, e depois **escreva o fluxograma em Python**. Isso é um workflow, e ele será mais barato, mais rápido, testável e explicável quando der errado.

Se não consegue, porque o próximo passo depende do que o passo anterior descobrir e você não sabe de antemão o que ele vai descobrir, aí sim existe motivo para autonomia.

Três perguntas de apoio, quando a resposta não for óbvia:

| Pergunta | Se "sim" | Se "não" |
|---|---|---|
| **(a)** O número de passos é conhecido antes de rodar? | tende a workflow | tende a agente |
| **(b)** A ordem depende de dados que só existem em execução? | tende a agente | tende a workflow |
| **(c)** O erro é caro ou irreversível? | **reduza a autonomia** e adicione confirmação | autonomia é mais tolerável |

A pergunta (c) tem um peso próprio: ela não é sobre a natureza da tarefa, é sobre a **consequência do erro**. Uma tarefa que tecnicamente pediria um agente, mas cujo erro emite dinheiro, é uma tarefa onde você aceita um sistema pior para ter um sistema previsível.

#### 2.1 O que cada nível custa

O espectro de autonomia da Aula 01 (nota 03, §1.4) tinha seis níveis. O que aquela nota não detalhou foi o **preço** de subir:

| Nível | Chamadas por tarefa | Previsível? | Testável? | Depurável quando falha? |
|---|---|---|---|---|
| Prompt único | 1 | sim | sim, trivialmente | sim |
| Chain | N fixo | sim | sim, etapa por etapa | sim |
| Roteador | 1 + o custo da rota | sim | sim, por rota | sim |
| Workflow com ferramentas | N fixo | sim | sim | sim |
| **Agente** | **desconhecido** | **não** | só por propriedade | **só com log de trajetória** |
| Multi-agente | desconhecido × agentes | não | difícil | muito difícil |

A coluna que mais dói na prática é a última. Num workflow, quando o resultado sai errado, você sabe em qual etapa olhar. Num agente, a trajetória é diferente a cada execução, e sem log estruturado você tem uma saída errada e nenhuma pista. É por isso que a nota 03 desta aula insiste tanto em registrar o estado — e é por isso que a aula de observabilidade existe.

> **A regra da Aula 01 continua valendo, e agora com fundamento:** use a menor autonomia que resolve o problema. Não por conservadorismo — porque cada nível acima custa mais dinheiro, mais latência e, principalmente, mais tempo seu quando algo der errado.

---

### 3. Autópsia do exercício 03

O melhor exemplo de workflow disfarçado de agente que temos à mão foi escrito por você, na semana passada.

O exercício 03 pedia um atendimento em cinco etapas. Classifique cada uma:

| Etapa | O que fazia | Workflow ou agente? |
|---|---|---|
| 1. **Coleta** | conversar até ter número do pedido e o problema | **workflow** — a condição de parada é "tenho os dois campos", e isso é um `if` |
| 2. **Consulta** | `consultar_pedido(id)` | **workflow** — sempre acontece, sempre nessa posição |
| 3. **Raciocínio** | comparar previsão × hoje, cliente × sistema, concluir | **decisão do modelo** — mas de **uma** decisão, não de uma sequência |
| 4. **Ação** | `abrir_chamado(...)`, com confirmação | **workflow condicional** — roda se a etapa 3 concluiu que cabe chamado |
| 5. **Consulta final** | `consultar_chamado(id)` | **workflow** — sempre, e sempre por último |

Quatro das cinco etapas estavam no código. A sequência inteira estava desenhada no enunciado — literalmente numerada de 1 a 5. **Aquilo era um workflow com dois momentos de decisão**, e você o escreveu como agente.

Isso foi um erro? Não: o objetivo pedagógico daquele exercício era o **mecanismo** — declarar ferramenta, ler a chamada, devolver o resultado. Para aprender o mecanismo, o laço é o veículo certo.

Mas se aquele programa fosse para produção, a versão workflow seria melhor em todos os eixos que importam: menos chamadas, sem risco de o modelo pular a consulta final, sem risco de abrir dois chamados, e testável etapa por etapa. A única coisa que ela perderia é a capacidade de lidar com um atendimento que fugisse do roteiro — e é exatamente aí que se decide.

> **O padrão que você deve reconhecer:** quando o enunciado do problema vem numerado, o código também deveria vir.

---

### 4. Os cinco padrões

A taxonomia a seguir segue *Building Effective Agents* (Anthropic, 2024), que é a formulação mais usada hoje. Cada padrão aparece no mesmo formato: **o que é · o esqueleto · quando usar · quando não usar**.

Todos os cinco são **workflows**: o caminho está no seu código. O agente propriamente dito é assunto da [nota 02](02-o-agente-e-o-estado.md).

---

#### 4.1 Prompt chaining com portão

**O que é.** Chamadas em sequência fixa, onde a saída de uma alimenta a próxima — com **validação entre elas**.

```
   entrada ──> [ LLM 1 ] ──> portão ──> [ LLM 2 ] ──> saída
                              │
                              └── falhou ──> tratamento (não segue adiante)
```

O *chaining* em si você já viu na Aula 03 (nota 01) como técnica de prompt: decompor em vez de pedir tudo de uma vez. O que muda aqui é o **portão** — e ele é a diferença entre uma cadeia e uma cadeia que serve para produção.

```python
def extrair_e_formatar(texto: str) -> dict:
    bruto = chamar(PROMPT_EXTRACAO, texto, schema=SCHEMA_EXTRACAO)

    # O PORTÃO — código, não modelo.
    if not bruto.get("valor") or bruto["valor"] <= 0:
        raise ValorInvalido(bruto)          # a etapa 2 não roda
    if bruto["data"] > date.today():
        raise DataFutura(bruto)

    return chamar(PROMPT_PARECER, bruto, schema=SCHEMA_PARECER)
```

O portão é **código determinístico**. A tentação é usar o modelo para validar a saída do modelo, e às vezes isso é necessário (é o padrão 4.5) — mas quando a regra é expressável em Python, ela vai em Python. Sai mais barato, é mais confiável e não alucina.

**Quando usar:** a tarefa tem etapas conhecidas e cada uma fica melhor sozinha do que junto — extrair → validar → formatar, traduzir → revisar, resumir → classificar.

**Quando não usar:** quando as etapas não são realmente independentes e você está partindo algo que o modelo resolveria melhor de uma vez — cada elo é uma chance de perder informação. E quando o número de etapas depende da entrada: aí não é *chain*, é agente.

---

#### 4.2 Roteador

**O que é.** O modelo **classifica**; o seu código **despacha**.

```
                    ┌──> rota A: código puro (sem LLM)
   entrada ──> [ classificador ] ──> rota B: outro prompt / outro modelo
                    │              └──> rota C: humano
                    └──> nenhuma das anteriores ──> rota padrão
```

Tecnicamente, um roteador é **saída estruturada com um enum** — exatamente o mecanismo da Aula 02 (nota 02, §7), agora decidindo o fluxo do programa:

```python
ROTAS = ["dentro_da_politica", "ambiguo", "acima_da_alcada", "nenhuma"]

SCHEMA_ROTA = {
    "type": "object",
    "properties": {
        "rota": {"type": "string", "enum": ROTAS},
        "justificativa": {"type": "string"},
    },
    "required": ["rota", "justificativa"],
}

def processar(item):
    r = classificar(item, SCHEMA_ROTA)          # 1 chamada, barata
    if r["rota"] == "dentro_da_politica":
        return aprovar_por_regra(item)          # ZERO chamadas de LLM
    if r["rota"] == "ambiguo":
        return agente_analisa(item)             # caro, e raro
    if r["rota"] == "acima_da_alcada":
        return encaminhar_humano(item, r["justificativa"])
    return fila_de_revisao(item)                # "nenhuma": não invente
```

Dois pontos de engenharia que separam um roteador bom de um ruim:

**A rota `nenhuma` é obrigatória.** Sem ela, o modelo é forçado a escolher entre opções que não servem, e ele escolhe — com confiança. Uma rota de escape transforma um erro silencioso numa fila de revisão.

**A rota mais valiosa costuma ser a que não chama o modelo.** No exemplo acima, `aprovar_por_regra` é código puro. Se 70% dos itens caem nela, você acabou de reduzir o custo do sistema em 70% sem perder nada — e essa é, de longe, a otimização mais rentável desta aula.

**Quando usar:** entradas heterogêneas com tratamentos distintos; triagem; escolher modelo pequeno para o caso fácil e grande para o difícil.

**Quando não usar:** quando as rotas fazem quase a mesma coisa (junte-as num prompt só) ou quando a classificação é tão confiável em `if` quanto no modelo — se dá para decidir por regex e faixa de valor, decida por regex e faixa de valor.

---

#### 4.3 Paralelização: duas coisas com o mesmo nome

Aqui mora uma confusão que vale desfazer, porque as duas formas resolvem problemas opostos.

**Sectioning — subtarefas independentes, ao mesmo tempo.**

```
                 ┌──> [ LLM: seção 1 ] ──┐
   entrada ──────┼──> [ LLM: seção 2 ] ──┼──> juntar ──> saída
                 └──> [ LLM: seção 3 ] ──┘
```

Você **conhece** as seções: elas estão no seu código. Um relatório com quatro capítulos, uma nota fiscal com N itens, um documento avaliado sob três critérios distintos. Ganha latência (roda em paralelo) e qualidade (cada chamada tem uma tarefa só).

**Voting — a mesma tarefa, N vezes, decisão por maioria.**

```
                 ┌──> [ LLM ] ──┐
   entrada ──────┼──> [ LLM ] ──┼──> voto majoritário ──> saída
                 └──> [ LLM ] ──┘
```

Isto é a **self-consistency** da Aula 03 (nota 01) promovida de técnica de prompt a padrão de arquitetura. A diferença é onde ela mora: lá, você passava `n=5` na chamada e o provedor gerava cinco amostras do mesmo prompt; aqui, **você** orquestra N chamadas, e por isso pode variar o prompt, o modelo ou a temperatura entre elas.

```python
def votar(item, n=3):
    vereditos = [classificar(item) for _ in range(n)]   # ou em paralelo
    contagem = Counter(v["rota"] for v in vereditos)
    rota, votos = contagem.most_common(1)[0]
    if votos < n // 2 + 1 or votos == n - votos:
        return {"rota": "nenhuma", "motivo": "sem maioria"}  # divergência é sinal
    return {"rota": rota, "confianca": votos / n}
```

O detalhe que quase todo mundo joga fora: **a divergência é informação**. Três vereditos diferentes não significam "escolha um" — significam que o caso é difícil, e provavelmente é caso de humano. Descartar isso é jogar fora o único sinal de incerteza barato que você tem.

**Quando usar sectioning:** a tarefa se decompõe naturalmente e as partes não dependem umas das outras; latência importa.
**Quando usar voting:** o custo do erro é maior que o custo de 3 chamadas, e a tarefa tem resposta discreta (classificar, aprovar/reprovar, escolher).

**Quando não usar:** *sectioning*, quando as partes dependem umas das outras — você vai concatenar pedaços que se contradizem. *Voting*, em tarefa aberta ("escreva um resumo"): não existe maioria entre três textos diferentes, e você acabou de triplicar a conta para nada.

---

#### 4.4 Orquestrador-trabalhador

**O que é.** Um modelo **decide quais são as subtarefas**; trabalhadores as executam; um sintetizador junta.

```
                    ┌──> [ trabalhador ] ──┐
   entrada ──> [ orquestrador ] ──> ... ───┼──> [ sintetizador ] ──> saída
                    └──> [ trabalhador ] ──┘
                     ↑
              decide quantas e quais
```

A diferença em relação ao *sectioning* é **uma só, e é toda a diferença**: no *sectioning*, as seções estão no seu código; aqui, elas são decididas em execução. Você não sabe se serão duas ou onze.

```python
def processar_lote(lote: list) -> dict:
    plano = chamar(PROMPT_ORQUESTRADOR, lote, schema=SCHEMA_PLANO)
    # plano["subtarefas"] só existe agora — não estava no código

    if len(plano["subtarefas"]) > MAX_SUBTAREFAS:      # orçamento, já aqui
        raise PlanoGrandeDemais(len(plano["subtarefas"]))

    resultados = [executar_subtarefa(s) for s in plano["subtarefas"]]
    return chamar(PROMPT_SINTESE, resultados, schema=SCHEMA_PARECER)
```

Repare no `if`. Este é o **primeiro padrão da nota com autonomia real**, e por isso o primeiro que precisa de orçamento: se o orquestrador decidir que a tarefa tem 400 subtarefas, alguém precisa impedir. Um orquestrador sem teto é uma conta aberta assinada por um modelo.

**Quando usar:** a decomposição depende do conteúdo — uma mudança de código que toca um número imprevisível de arquivos, uma pesquisa que exige um número imprevisível de consultas, um lote heterogêneo.

**Quando não usar:** quando você consegue listar as subtarefas — isso é *sectioning*, é mais barato e não pode surpreender. E, na dúvida, prefira *sectioning*: a diferença de custo entre "não sei quantas seções" e "sei" é a diferença entre um sistema que você limita e um que você monitora.

---

#### 4.5 Avaliador-otimizador (reflexão)

**O que é.** Um laço curto de crítica: gerar → avaliar contra um critério → revisar.

```
   entrada ──> [ gerador ] ──> candidato ──> [ avaliador ]
                    ↑                             │
                    └───── crítica ───────────────┤ reprovou
                                                  │
                                                  └─ aprovou ──> saída
```

```python
def gerar_com_revisao(entrada, max_rodadas=3):
    critica = None
    for rodada in range(max_rodadas):
        candidato = chamar(PROMPT_GERADOR, entrada, critica=critica)
        aval = chamar(PROMPT_AVALIADOR, candidato, schema=SCHEMA_AVALIACAO)
        if aval["aprovado"]:
            return candidato, rodada + 1
        critica = aval["o_que_corrigir"]
    return candidato, max_rodadas          # devolve o melhor esforço, e AVISA
```

Duas armadilhas, e as duas aparecem sempre:

**Sem critério escrito, o avaliador vira elogio.** Peça a um modelo que "avalie se o texto está bom" e ele dirá que está bom. O critério precisa ser uma lista verificável — *cita a regra aplicada? menciona o valor exato? conclui com aprovado ou reprovado?* — e o schema da avaliação deve forçar uma resposta item a item, não uma nota geral. Um avaliador que devolve `{"nota": 8}` não serve para nada; um que devolve `{"cita_regra": false, "o_que_corrigir": "..."}` serve.

**Sem teto de rodadas, o par oscila.** Gerador e avaliador entram em desacordo estável: o avaliador pede A, o gerador entrega A e perde B, o avaliador pede B. Três rodadas costumam bastar; o importante é que o retorno **diga** que saiu por teto, e não por aprovação. Um "melhor esforço" apresentado como aprovado é pior que uma reprovação honesta.

A formalização deste padrão é o **Reflexion** (Shinn et al., 2023), que acrescenta um ingrediente: a crítica é guardada em memória e realimenta as tentativas seguintes — é o que o parâmetro `critica` faz no esqueleto acima.

**Quando usar:** existe critério claro de qualidade, a revisão realmente melhora o resultado, e o custo de 2–3 chamadas extras se paga. Casos típicos: texto que precisa cumprir requisitos formais, código que precisa passar em teste, tradução com glossário obrigatório.

**Quando não usar:** quando você não consegue escrever o critério. Se você não sabe dizer o que torna a saída boa, o avaliador também não sabe — e você vai pagar o dobro por uma segunda opinião sem lastro. Nesse caso, o avaliador certo é uma pessoa.

---

### 5. A tabela de decisão

Leia de cima para baixo e **pare no primeiro que resolve**. A ordem vai do mais previsível para o menos.

| Padrão | Resolve | Custo em chamadas | Quando **não** usar |
|---|---|---|---|
| **Prompt único** | tarefa fechada | 1 | quando a saída precisa de validação que só outra chamada dá |
| **Chaining + portão** | etapas conhecidas, melhor separadas | N fixo | etapas interdependentes; número de etapas variável |
| **Roteador** | entradas heterogêneas | 1 + rota | rotas quase iguais; classificação trivial em código |
| **Sectioning** | partes independentes conhecidas | 1 por seção (paralelo) | partes que dependem umas das outras |
| **Voting** | erro caro, resposta discreta | N (3–5) | tarefa aberta, sem resposta majoritária possível |
| **Orquestrador-trabalhador** | decomposição desconhecida | variável — **exige teto** | quando você consegue listar as subtarefas |
| **Avaliador-otimizador** | qualidade com critério explícito | 2 por rodada | sem critério escrito |
| **Agente** ([nota 02](02-o-agente-e-o-estado.md)) | sequência imprevisível, ambiente dá feedback | desconhecido — **exige orçamento** | quando o fluxograma existe |

---

### 6. Os padrões se compõem

Nenhuma regra diz que você escolhe um. Sistemas reais encaixam vários, e cada peça no seu nível:

```
   lote ──> [ ROTEADOR ] ──┬──> regra em código (a maioria dos itens)
                           ├──> [ AGENTE ]  (os ambíguos, poucos)
                           └──> humano      (os caros)
                                  │
                                  ▼
                        [ ORQUESTRADOR-TRABALHADOR ]  (montar o parecer do lote)
                                  │
                                  ▼
                        [ AVALIADOR-OTIMIZADOR ]      (revisar o texto final)
```

Este é, deliberadamente, o desenho do exercício desta aula. Repare no que ele faz: a autonomia cara fica confinada ao caminho estreito onde ela é necessária, e o volume passa por código determinístico.

> **O princípio de projeto que resume a nota:** autonomia é um recurso escasso, e o trabalho de arquitetura é gastá-la só onde ela compra alguma coisa.

---

## Exemplos

### Exemplo 1 — O mesmo problema, três arquiteturas, três contas

Problema: classificar 1.000 despesas e, para as que violam a política, escrever uma justificativa.

**(a) Agente para tudo.** Um laço por despesa, com ferramentas de consulta à política e ao histórico.

```
1.000 despesas × ~4 chamadas por trajetória = ~4.000 chamadas
```

**(b) Chain fixa.** Classificar (1 chamada) e, se violar, justificar (1 chamada). Supondo 15% de violações:

```
1.000 + 150 = 1.150 chamadas
```

**(c) Roteador com regra.** Regra em código resolve os casos claros (70%); classificação por LLM nos 30% restantes; justificativa nos que violam.

```
0 + 300 + 150 = 450 chamadas
```

De (a) para (c), quase **nove vezes** menos chamadas — e o resultado de (c) é *mais* confiável nos 70% resolvidos por regra, porque regra não alucina. O ganho não veio de um prompt melhor nem de um modelo melhor. Veio de decidir o que **não** mandar para o modelo.

### Exemplo 2 — O avaliador sem critério

Mesmo texto, mesmo modelo, dois avaliadores:

```python
# Avaliador vago
PROMPT_A = "Avalie se o parecer abaixo está bom. Responda aprovado ou reprovado."
# -> "Aprovado. O parecer está claro, bem estruturado e cobre os pontos principais."

# Avaliador com critério verificável
PROMPT_B = """Verifique o parecer item a item e responda no schema:
- cita_regra: o parecer cita o artigo específico da política?
- cita_valor: o parecer menciona o valor exato da despesa?
- conclui: termina com APROVADO ou REPROVADO explícito?
- o_que_corrigir: se algum item é falso, o que exatamente falta."""
# -> {"cita_regra": false, "cita_valor": true, "conclui": true,
#     "o_que_corrigir": "não cita qual artigo da política foi violado"}
```

O primeiro aprovou um parecer que não cita a regra. Não porque o modelo é ruim — porque **ninguém disse o que era estar bom**. A qualidade de um avaliador é a qualidade do critério que você escreveu; o modelo só executa.

---

## Exercícios resolvidos

### 1. Workflow ou agente?

> Uma empresa quer um sistema que receba currículos em PDF, extraia dados estruturados, compare com os requisitos de uma vaga e produza um parecer de aderência. Todo currículo passa pelas mesmas três etapas.

**Workflow — *chaining* com portão.** A frase final entrega o caso: *todo currículo passa pelas mesmas três etapas*. O fluxograma existe e está escrito no enunciado, o número de chamadas é conhecido (três) e nada no processo depende de uma descoberta feita durante a execução.

O portão entre a etapa 1 e a 2 é onde mora o valor: se a extração não achou nem nome nem experiência, o PDF provavelmente é uma imagem escaneada, e mandar isso para a etapa 2 produz um parecer inventado sobre um currículo que ninguém leu.

Quando isso viraria agente? Se o parecer pudesse exigir buscas adicionais decididas em execução — verificar uma certificação mencionada, procurar o repositório citado, consultar o histórico do candidato na empresa. Aí o número de passos deixa de ser conhecido.

### 2. Sectioning ou orquestrador-trabalhador?

> Um sistema resume documentos jurídicos longos. Cada documento é dividido em trechos, cada trecho é resumido em paralelo, e os resumos são juntados.

**Sectioning.** A pergunta decisiva é: *quem decide quais são as partes?* Aqui, a divisão em trechos é feita pelo seu código — por número de páginas, por seções do documento ou por contagem de tokens. Nada disso é decisão do modelo.

Isso seria orquestrador-trabalhador se você pedisse ao modelo que **lesse o documento e decidisse** quais partes merecem análise separada — "identifique as cláusulas de risco e trate cada uma como uma subtarefa". Aí o número e a natureza das subtarefas passam a existir só em execução, e você precisa de um teto.

A pergunta prática que resolve os dois casos: **você consegue escrever `for parte in partes` sem chamar o modelo antes?** Se consegue, é *sectioning*.

---

## Síntese

- O laço da Aula 03 está correto e é caro. Antes de melhorá-lo, pergunte se a tarefa precisava de um laço.
- A pergunta operacional é uma só: **você consegue desenhar o fluxograma antes de rodar?** Se sim, escreva o fluxograma em código.
- Subir no espectro de autonomia custa dinheiro, previsibilidade, testabilidade e — o mais caro — capacidade de depurar quando falhar.
- Cinco padrões de workflow cobrem quase tudo: *chaining* com portão, roteador, *sectioning*, *voting*, orquestrador-trabalhador e avaliador-otimizador.
- No roteador, a rota mais valiosa costuma ser a que **não chama o modelo**; e a rota de escape (`nenhuma`) é obrigatória.
- *Sectioning* × orquestrador-trabalhador se distinguem por quem define as subtarefas: o seu código ou o modelo. O segundo exige teto.
- *Voting* transforma divergência em sinal de incerteza — não descarte esse sinal.
- Um avaliador sem critério escrito produz elogio; um par gerador-avaliador sem teto oscila.
- Os padrões se compõem, e o bom projeto confina a autonomia ao caminho estreito onde ela paga.

---

## Fontes e leituras

- **Building Effective Agents** — Anthropic (2024). A taxonomia adotada nesta nota: workflows × agentes, e os cinco padrões. Leitura obrigatória do tema.
- **A practical guide to building agents** — OpenAI (2025). Complementa com orquestração, *guardrails* e transferência entre agentes.
- **ReAct: Synergizing Reasoning and Acting in Language Models** — Yao et al. (2022). Origem do laço que você já implementou.
- **Reflexion: Language Agents with Verbal Reinforcement Learning** — Shinn et al. (2023). A formalização do avaliador-otimizador com memória de crítica.
- Aula 01, [nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — definição de agente, espectro de autonomia, projeto de ferramentas e a tabela de falhas.
- Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — tool calling e o laço ReAct em código.
