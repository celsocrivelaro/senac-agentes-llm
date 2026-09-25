# IA Aplicada com LLMs — Aula 07: RAG e Documentos — O que o embedding não vê

## Introdução

Esta nota nomeia uma coisa que a Aula 06 encontrou três vezes sem saber chamar pelo nome.

A [nota 01](../../aula-06-embeddings-e-rag/notas-de-aula/01-o-vetor-e-a-similaridade.md) de lá encerrou com uma distinção que não é defeito de modelo: similaridade não é relevância, porque são grandezas diferentes. A [nota 02](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md) mediu o `recall@k` e encontrou uma pergunta que falhava com qualquer estratégia de corte, porque três parágrafos se distinguiam apenas por faixa de valor. A [nota 03](../../aula-06-embeddings-e-rag/notas-de-aula/03-o-router-por-embedding.md) viu duas mensagens opostas caírem na mesma rota, sem nenhum modelo escrever uma palavra.

Os três casos são o mesmo caso, e esta nota trata dele: do que um vetor de embedding comprovadamente **não representa**.

O tema não é acessório. Sem ele, a conclusão razoável ao fim da Aula 06 seria que busca semântica resolve o problema de encontrar informação, e que o trabalho restante é de ajuste. As quatro demonstrações a seguir mostram o contrário: há classes inteiras de comparação para as quais o vetor é a ferramenta errada, e todas as quatro aparecem no domínio de prestação de contas usado no laboratório.

O método adotado é uniforme. Para cada par de textos, registra-se primeiro a **similaridade que a intuição prevê**, depois a **similaridade obtida**. A distância entre as duas é o conteúdo da nota.

> **Pré-requisitos:** Aula 06, notas [01](../../aula-06-embeddings-e-rag/notas-de-aula/01-o-vetor-e-a-similaridade.md), [02](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md) e [03](../../aula-06-embeddings-e-rag/notas-de-aula/03-o-router-por-embedding.md).
>
> **Código:** [`01-cegueiras.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula07-rag-e-documentos/01-cegueiras.py). Em sala, leia cada par em voz alta e registre a previsão da turma antes de mostrar a saída.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Demonstrar** que negação, magnitude numérica, identidade de entidade e anterioridade temporal não são representadas por similaridade de vetores.
- **Calibrar** a leitura de um valor de cosseno por comparação com um par de controle.
- **Decidir**, diante de um requisito de comparação, se ele pertence ao vetor ou a uma regra determinística.
- **Antecipar** o efeito de cada cegueira sobre um sistema de recuperação em produção.

---

## Desenvolvimento teórico

### 1. O par de controle

Antes de qualquer medição, é necessário estabelecer a referência. Um valor de cosseno isolado não é interpretável: não há como classificar 0,84 como alto ou baixo sem uma escala.

O par de controle estabelece essa escala. Ele reúne dois textos **sem relação alguma** entre si:

| | Texto |
|---|---|
| A | "a despesa foi aprovada pelo analista" |
| B | "a previsão do tempo indica chuva no litoral norte" |

O valor obtido para esse par — **0,6557** — é o piso prático da escala. Textos em português compartilham estrutura, e o modelo captura essa estrutura, de modo que o piso fica bem acima de zero, como a nota anterior estabeleceu.

Todas as medições seguintes devem ser lidas **em relação a esse piso**, e não em relação a zero. É a diferença entre concluir "o modelo achou os textos parecidos" e concluir "o modelo não distinguiu os textos mais do que distinguiria de uma frase sobre o tempo".

---

### 2. Negação

| | Texto |
|---|---|
| A | "a despesa foi aprovada pelo analista" |
| B | "a despesa foi **reprovada** pelo analista" |

**Previsão da intuição:** baixa. Os textos afirmam o oposto um do outro, e a decisão que cada um representa é contrária.

**Resultado obtido: 0,9400.** É a **mais baixa** das quatro cegueiras, e ainda assim fica 28 centésimos acima do controle. A polaridade é a única das quatro que o vetor registra minimamente — e o mínimo não serve para decidir nada.

**A explicação.** Um modelo de embedding representa o contexto de ocorrência de um termo. "Aprovado" e "reprovado" ocorrem exatamente nos mesmos contextos — mesmos sujeitos, mesmos complementos, mesmos documentos. É essa co-ocorrência que o vetor captura, e a polaridade da decisão não altera o contexto.

**Consequência em produção.** Um sistema de busca sobre pareceres que receba a consulta *"despesas reprovadas por falta de nota fiscal"* recupera com igual facilidade os pareceres de aprovação. Nenhum ajuste de *chunking* corrige isso, porque a informação não está sendo perdida no corte — ela nunca esteve no vetor.

**O que fazer.** A polaridade é um campo, não um trecho de texto. Um parecer tem `veredito ∈ {aprovado, reprovado}`, e filtrar por esse campo é uma cláusula de consulta, não uma comparação de vetores. O laboratório aplica exatamente isso: a memória episódica da Aula 08 guarda `veredito` como **metadado**, e a recuperação filtra por ele antes de ordenar por similaridade.

```python
# ERRADO — a polaridade viaja dentro do texto da consulta
indice.buscar("pareceres reprovados por falta de nota fiscal", k=5)
# devolve aprovações e reprovações embaralhadas: entre elas o cosseno é 0,9400

# CERTO — o campo filtra, e o vetor decide só o assunto
indice.colecao.query(
    query_embeddings=[gerar_vetor_embeddings("falta de nota fiscal").tolist()],
    where={"veredito": "reprovado"},   # filtra ANTES de ordenar por similaridade
    n_results=5,
)
```

O `where` do Chroma é o mesmo `WHERE` do banco relacional, e a ordem importa: filtrar **antes** reduz o espaço de busca; filtrar depois descarta bons resultados que já perderam lugar para os errados.

---

### 3. Número

| | Texto |
|---|---|
| A | "o teto de reembolso é de R$ 120,00 por refeição" |
| B | "o teto de reembolso é de R$ 260,00 por refeição" |

**Previsão da intuição:** baixa. Os dois valores decidem casos diferentes: uma despesa de R$ 200,00 é reprovada sob A e aprovada sob B.

**Resultado obtido: 0,9836.** É a **mais alta** das quatro, e o par difere em três caracteres. Trocar R$ 120,00 por R$ 260,00 move o vetor menos que trocar uma vírgula de lugar moveria o sentido para um leitor.

**A explicação.** Números são tokenizados como qualquer outra sequência, e a tokenização não preserva magnitude. A Aula 01 (nota 02) já havia observado que `120` e `260` podem ocupar quantidades diferentes de tokens sem qualquer relação com a diferença aritmética entre eles. Um modelo treinado para representar contexto não tem incentivo para codificar ordem numérica.

**Consequência em produção.** Este é o caso mais perigoso dos quatro, porque produz erro **silencioso e plausível**. O sistema recupera o artigo com o teto errado, e o texto recuperado é do mesmo assunto, com a mesma redação, citando o mesmo artigo. A revisão humana não desconfia.

O modo de falha 4 da [nota 04](04-o-pipeline-e-os-quatro-modos-de-falha.md) explora esse caso diretamente: o corpus do laboratório contém a versão vigente do Art. 4º §1º (R$ 120,00) e uma versão revogada de 2024 (R$ 90,00), e o pipeline recupera as duas sem notar que há duas.

**O que fazer.** Comparação de valor é `<=`. O teto aplicável a uma despesa não se descobre por busca — descobre-se recuperando o **artigo** e lendo o campo numérico dele. A distinção é entre usar o vetor para localizar a regra e usá-lo para aplicar a regra; apenas a primeira função é apropriada.

```python
# ERRADO — pedir ao vetor que compare grandezas
indice.buscar("uma refeição de R$ 340,00 está dentro do teto?", k=3)
# recupera o artigo de R$ 120,00 e o de R$ 260,00 sem distinguir qual se aplica

# CERTO — o vetor LOCALIZA a regra; o código APLICA a regra
artigo = indice.buscar("teto de refeição em viagem internacional", k=1)[0]
dentro_do_teto = despesa.valor <= artigo["valor"]   # 340.00 <= 260.00 -> False
```

Repare onde o vetor entra e onde sai: ele acha o **Art. 4º §2º** entre quarenta artigos, que é um problema difícil e que ele resolve bem. A comparação `340 <= 260` é uma linha de Python, exata e testável — e é a única forma de acertar sempre.

---

### 4. Entidade

| | Texto |
|---|---|
| A | "análise da despesa D-4471 do funcionário F-088" |
| B | "análise da despesa D-4472 do funcionário F-091" |

**Previsão da intuição:** baixa. São registros distintos, de funcionários distintos.

**Resultado obtido: 0,9738.** Segunda mais alta, e pelo mesmo motivo do caso anterior: o identificador é ruído para o modelo, e o que sobra — "análise da despesa do funcionário" — é idêntico nos dois textos.

**A explicação.** Um identificador não carrega significado distribuído: `D-4471` e `D-4472` são formalmente idênticos exceto por um dígito, e semanticamente ambos são "um identificador de despesa". O vetor representa a categoria, não a instância.

**Consequência em produção.** Buscar `D-4471` num corpus de mil despesas recupera mil trechos igualmente parecidos, e a ordenação entre eles é arbitrária. Um sistema que dependa disso funciona no teste com dez registros e falha em produção com dez mil — o modo de falha mais caro que existe, porque só aparece depois do *deploy*.

**O que fazer.** Identificador se resolve por `==` ou por índice invertido. O padrão adequado é o roteador da Aula 05 (nota 02, §2): se a consulta contém um identificador bem formado, a rota é uma consulta direta, sem chamada ao modelo e sem busca vetorial.

```python
import re

IDENTIFICADOR = re.compile(r"\b[DF]-\d{4}\b")   # D-4471, F-088

def recuperar(pergunta: str) -> list[dict]:
    if achado := IDENTIFICADOR.search(pergunta):
        return banco.por_id(achado.group())   # rota exata: zero chamadas
    return indice.buscar(pergunta, k=3)       # só aqui o vetor é usado
```

São seis linhas, e elas eliminam a cegueira inteira para a classe de pergunta em que ela aparece. É o degrau `regex` da escada da [nota 01](01-as-formas-de-recuperar.md), §7 — o mais barato de todos, e o primeiro a tentar.

---

### 5. Tempo

| | Texto |
|---|---|
| A | "em março de 2026, o teto de refeição passou a ser R$ 120,00" |
| B | "em agosto de 2026, o teto de refeição passou a ser R$ 150,00" |

**Previsão da intuição:** baixa. Um dos dois está revogado pelo outro.

**Resultado obtido: 0,9648.** Ambos os textos contêm uma data, e nenhum contém informação sobre qual delas é posterior. Para o vetor, "março de 2026" e "agosto de 2026" são duas datas; que uma venha antes da outra é aritmética, não semântica.

**A explicação.** O vetor não representa anterioridade. "Março" e "agosto" são dois meses; que um preceda o outro é conhecimento aritmético sobre o calendário, não propriedade do contexto textual.

**Consequência em produção.** Duas afirmações verdadeiras em momentos diferentes disputam a mesma consulta, e a que vence é a **mais parecida com a pergunta**, não a vigente. Se a pergunta for formulada no presente — *"qual **é** o teto?"* —, ela tende a favorecer o texto redigido no presente, que pode ser o revogado.

**O que fazer.** Carimbo de tempo como **metadado**, e desempate no código:

```python
# ERRADO — delegar a vigência ao modelo, junto com os dois textos
contexto = montar_contexto(indice.buscar("teto de refeição", k=2))
gerar("Considere apenas a versão vigente.\n" + contexto)
# ele erra, gasta uma chamada, e o acerto não é testável

# CERTO — a data é metadado, e o desempate é uma linha
def mais_recente(candidatos: list[dict], campo: str = "data") -> dict | None:
    return max(candidatos, key=lambda c: c[campo]) if candidatos else None

vigente = mais_recente(indice.buscar("teto de refeição", k=2))
```

Determinístico, sem chamada nenhuma, e com um teste de duas linhas. Delegar o desempate ao modelo é o antipadrão — e note que ele **não é uma versão mais fraca do conserto certo**: é um conserto que falha justamente quando as duas versões são parecidas, que é sempre.

> **Esta cegueira é a que tem consequência mais distante.** A Aula 08 a retoma como experimento central: dois fatos verdadeiros em datas diferentes, indexados na memória de um agente, e a pergunta de qual deles ele cita. A função acima é o conserto, e ela está em `memoria.py`.

---

### 6. O padrão que as quatro compartilham

As quatro cegueiras não são quatro defeitos independentes. São a mesma propriedade vista de quatro ângulos:

> **O embedding representa contexto de ocorrência. Não representa polaridade, magnitude, identidade nem ordem.**

E as quatro têm o mesmo tratamento, que é a lição recorrente do curso num lugar novo:

| A comparação é sobre | O instrumento correto |
|---|---|
| polaridade de uma decisão | campo com domínio fechado, filtrado na consulta |
| magnitude de um valor | operador de comparação numérica |
| identidade de um registro | igualdade, ou índice invertido |
| anterioridade | carimbo de tempo e `max()` |
| **assunto** | **similaridade de vetores** |

A última linha é o que sobra para o vetor — e é bastante. Localizar, entre quarenta artigos, os três que tratam do assunto da pergunta é um problema real, difícil por outros meios, e que a busca semântica resolve bem. O erro não é usar embeddings; é usá-los para as outras quatro linhas.

> **Regra determinística onde ela existe.** É a mesma conclusão do roteador da Aula 05 e da tática zero da Aula 05 (nota 03, §4), aplicada agora à recuperação: o caminho mais barato costuma ser o que não envolve o modelo.

---

### 7. Onde cada uma reaparece

Nenhuma das quatro é curiosidade de laboratório. Cada uma tem data marcada no curso, e é por isso que vale nomeá-las agora:

| Cegueira | Onde reaparece | Como |
|---|---|---|
| **Negação** | Aula 06, [nota 03](../../aula-06-embeddings-e-rag/notas-de-aula/03-o-router-por-embedding.md) · modo de falha 1 desta aula · Aula 08, nota 02 | o router manda "chegou" e "não chegou" para a mesma rota · o corpus menciona o assunto apenas para **excluí-lo** · a memória recupera episódios de aprovação quando se pergunta por reprovação |
| **Número** | [modo de falha 4](04-o-pipeline-e-os-quatro-modos-de-falha.md), desta aula | duas versões do mesmo artigo, diferindo só na magnitude do teto |
| **Tempo** | [modo de falha 4](04-o-pipeline-e-os-quatro-modos-de-falha.md) · Aula 08, nota 04 | qual das duas versões é a vigente, e o desempate por carimbo |
| **Entidade** | Aula 10, nota 03 §2 | identificadores de ferramentas MCP de mesmo formato se confundem — a nota de lá a chama de *"uma cegueira conhecida, em lugar novo"* |

Vale registrar a que a tabela torna visível: **a negação já cobrou o preço dela antes de existir RAG**. A [nota 03 da Aula 06](../../aula-06-embeddings-e-rag/notas-de-aula/03-o-router-por-embedding.md) roteou mensagens por embedding, e o par que difere em uma palavra caiu na mesma rota — sem que nenhum modelo tivesse escrito nada. As cegueiras não são um problema de geração de texto; são um problema de **representação**, e aparecem em qualquer sistema que decida por proximidade.

---

## Exemplos

### Exemplo 1 — O quadro completo, lido contra o controle

```
  NÚMERO      0.9836  █████████████████████████████████████████████████
  ENTIDADE    0.9738  ████████████████████████████████████████████████
  TEMPO       0.9648  ████████████████████████████████████████████████
  NEGAÇÃO     0.9400  ███████████████████████████████████████████████
  CONTROLE    0.6557  ████████████████████████████████  <- textos SEM RELAÇÃO
```

*(Valores medidos com `mistral-embed`; o `01-cegueiras.py` os reproduz.)*

A leitura decisiva está na comparação com a última linha. Textos que afirmam o oposto ficam em **0,94**; textos sem relação alguma, em **0,66**. São 28 centésimos de diferença — e a mais alta das quatro, o NÚMERO, chega a **0,98**. Para um leitor humano, a distância entre "aprovado" e "reprovado" é total. Para o vetor, ela é **menor que a distância entre qualquer um dos dois e uma frase sobre o tempo**.

Nenhuma das quatro cegueiras se aproxima do piso. Não se trata de o modelo distinguir mal: trata-se de o modelo não estar medindo aquilo.

### Exemplo 2 — Uma consulta que falha por duas cegueiras ao mesmo tempo

> *"A despesa D-4471 do F-088 foi reprovada?"*

A consulta contém três elementos, e o vetor erra em dois deles:

| Elemento | Cegueira | Efeito |
|---|---|---|
| `D-4471` | entidade | recupera qualquer despesa do mesmo formato |
| `F-088` | entidade | idem para funcionário |
| "reprovada" | negação | recupera pareceres de aprovação com igual facilidade |

O que resta de aproveitável na consulta é a palavra "despesa" — que não discrimina nada num corpus sobre despesas.

O tratamento correto não é ajustar a busca: é **não buscar**. A consulta contém dois identificadores bem formados, e a rota adequada é uma consulta direta pelo id, com filtro por `veredito`. É o roteador da Aula 05 decidindo, antes de qualquer chamada, que este caso não é de busca semântica.

### Exemplo 3 — O caso em que o vetor é a ferramenta certa

> *"Posso pedir reembolso do vinho que pedi no jantar com o cliente?"*

Nenhum identificador, nenhum valor, nenhuma negação, nenhuma data. A pergunta é sobre **assunto**, e o corpus contém um artigo que trata exatamente disso — o Art. 4º §4º, que veda bebida alcoólica.

O trecho relevante não compartilha com a pergunta nenhuma palavra de conteúdo: a pergunta diz "vinho", o artigo diz "bebida alcoólica"; a pergunta diz "jantar com o cliente", o artigo não menciona cliente algum. Uma busca por palavra-chave falha; a busca semântica acerta.

É esse o caso que justifica a aula. E a distinção entre este exemplo e o anterior — qual dos dois é trabalho de vetor — é a decisão de projeto que a nota inteira prepara.


### Exemplo 4 — A máscara que torna o empate visível

Se o identificador, o valor e a data são ruído para o vetor, a pergunta natural é: por que deixá-los no texto que vai ser embutido? A resposta é **canonicalizar na ingestão** — trocar cada um por um marcador genérico antes de calcular o embedding.

```python
MES = ("janeiro|fevereiro|março|abril|maio|junho|"
       "julho|agosto|setembro|outubro|novembro|dezembro")

MASCARAS = [
    (re.compile(r"\b[A-Z]-\d{3,4}\b"),                       "<ID>"),
    (re.compile(r"R\$ ?[\d.]+,\d{2}"),                        "<VALOR>"),
    (re.compile(rf"\b(?:{MES}) de (?:19|20)\d{{2}}\b", re.I), "<DATA>"),
]

def canonicalizar(texto: str) -> str:
    for padrao, marca in MASCARAS:
        texto = padrao.sub(marca, texto)
    return texto
```

Aplicada aos pares das §3, §4 e §5, ela produz isto:

```
  ENTIDADE   "análise da despesa D-4471 do funcionário F-088"
             "análise da despesa D-4472 do funcionário F-091"
          -> "análise da despesa <ID> do funcionário <ID>"          IDÊNTICOS

  NÚMERO     "o teto de reembolso é de R$ 120,00 por refeição"
             "o teto de reembolso é de R$ 260,00 por refeição"
          -> "o teto de reembolso é de <VALOR> por refeição"        IDÊNTICOS

  TEMPO      "em março de 2026, o teto de refeição passou a ser R$ 120,00"
             "em agosto de 2026, o teto de refeição passou a ser R$ 150,00"
          -> "em <DATA>, o teto de refeição passou a ser <VALOR>"   IDÊNTICOS
```

**O cosseno vai a 1,0000 nos três, e isso não é uma medição — é aritmética:** textos idênticos produzem o mesmo vetor. E é justamente aí que está o ganho, que não é o que a intuição sugere.

A máscara **não conserta** a cegueira. Ela a torna **visível**. Os 0,9738 da §4 davam a `D-4471` uma vantagem de 26 milésimos sobre a `D-4472` — uma diferença que é ruído puro, e que o sistema usava para ordenar a lista que o aluno lia como resultado. Com a máscara, o empate é explícito e a ordenação por similaridade fica obviamente impossível, o que força o caminho certo: filtrar por metadado. A informação nunca esteve no vetor; a canonicalização apenas para de fingir que estava.

#### Duas coleções de texto, não uma

É o ponto onde uma implementação real quebra. Se o texto mascarado for o texto indexado, o modelo recebe *"o teto é de `<VALOR>` por refeição"* e não tem como responder. **O texto que se embute e o texto que vai ao contexto são diferentes**, e o `IndiceChroma` do laboratório já separa os dois, porque passa os embeddings prontos:

```python
colecao.add(
    documents=[c["texto"] for c in chunks],                     # ORIGINAL -> vai ao LLM
    embeddings=gerar_matrix_embbeddings(
        [canonicalizar(c["texto"]) for c in chunks]).tolist(),  # CANÔNICO -> só compara
    metadatas=[{"artigo": c["id"], "valor": c["valor"], "data": c["data"]}
               for c in chunks],
)
```

Uma linha de diferença. E a consulta tem de passar pela **mesma** função: canonicalizar só o corpus é pior do que não canonicalizar nada, porque aí nenhum dos dois lados combina.

#### Três condições, e uma fragilidade

**A ordem é extrair, depois mascarar.** O valor precisa estar no metadado *antes* de sair do texto. Invertida, a ordem apaga a informação de vez, e nem o `<=` da §3 recupera.

**Só se mascara o que é ruído para as perguntas que se recebe.** Numa base em que alguém pergunta *"o que mudou na política de 2024?"*, o ano **é** o assunto, e `<DATA>` destrói a pergunta. A decisão exige conhecer as consultas reais.

**A máscara entra no carimbo.** Vale a regra que a Aula 06 estabeleceu para o *chunking*: dois índices construídos com listas de máscaras diferentes não são comparáveis, e mudar a lista invalida a medição anterior.

E a fragilidade está no próprio código acima. O padrão precisa ser `[A-Z]-\d{3,4}` e não `[DF]-\d{4}`, porque o corpus tem `D-4471` com quatro dígitos e `F-088` com três — e uma máscara que pega um identificador e deixa o outro produz o pior resultado possível: dois textos que ficaram **parcialmente** iguais, com um resíduo arbitrário decidindo a ordem. É a linha da tabela da [nota 01](01-as-formas-de-recuperar.md), §2, aparecendo de novo: a expressão regular custa zero e quebra quando a redação varia, e aqui ela quebra em silêncio.

---

## Fontes e leituras

MIKOLOV, T. et al. **Efficient estimation of word representations in vector space**. arXiv:1301.3781, 2013.

MUENNIGHOFF, N. et al. **MTEB**: massive text embedding benchmark. arXiv:2210.07316, 2022. (Os *benchmarks* de recuperação medem exatamente o que a §6 chama de "assunto"; nenhuma das quatro cegueiras aparece neles.)

REIMERS, N.; GUREVYCH, I. **Sentence-BERT**: sentence embeddings using siamese BERT-networks. arXiv:1908.10084, 2019.

**Material da disciplina.** Aula 01, [nota 02](../../aula-01-llms-e-agentes/notas-de-aula/02-llms-tokens-e-tokenizacao.md) — tokenização de números. Aula 05, [nota 01-2](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-2-router.md) — o roteador e a rota que não chama o modelo; [nota 03 §8](../../aula-05-arquitetura-de-agentes/notas-de-aula/03-context-engineering-dinamica.md) — a tática zero.
