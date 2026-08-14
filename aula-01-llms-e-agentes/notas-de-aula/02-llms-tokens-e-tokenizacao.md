# IA Aplicada com LLMs — Aula 01: Fundamentos de LLMs e agentes — Grandes modelos de linguagem, tokens e tokenização

## Introdução

A Parte 1 terminou num ponto específico: a última camada do Transformer produz uma **distribuição de probabilidade sobre o vocabulário**, e um token é sorteado dela. Ficaram duas perguntas em aberto.

A primeira é **o que é esse vocabulário**. "Token" foi usado dezenas de vezes como se fosse óbvio, mas não é: não é letra, não é palavra, e a escolha de como quebrar o texto tem consequências que você sente na fatura e no comportamento do modelo. É por isso que o modelo erra ao contar quantas letras "r" existem em "morango" — e você vai entender exatamente por quê.

A segunda é **o que "grande" significa** em "grande modelo de linguagem". Modelo de linguagem existe desde os anos 1950; a novidade não é o conceito, é a escala e o que ela produziu de qualitativamente novo.

Esta nota fecha as duas. E fecha também uma terceira coisa que a Parte 1 deixou pendente: **como a distribuição vira texto** — logits, softmax, `temperature`, `top_p`. É a parte da caixa que você controla diretamente por parâmetro de API.

> **Pré-requisito:** a Parte 1 desta aula ([`01-redes-neurais-e-transformer.md`](01-redes-neurais-e-transformer.md)). Especialmente a seção 6.6, que lista os 7 passos da chamada de API — esta nota detalha os passos 1 (tokenização) e 5 (sorteio).

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- Definir **modelo de linguagem** formalmente e explicar por que os modelos de n-gramas falharam.
- Explicar o que **"grande"** acrescenta: os três eixos de escala e a mudança qualitativa que veio com eles.
- Justificar por que **alucinação** é o comportamento esperado do objetivo de treino, não um defeito.
- Explicar por que o modelo **não tem memória** entre chamadas e o que isso implica para construir um chatbot.
- Descrever a **decodificação**: logits, softmax, `temperature`, `top_p`, `top_k` — e escolher os valores por tipo de tarefa.
- Explicar o que é um **token** e por que a tokenização por subpalavra venceu caracteres e palavras.
- Executar o algoritmo **BPE** à mão e em código.
- **Medir** a contagem de tokens de um texto e estimar custo e ocupação da janela de contexto.
- Identificar as falhas do modelo que são **consequência da tokenização**.

---

## Desenvolvimento teórico

### 1. O que é um modelo de linguagem

#### 1.1 A definição

Antes de "grande", vamos ao substantivo. Um **modelo de linguagem** é uma função que atribui **probabilidade a sequências de texto**. Nada além disso.

Dado um texto, ele responde: *quão provável é que esta sequência apareça na linguagem que eu modelo?* Um bom modelo de português atribui probabilidade alta a "o gato subiu no telhado" e baixa a "telhado no subiu gato o" — e baixíssima a "o gato subiu no fotossíntese".

Escrito com todo o rigor, o modelo estima $P(w_1, w_2, \ldots, w_n)$: a probabilidade conjunta de uma sequência inteira.

#### 1.2 A regra da cadeia: por que "prever o próximo token" é suficiente

Estimar a probabilidade de uma sequência inteira parece intratável. Mas a **regra da cadeia** da probabilidade a decompõe em fatores condicionais:

$$P(w_1, \ldots, w_n) = P(w_1) \cdot P(w_2 \mid w_1) \cdot P(w_3 \mid w_1, w_2) \cdots P(w_n \mid w_1, \ldots, w_{n-1})$$

$$P(w_1, \ldots, w_n) = \prod_{i=1}^{n} P(w_i \mid w_1, \ldots, w_{i-1})$$

Leia o que isso diz: **modelar a linguagem inteira se reduz a saber prever o próximo token dado o anterior.** Cada fator é exatamente a saída do Transformer que você viu na Parte 1.

Isso amarra três coisas que pareciam separadas:

| Conceito | O que é |
|---|---|
| Objetivo de treino | maximizar $P(w_i \mid \text{contexto})$ nos dados — a entropia cruzada |
| Arquitetura | o decoder-only com máscara causal, que só olha o passado |
| Geração | amostrar $w_i$, anexar ao contexto, repetir |

Não são três decisões independentes que por acaso combinam. São **a mesma decisão** vista de três ângulos — e é por isso que a máscara causal da Parte 1 não era um detalhe técnico, mas a própria definição do problema.

#### 1.3 Antes das redes: modelos de n-gramas

A ideia é dos anos 1940–50 (Shannon), e a implementação clássica é honestamente simples: **conte**.

Contar $P(w_i \mid \text{todo o passado})$ é impossível — sequências longas nunca se repetem. Então assume-se a **hipótese de Markov**: só as $n-1$ palavras anteriores importam. Num modelo de **bigramas** ($n = 2$):

$$P(w_i \mid w_{i-1}) = \frac{\text{contagem}(w_{i-1}, w_i)}{\text{contagem}(w_{i-1})}$$

Exemplo com um corpus minúsculo de três frases:

```
o gato subiu no telhado
o gato dormiu no sofá
o cachorro dormiu no tapete
```

Contagens depois de "o": `gato` 2, `cachorro` 1. Logo $P(\text{gato} \mid \text{o}) = 2/3$ e $P(\text{cachorro} \mid \text{o}) = 1/3$.
Contagens depois de "no": `telhado` 1, `sofá` 1, `tapete` 1 → um terço cada.

Funciona. E foi o estado da arte em reconhecimento de fala e tradução por **décadas**. Mas tem dois problemas fatais:

**1. Esparsidade.** Qualquer sequência não vista no corpus recebe probabilidade **zero** — o que zera a probabilidade da frase inteira, pela multiplicação da regra da cadeia. "o cachorro subiu no telhado" é perfeitamente português, mas o corpus acima lhe dá probabilidade zero. Existem técnicas de suavização (Katz, Kneser-Ney) para redistribuir massa, mas são remendos.

**2. Nenhuma generalização.** Este é o problema profundo. Para o modelo de n-gramas, "gato" e "cachorro" são **símbolos completamente distintos** — dois índices numa tabela, sem nenhuma relação. Ter aprendido que gatos dormem não ensina nada sobre cachorros. O modelo não tem **noção de similaridade**.

E o contexto é curto por construção: um modelo de 5-gramas simplesmente **não pode** ligar "chave" a "estava" com 17 palavras entre eles (o exemplo da Parte 1, seção 4.1). Aumentar $n$ não resolve — a tabela de contagens cresce exponencialmente e a esparsidade piora.

#### 1.4 O que a rede neural mudou

A rede neural atacou exatamente esses dois pontos:

- **Representação distribuída.** Em vez de um índice opaco, cada token vira um **vetor** (o *embedding* da Parte 1). Tokens de sentido parecido acabam com vetores parecidos, então o que se aprende sobre um **transfere** para o outro. Adeus problema 2.
- **Contexto longo.** Primeiro com RNNs, depois — de verdade — com a atenção do Transformer, que liga qualquer par de posições em um passo.

E há um bônus: em vez de uma tabela de contagens que cresce com o corpus, o conhecimento fica **comprimido nos pesos**. Um modelo de 7 bilhões de parâmetros treinado em trilhões de tokens é uma compressão com perdas da distribuição daquele texto — e essa palavra, **compressão**, é a chave da seção 2.4.

---

### 2. O que "grande" significa

#### 2.1 Os três eixos

"Grande" não é um adjetivo de marketing; refere-se a três grandezas que crescem juntas:

| Eixo | O que mede | Ordem de grandeza hoje |
|---|---|---|
| **Parâmetros** | os pesos aprendidos (a conta da Parte 1, seção 6.3) | $10^9$ a $10^{12}$ |
| **Dados de treino** | tokens vistos no pré-treino | $10^{12}$ a $10^{13}$ |
| **Computação** | operações de ponto flutuante no treino | $10^{22}$ a $10^{25}$ FLOPs |

As leis de escala da Parte 1 (seção 6.3) dizem que os três precisam crescer de forma equilibrada — e o Chinchilla deu a razão prática: **~20 tokens de treino por parâmetro**.

#### 2.2 A escala, em perspectiva

| Modelo | Ano | Parâmetros | Contexto original |
|---|---|---|---|
| n-gramas | 1990s | tabela de contagens | 2–5 tokens |
| LSTM de referência | 2015 | ~10 milhões | ~100 tokens |
| GPT-2 | 2019 | 1,5 bilhões | 1.024 tokens |
| GPT-3 | 2020 | 175 bilhões | 2.048 tokens |
| Mistral 7B | 2023 | 7,2 bilhões | 8.192 tokens (janela deslizante) |
| Modelos de fronteira | 2024– | não divulgado | $10^5$ a $10^6$ tokens |

Repare no salto de contexto: de 5 tokens nos n-gramas para centenas de milhares. É o que permite jogar um documento inteiro no prompt — e, pelo custo quadrático da Parte 1 (seção 5.9), também o que torna isso caro.

#### 2.3 A mudança qualitativa

Se fosse só "o mesmo modelo, maior", não haveria curso. O que aconteceu é que, passado certo ponto de escala, apareceram comportamentos que **ninguém programou**:

- **In-context learning** — descrever a tarefa no prompt, com zero ou poucos exemplos, e o modelo executa. Sem treinar nada. Essa é a base de tudo que você vai fazer no resto do curso: quando você escreve `"Resuma o texto em uma sentença"` no [`02-automacao.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula01-hello-world/02-automacao.py), não existe nenhum "modelo de sumarização" — existe um modelo que, dado esse contexto, continua o texto de um jeito que é um resumo.
- **Seguir instruções em linguagem natural**, incluindo formato ("responda em JSON") — o que o [`03-integracao-software.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula01-hello-world/03-integracao-software.py) explora para virar uma peça de software.
- **Raciocínio em passos**, quando induzido ("pense passo a passo").
- **Transferência entre idiomas e domínios** sem treino específico.

Esses comportamentos foram chamados de **capacidades emergentes** (Wei et al., 2022): ausentes em modelos pequenos, presentes nos grandes.

> **Honestidade intelectual:** a palavra "emergente" é disputada. Schaeffer et al. (2023) argumentaram que a aparência de surgimento súbito é em boa parte artefato de **métricas descontínuas** (acerto/erro exato) — com métricas contínuas, a melhora é gradual. Não há controvérsia sobre o *fato* de que modelos grandes fazem coisas que os pequenos não fazem; a controvérsia é se há um "salto" ou uma curva suave. Vale saber disso antes de repetir a palavra em prova.

#### 2.4 O que o modelo realmente guardou

Aqui está o ponto mais importante desta nota, e o que separa quem usa de quem constrói.

Um LLM **não é um banco de dados de fatos**. É uma **compressão com perdas da distribuição estatística do texto** em que foi treinado.

Pense na conta: um modelo de 7B com pesos de 16 bits ocupa ~14 GB. Ele foi treinado em trilhões de tokens — algo como dezenas de terabytes de texto. A razão de compressão é de milhares para um. **Nada** é armazenado literalmente; o que sobrevive são os **padrões** que ajudam a prever o próximo token.

Disso decorrem, diretamente, as duas propriedades do modelo que mais confundem quem está começando.

#### 2.5 Consequência 1: alucinação é o comportamento esperado

O objetivo de treino é **plausibilidade estatística**, não verdade. Em nenhum momento o modelo foi otimizado para "dizer a verdade" ou "admitir que não sabe" — foi otimizado para atribuir alta probabilidade ao token que de fato veio a seguir nos dados.

Pergunte a um LLM o título do terceiro artigo de um pesquisador obscuro. O modelo não tem uma tabela para consultar; ele tem uma distribuição sobre "como títulos de artigos dessa área se parecem". Então ele gera um título **plausível** — com a mesma fluência e a mesma confiança de quando acerta. Não existe, no mecanismo, nenhum sinal interno separando "recuperado" de "inventado".

Isso reposiciona a alucinação: **não é um bug a ser corrigido com prompt melhor**, é uma característica estrutural do objetivo de treino. Você não elimina alucinação; você **arquiteta em volta** dela:

- dando ao modelo o texto-fonte no contexto (**RAG**),
- dando a ele **ferramentas** para consultar a verdade em vez de recordá-la (**agentes**, Parte 3),
- e **verificando** a saída (avaliação, *evals*).

As três coisas são, respectivamente, três blocos deste curso.

#### 2.6 Consequência 2: o modelo não tem memória

O modelo é uma **função pura**: mesmos tokens de entrada, mesma distribuição de saída. Os pesos estão **congelados** desde o fim do treino — a sua conversa não os altera.

Ele não tem estado entre chamadas. Não "lembra" da mensagem anterior. Cada requisição HTTP é independente e amnésica.

Então como o chatbot do [`01-chatbot.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula01-hello-world/01-chatbot.py) parece lembrar? Olhe o que o código faz:

```python
messages.append({"role": "user", "content": user_input})
response = client.chat.completions.create(model="mistral-small-latest", messages=messages)
answer = response.choices[0].message.content
messages.append({"role": "assistant", "content": answer})
```

A cada turno, a lista `messages` **inteira** é reenviada. A memória não está no modelo — está no **seu programa**, que reconstrói o contexto a cada chamada. A "conversa" é uma ilusão mantida pelo cliente.

Três consequências práticas caem direto disso:

1. **O custo de um chat cresce.** Se você reenvia todo o histórico, o turno 20 processa muito mais tokens que o turno 1. O custo acumulado cresce de forma **quadrática** no número de turnos.
2. **A conversa estoura a janela.** Em algum ponto o histórico não cabe mais, e você precisa decidir o que descartar ou resumir.
3. **Gerenciar contexto é trabalho de engenharia** — o seu. É a disciplina que hoje se chama *context engineering*, e a razão pela qual "memória de agente" é um tópico de arquitetura, não um botão da API.

---

### 3. Da distribuição ao texto: decodificação

A Parte 1 (seção 5.8) parou em "a saída é uma distribuição sobre o vocabulário". Agora: como se escolhe um token.

#### 3.1 Logits e softmax

A última camada produz um vetor de números reais sem restrição — um por token do vocabulário. Esses números crus são os **logits**. Para um vocabulário de 32 mil tokens, são 32 mil logits por posição.

Logits não são probabilidades: podem ser negativos e não somam 1. O **softmax** faz a conversão:

$$P(\text{token}_i) = \frac{e^{z_i}}{\sum_j e^{z_j}}$$

#### 3.2 Temperatura

A **temperatura** é uma única modificação nessa fórmula: divida os logits por $T$ antes de exponenciar.

$$P(\text{token}_i) = \frac{e^{z_i / T}}{\sum_j e^{z_j / T}}$$

O efeito é geométrico. $T < 1$ **amplifica** as diferenças entre logits (a distribuição fica pontuda, o topo domina). $T > 1$ **encolhe** as diferenças (a distribuição achata, os candidatos improváveis ganham chance). No limite $T \to 0$, o maior logit leva toda a massa: vira `argmax`, determinístico.

Vejamos em números. Suponha que o modelo esteja completando "O céu estava..." e produza estes logits para quatro candidatos:

```python
import numpy as np

tokens = ["azul", "claro", "cinza", "roxo"]
logits = np.array([3.2, 2.1, 1.0, 0.2])

def softmax_T(logits, T):
    z = logits / T
    z = z - z.max()          # estabilidade numérica
    e = np.exp(z)
    return e / e.sum()

print(f"{'T':>5}  " + "  ".join(f"{t:>7s}" for t in tokens))
for T in [0.1, 0.5, 1.0, 1.5, 2.0]:
    p = softmax_T(logits, T)
    print(f"{T:5.1f}  " + "  ".join(f"{v:6.1%} " for v in p))
```

```
    T     azul    claro    cinza     roxo
  0.1  100.0%     0.0%     0.0%     0.0%
  0.5   88.8%     9.8%     1.1%     0.2%
  1.0   67.0%    22.3%     7.4%     3.3%
  1.5   54.2%    26.0%    12.5%     7.3%
  2.0   46.9%    27.0%    15.6%    10.5%
```

Leia a tabela como o painel de controle que ela é. Com $T = 0{,}1$, "azul" é praticamente certo — o modelo é previsível e repetitivo. Com $T = 2{,}0$, "roxo" tem 10,5% de chance, e o texto fica variado ao custo de coerência. Note que **os logits não mudaram**: a temperatura não altera o que o modelo "pensa", só o quanto ele arrisca ao escolher.

#### 3.3 top_k e top_p

Temperatura mexe na forma da distribuição inteira, inclusive na cauda de dezenas de milhares de tokens absurdos. Duas técnicas cortam a cauda antes de sortear:

- **top_k** — considere apenas os $k$ tokens mais prováveis, descarte o resto, renormalize. Simples, mas $k$ fixo é ruim: às vezes há 2 continuações razoáveis, às vezes 200.
- **top_p** (*nucleus sampling*, Holtzman et al., 2019) — ordene por probabilidade e mantenha os tokens até a soma acumulada atingir $p$. Com `top_p = 0.9`, você mantém o **núcleo** que concentra 90% da massa, seja ele de 3 ou de 300 tokens. O tamanho do corte se **adapta** à confiança do modelo.

Na prática, `top_p` é o mais usado dos dois. E a recomendação padrão é **ajustar um, não os dois** — eles interagem de formas difíceis de raciocinar.

| Tarefa | `temperature` | Por quê |
|---|---|---|
| Extração de dados, classificação, JSON | 0 – 0,3 | quer o token mais provável; variação é defeito |
| Resposta factual, código | 0,2 – 0,5 | precisão com um pouco de folga |
| Redação, explicação | 0,7 – 1,0 | fluência e variedade |
| Criação, brainstorm | 1,0 – 1,5 | quer o inesperado |

O [`03-integracao-software.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula01-hello-world/03-integracao-software.py) pede um JSON de cadastro — é caso de temperatura baixa. Como o script não passa `temperature`, ele usa o padrão do provedor; passar `temperature=0` explicitamente deixaria a saída bem mais estável.

#### 3.4 Determinismo não é garantido

Cuidado com uma armadilha: **`temperature=0` não garante saída idêntica** entre chamadas.

Com $T = 0$ a escolha *dentro de uma inferência* é determinística, mas o resultado ainda pode variar entre requisições porque:

- a infraestrutura agrupa requisições em **lotes** de tamanho variável, e a ordem das operações de ponto flutuante muda com o lote (soma de floats não é associativa);
- modelos do tipo *mixture-of-experts* roteiam tokens para especialistas de forma sensível ao lote;
- o provedor pode atualizar o modelo por trás de um alias como `mistral-small-latest`.

A lição de engenharia: **não escreva testes que exijam string idêntica** da saída de um LLM. Teste propriedades — "é JSON válido", "contém os três campos", "o CPF casa com a regex". Isso volta na aula de *evals*.

---

### 4. Tokens: a unidade de tudo

#### 4.1 O modelo lê inteiros, não texto

Toda a Parte 1 falou de "tokens" atravessando camadas. Concretamente: antes de qualquer conta, o texto passa por um **tokenizador**, que o quebra em pedaços e converte cada pedaço no seu **índice no vocabulário**. É essa sequência de inteiros que entra no modelo, e é dela que sai a resposta, decodificada de volta para texto.

```
"Por que o céu é azul?"
      ↓  tokenizador (encode)
[8582, 1744, 297, 12088, 1289, 21361, 30]        ← ids no vocabulário
      ↓  embeddings + posição + N blocos
distribuição sobre 32 mil tokens → sorteio → id
      ↓  tokenizador (decode)
"Porque"
```

(Os ids acima são ilustrativos — cada modelo tem seu próprio vocabulário, e o mesmo texto vira números diferentes em modelos diferentes.)

> **Conexão com Estruturas de Dados:** o vocabulário é, na prática, uma **tabela hash** `string → inteiro`, mais o vetor inverso `inteiro → string` para decodificar. Buscar o id de um token é O(1). O tokenizador em si é um pouco mais que isso — precisa decidir *como* quebrar, o que veremos em 4.3.

Duas propriedades valem registro: a tokenização é **determinística** (mesmo texto, mesmos ids) e **reversível** (`decode(encode(t)) == t`, nos tokenizadores modernos, inclusive para espaços e emoji).

#### 4.2 Por que não caracteres nem palavras

A escolha da unidade parece arbitrária, mas as duas opções óbvias falham por razões opostas.

**Por caractere?** Vocabulário minúsculo (~100 símbolos), nunca há palavra desconhecida. Mas as sequências ficam **muito longas** — "inteligência artificial" viraria 23 tokens em vez de 3 ou 4. E aqui a Parte 1 cobra a conta: o custo da atenção é **quadrático** no número de tokens (seção 5.9). Multiplicar o comprimento por 5 multiplica o custo da atenção por 25. Inviável.

**Por palavra?** Sequências curtas e unidades com significado. Mas:

- o vocabulário é **aberto** — sempre aparece palavra nova (nome próprio, gíria, termo técnico, erro de digitação), e o que não está no vocabulário vira `<UNK>`, perdendo a informação;
- em línguas morfologicamente ricas como o português, "caminhar / caminhava / caminharíamos / caminhante" são **entradas distintas e sem relação** — de novo o problema de generalização dos n-gramas;
- código-fonte e URLs viram um pesadelo.

**Subpalavra** é o meio-termo que venceu: palavras frequentes ficam inteiras (um token só), palavras raras se quebram em pedaços reaproveitáveis, e **nada** fica fora do vocabulário. O tokenizador *aprende* onde cortar, a partir de estatística do corpus.

| Estratégia | Vocabulário | Comprimento | Palavra desconhecida |
|---|---|---|---|
| Caractere | ~100 | péssimo | impossível |
| Palavra | aberto (infinito) | ótimo | `<UNK>` |
| **Subpalavra** | 32 mil – 130 mil | bom | **impossível** |

#### 4.3 BPE: como o tokenizador é treinado

O algoritmo dominante é o **BPE** (*Byte Pair Encoding*), trazido da compressão de dados (Gage, 1994) para NLP por Sennrich et al. (2016). A ideia é de uma simplicidade desconcertante:

> Comece com caracteres. Repita: encontre o **par de símbolos adjacentes mais frequente** no corpus e funda-o num símbolo novo. Pare quando o vocabulário atingir o tamanho desejado.

Cada fusão é uma **regra de merge**, e a lista ordenada dessas regras *é* o tokenizador. Vamos ver acontecendo, com um corpus de palavras portuguesas escolhidas para ter estrutura comum:

```python
from collections import Counter

corpus = """
baixo baixa baixinho baixaria caixa caixote caixinha
peixe peixinho peixaria trouxe trouxa
""".split()

# cada palavra vira uma sequência de símbolos (caracteres); "_" marca o fim da palavra
vocab = Counter(tuple(p) + ("_",) for p in corpus)

def contar_pares(vocab):
    pares = Counter()
    for simbolos, freq in vocab.items():
        for a, b in zip(simbolos, simbolos[1:]):
            pares[(a, b)] += freq
    return pares

def fundir(vocab, par):
    a, b = par
    novo = Counter()
    for simbolos, freq in vocab.items():
        saida, i = [], 0
        while i < len(simbolos):
            if i < len(simbolos) - 1 and simbolos[i] == a and simbolos[i + 1] == b:
                saida.append(a + b); i += 2
            else:
                saida.append(simbolos[i]); i += 1
        novo[tuple(saida)] += freq
    return novo

for passo in range(1, 9):
    pares = contar_pares(vocab)
    par, freq = pares.most_common(1)[0]
    if freq < 2:
        break
    vocab = fundir(vocab, par)
    print(f"{passo}. funde {par[0]!r} + {par[1]!r}  (aparece {freq}x)  ->  {par[0] + par[1]!r}")

print("\ncomo algumas palavras ficaram:")
for alvo in ("caixinha_", "peixaria_", "trouxa_"):
    for simbolos in vocab:
        if "".join(simbolos) == alvo:
            print(f"  {alvo:12s} -> {list(simbolos)}")
```

Saída:

```
1. funde 'i' + 'x'  (aparece 10x)  ->  'ix'
2. funde 'a' + 'ix'  (aparece 7x)  ->  'aix'
3. funde 'a' + '_'  (aparece 6x)  ->  'a_'
4. funde 'b' + 'aix'  (aparece 4x)  ->  'baix'
5. funde 'o' + '_'  (aparece 3x)  ->  'o_'
6. funde 'i' + 'n'  (aparece 3x)  ->  'in'
7. funde 'in' + 'h'  (aparece 3x)  ->  'inh'
8. funde 'c' + 'aix'  (aparece 3x)  ->  'caix'

como algumas palavras ficaram:
  caixinha_    -> ['caix', 'inh', 'a_']
  peixaria_    -> ['p', 'e', 'ix', 'a', 'r', 'i', 'a_']
  trouxa_      -> ['t', 'r', 'o', 'u', 'x', 'a_']
```

Olhe o que apareceu sem que ninguém tenha ensinado gramática ao algoritmo:

- `caix` — o **radical** de "caixa/caixote/caixinha";
- `inh` — o **sufixo de diminutivo**;
- `a_` e `o_` — as **terminações de gênero** do português.

O BPE redescobriu morfologia por pura contagem de frequência. É isso que faz a tokenização por subpalavra funcionar tão bem: os pedaços tendem a ter significado, e o modelo pode generalizar entre palavras que compartilham pedaços. "caixinha" e "caixote" compartilham `caix`, então o que a rede aprende sobre um informa o outro — exatamente o que faltava aos n-gramas (seção 1.3).

Note também `peixaria_`, que ficou fragmentada: só apareceu uma vez no corpus e teve poucas fusões aplicáveis. Com um corpus real de trilhões de tokens, isso praticamente não acontece com palavras comuns — mas acontece, sim, com nomes próprios, termos técnicos e palavras de línguas sub-representadas. Guarde esse fato para a seção 5.2.

#### 4.4 Variantes que você vai encontrar

| Variante | Onde | Diferença |
|---|---|---|
| **BPE** | GPT-2, GPT-4, Llama, Mistral | funde o par mais frequente |
| **WordPiece** | BERT | funde o par que mais aumenta a verossimilhança, não o mais frequente |
| **SentencePiece** | Llama, T5 | opera no texto cru, sem pré-segmentar por espaço — trata o espaço como caractere (`▁`), o que funciona melhor em línguas sem espaçamento |
| **Byte-level BPE** | GPT-2 em diante | opera sobre **bytes**, não caracteres Unicode — garante que *qualquer* entrada possível seja representável, sem nunca precisar de `<UNK>` |

O truque do **byte-level** merece destaque: com 256 bytes como alfabeto base, qualquer sequência de bytes — emoji, ideograma, arquivo binário colado por engano — é sempre tokenizável. É por isso que tokenizadores modernos nunca falham com `<UNK>`.

Tamanhos típicos de vocabulário: 32 mil (Llama 2, Mistral 7B), ~100 mil (GPT-4), até ~130 mil nos tokenizadores multilíngues mais recentes. Vocabulário maior encurta as sequências (bom para custo) ao preço de uma matriz de embeddings maior — mais um trade-off de engenharia, não uma resposta certa.

---

### 5. Tokenização na prática

#### 5.1 Medindo de verdade

Estimar tokens "de cabeça" é fonte inesgotável de erro. Meça. O `tiktoken` já está no [`requirements.txt`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/requirements.txt) do repositório de código do curso:

```python
import tiktoken
enc = tiktoken.get_encoding("cl100k_base")

for palavra in ["morango", "strawberry", "inteligência", "paralelepípedo", "tokenização"]:
    ids = enc.encode(palavra)
    pedacos = [enc.decode([i]) for i in ids]
    print(f"{palavra:16s} {len(ids)} tokens -> {pedacos}")
```

```
morango          2 tokens -> ['mor', 'ango']
strawberry       3 tokens -> ['str', 'aw', 'berry']
inteligência     3 tokens -> ['int', 'elig', 'ência']
paralelepípedo   5 tokens -> ['par', 'ale', 'lep', 'í', 'pedo']
tokenização      2 tokens -> ['token', 'ização']
```

Repare em `tokenização` → `['token', 'ização']`: o tokenizador achou o radical técnico e o sufixo. E em `paralelepípedo`, quebrada em 5 pedaços sem nenhum significado — palavra longa e rara é onde a fragmentação aparece.

> **Nota sobre qual tokenizador:** `cl100k_base` é o tokenizador do GPT-4, usado aqui porque é público e instalável sem autenticação. O `mistral-small-latest` usa um tokenizador **diferente**, e os números seriam outros. Os *padrões* que veremos a seguir valem para os dois; as **contagens exatas**, não. Para a contagem exata da sua chamada, use 5.3.

#### 5.2 Português custa mais — e quanto

Os tokenizadores dos modelos populares são treinados em corpora majoritariamente em inglês. Consequência: padrões do inglês viram um token; palavras em português se quebram em vários. Medindo o mesmo parágrafo nos dois idiomas:

```python
en = ("Large language models are trained to predict the next token in a sequence. "
      "This simple objective, applied to enormous amounts of text, produces systems "
      "capable of translating, summarizing, writing code and holding a conversation.")
pt = ("Grandes modelos de linguagem são treinados para prever o próximo token de uma "
      "sequência. Esse objetivo simples, aplicado a quantidades enormes de texto, produz "
      "sistemas capazes de traduzir, resumir, escrever código e manter uma conversa.")

for nome, texto in [("inglês", en), ("português", pt)]:
    ids = enc.encode(texto)
    print(f"{nome:11s} {len(texto):4d} caracteres  {len(ids):4d} tokens  "
          f"{len(texto)/len(ids):.2f} chars/token")
```

```
inglês        229 caracteres    41 tokens  5.59 chars/token
português     237 caracteres    60 tokens  3.95 chars/token

português usa 46.3% mais tokens para o mesmo conteúdo
```

**46% mais tokens** — para o mesmo conteúdo, com quantidade de caracteres praticamente igual. Isso é 46% mais custo, 46% mais ocupação da janela de contexto e mais latência.

Mas cuidado com generalizar demais. Medindo palavra por palavra:

| Inglês | tokens | Português | tokens |
|---|---|---|---|
| `cat` | 1 | `gato` | 2 |
| `dog` | 1 | `cachorro` | 3 |
| `house` | 1 | `casa` | 2 |
| `The book is on the table` | 6 | `O livro esta sobre a mesa` | **6** |

A última linha é um **contraexemplo**: mesma contagem. A regra não é "português sempre custa mais por palavra" — é que **palavras portuguesas frequentemente se fragmentam mais**, e o efeito se acumula em textos longos. Frases com palavras curtas e comuns podem empatar. Meça o seu caso; não confie na regra de bolso.

#### 5.3 Contando os tokens da sua chamada real

A resposta da API traz a contagem exata, cobrada pelo provedor, no campo `usage`:

```python
response = client.chat.completions.create(
    model="mistral-small-latest",
    messages=[{"role": "user", "content": "Por que o céu é azul?"}],
)

print(response.choices[0].message.content)
print("tokens de entrada :", response.usage.prompt_tokens)
print("tokens de saída   :", response.usage.completion_tokens)
print("total             :", response.usage.total_tokens)
```

Esta é a fonte de verdade — o tokenizador do próprio modelo, incluindo os tokens especiais que o provedor injeta para marcar os papéis `system`/`user`/`assistant` (que também são cobrados). Rode isso no seu ambiente com a chave configurada; é o exercício da aula.

#### 5.4 Onde a tokenização vaza para o comportamento

Várias falhas famosas de LLM não são falhas de raciocínio — são consequência direta de o modelo **nunca ver letras**.

**Contar letras.** "Quantos R tem a palavra `strawberry`?" O modelo vê `['str', 'aw', 'berry']`. A informação "existem três caracteres `r` distribuídos por estes três pedaços" simplesmente **não está** na entrada de forma acessível — está diluída nos vetores dos tokens. Ele precisa ter memorizado a grafia indiretamente. Errar é o esperado; acertar é que é notável.

**Aritmética.** Números se quebram de forma inconsistente (`1234` pode virar um ou vários tokens, dependendo do tokenizador). O algoritmo de soma que você aprendeu opera em **dígitos alinhados**; o modelo opera em pedaços de tamanho arbitrário. Daí LLMs errarem multiplicações longas e a solução correta ser dar a ele uma **calculadora como ferramenta** (Parte 3) em vez de melhorar o prompt.

**Inverter string, contar sílabas, rimar, acrósticos.** Todas tarefas em nível de caractere, todas prejudicadas pelo mesmo motivo.

**Código e espaços em branco.** Em Python a indentação é sintaxe. Tokenizadores modernos tratam sequências de espaços como tokens próprios exatamente para não destruir isso — um dos motivos pelos quais modelos ficaram melhores em código.

O padrão a extrair: **quando a tarefa é sobre os caracteres, e não sobre o significado, desconfie do modelo e dê a ele uma ferramenta.** Essa é literalmente a tese da Parte 3.

#### 5.5 Custo e janela: a conta que você vai fazer sempre

Duas contas que todo AI Engineer faz de cabeça.

**Cabe na janela?** Some os tokens de: *system prompt* + histórico da conversa + documentos que você injetou (RAG) + **espaço reservado para a resposta**. Esquecer o último é erro clássico: se o prompt ocupa a janela inteira, não sobra espaço para gerar.

**Quanto custa?** Provedores cobram **por token**, com preços diferentes para entrada e saída (saída costuma ser mais caro). O padrão é preço por milhão de tokens:

$$\text{custo} = \frac{\text{tokens}_{\text{entrada}}}{10^6} \times p_{\text{entrada}} + \frac{\text{tokens}_{\text{saída}}}{10^6} \times p_{\text{saída}}$$

Consulte a tabela de preços vigente do provedor — ela muda com frequência, e qualquer número escrito aqui estaria errado em poucos meses. O exercício 1 faz essa conta com valores explicitamente fictícios, para você praticar a estrutura do cálculo.

---

### 6. Fechando: a definição operacional de um LLM

Juntando as duas notas desta aula, aqui está a definição com que você vai trabalhar no resto do curso:

> Um **LLM** é uma função sem estado que recebe uma sequência de tokens e devolve uma distribuição de probabilidade sobre o próximo token. Ela é implementada como um Transformer decoder-only com bilhões de parâmetros, treinado para minimizar a surpresa sobre trilhões de tokens de texto e depois alinhado para seguir instruções. Texto é gerado aplicando essa função repetidamente, sorteando um token por vez.

```
tokens de entrada ──> [ LLM: função pura, pesos congelados ] ──> distribuição
                                                                      │
        contexto + token sorteado <────── sorteio (temperature) <─────┘
```

E o que ele **não** é — cada item aqui é um problema de arquitetura que o resto do curso resolve:

| Não é | Por quê | Solução |
|---|---|---|
| Banco de dados de fatos | é compressão com perdas; alucina com fluência | RAG |
| Calculadora ou intérprete | opera em tokens, não em dígitos ou semântica de execução | ferramentas |
| Sistema com memória | função pura, sem estado entre chamadas | memória externa gerenciada por você |
| Agente | só produz o próximo token; não age no mundo | **o laço agente + ferramentas** |

A última linha é a Parte 3.

---

## Exemplos

### Exemplo 1 — O efeito da temperatura, no seu terminal

O exemplo da seção 3.2 mostra a matemática. Este mostra o efeito no texto, usando a API do curso:

```python
import os
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv()
client = OpenAI(base_url="https://api.mistral.ai/v1",
                api_key=os.environ.get("OPENAI_API_KEY"))

prompt = "Invente um nome para uma cafeteria especializada em café coado."

for T in [0.0, 0.7, 1.5]:
    print(f"\n--- temperature = {T} ---")
    for tentativa in range(3):
        r = client.chat.completions.create(
            model="mistral-small-latest",
            messages=[{"role": "user", "content": prompt}],
            temperature=T,
        )
        print(" ", r.choices[0].message.content.strip())
```

O que observar: com `T = 0.0`, as três tentativas tendem a ser iguais ou quase (com as ressalvas da seção 3.4); com `T = 1.5`, as três divergem e podem ficar estranhas. Rode e cole o resultado no seu relatório — é o entregável desta parte.

### Exemplo 2 — Quanto do seu prompt é *system prompt*?

Uma medição que costuma surpreender: em prompts curtos, a instrução de sistema domina o custo.

```python
import tiktoken
enc = tiktoken.get_encoding("cl100k_base")

system = ("Você é um assistente de cadastro de pessoas em um sistema. Seja amigável "
          "nos seus pedidos. Peça os dados de cadastro: Nome completo, CPF e data de "
          "nascimento.")
usuario = "Oi, quero me cadastrar"

t_sys, t_usr = len(enc.encode(system)), len(enc.encode(usuario))
print(f"system : {t_sys:3d} tokens")
print(f"user   : {t_usr:3d} tokens")
print(f"o system prompt é {t_sys / (t_sys + t_usr):.0%} da entrada")
```

```
system :  42 tokens
user   :   7 tokens
o system prompt é 86% da entrada
```

**86% do que você paga na primeira requisição é instrução sua, não pergunta do usuário.** E o *system prompt* do [`03-integracao-software.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula01-hello-world/03-integracao-software.py) é reenviado **em toda** requisição da conversa. Num chat de 20 turnos, você paga por ele 20 vezes — o que faz de "encurtar o system prompt" uma otimização de custo real, não microotimização.

---

## Exercícios resolvidos

### 1. Estimar o custo de um chatbot

**Enunciado.** Um chatbot de atendimento tem um *system prompt* de 400 tokens. Cada turno adiciona ~80 tokens do usuário e ~150 tokens de resposta, e o histórico inteiro é reenviado a cada turno. Suponha preços fictícios de **US\$ 0,20 por milhão de tokens de entrada** e **US\$ 0,60 por milhão de tokens de saída**. Qual o custo de uma conversa de 10 turnos?

**Resolução.** A saída é simples: 10 turnos × 150 tokens = **1.500 tokens de saída**.

A entrada é o ponto do exercício. No turno $n$, o prompt contém o *system prompt* mais todos os turnos anteriores mais a mensagem atual:

$$\text{entrada}(n) = 400 + (n-1)\times(80 + 150) + 80 = 480 + 230(n-1)$$

| Turno | Tokens de entrada |
|---|---|
| 1 | 480 |
| 2 | 710 |
| 5 | 1.400 |
| 10 | 2.550 |

Somando os 10 turnos — soma de uma progressão aritmética:

$$\sum_{n=1}^{10} [480 + 230(n-1)] = 10 \times 480 + 230 \times \frac{9 \times 10}{2} = 4.800 + 10.350 = 15.150$$

Custo:

$$\frac{15.150}{10^6} \times 0{,}20 + \frac{1.500}{10^6} \times 0{,}60 = 0{,}00303 + 0{,}00090 = \mathbf{US\$\ 0{,}0039}$$

Quatro décimos de centavo — barato. Mas veja a estrutura. Todo o texto **distinto** que chega a aparecer no prompt são 2.550 tokens (400 do *system* + 800 do usuário + 1.350 de respostas anteriores), e você pagou entrada por **15.150**: quase **6× de amplificação**, causada pelo reenvio do histórico (seção 2.6).

E a amplificação piora com o comprimento da conversa, porque a soma cresce com $n^2$: em 40 turnos a entrada acumulada seria de **198.600 tokens**, treze vezes a de 10 turnos para quatro vezes mais conversa. Multiplique por dez mil conversas por dia e a decisão de arquitetura (resumir o histórico? truncar? qual janela?) passa a valer dinheiro de verdade.

### 2. Aplicar temperatura à mão

**Enunciado.** Um modelo produz logits `[2,0; 1,0; 0,0]` para os tokens A, B, C. Calcule as probabilidades com $T = 1{,}0$ e com $T = 0{,}5$, e explique o efeito.

**Resolução.** Com $T = 1$, aplica-se o softmax direto. Exponenciais: $e^2 = 7{,}389$, $e^1 = 2{,}718$, $e^0 = 1{,}000$. Soma: $11{,}107$.

$$P = \left[\frac{7{,}389}{11{,}107},\ \frac{2{,}718}{11{,}107},\ \frac{1{,}000}{11{,}107}\right] = [0{,}665,\ 0{,}245,\ 0{,}090]$$

Com $T = 0{,}5$, primeiro divida os logits: `[4,0; 2,0; 0,0]`. Exponenciais: $e^4 = 54{,}598$, $e^2 = 7{,}389$, $e^0 = 1{,}000$. Soma: $62{,}987$.

$$P = [0{,}867,\ 0{,}117,\ 0{,}016]$$

| Token | $T = 1{,}0$ | $T = 0{,}5$ |
|---|---|---|
| A | 66,5% | **86,7%** |
| B | 24,5% | 11,7% |
| C | 9,0% | 1,6% |

Baixar a temperatura **concentrou** a massa no token mais provável: A subiu 20 pontos e C caiu para quase nada. Os logits não mudaram — só a forma da distribuição. É por isso que temperatura baixa produz saída previsível e repetitiva, e temperatura alta produz variedade com risco de incoerência.

### 3. BPE à mão

**Enunciado.** Corpus: `banana bandana banda` (uma ocorrência de cada), com `_` marcando o fim da palavra. Execute 4 rodadas de BPE. Em caso de empate, escolha o par encontrado primeiro na ordem de leitura do corpus.

**Resolução.** Representação inicial:

```
banana_   → b a n a n a _
bandana_  → b a n d a n a _
banda_    → b a n d a _
```

**Rodada 1.** Contando pares em todo o corpus:

| Par | Ocorrências |
|---|---|
| `(a, n)` | **5** |
| `(b, a)` | 3 |
| `(n, a)` | 3 |
| `(a, _)` | 3 |

Vence `(a, n)` → novo símbolo `an`:

```
banana_   → b an an a _
bandana_  → b an d an a _
banda_    → b an d a _
```

**Rodada 2.** Agora `(b, an)` aparece 3× e `(a, _)` também 3× — empate, resolvido pela ordem de leitura, que encontra `(b, an)` primeiro. Vence `(b, an)` → `ban`:

```
banana_   → ban an a _
bandana_  → ban d an a _
banda_    → ban d a _
```

**Rodada 3.** `(a, _)` aparece 3× e é o máximo. Vence → `a_`:

```
banana_   → ban an a_
bandana_  → ban d an a_
banda_    → ban d a_
```

**Rodada 4.** Empate em 2× entre `(an, a_)` e `(ban, d)`; a ordem de leitura encontra `(an, a_)` primeiro. Vence → `ana_`:

```
banana_   → [ban, ana_]
bandana_  → [ban, d, ana_]
banda_    → [ban, d, a_]
```

**Resultado.** Quatro merges: `an`, `ban`, `a_`, `ana_`. "banana" passou de 6 símbolos para **2 tokens**. E o vocabulário aprendido é linguisticamente sensato: `ban` como início compartilhado, `ana_`/`a_` como terminações.

Duas observações. A regra de desempate é **detalhe de implementação** — bibliotecas diferentes podem desempatar de outro jeito e produzir vocabulários ligeiramente diferentes do mesmo corpus. E uma 5ª rodada fundiria `(ban, d)` → `band`, deixando `bandana_ → [band, ana_]` e `banda_ → [band, a_]` — a cada merge, as palavras encurtam e os pedaços ficam mais parecidos com morfemas.

### 4. Por que o modelo erra ao contar letras

**Enunciado.** Explique por que um LLM frequentemente erra "quantas letras R existem em morango?" e por que insistir no prompt ("conte com cuidado", "pense passo a passo") ajuda pouco. Proponha duas soluções de engenharia.

**Resolução.** O modelo nunca recebe a palavra como sequência de caracteres. O tokenizador entrega `['mor', 'ango']` — dois inteiros. A partir daí, tudo que existe são vetores densos associados a esses dois ids. A informação "o primeiro pedaço contém um caractere `r` na posição 3" **não está disponível** como dado de entrada; ela só existe na medida em que o treino tenha, indiretamente, correlacionado esses tokens com afirmações sobre sua grafia.

Ou seja: o modelo não está *contando* nada. Está **recordando** ou **adivinhando** um número plausível — pelo mesmo mecanismo da alucinação (seção 2.5).

Por que o prompt ajuda pouco: "pense passo a passo" faz o modelo *gerar* passos intermediários, o que ajuda de fato em problemas de **raciocínio** (onde a informação está no contexto e falta encadeá-la). Aqui a informação **não está** no contexto. Gerar passos sobre dados que você não tem produz passos plausíveis e igualmente errados. Prompt não cria informação inexistente na entrada.

Duas soluções de engenharia:

1. **Dar a informação em forma acessível.** Peça a contagem sobre a palavra já separada: `"m-o-r-a-n-g-o"`. Com hifens, cada letra tende a virar um token próprio, e a tarefa passa a ser possível.
2. **Dar uma ferramenta.** Deixe o modelo chamar uma função `contar_letras(palavra, letra)` implementada em Python — que acerta 100% das vezes, é auditável e mais barata que qualquer prompt elaborado.

A segunda é a resposta canônica, e é o assunto da Parte 3: **o modelo decide o que fazer; o código faz.**

### 5. Cabe na janela?

**Enunciado.** Você monta um sistema de RAG sobre um modelo com janela de 8.192 tokens. O *system prompt* tem 300 tokens; cada documento recuperado tem em média 600 tokens; a pergunta do usuário, 50; e você quer reservar espaço para uma resposta de até 800 tokens. Quantos documentos você pode injetar?

**Resolução.** A janela é compartilhada entre **entrada e saída**. Orçamento disponível para os documentos:

$$8.192 - 300 - 50 - 800 = 7.042 \text{ tokens}$$

$$\left\lfloor \frac{7.042}{600} \right\rfloor = \mathbf{11 \text{ documentos}}$$

Mas 11 é o **teto teórico**, e projetar no teto é imprudente por três motivos: 600 é uma média (um documento pode vir com 1.200), o texto em português pode inflar a contagem (seção 5.2), e você precisa de margem para o histórico da conversa. Na prática se usa 5 ou 6 documentos e se mede a ocupação real com `usage.prompt_tokens`.

E há um argumento que não é de espaço: mais contexto não é linearmente melhor. O custo da atenção é **quadrático** (Parte 1, seção 5.9), e enfiar 11 documentos medianamente relevantes tende a produzir resposta pior do que 4 documentos muito relevantes — o sinal se dilui no ruído. Recuperar melhor vale mais que recuperar mais.

---

## Síntese

- Um **modelo de linguagem** atribui probabilidade a sequências. Pela **regra da cadeia**, isso se reduz a prever o próximo token — o que amarra objetivo de treino, arquitetura causal e geração numa coisa só.
- Os modelos de **n-gramas** falharam por **esparsidade** e por **não generalizar** (símbolos sem relação entre si). A rede resolveu ambos com **representações distribuídas** e contexto longo.
- **"Grande"** são três eixos — parâmetros, dados, computação — e o que a escala trouxe de qualitativamente novo: **in-context learning**, seguir instruções, raciocínio em passos.
- Um LLM é **compressão com perdas** da distribuição do texto, não banco de fatos. Daí: **alucinação é o comportamento esperado** do objetivo de treino, e se contorna com arquitetura (RAG, ferramentas, evals), não com prompt melhor.
- O modelo é uma **função pura sem estado**. A memória do chat está no **seu programa**, que reenvia o histórico — o que faz o custo de uma conversa crescer com o **quadrado** do número de turnos.
- **Decodificação**: logits → softmax → sorteio. `temperature` reformata a distribuição ($T$ baixo concentra, $T$ alto achata); `top_p` corta a cauda de forma adaptativa. Ajuste **um** dos dois. `temperature=0` **não** garante saída idêntica.
- **Token** é a unidade real de tudo — custo, janela, cobrança. Subpalavra venceu porque caractere estoura o custo quadrático e palavra estoura o vocabulário.
- **BPE** funde o par adjacente mais frequente, repetidamente. Ele **redescobre morfologia** sozinho: radicais, sufixos e terminações de gênero apareceram no nosso exemplo sem nenhuma regra gramatical.
- **Português usa ~46% mais tokens** que inglês para o mesmo conteúdo (medido) — mas há contraexemplos por frase. **Meça**, com `tiktoken` para estimar e `usage.prompt_tokens` para a verdade.
- Falhas em **contar letras, aritmética e rimas** são consequência da tokenização, não de raciocínio. Quando a tarefa é sobre caracteres, dê uma **ferramenta** ao modelo — o que abre a Parte 3.

---

## Fontes e leituras

**Papers**

- Sennrich, R., Haddow, B., Birch, A. — *Neural Machine Translation of Rare Words with Subword Units* (2016). [arxiv.org/abs/1508.07909](https://arxiv.org/abs/1508.07909) — o BPE aplicado a NLP.
- Kudo, T., Richardson, J. — *SentencePiece* (2018). [arxiv.org/abs/1808.06226](https://arxiv.org/abs/1808.06226)
- Holtzman, A. et al. — *The Curious Case of Neural Text Degeneration* (2019). [arxiv.org/abs/1904.09751](https://arxiv.org/abs/1904.09751) — de onde vem o `top_p`.
- Wei, J. et al. — *Emergent Abilities of Large Language Models* (2022). [arxiv.org/abs/2206.07682](https://arxiv.org/abs/2206.07682)
- Schaeffer, R., Miranda, B., Koyejo, S. — *Are Emergent Abilities of Large Language Models a Mirage?* (2023). [arxiv.org/abs/2304.15004](https://arxiv.org/abs/2304.15004) — leia junto com o anterior.

**Material didático**

- Karpathy, A. — *Let's build the GPT Tokenizer* (YouTube, 2024) — duas horas construindo um BPE do zero. A continuação natural da seção 4.3.
- Jurafsky, D., Martin, J. — *Speech and Language Processing*, 3ª ed., capítulo 3 (modelos de n-gramas). [web.stanford.edu/~jurafsky/slp3](https://web.stanford.edu/~jurafsky/slp3/) — gratuito.
- OpenAI — *Tokenizer* ([platform.openai.com/tokenizer](https://platform.openai.com/tokenizer)) — cole texto e veja a segmentação. Útil para mostrar em aula.
- Documentação da Mistral: [docs.mistral.ai](https://docs.mistral.ai) — parâmetros de amostragem e tokenização do modelo que usamos.

**Nesta disciplina**

- [Parte 1 desta aula](01-redes-neurais-e-transformer.md) — a arquitetura que produz a distribuição sobre tokens.
- [`codigo/aula01-hello-world/`](https://github.com/celsocrivelaro/senac-llm-code/tree/main/aula01-hello-world) — os quatro exemplos, agora legíveis em termos de tokens e amostragem.
