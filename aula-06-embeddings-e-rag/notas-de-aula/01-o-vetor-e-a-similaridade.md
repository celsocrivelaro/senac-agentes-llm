# IA Aplicada com LLMs — Aula 06: Embeddings e RAG — O vetor e a similaridade

## Introdução

A Aula 02 (nota 01, §5) apresentou uma tabela de modalidades em que uma linha se distinguia das demais:

> **Embeddings** — transformar texto em vetor — **não gera texto**; é a base da Aula 06.

A Aula 01 (nota 01, §6.1) já havia dito de onde essa modalidade vem. Ao separar as três linhagens do Transformer, aquela nota encerrou assim: *"você vai reencontrá-lo no curso: os modelos de **embedding** usados em busca semântica e RAG são tipicamente encoder-only"*.

Esta é a nota do reencontro. E ela inaugura a única aula do curso em que o modelo generativo não é chamado nenhuma vez.

O problema que a motiva vem da Aula 05. O agente construído lá consulta a política de reembolso por uma **ferramenta determinística**: um dicionário com teto por categoria, escrito à mão por alguém. Funciona porque alguém estruturou a política. A pergunta desta aula é o que fazer quando o conhecimento é um documento de quarenta artigos que ninguém vai estruturar.

> **Pré-requisitos:** Aula 01, [nota 01 §6.1](../../aula-01-llms-e-agentes/notas-de-aula/01-redes-neurais-e-transformer.md) e [nota 02](../../aula-01-llms-e-agentes/notas-de-aula/02-llms-tokens-e-tokenizacao.md) · Aula 02, [nota 01 §5](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md).
>
> **Código:** [`aula06-embeddings-e-rag/`](https://github.com/celsocrivelaro/senac-llm-code/tree/main/aula06-embeddings-e-rag) — os scripts `00-o-vetor.py` e `01-similaridade.py` correspondem a esta nota. O `gerar_matrix_embbeddings` vem do `embedding.py`, a única porta da aula para a API, e o `cosseno` do `similaridade.py`.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Explicar** a origem arquitetural de um modelo de embedding e por que ele não produz token algum.
- **Calcular** similaridade de cosseno e justificar a escolha do cosseno em lugar da distância euclidiana.
- **Implementar** uma busca por ranking sem biblioteca de busca.
- **Demonstrar** que o significado de um embedding não está dentro do vetor, e sim na comparação entre dois.
- **Definir** a norma de um vetor como o comprimento dele, e explicar por que o cosseno a descarta.
- **Reconhecer** que a faixa de valores de cosseno entre textos reais não corresponde à intuição de 0 a 1.
- **Distinguir similaridade de relevância** e identificar o caso em que o trecho mais similar não é o que responde.

---

## Desenvolvimento teórico

### 1. De onde vem o vetor

O Transformer original (VASWANI et al., 2017) era uma arquitetura *encoder-decoder* para tradução. Dela derivaram três linhagens, e apenas uma produziu as LLMs. A tabela da Aula 01 (nota 01, §6.1) resume a distinção pertinente aqui:

| | **Encoder-only** | **Decoder-only** |
|---|---|---|
| Exemplo | BERT (DEVLIN et al., 2018) | GPT, Mistral, Claude |
| Atenção | bidirecional — vê o texto inteiro | causal — apenas o passado |
| Treinado para | preencher lacunas | prever o próximo token |
| Gera texto livre | **não** | sim |
| Uso corrente | embeddings, classificação, busca | assistentes, agentes |

Um modelo de embedding pertence à primeira coluna. Ele lê o texto inteiro de uma vez e produz **representação**, não continuação — e é justamente a atenção bidirecional que o torna melhor em *entender* do que em *escrever*.

A consequência prática é imediata: `client.embeddings.create()` não aceita `temperature`, não tem `max_tokens` e não devolve `choices`. A superfície da API é outra, porque a tarefa é outra.

REIMERS e GUREVYCH (2019) acrescentaram o passo que torna esses modelos utilizáveis em busca. Um BERT puro produz um vetor por token; o que se quer é **um vetor por sentença**, e obtê-lo pela média dos vetores de token produz representações ruins. O treinamento siamês do Sentence-BERT ajusta o modelo para que a similaridade entre vetores de sentença signifique alguma coisa — e é essa linhagem que os modelos de embedding comerciais seguem.

---

### 2. O vetor, na prática

```python
from embedding import gerar_vetor_embeddings

REGRA = "O reembolso de refeições em viagem fica limitado a R$ 120,00."
vetor = gerar_vetor_embeddings(REGRA)
```

O resultado é um arranjo de números de ponto flutuante. O `00-o-vetor.py` imprime dimensão, norma e faixa de valores.

O ponto didático dessa impressão é uma decepção útil: **nenhuma posição do vetor significa alguma coisa isoladamente**. Não existe "a dimensão 42 mede o quanto o texto fala de dinheiro". O que existe é a posição do vetor em relação a outros vetores, e é apenas isso que toda a aula explora.

A dimensão é fixa por modelo e independe do comprimento do texto: uma frase de cinco palavras e um artigo de trezentas produzem vetores do mesmo tamanho. O que pode variar é a **norma** — o comprimento do vetor —, e a §2.2 trata dela, porque é o que a comparação por cosseno descarta.

#### 2.1 Um vetor não diz nada; dois dizem tudo

A decepção da §2 tem resolução imediata, e ela cabe em três textos:

```python
REGRA     = "O reembolso de refeições em viagem fica limitado a R$ 120,00."
PARAFRASE = "Quanto posso gastar almoçando numa viagem a trabalho?"
ALHEIO    = "O campeonato de futebol começa no próximo domingo às dezesseis horas."

v_regra, v_parafrase, v_alheio = gerar_matrix_embbeddings([REGRA, PARAFRASE, ALHEIO])

cosseno(v_regra, v_parafrase)   # 0,8464
cosseno(v_regra, v_alheio)      # 0,7292
```

A regra e a paráfrase **quase não compartilham palavras** — *refeições* contra *almoçando*, *limitado* contra *gastar* — e ainda assim ficam próximas. É precisamente o que a busca por palavra-chave não faz, e é a razão de o embedding existir.

O sentido não está **dentro** do vetor: está **entre** vetores, e só aparece quando há mais de um para comparar. Esse número é o único que a aula inteira usa.

O segundo valor já levanta a pergunta da §5: dois textos sem relação alguma não dão zero, e o piso real fica bem acima disso.

---

#### 2.2 A norma, e por que ela sai da conta

A **norma** de um vetor é o **comprimento** dele. Um vetor é uma lista de números, e uma lista de números é uma seta no espaço: a norma é a distância da origem até a ponta dessa seta.

Em duas dimensões, dá para ver. O vetor `[3, 4]`:

```
   4 ┤        ● (3,4)
     │      ╱
     │    ╱        <- a seta. Qual o comprimento dela?
     │  ╱
   0 ┼────────────
     0    3
```

É o teorema de Pitágoras: a seta é a hipotenusa de um triângulo de catetos 3 e 4.

$$\|v\| = \sqrt{3^2 + 4^2} = \sqrt{25} = 5$$

Em 1024 dimensões não há o que desenhar, mas a fórmula não muda — apenas ganha mais termos, e é exatamente o que `np.linalg.norm` calcula:

$$\|v\| = \sqrt{v_1^2 + v_2^2 + \cdots + v_{1024}^2}$$

**Por que isso importa aqui.** Um vetor carrega duas informações independentes, e a busca quer apenas uma delas:

| | O que é | Interessa à busca? |
|---|---|---|
| **Direção** | para onde a seta aponta | **sim** — é o assunto do texto |
| **Norma** | quanto ela mede | não |

Duas setas podem apontar para o mesmo lado e ter tamanhos diferentes:

```
        ╱ ●   "O reembolso de refeições em viagem fica limitado a
      ╱        R$ 120,00 por pessoa, exigida a nota fiscal, conforme
    ╱          o Art. 4º §1º do regulamento vigente."
  ╱ ●   "teto de refeição"
╱
```

Os dois textos tratam da mesma coisa; um apenas tem mais palavras. Comparar por **distância** afastaria o segundo do primeiro por um motivo que nada tem a ver com significado. É esse o problema que a §3 resolve: o cosseno mede o **ângulo** e descarta o comprimento, dividindo cada vetor pela própria norma.

> **E no laboratório, a norma é sempre 1.** O `mistral-embed` devolve vetores **já normalizados** — o `00-o-vetor.py` imprime `1.0000` para qualquer texto, e imprimiria para um artigo de trezentas palavras também. O que isso significa é que **a decisão de descartar o comprimento foi tomada rio acima**, pelo próprio modelo, em vez de na hora da comparação. Duas consequências: o cosseno e o produto escalar passam a ser a mesma operação, e a divisão pelas normas na fórmula da §3 está dividindo por 1. Ela continua no código porque nem todo modelo normaliza, e um código que assume isso quebra em silêncio ao trocar de modelo.

---

### 3. Similaridade de cosseno

Dois vetores comparam-se pelo ângulo entre eles:

$$\cos(\theta) = \frac{a \cdot b}{\|a\| \, \|b\|}$$

Em código, são quatro linhas:

```python
def cosseno(a: np.ndarray, b: np.ndarray) -> float:
    return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))
```

**Por que cosseno e não distância euclidiana.** O comprimento do texto vira norma do vetor. Um artigo longo e um resumo dele apontam aproximadamente na mesma direção, mas têm normas diferentes — e a distância euclidiana registraria essa diferença como distância semântica, que ela não é. O cosseno divide a norma fora e olha apenas a direção.

A operação escala de um para muitos sem laço explícito:

```python
def cosseno_lote(consulta: np.ndarray, matriz: np.ndarray) -> np.ndarray:
    normas = np.linalg.norm(matriz, axis=1) * np.linalg.norm(consulta)
    return (matriz @ consulta) / normas
```

Uma multiplicação de matriz por vetor compara a pergunta com o corpus inteiro. Com quarenta chunks, o tempo é imperceptível; a complexidade é $O(n \cdot d)$, linear no número de documentos e na dimensão. O ponto em que essa conta deixa de bastar é o ponto em que um índice aproximado se justifica — e ele fica muito além do que esta disciplina exercita.

---

### 4. Ordenar é buscar

```python
def ranquear(consulta_vec, matriz, rotulos, k=5):
    scores = cosseno_lote(consulta_vec, matriz)
    ordem = np.argsort(-scores)[:k]
    return [(rotulos[i], float(scores[i])) for i in ordem]
```

Não há biblioteca de busca envolvida. Embutir os textos, embutir a pergunta, ordenar por cosseno decrescente: **isso já é um buscador**, e é o buscador que o `01-similaridade.py` executa sobre dez frases.

O que ainda falta a essa função para virar um sistema é persistência — o índice acima vive em memória e morre com o processo. É o que a Aula 07 acrescenta ao trocar o arranjo de numpy por um banco vetorial.

---

### 5. A faixa de valores reais

A intuição estabelecida pela fórmula é que o cosseno varia de $-1$ a $1$, e que textos sem relação ficam próximos de zero. **Essa intuição está errada para textos reais.**

Frases em português compartilham estrutura sintática, vocabulário funcional e registro. Um modelo de embedding captura isso, e o resultado é que dois textos completamente alheios um ao outro raramente descem abaixo de **0,7**. O `01-similaridade.py` inclui, entre as dez frases sobre política de reembolso, duas sobre previsão do tempo e futebol — e elas não chegam perto de zero.

A consequência é uma decisão de projeto, não uma curiosidade:

> **Limiar absoluto escolhido por intuição não funciona.** "Aceito acima de 0,8" é um número sem lastro — e, nesta escala, cortaria fora quase tudo, inclusive o trecho certo. O que funciona é o **ranking** — e, quando um corte for necessário, ele se calibra com dados, como a [nota 02](02-chunking-e-a-medida-da-busca.md) demonstra.

---

### 6. Similaridade não é relevância

Esta é a distinção que organiza o restante da aula, e ela não é um defeito do modelo: é uma propriedade do que se está medindo. Os defeitos propriamente ditos — o que o vetor comprovadamente **não representa** — são assunto da Aula 07.

O buscador ordena por **similaridade**. O que se deseja é **relevância**. As duas coincidem com frequência suficiente para que a diferença passe despercebida — até o dia em que custa caro.

O caso é fácil de construir. Dada a pergunta *"qual o teto de reembolso para refeição em viagem?"*, considere três trechos:

| | Trecho | Relação com a pergunta |
|---|---|---|
| A | "Dúvidas sobre o teto de reembolso para refeição em viagem devem ser encaminhadas ao setor de prestação de contas." | **repete a pergunta**, e não responde |
| B | "Art. 4º §1º — O reembolso de refeições em viagem a serviço fica limitado a R$ 120,00 por pessoa por refeição." | **responde** |
| C | "O colaborador deve apresentar a nota fiscal no prazo de 30 dias corridos contados da data da despesa." | não trata do assunto |

O trecho A compartilha com a pergunta quase todas as palavras de conteúdo. O trecho B compartilha menos, e é o único que contém a informação pedida. A ordenação por cosseno não distingue as duas situações, porque não representa a diferença entre *falar sobre* um assunto e *responder* uma pergunta a respeito dele.

Duas consequências para o resto do curso:

1. **O `k` da busca precisa ser maior que 1.** Se apenas o primeiro colocado for recuperado, o trecho A elimina o B. Recuperar três e deixar a decisão para a etapa seguinte é mais barato que acertar o primeiro sempre.
2. **A etapa seguinte precisa saber recusar.** Se os três trechos recuperados forem do tipo A e C, a resposta correta é declarar que a informação não está disponível — e é o assunto da Aula 07.

---

## Exemplos

### Exemplo 1 — Um vetor, e depois dois

```python
REGRA = "O reembolso de refeições em viagem fica limitado a R$ 120,00."
vetor = gerar_vetor_embeddings(REGRA)

vetor.shape[0]              # dimensão: fixa por modelo
np.linalg.norm(vetor)       # norma: o comprimento do vetor (§2.2)
vetor[:8]                   # oito números sem significado individual
```

A dimensão não muda ao trocar o texto por um artigo de trezentas palavras. A norma é o comprimento do vetor, e é o que o cosseno divide fora — é por isso que um resumo e o texto resumido ficam próximos, apesar da diferença de tamanho. No laboratório essa divisão já veio feita: o `mistral-embed` normaliza, e a norma impressa é sempre `1.0000` (§2.2).

Nada do que está acima é utilizável. O que é utilizável aparece com o segundo vetor:

```
A  "O reembolso de refeições em viagem fica limitado a R$ 120,00."
B  "Quanto posso gastar almoçando numa viagem a trabalho?"
C  "O campeonato de futebol começa no próximo domingo às dezesseis horas."

cosseno(A, B) = 0,8464    <- dizem a mesma coisa, sem compartilhar palavras
cosseno(A, C) = 0,7292    <- não têm relação nenhuma
```

*(Valores medidos com `mistral-embed`; o `00-o-vetor.py` os reproduz.)*

Dois números, e os dois ensinam. O primeiro é a razão de existir do embedding: A e B não compartilham quase nenhum termo e a busca por palavra-chave não os aproximaria. O segundo é a surpresa que a §5 trata: o piso não é zero.

### Exemplo 2 — A busca, com a faixa visível

Pergunta: *"Quanto posso gastar em uma refeição durante uma viagem?"*, contra dez frases das quais duas são alheias ao assunto.

```
  0.8702  ███████████████████████████  refeições em viagem ... R$ 120,00
  0.8019  ████████████████████████     diária de hotel ...
  0.7919  ████████████████████████     despesas acima de R$ 500,00 ...
  0.7890  ███████████████████████      bebida alcoólica não é reembolsável
  0.7835  ███████████████████████      prazo de 30 dias ...
  0.7765  ███████████████████████      multas e estacionamento ...
  0.7697  ███████████████████████      material de escritório ...
  0.7597  ██████████████████████       corridas de aplicativo ...
  0.7488  ██████████████████████       previsão do tempo ...
  0.7487  ██████████████████████       campeonato de futebol ...
```

*(Valores medidos com `mistral-embed`.)*

A leitura que importa não é o primeiro colocado — é o **último**, e a **distância entre os dois**.

A frase sobre futebol não tem relação alguma com a pergunta e pontua **0,7487**. A que responde pontua **0,8702**. A faixa inteira, do mais relevante ao completamente alheio, cabe em **0,12** — e não nos 0,87 que a intuição de "0 a 1" sugere.

Duas consequências práticas saem daí:

**Um limiar absoluto é inútil.** Qualquer corte entre 0,75 e 0,87 separa relevante de irrelevante *neste* conjunto, e nada garante que o mesmo valor sirva na próxima pergunta. O que ordena é o **ranking**.

**As barras enganam quando começam em zero.** Desenhadas de 0 a 1, as dez ficam quase do mesmo tamanho — o que é fiel ao número e inútil para o olho. Ler esta escala exige olhar a diferença, não o comprimento.

---

## Fontes e leituras

DEVLIN, J. et al. **BERT**: pre-training of deep bidirectional transformers for language understanding. arXiv:1810.04805, 2018.

MIKOLOV, T. et al. **Efficient estimation of word representations in vector space**. arXiv:1301.3781, 2013.

MUENNIGHOFF, N. et al. **MTEB**: massive text embedding benchmark. arXiv:2210.07316, 2022.

REIMERS, N.; GUREVYCH, I. **Sentence-BERT**: sentence embeddings using siamese BERT-networks. arXiv:1908.10084, 2019.

VASWANI, A. et al. **Attention is all you need**. arXiv:1706.03762, 2017.

MISTRAL AI. **Embeddings**: documentação do modelo `mistral-embed`.

**Material da disciplina.** Aula 01, [nota 01 §6.1](../../aula-01-llms-e-agentes/notas-de-aula/01-redes-neurais-e-transformer.md) — as três linhagens do Transformer. Aula 02, [nota 01 §5](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — modalidade e especialização.
