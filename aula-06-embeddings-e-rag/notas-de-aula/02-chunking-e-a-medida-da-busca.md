# IA Aplicada com LLMs — Aula 06: Embeddings e RAG — Chunking e a medida da busca

## Introdução

A nota anterior tratou do vetor: como se obtém e como se compara. Esta trata do que se embute — e de como saber se a escolha foi boa.

O objeto novo é o **chunk**: a unidade em que o documento é cortado antes de ser indexado. Ele determina o que a busca pode devolver, porque é literalmente o que volta. Um corte inadequado torna irrecuperável informação que está no corpus, e nenhum ajuste posterior compensa isso.

A segunda metade da nota estabelece a medida. Afirmações sobre estratégia de *chunking* circulam em abundância e quase sempre sem número associado; a que esta aula adota — em documento normativo, o corte por estrutura supera o corte por contagem de caracteres — só tem valor se for verificável. O instrumento é o `recall@k` sobre um conjunto de perguntas com resposta conhecida, e construir esse conjunto é o trabalho principal da aula.

> **Pré-requisitos:** [nota 01](01-o-vetor-e-a-similaridade.md) desta aula.
>
> **Código:** [`02-chunking.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula06-embeddings-e-rag/02-chunking.py) e [`03-buscador.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula06-embeddings-e-rag/03-buscador.py). As três estratégias de corte estão em `estrategias_chunking.py`, juntas de propósito; o `recall@k` está no próprio `02-chunking.py`; o `indice_memoria.py` guarda o `IndiceMemoria`; o cosseno está no `similaridade.py`; e o `gerar_matrix_embbeddings`, no `embedding.py`, a única porta da aula para a API.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Justificar** uma estratégia de *chunking* pelo critério da unidade de recuperação.
- **Implementar** corte por contagem, por contagem com sobreposição e por estrutura.
- **Construir** um conjunto de perguntas com resposta conhecida a partir de um documento.
- **Medir** `recall@k` e escolher o `k` como decisão de projeto.
- **Explicar** por que o índice é construído uma vez e consultado sempre, e o que muda quando ele passa a persistir.
- **Diagnosticar** uma falha de recuperação, distinguindo defeito de corte, pergunta mal formulada e limitação do próprio vetor.

---

## Desenvolvimento teórico

### 1. O chunk é a unidade de recuperação

A definição operacional é a única de que se precisa:

> **Chunk** é o trecho que a busca devolve. Ele chegará ao leitor — humano ou modelo — **sem o texto em volta**.

Dessa definição decorre o critério: um chunk precisa fazer sentido isoladamente. Um parágrafo que diga *"§2º Em viagem internacional, o limite de que trata o §1º passa a R$ 260,00"* é inútil recuperado sozinho — não informa de qual limite se trata, nem de que categoria de despesa.

O critério tem duas implicações opostas, e a tensão entre elas é o que torna a escolha uma decisão:

| Chunk grande | Chunk pequeno |
|---|---|
| carrega o contexto necessário | é preciso: menos assunto por trecho |
| dilui o assunto: o vetor representa a média de tudo que está ali | chega mutilado, sem o que o torna interpretável |
| consome janela na etapa de geração | consome menos, e permite recuperar mais trechos |

Não há valor universalmente correto. Há um valor mensurável para um corpus e um conjunto de perguntas, e a §5 mostra como obtê-lo.

---

### 2. Três estratégias

#### 2.1 Corte por contagem de caracteres

```python
def por_caracteres(texto: str, tamanho: int = 400) -> list[dict]:
    limpo = texto.strip()
    return [{"id": f"c{i//tamanho}", "texto": limpo[i:i + tamanho]}
            for i in range(0, len(limpo), tamanho)]
```

A estratégia mais simples, e a que **ignora integralmente a estrutura que o autor do documento escreveu**. Corta no meio de frases, separa o parágrafo do artigo a que pertence e produz chunks cujo início é arbitrário.

Sua vantagem é operar sobre qualquer texto, sem conhecimento do formato. É a escolha correta quando o corpus é heterogêneo e não há estrutura a aproveitar.

#### 2.2 Corte por contagem com sobreposição

```python
def por_caracteres_sobrepostos(texto: str, tamanho: int = 400,
                               sobreposicao: int = 100) -> list[dict]:
    limpo = texto.strip()
    passo = tamanho - sobreposicao
    return [{"id": f"s{i//passo}", "texto": limpo[i:i + tamanho]}
            for i in range(0, len(limpo), passo)]
```

A sobreposição existe para que uma frase partida ao meio apareça íntegra em pelo menos um dos chunks. Reduz o dano do corte arbitrário sem eliminá-lo.

O desperdício é medível: com `tamanho=400` e `sobreposicao=100`, cada caractere do corpus é embutido em média 1,33 vez. O índice cresce na mesma proporção, e o trabalho de construção também.

#### 2.3 Corte por estrutura

```python
def por_estrutura(texto: str) -> list[dict]:
    """Cada chunk carrega o cabeçalho do artigo a que pertence."""
```

A estratégia parte da observação de que **o documento já vem cortado pelo autor**. Artigos e parágrafos são a unidade de sentido escolhida por quem redigiu o texto normativo; ignorá-la é descartar informação disponível de graça.

A implementação do laboratório faz uma coisa a mais, e é ela que produz a diferença: **cada parágrafo é indexado junto com o cabeçalho do artigo a que pertence**. O chunk correspondente ao §2º do Art. 4º não é o parágrafo isolado, e sim:

```
Art. 4º — Das despesas com alimentação.
§2º Em viagem internacional, o limite de que trata o §1º passa a R$ 260,00
por pessoa por refeição.
```

Sem o cabeçalho, esse parágrafo é ininterpretável isolado — que é exatamente a condição estabelecida na §1. Com ele, o chunk é autossuficiente.

O identificador produzido (`Art. 4º §2º`) tem consequência além da organização: é o rótulo que a Aula 07 usa para tornar a **citação verificável**. Uma resposta que cite "Art. 4º §2º" pode ser conferida contra a lista de chunks recuperados, por comparação de cadeia de caracteres, sem envolver o modelo.

---

### 3. A assimetria entre pergunta e documento

Há um limite que nenhuma das três estratégias resolve. A pergunta e o trecho que a responde não têm a mesma forma:

| | Exemplo | Extensão |
|---|---|---|
| Pergunta | "quanto posso gastar em refeição numa viagem?" | ~8 palavras, registro coloquial |
| Trecho | "Art. 4º §1º — O reembolso de refeições em viagem a serviço fica limitado a..." | ~30 palavras, registro normativo |

O cosseno compara os dois vetores como se fossem objetos do mesmo tipo, e eles não são. Uma pergunta curta e informal fica sistematicamente mais próxima de outra pergunta curta e informal do que do artigo que a responde — que é a origem do fenômeno descrito na [nota 01](01-o-vetor-e-a-similaridade.md), §6.

O problema tem nome e tem tratamentos, nenhum deles objeto desta aula:

- **reescrita da consulta** (*query rewriting*): uma chamada ao modelo transforma a pergunta em algo com a forma do documento antes de embutir. Custa uma chamada por consulta e está entre os desafios opcionais do exercício;
- **modelos assimétricos**: alguns modelos de embedding são treinados com prefixos distintos para consulta e para documento, justamente para tratar esse caso.

O que esta nota estabelece é a existência do problema. A Aula 07 volta a ele por outro caminho: em vez de melhorar uma consulta, emitir várias.

---

### 4. O índice, e quando ele basta em memória

```python
class IndiceMemoria:
    def __init__(self, chunks: list[dict]):
        self.chunks = chunks
        self.matriz = gerar_matrix_embbeddings([c["texto"] for c in chunks])

    def buscar(self, pergunta: str, k: int = 3) -> list[dict]:
        scores = cosseno_lote(gerar_vetor_embeddings(pergunta), self.matriz)
        ordem = np.argsort(-scores)[:k]
        return [{**self.chunks[i], "score": float(scores[i])} for i in ordem]
```

O índice desta aula é um arranjo de numpy e uma lista de dicionários. A busca é exaustiva: compara a pergunta com todos os chunks, sempre.

Isso não é uma limitação didática — é a escolha correta na escala em questão. Com quarenta chunks de dimensão típica, a multiplicação de matriz é imperceptível, e um índice aproximado introduziria erro de recuperação para resolver um problema de desempenho que não existe.

A pergunta que decide a favor de um banco vetorial não é de velocidade, e sim de **persistência**: o índice acima morre com o processo, e reconstruí-lo custa uma chamada de embedding sobre o corpus inteiro a cada execução. É esse o argumento que introduz o Chroma na Aula 07.

#### 4.1 Construir uma vez, consultar sempre

A frase acima — *"reconstruí-lo custa uma chamada de embedding sobre o corpus inteiro"* — esconde a propriedade que faz o índice valer a pena. Não é a assimetria de forma da §3, entre pergunta curta e artigo longo: é uma assimetria de **frequência**, e o `03-buscador.py` a imprime:

```
construir o índice ... 1 chamada, com os 28 chunks juntos   UMA VEZ
esta consulta ........ 1 chamada, só com a pergunta        A CADA PERGUNTA
```

| Operação | Natureza | Frequência |
|---|---|---|
| Embutir o corpus | trabalho de **construção** | uma vez |
| Embutir a pergunta | trabalho de **operação** | a cada consulta |

A segunda linha é o que se paga em toda pergunta; a primeira, uma vez só. É essa razão — e não o tamanho absoluto de nenhuma das duas — que torna a busca vetorial viável. Um sistema que reconstruísse o índice a cada consulta pagaria a primeira linha sempre, e não teria vantagem alguma sobre ler o documento inteiro.

O "uma vez" desta aula vale **dentro de uma execução**: o índice morre com o processo. Na Aula 07 ele vai para disco, e o "uma vez" passa a valer entre execuções — que é quando a assimetria começa a pagar de verdade.

> O que ela significa **em dinheiro** é assunto da Aula 13. Aqui se mede em tokens, que é o que decide o desenho.

---

### 5. O conjunto de perguntas com resposta conhecida

A medida exige um instrumento, e o instrumento é um conjunto de pares:

```python
PERGUNTAS = [
    {"pergunta": "Qual o teto de reembolso para uma refeição em viagem?",
     "artigo": "Art. 4º §1º"},
    {"pergunta": "Quanto posso gastar em refeição numa viagem para Portugal?",
     "artigo": "Art. 4º §2º"},
    ...
]
```

Cada entrada declara uma pergunta e **qual trecho do documento a responde**. Construir esse conjunto é o trabalho da aula; medir, depois de tê-lo, é trivial.

Três critérios de construção, e os três importam mais que o tamanho do conjunto:

1. **A pergunta é redigida como um usuário a formularia**, não como o documento a responde. *"Quanto posso gastar em refeição numa viagem para Portugal?"* é uma pergunta; *"qual o limite de reembolso de alimentação em viagem internacional?"* é uma paráfrase do artigo, e mede o modelo contra ele mesmo.
2. **A resposta é um trecho específico**, não o documento. Se a resposta correta for "está no regulamento", não há o que medir.
3. **O conjunto inclui casos difíceis de propósito.** Um conjunto em que todas as perguntas acertam não informa nada sobre a próxima mudança de configuração.

Este conjunto é o primeiro **dataset de avaliação** do curso. A Aula 11 o retoma sob esse nome, e a única diferença será a escala.

---

### 6. `recall@k`

```python
def recall_at_k(indice: IndiceMemoria, perguntas: list[dict], k: int = 3) -> dict:
    acertos, falhas = 0, []
    for caso in perguntas:
        recuperados = indice.buscar(caso["pergunta"], k=k)
        if acertou(recuperados, caso["artigo"]):
            acertos += 1
        else:
            falhas.append({**caso, "veio": recuperados[0]["id"],
                           "score": recuperados[0]["score"]})
    return {"k": k, "acertos": acertos, "total": len(perguntas),
            "recall": acertos / len(perguntas), "falhas": falhas}
```

A métrica responde a uma pergunta única: **em quantas das perguntas o trecho correto apareceu entre os `k` primeiros**.

Duas propriedades merecem registro. A primeira é que ela **não envolve geração**: mede a busca isoladamente, o que é precisamente o que a torna útil. A Aula 07 mostra o que se perde ao não separar essa medida da qualidade da resposta.

A segunda é que a função devolve as **falhas**, não apenas o número. É onde está a informação acionável, e a §8 trata disso.

#### 6.1 A verificação por conteúdo

A função `acertou` compara por **conteúdo**, não por identificador:

```python
def acertou(chunks: list[dict], artigo_esperado: str) -> bool:
    alvo = _normalizar(artigo_esperado)
    return any(alvo in _normalizar(c["texto"]) or alvo in _normalizar(c["id"])
               for c in chunks)
```

A razão é metodológica. As estratégias por contagem produzem identificadores como `c7` e `s12`, que não guardam relação com a numeração do documento — mas o chunk `c7` pode perfeitamente conter o texto do Art. 4º §1º. Comparar por identificador puniria essas estratégias por uma característica que não é o objeto da medida, e o resultado da comparação deixaria de ser honesto.

---

### 7. A escolha do `k`

O `k` não é detalhe de implementação. É decisão de projeto com dois efeitos opostos:

| `k` maior | `k` menor |
|---|---|
| `recall` sobe: mais chances de o trecho certo estar na lista | `recall` cai |
| **esconde defeito de índice**: um corte ruim é compensado por trazer mais coisa | expõe o defeito |
| **enche a janela na Aula 07**: cada chunk ocupa contexto em toda pergunta | contexto menor e mais preciso |

O `02-chunking.py` mede o efeito diretamente, variando `k` em 1, 3, 5 e 10 sobre a estratégia vencedora:

```
  k=1    recall =  7/10  ( 70%)
  k=3    recall = 10/10  (100%)
  k=5    recall = 10/10  (100%)
  k=10   recall = 10/10  (100%)
```

A curva sobe de uma vez e estabiliza em `k=3`. Esse é o `k` defensável neste corpus: adotar 5 ou 10 é pagar contexto na Aula 07 por `recall` que já foi ganho.

O salto de 70% para 100% entre `k=1` e `k=3` é a medida exata do que se ganha ao **não** apostar no primeiro colocado — e o Exemplo 3 mostra quais três perguntas vivem nessa diferença.

---

### 8. A pergunta que falha

Toda configuração tem pelo menos uma. Identificá-la e diagnosticá-la vale mais que o `recall` médio, porque a média é um número e o diagnóstico é uma ação.

O diagnóstico segue três hipóteses, em ordem:

| Hipótese | Como confirmar | Correção |
|---|---|---|
| **O vetor não representa o que a pergunta exige** | a pergunta depende de número, identificador, negação ou data | não é caso de busca; usar regra — e a Aula 07 mede o tamanho deste problema |
| **Defeito de corte** | o trecho correto existe, mas está partido ou sem cabeçalho | mudar a estratégia ou o tamanho |
| **Pergunta mal formulada** | um leitor humano com o documento na mão também não saberia responder | corrigir o conjunto de perguntas |

A terceira hipótese é a mais desconfortável e a mais frequente na primeira montagem do conjunto. Uma pergunta ambígua produz falha de recuperação que parece defeito de índice, e horas podem ser gastas ajustando *chunking* para corrigir um enunciado.

---

## Exemplos

### Exemplo 1 — O mesmo artigo, cortado de três formas

Trecho do Art. 4º, sob as três estratégias:

```
[caracteres]  "...limitado a R$ 120,00 por pessoa por refeição, exigida a
               nota fiscal. §2º Em viagem internacional, o limite de que
               trata o §1º passa a R$ 260,00 por pes"
              -> começa no meio de uma frase e termina no meio de outra

[sobreposto]  o mesmo, com 100 caracteres do chunk anterior repetidos
              -> a frase partida aparece íntegra no chunk vizinho

[estrutura]   "Art. 4º — Das despesas com alimentação.
               §2º Em viagem internacional, o limite de que trata o §1º
               passa a R$ 260,00 por pessoa por refeição."
              -> id: "Art. 4º §2º"
```

O terceiro chunk é o único autossuficiente, e o único cujo identificador serve como citação verificável.

### Exemplo 2 — Recall comparado

```
  caracteres    9 chunks   recall@3 =  1/10 ( 10%)  ████
  sobreposto   12 chunks   recall@3 =  1/10 ( 10%)  ████
  estrutura    28 chunks   recall@3 = 10/10 (100%)  ████████████████████████████████████████
```

*(Valores medidos com `mistral-embed`.)*

A diferença não decorre do número de chunks — a estratégia por estrutura produz mais chunks *e* menores. Decorre de cada chunk conter exatamente uma regra, com o contexto necessário para interpretá-la.

### Exemplo 3 — As três falhas, e o que elas têm em comum

Com o corte por estrutura, `recall@3` é **10/10**. Em `k=1`, cai para **7/10**, e as três que falham são estas:

```
  pergunta ... Qual o limite por corrida de aplicativo?
  esperado ... Art. 5º §1º   ·  veio: Art. 5º §2º  (0.7908)

  pergunta ... Preciso de nota fiscal para táxi?
  esperado ... Art. 3º §2º   ·  veio: Art. 3º §1º  (0.8233)

  pergunta ... Uma despesa de R$ 1.240 é aprovada por quem?
  esperado ... Art. 9º §2º   ·  veio: Art. 9º §1º  (0.8745)
```

*(Valores medidos com `mistral-embed`, via `python 02-chunking.py 1`.)*

**As três erram da mesma maneira: acertam o artigo e erram o parágrafo.** Nenhuma trouxe um assunto alheio — o vetor localizou corretamente a região do regulamento em todos os casos, e falhou em distinguir dispositivos vizinhos do mesmo artigo.

A terceira explica por quê. O Art. 9º trata das alçadas em três parágrafos distinguidos **apenas por faixas de valor**: até R$ 500,00 (§1º), de R$ 500,00 a R$ 5.000,00 (§2º), acima de R$ 5.000,00 (§3º). Os três chunks são quase idênticos para o vetor, porque a única diferença entre eles é **numérica**, e o vetor não representa magnitude — a Aula 07, §3, mede o tamanho desse problema. A pergunta cita R$ 1.240, e nenhum modelo de embedding compara 1.240 com 500 e 5.000.

Daí saem as duas decisões que fecham a aula:

**`k = 1` é uma aposta.** Os três casos entram no top-3 e nenhum é o top-1. Recuperar três e deixar a decisão para a etapa seguinte custa contexto e resolve o problema — é o que a Aula 07 faz.

**Onde a distinção é numérica, o vetor não decide.** A alçada se resolve com `<=`, não com cosseno. Recuperar o Art. 9º inteiro e aplicar a faixa em código acerta 10 em 10, sempre — e é mais barato.

---

## Fontes e leituras

MUENNIGHOFF, N. et al. **MTEB**: massive text embedding benchmark. arXiv:2210.07316, 2022. (As tarefas de recuperação do *benchmark* usam exatamente a estrutura de pares pergunta-documento descrita na §5.)

REIMERS, N.; GUREVYCH, I. **Sentence-BERT**: sentence embeddings using siamese BERT-networks. arXiv:1908.10084, 2019. (Fundamenta a assimetria entre consulta e documento discutida na §3.)

MISTRAL AI. **Embeddings**: documentação do modelo `mistral-embed`. (Limite de tokens por entrada, que restringe o tamanho máximo de chunk.)

**Material da disciplina.** Aula 05, [nota 01-6](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-6-qual-padrao-usar.md) — a tabela de decisão, cujo critério "o mais barato é não usar o modelo" reaparece na §8. Aula 03, [nota 03](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) — versionamento; a estratégia de *chunking* entra no carimbo pelo mesmo motivo que o prompt.
