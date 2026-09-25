# IA Aplicada com LLMs — Aula 07: RAG e Documentos — As formas de recuperar

## Introdução

A Aula 06 montou um pipeline de recuperação sobre um índice vetorial. Uma decisão ficou implícita nesse arranjo, e ela é de projeto: **qual busca**.

A sigla RAG abrevia *retrieval-augmented generation*, e nada no nome nem na arquitetura determina que a recuperação seja vetorial. A Aula 06 adotou vetores; há outras famílias em uso, mais baratas e mais exatas, e a escolha entre elas é anterior a qualquer ajuste de `k` ou de *chunking*.

O aluno chega aqui supondo que *recuperar* significa *embutir e comparar*. Essa confusão custa caro na montagem de uma base de conhecimento: indexa-se por vetor o que deveria ser consulta, e depois se ajusta *chunking* para consertar um problema que nunca foi de *chunking*.

Esta nota vem depois da [nota 00](00-do-pdf-ao-texto.md) — não há o que recuperar antes de haver texto — e é anterior ao pipeline da [nota 04](04-o-pipeline-e-os-quatro-modos-de-falha.md), do qual não depende.

> **Pré-requisitos:** Aula 06, [nota 01](../../aula-06-embeddings-e-rag/notas-de-aula/01-o-vetor-e-a-similaridade.md) — o vetor e o cosseno — e [nota 02](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md) — *chunking* e `recall@k` · Aula 02, [nota 02 §7](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md) — saída estruturada.
>
> **Código:** esta nota não tem código, de propósito. A comparação entre as formas de recuperar é de projeto, e implementar as cinco exigiria cinco infraestruturas que a aula não tem tempo de montar.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Classificar** uma pergunta pelo formato da resposta — registro, termo, assunto, relação ou agregação — e **selecionar** a forma de recuperação correspondente.
- **Justificar** por que consulta estruturada e busca textual costumam vencer a busca vetorial quando cabem, e quando a combinação das duas se paga.
- **Reconhecer** a pergunta de caminho, que nenhuma das outras formas resolve, e **descrever** como um banco de grafo a responde.
- **Aplicar** a escada de custo: começar no degrau mais barato que resolve.

---

## Desenvolvimento teórico

### 1. A pergunta que classifica

O `R` de RAG é *retrieval*. O pipeline da Aula 06 — `pergunta → busca → portão → contexto + LLM → resposta` — é indiferente a como a busca ocorre: o que ele exige é que algo devolva trechos relevantes, e nada além disso.

Antes de escolher a ferramenta, classifica-se a pergunta pelo **formato da resposta**:

| A resposta é um… | Exemplo | Recuperação adequada |
|---|---|---|
| **registro** | *"qual o saldo do cliente X?"* · *"quem aprovou a D-4471?"* | consulta estruturada |
| **termo** | *"onde aparece a cláusula de força maior?"* | busca textual |
| **assunto** | *"posso reembolsar almoço em viagem?"* | busca vetorial |
| **relação** | *"quais artigos alteram o limite de refeição?"* | grafo |
| **agregação** | *"quais os temas recorrentes nas reclamações?"* | grafo, ou pré-cálculo |

**Só a terceira linha pede *embedding*.** É a frase que esta nota inteira sustenta, e ela contraria a expectativa com que o assunto costuma ser apresentado.

### 2. Consulta estruturada

A consulta é uma condição sobre campos, e o índice devolve as linhas que a satisfazem: `SELECT aprovador FROM despesas WHERE id = 'D-4471'`, resolvido por B-tree em microssegundos, sem nenhuma chamada a modelo.

O trabalho está em obter os parâmetros a partir da pergunta em linguagem natural, e há duas formas:

| | **Expressão regular** | **Modelo com saída estruturada** |
|---|---|---|
| Custo | zero chamadas | uma chamada |
| Resolve | campo de formato fixo: identificador, CNPJ, data, código | o resto |
| Quebra | quando a redação varia | raramente, e falha de outro modo |

A regra prática é usar a primeira quando o campo tem formato fixo, e a segunda quando a mesma pergunta admite muitas redações: *"quem autorizou a D-4471"*, *"a 4471 passou por quem?"* e *"aprovação da despesa 4471"* devem produzir o mesmo parâmetro, e só a segunda generaliza.

Nada disso é novo em relação à Aula 03: **isto já é *tool calling***. A descrição da ferramenta é o *prompt*, o *schema* é o contrato (Aula 02, [nota 02 §7](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md)), e o que muda é apenas que a ferramenta serve a uma pergunta de conhecimento em vez de a uma ação.

**O aviso sobre *text-to-SQL*.** Pedir ao modelo que escreva a consulta inteira é popular e é a versão arriscada: ele pode consultar tabela que não devia, gerar varredura completa ou compor a consulta com texto do usuário. A forma defensável é o modelo **preencher os parâmetros de uma consulta escrita pela aplicação** — é o princípio de *allowlist* da Aula 05, aplicado a banco de dados.

> **O teste de um minuto:** se um `SELECT` responde a pergunta, não há RAG a construir.

### 3. Busca textual

A consulta é um conjunto de termos, e o índice — invertido — devolve os documentos em que eles ocorrem. É a estrutura espelhada da do índice vetorial:

```
  índice vetorial     chunk ──> vetor         "o que se parece com isto?"
  índice invertido    termo ──> [documentos]  "onde aparece esta palavra?"
```

**Como ordena.** A função padrão é o BM25: pontua alto o termo que aparece muito **neste documento** e pouco **no corpus inteiro**, corrigido pelo tamanho do documento. Raridade vale mais que frequência, e é por isso que ela é boa justamente no termo incomum.

**O que ela dá e o vetor não dá.** Operador booleano, busca por prefixo, proximidade entre termos, restrição por campo e citação exata.

| Ganha do vetor em | Por quê |
|---|---|
| identificador e código | `D-4471` é literal, e não é assunto |
| nome próprio raro | aparece pouco no treino, e o vetor não o distingue de um parecido |
| citação exata | quem pergunta quer **aquela** frase, não uma parecida |

**O que ela não dá.** Sinônimo e paráfrase. *"Reembolso de almoço"* não encontra *"despesa com alimentação"* — que é exatamente o que o vetor resolve, e a razão de os dois coexistirem.

**Onde ela já está.** Elasticsearch, OpenSearch e Solr são serviços dedicados, mas uma aplicação que roda sobre PostgreSQL **já tem busca textual instalada**: `tsvector`, `tsquery` e um índice GIN. O SQLite tem o FTS5. Convém saber disso antes de subir um serviço novo para operar.

### 4. Busca vetorial

É o mecanismo da Aula 06, e a única das formas que tolera pergunta mal formulada, sinônimo e paráfrase: a pergunta *"posso reembolsar o jantar?"* recupera o artigo sobre alimentação sem que a palavra "jantar" ocorra nele.

É também a única probabilística. A mesma consulta sobre o mesmo índice devolve a mesma lista, mas não há regra legível que explique por que o quinto resultado ficou em quinto — e é essa opacidade que torna a tabela da §1 necessária.

A linha "o que ela erra" reaparece inteira na [nota 02](02-o-que-o-embedding-nao-ve.md) desta aula, sob o título de cegueiras do *embedding*: identificador, negação, magnitude e tempo. Lida ao lado das outras formas, ela muda de natureza:

> Três das quatro cegueiras — identificador, magnitude e tempo — são casos que um `WHERE` resolve de forma exata e barata, e a quarta é caso de busca textual. Elas não são defeito do RAG: são sintoma de ter escolhido **uma** das formas de recuperar para um corpus que precisava de duas.

Recuperar por palavra-chave não é o passo atrás que parece. É a ferramenta certa para uma classe de pergunta que o vetor não atende.

### 5. Recuperação por grafo

#### 5.1 A pergunta que as outras formas não alcançam

As três formas anteriores recuperam **objetos**: uma linha, um documento, uma passagem. Existe uma classe de pergunta cuja resposta não é nenhum objeto — é o **caminho entre eles**.

> *"Uma refeição de R$ 340,00 por pessoa em Lisboa está dentro da política?"*

É a despesa da [nota 03](03-medir-o-rag-e-a-recuperacao-como-ferramenta.md), e ela exige três artigos que **estão ligados entre si**: o Art. 4º §1º fixa o teto da categoria, o Art. 4º §2º **excetua** esse teto em viagem internacional, e o Art. 9º §2º define **quem aprova** a faixa de valor resultante. Nenhum dos três, sozinho, responde. A relação entre eles é que responde — e a relação não está em nenhum dos textos: está *entre* eles.

O vetor não a representa: ele mede parecença de assunto, e "excetua" não é um assunto. O `SELECT` só a alcança com junções encadeadas, e quantas junções depende da pergunta, o que uma consulta fixa não sabe de antemão. É aqui que o grafo é o degrau certo.

#### 5.2 O modelo: nó, aresta e propriedade

Um banco de grafo — o **Neo4j** é o de uso mais corrente — guarda três coisas, e só três:

| | O que é | Exemplo |
|---|---|---|
| **nó** | a entidade | um artigo, uma categoria, uma faixa de alçada |
| **aresta** | a relação, **dirigida e tipada** | `EXCETUA`, `LIMITA`, `APROVA` |
| **propriedade** | o dado no nó ou na aresta | `valor: 120.00`, `condicao: "internacional"` |

A diferença para o modelo relacional é onde a relação mora. Numa tabela, o vínculo é uma chave estrangeira que só existe quando alguém escreve a junção na consulta. No grafo, **a aresta é um objeto de primeira classe, armazenada junto do nó** — percorrer de um vizinho a outro é um salto de ponteiro, e o custo não cresce com o tamanho do banco, só com o tamanho da vizinhança visitada.

#### 5.3 O grafo do regulamento

Este é o recorte do regulamento da disciplina, desenhado:

```
                         ┌───────────────────────┐
                         │  Categoria            │
                         │  nome: "alimentação"  │
                         └───────────┬───────────┘
                                     ▲
                                     │ :LIMITA
                                     │ (valor: 120.00)
                         ┌───────────┴───────────┐
                         │  Artigo               │
                         │  id: "Art. 4º §1º"    │
                         └───────────▲───────────┘
                                     │
                                     │ :EXCETUA
                                     │
                         ┌───────────┴───────────┐        ┌──────────────────┐
                         │  Artigo               │        │  Condicao        │
                         │  id: "Art. 4º §2º"    ├───────►│  nome:           │
                         │  valor: 260.00        │:VALE_EM│  "internacional" │
                         └───────────────────────┘        └──────────────────┘

                         ┌───────────────────────┐        ┌──────────────────┐
                         │  Artigo               │        │  Faixa           │
                         │  id: "Art. 9º §2º"    ├───────►│  min:  500.00    │
                         └───────────────────────┘ :APROVA│  max: 5000.00    │
                                                          │  quem: "gestor"  │
                                                          └──────────────────┘
```

Três tipos de nó, quatro tipos de aresta. O que o desenho torna visível e o texto corrido escondia: **o Art. 4º §2º não é um artigo parecido com o §1º — é um artigo que aponta para ele**, com um tipo de relação que diz o que faz com ele.

#### 5.4 Como se busca: a linguagem Cypher

A linguagem de consulta do Neo4j é o **Cypher**, e a ideia que a torna fácil de ler é que **o padrão da consulta é o desenho do grafo em ASCII**. `(a)-[:REL]->(b)` é um nó, uma seta e outro nó.

Criar os nós e as arestas do desenho acima:

```cypher
CREATE (a41:Artigo {id: 'Art. 4º §1º', valor: 120.00})
CREATE (a42:Artigo {id: 'Art. 4º §2º', valor: 260.00})
CREATE (a92:Artigo {id: 'Art. 9º §2º'})
CREATE (alim:Categoria {nome: 'alimentação'})
CREATE (intl:Condicao {nome: 'internacional'})
CREATE (fx:Faixa {min: 500.00, max: 5000.00, quem: 'gestor da área'})

CREATE (a41)-[:LIMITA {valor: 120.00}]->(alim)
CREATE (a42)-[:EXCETUA]->(a41)
CREATE (a42)-[:VALE_EM]->(intl)
CREATE (a92)-[:APROVA]->(fx)
```

**A consulta de Lisboa**, que é a pergunta que as outras formas não alcançavam:

```cypher
MATCH (base:Artigo)-[:LIMITA]->(:Categoria {nome: 'alimentação'})
OPTIONAL MATCH (excecao:Artigo)-[:EXCETUA]->(base)
WHERE (excecao)-[:VALE_EM]->(:Condicao {nome: 'internacional'})
MATCH (alcada:Artigo)-[:APROVA]->(f:Faixa)
WHERE f.min <= 340.00 <= f.max
RETURN base.id, excecao.id, excecao.valor, alcada.id, f.quem
```

Devolve, numa consulta só, exatamente os três artigos que o pipeline vetorial da nota 03 não conseguiu reunir:

```
base.id       excecao.id     excecao.valor  alcada.id      f.quem
"Art. 4º §1º" "Art. 4º §2º"  260.00         "Art. 9º §2º"  "gestor da área"
```

**O caminho de comprimento variável** é o que nenhuma das outras formas tem. Se um artigo remete a outro, que remete a um terceiro, o asterisco percorre a cadeia inteira sem que a consulta saiba de antemão quantos saltos existem:

```cypher
MATCH caminho = (a:Artigo {id: 'Art. 4º §1º'})<-[:EXCETUA|REMETE_A*1..5]-(afetado:Artigo)
RETURN [n IN nodes(caminho) | n.id] AS cadeia
```

> *"Quais artigos, direta ou indiretamente, alteram o limite de refeição?"* — em SQL isso é uma CTE recursiva; no vetor, não é nada.

**E a agregação**, que é a última linha da tabela da §1:

```cypher
MATCH (a:Artigo)-[:EXCETUA]->(base:Artigo)-[:LIMITA]->(c:Categoria)
RETURN c.nome, count(a) AS excecoes
ORDER BY excecoes DESC
```

A resposta — *"alimentação tem 1 exceção, hospedagem tem 1, transporte tem 0"* — não está em nenhum trecho do documento. Ela só existe depois de contar, e é por isso que a linha "agregação" da tabela da §1 não aponta para busca nenhuma.

#### 5.5 O preço, e quando não vale

O grafo é o degrau mais caro da escada, e o custo **não está na consulta — está na construção**:

| Custo | O que é |
|---|---|
| **extrair entidades e relações** | alguém tem de decidir que `EXCETUA` existe e onde ela ocorre. Manual é caro; automático com LLM é a proposta do *GraphRAG*, e erra |
| **manter** | o regulamento muda, e o grafo precisa mudar junto — um índice vetorial se reconstrói sozinho, um esquema de grafo não |
| **operar** | é mais um serviço, com mais um modelo de dados e mais uma linguagem na equipe |

Por isso a regra é a mesma das outras formas, e é a da §7: **só se sobe ao grafo quando a pergunta é de caminho ou de agregação**, e quando essas perguntas são frequentes o bastante para pagar a construção. Num corpus pequeno, com perguntas de assunto, o grafo é infraestrutura parada.

> **O meio-termo que costuma bastar.** Para uma cadeia rasa e conhecida — "este artigo tem exceção?" — um campo `excetua_id` numa tabela e uma junção resolvem, sem banco novo. O grafo se justifica quando a profundidade é **desconhecida de antemão**, que é exatamente o que o `*1..5` expressa e o `JOIN` não.

### 6. A busca híbrida, e por que somar escores não funciona

Rodar a vetorial e a textual sobre a mesma consulta e juntar os resultados é o padrão de mercado. O detalhe que costuma faltar: **as duas escalas não são comparáveis**. O cosseno vive entre −1 e 1; o BM25 não tem teto e depende do corpus. Somar os dois é comparar régua com balança.

A saída usual é fundir por **posição**, e não por escore. Na *reciprocal rank fusion*, cada resultado pontua `1/(k + posição)` em cada lista, e as pontuações se somam. Não exige calibragem e é robusta a escalas diferentes.

O filtro por campo entra **antes** da busca, e não depois: restringir o índice às linhas vigentes e só então buscar por similaridade é mais barato e mais correto do que recuperar por similaridade e descartar o que estiver fora de prazo.

O preço da combinação é operacional — dois índices a sincronizar, duas latências a somar e um parâmetro de fusão a calibrar. Num corpus pequeno e homogêneo, a busca vetorial sozinha basta; a combinação se justifica quando o corpus mistura prosa e identificadores, que é o caso de quase todo acervo corporativo.

### 7. A escada

```
                         CUSTO            QUANDO É O DEGRAU CERTO
                         ─────            ───────────────────────
  regex                  zero             o campo tem formato fixo
  consulta estruturada   zero ou uma      a resposta é um REGISTRO
  busca textual          zero             termo exato, citação, id
  busca vetorial         uma (embedding)  a resposta é uma PASSAGEM
  grafo                  construção cara  a resposta é o CAMINHO, ou a AGREGAÇÃO
```

É a cascata do *router* da Aula 05 num domínio novo: começa-se no degrau mais barato que resolve, e só se sobe quando o de baixo tem limite conhecido.

---

## Exemplos

### Exemplo 1 — A mesma pergunta, para cada forma

| Pergunta | Recuperação adequada | Por quê |
|---|---|---|
| *"quem aprovou a despesa D-4471?"* | consulta estruturada | a resposta é um registro, e `D-4471` é um campo |
| *"o que diz o Art. 7º §2º?"* | busca textual | o termo **é** a consulta; o vetor aproxima todos os artigos entre si |
| *"posso reembolsar o jantar de ontem?"* | busca vetorial | o assunto é alimentação, e a palavra "jantar" não está no texto |
| *"o notebook entra em material de escritório?"* | vetorial **e** textual | o assunto pede semântica; o desfecho depende do termo "informática", que ocorre uma única vez |
| *"que artigos alteram o teto de refeição?"* | grafo | a resposta é a cadeia de exceções, e não se sabe quantos saltos ela tem |

A quarta linha é o modo de falha 1 da [nota 04](04-o-pipeline-e-os-quatro-modos-de-falha.md), lido pelo lado da escolha de recuperação. A quinta é a despesa de Lisboa da [nota 03](03-medir-o-rag-e-a-recuperacao-como-ferramenta.md), lida pelo mesmo lado: lá ela se resolve com um laço de buscas, aqui com uma consulta só — e a comparação entre as duas saídas é a decisão de arquitetura que a aula toda prepara.

---

## Fontes e leituras

ROBERTSON, S.; ZARAGOZA, H. **The probabilistic relevance framework**: BM25 and beyond. Foundations and Trends in Information Retrieval, v. 3, n. 4, 2009. (A função de relevância da §3.)

CORMACK, G. V.; CLARKE, C. L. A.; BUETTCHER, S. **Reciprocal rank fusion outperforms Condorcet and individual rank learning methods**. SIGIR, 2009. (A fusão da §6.)

ROBINSON, I.; WEBBER, J.; EIFREM, E. **Graph databases**: new opportunities for connected data. 2. ed. O'Reilly, 2015. (O modelo de nó, aresta e propriedade da §5.2, e a adjacência sem índice.)

FRANCIS, N. et al. **Cypher: an evolving query language for property graphs**. SIGMOD, 2018. (A linguagem da §5.4.)

EDGE, D. et al. **From local to global**: a graph RAG approach to query-focused summarization. arXiv:2404.16130, 2024. (A construção automática de grafo por LLM, e os limites dela, §5.5.)

LEWIS, P. et al. **Retrieval-augmented generation for knowledge-intensive NLP tasks**. arXiv:2005.11401, 2020. (A formulação original, em que a recuperação é um componente substituível.)

**Material da disciplina.** Aula 02, [nota 02 §7](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md) — a saída estruturada que extrai parâmetro. Aula 05, [nota 01-2](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-2-router.md) — o *router*, cuja cascata a §7 reaproveita. Aula 06, [nota 01](../../aula-06-embeddings-e-rag/notas-de-aula/01-o-vetor-e-a-similaridade.md) — o que o cosseno mede; [nota 02](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md) — o `recall@k`. Desta aula, [nota 00](00-do-pdf-ao-texto.md) — a extração, que é anterior a qualquer forma de recuperar; [nota 02](02-o-que-o-embedding-nao-ve.md) — as cegueiras, aqui lidas como escolha de recuperação; [nota 03](03-medir-o-rag-e-a-recuperacao-como-ferramenta.md) — a despesa de Lisboa, que a §5.1 relê como pergunta de caminho.
