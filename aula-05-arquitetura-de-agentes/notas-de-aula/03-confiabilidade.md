# IA Aplicada com LLMs — Aula 05: Arquitetura de agentes — Confiabilidade

## Introdução

A Aula 01 (nota 03, §4) apresentou uma tabela com oito modos de falha de agentes. Naquele momento tratava-se de um mapa: nenhum agente havia sido implementado, e as salvaguardas eram nomes. Esta nota converte cada linha em código.

| Falha | Manifestação | Tratada em |
|---|---|---|
| **Laço** | repetição indefinida da mesma chamada | §6 |
| **Deriva de objetivo** | afastamento do pedido em trajetória longa | §7 |
| **Alucinação de argumento** | invenção de identificador plausível e inexistente | §3 e §4 |
| **Ferramenta errada** | seleção de escrita onde caberia leitura | [nota 02](02-o-agente-e-o-estado.md) §6 |
| **Contexto estourado** | a trajetória excede a janela | [nota 04](04-context-engineering-dinamica.md) |
| **Erro em cascata** | uma observação incorreta contamina o restante | §3 e §8 |
| **Custo descontrolado** | consumo em ordem de grandeza acima do previsto | §1 |
| **Silêncio** | falha sem causa identificável | §2 e §10 |

Todo o conteúdo desta nota depende do objeto de estado da [nota 02](02-o-agente-e-o-estado.md). A dependência não é conceitual: os campos empregados aqui **não existem** no laço da Aula 03.

> **Nenhuma salvaguarda é gratuita.** Orçamento restritivo interrompe tarefa legítima. Detector agressivo interrompe agente em progresso lento. Confirmação em excesso suprime a autonomia que justificava o agente. Cada seção apresenta o custo junto com o mecanismo; a calibragem da dose é trabalho de engenharia, não aplicação de receita.

> **Pré-requisitos:** notas [01](01-padroes-de-arquitetura.md) e [02](02-o-agente-e-o-estado.md) desta aula · Aula 01, [nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) §3 e §4 · Aula 02, [nota 01](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) §8.2 e [nota 04](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md).
>
> **Código:** [`04-orcamento-e-terminacao.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/04-orcamento-e-terminacao.py) e [`05-erros-e-laco.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/05-erros-e-laco.py).

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Impor** orçamento em quatro moedas — passos, tokens, unidade monetária e tempo de parede — e justificar cada teto.
- **Implementar** as quatro formas de terminação de um agente, registrando qual ocorreu.
- **Classificar** um erro de ferramenta em recuperável ou fatal e tratar cada classe distintamente.
- **Redigir** um retorno de erro que induza correção pelo modelo, em vez de repetição da falha.
- **Distinguir** o que admite retentativa automática do que só o modelo resolve.
- **Detectar** laço por repetição de chamada e por ausência de progresso.
- **Proteger** ferramentas de escrita com chave de idempotência e **retomar** a execução a partir de *checkpoint*.

---

## Desenvolvimento teórico

### 1. Orçamento em quatro moedas

`max_passos=6` é necessário e é o mais fraco dos quatro tetos, porque **passo não é unidade de custo**. Um passo que consulta um identificador consome cerca de 300 tokens; um passo que lê um documento de 30 páginas consome cerca de 40.000. "No máximo 10 passos" equivale a "entre 3 mil e 400 mil tokens", o que não constitui limite.

```python
@dataclass
class Orcamento:
    max_passos: int = 12
    max_tokens: int = 60_000
    max_reais: float = 0.50
    max_segundos: float = 120.0
    inicio: float = field(default_factory=time.monotonic)

    def excedido(self, estado: Estado) -> str | None:
        """Devolve QUAL teto foi excedido — ou None. O motivo é relevante."""
        if estado.n_passos >= self.max_passos:
            return f"passos: {estado.n_passos}/{self.max_passos}"
        if estado.tokens_gastos >= self.max_tokens:
            return f"tokens: {estado.tokens_gastos}/{self.max_tokens}"
        if estado.custo_estimado >= self.max_reais:
            return f"custo: R$ {estado.custo_estimado:.4f}/{self.max_reais}"
        if time.monotonic() - self.inicio >= self.max_segundos:
            return f"tempo: {time.monotonic() - self.inicio:.0f}s"
        return None
```

| Decisão de implementação | Justificativa |
|---|---|
| **Orçamento é parâmetro**, não constante | tarefas distintas admitem tetos distintos, e constante no interior do arquivo não é ajustada. Triagem simples e investigação de divergência não têm o mesmo direito de consumo |
| **Custo monetário** provém da Aula 02 (nota 04) | `entrada × preço_in + saída × preço_out`, acumulado no estado. É a unidade compreendida pela gestão |
| **Tempo de parede** é o teto usualmente omitido | cobre o caso que os outros três não cobrem: a ferramenta bloqueada aguardando resposta. Sem ele, o agente não excede nenhum limite — apenas não termina |
| **A função devolve *qual* teto foi excedido** | um retorno booleano seria mais simples e inútil: encerramento por tempo e por tokens exigem diagnósticos opostos |

> **Custo da salvaguarda:** todo teto produz falso positivo — tarefa legítima e difícil encerra sem resposta. Daí o registro do motivo: ele permite distinguir "o agente falhou" de "o orçamento estava subdimensionado", que demandam correções opostas.

---

### 2. As quatro formas de terminação

```
   1. RESPONDEU     o modelo devolveu sem tool_calls      sucesso
   2. ORCAMENTO     um dos quatro tetos foi excedido      inconcluso
   3. ERRO_FATAL    não há como prosseguir                falha
   4. HUMANO        execução suspensa aguardando          suspenso
                    confirmação
```

O laço da Aula 03 tratava a primeira com `return` e a segunda com `raise RuntimeError`. A terceira derrubava o programa por exceção não tratada, e a quarta não existia. Com o objeto de estado, o término converte-se em **dado**:

```python
def rodar(objetivo: str, orcamento: Orcamento) -> Estado:
    estado = Estado(objetivo=objetivo, ferramentas_ativas=FASES["coleta"])
    try:
        while True:
            if (motivo := orcamento.excedido(estado)):
                estado.termino, estado.motivo = Termino.ORCAMENTO, motivo
                return estado
            ...
    except PausaParaHumano:
        return estado                       # termino já marcado como HUMANO
    except ErroFatal as e:
        estado.termino, estado.motivo = Termino.ERRO_FATAL, str(e)
        return estado
    finally:
        registrar_trace(estado)             # sempre, inclusive nas falhas
```

O bloco `finally` é essencial: o *trace* é persistido **em todos os caminhos**, e é nos caminhos de falha que ele tem maior valor. Um tratamento que registre apenas o sucesso assegura que a causa do problema jamais será identificada.

> A linha "silêncio" da tabela de modos de falha é eliminada aqui. Um agente que sempre declara a causa do encerramento e sempre persiste a trajetória continuará falhando, mas a localização da falha será conhecida.

---

### 3. Erro de ferramenta: recuperável × fatal

Esta é a decisão de projeto mais consequente da nota, e ela cabe **ao código**, não ao modelo.

```
   ferramenta levantou exceção
              │
     ┌────────┴─────────┐
  RECUPERÁVEL         FATAL
     │                  │
  vira mensagem       aborta o laço
  role="tool"         Termino.ERRO_FATAL
     │
  o modelo repete a tentativa, com informação nova
```

**Recuperável** é o erro contornável por alteração da chamada: argumento inválido, registro inexistente, formato incorreto, resultado vazio. **Fatal** é o erro que nenhuma decisão do modelo resolve: credencial, indisponibilidade de banco de dados, disco cheio. Devolver um erro fatal ao modelo produz sucessivas reformulações da chamada até a exaustão do orçamento.

```python
class ErroRecuperavel(Exception):
    """Contornável por alteração da chamada."""
    def __init__(self, mensagem, **contexto):
        self.payload = {"erro": mensagem, **contexto}

class ErroFatal(Exception):
    """Nenhuma decisão do modelo resolve. Aborta."""

def buscar_despesa(id: str) -> dict:
    if not RE_ID.match(id):
        raise ErroRecuperavel("formato de id inválido", esperado="D-9999",
                              recebido=id, sugestao="use listar_despesas")
    try:
        registro = BANCO.get(id)
    except ConnectionError as e:
        raise ErroFatal(f"banco indisponível: {e}") from e   # fora do alcance do modelo
    if registro is None:
        raise ErroRecuperavel("despesa não encontrada", id=id,
                              sugestao="confira o id com listar_despesas")
    return registro
```

```python
def executar(chamada, estado: Estado) -> Passo:
    nome, argumentos = chamada.function.name, json.loads(chamada.function.arguments)
    passo = Passo(indice=estado.n_passos, ferramenta=nome, argumentos=argumentos)
    try:
        passo.resultado = FERRAMENTAS[nome](**argumentos)
    except ErroRecuperavel as e:
        passo.erro, passo.resultado = e.payload["erro"], e.payload   # retorna ao modelo
    except TypeError as e:                    # parâmetro inexistente na função
        passo.erro = str(e)
        passo.resultado = {"erro": "argumentos inválidos", "detalhe": str(e)}
    return passo
```

O tratamento de `TypeError` merece registro: o modelo eventualmente invoca a função com um parâmetro que ela não declara. Sem esse tratamento, uma alucinação de argumento derruba o processo; com ele, converte-se em observação, e o modelo corrige na volta seguinte.

> **Custo da salvaguarda:** classificar como fatal o que era recuperável encerra trajetórias recuperáveis; o inverso consome o orçamento integralmente contra uma condição irreversível. Em caso de dúvida, examina-se a origem: se está no **argumento**, é recuperável; se está na **infraestrutura**, é fatal.

---

### 4. O retorno de erro como prompt

A Aula 03 (nota 04, §4) estabeleceu que a descrição da ferramenta é prompt. O corolário raramente é enunciado:

> **A mensagem de erro também é prompt.** É o único texto que o modelo lê para decidir como se corrigir.

```python
{"erro": "falhou"}                                             # inútil

{"erro": "ValueError: invalid literal for int(): 'D-4471'"}    # verdadeiro,
                                                               # e não acionável
{"erro": "formato de id inválido",                             # útil
 "esperado": "D seguido de 4 dígitos, ex: D-4471",
 "recebido": "4471",
 "sugestao": "chame listar_despesas para obter os ids válidos"}
```

O terceiro retorno é eficaz porque responde às três perguntas necessárias à ação: **o que estava incorreto**, **qual seria o correto** e **qual a próxima ação**. Os dois primeiros deixam a inferência a cargo do modelo, e inferência consome passos.

Consequência prática: os retornos de erro devem ser revisados sobre trajetórias reais. Quando o agente se desorganiza após um erro, o defeito reside quase sempre no texto devolvido, não no modelo. O caso de maior valor é o do erro que **indica o caminho** — a chave `sugestao` apontando outra ferramenta converte um impasse em passo produtivo.

---

### 5. Retentativa, e onde ela não se aplica

Num agente coexistem **dois** mecanismos de recuperação, cuja confusão é onerosa:

| Tipo de falha | Responsável | Mecanismo |
|---|---|---|
| `429`, `500`, tempo esgotado de rede | **o código** | *backoff* exponencial, transparente ao modelo |
| argumento inválido, registro inexistente | **o modelo** | o erro retorna como observação; a chamada é alterada |
| credencial, serviço indisponível | nenhum | `ErroFatal`, aborta |

```python
def chamar_com_retry(**kwargs):
    for tentativa in range(5):
        try:
            return client.chat.completions.create(**kwargs)
        except (RateLimitError, APIConnectionError):
            time.sleep(2 ** tentativa)          # 1, 2, 4, 8, 16
    raise ErroFatal("API indisponível após 5 tentativas")
```

```python
# INADEQUADO: repetir a chamada de ferramenta que falhou por argumento
for tentativa in range(3):
    try:
        return FERRAMENTAS[nome](**argumentos)
    except ErroRecuperavel:
        continue          # os argumentos são idênticos. O erro será idêntico.
```

Repetir uma chamada determinística com os mesmos argumentos constitui retentativa sem informação nova. Nesse caso, quem repete a tentativa é **o modelo**, e apenas porque recebeu a observação do erro.

> **Regra:** retentativa automática destina-se a falha de **transporte**. Falha de **conteúdo** retorna ao modelo.

---

### 6. Detecção de laço e de progresso nulo

`max_passos` limita o dano do laço; não o detecta. Com teto de 12, um agente bloqueado na primeira volta consome 12 passos antes de encerrar, e a mensagem produzida é "orçamento esgotado" — diagnóstico incorreto para o problema efetivo.

**Laço estrito** é a repetição da mesma ferramenta com os mesmos argumentos:

```python
def assinatura(passo: Passo) -> tuple:
    return (passo.ferramenta, json.dumps(passo.argumentos, sort_keys=True))

def detectar_laco(estado: Estado, limite: int = 3) -> bool:
    if estado.n_passos < limite:
        return False
    return len({assinatura(p) for p in estado.passos[-limite:]}) == 1
```

O parâmetro `sort_keys=True` não é detalhe: sem ele, `{"id": "D-1", "ano": 2026}` e `{"ano": 2026, "id": "D-1"}` produzem assinaturas distintas, e o detector não detecta nada.

**Progresso nulo** é a variante sutil e mais frequente: as chamadas *variam*, mas o estado não avança — política, histórico, política novamente, despesa, política. Cada chamada difere da anterior; o conjunto se repete.

```python
def sem_progresso(estado: Estado, janela: int = 6) -> bool:
    """Nas últimas `janela` chamadas, nenhuma trouxe informação nova."""
    if estado.n_passos < janela:
        return False
    recentes = estado.passos[-janela:]
    distintas = {assinatura(p) for p in recentes}
    # muitas chamadas, poucas distintas, nenhuma escrita: a execução gira
    return len(distintas) <= janela // 2 and not any(
        p.ferramenta in FERRAMENTAS_DE_ESCRITA for p in recentes)
```

Detectada a condição, há três respostas, da mais branda à mais severa:

| Resposta | Ação | Aplicação |
|---|---|---|
| **Injetar observação** | *"a ferramenta X já foi chamada com estes argumentos e devolveu este resultado. Utilize a informação disponível ou declare o que falta."* | primeira detecção — é a intervenção mais barata e usualmente suficiente |
| **Reduzir ferramentas ativas** | remover a ferramenta repetida força outro caminho | a observação não surtiu efeito |
| **Abortar** | `Termino.ERRO_FATAL`, motivo `laco_detectado` | diagnóstico correto e custo encerrado |

> **Custo da salvaguarda:** detector agressivo interrompe agente em progresso lento. Tarefa que legitimamente consulta a mesma ferramenta com argumentos semelhantes — varredura de lista, por exemplo — produz falso positivo. A calibragem do limite deve ser feita sobre trajetórias reais, partindo de valores permissivos.

---

### 7. Deriva de objetivo

Numa trajetória de trinta passos, o pedido original está distante e cercado de observações — precisamente a posição em que o modelo recupera pior a informação (*lost in the middle*, LIU et al., 2023, retomado na Aula 02, nota 01, §4.3). A manifestação típica: o agente resolve com esmero um subproblema não solicitado.

```python
REANCORAR_A_CADA = 5

if estado.n_passos and estado.n_passos % REANCORAR_A_CADA == 0:
    msgs.append({"role": "user",
                 "content": f"Lembrete do objetivo: {estado.objetivo}"})
```

O custo é de algumas dezenas de tokens a cada cinco passos, o que a torna a intervenção de melhor relação custo-benefício desta nota. Só é viável porque `objetivo` é campo do estado, e não mensagem passível de ser absorvida pela *compaction* ([nota 02](02-o-agente-e-o-estado.md) §3).

---

### 8. Idempotência: de prudência a necessidade

A Aula 01 (nota 03, §3) já prescrevia chave de idempotência em ferramenta de escrita, como prudência. Passa a ser necessidade por um motivo específico desta nota: **as salvaguardas introduzidas aqui aumentam a probabilidade de repetição de escrita.**

- a **retentativa** de rede reenvia requisição cujo efeito já ocorreu — o que se perdeu foi a resposta;
- o **checkpoint** (§9) retoma de um estado persistido, e o último passo pode ter executado sem ter sido registrado;
- a **detecção de laço**, ao injetar observação e permitir a continuação, pode induzir o agente a repetir a ação que supôs malsucedida.

```python
def registrar_parecer(despesa_id: str, veredito: str, justificativa: str,
                      chave: str) -> dict:
    """`chave` identifica a operação, não a tentativa."""
    if (existente := PARECERES.get(chave)):
        return {**existente, "ja_existia": True}     # idempotente: mesma resposta
    parecer = {"id": novo_id(), "despesa_id": despesa_id,
               "veredito": veredito, "justificativa": justificativa}
    PARECERES[chave] = parecer
    return parecer
```

A chave precisa ser **determinística e derivada do conteúdo** — `f"{despesa_id}:{veredito}"` ou um resumo criptográfico dos campos. Chave gerada por `uuid4()` a cada chamada não é chave de idempotência: é identificador novo por tentativa, precisamente o que se pretende evitar.

O retorno inclui `ja_existia: True`. Trata-se de informação destinada ao **modelo**: ele constata que a ação já ocorreu e não a repete. Escrita idempotente que devolve resposta idêntica oculta do agente o fato de que houve repetição.

---

### 9. Checkpoint

A implementação consta da [nota 02, exemplo 4](02-o-agente-e-o-estado.md) — quatro linhas de serialização sobre o estado. O mecanismo viabiliza:

1. **Confirmação humana assíncrona** — o agente suspende a execução, persiste o estado e devolve o controle; a aprovação ocorre horas depois e a execução prossegue do ponto de suspensão.
2. **Retomada após queda** — o processo encerrou no passo 9 de uma trajetória custosa; a retomada não repete os 9 passos nem seus efeitos colaterais.
3. **Depuração por reprodução** — carregar o estado de uma execução malsucedida e prosseguir a partir dele, com registro detalhado ativo.

A armadilha: **persistir depois de executar, nunca antes**. Se o *checkpoint* registrar a intenção e o processo encerrar entre o registro e a execução, a retomada reexecuta a ação. É a idempotência da §8 que fecha essa lacuna — as duas salvaguardas operam em conjunto, e nenhuma é suficiente isoladamente.

> Persistir estado **entre execuções distintas** — o agente recuperar o que ocorreu na semana anterior — é objeto da aula de memória. O *checkpoint* opera no interior de uma única execução.

---

### 10. O trace como subproduto

Cabe registrar o que foi construído sem que constituísse requisito. Para detectar laço, foi necessário armazenar ferramenta e argumentos de cada passo. Para o orçamento, tokens e custo. Para o término, o motivo. O conjunto constitui, por execução, um registro estruturado do que o agente executou, quanto consumiu, o que falhou e por que encerrou.

Esse registro denomina-se **trace**, e é a matéria-prima de duas aulas subsequentes: **observabilidade** — agregação de *traces* para responder qual ferramenta falha mais e qual o custo médio por tarefa — e **evals** — sem trajetória registrada, avalia-se apenas a resposta final; com ela, avalia-se **o percurso**, onde reside a maioria dos defeitos.

> **Não se avalia o que não se rastreia.** É a razão pela qual o objeto de estado precede todo o restante.

---

## Exemplos

### Exemplo 1 — A mesma trajetória, com e sem erro informativo

Despesa cujo identificador foi digitado incorretamente. Orçamento de 12 passos.

```
SEM detector e com erro não informativo ({"erro": "falhou"}):
  passo  0..11  consultar_historico {"funcionario": "F-88"} -> erro
  TERMINO: orcamento_esgotado (passos: 12/12) | 14.200 tokens | R$ 0,036
```

```
COM erro informativo:
  passo  0  consultar_historico {"funcionario": "F-88"}
            -> {"erro": "funcionário não encontrado", "recebido": "F-88",
                "esperado": "F seguido de 3 dígitos, ex: F-088",
                "sugestao": "chame listar_funcionarios"}
  passo  1  consultar_historico {"funcionario": "F-088"}  -> ok
  passo  2  consultar_politica  {"categoria": "refeicao"} -> ok
  TERMINO: respondeu | 1.480 tokens | R$ 0,004
```

A redução é de nove vezes em tokens — e **o detector de laço não chegou a ser acionado**. A resolução decorreu do texto do erro. O detector é a rede de segurança para os casos em que o texto não basta, e a maior parte do ganho está em não depender dele.

### Exemplo 2 — Progresso nulo: o laço não capturado pelo detector estrito

```
passo 3  consultar_politica   {"categoria": "refeicao"}
passo 4  consultar_historico  {"funcionario": "F-088"}
passo 5  consultar_politica   {"categoria": "refeicao"}     <- repetida
passo 6  buscar_despesa       {"id": "D-4471"}
passo 7  consultar_politica   {"categoria": "refeicao"}     <- repetida
passo 8  consultar_historico  {"funcionario": "F-088"}      <- repetida
```

`detectar_laco` não é acionado: não há três assinaturas idênticas em sequência. `sem_progresso` é acionado: seis chamadas, **três** assinaturas distintas, nenhuma escrita. O agente dispõe de toda a informação e não conclui — o problema não é ausência de dado, e sim vagueza do critério de conclusão no *system prompt*.

### Exemplo 3 — As quatro terminações no registro

```
exec 3f9a1c  termino=respondeu           passos=4   tokens=2.180  R$ 0,006
exec 7b2e04  termino=orcamento_esgotado  passos=12  tokens=14.200 R$ 0,036
              motivo="passos: 12/12"
exec c81d55  termino=erro_fatal          passos=2   tokens=780    R$ 0,002
              motivo="banco indisponível: connection refused"
exec 09fa7e  termino=aguardando_humano   passos=6   tokens=3.940  R$ 0,010
              pendencia={"ferramenta": "registrar_parecer", "argumentos": {...}}
```

Quatro execuções, quatro diagnósticos e três ações operacionais distintas: a segunda demanda revisão do orçamento ou do prompt; a terceira, comunicação à infraestrutura; a quarta, aprovação humana. Sem o campo `termino`, as quatro seriam reportadas como falha genérica.

### Exemplo 4 — Idempotência impedindo parecer duplicado

```python
# Passo 7: o agente registra. A rede falha APÓS a gravação, antes da resposta.
# A retentativa reenvia a requisição.
chave = "D-4471:aprovado"

registrar_parecer("D-4471", "aprovado", "dentro do teto", chave=chave)
# -> {"id": "P-1001", "despesa_id": "D-4471", ...}

registrar_parecer("D-4471", "aprovado", "dentro do teto", chave=chave)
# -> {"id": "P-1001", ..., "ja_existia": True}    mesmo parecer, não um novo
```

Sem a chave, o segundo registro cria `P-1002`, e a auditoria posterior encontra dois pareceres idênticos para a mesma despesa, sem critério de desempate.

---

## Fontes e leituras

ANTHROPIC. **Building effective agents**. 2024. (Seção sobre *guardrails* e limites de execução.)

LIU, N. F. et al. **Lost in the middle**: how language models use long contexts. arXiv:2307.03172, 2023.

OPENAI. **A practical guide to building agents**. 2025. (Capítulo de *guardrails* e tratamento de exceções.)

OWASP. **Agentic Security Initiative**: catálogo de riscos específicos de agentes. (Citado como referência; o tratamento é objeto da aula de segurança.)

**Material da disciplina.** Aula 01, [nota 03](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) §3 e §4 — a tabela de modos de falha que organiza esta nota e a idempotência. Aula 02, [nota 01](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) §8.2 — *backoff* exponencial; [nota 04](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md) — a conta por chamada, que fundamenta o teto monetário. Aula 03, [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) §4 e §7 — a descrição da ferramenta como prompt, aqui estendida ao retorno de erro.
