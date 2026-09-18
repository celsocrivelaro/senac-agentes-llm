# IA Aplicada com LLMs — Aula 06: Embeddings e busca semântica — O que o embedding não vê

## Introdução

A [nota anterior](01-o-vetor-e-a-similaridade.md) encerrou com uma distinção que não é defeito de modelo: similaridade não é relevância, porque são grandezas diferentes. Esta nota trata do que **é** defeito — ou, mais precisamente, do que um vetor de embedding comprovadamente não representa.

O tema não é acessório. Sem ele, a conclusão razoável ao fim da nota anterior seria que busca semântica resolve o problema de encontrar informação, e que o trabalho restante é de ajuste. As quatro demonstrações a seguir mostram o contrário: há classes inteiras de comparação para as quais o vetor é a ferramenta errada, e todas as quatro aparecem no domínio de prestação de contas usado no laboratório.

O método adotado é uniforme. Para cada par de textos, registra-se primeiro a **similaridade que a intuição prevê**, depois a **similaridade obtida**. A distância entre as duas é o conteúdo da nota.

> **Pré-requisitos:** [nota 01](01-o-vetor-e-a-similaridade.md) desta aula.
>
> **Código:** [`02-cegueiras.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula06-busca/02-cegueiras.py). O script pausa entre os pares para o registro da previsão; `--sem-pausa` desativa a interação.

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

Antes de qualquer medição, é necessário estabelecer a referência. Um valor de cosseno isolado não é interpretável: 0,84 é alto ou baixo?

O par de controle responde a essa pergunta. Ele reúne dois textos **sem relação alguma** entre si:

| | Texto |
|---|---|
| A | "a despesa foi aprovada pelo analista" |
| B | "a previsão do tempo indica chuva no litoral norte" |

O valor obtido para esse par é o piso prático da escala. Textos em português compartilham estrutura, e o modelo captura essa estrutura — de modo que o piso fica bem acima de zero, como a nota anterior estabeleceu.

Todas as medições seguintes devem ser lidas **em relação a esse piso**, e não em relação a zero. É a diferença entre concluir "o modelo achou os textos parecidos" e concluir "o modelo não distinguiu os textos mais do que distinguiria de uma frase sobre o tempo".

---

### 2. Negação

| | Texto |
|---|---|
| A | "a despesa foi aprovada pelo analista" |
| B | "a despesa foi **reprovada** pelo analista" |

**Previsão da intuição:** baixa. Os textos afirmam o oposto um do outro, e a decisão que cada um representa é contrária.

**Resultado obtido:** alta, e tipicamente acima de qualquer outro par desta nota.

**A explicação.** Um modelo de embedding representa o contexto de ocorrência de um termo. "Aprovado" e "reprovado" ocorrem exatamente nos mesmos contextos — mesmos sujeitos, mesmos complementos, mesmos documentos. É essa co-ocorrência que o vetor captura, e a polaridade da decisão não altera o contexto.

**Consequência em produção.** Um sistema de busca sobre pareceres que receba a consulta *"despesas reprovadas por falta de nota fiscal"* recupera com igual facilidade os pareceres de aprovação. Nenhum ajuste de *chunking* corrige isso, porque a informação não está sendo perdida no corte — ela nunca esteve no vetor.

**O que fazer.** A polaridade é um campo, não um trecho de texto. Um parecer tem `veredito ∈ {aprovado, reprovado}`, e filtrar por esse campo é uma cláusula de consulta, não uma comparação de vetores. O laboratório aplica exatamente isso: a memória episódica da Aula 08 guarda `veredito` como **metadado**, e a recuperação filtra por ele antes de ordenar por similaridade.

---

### 3. Número

| | Texto |
|---|---|
| A | "o teto de reembolso é de R$ 120,00 por refeição" |
| B | "o teto de reembolso é de R$ 260,00 por refeição" |

**Previsão da intuição:** baixa. Os dois valores decidem casos diferentes: uma despesa de R$ 200,00 é reprovada sob A e aprovada sob B.

**Resultado obtido:** muito alta — os textos diferem em três caracteres.

**A explicação.** Números são tokenizados como qualquer outra sequência, e a tokenização não preserva magnitude. A Aula 01 (nota 02) já havia observado que `120` e `260` podem ocupar quantidades diferentes de tokens sem qualquer relação com a diferença aritmética entre eles. Um modelo treinado para representar contexto não tem incentivo para codificar ordem numérica.

**Consequência em produção.** Este é o caso mais perigoso dos quatro, porque produz erro **silencioso e plausível**. O sistema recupera o artigo com o teto errado, e o texto recuperado é do mesmo assunto, com a mesma redação, citando o mesmo artigo. A revisão humana não desconfia.

A Aula 07 explora esse caso diretamente: o corpus do laboratório contém a versão vigente do Art. 4º §1º (R$ 120,00) e uma versão revogada de 2024 (R$ 90,00), e o pipeline recupera as duas sem notar que há duas.

**O que fazer.** Comparação de valor é `<=`. O teto aplicável a uma despesa não se descobre por busca — descobre-se recuperando o **artigo** e lendo o campo numérico dele. A distinção é entre usar o vetor para localizar a regra e usá-lo para aplicar a regra; apenas a primeira função é apropriada.

---

### 4. Entidade

| | Texto |
|---|---|
| A | "análise da despesa D-4471 do funcionário F-088" |
| B | "análise da despesa D-4472 do funcionário F-091" |

**Previsão da intuição:** baixa. São registros distintos, de funcionários distintos.

**Resultado obtido:** altíssima, e pelo mesmo motivo do caso anterior.

**A explicação.** Um identificador não carrega significado distribuído: `D-4471` e `D-4472` são formalmente idênticos exceto por um dígito, e semanticamente ambos são "um identificador de despesa". O vetor representa a categoria, não a instância.

**Consequência em produção.** Buscar `D-4471` num corpus de mil despesas recupera mil trechos igualmente parecidos, e a ordenação entre eles é arbitrária. Um sistema que dependa disso funciona no teste com dez registros e falha em produção com dez mil — o modo de falha mais caro que existe, porque só aparece depois do *deploy*.

**O que fazer.** Identificador se resolve por `==` ou por índice invertido. O padrão adequado é o roteador da Aula 05 (nota 02, §2): se a consulta contém um identificador bem formado, a rota é uma consulta direta, sem chamada ao modelo e sem busca vetorial.

---

### 5. Tempo

| | Texto |
|---|---|
| A | "em março de 2026, o teto de refeição passou a ser R$ 120,00" |
| B | "em agosto de 2026, o teto de refeição passou a ser R$ 150,00" |

**Previsão da intuição:** baixa. Um dos dois está revogado pelo outro.

**Resultado obtido:** alta. Ambos os textos contêm uma data, e nenhum contém informação sobre qual data é posterior.

**A explicação.** O vetor não representa anterioridade. "Março" e "agosto" são dois meses; que um preceda o outro é conhecimento aritmético sobre o calendário, não propriedade do contexto textual.

**Consequência em produção.** Duas afirmações verdadeiras em momentos diferentes disputam a mesma consulta, e a que vence é a **mais parecida com a pergunta**, não a vigente. Se a pergunta for formulada no presente — *"qual **é** o teto?"* —, ela tende a favorecer o texto redigido no presente, que pode ser o revogado.

**O que fazer.** Carimbo de tempo como **metadado**, e desempate no código:

```python
def mais_recente(candidatos: list[dict], campo: str = "data") -> dict | None:
    return max(candidatos, key=lambda c: c[campo]) if candidatos else None
```

Uma linha, determinística, sem chamada nenhuma. Delegar o desempate ao modelo é o antipadrão: ele erra, gasta uma chamada e não é testável.

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

## Exemplos

### Exemplo 1 — O quadro completo, lido contra o controle

```
  NEGAÇÃO     0.9412  ██████████████████████████████████████████████
  NÚMERO      0.9738  ████████████████████████████████████████████████
  ENTIDADE    0.9615  ███████████████████████████████████████████████
  TEMPO       0.9203  ██████████████████████████████████████████████
  CONTROLE    0.5518  ███████████████████████████     <- textos SEM RELAÇÃO
```

*(Valores ilustrativos; a execução do `02-cegueiras.py` produz os do modelo em uso.)*

A leitura decisiva está na comparação com a última linha. Textos que afirmam o oposto ficam em 0,94; textos sem relação alguma ficam em 0,55. Para um leitor humano, a distância entre "aprovado" e "reprovado" é total. Para o vetor, ela é **menor que a distância entre qualquer um dos dois e uma frase sobre o tempo**.

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

---

## Fontes e leituras

MIKOLOV, T. et al. **Efficient estimation of word representations in vector space**. arXiv:1301.3781, 2013.

MUENNIGHOFF, N. et al. **MTEB**: massive text embedding benchmark. arXiv:2210.07316, 2022. (Os *benchmarks* de recuperação medem exatamente o que a §6 chama de "assunto"; nenhuma das quatro cegueiras aparece neles.)

REIMERS, N.; GUREVYCH, I. **Sentence-BERT**: sentence embeddings using siamese BERT-networks. arXiv:1908.10084, 2019.

**Material da disciplina.** Aula 01, [nota 02](../../aula-01-llms-e-agentes/notas-de-aula/02-llms-tokens-e-tokenizacao.md) — tokenização de números. Aula 05, [nota 01-2](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-2-router.md) — o roteador e a rota que não chama o modelo; [nota 03 §8](../../aula-05-arquitetura-de-agentes/notas-de-aula/04-context-engineering-dinamica.md) — a tática zero.
