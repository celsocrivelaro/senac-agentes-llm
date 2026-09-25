# IA Aplicada com LLMs — Aula 08: Memória — O agente e o estado

## Introdução

As [notas de padrões da Aula 05](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-6-qual-padrao-usar.md) apresentaram cinco arquiteturas em defesa da tese de que, na maioria dos casos, a solução adequada é um *workflow*. Esta nota trata do caso restante — aquele em que a autonomia se justifica. Aí o problema deixa de ser a escolha da arquitetura e passa a ser o controle da arquitetura escolhida.

> **Por que esta nota abre a aula de memória.** A Aula 05 declarou o **estado mínimo** que as salvaguardas dela exigiam — seis campos, na abertura da [nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — e parou ali. O objeto inteiro fica aqui porque é aqui que ele encontra a pergunta que o justifica: **o que, deste objeto, sobrevive entre execuções?**
>
> *Checkpoint* é propriedade do estado, e é o assunto da **§8** desta nota: o estado gravado em disco, e o que isso destrava. Memória é o que vem depois dele, na [nota 02](02-checkpoint-nao-e-memoria.md). Nenhuma das duas é formulável antes de o objeto existir.

É a nota mais curta da aula e a mais consequente, porque contém uma única alteração de código, aparentemente burocrática, da qual dependem **todas** as salvaguardas da Aula 05 e tudo o que vem a seguir:

> **A lista de mensagens não pode ser a estrutura de estado do agente.**

Enquanto o for, não é possível impor orçamento, detectar laço, salvar *checkpoint* ou comprimir contexto. Não por dificuldade de implementação: a informação necessária **não está representada**.

> **Pré-requisitos:** [nota 01](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-0-padroes-de-arquitetura.md) e a série de padrões ([01-1](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-1-sequencial.md) a [01-6](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-6-qual-padrao-usar.md)) · Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) §5 e §6 · Aula 01, [nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) §1.2 e §3.
>
> **Código:** [`04-agente-com-estado.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/04-agente-com-estado.py) — o laço da Aula 03 e a versão desta nota, lado a lado — e [`00-o-checkpoint.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula08-memoria/00-o-checkpoint.py), que grava o arquivo da §8, o imprime por extenso e retoma a execução a partir dele.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Reconhecer** as três condições que justificam um agente, inclusive a que raramente é enunciada.
- **Enumerar** as perguntas que a lista de mensagens não responde.
- **Modelar** a execução como objeto de estado explícito, e **derivar** dele a lista de mensagens.
- **Expor dinamicamente** apenas o subconjunto de ferramentas pertinente à fase corrente.
- **Posicionar** a confirmação humana como forma de terminação, e não como `input()` no interior do laço.
- **Serializar** o estado em *checkpoint*, **retomar** a execução a partir dele sem repetir passos, e **justificar** por que a persistência ocorre depois da execução e nunca antes.

---

## Desenvolvimento teórico

### 1. As três condições do agente

A nota anterior estabeleceu a condição negativa: não existe fluxograma. Ela é necessária e insuficiente. Um agente se justifica quando **três** condições valem simultaneamente:

1. **A tarefa é aberta** — o objetivo é claro, o caminho não.
2. **O número de passos é desconhecido** antes da execução.
3. **O ambiente devolve *feedback* verificável.**

A terceira raramente é enunciada e é a que decide. Um agente opera porque **erra e se corrige**. Se toda ferramenta responde afirmativamente e nada contradiz o modelo, o laço não corrige coisa alguma: acumula passos sobre uma premissa incorreta, com a confiança declarada intacta.

| Tarefa | *Feedback* verificável | Veredito |
|---|---|---|
| Corrigir um defeito de software: o teste passa ou falha | **sim**, binário | agente é adequado |
| Investigar despesa: os valores conferem com a política ou não | **sim**, parcial | agente é adequado, com verificação |
| Redigir texto persuasivo: nenhuma ferramenta determina se convenceu | **não** | não é agente — é uma chamada, eventualmente com avaliador |

> **Regra:** autonomia só compensa onde há correção. Sem sinal de erro proveniente do ambiente, mais passos significam mais oportunidades de errar, não de acertar.

---

### 2. O que a lista de mensagens não representa

Considere o agente da Aula 03 na quadragésima mensagem. As perguntas seguintes não são respondíveis a partir da lista:

| Pergunta | Situação da informação |
|---|---|
| Quantos passos foram executados? | ausente — é possível *inferir* contando mensagens `assistant`, o que é frágil |
| Quantos tokens foram consumidos? Qual o custo? | ausente: os campos `usage` das respostas foram descartados |
| Qual era o objetivo original? | na mensagem 2, soterrada no meio da janela (*lost in the middle*) |
| A ferramenta X já falhou anteriormente? | dispersa em `content` de mensagens `tool`, como **texto** |
| Alguma chamada se repetiu? | recuperável apenas por varredura e reanálise de JSON a cada volta |
| Por que o agente parou? | **ausente** |

O padrão é regular: a informação ou **não existe**, ou existe **como texto no interior de um campo destinado ao modelo**. Trata-se do erro de projeto que seria reconhecido de imediato em qualquer outro sistema — empregar o formato de serialização da API como estrutura de dados interna. O estado de um pedido não é armazenado dentro do JSON enviado ao cliente HTTP.

```
   ANTES:   mensagens[]  é o estado E é o transporte
            → o que não couber numa mensagem, desaparece

   DEPOIS:  Estado       é o estado
            mensagens[]  é DERIVADO do estado
            → o estado armazena o que a API não carrega
```

---

### 3. O objeto de estado

A Aula 01 (nota 03, §1.2) definiu o agente por quatro componentes: objetivo, LLM, ferramentas e laço. O objeto de estado é essa definição convertida em estrutura de dados, acrescida do que a execução produz.

```python
class Termino(str, Enum):
    RESPONDEU = "respondeu"            # o modelo concluiu
    ORCAMENTO = "orcamento_esgotado"   # teto atingido
    ERRO_FATAL = "erro_fatal"          # não há como continuar
    HUMANO = "aguardando_humano"       # confirmação pendente

@dataclass
class Passo:
    indice: int
    ferramenta: str | None
    argumentos: dict
    resultado: dict | None = None
    erro: str | None = None
    tokens: int = 0
    resumo: str | None = None          # tool clearing (Aula 05, nota 03)
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

Três decisões de projeto não são evidentes:

| Decisão | Justificativa |
|---|---|
| **`objetivo` é campo próprio**, e não a primeira mensagem | é a única informação que precisa estar disponível durante toda a execução — para reancoragem (Aula 05, nota 02 §7) e para sobreviver à *compaction* (Aula 05, nota 03). Armazenado como `mensagens[1]`, pode ser comprimido junto com o restante |
| **`passos` é lista de objetos**, e não de dicionários de mensagem | ferramenta, argumentos e erro em **campos**, não em texto. É o que converte a detecção de laço em comparação de tuplas, em vez de análise sintática |
| **`termino` é obrigatório ao final** | um agente que encerrou sem registrar a causa não é depurável em produção |

---

### 4. A derivação: estado → mensagens

A lista de mensagens não desaparece: passa a ser montada a cada volta, a partir do estado.

```python
def montar_mensagens(estado: Estado) -> list[dict]:
    """A lista de mensagens é uma VISÃO do estado, montada para a API."""
    mensagens = [{"role": "system", "content": SYSTEM},
                 {"role": "user", "content": estado.objetivo}]
    mensagens.extend(estado.historico)

    if estado.n_passos >= REANCORAR_A_CADA:      # o objetivo retorna ao FIM
        mensagens.append({"role": "user",        # da janela, onde é melhor
                          "content": f"Objetivo: {estado.objetivo}"})  # aproveitado
    return mensagens
```

O laço passa a operar sobre o estado:

```python
def rodar(objetivo: str, orcamento: Orcamento) -> Estado:
    estado = Estado(objetivo=objetivo, ferramentas_ativas=list(FERRAMENTAS))

    while True:
        if orcamento.excedido(estado):                  # Aula 05, nota 02 §1
            estado.termino = Termino.ORCAMENTO
            return estado

        r = client.chat.completions.create(
            model=MODELO, messages=montar_mensagens(estado),
            tools=declaracoes(estado.ferramentas_ativas), temperature=0)
        msg = r.choices[0].message
        estado.tokens_gastos += r.usage.total_tokens    # agora representado

        if not msg.tool_calls:
            estado.termino, estado.resposta = Termino.RESPONDEU, msg.content
            return estado

        estado.historico.append(msg)
        for chamada in msg.tool_calls:
            passo = executar(chamada, estado)           # Aula 05, nota 02 §3
            estado.registrar(passo)
            estado.historico.append(mensagem_de_tool(passo, chamada.id))
```

Três diferenças em relação ao original, nenhuma cosmética:

1. **A função devolve `Estado`, não `str`.** O chamador recebe a resposta *e* a trajetória, o custo e a causa do término. Devolver apenas a cadeia de caracteres descarta tudo o que serve ao diagnóstico.
2. **`while True` com saídas nomeadas**, em lugar de `for passo in range(max_passos)`. O `for` codificava uma única condição de parada; as outras três não são expressáveis nele.
3. **`historico` é campo do estado** e, por isso, pode ser reescrito (*compaction*, Aula 05, nota 03) sem conhecimento do restante do agente.

Nenhuma dessas alterações torna o agente mais capaz. Tornam-no **observável**, que é a pré-condição de todo o restante.

---

### 5. O que o estado viabiliza

| Salvaguarda | Requer | Localização |
|---|---|---|
| Orçamento em quatro moedas | `tokens_gastos`, `custo_estimado`, `n_passos`, instante inicial | [nota 02 da Aula 05](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) §1 |
| Motivo de término | `termino` | Aula 05, nota 02 §2 |
| Erro recuperável × fatal | `Passo.erro` por passo | Aula 05, nota 02 §3 |
| Detecção de laço | `passos` com ferramenta e argumentos em campos distintos | Aula 05, nota 02 §6 |
| Reancoragem do objetivo | `objetivo` fora do histórico | Aula 05, nota 02 §7 |
| *Checkpoint* | o estado íntegro, serializável | **§8 desta nota** |
| *Compaction* | `historico` isolado e reescrevível | [nota 03 da Aula 05](../../aula-05-arquitetura-de-agentes/notas-de-aula/03-context-engineering-dinamica.md) · técnica na Aula 08 |

Sete salvaguardas com um pré-requisito comum. É a razão pela qual esta nota precede as duas seguintes.

---

### 6. Múltiplas ferramentas e a escolha entre elas

Toda ferramenta declarada consome tokens em **todas** as chamadas: descrição e schema trafegam na requisição a cada volta. Vinte ferramentas de aproximadamente 120 tokens somam cerca de 2.400 tokens por volta, multiplicados por toda a trajetória.

O custo financeiro é o menor dos dois problemas. O maior é a **qualidade da seleção**: quanto mais ferramentas semelhantes constam da lista, maior a taxa de erro na escolha — condição registrada como "ferramenta errada" na tabela de modos de falha da Aula 01 (nota 03, §4).

```python
FASES = {
    "coleta":   ["buscar_despesa", "consultar_politica"],
    "analise":  ["consultar_politica", "consultar_historico"],
    "registro": ["registrar_parecer"],          # escrita, e SÓ ela
}

def declaracoes(ativas: list[str]) -> list[dict]:
    return [DECLARACOES[nome] for nome in ativas]
```

O ganho não financeiro: na fase `coleta`, `registrar_parecer` **não existe** para o modelo. Não se trata de instrução no *system prompt* proibindo o registro antecipado — trata-se da ausência da ferramenta. Instrução é passível de ser ignorada; ferramenta não declarada não é invocável.

> **Restrição por arquitetura supera restrição por prompt.** Sempre que a alternativa existir, torna-se a ação indisponível em vez de solicitar que não seja executada.

---

### 7. Humano no laço

A Aula 01 (nota 03, §3) estabeleceu a fronteira: leitura é executada livremente pelo agente; **escrita irreversível exige confirmação**. O que se altera aqui é a forma de implementação.

```python
if chamada.function.name == "registrar_parecer":
    if input("confirma? [s/N] ") != "s":       # implementação inadequada
        ...
```

Essa forma opera em laboratório e em nenhum outro contexto: bloqueia um processo à espera de intervenção humana, não sobrevive a reinício e não se aplica quando o agente executa como *job* ou por trás de uma fila. A forma adequada trata a confirmação como **uma das quatro formas de terminação**:

```python
def executar(chamada, estado: Estado) -> Passo:
    nome = chamada.function.name
    argumentos = json.loads(chamada.function.arguments)

    if nome in EXIGE_CONFIRMACAO:
        estado.termino = Termino.HUMANO
        estado.pendencia = {"ferramenta": nome, "argumentos": argumentos}
        salvar_checkpoint(estado)              # §8
        raise PausaParaHumano(estado)

    return Passo(indice=estado.n_passos, ferramenta=nome, argumentos=argumentos,
                 resultado=FERRAMENTAS[nome](**argumentos))
```

O agente **interrompe a execução**, persiste o estado e devolve o controle. A aprovação ocorre posteriormente — em outro processo, outra interface, outro dia — e a execução é retomada a partir do *checkpoint*, sem repetição de passos já executados. Isso só é possível porque o estado é serializável: com o estado contido em `mensagens[]` na memória do processo, a retomada posterior não é uma opção, e sim uma reescrita.

Há um custo a registrar: **cada ponto de confirmação suprime parte da autonomia que justificava o agente**. Um agente que solicita confirmação a cada passo é um assistente de digitação de custo elevado. A confirmação se posiciona onde o erro é irreversível, e em nenhum outro ponto.

---

### 8. O *checkpoint*

A §7 terminou com uma chamada a `salvar_checkpoint(estado)` e seguiu adiante. Esta seção abre o arquivo que essa chamada grava.

*Checkpoint* é o estado **serializado em disco**, um arquivo por execução, nomeado pelo identificador dela. A implementação cabe em quatro linhas, e é essa desproporção — quatro linhas destravando três capacidades distintas — que a torna fácil de subestimar:

```python
CHECKPOINTS = Path(__file__).parent / "checkpoints"

def salvar_checkpoint(estado: Estado) -> None:
    CHECKPOINTS.mkdir(exist_ok=True)
    (CHECKPOINTS / f"{estado.execucao_id}.json").write_text(
        json.dumps(asdict(estado), ensure_ascii=False, indent=2))
```

Na nomenclatura da literatura, o que essas quatro linhas persistem é a **memória de curto prazo** do agente — o estado da execução corrente, e nada além dela. A [nota 02](02-checkpoint-nao-e-memoria.md), §2, situa o termo.

O `asdict` só é possível porque o estado é uma `dataclass` (§3). Com a execução representada em `mensagens[]`, não há o que serializar: o que existe é uma lista de textos, sem `termino`, sem `pendencia`, sem `passos` com argumentos separados.

#### 8.1 O que ele destrava

| Capacidade | O que o arquivo permite |
|---|---|
| **Confirmação assíncrona** | o processo encerra; a aprovação chega horas depois, de outro processo, e a execução prossegue do ponto de suspensão |
| **Retomada após queda** | o processo caiu no passo 9 de uma trajetória cara; a retomada não repete os 9 passos **nem seus efeitos colaterais** |
| **Depuração por reprodução** | carregar o estado de uma execução malsucedida e prosseguir a partir dele, com registro detalhado ativo |

A primeira é a que sustenta a §7. Sem o arquivo, "aprovar" significaria reexecutar a tarefa do zero — e a aprovação do humano recairia sobre uma execução **diferente** daquela que ele analisou.

#### 8.2 A armadilha: depois de executar, nunca antes

Se o *checkpoint* registrar a **intenção** e o processo encerrar entre o registro e a execução, a retomada reexecuta a ação. Quando a ação é escrita, ela ocorre duas vezes.

> A ordem correta é **executar, depois persistir**. A fresta que ainda assim resta — o processo cai entre a execução e o registro — é fechada pela **chave de idempotência**: o identificador da escrita é derivado do seu conteúdo, e não da tentativa, de modo que a reexecução encontra o registro anterior em vez de criar um segundo. As duas salvaguardas operam em conjunto, e nenhuma é suficiente isoladamente.

O campo `pendencia` é a exceção aparente: ele registra uma escrita que **ainda não ocorreu**. A exceção é segura porque a ação está bloqueada aguardando aprovação, e não em curso — o que o arquivo registra não é "executei", e sim "parei aqui, aguardando".

#### 8.3 O que ele não resolve

O arquivo descreve **uma** execução, íntegra, e serve para retomá-la. Diante de uma tarefa nova, a pergunta muda:

> *"o que já se aprendeu nas execuções anteriores que ajude nesta?"*

O *checkpoint* não responde — não por estar incompleto, mas porque **não há por onde perguntar a ele**. Serializar todas as execuções não altera isso: multiplica o arquivo sem criar a consulta.

É a pergunta que abre a [nota 02](02-checkpoint-nao-e-memoria.md), e a fronteira entre esta aula e a anterior.

---

## Exemplos

### Exemplo 1 — A mesma execução sob as duas estruturas

Agente investigando uma despesa, em quatro passos, com falha no terceiro.

**Com `mensagens[]`**, o resultado final são nove dicionários. A constatação de que houve erro exige a leitura do campo `content` da sétima entrada e a identificação de uma chave `erro` no JSON.

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

A segunda estrutura responde, sem análise sintática: quantos passos, qual o custo, o que falhou, que o modelo **se corrigiu autonomamente** (`F-88` → `F-088`) e por que encerrou. É o objeto que a aula de observabilidade denominará *trace*.

### Exemplo 2 — Três consultas, uma linha cada

Operações que o estado torna triviais e que a lista de mensagens converteria em projeto:

```python
sum(1 for p in estado.passos if p.erro)                          # quantos erros?
Counter((p.ferramenta, str(p.argumentos)) for p in estado.passos) # chamada repetida?
[p.ferramenta for p in estado.passos if p.tokens > 5_000]         # maior consumo?
```

Nenhuma delas é expressável sobre `mensagens[]` sem varredura e reanálise de JSON a cada volta.

### Exemplo 3 — A ferramenta indisponível

```python
estado.ferramentas_ativas = FASES["analise"]   # registrar_parecer NÃO declarada

# Tentativa de registro pelo modelo: inviável, a ferramenta não consta da
# requisição. O comportamento máximo é DECLARAR que registrou — e a ausência
# do Passo correspondente evidencia a alucinação:
assert not any(p.ferramenta == "registrar_parecer" for p in estado.passos)
```

O `assert` acima constitui teste efetivo, e só é redigível porque o estado armazena os passos como dados. Sobre a lista de mensagens, o teste equivalente seria uma busca por subcadeia.

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
    estado.termino, estado.pendencia = None, None    # prossegue de onde parou
    return continuar(estado)
```

Quatro linhas de serialização separam "o agente pausou" de "o agente encerrou". Note-se o que **não** consta do trecho: nenhum passo é reexecutado — os já executados estão em `estado.passos`, com resultado.

O [`00-o-checkpoint.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula08-memoria/00-o-checkpoint.py) executa essa sequência e imprime o arquivo entre as duas metades. A execução para no passo 2 com `termino: HUMANO` e a pendência registrada, grava, e retoma:

```
  carregado de exec-0a1f.json: 2 passos já executados,
  1150 tokens já gastos, e uma pendência aguardando.

  aprovado -> executa o passo 2, e só ele:
      registrar_parecer({'despesa': 'D-4102', 'veredito': 'aprovado'})
          -> {'parecer': 'P-0993'}

  término: CONCLUIDO
  passos:  2 antes + 1 agora = 3
```

Os 1.150 tokens dos dois primeiros passos não são gastos de novo, e — o que importa mais — nenhuma consulta é repetida e nenhuma escrita é duplicada.

---

## Fontes e leituras

ANTHROPIC. **Building effective agents**. 2024. (Seção sobre agentes autônomos e sobre a exigência de ambientes que devolvam sinal.)

OPENAI. **A practical guide to building agents**. 2025. (Capítulos de orquestração e de *human-in-the-loop*.)

YAO, S. et al. **ReAct**: synergizing reasoning and acting in language models. arXiv:2210.03629, 2022.

**Material da disciplina.** Aula 01, [nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) §1.2, §3 e §4 — os quatro componentes, projeto de ferramentas e a tabela de modos de falha. Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — o laço original que esta nota reescreve.
