# IA Aplicada com LLMs — Aula 05: Arquitetura de agentes — O agente e o estado

## Introdução

A [nota anterior](01-padroes-de-arquitetura.md) gastou cinco padrões defendendo que você quase sempre deveria escrever um workflow. Esta nota trata do caso em que você não deve — e aí o problema deixa de ser *escolher a arquitetura* e passa a ser *controlar a que você escolheu*.

É a nota mais curta da aula e a mais importante, porque contém uma única mudança de código, aparentemente burocrática, da qual dependem **todas** as salvaguardas das notas 03 e 04:

> **A lista de mensagens não pode ser o estado do agente.**

Enquanto ela for, você não impõe orçamento, não detecta laço, não salva checkpoint e não comprime contexto. Não porque seja difícil — porque a informação necessária **não está lá**.

> **Pré-requisitos:** [nota 01](01-padroes-de-arquitetura.md) desta aula · Aula 03, [nota 04 §5 e §6](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) · Aula 01, [nota 03 §1.2 e §3](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md).
>
> **Código:** [`03-agente-com-estado.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/03-agente-com-estado.py) — o laço da Aula 03 e esta versão lado a lado.

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Reconhecer** as três condições que justificam um agente — inclusive a que quase nunca é dita.
- **Enumerar** as perguntas que a lista de mensagens não responde.
- **Modelar** a execução como objeto de estado explícito, e **derivar** dele a lista de mensagens.
- **Expor dinamicamente** só o subconjunto de ferramentas da fase atual.
- **Posicionar** a confirmação humana como forma de terminação, não como `input()` no meio do laço.

---

## Desenvolvimento teórico

### 1. As três condições do agente

A nota anterior deu a condição negativa: *não existe fluxograma*. Necessária, não suficiente. Um agente se justifica quando **três** coisas valem ao mesmo tempo:

1. **A tarefa é aberta** — objetivo claro, caminho não.
2. **O número de passos é desconhecido** antes de rodar.
3. **O ambiente devolve feedback verificável.**

A terceira quase nunca é dita, e é a que decide. Um agente funciona porque **erra e se corrige**. Se toda ferramenta responde "ok" e nada nunca contradiz o modelo, o laço não corrige nada — só empilha passos sobre uma premissa errada, com a confiança intacta.

| Tarefa | Feedback verificável? | Veredito |
|---|---|---|
| Corrigir um bug: roda o teste, passa ou falha | **sim**, binário | agente funciona bem |
| Investigar despesa: valores batem com a política ou não | **sim**, parcial | agente funciona, com verificação |
| Escrever texto persuasivo: nada diz se convenceu | **não** | não é agente — é uma chamada, talvez com avaliador |

> **A regra:** autonomia só compensa onde existe correção. Sem sinal de erro vindo do mundo, mais passos são mais chances de errar, não de acertar.

---

### 2. O que a lista de mensagens não sabe

Imagine o agente da Aula 03 na quadragésima mensagem. Responda **a partir da lista**:

| Pergunta | Onde está a resposta? |
|---|---|
| Quantos passos já foram dados? | em lugar nenhum — dá para *inferir* contando `assistant`, o que é frágil |
| Quantos tokens? Quanto custou? | não está: os `usage` foram descartados |
| Qual era o objetivo original? | na mensagem 2, soterrada (*lost in the middle*) |
| A ferramenta X já falhou antes? | espalhada em `content` de mensagens `tool`, como **texto** |
| Alguma chamada se repetiu? | daria para descobrir varrendo e reparseando JSON a cada volta |
| Por que o agente parou? | **não está em lugar nenhum** |

O padrão: a informação ou **não existe**, ou existe **como texto dentro de um campo destinado ao modelo**. É o erro de projeto que você reconheceria em qualquer outro sistema — usar o formato de serialização da API como estrutura de dados interna. Ninguém guarda o estado de um pedido dentro do JSON que vai para o cliente HTTP.

```
   ANTES:   mensagens[]  é o estado E é o transporte
            → tudo que não cabe numa mensagem, some

   DEPOIS:  Estado       é o estado
            mensagens[]  é DERIVADO do estado
            → o estado guarda o que a API não carrega
```

---

### 3. O objeto de estado

A Aula 01 (nota 03, §1.2) definiu o agente por quatro componentes: objetivo, LLM, ferramentas, laço. O objeto de estado é essa definição virando estrutura de dados, mais o que a execução produz.

```python
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
    resumo: str | None = None          # preenchido pelo tool clearing (nota 04)
    limpo: bool = False

@dataclass
class Estado:
    objetivo: str                                    # imutável, a âncora
    execucao_id: str = field(default_factory=lambda: uuid4().hex[:8])
    passos: list[Passo] = field(default_factory=list)
    tokens_gastos: int = 0
    custo_estimado: float = 0.0
    ferramentas_ativas: list[str] = field(default_factory=list)
    termino: Termino | None = None
    motivo: str | None = None
    resposta: str | None = None
    pendencia: dict | None = None                    # ação aguardando humano
    historico: list[dict] = field(default_factory=list)   # DERIVADO, reconstruível
```

Três decisões que não são óbvias:

| Decisão | Por quê |
|---|---|
| **`objetivo` é campo próprio**, não a primeira mensagem | é a única informação que precisa estar disponível o tempo todo — para reancorar (nota 03 §7) e sobreviver à compaction (nota 04). Como `mensagens[1]`, ele pode ser comprimido junto com o resto |
| **`passos` é lista de objetos**, não de dicionários de mensagem | ferramenta, argumentos e erro em **campos**, não em texto. É o que torna a detecção de laço uma comparação de tuplas em vez de *parsing* |
| **`termino` é obrigatório no fim** | um agente que terminou sem registrar por quê é um agente que você não depura às onze da noite com o sistema em produção |

---

### 4. A derivação: estado → mensagens

A lista não desaparece — ela é montada a cada volta, a partir do estado:

```python
def montar_mensagens(estado: Estado) -> list[dict]:
    """A lista de mensagens é uma VISÃO do estado, montada para a API."""
    mensagens = [{"role": "system", "content": SYSTEM},
                 {"role": "user", "content": estado.objetivo}]
    mensagens.extend(estado.historico)

    if estado.n_passos >= REANCORAR_A_CADA:      # o objetivo volta ao FIM
        mensagens.append({"role": "user",        # da janela, onde o modelo
                          "content": f"Objetivo: {estado.objetivo}"})  # o aproveita
    return mensagens
```

E o laço passa a operar sobre o estado:

```python
def rodar(objetivo: str, orcamento: Orcamento) -> Estado:
    estado = Estado(objetivo=objetivo, ferramentas_ativas=list(FERRAMENTAS))

    while True:
        if orcamento.excedido(estado):                  # nota 03, §1
            estado.termino = Termino.ORCAMENTO
            return estado

        r = client.chat.completions.create(
            model=MODELO, messages=montar_mensagens(estado),
            tools=declaracoes(estado.ferramentas_ativas), temperature=0)
        msg = r.choices[0].message
        estado.tokens_gastos += r.usage.total_tokens    # agora existe

        if not msg.tool_calls:
            estado.termino, estado.resposta = Termino.RESPONDEU, msg.content
            return estado

        estado.historico.append(msg)
        for chamada in msg.tool_calls:
            passo = executar(chamada, estado)           # nota 03, §3
            estado.registrar(passo)
            estado.historico.append(mensagem_de_tool(passo, chamada.id))
```

Três diferenças em relação ao original, nenhuma cosmética:

1. **Devolve `Estado`, não `str`.** Quem chamou recebe a resposta *e* a trajetória, o custo e o motivo do término. Devolver só a string joga fora tudo que serve para depurar.
2. **`while True` com saídas nomeadas**, no lugar de `for passo in range(max_passos)`. O `for` codificava uma única condição de parada; as outras três não cabiam nele.
3. **`historico` é campo do estado**, e por isso pode ser reescrito (compaction) sem que o resto do agente saiba.

Nada disso deixa o agente mais inteligente. Deixa o agente **observável**, que é a pré-condição de tudo o mais.

---

### 5. O que o estado destrava

| Salvaguarda | Precisa de | Onde |
|---|---|---|
| Orçamento em quatro moedas | `tokens_gastos`, `custo_estimado`, `n_passos`, início | [nota 03](03-confiabilidade.md) §1 |
| Motivo de término | `termino` | nota 03 §2 |
| Erro recuperável × fatal | `Passo.erro` por passo | nota 03 §3 |
| Detecção de laço | `passos` com ferramenta e argumentos separados | nota 03 §6 |
| Reancoragem do objetivo | `objetivo` fora do histórico | nota 03 §7 |
| Checkpoint | o estado inteiro, serializável | nota 03 §9 |
| Compaction | `historico` isolado e reescrevível | [nota 04](04-context-engineering-dinamica.md) |

Sete salvaguardas, um pré-requisito comum. É por isso que esta nota vem antes das outras duas.

---

### 6. Múltiplas ferramentas, e a escolha entre elas

Toda ferramenta declarada custa tokens em **todas** as chamadas: descrição e schema viajam na requisição a cada volta. Vinte ferramentas de ~120 tokens são ~2.400 tokens por volta, multiplicados por toda a trajetória.

E o dinheiro é o menor dos dois problemas. O maior é a **qualidade da escolha**: quanto mais ferramentas parecidas, mais o modelo erra qual chamar — a linha "ferramenta errada" da tabela de falhas da Aula 01 (nota 03, §4).

```python
FASES = {
    "coleta":   ["buscar_despesa", "consultar_politica"],
    "analise":  ["consultar_politica", "consultar_historico"],
    "registro": ["registrar_parecer"],          # escrita, e SÓ ela
}

def declaracoes(ativas: list[str]) -> list[dict]:
    return [DECLARACOES[nome] for nome in ativas]
```

O ganho que não é de custo: na fase `coleta`, `registrar_parecer` **não existe** para o modelo. Não é uma instrução no system prompt pedindo que ele não registre antes da hora — é a ausência da ferramenta. Instrução ele pode ignorar; ferramenta não declarada ele não tem como chamar.

> **Restrição por arquitetura vence restrição por prompt.** Sempre que der, prefira tornar a ação indisponível a pedir que ela não seja tomada.

---

### 7. Humano no laço

A Aula 01 (nota 03 §3) estabeleceu a fronteira: leitura o agente faz livremente; **escrita irreversível exige confirmação**. O que muda é *como* isso é implementado.

```python
if chamada.function.name == "registrar_parecer":
    if input("confirma? [s/N] ") != "s":       # NÃO faça isto
        ...
```

Funciona no laboratório e em lugar nenhum além dele: prende um processo esperando um humano, não sobrevive a reinício e não existe quando o agente roda como job ou atrás de uma fila. A versão correta trata a confirmação como **uma das quatro formas de terminar**:

```python
def executar(chamada, estado: Estado) -> Passo:
    nome = chamada.function.name
    argumentos = json.loads(chamada.function.arguments)

    if nome in EXIGE_CONFIRMACAO:
        estado.termino = Termino.HUMANO
        estado.pendencia = {"ferramenta": nome, "argumentos": argumentos}
        salvar_checkpoint(estado)              # nota 03, §9
        raise PausaParaHumano(estado)

    return Passo(indice=estado.n_passos, ferramenta=nome, argumentos=argumentos,
                 resultado=FERRAMENTAS[nome](**argumentos))
```

O agente **para**, grava o estado e devolve o controle. A aprovação chega depois — por outro processo, outra tela, outro dia — e a execução é retomada do checkpoint, sem repetir nenhum passo já dado. Só é possível porque o estado é serializável: com o estado dentro de `mensagens[]` na memória do processo, "retomar depois" não é opção, é reescrita.

E há um custo a admitir: **cada ponto de confirmação mata um pedaço da autonomia que justificava o agente**. Um agente que pede confirmação a cada passo é um assistente de digitação caro. A confirmação vai onde o erro é irreversível — e em nenhum outro lugar.

---

## Exemplos

### Exemplo 1 — A mesma execução, vista pelas duas estruturas

Agente investigando uma despesa. Quatro passos, e o terceiro falha.

**Com `mensagens[]`**, o que sobra no fim são nove dicionários. Para saber que houve erro, alguém precisa ler o `content` da sétima entrada e reparar que o JSON tem uma chave `erro`.

**Com `Estado`:**

```python
Estado(
    objetivo="Analisar a despesa D-4471 segundo a política vigente",
    passos=[
      Passo(0, "buscar_despesa",      {"id": "D-4471"},          resultado={...}, tokens=310),
      Passo(1, "consultar_politica",  {"categoria": "refeicao"},  resultado={...}, tokens=402),
      Passo(2, "consultar_historico", {"funcionario": "F-88"},
            erro="funcionario inexistente: F-88", tokens=180),
      Passo(3, "consultar_historico", {"funcionario": "F-088"},   resultado={...}, tokens=395),
    ],
    tokens_gastos=1287, custo_estimado=0.0031, termino=Termino.RESPONDEU,
    resposta="Despesa D-4471 aprovada: R$ 84,00 dentro do teto de R$ 120,00 ...")
```

Sem *parsing*, responde: quantos passos, quanto custou, o que falhou, que o modelo **se corrigiu sozinho** (`F-88` → `F-088`) e por que terminou. Este é o objeto que a aula de observabilidade vai chamar de *trace*.

### Exemplo 2 — Três perguntas, uma linha cada

O que o estado torna trivial e a lista de mensagens torna um projeto:

```python
sum(1 for p in estado.passos if p.erro)                          # quantos erros?
Counter((p.ferramenta, str(p.argumentos)) for p in estado.passos) # repetiu chamada?
[p.ferramenta for p in estado.passos if p.tokens > 5_000]         # quem come a janela?
```

Nenhuma delas é escrevível sobre `mensagens[]` sem varrer e reparsear JSON a cada volta.

### Exemplo 3 — A ferramenta que some

```python
estado.ferramentas_ativas = FASES["analise"]   # registrar_parecer NÃO declarada

# O modelo tenta registrar mesmo assim? Não tem como. O máximo que ele faz é
# DIZER que registrou — e a ausência do Passo denuncia a alucinação:
assert not any(p.ferramenta == "registrar_parecer" for p in estado.passos)
```

Esse `assert` é um teste de verdade, e só é escrevível porque o estado guarda os passos como dados. Com a lista de mensagens, seria uma busca por substring.

### Exemplo 4 — O checkpoint que retoma

```python
def salvar_checkpoint(estado: Estado) -> None:
    Path(f"ckpt/{estado.execucao_id}.json").write_text(
        json.dumps(asdict(estado), ensure_ascii=False, default=str))

def retomar(execucao_id: str, aprovado: bool) -> Estado:
    estado = Estado(**json.loads(Path(f"ckpt/{execucao_id}.json").read_text()))
    if not aprovado:
        estado.termino, estado.motivo = Termino.HUMANO, "reprovado pelo aprovador"
        return estado
    estado.termino, estado.pendencia = None, None    # segue de onde parou
    return continuar(estado)
```

Quatro linhas de `json.dumps` são tudo o que separa "o agente pausou" de "o agente morreu". E repare no que **não** aparece aqui: nenhum passo é reexecutado — os três primeiros já estão em `estado.passos`, com resultado.

---

## Exercícios resolvidos

### 1. Agente ou não?

> *"Quero um agente que leia as reclamações do mês e escreva um relatório executivo com os três principais problemas."*

**Não é agente.** Tarefa aberta? Só em parte — o formato é conhecido. Passos desconhecidos? Não: ler, agrupar, escolher três, escrever. **Feedback verificável? Não** — nenhuma ferramenta diz se o relatório identificou os problemas certos. A terceira condição sozinha resolve o caso.

A arquitetura adequada é *sectioning* (resumir lotes em paralelo) + uma chamada de síntese; se a qualidade do texto importar, um avaliador-otimizador com critério escrito.

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

| # | Defeito | Consequência |
|---|---|---|
| 1 | `return "não consegui"` é indistinguível de uma resposta | quem chamou não sabe se concluiu ou estourou o teto. Motivo de término é dado, não texto |
| 2 | 20 ferramentas em toda chamada | custo fixo por volta e escolha pior (§6) |
| 3 | `r.usage` é descartado | sem tokens não há orçamento, nem custo, nem consumo conhecido |
| 4 | `max_passos` é a única barreira | um passo pode custar 200 tokens ou 40.000 ([nota 03 §1](03-confiabilidade.md)) |

O quinto defeito é o que gera todos os outros: **`msgs` é o estado**.

---

## Síntese

- Três condições justificam um agente: tarefa aberta, passos desconhecidos e **feedback verificável**. Sem a terceira, autonomia só amplifica o erro.
- A lista de mensagens é **transporte**, não estado. Ela não sabe quantos passos houve, quanto custou, o que falhou nem por que parou.
- O objeto de estado é a definição de agente da Aula 01 virando `dataclass`.
- O histórico passa a ser **derivado** do estado — e por isso reescrevível.
- Sete salvaguardas das notas 03 e 04 dependem dessa separação. É a mudança que paga o resto da aula.
- Exponha só as ferramentas da fase atual. **Restrição por arquitetura vence restrição por prompt.**
- Confirmação humana é **forma de terminação** com checkpoint, não `input()` no meio do laço — e cada uma custa um pedaço da autonomia.

---

## Fontes e leituras

- Aula 01, [nota 03 §1.2, §3 e §4](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — os quatro componentes, projeto de ferramentas e a tabela de falhas.
- Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — o laço original que esta nota reescreve.
- **A practical guide to building agents** — OpenAI (2025). Orquestração e *human-in-the-loop*.
- **Building Effective Agents** — Anthropic (2024). Agentes autônomos e a insistência em ambientes que devolvem sinal.
