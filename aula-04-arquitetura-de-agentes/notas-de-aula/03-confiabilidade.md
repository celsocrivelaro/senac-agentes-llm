# IA Aplicada com LLMs — Aula 04: Arquitetura de agentes — Confiabilidade

## Introdução

A Aula 01 (nota 03, §4) trouxe uma tabela com oito modos de falha de agentes e, ao lado de cada um, a salvaguarda correspondente. Naquele momento, aquilo era um mapa: você ainda não tinha escrito um agente, e as salvaguardas eram nomes.

Esta nota transforma cada linha daquela tabela em código.

| Falha | Como aparece | Onde está resolvida aqui |
|---|---|---|
| **Laço** | repete a mesma chamada indefinidamente | §6 |
| **Deriva de objetivo** | numa trajetória longa, se afasta do pedido | §7 |
| **Alucinação de argumento** | inventa um id plausível que não existe | §3 e §4 |
| **Ferramenta errada** | escolhe escrever quando devia ler | [nota 02, §6](02-o-agente-e-o-estado.md) |
| **Contexto estourado** | a trajetória não cabe mais na janela | [nota 04](04-context-engineering-dinamica.md) |
| **Erro em cascata** | uma observação errada contamina o resto | §3 e §8 |
| **Custo descontrolado** | consome mil vezes o previsto | §1 |
| **Silêncio** | falhou e você não sabe por quê | §2 e §10 |

Tudo o que vem a seguir depende do objeto de estado da [nota 02](02-o-agente-e-o-estado.md). Não é uma dependência conceitual: é literalmente que os campos usados aqui não existem no laço da Aula 03.

E há um aviso que vale abrir a nota, porque ele é o antídoto contra a leitura errada de tudo que vem depois:

> **Nenhuma salvaguarda é grátis.** Orçamento apertado mata tarefa legítima. Detector de laço agressivo interrompe agente que estava progredindo devagar. Confirmação humana em excesso mata a autonomia que justificava o agente. Cada seção desta nota apresenta o preço junto com o remédio — e escolher a dose é trabalho de engenharia, não de receita.

> **Pré-requisitos:** as notas [01](01-padroes-de-arquitetura.md) e [02](02-o-agente-e-o-estado.md) desta aula. Da Aula 01, a [nota 03 §3 e §4](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md). Da Aula 02, a [nota 01 §8.2](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) (backoff exponencial) e a [nota 04](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md) (a conta por chamada).

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Impor** um orçamento em quatro moedas — passos, tokens, dinheiro e tempo — e justificar o valor de cada teto.
- **Implementar** as quatro formas de terminar um agente, registrando qual delas ocorreu.
- **Classificar** um erro de ferramenta em recuperável ou fatal, e tratar cada classe de forma diferente.
- **Escrever** um retorno de erro que faz o modelo se corrigir, em vez de repetir a falha.
- **Distinguir** o que merece retry automático do que só o modelo pode resolver.
- **Detectar** laço por repetição de chamada e por ausência de progresso.
- **Proteger** ferramentas de escrita com chave de idempotência.
- **Salvar e retomar** a execução a partir de um checkpoint.

---

## Desenvolvimento teórico

### 1. Orçamento em quatro moedas

O laço da Aula 03 tinha uma proteção: `max_passos=6`. Ela é necessária e é a mais fraca das quatro, porque **passo não é uma unidade de custo**. Um passo que consulta um id custa 300 tokens; um passo que lê um documento de 30 páginas custa 40.000. Dizer "no máximo 10 passos" é dizer "no máximo entre 3 mil e 400 mil tokens", o que não é um limite.

Um agente sério tem teto em quatro moedas:

```python
import time
from dataclasses import dataclass


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

Quatro observações sobre este código:

**O orçamento é parâmetro, não constante.** Ele entra na chamada de `rodar()`. Tarefas diferentes merecem tetos diferentes, e um número mágico no meio do arquivo garante que ninguém vai ajustá-lo. Uma triagem de item simples e uma investigação de divergência não têm o mesmo direito de gastar.

**O custo em reais vem da Aula 02** (nota 04): tokens de entrada e de saída têm preços diferentes, e a conta por passo é `entrada × preço_in + saída × preço_out`. O `custo_estimado` do estado é essa soma acumulada. É a moeda que o seu chefe entende.

**O tempo de parede é o teto esquecido**, e o que mais salva. Ele pega o caso que as outras três não pegam: a ferramenta que travou esperando uma resposta que não vem. Sem `max_segundos`, o agente não estoura nenhum limite — ele simplesmente não termina.

**A função devolve *qual* teto estourou.** `bool` seria mais simples e inútil: um agente que morreu por tempo e um que morreu por tokens têm diagnósticos diferentes.

> **O preço:** todo teto tem um falso positivo. Uma tarefa legítima e difícil vai bater no orçamento e morrer sem resposta. É por isso que o motivo é registrado — para você distinguir "o agente falhou" de "o orçamento estava curto", que exigem correções opostas.

---

### 2. As quatro formas de terminar

Um agente termina de exatamente quatro maneiras, e as quatro precisam ser código explícito:

```
   ┌─────────────────────────────────────────────────────────┐
   │ 1. RESPONDEU        o modelo devolveu sem tool_calls     │  ✓ sucesso
   │ 2. ORCAMENTO        estourou um dos quatro tetos         │  ✗ inconcluso
   │ 3. ERRO_FATAL       não há como continuar                │  ✗ falha
   │ 4. HUMANO           pausou aguardando confirmação        │  ⏸ suspenso
   └─────────────────────────────────────────────────────────┘
```

O laço da Aula 03 tratava a primeira com `return` e a segunda com `raise RuntimeError`. A terceira derrubava o programa por exceção não tratada, e a quarta não existia.

O que muda com o estado é que o término vira **dado**:

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

Repare no `finally`. O trace é gravado **em todos os caminhos**, e é justamente nos caminhos de falha que ele vale mais — o `except` que registra só o sucesso é o `except` que garante que você nunca vai descobrir a causa do problema.

> **A linha "silêncio" da tabela de falhas morre aqui.** Um agente que sempre diz por que parou e sempre grava a trajetória não falha em silêncio. Ele pode falhar — vai falhar —, mas você saberá onde.

---

### 3. Erro de ferramenta: recuperável × fatal

Esta é a decisão de projeto mais importante da nota, e ela é **do seu código**, não do modelo.

```
   ferramenta levantou exceção
              │
     ┌────────┴─────────┐
     │                  │
  RECUPERÁVEL         FATAL
     │                  │
  vira mensagem       aborta o laço
  role="tool"         Termino.ERRO_FATAL
     │
  o modelo tenta de novo,
  com informação nova
```

**Recuperável** é o erro que o modelo pode contornar mudando o que faz: argumento inválido, registro inexistente, formato errado, resultado vazio, permissão negada para aquele recurso específico. A informação volta como observação e a trajetória continua.

**Fatal** é o erro que nenhuma decisão do modelo resolve: credencial inválida, banco fora do ar, biblioteca quebrada, disco cheio. Devolver isso ao modelo produz tentativas caras contra uma parede — ele vai reformular o argumento educadamente umas quatro vezes até o orçamento acabar.

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

E no executor:

```python
def executar(chamada, estado: Estado) -> Passo:
    nome = chamada.function.name
    argumentos = json.loads(chamada.function.arguments)
    passo = Passo(indice=estado.n_passos, ferramenta=nome, argumentos=argumentos)
    try:
        passo.resultado = FERRAMENTAS[nome](**argumentos)
    except ErroRecuperavel as e:
        passo.erro = e.payload["erro"]
        passo.resultado = e.payload          # volta para o modelo
    except TypeError as e:                    # argumento que não existe na função
        passo.erro = str(e)
        passo.resultado = {"erro": "argumentos inválidos", "detalhe": str(e)}
    return passo
```

O `except TypeError` merece uma linha: o modelo às vezes inventa um parâmetro que a função não tem. Sem esse tratamento, uma alucinação de argumento derruba o processo inteiro. Com ele, vira observação — e o modelo corrige na volta seguinte.

> **O preço:** classificar errado custa nos dois sentidos. Marcar como fatal o que era recuperável mata trajetórias que se salvariam; marcar como recuperável o que era fatal queima o orçamento inteiro contra uma parede. Na dúvida, olhe a causa: se a origem é o **argumento**, é recuperável; se é a **infraestrutura**, é fatal.

---

### 4. O retorno de erro é prompt

A Aula 03 (nota 04, §4) estabeleceu que **a descrição da ferramenta é prompt** — é o único texto que o modelo lê para decidir quando chamar. O corolário quase nunca é dito:

> **A mensagem de erro também é prompt.** É o único texto que o modelo lê para decidir como se corrigir.

Compare:

```python
# Inútil
{"erro": "falhou"}

# Inútil de outra forma — verdadeiro e sem ação possível
{"erro": "ValueError: invalid literal for int() with base 10: 'D-4471'"}

# Útil
{"erro": "formato de id inválido",
 "esperado": "D seguido de 4 dígitos, ex: D-4471",
 "recebido": "4471",
 "sugestao": "chame listar_despesas para obter os ids válidos"}
```

O terceiro funciona porque responde às três perguntas que o modelo precisa responder para agir: **o que estava errado**, **qual era o certo** e **o que fazer agora**. Os dois primeiros deixam o modelo adivinhar, e adivinhação custa passos.

Uma consequência prática: escreva os retornos de erro com o mesmo cuidado com que escreve a descrição da ferramenta. Na prática, isso significa revisá-los olhando trajetórias reais — quando o agente se perde depois de um erro, o defeito quase sempre está no texto que você devolveu, não no modelo.

O caso mais valioso é o do erro que **ensina o caminho**: a `sugestao` apontando outra ferramenta transforma um beco sem saída em um passo produtivo. É o mesmo princípio do exemplo 2 da Aula 03 (nota 04), agora deliberado.

---

### 5. Retry, e onde ele não vale

A Aula 02 (nota 01, §8.2) ensinou o backoff exponencial para o `429`. Ele continua valendo — e agora precisa de uma fronteira clara, porque num agente existem **dois** mecanismos de recuperação, e confundi-los é caro:

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
        except RateLimitError:
            time.sleep(2 ** tentativa)          # 1, 2, 4, 8, 16
        except APIConnectionError:
            time.sleep(2 ** tentativa)
    raise ErroFatal("API indisponível após 5 tentativas")
```

E o que **não** se faz:

```python
# ERRADO: repetir a mesma chamada de ferramenta que falhou por argumento
for tentativa in range(3):
    try:
        return FERRAMENTAS[nome](**argumentos)
    except ErroRecuperavel:
        continue          # os argumentos são os mesmos. O erro será o mesmo.
```

Repetir uma chamada determinística com os mesmos argumentos produz o mesmo resultado — é retry sem informação nova. Quem "tenta de novo" nesse caso é **o modelo**, e ele só consegue porque recebeu a observação do erro.

> **A regra de bolso:** retry automático é para falha de **transporte**. Falha de **conteúdo** volta para o modelo.

---

### 6. Detecção de laço e de progresso nulo

O `max_passos` limita o dano do laço; ele não detecta o laço. A diferença importa: com um teto de 12 passos, um agente preso na primeira volta gasta 12 passos antes de morrer, e a mensagem que você recebe é "orçamento esgotado" — diagnóstico errado para o problema certo.

**Laço estrito** é a mesma ferramenta com os mesmos argumentos, repetida:

```python
def assinatura(passo: Passo) -> tuple:
    return (passo.ferramenta, json.dumps(passo.argumentos, sort_keys=True))


def detectar_laco(estado: Estado, limite: int = 3) -> bool:
    if estado.n_passos < limite:
        return False
    recentes = [assinatura(p) for p in estado.passos[-limite:]]
    return len(set(recentes)) == 1
```

O `sort_keys=True` não é detalhe: sem ele, `{"id": "D-1", "ano": 2026}` e `{"ano": 2026, "id": "D-1"}` são assinaturas diferentes, e o detector não detecta nada.

**Progresso nulo** é a versão sutil, e a mais comum na prática: as chamadas *variam*, mas nada avança. O agente consulta a política, consulta o histórico, consulta a política de novo, consulta a despesa, volta à política. Cada chamada é diferente da anterior; o conjunto se repete.

```python
def sem_progresso(estado: Estado, janela: int = 6) -> bool:
    """Nas últimas `janela` chamadas, nenhuma trouxe informação nova."""
    if estado.n_passos < janela:
        return False
    recentes = estado.passos[-janela:]
    distintas = {assinatura(p) for p in recentes}
    # muitas chamadas, poucas distintas, e nenhuma escrita: está girando
    return len(distintas) <= janela // 2 and not any(
        p.ferramenta in FERRAMENTAS_DE_ESCRITA for p in recentes
    )
```

E o que fazer ao detectar. Três respostas, da mais suave à mais dura:

1. **Injetar uma observação**: *"você já chamou `consultar_politica` com estes argumentos e recebeu este resultado. Use a informação que já tem ou responda com o que falta."* Muitas vezes basta, e é a intervenção mais barata.
2. **Reduzir as ferramentas ativas** — tirar a que está sendo repetida força outro caminho.
3. **Abortar** com `Termino.ERRO_FATAL` e motivo `laco_detectado`. Diagnóstico honesto e conta fechada.

> **O preço:** um detector agressivo interrompe agentes que estavam progredindo devagar. Uma tarefa que legitimamente consulta a mesma ferramenta com argumentos parecidos várias vezes — varrer uma lista, por exemplo — dispara falso positivo. Calibre o `limite` com trajetórias reais, e comece frouxo.

---

### 7. Deriva de objetivo

Numa trajetória de trinta passos, o pedido original está lá atrás, cercado de observações. É exatamente a posição em que o modelo aproveita pior a informação — o viés em U do *lost in the middle* (Aula 02, nota 01, §4.3). O sintoma: o agente resolve com esmero um sub-problema que ninguém pediu.

A intervenção mais barata que existe é reafirmar o objetivo **no fim** do contexto, periodicamente:

```python
REANCORAR_A_CADA = 5

def montar_mensagens(estado: Estado) -> list[dict]:
    msgs = [{"role": "system", "content": SYSTEM},
            {"role": "user", "content": estado.objetivo}]
    msgs.extend(estado.historico)
    if estado.n_passos and estado.n_passos % REANCORAR_A_CADA == 0:
        msgs.append({"role": "user",
                     "content": f"Lembrete do objetivo: {estado.objetivo}"})
    return msgs
```

Custa algumas dezenas de tokens a cada cinco passos e é, em relação custo-benefício, a melhor linha de código desta nota. Ela só é possível porque `objetivo` é campo do estado, e não uma mensagem que a compaction pode engolir ([nota 02, §3](02-o-agente-e-o-estado.md)).

---

### 8. Idempotência: agora não é mais hipótese

A Aula 01 (nota 03, §3) já dizia que ferramentas de escrita deveriam aceitar chave de idempotência. Naquele momento, era prudência. Agora é necessidade, e por um motivo específico desta nota: **as salvaguardas que acabamos de adicionar aumentam a chance de repetir uma escrita.**

Três caminhos novos levam à repetição:

- o **retry** de rede pode reenviar uma requisição cujo efeito já ocorreu (a resposta é que se perdeu);
- o **checkpoint** (§9) retoma de um estado salvo, e o último passo pode ter executado sem ter sido gravado;
- a **detecção de laço**, ao injetar observação e deixar o agente continuar, pode levá-lo a repetir a ação que ele achou que não tinha funcionado.

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

A chave precisa ser **determinística e derivada do conteúdo** — por exemplo `f"{despesa_id}:{veredito}"` ou um hash dos campos. Uma chave gerada com `uuid4()` a cada chamada não é chave de idempotência: é um identificador novo por tentativa, que é precisamente o que se quer evitar.

E note que o retorno diz `ja_existia: True`. Isso é informação para o **modelo**: ele descobre que a ação já havia ocorrido e não precisa tentar de novo. Uma escrita idempotente que devolve resposta idêntica esconde do agente o fato de que ele se repetiu.

---

### 9. Checkpoint

Se o estado é um objeto serializável, salvá-lo a cada passo é barato — e destrava três coisas:

```python
def salvar_checkpoint(estado: Estado) -> None:
    caminho = CHECKPOINTS / f"{estado.execucao_id}.json"
    caminho.write_text(json.dumps(asdict(estado), ensure_ascii=False, default=str))


def retomar(execucao_id: str) -> Estado:
    dados = json.loads((CHECKPOINTS / f"{execucao_id}.json").read_text())
    return Estado(**dados)
```

O que isso permite:

1. **Confirmação humana assíncrona** — o agente pausa, grava e devolve o controle; a aprovação chega horas depois e a execução continua de onde parou ([nota 02, §7](02-o-agente-e-o-estado.md)).
2. **Retomada após queda** — o processo morreu no passo 9 de uma trajetória cara; retomar do checkpoint não repete os 9 passos nem os efeitos colaterais deles.
3. **Depuração por reprodução** — carregar o estado de uma execução que deu errado e continuar dali, com log ligado.

A armadilha a evitar: **salvar depois de executar, nunca antes**. Se o checkpoint gravar a intenção e o processo cair entre a gravação e a execução, a retomada vai reexecutar. É a idempotência da §8 que fecha essa fresta — as duas salvaguardas trabalham juntas, e nenhuma das duas sozinha resolve.

> Persistir estado **entre execuções diferentes** — o agente lembrar o que aconteceu na semana passada — é outro assunto, e é a aula de memória. O checkpoint é dentro de uma execução só.

---

### 10. O trace: o subproduto que vale mais que o produto

Repare no que foi construído sem que ninguém pedisse. Para detectar laço, foi preciso guardar ferramenta e argumentos de cada passo. Para o orçamento, tokens e custo. Para o término, o motivo. Junte tudo e você tem, para cada execução, um registro estruturado de o que o agente fez, quanto gastou, o que falhou e por que parou.

Isso tem nome: **trace**. E ele é a matéria-prima de duas aulas que ainda vêm:

- **observabilidade** — agregar traces para responder "qual ferramenta mais falha?", "qual o custo médio por tarefa?", "onde os agentes travam?";
- **evals** — sem trajetória registrada, você só consegue avaliar a resposta final; com ela, avalia **o caminho**, que é onde a maioria dos defeitos mora.

> **Não se avalia o que não se rastreia.** Foi por isso que o estado veio antes de tudo.

---

## Exemplos

### Exemplo 1 — A mesma trajetória, com e sem detector

Agente investigando uma despesa cujo id o usuário digitou errado. Orçamento de 12 passos.

**Sem detector de laço:**

```
passo  0  consultar_historico {"funcionario": "F-88"}  -> erro: inexistente
passo  1  consultar_historico {"funcionario": "F-88"}  -> erro: inexistente
passo  2  consultar_historico {"funcionario": "F-88"}  -> erro: inexistente
...
passo 11  consultar_historico {"funcionario": "F-88"}  -> erro: inexistente
TERMINO: orcamento_esgotado (passos: 12/12) | 14.200 tokens | R$ 0,036
```

Doze passos, o diagnóstico errado ("orçamento") e a conta paga inteira.

**Com detector (limite=3) e erro que ensina:**

```
passo  0  consultar_historico {"funcionario": "F-88"}
          -> {"erro": "funcionário não encontrado", "recebido": "F-88",
              "esperado": "F seguido de 3 dígitos, ex: F-088",
              "sugestao": "chame listar_funcionarios"}
passo  1  consultar_historico {"funcionario": "F-088"} -> ok
passo  2  consultar_politica  {"categoria": "refeicao"} -> ok
TERMINO: respondeu | 1.480 tokens | R$ 0,004
```

Nove vezes menos tokens. E repare que **o detector nem chegou a disparar**: quem resolveu foi o texto do erro. O detector é a rede de segurança para quando isso não bastar — e a maior parte do ganho está em não precisar dele.

### Exemplo 2 — Idempotência salvando um parecer duplicado

```python
# Passo 7: o agente registra o parecer. A rede cai DEPOIS da gravação,
# antes da resposta chegar. O retry reenvia.
chave = f"D-4471:aprovado"

registrar_parecer("D-4471", "aprovado", "dentro do teto", chave=chave)
# -> {"id": "P-1001", "despesa_id": "D-4471", ...}

registrar_parecer("D-4471", "aprovado", "dentro do teto", chave=chave)
# -> {"id": "P-1001", ..., "ja_existia": True}    mesmo parecer, não um novo
```

Sem a chave, o segundo registro cria `P-1002`, e alguém vai auditar uma despesa com dois pareceres idênticos, sem saber qual vale.

---

## Exercícios resolvidos

### 1. Este erro é recuperável ou fatal?

Classifique e justifique:

| Erro | Classe | Por quê |
|---|---|---|
| `consultar_politica("alimentacao")` — categoria não existe, as válidas são `refeicao`, `transporte`, `hospedagem` | **recuperável** | é argumento; devolva as categorias válidas e o modelo escolhe |
| Token da API do ERP expirado | **fatal** | nenhuma decisão do modelo renova credencial |
| `buscar_despesa("D-9999")` — id bem formado, registro inexistente | **recuperável** | o modelo pode listar e escolher outro |
| Timeout de 30s no banco | **depende** | primeira ocorrência: retry no seu código; após N tentativas, fatal |
| `registrar_parecer` sem o campo `justificativa` | **recuperável** | é argumento faltando; devolva o campo exigido |
| Disco cheio ao gravar o parecer | **fatal** | infraestrutura |

A pergunta que resolve todos: **existe alguma chamada diferente que o modelo poderia fazer para contornar?** Se existe, é recuperável — e o retorno de erro deve dizer qual é.

### 2. Calcular o orçamento de uma tarefa

> Uma triagem de despesa gasta, em média, 3 passos de ~1.200 tokens. Preço: R$ 0,60 por milhão de tokens de entrada e R$ 1,80 por milhão de saída, com cerca de 15% de saída. Qual orçamento?

Custo médio da tarefa:

```
tokens por tarefa   = 3 × 1.200 = 3.600
entrada (85%)       = 3.060 × 0,60 / 1e6 = R$ 0,0018
saída   (15%)       =   540 × 1,80 / 1e6 = R$ 0,0010
custo médio                              ≈ R$ 0,0028
```

O teto **não** é a média — é o ponto a partir do qual você prefere uma tarefa incompleta a uma conta aberta. Uma folga de 3× a 4× sobre a média cobre os casos difíceis legítimos e ainda pega o agente descontrolado:

```python
Orcamento(max_passos=10, max_tokens=15_000, max_reais=0.02, max_segundos=90)
```

E a conta que justifica o esforço: em 10.000 despesas por mês, o custo esperado é ~R$ 28. Um agente sem teto que entre em laço em 1% dos casos, gastando 12 passos cada, **dobra** essa conta — e a linha do orçamento que impede isso tem quatro linhas de código.

---

## Síntese

- `max_passos` não é orçamento: passo não é unidade de custo. Meça em **passos, tokens, dinheiro e tempo de parede** — e passe o orçamento como parâmetro, nunca como constante.
- Um agente termina de quatro formas: respondeu, estourou orçamento, erro fatal, aguarda humano. As quatro viram **dado** no estado, e o trace é gravado no `finally`.
- Erro de ferramenta é **recuperável** (o modelo contorna) ou **fatal** (aborta). Quem classifica é o seu código.
- **A mensagem de erro é prompt.** Ela precisa dizer o que estava errado, qual era o certo e o que fazer agora.
- Retry automático é para falha de **transporte**; falha de **conteúdo** volta para o modelo com informação nova.
- Detecte laço por assinatura repetida e **progresso nulo** por chamadas que variam sem avançar. Intervenha antes de abortar.
- Reafirme o objetivo no fim do contexto a cada N passos: é a linha de código de melhor custo-benefício da nota.
- Chave de idempotência derivada do **conteúdo**, não da tentativa — e o retorno avisa quando a ação já existia.
- Checkpoint depois de executar, nunca antes; ele destrava confirmação assíncrona, retomada e reprodução.
- Nada disso é grátis: cada salvaguarda tem um falso positivo, e calibrar a dose é o trabalho.
- O subproduto de tudo isso é o **trace** — a matéria-prima das aulas de observabilidade e evals.

---

## Fontes e leituras

- Aula 01, [nota 03 §3 e §4](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — a tabela de falhas que organiza esta nota, e a idempotência.
- Aula 02, [nota 01 §8.2](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — backoff exponencial; [nota 04](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md) — a conta por chamada, que vira o teto em reais.
- Aula 03, [nota 04 §4 e §7](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — a descrição da ferramenta como prompt, estendida aqui ao retorno de erro.
- **Building Effective Agents** — Anthropic (2024), seção sobre *guardrails* e limites de execução.
- **A practical guide to building agents** — OpenAI (2025), capítulo de *guardrails* e tratamento de exceções.
- **OWASP Agentic Security Initiative** — catálogo de riscos específicos de agentes. Citado aqui só como referência; o tratamento é da aula de segurança.
