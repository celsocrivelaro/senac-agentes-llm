# IA Aplicada com LLMs — Aula 05: Arquitetura de agentes — Confiabilidade

## Introdução

A Aula 01 (nota 03, §4) trouxe uma tabela com oito modos de falha de agentes. Naquele momento era um mapa: você ainda não tinha escrito um agente, e as salvaguardas eram nomes. Esta nota transforma cada linha em código.

| Falha | Como aparece | Resolvida em |
|---|---|---|
| **Laço** | repete a mesma chamada indefinidamente | §6 |
| **Deriva de objetivo** | numa trajetória longa, se afasta do pedido | §7 |
| **Alucinação de argumento** | inventa um id plausível que não existe | §3 e §4 |
| **Ferramenta errada** | escolhe escrever quando devia ler | [nota 02 §6](02-o-agente-e-o-estado.md) |
| **Contexto estourado** | a trajetória não cabe mais na janela | [nota 04](04-context-engineering-dinamica.md) |
| **Erro em cascata** | uma observação errada contamina o resto | §3 e §8 |
| **Custo descontrolado** | consome mil vezes o previsto | §1 |
| **Silêncio** | falhou e você não sabe por quê | §2 e §10 |

Tudo aqui depende do objeto de estado da [nota 02](02-o-agente-e-o-estado.md) — não conceitualmente: os campos usados nesta nota **não existem** no laço da Aula 03.

> **Nenhuma salvaguarda é grátis.** Orçamento apertado mata tarefa legítima. Detector agressivo interrompe agente que progredia devagar. Confirmação em excesso mata a autonomia que justificava o agente. Cada seção traz o preço junto com o remédio — escolher a dose é engenharia, não receita.

> **Pré-requisitos:** notas [01](01-padroes-de-arquitetura.md) e [02](02-o-agente-e-o-estado.md) desta aula · Aula 01, [nota 03 §3 e §4](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) · Aula 02, [nota 01 §8.2](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) e [nota 04](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md).
>
> **Código:** [`04-orcamento-e-terminacao.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/04-orcamento-e-terminacao.py) e [`05-erros-e-laco.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/05-erros-e-laco.py) — o script central da aula.

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Impor** um orçamento em quatro moedas — passos, tokens, dinheiro e tempo — e justificar cada teto.
- **Implementar** as quatro formas de terminar um agente, registrando qual ocorreu.
- **Classificar** um erro de ferramenta em recuperável ou fatal, e tratar cada classe de forma diferente.
- **Escrever** um retorno de erro que faz o modelo se corrigir em vez de repetir a falha.
- **Distinguir** o que merece retry automático do que só o modelo resolve.
- **Detectar** laço por repetição de chamada e por ausência de progresso.
- **Proteger** ferramentas de escrita com chave de idempotência, e **retomar** de um checkpoint.

---

## Desenvolvimento teórico

### 1. Orçamento em quatro moedas

`max_passos=6` é necessário e é o mais fraco dos quatro tetos, porque **passo não é unidade de custo**. Um passo que consulta um id custa 300 tokens; um que lê um documento de 30 páginas custa 40.000. "No máximo 10 passos" significa "entre 3 mil e 400 mil tokens" — o que não é um limite.

```python
@dataclass
class Orcamento:
    max_passos: int = 12
    max_tokens: int = 60_000
    max_reais: float = 0.50
    max_segundos: float = 120.0
    inicio: float = field(default_factory=time.monotonic)

    def excedido(self, estado: Estado) -> str | None:
        """Devolve QUAL teto estourou — ou None. O motivo importa."""
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

| Decisão no código | Por quê |
|---|---|
| **Orçamento é parâmetro**, não constante | tarefas diferentes merecem tetos diferentes, e número mágico no meio do arquivo ninguém ajusta. Triagem simples e investigação de divergência não têm o mesmo direito de gastar |
| **Custo em reais** vem da Aula 02 (nota 04) | `entrada × preço_in + saída × preço_out`, acumulado no estado. É a moeda que o seu chefe entende |
| **Tempo de parede** é o teto esquecido | pega o caso que os outros três não pegam: a ferramenta travada esperando resposta que não vem. Sem ele o agente não estoura nada — só não termina |
| **Devolve *qual* teto estourou** | `bool` seria mais simples e inútil: morrer por tempo e morrer por tokens têm diagnósticos opostos |

> **O preço:** todo teto tem falso positivo — tarefa legítima e difícil morre sem resposta. Por isso o motivo é registrado: para distinguir "o agente falhou" de "o orçamento estava curto", que exigem correções opostas.

---

### 2. As quatro formas de terminar

```
   1. RESPONDEU     o modelo devolveu sem tool_calls      ✓ sucesso
   2. ORCAMENTO     estourou um dos quatro tetos          ✗ inconcluso
   3. ERRO_FATAL    não há como continuar                 ✗ falha
   4. HUMANO        pausou aguardando confirmação         ⏸ suspenso
```

O laço da Aula 03 tratava a 1 com `return` e a 2 com `raise RuntimeError`. A 3 derrubava o programa por exceção não tratada, e a 4 não existia. Com o estado, o término vira **dado**:

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

Repare no `finally`: o trace é gravado **em todos os caminhos**, e é nos de falha que ele vale mais. O `except` que registra só o sucesso é o que garante que você nunca vai descobrir a causa.

> **A linha "silêncio" da tabela de falhas morre aqui.** Um agente que sempre diz por que parou e sempre grava a trajetória vai falhar — mas você saberá onde.

---

### 3. Erro de ferramenta: recuperável × fatal

A decisão de projeto mais importante da nota, e ela é **do seu código**, não do modelo.

```
   ferramenta levantou exceção
              │
     ┌────────┴─────────┐
  RECUPERÁVEL         FATAL
     │                  │
  vira mensagem       aborta o laço
  role="tool"         Termino.ERRO_FATAL
     │
  o modelo tenta de novo, com informação nova
```

**Recuperável** é o que o modelo contorna mudando o que faz: argumento inválido, registro inexistente, formato errado, resultado vazio. **Fatal** é o que nenhuma decisão do modelo resolve: credencial, banco fora do ar, disco cheio — devolver isso ao modelo produz quatro reformulações educadas até o orçamento acabar.

```python
class ErroRecuperavel(Exception):
    """O modelo pode contornar mudando a chamada."""
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
        raise ErroFatal(f"banco indisponível: {e}") from e   # o modelo não resolve
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
        passo.erro, passo.resultado = e.payload["erro"], e.payload   # volta ao modelo
    except TypeError as e:                    # parâmetro que a função não tem
        passo.erro = str(e)
        passo.resultado = {"erro": "argumentos inválidos", "detalhe": str(e)}
    return passo
```

O `except TypeError` merece uma linha: o modelo às vezes inventa um parâmetro. Sem esse tratamento, uma alucinação de argumento derruba o processo inteiro; com ele, vira observação e o modelo corrige na volta seguinte.

> **O preço:** marcar como fatal o que era recuperável mata trajetórias que se salvariam; o inverso queima o orçamento contra uma parede. Na dúvida, olhe a causa — se a origem é o **argumento**, é recuperável; se é a **infraestrutura**, é fatal.

---

### 4. O retorno de erro é prompt

A Aula 03 (nota 04, §4) estabeleceu que a descrição da ferramenta é prompt. O corolário quase nunca é dito:

> **A mensagem de erro também é prompt.** É o único texto que o modelo lê para decidir como se corrigir.

```python
{"erro": "falhou"}                                             # inútil

{"erro": "ValueError: invalid literal for int(): 'D-4471'"}    # verdadeiro,
                                                               # e sem ação possível
{"erro": "formato de id inválido",                             # útil
 "esperado": "D seguido de 4 dígitos, ex: D-4471",
 "recebido": "4471",
 "sugestao": "chame listar_despesas para obter os ids válidos"}
```

O terceiro funciona porque responde às três perguntas que o modelo precisa responder para agir: **o que estava errado**, **qual era o certo**, **o que fazer agora**. Os dois primeiros deixam adivinhar, e adivinhação custa passos.

Consequência prática: revise os retornos de erro olhando trajetórias reais. Quando o agente se perde depois de um erro, o defeito quase sempre está no texto que você devolveu, não no modelo. O caso mais valioso é o erro que **ensina o caminho** — a `sugestao` apontando outra ferramenta transforma beco sem saída em passo produtivo.

---

### 5. Retry, e onde ele não vale

Num agente existem **dois** mecanismos de recuperação, e confundi-los é caro:

| Tipo de falha | Quem resolve | Como |
|---|---|---|
| `429`, `500`, timeout de rede | **o seu código** | backoff exponencial, transparente para o modelo |
| argumento inválido, registro inexistente | **o modelo** | erro volta como observação; ele muda a chamada |
| credencial, serviço fora | **ninguém** | `ErroFatal`, aborta |

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
# ERRADO: repetir a mesma chamada de ferramenta que falhou por argumento
for tentativa in range(3):
    try:
        return FERRAMENTAS[nome](**argumentos)
    except ErroRecuperavel:
        continue          # os argumentos são os mesmos. O erro será o mesmo.
```

Repetir uma chamada determinística com os mesmos argumentos é retry sem informação nova. Quem "tenta de novo" nesse caso é **o modelo**, e ele só consegue porque recebeu a observação do erro.

> **Regra de bolso:** retry automático é para falha de **transporte**. Falha de **conteúdo** volta para o modelo.

---

### 6. Detecção de laço e de progresso nulo

`max_passos` limita o dano do laço; não detecta o laço. Com teto de 12, um agente preso na primeira volta gasta 12 passos antes de morrer, e a mensagem que chega é "orçamento esgotado" — diagnóstico errado para o problema certo.

**Laço estrito** — a mesma ferramenta com os mesmos argumentos:

```python
def assinatura(passo: Passo) -> tuple:
    return (passo.ferramenta, json.dumps(passo.argumentos, sort_keys=True))

def detectar_laco(estado: Estado, limite: int = 3) -> bool:
    if estado.n_passos < limite:
        return False
    return len({assinatura(p) for p in estado.passos[-limite:]}) == 1
```

O `sort_keys=True` não é detalhe: sem ele `{"id": "D-1", "ano": 2026}` e `{"ano": 2026, "id": "D-1"}` são assinaturas diferentes, e o detector não detecta nada.

**Progresso nulo** é a versão sutil, e a mais comum: as chamadas *variam*, mas nada avança. Política, histórico, política de novo, despesa, política. Cada chamada difere da anterior; o conjunto se repete.

```python
def sem_progresso(estado: Estado, janela: int = 6) -> bool:
    """Nas últimas `janela` chamadas, nenhuma trouxe informação nova."""
    if estado.n_passos < janela:
        return False
    recentes = estado.passos[-janela:]
    distintas = {assinatura(p) for p in recentes}
    # muitas chamadas, poucas distintas, e nenhuma escrita: está girando
    return len(distintas) <= janela // 2 and not any(
        p.ferramenta in FERRAMENTAS_DE_ESCRITA for p in recentes)
```

Ao detectar, três respostas — da mais suave à mais dura:

| Resposta | O que faz | Quando |
|---|---|---|
| **Injetar observação** | *"você já chamou X com estes argumentos e recebeu isto. Use o que já tem ou diga o que falta."* | primeira detecção — é a intervenção mais barata e costuma bastar |
| **Reduzir ferramentas ativas** | tirar a que está sendo repetida força outro caminho | a observação não resolveu |
| **Abortar** | `Termino.ERRO_FATAL`, motivo `laco_detectado` | diagnóstico honesto e conta fechada |

> **O preço:** detector agressivo interrompe agente que progredia devagar. Tarefa que legitimamente consulta a mesma ferramenta com argumentos parecidos — varrer uma lista — dispara falso positivo. Calibre com trajetórias reais, e comece frouxo.

---

### 7. Deriva de objetivo

Numa trajetória de trinta passos, o pedido original está lá atrás, cercado de observações — exatamente a posição em que o modelo aproveita pior a informação (*lost in the middle*, Aula 02 nota 01 §4.3). O sintoma: o agente resolve com esmero um sub-problema que ninguém pediu.

```python
REANCORAR_A_CADA = 5

if estado.n_passos and estado.n_passos % REANCORAR_A_CADA == 0:
    msgs.append({"role": "user",
                 "content": f"Lembrete do objetivo: {estado.objetivo}"})
```

Custa algumas dezenas de tokens a cada cinco passos e é, em custo-benefício, a melhor linha desta nota. Só é possível porque `objetivo` é campo do estado, e não uma mensagem que a compaction pode engolir ([nota 02 §3](02-o-agente-e-o-estado.md)).

---

### 8. Idempotência: agora não é mais hipótese

A Aula 01 (nota 03, §3) já pedia chave de idempotência em ferramenta de escrita. Era prudência. Agora é necessidade, por um motivo desta nota: **as salvaguardas que acabamos de adicionar aumentam a chance de repetir uma escrita.**

- o **retry** de rede reenvia requisição cujo efeito já ocorreu (o que se perdeu foi a resposta);
- o **checkpoint** (§9) retoma de um estado salvo, e o último passo pode ter executado sem ser gravado;
- a **detecção de laço**, ao injetar observação e deixar continuar, pode levar o agente a repetir a ação que ele achou que não funcionou.

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

A chave precisa ser **determinística e derivada do conteúdo** — `f"{despesa_id}:{veredito}"` ou um hash dos campos. Chave gerada com `uuid4()` a cada chamada não é chave de idempotência: é um identificador novo por tentativa, que é exatamente o que se quer evitar.

E o retorno diz `ja_existia: True`. Isso é informação para o **modelo**: ele descobre que a ação já ocorreu e não tenta de novo. Escrita idempotente que devolve resposta idêntica esconde do agente o fato de que ele se repetiu.

---

### 9. Checkpoint

O código está na [nota 02, exemplo 4](02-o-agente-e-o-estado.md) — quatro linhas de `json.dumps` sobre o estado. O que ele destrava:

1. **Confirmação humana assíncrona** — o agente pausa, grava e devolve o controle; a aprovação chega horas depois e a execução continua de onde parou.
2. **Retomada após queda** — o processo morreu no passo 9 de uma trajetória cara; retomar não repete os 9 passos nem os efeitos colaterais deles.
3. **Depuração por reprodução** — carregar o estado de uma execução que deu errado e continuar dali, com log ligado.

A armadilha: **salvar depois de executar, nunca antes**. Se o checkpoint gravar a intenção e o processo cair entre gravação e execução, a retomada reexecuta. É a idempotência da §8 que fecha essa fresta — as duas trabalham juntas, e nenhuma sozinha resolve.

> Persistir estado **entre execuções diferentes** — o agente lembrar da semana passada — é a aula de memória. Checkpoint é dentro de uma execução só.

---

### 10. O trace: o subproduto que vale mais que o produto

Repare no que foi construído sem que ninguém pedisse. Para detectar laço, foi preciso guardar ferramenta e argumentos de cada passo. Para o orçamento, tokens e custo. Para o término, o motivo. Junte tudo e você tem, por execução, um registro estruturado do que o agente fez, quanto gastou, o que falhou e por que parou.

Isso tem nome — **trace** — e é a matéria-prima de duas aulas que ainda vêm: **observabilidade** (agregar traces: qual ferramenta mais falha? qual o custo médio por tarefa?) e **evals** (sem trajetória você avalia só a resposta final; com ela, avalia **o caminho**, onde a maioria dos defeitos mora).

> **Não se avalia o que não se rastreia.** Foi por isso que o estado veio antes de tudo.

---

## Exemplos

### Exemplo 1 — A mesma trajetória, com e sem erro que ensina

Despesa cujo id o usuário digitou errado. Orçamento de 12 passos.

```
SEM detector e com erro inútil ({"erro": "falhou"}):
  passo  0..11  consultar_historico {"funcionario": "F-88"} -> erro
  TERMINO: orcamento_esgotado (passos: 12/12) | 14.200 tokens | R$ 0,036
```

```
COM erro que ensina:
  passo  0  consultar_historico {"funcionario": "F-88"}
            -> {"erro": "funcionário não encontrado", "recebido": "F-88",
                "esperado": "F seguido de 3 dígitos, ex: F-088",
                "sugestao": "chame listar_funcionarios"}
  passo  1  consultar_historico {"funcionario": "F-088"}  -> ok
  passo  2  consultar_politica  {"categoria": "refeicao"} -> ok
  TERMINO: respondeu | 1.480 tokens | R$ 0,004
```

Nove vezes menos tokens — e **o detector de laço nem chegou a disparar**. Quem resolveu foi o texto do erro. O detector é a rede de segurança para quando isso não bastar, e a maior parte do ganho está em não precisar dele.

### Exemplo 2 — Progresso nulo: o laço que o detector estrito não pega

```
passo 3  consultar_politica   {"categoria": "refeicao"}
passo 4  consultar_historico  {"funcionario": "F-088"}
passo 5  consultar_politica   {"categoria": "refeicao"}     <- repetida
passo 6  buscar_despesa       {"id": "D-4471"}
passo 7  consultar_politica   {"categoria": "refeicao"}     <- de novo
passo 8  consultar_historico  {"funcionario": "F-088"}      <- de novo
```

`detectar_laco` não dispara: nunca há três iguais em sequência. `sem_progresso` dispara: seis chamadas, **três** assinaturas distintas, nenhuma escrita. O agente tem toda a informação e não consegue concluir — o problema não é falta de dado, é o critério de conclusão estar vago no system prompt.

### Exemplo 3 — Os quatro términos, no log

```
exec 3f9a1c  termino=respondeu           passos=4   tokens=2.180  R$ 0,006
exec 7b2e04  termino=orcamento_esgotado  passos=12  tokens=14.200 R$ 0,036
              motivo="passos: 12/12"
exec c81d55  termino=erro_fatal          passos=2   tokens=780    R$ 0,002
              motivo="banco indisponível: connection refused"
exec 09fa7e  termino=aguardando_humano   passos=6   tokens=3.940  R$ 0,010
              pendencia={"ferramenta": "registrar_parecer", "argumentos": {...}}
```

Quatro execuções, quatro diagnósticos diferentes — e três ações diferentes de quem opera: a segunda pede revisar o orçamento **ou** o prompt; a terceira, avisar a infraestrutura; a quarta, alguém aprovar. Sem o campo `termino`, as quatro chegariam como "não deu certo".

### Exemplo 4 — Idempotência salvando um parecer duplicado

```python
# Passo 7: o agente registra. A rede cai DEPOIS da gravação, antes da resposta.
# O retry reenvia.
chave = "D-4471:aprovado"

registrar_parecer("D-4471", "aprovado", "dentro do teto", chave=chave)
# -> {"id": "P-1001", "despesa_id": "D-4471", ...}

registrar_parecer("D-4471", "aprovado", "dentro do teto", chave=chave)
# -> {"id": "P-1001", ..., "ja_existia": True}    mesmo parecer, não um novo
```

Sem a chave, o segundo registro cria `P-1002`, e alguém vai auditar uma despesa com dois pareceres idênticos sem saber qual vale.

---

## Exercícios resolvidos

### 1. Este erro é recuperável ou fatal?

| Erro | Classe | Por quê |
|---|---|---|
| `consultar_politica("alimentacao")` — as válidas são `refeicao`, `transporte`, `hospedagem` | **recuperável** | é argumento; devolva as válidas e o modelo escolhe |
| Token da API do ERP expirado | **fatal** | nenhuma decisão do modelo renova credencial |
| `buscar_despesa("D-9999")` — id bem formado, registro inexistente | **recuperável** | o modelo pode listar e escolher outro |
| Timeout de 30s no banco | **depende** | primeira ocorrência: retry no seu código; após N tentativas, fatal |
| `registrar_parecer` sem `justificativa` | **recuperável** | argumento faltando; devolva o campo exigido |
| Disco cheio ao gravar | **fatal** | infraestrutura |

A pergunta que resolve todos: **existe alguma chamada diferente que o modelo poderia fazer para contornar?** Se existe, é recuperável — e o retorno de erro deve dizer qual é.

### 2. Calcular o orçamento de uma tarefa

> Triagem de despesa: em média 3 passos de ~1.200 tokens. R$ 0,60 por milhão de tokens de entrada, R$ 1,80 de saída, ~15% de saída. Qual orçamento?

```
tokens por tarefa = 3 × 1.200 = 3.600
entrada (85%)     = 3.060 × 0,60 / 1e6 = R$ 0,0018
saída   (15%)     =   540 × 1,80 / 1e6 = R$ 0,0010
custo médio                            ≈ R$ 0,0028
```

O teto **não** é a média — é o ponto a partir do qual você prefere uma tarefa incompleta a uma conta aberta. Folga de 3× a 4× cobre os casos difíceis legítimos e ainda pega o agente descontrolado:

```python
Orcamento(max_passos=10, max_tokens=15_000, max_reais=0.02, max_segundos=90)
```

A conta que justifica o esforço: em 10.000 despesas/mês, o custo esperado é ~R$ 28. Um agente sem teto que entre em laço em 1% dos casos, gastando 12 passos cada, **dobra** essa conta — e a linha que impede isso tem quatro linhas de código.

---

## Síntese

- `max_passos` não é orçamento: passo não é unidade de custo. Meça em **passos, tokens, dinheiro e tempo de parede** — e passe o orçamento como parâmetro.
- Um agente termina de quatro formas, e as quatro viram **dado** no estado. O trace é gravado no `finally`.
- Erro de ferramenta é **recuperável** (o modelo contorna) ou **fatal** (aborta). Quem classifica é o seu código.
- **A mensagem de erro é prompt:** o que estava errado, qual era o certo, o que fazer agora.
- Retry automático é para falha de **transporte**; falha de **conteúdo** volta para o modelo com informação nova.
- Detecte laço por assinatura repetida e **progresso nulo** por chamadas que variam sem avançar. Intervenha antes de abortar.
- Reafirme o objetivo no fim do contexto a cada N passos: melhor custo-benefício da nota.
- Chave de idempotência derivada do **conteúdo**, não da tentativa — e o retorno avisa quando a ação já existia.
- Checkpoint depois de executar, nunca antes.
- Nada disso é grátis: cada salvaguarda tem falso positivo, e calibrar a dose é o trabalho.
- O subproduto de tudo é o **trace** — matéria-prima das aulas de observabilidade e evals.

---

## Fontes e leituras

- Aula 01, [nota 03 §3 e §4](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — a tabela de falhas que organiza esta nota, e a idempotência.
- Aula 02, [nota 01 §8.2](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — backoff exponencial; [nota 04](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md) — a conta por chamada, que vira o teto em reais.
- Aula 03, [nota 04 §4 e §7](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — a descrição da ferramenta como prompt, estendida aqui ao retorno de erro.
- **Building Effective Agents** — Anthropic (2024), seção sobre *guardrails* e limites de execução.
- **A practical guide to building agents** — OpenAI (2025), capítulo de *guardrails* e tratamento de exceções.
- **OWASP Agentic Security Initiative** — riscos específicos de agentes. Citado só como referência; o tratamento é da aula de segurança.
