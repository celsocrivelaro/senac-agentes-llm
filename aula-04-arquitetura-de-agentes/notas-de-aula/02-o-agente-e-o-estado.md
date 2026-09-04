# IA Aplicada com LLMs — Aula 04: Arquitetura de agentes — O agente e o estado

## Introdução

A [nota anterior](01-padroes-de-arquitetura.md) gastou cinco padrões defendendo que você quase sempre deveria escrever um workflow. Esta nota trata do caso em que você não deve.

Quando a sequência de passos realmente não pode ser prevista, o agente é a resposta certa — e aí o problema deixa de ser *escolher a arquitetura* e passa a ser *conseguir controlar o que você escolheu*.

Esta é a nota mais curta da aula e a mais importante, porque ela contém uma única mudança de código. Uma mudança pequena, aparentemente burocrática, e da qual dependem **todas** as salvaguardas das notas 03 e 04:

> **A lista de mensagens não pode ser o estado do agente.**

Enquanto ela for, você não consegue impor orçamento, não consegue detectar laço, não consegue salvar checkpoint e não consegue comprimir contexto. Não porque seja difícil — porque a informação necessária para fazer qualquer uma dessas coisas simplesmente não está lá.

> **Pré-requisitos:** a [nota 01](01-padroes-de-arquitetura.md) desta aula. Da Aula 03, a [nota 04 §5 e §6](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) (o laço e o formato das mensagens). Da Aula 01, a [nota 03 §1.2 e §3](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) (os quatro componentes do agente e o projeto de ferramentas).

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Reconhecer** as três condições que justificam um agente — inclusive a que quase nunca é dita.
- **Enumerar** as perguntas que a lista de mensagens não responde, e por que cada uma delas é necessária.
- **Modelar** a execução de um agente como um objeto de estado explícito.
- **Derivar** a lista de mensagens a partir do estado, tratando-a como formato de transporte.
- **Expor dinamicamente** o subconjunto de ferramentas relevante à fase da tarefa.
- **Posicionar** a confirmação humana como uma forma de terminação, e não como um `input()` no meio do laço.

---

## Desenvolvimento teórico

### 1. As três condições do agente

A nota anterior deu a condição negativa: *não existe fluxograma*. Ela é necessária e não é suficiente. Um agente se justifica quando **três** coisas valem ao mesmo tempo:

1. **A tarefa é aberta** — o objetivo é claro, o caminho não.
2. **O número de passos é desconhecido** antes de rodar, e depende do que for descoberto durante a execução.
3. **O ambiente devolve feedback verificável.**

A terceira quase nunca é dita, e é a que decide. Um agente funciona porque **erra e se corrige**: chama uma ferramenta, recebe algo inesperado, ajusta. Se o mundo não devolve sinal de erro — se toda ferramenta responde "ok" e nada nunca contradiz o modelo —, o laço não corrige nada. Ele só acumula passos em cima de uma premissa errada, com a confiança intacta.

Compare dois casos:

| Tarefa | Feedback verificável? | Veredito |
|---|---|---|
| Corrigir um bug: roda o teste, o teste falha ou passa | **sim**, e é binário | agente funciona bem |
| Investigar uma despesa: consulta política, consulta histórico, valores batem ou não | **sim**, parcialmente | agente funciona, com verificação |
| Escrever um texto persuasivo: nenhuma ferramenta diz se convenceu | **não** | agente não ajuda; é uma chamada, talvez com avaliador |

> **A regra:** autonomia só compensa onde existe correção. Sem sinal de erro vindo do mundo, mais passos significam mais chances de errar, não mais chances de acertar.

---

### 2. O que a lista de mensagens não sabe

Volte ao laço da Aula 03. Ele guarda tudo numa lista:

```python
mensagens = [
    {"role": "system", "content": SYSTEM},
    {"role": "user", "content": pergunta},
]
```

Agora imagine esse agente na quadragésima mensagem, e tente responder estas perguntas **a partir da lista**:

| Pergunta | Onde está a resposta na lista? |
|---|---|
| Quantos passos já foram dados? | em lugar nenhum — dá para *inferir* contando mensagens `assistant`, o que é frágil |
| Quantos tokens já foram gastos? | não está: os `usage` das respostas foram descartados |
| Quanto isso já custou em dinheiro? | idem |
| Qual era o objetivo original? | está na mensagem 2, soterrada no meio (*lost in the middle*) |
| A ferramenta X já falhou antes? | espalhada em `content` de mensagens `tool`, como texto |
| Alguma chamada se repetiu com os mesmos argumentos? | daria para descobrir varrendo e reparseando JSON a cada volta |
| Por que o agente parou? | **não está em lugar nenhum** |

Repare no padrão. A informação ou **não existe** (tokens, custo, motivo de parada) ou existe **como texto dentro de um campo destinado ao modelo**, e teria que ser extraída de volta a cada volta.

É o mesmo erro de projeto que você reconheceria imediatamente em qualquer outro sistema: usar o formato de serialização da API como estrutura de dados interna. Ninguém guarda o estado de um pedido dentro do JSON que vai para o cliente HTTP — mas é exatamente isso que o laço da Aula 03 faz.

```
   ┌──────────────────────────────────────────────────────┐
   │  ANTES:  mensagens[]  é o estado E é o transporte    │
   │          → tudo que não cabe numa mensagem, some     │
   └──────────────────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────┐
   │  DEPOIS: Estado  é o estado                          │
   │          mensagens[]  é DERIVADO do estado           │
   │          → o estado guarda o que a API não carrega   │
   └──────────────────────────────────────────────────────┘
```

---

### 3. O objeto de estado

A Aula 01 (nota 03, §1.2) definiu o agente por quatro componentes: objetivo, LLM que decide, ferramentas e laço. O objeto de estado é literalmente essa definição virando estrutura de dados — com o acréscimo do que a execução produz.

```python
from dataclasses import dataclass, field
from enum import Enum
from uuid import uuid4


class Termino(str, Enum):
    RESPONDEU = "respondeu"            # o modelo concluiu
    ORCAMENTO = "orcamento_esgotado"   # bateu o teto
    ERRO_FATAL = "erro_fatal"          # não dá para continuar
    HUMANO = "aguardando_humano"       # precisa de confirmação


@dataclass
class Passo:
    indice: int
    ferramenta: str | None
    argumentos: dict
    resultado: dict | None = None
    erro: str | None = None
    tokens: int = 0
    resumo: str | None = None                     # preenchido pelo tool clearing
    limpo: bool = False


@dataclass
class Estado:
    objetivo: str                                   # imutável, a âncora
    execucao_id: str = field(default_factory=lambda: uuid4().hex[:8])
    passos: list[Passo] = field(default_factory=list)
    tokens_gastos: int = 0
    custo_estimado: float = 0.0
    ferramentas_ativas: list[str] = field(default_factory=list)
    termino: Termino | None = None
    motivo: str | None = None                      # detalhe do término
    resposta: str | None = None
    pendencia: dict | None = None                   # ação aguardando humano
    # o histórico como o modelo o vê — DERIVADO, e reconstruível
    historico: list[dict] = field(default_factory=list)

    @property
    def n_passos(self) -> int:
        return len(self.passos)

    def registrar(self, passo: Passo) -> None:
        self.passos.append(passo)
        self.tokens_gastos += passo.tokens
```

Três decisões de projeto que valem ser explicadas, porque não são óbvias:

**`objetivo` é campo próprio, não a primeira mensagem.** É a única informação que precisa estar disponível o tempo todo — para reafirmar no contexto quando a trajetória fica longa (nota 03, §7) e para sobreviver a qualquer compaction (nota 04). Guardá-lo apenas como `mensagens[1]` é aceitar que ele pode ser comprimido junto com o resto.

**`passos` é uma lista de objetos, não de dicionários de mensagem.** Cada `Passo` guarda ferramenta, argumentos e erro em **campos**, e não em texto. É isso que torna a detecção de laço uma comparação de tuplas em vez de um exercício de *parsing*.

**`termino` é obrigatório no fim.** Um agente que terminou sem registrar por quê é um agente que você não vai conseguir depurar às onze da noite com o sistema em produção.

---

### 4. A derivação: estado → mensagens

A lista de mensagens não desaparece — ela é montada a cada volta, a partir do estado:

```python
def montar_mensagens(estado: Estado) -> list[dict]:
    """A lista de mensagens é uma VISÃO do estado, montada para a API."""
    mensagens = [
        {"role": "system", "content": SYSTEM},
        {"role": "user", "content": estado.objetivo},
    ]
    mensagens.extend(estado.historico)

    # Reancoragem: em trajetória longa, o objetivo volta ao FIM da janela,
    # onde o modelo o aproveita melhor (aula 02, nota 01 §4.3).
    if estado.n_passos >= REANCORAR_A_CADA:
        mensagens.append({
            "role": "user",
            "content": f"Lembrete do objetivo: {estado.objetivo}",
        })

    return mensagens
```

E o laço passa a operar sobre o estado:

```python
def rodar(objetivo: str, orcamento: Orcamento) -> Estado:
    estado = Estado(objetivo=objetivo,
                    ferramentas_ativas=list(FERRAMENTAS))

    while True:
        if (parada := orcamento.excedido(estado)):     # nota 03, §1
            estado.termino = Termino.ORCAMENTO
            return estado

        r = client.chat.completions.create(
            model=MODELO,
            messages=montar_mensagens(estado),
            tools=declaracoes(estado.ferramentas_ativas),
            temperature=0,
        )
        msg = r.choices[0].message
        estado.tokens_gastos += r.usage.total_tokens   # agora existe

        if not msg.tool_calls:
            estado.termino = Termino.RESPONDEU
            estado.resposta = msg.content
            return estado

        estado.historico.append(msg)
        for chamada in msg.tool_calls:
            passo = executar(chamada, estado)          # nota 03, §3
            estado.registrar(passo)
            estado.historico.append(mensagem_de_tool(passo, chamada.id))
```

Compare com o original da Aula 03. As diferenças são três, e nenhuma é cosmética:

1. **A função devolve `Estado`, não `str`.** Quem chamou recebe a resposta *e* a trajetória, o custo e o motivo do término. Uma função que devolve só a string joga fora tudo que serve para depurar.
2. **`while True` com saídas nomeadas**, no lugar de `for passo in range(max_passos)`. O `for` codificava uma única condição de parada — o número de voltas. As outras três formas de terminar não cabiam nele.
3. **`estado.historico` é campo do estado**, e por isso pode ser reescrito (compaction, nota 04) sem que o resto do agente saiba.

Nada disso deixa o agente mais inteligente. Deixa o agente **observável**, que é a pré-condição de tudo o mais.

---

### 5. O que o estado destrava

Vale ser explícito sobre o retorno do investimento, porque a mudança dá trabalho:

| Salvaguarda | Precisa de | Onde |
|---|---|---|
| Orçamento em quatro moedas | `tokens_gastos`, `custo_estimado`, `n_passos`, hora de início | [nota 03](03-confiabilidade.md), §1 |
| Motivo de término | `termino` | nota 03, §2 |
| Erro recuperável × fatal | `Passo.erro` por passo | nota 03, §3 |
| Detecção de laço | `passos` com ferramenta e argumentos separados | nota 03, §6 |
| Reancoragem do objetivo | `objetivo` fora do histórico | nota 03, §7 |
| Checkpoint | o estado inteiro, serializável | nota 03, §9 |
| Compaction | `historico` isolado e reescrevível | [nota 04](04-context-engineering-dinamica.md) |

Sete salvaguardas, um pré-requisito comum. É por isso que esta nota vem antes das outras duas.

---

### 6. Múltiplas ferramentas, e a escolha entre elas

Com o estado no lugar, uma segunda melhoria fica barata.

Toda ferramenta declarada custa tokens em **todas** as chamadas — a declaração inteira, com descrição e schema, viaja na requisição a cada volta. Vinte ferramentas de ~120 tokens cada são ~2.400 tokens por volta, multiplicados por todas as voltas da trajetória.

E o custo em dinheiro é o menor dos dois problemas. O maior é a **qualidade da escolha**: quanto mais ferramentas parecidas na lista, mais o modelo erra qual chamar — é a linha "ferramenta errada" da tabela de falhas da Aula 01 (nota 03, §4).

A solução é expor apenas o subconjunto relevante à fase atual, o que o estado torna trivial:

```python
FASES = {
    "coleta":   ["buscar_despesa", "consultar_politica"],
    "analise":  ["consultar_politica", "consultar_historico"],
    "registro": ["registrar_parecer"],          # escrita, e SÓ ela
}

def declaracoes(ativas: list[str]) -> list[dict]:
    return [DECLARACOES[nome] for nome in ativas]
```

Repare no ganho que não é de custo: na fase `coleta`, `registrar_parecer` **não existe** para o modelo. Não é uma instrução no system prompt pedindo que ele não registre antes da hora — é a ausência da ferramenta. Instrução o modelo pode ignorar; ferramenta que não foi declarada ele não tem como chamar.

> **Restrição por arquitetura vence restrição por prompt.** Sempre que a diferença for possível, prefira tornar a ação indisponível a pedir que ela não seja tomada.

---

### 7. Humano no laço

A Aula 01 (nota 03, §3 e exercício resolvido 5) estabeleceu a fronteira: leitura o agente faz livremente; **escrita irreversível exige confirmação**. O que muda agora é *como* isso é implementado.

A versão ingênua põe um `input()` no meio do laço:

```python
if chamada.function.name == "registrar_parecer":
    if input("confirma? [s/N] ") != "s":       # não faça isto
        ...
```

Isso funciona no laboratório e não funciona em lugar nenhum além dele: prende um processo esperando um humano, não sobrevive a reinício e não existe quando o agente roda como job ou atrás de uma fila.

A versão correta trata a confirmação como **uma das quatro formas de terminar**:

```python
def executar(chamada, estado: Estado) -> Passo:
    nome = chamada.function.name
    argumentos = json.loads(chamada.function.arguments)

    if nome in EXIGE_CONFIRMACAO:
        estado.termino = Termino.HUMANO
        estado.pendencia = {"ferramenta": nome, "argumentos": argumentos}
        salvar_checkpoint(estado)              # nota 03, §9
        raise PausaParaHumano(estado)

    return Passo(indice=estado.n_passos, ferramenta=nome,
                 argumentos=argumentos, resultado=FERRAMENTAS[nome](**argumentos))
```

O agente **para**, grava o estado e devolve o controle. A aprovação chega depois — por outro processo, outra tela, outro dia — e a execução é retomada do checkpoint, sem repetir nenhum passo já dado.

Repare que isso só é possível porque o estado é um objeto serializável. Com o estado dentro de `mensagens[]` na memória do processo, "retomar depois" não é uma opção — é uma reescrita.

E há um custo a admitir: **cada ponto de confirmação mata um pedaço da autonomia que justificava o agente**. Um agente que pede confirmação a cada passo é um assistente de digitação caro. A confirmação vai onde o erro é irreversível — e em nenhum outro lugar.

---

## Exemplos

### Exemplo 1 — A mesma execução, vista pelas duas estruturas

Agente investigando uma despesa. Quatro passos, e o terceiro falha.

**Com `mensagens[]`** — o que o programa tem no fim:

```python
[{"role": "system", ...}, {"role": "user", ...},
 {"role": "assistant", "tool_calls": [...]}, {"role": "tool", "content": "{...}"},
 {"role": "assistant", "tool_calls": [...]}, {"role": "tool", "content": "{...}"},
 {"role": "assistant", "tool_calls": [...]}, {"role": "tool", "content": "{\"erro\": ...}"},
 {"role": "assistant", "content": "Não foi possível concluir."}]
```

Nove dicionários. Para saber que houve um erro, alguém precisa ler o `content` da sétima entrada e reparar que o JSON tem uma chave `erro`.

**Com `Estado`** — o que o programa tem no fim:

```python
Estado(
    objetivo="Analisar a despesa D-4471 segundo a política vigente",
    passos=[
        Passo(0, "buscar_despesa",      {"id": "D-4471"},   resultado={...}, tokens=310),
        Passo(1, "consultar_politica",  {"categoria": "refeicao"}, resultado={...}, tokens=402),
        Passo(2, "consultar_historico", {"funcionario": "F-88"},
              erro="funcionario inexistente: F-88", tokens=180),
        Passo(3, "consultar_historico", {"funcionario": "F-088"}, resultado={...}, tokens=395),
    ],
    tokens_gastos=1287, custo_estimado=0.0031,
    termino=Termino.RESPONDEU,
    resposta="Despesa D-4471 aprovada: R$ 84,00 dentro do teto de R$ 120,00 ...",
)
```

A segunda versão responde, sem *parsing*: quantos passos, quanto custou, o que falhou, que o modelo **se corrigiu sozinho** (`F-88` → `F-088`) e por que terminou. Este é o objeto que a aula de observabilidade vai chamar de *trace* — você acabou de escrever o seu primeiro.

### Exemplo 2 — A ferramenta que some

```python
# Fase de análise: registrar_parecer NÃO está declarada.
estado.ferramentas_ativas = FASES["analise"]

# O modelo tenta registrar mesmo assim? Não tem como: a ferramenta não existe
# na requisição. O máximo que ele faz é DIZER que registrou — e aí a ausência
# de um Passo com ferramenta="registrar_parecer" denuncia a alucinação.
assert not any(p.ferramenta == "registrar_parecer" for p in estado.passos)
```

O `assert` acima é um teste de verdade, e ele só é escrevível porque o estado guarda os passos como dados. Com a lista de mensagens, o mesmo teste seria uma busca por substring.

---

## Exercícios resolvidos

### 1. Agente ou não?

> *"Quero um agente que leia as reclamações do mês e escreva um relatório executivo com os três principais problemas."*

**Não é agente.** Aplique as três condições:

1. Tarefa aberta? Só parcialmente — o formato do relatório é conhecido.
2. Número de passos desconhecido? Não: ler tudo, agrupar, escolher três, escrever. Quatro etapas.
3. **Feedback verificável? Não.** Nenhuma ferramenta consegue dizer se o relatório identificou os problemas certos. O laço não teria com que se corrigir.

A terceira condição sozinha já resolve o caso. A arquitetura adequada é *sectioning* (resumir os lotes de reclamações em paralelo) seguido de uma chamada de síntese — e, se a qualidade do texto importar, um avaliador-otimizador com critério escrito.

Se o pedido fosse *"investigue por que as reclamações de entrega dobraram em março"*, a resposta mudaria: o caminho depende do que cada consulta revelar, e os dados devolvem contradição — ou seja, feedback.

### 2. O que está errado neste laço?

```python
def rodar(objetivo, max_passos=10):
    msgs = [{"role": "system", "content": SYSTEM},
            {"role": "user", "content": objetivo}]
    for _ in range(max_passos):
        r = client.chat.completions.create(model=MODELO, messages=msgs,
                                           tools=TODAS_AS_20_FERRAMENTAS)
        if not r.choices[0].message.tool_calls:
            return r.choices[0].message.content
        ...
    return "não consegui"
```

Quatro defeitos, em ordem de gravidade:

1. **`return "não consegui"` é indistinguível de uma resposta.** Quem chamou recebe uma string e não tem como saber se o agente concluiu ou estourou o teto. Motivo de término precisa ser dado estruturado, não texto.
2. **As 20 ferramentas em toda chamada** — custo fixo por volta e escolha pior. Deveria ser o subconjunto da fase (§6).
3. **`r.usage` é descartado.** Sem tokens, não há orçamento, não há custo e não há como saber o que a execução consumiu.
4. **`max_passos` é a única barreira.** Um passo pode custar 200 tokens ou 40.000; dez passos não é um limite de nada. É a discussão da [nota 03, §1](03-confiabilidade.md).

O quinto defeito é o que gera todos os outros: **`msgs` é o estado**.

---

## Síntese

- Um agente se justifica com três condições: tarefa aberta, número de passos desconhecido e **feedback verificável do ambiente**. Sem a terceira, autonomia só amplifica o erro.
- A lista de mensagens é um **formato de transporte**, não uma estrutura de estado. Ela não sabe quantos passos houve, quanto custou, o que falhou nem por que parou.
- O objeto de estado é a definição de agente da Aula 01 virando `dataclass`: objetivo, passos, orçamento consumido, ferramentas ativas, término.
- O histórico passa a ser **derivado** do estado e montado a cada volta — o que o torna reescrevível.
- Sete salvaguardas das notas 03 e 04 dependem dessa separação. É a mudança que paga o resto da aula.
- Exponha só as ferramentas da fase atual: economiza tokens e melhora a escolha. **Restrição por arquitetura vence restrição por prompt.**
- A confirmação humana é uma **forma de terminação** com checkpoint, não um `input()` no meio do laço — e cada uma delas custa um pedaço da autonomia.

---

## Fontes e leituras

- Aula 01, [nota 03 §1.2, §3 e §4](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — os quatro componentes, projeto de ferramentas e a tabela de falhas.
- Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — o laço original que esta nota reescreve.
- **A practical guide to building agents** — OpenAI (2025). Os capítulos de orquestração e de *human-in-the-loop* tratam do mesmo problema por outro caminho.
- **Building Effective Agents** — Anthropic (2024). A seção sobre agentes autônomos e a insistência em ambientes que devolvem sinal.
