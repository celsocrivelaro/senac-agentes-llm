# IA Aplicada com LLMs — Aula 03: Prompt engineering — Tool calling: quando o prompt vira schema

## Introdução

A Aula 02 terminou uma seção com uma frase pedindo para ser guardada:

> *"Quando o modelo 'chama uma função', ele está gerando um JSON restrito pelo schema dos parâmetros dessa função."*
> — Aula 02, [nota 02, §7.5](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md)

Esta nota paga essa promessa. E ela fecha o arco de toda a aula 03: começamos com prompt como **texto livre** — instruções em português, exemplos, "pense passo a passo" — e terminamos com o prompt virando **contrato tipado**, verificável por máquina.

A mudança é maior do que parece. Enquanto a saída do modelo é texto, o seu programa **interpreta**. Quando ela é uma chamada de função validada contra um schema, o seu programa **executa**. É a diferença entre um gerador de texto e um componente que participa do sistema.

E é o que transforma este curso no que o nome dele promete: até aqui o aluno construiu programas que leem, processam e geram. A partir daqui, eles **decidem e agem**.

> **Pré-requisitos:** Aula 02, [nota 02 §7](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md) — saída estruturada e decodificação restrita. Esta nota é a continuação direta. E Aula 01, [nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — o laço do agente, que aqui roda pela primeira vez em código seu.

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Explicar** por que tool calling é o mesmo mecanismo da saída estruturada.
- **Descrever os quatro tempos** de uma chamada de ferramenta, dizendo quem executa cada um.
- **Declarar** uma ferramenta com nome, descrição e schema de parâmetros.
- **Ler** `tool_calls` na resposta e **devolver** o resultado com `role: "tool"`.
- **Implementar** uma volta completa do laço do agente, e **reconhecê-lo como o padrão ReAct**.
- **Explicar** por que o formato *Thought / Action / Observation* existe, e por que você não precisa mais escrevê-lo no prompt.
- **Projetar** uma ferramenta boa: granularidade, descrição, erros legíveis pelo modelo.
- **Reconhecer** que a descrição da ferramenta **é prompt** — e o único que o modelo lê para decidir.

---

## Desenvolvimento teórico

### 1. É o mesmo mecanismo — literalmente

Compare o que você escreveu na Aula 02 com o que vai escrever agora:

```python
# Aula 02 — forçar a saída num formato
response_format={
    "type": "json_schema",
    "json_schema": {"name": "cadastro", "schema": SCHEMA, "strict": True},
}
```

```python
# Aula 03 — oferecer uma ferramenta
tools=[{
    "type": "function",
    "function": {
        "name": "consultar_pedido",
        "description": "Consulta a situação de um pedido pelo número.",
        "parameters": SCHEMA,          # ← o MESMO tipo de objeto
    },
}]
```

O campo `parameters` **é um JSON Schema** — o mesmo formato, com a mesma decodificação restrita por trás (Aula 02, nota 02 §7.3). Os tokens que quebrariam o schema têm probabilidade **zero**, não "baixa".

A diferença não está no mecanismo. Está em **o que você faz com o resultado**:

| | Saída estruturada | Tool calling |
|---|---|---|
| O modelo produz | um JSON com os dados | um JSON com **nome da função + argumentos** |
| O seu código | grava, exibe, valida | **executa a função** |
| `finish_reason` | `"stop"` | `"tool_calls"` |

> **Sem saída estruturada confiável não existe agente confiável.** Agora dá para ver por quê: a chamada de ferramenta *é* saída estruturada. Se o formato não fosse garantido, você estaria executando funções a partir de texto adivinhado.

---

### 2. Os quatro tempos

O ponto que mais confunde quem está começando: **o modelo não executa nada.** Ele nem sabe que as suas funções existem como código — só viu uma descrição delas.

```
   ┌────────────────────────────────────────────────────────────────┐
   │ 1. VOCÊ declara                                                │
   │    tools=[...] na chamada, junto com as mensagens              │
   └───────────────────────────┬────────────────────────────────────┘
                               ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ 2. O MODELO decide                                             │
   │    devolve finish_reason="tool_calls" e, em message.tool_calls,│
   │    o nome da função e os argumentos em JSON                    │
   └───────────────────────────┬────────────────────────────────────┘
                               ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ 3. O SEU CÓDIGO executa                                        │
   │    valida os argumentos, chama a função Python, pega o retorno │
   └───────────────────────────┬────────────────────────────────────┘
                               ▼
   ┌────────────────────────────────────────────────────────────────┐
   │ 4. VOCÊ devolve                                                │
   │    acrescenta o resultado como mensagem role="tool" e chama    │
   │    o modelo DE NOVO, agora com o resultado no contexto         │
   └────────────────────────────────────────────────────────────────┘
                               │
                    volta ao passo 2, até o modelo
                    responder sem pedir ferramenta
```

Dois tempos são seus (1 e 3–4), um é do modelo (2). **O modelo decide; o seu código age.** Toda a segurança do sistema mora nessa separação: quem controla o passo 3 é você, e o modelo nunca toca em nada por conta própria.

---

### 3. Declarando a ferramenta

```python
def consultar_pedido(numero: str) -> dict:
    """A função Python de verdade. O modelo nunca a executa."""
    pedido = BANCO.get(numero)
    if pedido is None:
        return {"erro": f"pedido {numero} não encontrado"}
    return {"numero": numero, "situacao": pedido["situacao"],
            "previsao": pedido["previsao"]}


FERRAMENTAS = {"consultar_pedido": consultar_pedido}

DECLARACOES = [{
    "type": "function",
    "function": {
        "name": "consultar_pedido",
        "description": (
            "Consulta a situação atual e a previsão de entrega de um pedido. "
            "Use quando o cliente citar um número de pedido e a resposta "
            "depender do status real. Não use para dúvidas gerais."
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "numero": {
                    "type": "string",
                    "pattern": "^[0-9]+$",
                    "description": "Número do pedido, somente dígitos.",
                },
            },
            "required": ["numero"],
            "additionalProperties": False,
        },
    },
}]
```

Três detalhes que não são decoração:

- **`pattern` no schema** — a mesma restrição que você usou na Aula 02 para o CPF. O modelo não consegue emitir `"pedido 48219"` no campo; só dígitos passam;
- **`additionalProperties: False`** — impede argumento inventado;
- **a função devolve `{"erro": ...}` em vez de levantar exceção** — §7.

---

### 4. A descrição da ferramenta é prompt

Este é o ponto que fecha o arco da aula, e é onde mais gente erra.

O modelo **não vê o seu código**. Ele não sabe o que `consultar_pedido` faz, não conhece o banco, não sabe o que acontece depois. Tudo o que ele tem para decidir **se e quando** chamar é:

1. o **nome** da função;
2. a **descrição**;
3. a **descrição de cada parâmetro**.

Ou seja: a descrição de ferramenta é um **prompt** — sujeito a tudo o que a nota 01 ensinou. Cada frase precisa eliminar uma possibilidade.

| Descrição | Problema |
|---|---|
| `"Consulta pedido"` | não diz **quando** usar; o modelo chama demais ou de menos |
| `"Esta ferramenta consulta de forma eficiente e confiável..."` | adjetivos não restringem nada (nota 01, §1) |
| `"Consulta a situação... Use quando o cliente citar um número de pedido e a resposta depender do status real. Não use para dúvidas gerais."` | ✅ diz o que faz, **quando usar** e **quando não** |

O *"não use para..."* é o **exemplo negativo** da nota 01, §4, aplicado a ferramenta. Sem ele, o sintoma clássico é o agente consultando o banco para responder *"vocês entregam no sábado?"*.

> **A frase que resume a aula inteira:** o prompt não desapareceu — ele foi para dentro do schema. Continua sendo texto em português que decide comportamento, só que agora **num campo com contrato**.

---

### 5. Lendo a chamada e devolvendo o resultado

```python
import json

mensagens = [
    {"role": "system", "content": SYSTEM},
    {"role": "user", "content": "meu pedido 48219 chegou?"},
]

resposta = client.chat.completions.create(
    model=MODELO,
    messages=mensagens,
    tools=DECLARACOES,
    temperature=0,           # decidir ferramenta é decisão: variação é defeito
)

msg = resposta.choices[0].message

if msg.tool_calls:
    mensagens.append(msg)                      # 1x: a fala do assistente

    for chamada in msg.tool_calls:             # pode vir mais de uma
        nome = chamada.function.name
        argumentos = json.loads(chamada.function.arguments)

        print(f"[ferramenta] {nome}({argumentos})")

        funcao = FERRAMENTAS.get(nome)
        if funcao is None:                     # o modelo inventou o nome
            resultado = {"erro": f"ferramenta desconhecida: {nome}"}
        else:
            resultado = funcao(**argumentos)

        mensagens.append({                     # 1x por chamada
            "role": "tool",
            "tool_call_id": chamada.id,        # amarra o resultado à chamada
            "name": nome,
            "content": json.dumps(resultado, ensure_ascii=False),
        })

    final = client.chat.completions.create(
        model=MODELO, messages=mensagens, tools=DECLARACOES, temperature=0,
    )
    print(final.choices[0].message.content)
```

Quatro armadilhas, todas comuns na primeira implementação:

1. **acrescentar `msg` uma vez, e uma mensagem `tool` por chamada.** Trocar a ordem ou repetir a mensagem do assistente costuma render erro 400 do provedor;
2. **`tool_call_id` é obrigatório** — é o que amarra cada resultado à chamada correspondente quando há várias;
3. **`content` da mensagem `tool` é string.** Serialize com `json.dumps`, não passe o dicionário;
4. **`temperature=0`** — a Aula 02 (nota 02, §9) já dizia: escolher ferramenta é a decisão mais crítica, porque um erro aqui não gera texto ruim, gera **ação errada no mundo**.

---

### 6. O laço

Uma volta raramente basta: o modelo pode precisar de duas ferramentas, ou usar o resultado da primeira para decidir a segunda. Isso é o laço da Aula 01, nota 03 — agora em código seu:

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

        if not msg.tool_calls:                 # o modelo respondeu: terminou
            return msg.content

        mensagens.append(msg)
        for chamada in msg.tool_calls:
            resultado = executar(chamada)      # o bloco da §5
            mensagens.append({
                "role": "tool",
                "tool_call_id": chamada.id,
                "name": chamada.function.name,
                "content": json.dumps(resultado, ensure_ascii=False),
            })

    raise RuntimeError(f"não concluiu em {max_passos} passos")
```

**A condição de parada é o modelo devolver uma resposta sem `tool_calls`.** E o `max_passos` não é paranoia: modelos entram em laço — chamam a mesma ferramenta repetidamente, ou alternam entre duas sem convergir. Sem teto, isso é uma conta aberta rodando sozinha.

E lembre que **cada volta reenvia todo o histórico**: o contexto vai inchando a cada passo, com o raciocínio e os resultados das voltas anteriores. É onde a nota 02 desta aula reaparece — num agente, a janela é o recurso que mais rápido se esgota.

#### 6.1 Este laço tem nome: **ReAct**

O que você acabou de escrever não é um arranjo improvisado. É o padrão **ReAct** — de *Reasoning + Acting* (Yao et al., 2022) —, e você já o encontrou na Aula 01, [nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md), onde ele aparece como o padrão de agente mais simples e mais usado.

A correspondência é direta:

| No paper | No seu código |
|---|---|
| **Thought** (raciocínio) | o modelo decide qual ferramenta chamar → `finish_reason="tool_calls"` |
| **Action** (ação) | `FERRAMENTAS[nome](**argumentos)` — o seu código executa |
| **Observation** (observação) | a mensagem `role: "tool"` que você acrescenta com o resultado |
| o laço | o `for passo in range(max_passos)` |

A ideia central do paper é que **raciocinar e agir se ajudam mutuamente**: o raciocínio decide qual ação tomar, e a observação resultante corrige o raciocínio seguinte. É por isso que, no `04-tool-calling.py`, o modelo consegue consultar o pedido e **depois** decidir calcular o prazo com a data que recebeu — a segunda decisão depende do resultado da primeira.

#### 6.2 Como era antes do tool calling — e por que isso importa

Aqui está a parte que explica um formato que você vai encontrar em muito código e material por aí.

O ReAct foi proposto em 2022, **antes de existir tool calling nativo**. Não havia campo `tools` na API, nem `finish_reason="tool_calls"`. O laço era emulado **no texto**: você escrevia um prompt pedindo que o modelo respondesse num formato fixo, e fazia o *parsing* da saída.

```text
Responda sempre neste formato:

Thought: <seu raciocínio>
Action: <nome_da_ferramenta>[<argumento>]
Observation: <preenchido pelo sistema>
... (repita quantas vezes precisar)
Thought: já sei a resposta
Final Answer: <a resposta>
```

E o programa procurava a linha `Action:` com uma expressão regular, extraía o nome e o argumento, executava, e colava o resultado numa linha `Observation:` antes de chamar o modelo de novo.

**Funcionava — e quebrava do jeito que a Aula 02 previu.** O modelo escrevia `Action : consultar_pedido` com um espaço a mais, ou `Ação:` em português, ou explicava o que ia fazer antes da linha, ou esquecia os colchetes. Cada uma dessas variações quebrava o regex, e a correção era… escrever um prompt melhor, e torcer.

**O que mudou:** o `tools` faz exatamente a mesma coisa, com **schema em vez de texto**. Em vez de *pedir* que o modelo escreva `Action:` no formato certo, a decodificação restrita **impede** que ele escreva outra coisa (Aula 02, nota 02 §7.3).

> É a mesma passagem que esta nota vem descrevendo desde a introdução — de **texto livre** para **contrato tipado** —, agora com uma data. O ReAct não mudou; o que mudou foi que a estrutura saiu do prompt e foi para o schema.

Duas consequências práticas:

- **você não precisa mais escrever "Thought/Action/Observation" no prompt.** Se encontrar um tutorial mandando fazer isso com um modelo que tem tool calling, é material anterior a 2023 — ou alguém copiando material anterior a 2023;
- **mas o vocabulário continua valendo.** "Trajetória", "passo", "observação" são os termos com que a área discute agentes, e você vai lê-los na aula de agentes, nos frameworks e nos papers.

---

### 7. Projetar a ferramenta é o trabalho de verdade

A Aula 01 (nota 03, §3) já avisava: as ferramentas são a **maior superfície de falha** do sistema. Agora que você implementou uma, os critérios ficam concretos:

| Critério | Regra |
|---|---|
| **Nome** | verbo + objeto, sem ambiguidade: `consultar_pedido`, não `pedido` ou `get_data` |
| **Granularidade** | poucas ferramentas boas > 40 mal definidas. Se duas fazem quase a mesma coisa, o modelo erra a escolha |
| **Descrição** | o que faz, **quando usar**, **quando não usar** (§4) |
| **Erros legíveis** | devolva `{"erro": "pedido 48219 não encontrado"}`, **nunca** levante exceção |
| **Retorno enxuto** | só o que o modelo precisa. 200 registros viram 8 mil tokens **em toda volta** (nota 02, §3) |
| **Leitura × escrita** | ferramenta que escreve precisa de confirmação, teto e registro (nota 03, §9) |

O item dos **erros legíveis** é o mais contraintuitivo para quem vem de programação comum. Ali, uma exceção é o certo: falhe cedo e ruidosamente. Aqui, a exceção mata o laço; já um erro **devolvido como texto** vira **informação no contexto**, e o modelo se recupera sozinho — pede o número de novo, tenta outra ferramenta, ou explica ao usuário. A ferramenta conversa com um leitor de texto, não com um `try/except`.

---

### 8. `tool_choice`: quem decide se usa

Além de oferecer as ferramentas, dá para controlar a decisão:

| Valor | Comportamento |
|---|---|
| `"auto"` | o modelo decide (padrão) |
| `"none"` | proíbe ferramentas nesta chamada |
| exigir chamada | força usar alguma — o nome do valor **varia por provedor** |
| função específica | força aquela função |

Note o "varia por provedor": é o **contrato de API** da Aula 02 (nota 02, §2) outra vez. Confira na documentação em vez de supor.

Um uso comum: `tool_choice="none"` na **última** volta do laço, para garantir que o modelo redija a resposta final em vez de pedir mais uma ferramenta.

---

### 9. O que fica para a aula de agentes

Você tem uma volta funcionando. Falta o que transforma isso em sistema:

- **estado e memória** entre execuções;
- **múltiplas ferramentas** e a escolha entre elas;
- **outros padrões de arquitetura** — o ReAct você já tem (§6.1); faltam planejamento, reflexão, roteador, orquestrador-trabalhador e humano no laço;
- **MCP** — o padrão que conecta agente a ferramentas de terceiros sem você escrever cada integração;
- **confiabilidade**: retry, detecção de laço, orçamento de passos, checkpoint;
- **context engineering dinâmica** — a nota 02 desta aula, agora ao longo da trajetória.

Mas o mecanismo você já viu inteiro. O resto é engenharia em volta destes quatro tempos.

---

## Exemplos

### Exemplo 1 — A mesma pergunta, com e sem ferramenta

```python
PERGUNTA = "O pedido 48219 já saiu para entrega?"

# Sem ferramenta
sem = client.chat.completions.create(
    model=MODELO, messages=[{"role": "user", "content": PERGUNTA}])
print(sem.choices[0].message.content)
# -> "Não tenho acesso a informações de pedidos..." (bom)
# -> ou uma previsão inventada, com fluência total (péssimo)

# Com ferramenta
com = client.chat.completions.create(
    model=MODELO, messages=[{"role": "user", "content": PERGUNTA}],
    tools=DECLARACOES, temperature=0)
print(com.choices[0].finish_reason)      # -> "tool_calls"
print(com.choices[0].message.tool_calls[0].function)
# -> name='consultar_pedido' arguments='{"numero": "48219"}'
```

O que observar: **o modelo não ficou mais inteligente.** Ele continua sem saber nada sobre o pedido. O que mudou foi que agora existe um caminho para a informação chegar — e o modelo reconheceu que devia usá-lo. É a ideia da Aula 01, nota 03: *pare de tentar fazer o LLM saber coisas; faça-o decidir coisas.*

### Exemplo 2 — Um erro de ferramenta que o modelo resolve

```python
def consultar_pedido(numero: str) -> dict:
    if numero not in BANCO:
        return {"erro": f"pedido {numero} não encontrado",
                "dica": "confirme o número com o cliente"}
    ...
```

Trajetória com um pedido inexistente:

```
[ferramenta] consultar_pedido({'numero': '99999'})
   -> {"erro": "pedido 99999 não encontrado", "dica": "confirme o número..."}

Resposta: "Não localizei o pedido 99999 no sistema. Você poderia confirmar
o número? Ele costuma ter 5 dígitos e aparece no e-mail de confirmação."
```

O modelo **leu o erro e se recuperou** — porque o erro veio como texto no contexto. Se `consultar_pedido` tivesse levantado `KeyError`, o programa teria morrido e o cliente teria visto uma tela de erro.

Compare com a versão ruim: `return {"erro": "KeyError: '99999'"}`. Tecnicamente é um erro devolvido, mas não diz ao modelo **o que fazer**. Erro de ferramenta é uma mensagem para um leitor — escreva-a para ser lida.

---

## Exercícios resolvidos

### 1. Por que o agente não chama a ferramenta?

**Enunciado.** Um agente tem `consultar_pedido` declarada, mas ignora a ferramenta e responde *"não tenho acesso a essa informação"*. O schema está correto e o modelo suporta function calling. Liste as causas prováveis, em ordem de probabilidade.

**Resolução.**

**1. A descrição não diz quando usar.** Causa mais comum. `"Consulta pedido"` não dá critério — o modelo não sabe que aquela pergunta é caso de uso. Correção: *"Use quando o cliente citar um número de pedido e a resposta depender do status real."*

**2. O system prompt contradiz.** Instruções como *"responda apenas com base no que você sabe"* ou *"não invente informações"* às vezes são interpretadas como proibição de agir. O system prompt precisa **autorizar**: *"use as ferramentas disponíveis quando precisar de dados reais"*.

**3. O nome da função é ruim.** `get_data`, `f1`, `processar` não dizem nada. O nome é a primeira coisa que o modelo lê.

**4. Falta contexto na pergunta.** Se o usuário escreveu "e o meu pedido?" sem número, o modelo pode não ter o argumento obrigatório e desistir em vez de pedir. Correção: instruir a **perguntar** o que falta.

**5. O modelo não suporta bem.** Último da lista, não o primeiro — mas confira `capabilities` no `00-catalogo-modelos.py` da Aula 02. Modelos pequenos declaram suporte e mesmo assim erram muito na decisão.

**Como diagnosticar em vez de adivinhar:** force com `tool_choice` na função específica. Se com a chamada forçada o resultado sai correto, o problema é de **decisão** (causas 1–4). Se o resultado sai errado mesmo assim, é do schema ou do modelo.

### 2. Uma ferramenta ou quatro?

**Enunciado.** Você precisa dar ao agente acesso a: consultar pedido, consultar cliente, listar pedidos de um cliente e cancelar pedido. Um colega propõe uma ferramenta única `consultar_dados(tipo, parametros)`. Avalie.

**Resolução.**

**A proposta é ruim, por três motivos.**

**1. O schema deixa de proteger.** Com `parametros` genérico (um objeto livre), você perde `pattern`, `required` e `additionalProperties` — exatamente as garantias da §3. A validação volta para dentro da sua função, em tempo de execução, depois de o modelo já ter chutado.

**2. A descrição fica impossível.** Uma descrição teria que explicar quatro operações e quando usar cada uma, dentro do texto de uma. É o oposto de "cada frase elimina uma possibilidade".

**3. E o problema grave: junta leitura com escrita.** `cancelar_pedido` é **irreversível**; as outras três são inócuas. Numa ferramenta só, não dá para exigir confirmação apenas para o cancelamento, nem para conceder acesso somente-leitura a um agente que não deveria cancelar nada. É o erro que a nota 03, §9 descreve.

**A resposta certa, e ela não é "quatro":**

- **três ferramentas de leitura** — `consultar_pedido`, `consultar_cliente`, `listar_pedidos_do_cliente` — separadas, cada uma com schema restrito;
- **`cancelar_pedido` tratada como categoria diferente**: confirmação humana obrigatória, registro em log, e — se possível — fora do conjunto de ferramentas deste agente.

**A ressalva que evita o erro oposto:** "separar" não quer dizer "multiplicar". Se amanhã surgirem 40 ferramentas, o modelo passa a errar a escolha, e as declarações ocupam janela (elas vão em **toda** chamada). O critério não é o número — é: **cada ferramenta faz uma coisa nomeável, com um schema que a protege, e o conjunto cabe numa descrição que o modelo consegue distinguir.**

---

## Síntese

- **Tool calling é saída estruturada.** O `parameters` da função é um **JSON Schema**, com a mesma decodificação restrita da Aula 02. A diferença é o que você faz com o resultado: em vez de gravar, **executar**.
- **Quatro tempos:** você declara → **o modelo decide** → **o seu código executa** → você devolve o resultado como `role: "tool"` e chama de novo.
- **O modelo nunca executa nada.** Ele não vê o seu código. Toda a segurança mora nessa separação.
- **A descrição da ferramenta é prompt** — e o único texto que o modelo lê para decidir. Diga o que faz, **quando usar** e **quando não usar**.
- `finish_reason="tool_calls"`; `tool_call_id` é obrigatório; `content` da mensagem `tool` é **string**; acrescente a fala do assistente **uma vez** e uma mensagem `tool` **por chamada**.
- **`temperature=0`**: escolher ferramenta é a decisão mais crítica — erro aqui vira **ação errada no mundo**, não texto ruim.
- **O laço** termina quando o modelo responde sem `tool_calls`. **Teto de passos** é obrigatório: modelos entram em laço, e cada volta reenvia todo o histórico.
- **Este laço é o ReAct** (Yao et al., 2022): *Thought* = a decisão do modelo, *Action* = o seu código executando, *Observation* = a mensagem `role: "tool"`. Antes do tool calling nativo, isso era emulado **em texto** e quebrava no regex; hoje o schema garante a estrutura. **O padrão não mudou — ele saiu do prompt e foi para o schema.**
- **Erro de ferramenta se devolve, não se levanta.** Exceção mata o laço; erro no contexto o modelo lê e se recupera. E escreva a mensagem de erro **para ser lida**.
- **Retorno enxuto**: a saída de ferramenta é a fonte de contexto que mais cresce num agente.
- **Separe leitura de escrita.** Ferramenta que escreve precisa de confirmação, teto e registro.
- O prompt não desapareceu — **ele foi para dentro do schema**.

---

## Fontes e leituras

**Papers**

- Yao, S. et al. — *ReAct: Synergizing Reasoning and Acting in Language Models* (2022). [arxiv.org/abs/2210.03629](https://arxiv.org/abs/2210.03629) — o padrão que você implementou na §6, e a formulação original em texto que a §6.2 descreve.

**Engenharia**

- Documentação de *function calling* da Mistral: [docs.mistral.ai](https://docs.mistral.ai) — o formato exato de `tools`, `tool_calls` e `tool_choice`.
- OpenAI — *Function calling*: [platform.openai.com/docs/guides/function-calling](https://platform.openai.com/docs/guides/function-calling) — a referência mais detalhada do mesmo formato.
- Anthropic — *Building Effective Agents* — projeto de ferramentas e a discussão workflow × agente, leitura recomendada antes da próxima aula.
- Model Context Protocol: [modelcontextprotocol.io](https://modelcontextprotocol.io) — o padrão para conectar agentes a ferramentas de terceiros, tema próprio mais à frente.

**Nesta disciplina**

- [Aula 01 — nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — o laço do agente e o projeto de ferramentas. Esta nota é a implementação do que lá foi apresentado.
- [Aula 02 — nota 02, §7](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md) — saída estruturada e decodificação restrita: o mecanismo desta nota.
- [Nota 01 desta aula](01-anatomia-e-tecnicas.md) — a descrição da ferramenta é prompt, e vale tudo o que está lá.
- [Nota 02 desta aula](02-context-engineering.md) — a saída de ferramenta como fonte de contexto.
- [Nota 03 desta aula](03-prompt-como-codigo.md) — a descrição da ferramenta é prompt: entra no mesmo versionamento e na mesma suíte de regressão.
- [Exercício da aula](../exercicios/exercicio_03.md) — um atendimento completo: o programa pergunta ao cliente, consulta o pedido, raciocina sobre os dados, abre o chamado com confirmação e lê de volta o que gravou.
