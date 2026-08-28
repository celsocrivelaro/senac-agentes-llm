# IA Aplicada com LLMs — Aula 03: Prompt engineering — Context engineering

## Introdução

A nota anterior terminou com uma tabela de técnicas e uma constatação incômoda: **todas ocupam espaço**. Few-shot infla a entrada em toda chamada. CoT infla a saída. Self-consistency multiplica por N. Generated knowledge dobra o número de chamadas.

Todas elas fazem a mesma coisa por baixo: **colocam mais coisa na janela de contexto.**

E aí aparece a pergunta que a nota anterior deixou em aberto — *e quando não cabe mais?* Ou, na versão que importa mais, porque acontece muito antes de estourar a janela: *e quando ainda cabe, mas o resultado começa a piorar?*

A distinção que organiza esta nota é pequena e muda tudo:

> **Prompt é o que você escreve.**
> **Contexto é o que o modelo recebe.**

Na Aula 01 o prompt era o que você digitava. No exercício da Aula 02 já havia duas etapas, cada uma montando o seu contexto. Num agente, o contexto é montado **em tempo de execução** a partir de meia dúzia de fontes, e o texto que você escreveu é uma fatia pequena dele.

**Context engineering** é a disciplina de decidir o que entra nessa janela — e, principalmente, **o que não entra**.

> **Pré-requisitos:** a [nota 01](01-anatomia-e-tecnicas.md) desta aula (as técnicas e o espaço que cada uma ocupa) e, da Aula 02, [nota 01 §4](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) (janela de contexto, custo quadrático, *lost in the middle*).

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Distinguir** prompt de contexto, e enumerar as fontes que compõem a janela numa aplicação real.
- **Explicar** por que "mais contexto" degrada o resultado antes de estourar o limite.
- **Decidir o que não colocar** na janela, e justificar cada exclusão.
- **Posicionar** conteúdo na janela para que o modelo encontre o que importa.
- **Montar um orçamento de contexto** para um caso de uso.
- **Distinguir** curadoria **estática** (esta nota) de curadoria **dinâmica** (aula de agentes).

---

## Desenvolvimento teórico

### 1. O que realmente está na janela

Numa aplicação de verdade, a janela de uma única chamada costuma ter seis origens:

```
┌─────────────────────────────────────────────┐
│ 1. system prompt      (você escreveu)       │  estável
│ 2. exemplos few-shot  (você escolheu)       │  estável
│ 3. schema de saída    (você definiu)        │  estável
├─────────────────────────────────────────────┤
│ 4. histórico          (acumulou)            │  cresce
│ 5. documentos         (recuperados)         │  varia
│ 6. saída de ferramentas (chegou do mundo)   │  varia
├─────────────────────────────────────────────┤
│ 7. a mensagem do usuário                    │  pequena
└─────────────────────────────────────────────┘
```

Repare em duas coisas.

**A primeira:** a mensagem do usuário — a coisa que a maioria das pessoas chama de "o prompt" — é normalmente a **menor** fatia. Nos itens 4, 5 e 6 estão os tokens que decidem a conta.

**A segunda:** você escreve os três de cima. Os três do meio são **montados pelo seu código em tempo de execução**. É aí que mora a engenharia.

---

### 2. Por que "mais" piora antes de estourar

A intuição natural é que contexto extra é, no pior caso, inofensivo — o modelo ignora o que não precisa. **Não é o que acontece.** Três razões, e nenhuma delas é o limite da janela:

**1. *Lost in the middle*.** A Aula 02 (nota 01, §4.3) mostrou o viés em U: a informação no meio do contexto é sistematicamente pior aproveitada. Acrescentar 15 documentos irrelevantes **empurra o relevante para o meio**. Você não adicionou ruído neutro; você mudou de lugar o que importava.

**2. Contexto contraditório.** Dois documentos que discordam não fazem o modelo dizer "há divergência". Fazem ele escolher um, sem avisar. Recuperar "por precaução" é uma forma de introduzir contradição.

**3. Diluição da instrução.** A sua instrução compete com todo o resto pela atenção do modelo. Um system prompt de 300 tokens cercado de 50 mil tokens de documento tem menos força relativa do que o mesmo system prompt com 2 mil.

> **A formulação que vale guardar:** o contexto não é um saco onde se joga tudo o que talvez ajude. É uma **seleção**, e toda seleção implica excluir.

---

### 3. Curadoria: o que não colocar

A pergunta operacional não é "o que pode ajudar?" — quase tudo pode. É:

> **Se eu remover isto, o que o modelo deixa de conseguir fazer?**

Se você não souber responder, o item sai. Aplicada às fontes da §1:

| Fonte | O que tirar |
|---|---|
| **System prompt** | instruções que repetem o schema; regras para casos que nunca ocorrem; adjetivos ("cuidadosamente", "detalhadamente") |
| **Few-shot** | exemplos redundantes — cobertura de rótulos importa mais que quantidade (nota 01, §6.2) |
| **Histórico** | turnos antigos irrelevantes; o resumo custa menos que o histórico bruto |
| **Documentos** | tudo além dos poucos realmente relevantes. Se você precisa de 20, o problema é **recuperação** |
| **Saída de ferramenta** | JSON gigante devolvido inteiro; trunque, pagine, ou resuma antes de devolver ao modelo |

O caso da **saída de ferramenta** é o que mais surpreende, e volta com força na aula de agentes: uma consulta que devolve 200 registros de banco vira 8 mil tokens no contexto — a cada volta do laço. Ferramenta bem projetada devolve **o que o modelo precisa**, não o que a API dela retorna.

---

### 4. Posição importa

Dado que você decidiu **o que** entra, falta decidir **onde**.

O critério vem direto do *lost in the middle*: **instrução crítica e pergunta no começo ou no fim**, nunca soterradas no meio. Um padrão que funciona bem com documentos longos:

```
┌──────────────────────────────────────────┐
│ instrução + exemplos + schema            │ ← o que o modelo precisa saber
├──────────────────────────────────────────┤
│ documentos recuperados                   │ ← o material de trabalho
├──────────────────────────────────────────┤
│ histórico recente + A PERGUNTA           │ ← por último, para não se perder
└──────────────────────────────────────────┘
```

Repare que a **pergunta fica no fim**, mesmo que ela já apareça no início. Com contexto longo, repeti-la ao final é uma das intervenções mais baratas que existem — e uma das que mais mudam o resultado.

---

### 5. O orçamento de contexto

A forma madura de tratar isso é como **orçamento**: decidir de antemão quanto cada fonte pode ocupar, e fazer o código respeitar.

| Fonte | Teto | O que fazer ao estourar |
|---|---|---|
| system + schema | 800 | não estoura — é fixo e revisado |
| few-shot | 600 | reduzir exemplos, mantendo cobertura de rótulos |
| documentos | 3.000 | recuperar menos; melhorar o ranqueamento |
| histórico | 1.500 | **resumir** os turnos antigos |
| pergunta | 200 | — |
| **reserva para a saída** | 1.000 | — |

Duas coisas que o orçamento resolve e que a improvisação não resolve:

1. **A reserva para a saída.** A janela é entrada + saída (Aula 02, nota 01 §4.1). Sem reservar, você monta um contexto lindo e recebe `finish_reason="length"`.
2. **Comportamento previsível sob carga.** Sem teto, o sistema funciona bem no protótipo e degrada silenciosamente quando as conversas ficam longas — o "o bot esqueceu" da Aula 02, nota 01 §4.4.

E há um efeito que só aparece com o sistema rodando: **o orçamento torna o comportamento reproduzível.** Sem teto, dois usuários com conversas de tamanhos diferentes recebem qualidades diferentes — e você não consegue nem dizer por quê.

---

### 6. Estático e dinâmico

Tudo nesta nota é **curadoria estática**: você decide, antes de rodar, o que compõe a chamada.

Num agente, o contexto muda **ao longo da execução**: cada volta acrescenta raciocínio, chamada de ferramenta e resultado. Depois de oito passos, a maior parte da janela é história que o agente produziu — e a curadoria precisa acontecer **durante**, não antes.

| | Estático (aqui) | Dinâmico (aula de agentes) |
|---|---|---|
| Quando decide | ao escrever o código | em tempo de execução |
| Técnicas | seleção, orçamento, posição | *compaction*, resumo de trajetória, *tool clearing*, sub-agentes que devolvem resumos |
| Pergunta | o que **montar** | o que **descartar**, e quando |

Vale saber que os nomes existem — *compaction* (compactar a trajetória), *tool clearing* (remover resultados de ferramenta já usados), **isolamento por sub-agentes** (delegar uma sub-tarefa a um agente próprio, que devolve só um resumo curto) —, mas eles só fazem sentido quando existir um laço. Por ora, guarde a ideia de que **a janela é um recurso que se administra**, e não um espaço que se preenche.

---

## Exemplos

### Exemplo 1 — O mesmo pedido, três montagens de contexto

A §2 afirmou que mais contexto **piora** antes de estourar. Este exemplo é essa afirmação virando algo que você roda em três minutos.

A pergunta do cliente é sempre a mesma. O que muda é **o que você coloca junto com ela**.

```python
PERGUNTA = "meu pedido chegou?"

PEDIDO_ATUAL = """Pedido 48219 — status: em transporte
Previsão de entrega: 02/09/2026 — Transportadora: RápidoLog"""

HISTORICO_COMPLETO = """Pedido 11002 — entregue em 03/2025 — Liquidificador
Pedido 20551 — entregue em 05/2025 — Jogo de panelas
Pedido 31890 — cancelado em 07/2025 — Cafeteira
... (mais 40 pedidos antigos) ...
Pedido 48219 — status: em transporte — previsão 02/09/2026
... (mais 15 pedidos posteriores, já entregues) ..."""

INSTRUCAO = "Você é o atendimento de uma transportadora. Responda ao cliente."

# (a) só a pergunta
contexto_a = f"{INSTRUCAO}\n\nCliente: {PERGUNTA}"

# (b) a pergunta + o que ela precisa
contexto_b = f"{INSTRUCAO}\n\n{PEDIDO_ATUAL}\n\nCliente: {PERGUNTA}"

# (c) a pergunta + tudo o que existe sobre o cliente
contexto_c = f"{INSTRUCAO}\n\n{HISTORICO_COMPLETO}\n\nCliente: {PERGUNTA}"
```

**O que esperar de cada uma:**

| | O que o modelo faz |
|---|---|
| **(a)** | não tem como saber. Ou diz que não sabe — resposta honesta e inútil — ou **inventa** um status com fluência total |
| **(b)** | responde certo: pedido 48219, em transporte, previsão 02/09 |
| **(c)** | **piora**: costuma citar um pedido errado, misturar datas de pedidos diferentes, ou responder "seus pedidos foram entregues" — porque o 48219 está soterrado no meio de 55 outros |

**A leitura que interessa:** de (a) para (b), acrescentar contexto **resolveu**. De (b) para (c), acrescentar mais contexto **estragou** — e note que a informação certa **estava lá** em (c), inteira e correta. Ela só estava no meio.

> Este é o momento em que o *lost in the middle* deixa de ser um gráfico de paper e vira uma coisa que aconteceu com o seu código.

**Experimente duas variações**, porque elas separam as duas causas:

1. **mova o pedido 48219 para o fim** do histórico em (c). Se a resposta melhorar, você mediu o efeito da **posição**;
2. **corte o histórico para os 3 pedidos mais recentes**. Se melhorar mais ainda, você mediu o efeito da **seleção** — e acabou de fazer, à mão, o que a aula de RAG vai automatizar.

### Exemplo 2 — Medindo a composição real da sua janela

Antes de otimizar, meça. Isto revela quase sempre uma surpresa:

```python
partes = {
    "system":     SYSTEM,
    "exemplos":   FEW_SHOT,
    "documentos": "\n\n".join(docs),
    "historico":  formatar(historico),
    "pergunta":   pergunta,
}

# tiktoken serve para PLANEJAR; usage é o que foi COBRADO (Aula 02, nota 04 §1)
import tiktoken
enc = tiktoken.get_encoding("cl100k_base")

total = sum(len(enc.encode(v)) for v in partes.values())
for nome, texto in sorted(partes.items(), key=lambda kv: -len(enc.encode(kv[1]))):
    n = len(enc.encode(texto))
    print(f"{nome:<12} {n:>6} tokens  {n/total:>5.1%}  {'#' * int(40 * n / total)}")
print(f"{'TOTAL':<12} {total:>6} tokens")
```

O resultado típico: **documentos e histórico ocupam mais de 80%**, e o system prompt — onde a discussão do time costuma se concentrar — ocupa 5%. Otimizar o system enquanto o histórico cresce sem teto é arrumar a gaveta com a casa inundando.

### Exemplo 3 — Histórico com teto e resumo

O padrão que resolve o crescimento quadrático da Aula 02 (nota 04, §3):

```python
LIMITE_HISTORICO = 1500       # tokens, do orçamento da §5

def montar_historico(turnos, resumo_anterior=""):
    recentes, ocupado = [], 0
    for turno in reversed(turnos):            # do mais novo para o mais antigo
        n = contar_tokens(turno)
        if ocupado + n > LIMITE_HISTORICO:
            break
        recentes.insert(0, turno)
        ocupado += n

    antigos = turnos[: len(turnos) - len(recentes)]
    if antigos:
        # resumir com o modelo PEQUENO: resumir é tarefa fácil
        # (Aula 02, nota 01 §2.2 — o menor modelo que passa no teste)
        resumo_anterior = resumir(antigos, resumo_anterior)

    return resumo_anterior, recentes
```

Três decisões embutidas, e vale reconhecê-las:

1. **de trás para frente** — os turnos recentes são os que importam;
2. **teto em tokens, não em número de turnos** — turnos têm tamanhos muito diferentes;
3. **resumo com modelo pequeno** — resumir é tarefa fácil, e não precisa do modelo bom (Aula 02, nota 01 §2.2).

---

## Exercícios resolvidos

### 1. Cortando um contexto que não cabe

**Enunciado.** Um assistente jurídico monta este contexto por chamada: system 1.200 · 6 exemplos few-shot 2.400 · 12 cláusulas recuperadas 9.000 · histórico completo 4.000 · pergunta 150. O modelo tem janela de 16k. As respostas pioraram desde que o histórico começou a crescer. O que fazer, em ordem?

**Resolução.**

**Diagnóstico primeiro.** Total: 16.750 tokens de entrada numa janela de 16k — **já estourou**, e sem reserva nenhuma para a saída. O sintoma "piorou desde que o histórico cresceu" é consistente com duas causas simultâneas: truncamento (silencioso, se o framework corta sozinho) e *lost in the middle* (as cláusulas foram empurradas para o meio pelo histórico).

**Ordem de corte, do maior impacto para o menor:**

1. **Documentos: 9.000 → ~3.000.** É a maior fatia e a mais provavelmente inflada. Doze cláusulas para responder uma pergunta é sintoma de **recuperação ruim**, não de necessidade. Reduzir para as 3–4 mais relevantes (com reranking) melhora *e* barateia. Se a qualidade cair, o problema é o ranqueamento — assunto de RAG, não de janela.
2. **Histórico: 4.000 → 1.500 com resumo.** É a fonte que **cresce sozinha**; sem teto, qualquer ajuste é temporário. Resumir com modelo pequeno.
3. **Few-shot: 2.400 → ~800.** Seis exemplos é muito. Pelo Min et al. (nota 01, §6), o que importa é cobrir o espaço de rótulos e o formato — 2 ou 3 bons exemplos costumam entregar quase o mesmo.
4. **System: 1.200.** Mexer por último. É 7% do total, e é a parte revisada e testada — o risco de quebrar comportamento não compensa a economia.

**Resultado:** ~6.650 de entrada, com ~9k de folga para a saída. E a ordem importa: começar pelo system (o instinto de muita gente) mexeria no que menos pesa e mais arrisca.

**A lição:** ao cortar contexto, **ataque a fonte que cresce e a que está inflada** — não a que você escreveu e portanto conhece melhor.

### 2. Onde vai cada coisa?

**Enunciado.** Você está montando um assistente de atendimento. Para cada informação abaixo, diga **onde ela entra** — system prompt, mensagem do usuário, ou nem uma coisa nem outra:

**(a)** As 5 categorias possíveis de chamado.
**(b)** O nome do cliente que está falando agora.
**(c)** A instrução "responda sempre em português, de forma cordial".
**(d)** Os 3 chamados anteriores desse mesmo cliente.
**(e)** A data de hoje.
**(f)** A lista completa dos 4.000 produtos do catálogo.

**Resolução.**

**(a) System prompt.** Vale para toda chamada e nunca muda entre requisições. É parte do contrato de saída (nota 01, §4) — e, melhor ainda, deveria estar também no `enum` do schema.

**(b) Mensagem do usuário.** Muda a cada chamada; é contexto da tarefa. Colocar no system parece inofensivo, mas cria um system prompt diferente por cliente — e aí ele deixa de ser o artefato estável que você versiona e testa (nota 03).

**(c) System prompt.** É comportamento, não dado. Exatamente o tipo de coisa que o system existe para carregar.

**(d) Mensagem do usuário — e com teto.** É a fonte que **cresce**: sem limite, três chamados viram trinta. Aplique o orçamento da §5, e resuma os antigos se passar.

**(e) Depende — e a resposta certa é quase sempre "nem uma coisa nem outra".** Pergunte primeiro: **alguma instrução sua depende da data?** Se o assistente calcula prazos, ela precisa entrar (na mensagem, não no system, porque muda a cada chamada). Se nada depende dela, ela não deveria estar lá — muito system prompt carrega data por hábito, sem nenhuma instrução que a use.

**(f) Nem uma coisa nem outra.** Quatro mil produtos não cabem na janela, e mesmo que coubessem seriam o caso clássico do §2: você teria empurrado a instrução e a pergunta para o meio de um monte de texto irrelevante. O que se faz é **recuperar os 3 ou 4 produtos que a mensagem menciona** — e isso é RAG, a próxima aula.

**O padrão nas seis respostas:** a pergunta que decide não é "isso é importante?", é **"isso muda entre chamadas?"** e **"alguma instrução minha depende disso?"**. A primeira separa system de mensagem. A segunda elimina o que não deveria estar em lugar nenhum.

---

## Síntese

- **Prompt é o que você escreve; contexto é o que o modelo recebe.** A mensagem do usuário costuma ser a **menor** fatia da janela.
- Numa chamada real, a janela vem de seis fontes: system, exemplos, schema (estáveis) · histórico, documentos, saída de ferramenta (montados em tempo de execução).
- **Mais contexto piora antes de estourar**, por três motivos: *lost in the middle*, contradição entre fontes e diluição da instrução.
- A pergunta de curadoria é **"se eu remover isto, o que o modelo deixa de conseguir fazer?"** — sem resposta, o item sai.
- **Se você precisa de 20 documentos, o problema é de recuperação**, não de janela.
- **Saída de ferramenta é a fonte mais subestimada**: ferramenta boa devolve o que o modelo precisa, não o que a API dela retorna.
- **Posição importa**: instrução crítica e pergunta no **começo ou no fim**, nunca no meio.
- **Orçamento de contexto** com teto por fonte **e reserva para a saída**. Sem teto, o sistema degrada silenciosamente quando as conversas crescem.
- **A pergunta vai no fim**, mesmo repetida: é a intervenção mais barata de escrever e uma das que mais mudam o resultado em contexto longo.
- Isto é curadoria **estática**. A **dinâmica** — *compaction*, resumo de trajetória, *tool clearing*, sub-agentes — só faz sentido quando existir um laço, e é a aula de agentes.

---

## Fontes e leituras

**Papers**

- Liu, N. F. et al. — *Lost in the Middle: How Language Models Use Long Contexts* (2023). [arxiv.org/abs/2307.03172](https://arxiv.org/abs/2307.03172) — o viés em U que sustenta a §2.

**Engenharia**

- Anthropic — *Context engineering: memory, compaction and tool clearing* — as técnicas dinâmicas que esta nota apenas nomeia.

**Nesta disciplina**

- [Aula 02 — nota 01, §4](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — janela de contexto, custo quadrático, *lost in the middle*, e o histórico que some.
- [Nota 01 desta aula](01-anatomia-e-tecnicas.md) — as técnicas cujo espaço ocupado motiva esta nota.
- [Nota 04 desta aula](04-tool-calling.md) — a saída de ferramenta, que é a fonte de contexto que mais cresce num agente.
