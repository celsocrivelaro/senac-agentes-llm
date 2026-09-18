# IA Aplicada com LLMs — Aula 06: Embeddings e RAG — Bancos vetoriais

## Introdução

Até aqui os vetores moraram numa matriz de `numpy` que morre com o processo. A nota 02 defendeu essa escolha com número, e ela continua certa para 28 chunks.

Esta nota apresenta a alternativa — o **banco vetorial** — e o que ela cobra. Vale adiantar a parte desconfortável: **no corpus desta aula, o banco não traz ganho nenhum de velocidade.** Entender por que ele ainda assim é a escolha certa em produção é o conteúdo da nota.

São quatro perguntas, nesta ordem: como funciona, quando compensa, como se insere e como se busca — mais a que costuma faltar, que é como se lê o que volta.

> **Pré-requisitos:** notas [01](01-o-vetor-e-a-similaridade.md) e [02](02-chunking-e-a-medida-da-busca.md) desta aula.
>
> **Código:** [`05-banco-vetorial.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula06-embeddings-e-rag/05-banco-vetorial.py). As distâncias desta nota saíram de uma execução dele. O `indice_chroma.py` embrulha o mesmo mecanismo com a interface do `IndiceMemoria`.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Explicar** as três funções de um banco vetorial — armazenar, indexar e buscar — e o que cada uma acrescenta a uma matriz de vetores.
- **Distinguir** busca exata de busca aproximada (ANN), e nomear o preço de cada uma.
- **Inserir** dados num banco vetorial, sabendo o papel de cada um dos quatro campos.
- **Buscar** com e sem filtro de metadado, e **decidir** qual das duas o problema pede.
- **Ler** o resultado de uma consulta, distinguindo distância de similaridade e reconhecendo que o banco **nunca** diz "não tenho isso".
- **Justificar** o uso de um banco vetorial pelo argumento correto para a escala em questão.

---

## Desenvolvimento teórico

### 1. O que um banco vetorial faz

Um banco vetorial guarda vetores e devolve os mais parecidos com um vetor de consulta. São três funções, e vale separá-las porque cada uma resolve um problema diferente:

| Função | O que faz | O que a matriz de numpy já fazia |
|---|---|---|
| **Armazenar** | guarda o vetor **e os metadados** ao lado, sem recalcular nada | guardava o vetor; metadado ficava por sua conta |
| **Indexar** | monta uma estrutura que acelera a busca em espaço de alta dimensão | não indexava: comparava com todos |
| **Buscar** | calcula similaridade entre a consulta e os vetores indexados | fazia isso, exaustivamente |

A diferença real está na segunda linha. Armazenar e buscar, `numpy` já faz. **Indexar é o que um banco acrescenta** — e é também de onde vem o único defeito dele, que a §3 mede.

#### 1.1 O que entra no banco

O vetor que se insere pode representar praticamente qualquer coisa. Texto é o caso desta aula, mas a mesma estrutura recebe imagem, áudio e vídeo — e é por isso que a busca por similaridade aparece em recomendação, detecção de fraude e busca reversa de imagem, não só em RAG.

![Tipos de embeddings](https://media.datacamp.com/legacy/v1694511621/image3_a593ee57f6.png)

*Figura: tipos de embeddings. Fonte: DataCamp, [The Top 5 Vector Databases](https://www.datacamp.com/blog/the-top-5-vector-databases).*

O banco não sabe nem se importa com o que o vetor representa. Ele compara números.

---

### 2. Por que a busca não é exaustiva

A matriz de numpy da nota 02 faz o óbvio: compara a consulta com **todos** os vetores e ordena. É exato, e custa `O(n)`.

Com um milhão de vetores, isso deixa de caber no tempo de uma requisição. A saída da indústria é desistir da exatidão:

> **ANN — *approximate nearest neighbour*.** Recupera vetores parecidos **sem varrer o conjunto inteiro**. Em troca, pode não devolver o vizinho mais próximo de verdade.

O nome é honesto: *aproximado* quer dizer que o resultado pode estar errado. A pergunta de projeto não é "quero exato ou aproximado?", e sim **quanto de recall estou disposto a perder por quanto de latência**.

Os algoritmos que aparecem no mercado:

| Algoritmo | Ideia | Onde se vê |
|---|---|---|
| **HNSW** | um grafo de várias camadas: ligações longas em cima, vizinhança densa embaixo. A busca desce de camada em camada | Chroma, Qdrant, pgvector, Weaviate |
| **IVF** | particiona o espaço em células e só varre as células próximas da consulta | Milvus, FAISS |
| **LSH** | uma função de hash joga vetores parecidos no mesmo balde | implementações mais antigas |
| **PQ** | comprime cada vetor num código curto que **preserva distância relativa** | combinado com IVF, para caber em memória |

O HNSW é o mais comum hoje, e é o que o Chroma usa. Vale reter a intuição, não a implementação: **em vez de comparar com todos, navega-se por vizinhos até parar de melhorar.**

#### 2.1 A métrica é escolha, e ela é declarada

O banco não adivinha como comparar. No Chroma, a métrica entra na criação da coleção:

```python
colecao = cliente.get_or_create_collection(
    name="regulamento", metadata={"hnsw:space": "cosine"})
```

As opções usuais são **cosseno** (ângulo — o que esta aula usa), **produto escalar** (ângulo e magnitude juntos) e **distância euclidiana** (a reta entre as pontas). A nota 01 §2.2 já explicou por que o cosseno é o adequado aqui: ele descarta o comprimento do vetor, que não é significado.

> **A métrica entra no carimbo da versão**, junto com a estratégia de corte e o modelo de embedding. Trocá-la muda o `recall` sem que ninguém tenha mudado o código.

---

### 3. Quando o banco compensa

O ganho da busca aproximada está na **escala**, e vem exatamente de não comparar com todos:

| | Complexidade | |
|---|---|---|
| busca exaustiva | `O(n)` | compara com os `n` vetores |
| busca aproximada | `~O(log n)` | navega um grafo de vizinhos |

Traduzindo em decisão:

| Escala | Exaustivo | Veredito |
|---|---|---|
| 28 vetores (esta aula) | instantâneo | o banco é **desnecessário** |
| 100 mil | ainda viável | o banco **começa a compensar** |
| 10 milhões | inviável | o banco é **obrigatório** |

> **A pergunta não é "banco ou numpy". É quantos vetores você tem.**

Em escala pequena o banco chega a **perder** para a matriz: ele paga serialização, checagem de filtro e travessia de grafo para economizar comparações que, em dezenas de vetores, custam quase nada.

Por isso, nesta aula, o argumento para adotar um banco **não é velocidade**. É **persistência**: o índice em numpy morre com o processo, e reconstruí-lo custa uma chamada de embedding sobre o corpus inteiro, toda vez. É a assimetria construir × consultar da nota 02, cobrada entre execuções.

---

### 4. Como se inserem dados

Quatro listas paralelas, e é assim em qualquer banco vetorial — muda o nome dos campos, não o modelo:

```python
colecao.add(
    ids=[f"{i:03d}" for i in range(len(chunks))],          # a chave
    documents=[c["texto"] for c in chunks],                # o texto original
    metadatas=[{"artigo": c["id"],                         # o que dá para filtrar
                "numero": numero_do_artigo(c["id"]),
                "tamanho": len(c["texto"])} for c in chunks],
    embeddings=vetores.tolist(),                           # o vetor, pronto
)
```

| Campo | Papel |
|---|---|
| `ids` | a chave primária. **Reinserir o mesmo id substitui** — é o que torna a atualização de um documento barata |
| `documents` | o texto original. O banco guarda e devolve; não interpreta |
| `metadatas` | os campos estruturados. **É por eles que se filtra**, e é aqui que se resolve o que o vetor não representa |
| `embeddings` | o vetor, calculado fora |

**O vetor vai pronto, e isso é decisão de projeto.** Vários bancos se oferecem para embutir por você, a partir do texto. Aceitar essa oferta esconde qual modelo produziu os vetores — e no dia em que ele mudar, o índice inteiro muda de significado sem aviso. Manter a chamada visível é o mesmo princípio que levou o `embedding.py` a ser a única porta da aula para a API.

---

### 5. Como se buscam dados — quatro situações

#### 5.1 A busca comum

```python
r = colecao.query(query_embeddings=[v_pergunta.tolist()], n_results=3)
```

> *"qual o teto de refeição em viagem?"*

```
  Art. 4º §1º    distância 0,2356
  Art. 4º §2º    distância 0,2402
  Art. 6º §2º    distância 0,2585
```

#### 5.2 A busca com filtro de metadado

A mesma pergunta, restrita ao Art. 9º:

```python
r = colecao.query(query_embeddings=[v_pergunta.tolist()], n_results=3,
                  where={"numero": 9})
```

```
  Art. 9º §2º    distância 0,3269
  Art. 9º §1º    distância 0,3284
  Art. 9º §3º    distância 0,3327
```

**O filtro roda antes da comparação de vetores**, e é a resposta para metade dos problemas que o vetor não resolve: faixa de valor, data e identificador se decidem aqui, com `==` e `<=`, e não com cosseno. É a regra determinística da Aula 05 — o caminho mais barato é o que não usa o modelo — disponível dentro do próprio banco.

> Quando o filtro e o vetor se combinam na mesma consulta, o nome de mercado é **busca híbrida**.

#### 5.3 O texto procurando a si mesmo

```
  Art. 1º        distância 0,0000
```

O piso de baixo. Serve de teste de sanidade: se um texto não encontra a si mesmo em distância zero, alguma coisa está errada no pipeline de indexação — vetores trocados, coleção errada, métrica diferente da esperada.

#### 5.4 A pergunta fora do domínio

> *"qual o índice de reajuste da tabela de fretes marítimos?"*

```
  Art. 5º §1º    distância 0,2586
```

O regulamento não fala de fretes marítimos. E a distância **não é alta** — é praticamente a mesma da pergunta pertinente da §5.1.

É o piso de cosseno da nota 01, reaparecendo com consequência operacional. E leva à observação mais importante desta nota:

> **O banco nunca diz "não tenho isso".** Ele devolve os `k` mais próximos, sempre, mesmo que os `k` sejam todos irrelevantes. Quem decide se o mais próximo é próximo o bastante é o seu código.

Essa decisão é o **portão**, e é a nota 05.

---

### 6. Como se entende o resultado

O retorno do Chroma é um dicionário de **listas de listas**, porque a API aceita várias consultas numa chamada. Com uma consulta só, tudo mora no índice `[0]`:

```python
r["ids"][0]         ['008', '009', '016']
r["distances"][0]   [0.2356, 0.2402, 0.2585]
r["metadatas"][0]   [{'artigo': 'Art. 4º §1º', ...}, ...]
r["documents"][0]   os textos, na mesma ordem
```

As quatro listas são **paralelas**: a posição `i` de cada uma se refere ao mesmo resultado. Percorrê-las com `zip` não é estilo, é o único jeito correto.

E o número que volta é **distância**, não score:

$$d = 1 - \cos(\theta)$$

| Situação | Distância | Cosseno |
|---|---|---|
| texto igual a si mesmo | 0,0000 | 1,0000 |
| pergunta pertinente | 0,2356 | 0,7644 |
| pergunta fora do domínio | 0,2586 | 0,7414 |

**Baixo é bom** — o contrário de tudo o que as notas 01 a 03 imprimiram. Trocar um pelo outro não produz erro de execução: produz um ranking invertido, em silêncio, e o teste que pega isso é a §5.3.

E repare na distância entre a segunda e a terceira linha: **0,023**. É toda a margem que separa uma pergunta que o corpus responde de uma que ele não responde. Qualquer limiar chutado cai fora dessa faixa.

---

### 7. Os bancos que existem

Não há comparação a fazer aqui — a escolha depende de escala, de operação e de o que já existe na casa. O que vale é reconhecer os nomes:

| | Banco | O que é |
|---|---|---|
| <img src="https://github.com/chroma-core.png" width="28"> | **Chroma** | embutido, roda num diretório local. É o desta aula, escolhido por não exigir serviço nenhum |
| <img src="https://github.com/pinecone-io.png" width="28"> | **Pinecone** | serviço gerenciado, sem versão para rodar em casa. Escala para bilhões de vetores |
| <img src="https://github.com/weaviate.png" width="28"> | **Weaviate** | código aberto, com módulos que integram modelos de embedding de vários provedores |
| <img src="https://github.com/qdrant.png" width="28"> | **Qdrant** | código aberto, escrito em Rust, com filtro avançado sobre o payload |
| <img src="https://github.com/milvus-io.png" width="28"> | **Milvus** | código aberto, arquitetura distribuída, pensado para bilhões de vetores |
| <img src="https://github.com/pgvector.png" width="28"> | **pgvector** | extensão do PostgreSQL. **Não é um banco novo** — é vetor dentro do banco que a empresa já tem |
| <img src="https://github.com/opensearch-project.png" width="28"> | **OpenSearch** | motor de busca com suporte a vetor, para quem já usa busca textual e quer somar as duas |

Duas observações que economizam discussão:

**A última linha é a mais subestimada.** Se a aplicação já roda sobre PostgreSQL, o `pgvector` evita operar um sistema a mais — e a maioria dos projetos não tem vetores suficientes para justificar o sistema a mais.

**A API muda, o modelo não.** Os sete guardam id, documento, metadado e vetor; todos indexam com alguma variante de ANN; todos devolvem os `k` mais próximos com uma medida de proximidade. Aprender um é aprender a categoria.

---

## Exemplos

### Exemplo 1 — Quando não usar banco nenhum

Um assistente interno responde sobre um manual de 40 páginas. Cortado por estrutura, dá 120 chunks.

`120 × 1024` floats cabem em meio megabyte de RAM, e a busca exaustiva leva menos de um milissegundo. **Não há banco a introduzir** — há um arquivo `.npy` a salvar em disco, que resolve a persistência sem operar um sistema novo.

O erro aqui não é técnico, é de escala: adotar a ferramenta de um milhão de vetores para cento e vinte.

### Exemplo 2 — Quando o filtro vale mais que o vetor

> *"quem aprovou a despesa D-4471?"*

O vetor é a ferramenta errada: `D-4471` é identificador, e a nota 02 já mostrou que identificadores de mesmo formato ocupam regiões próximas.

Com metadado, a consulta deixa de ser uma busca:

```python
colecao.get(where={"despesa": "D-4471"})
```

Repare que nem é `query` — é `get`. Nenhum vetor foi comparado, e a resposta é exata.

### Exemplo 3 — O erro que o teste de sanidade pega

Uma equipe reindexou o corpus com um modelo de embedding novo, mas esqueceu de recriar a coleção. Os vetores antigos ficaram, os novos entraram, e as consultas passaram a devolver resultados estranhos sem erro nenhum.

O sintoma que denuncia: **um texto do corpus, buscado por si mesmo, deixa de vir em distância zero.** É uma linha de teste, roda em milissegundos, e é a primeira coisa a checar quando um índice "está esquisito".

---

## Fontes e leituras

IBM. **What is a vector database?** IBM Think, 2024. Disponível em: https://www.ibm.com/think/topics/vector-database. (As três funções, os algoritmos de indexação e as métricas de distância.)

DATACAMP. **The top 5 vector databases**. 2023. Disponível em: https://www.datacamp.com/blog/the-top-5-vector-databases. (A lista de bancos e a figura de tipos de embeddings.)

MALKOV, Y.; YASHUNIN, D. **Efficient and robust approximate nearest neighbor search using hierarchical navigable small world graphs**. arXiv:1603.09320, 2016. (O artigo do HNSW.)

CHROMA. **Documentação**: coleções, métricas de distância e filtros de metadado.

**Material da disciplina.** Nota [01](01-o-vetor-e-a-similaridade.md) §2.2 e §5 — por que cosseno, e o piso que explica a §5.4. Nota [02](02-chunking-e-a-medida-da-busca.md) §4 — a assimetria construir × consultar, que é o argumento de persistência. Aula 05, [nota 01-1](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-1-sequencial.md) — a regra determinística que o filtro de metadado realiza.
