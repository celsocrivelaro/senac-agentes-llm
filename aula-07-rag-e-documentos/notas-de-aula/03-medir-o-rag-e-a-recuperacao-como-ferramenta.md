# IA Aplicada com LLMs — Aula 07: RAG e Documentos — Medir, e a recuperação como ferramenta

## Introdução

As duas notas anteriores construíram o sistema e o corrigiram. Esta trata de saber se ele funciona — e do caso em que a arquitetura escolhida não basta.

A primeira metade estabelece as duas métricas e, sobretudo, a **razão de mantê-las separadas**. Um número único de qualidade não indica onde intervir, e a consequência prática é conhecida: ajusta-se o *chunking* durante horas para corrigir um defeito que reside no contrato de saída.

A segunda metade fecha a aula com uma pergunta que o pipeline não responde, e ao fazê-lo cobra uma dívida declarada duas aulas antes. O plano da Aula 05 registrou: *"o orquestrador-trabalhador é a mesma forma do RAG com múltiplas consultas; quando chegar lá, a arquitetura já será conhecida e só o retrieval será novo"*.

> **Pré-requisitos:** notas [00](00-do-pdf-ao-texto.md), [01](01-as-formas-de-recuperar.md) e [02](02-o-que-o-embedding-nao-ve.md) desta aula · Aula 05, [nota 01-4](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-4-orquestrador.md) e [nota 02 §1](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) · Aula 06, [nota 03 §6](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md).
>
> **Código:** [`02-avaliar.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula07-rag-e-documentos/02-avaliar.py) e [`03-recuperacao-como-ferramenta.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula07-rag-e-documentos/03-recuperacao-como-ferramenta.py).

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Medir** `recall@k` e fidelidade separadamente, e explicar por que a separação é necessária.
- **Diagnosticar**, a partir do par de valores, se o defeito reside no índice, no prompt ou no corpus.
- **Reconhecer** a classe de perguntas que uma única recuperação não responde.
- **Converter** a recuperação em ferramenta de um agente, com orçamento de consultas.
- **Decidir** entre pipeline e laço em função da pergunta, não do sistema.

---

## Desenvolvimento teórico

### 1. Duas perguntas, duas métricas

Um sistema RAG pode falhar em dois lugares independentes, e cada um tem sua própria pergunta:

| Pergunta | Métrica | Envolve geração |
|---|---|---|
| A busca trouxe o trecho correto? | `recall@k` | não |
| A resposta se apoia no que veio? | fidelidade | sim, uma chamada por resposta |

O `recall@k` é o da Aula 06 ([nota 02](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md), §6), com uma diferença de implementação: aqui a comparação usa o metadado `artigo` que o Chroma armazena, em vez de buscar o texto no chunk.

A fidelidade é nova:

```python
PROMPT_FIDELIDADE = """Verifique se a RESPOSTA se sustenta integralmente nos
TRECHOS fornecidos.
...
Liste em `afirmacoes_sem_lastro` toda afirmação da resposta que não pode ser
verificada nos trechos. `sustentada` é true apenas se a lista estiver vazia.
Não avalie se a resposta é boa; avalie apenas se ela está nos trechos."""
```

O último parágrafo delimita o escopo da avaliação, e a delimitação é o que torna a métrica utilizável. Um avaliador solicitado a julgar se a resposta *está boa* produz elogio — é a armadilha do avaliador-otimizador da Aula 05 (nota 02, §2). Um avaliador solicitado a **enumerar afirmações sem lastro** produz uma lista verificável, e o campo booleano deriva do tamanho dela.

Trata-se da mesma métrica que ES et al. (2023) denominam *faithfulness*, implementada aqui de forma direta para que o mecanismo permaneça visível.

---

### 2. A tabela de diagnóstico

O produto da medição não é um número: é uma decisão sobre onde intervir.

| `recall@k` | Fidelidade | O defeito está em | Onde mexer |
|---|---|---|---|
| baixo | qualquer | **índice** | estratégia de corte, modelo de embedding, `k` |
| alto | baixa | **prompt** | contrato de saída, ordem dos trechos, tamanho do contexto |
| alto | alta, e a resposta ainda incorreta | **corpus** | o documento não contém a informação, ou contém versão desatualizada |

A terceira linha é a do modo de falha 4, que a [nota 04](04-o-pipeline-e-os-quatro-modos-de-falha.md) disseca. As duas métricas aprovam a execução, e a resposta está errada — porque o corpus contém duas versões do mesmo dispositivo e nenhuma métrica sobre recuperação e fidelidade detecta isso.

> **A consequência de não separar.** Com um número único de qualidade, as três linhas se tornam indistinguíveis. O comportamento previsível de quem só dispõe desse número é ajustar o *chunking*, porque é o parâmetro mais visível — e o ajuste não produz efeito quando o defeito está no contrato de saída.

---

### 3. A pergunta que uma consulta não responde

> *"Uma refeição de R$ 340,00 por pessoa em Lisboa, durante viagem a serviço, está dentro da política? E quem aprova?"*

A pergunta exige três dispositivos do regulamento:

| Dispositivo | Conteúdo |
|---|---|
| Art. 4º §1º | o teto da categoria: R$ 120,00 por pessoa |
| Art. 4º §2º | a exceção de viagem internacional: R$ 260,00 |
| Art. 9º §2º | a alçada de aprovação acima de R$ 500,00 |

Uma busca com `k=3` recupera os três mais próximos da pergunta **como um todo**, e a pergunta como um todo não é semelhante a nenhum dos três isoladamente. O resultado típico recupera o Art. 4º §1º e dois trechos adjacentes sobre alimentação, sem a exceção e sem a alçada.

A resposta produzida é errada **e citada**: afirma que R$ 340,00 excede o teto de R$ 120,00, o que é verdadeiro sob o §1º e falso sob o §2º, que era o dispositivo aplicável. A citação torna a resposta mais convincente e, portanto, mais perigosa.

#### 3.1 A dívida da Aula 05

Esta é a mesma despesa do exemplo 2 da Aula 05, [nota 03](../../aula-05-arquitetura-de-agentes/notas-de-aula/03-context-engineering-dinamica.md). Lá, um sub-agente havia lido a exceção de viagem internacional e não a incluíra no resumo devolvido, porque a pergunta delegada não a solicitava; o agente principal reprovou uma despesa legítima **sem qualquer indício de informação faltante**.

Aqui a mesma despesa falha por outra causa: recuperação insuficiente em lugar de perda de contexto. Duas causas distintas produzindo o mesmo sintoma — e a coincidência não é acidental. Ambas decorrem de a pergunta delegada não carregar o critério íntegro:

- na Aula 05, *"qual o teto de reembolso para refeição?"* — sem mencionar destino;
- nesta aula, uma única consulta para três dispositivos.

---

### 4. Recuperação como ferramenta

A saída é conhecida: converter a busca em **ferramenta**, e deixar o laço decidir quantas consultas emitir.

```python
FERRAMENTAS = [{
    "type": "function",
    "function": {
        "name": "buscar_politica",
        "description": ("Busca trechos da política de reembolso. Use uma "
                        "consulta por assunto: chamadas separadas para teto "
                        "da categoria, exceções e alçada de aprovação "
                        "recuperam mais que uma consulta longa."),
        ...
    },
}]
```

A descrição da ferramenta faz trabalho pedagógico e operacional ao mesmo tempo. Ela instrui o modelo a **decompor** a pergunta em consultas por assunto — que é exatamente o que o pipeline não faz —, e é prompt no sentido que a Aula 03 (nota 04, §4) estabeleceu: o único texto que o modelo lê para decidir como usar a ferramenta.

#### 4.1 O orçamento continua valendo

```python
MAX_BUSCAS = 4
...
if len(recuperados) >= MAX_BUSCAS:
    conteudo = {"erro": "teto de buscas atingido",
                "sugestao": "responda com o que já foi recuperado"}
```

Um agente que recupera sem limite é a mesma conta aberta que a Aula 05 (nota 02, §1) identificou no orquestrador sem teto. Cada busca acrescenta trechos ao contexto, e o contexto é reenviado a cada volta — de modo que o custo de uma recuperação excessiva não é linear no número de buscas, e sim quadrático no acumulado, pelo mecanismo da Aula 05 (nota 03, §1).

Note-se também a forma do retorno de erro: ele não apenas informa o teto, mas **indica a próxima ação** (*"responda com o que já foi recuperado"*). É a §4 da Aula 05, [nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md), aplicada aqui — a mensagem de erro é prompt.

#### 4.2 O nome disso

O mercado denomina a arquitetura *agentic RAG*. Pela taxonomia da Aula 05, trata-se do **orquestrador-trabalhador** (nota 01, §3.4): as subtarefas — quais consultas emitir — não constam do código e são determinadas em execução.

O aluno implementou esse padrão duas semanas antes, em outro domínio. O que é novo aqui é o *retrieval*, exatamente como o plano da Aula 05 havia previsto.

---

### 5. A decisão é por pergunta, não por sistema

A conclusão da aula não é que o laço substitui o pipeline.

| | Pipeline | Laço |
|---|---|---|
| Chamadas de geração | 1 | 2 a 5, variável |
| Previsibilidade de custo | total | nenhuma — exige teto |
| Testável etapa por etapa | sim | apenas por propriedade |
| Perguntas que resolve | as de dispositivo único | as que exigem composição |

A distribuição real de perguntas favorece o pipeline: a maioria das consultas a um regulamento envolve um dispositivo. Adotar o laço para todas é pagar variabilidade de custo por uma capacidade exercida em uma fração dos casos.

O instrumento que decide é o mesmo da Aula 05: um **roteador** à frente dos dois mecanismos.

```
   pergunta ──> [ triagem ] ──┬──> um dispositivo    ──> pipeline
                              ├──> composição        ──> laço com ferramenta
                              ├──> id ou valor       ──> consulta direta
                              └──> nenhuma           ──> revisão
```

É a composição de padrões da Aula 05 (nota 02, §4), agora com a recuperação como uma das rotas — e é o desenho que a Parte 2 do trabalho pede.

---

## Exemplos

### Exemplo 1 — O diagnóstico em números

```
  recall@3 = 9/10 (90%) · fidelidade = 6/10 (60%)
  -> PROMPT — o trecho chega e a resposta não se apoia nele
```

*(Valores ilustrativos; a execução do `02-avaliar.py` produz os do modelo em uso.)*

A leitura é imediata pela tabela da §2. O índice está adequado: em nove das dez perguntas o trecho correto foi recuperado. O defeito está entre a recuperação e a resposta — contrato de saída, ordem dos trechos ou instrução insuficiente.

Com um único número de qualidade — por exemplo, "60% de respostas corretas" —, a ação previsível seria revisar a estratégia de corte, que está correta.

### Exemplo 2 — Lisboa, pelos dois caminhos

```
  A — PIPELINE (uma busca, uma resposta)
      recuperou: ['Art. 4º §1º', 'Art. 4º §3º', 'Art. 4º §4º']
      faltou:    ['Art. 4º §2º', 'Art. 9º §2º']
      resposta:  A despesa excede o teto de R$ 120,00 por pessoa
                 estabelecido no Art. 4º §1º.
      fontes:    ['Art. 4º §1º']
      chamadas de geração: 1

  B — LAÇO COM FERRAMENTA
      passo 0: buscar_politica('teto refeição viagem') -> ['Art. 4º §1º', 'Art. 4º §3º']
      passo 1: buscar_politica('viagem internacional exceção') -> ['Art. 4º §2º', 'Art. 2º §3º']
      passo 2: buscar_politica('alçada de aprovação valor') -> ['Art. 9º §2º', 'Art. 9º §1º']
      resposta: Não. Em viagem internacional o teto é de R$ 260,00 por
                pessoa (Art. 4º §2º), e R$ 340,00 o excede. Por estar
                acima de R$ 500,00, a decisão cabe ao gestor (Art. 9º §2º).
      chamadas de geração: 4
```

*(Saída ilustrativa.)*

A execução A custa uma chamada e responde errado com fonte citada. A execução B custa quatro e responde corretamente. A diferença não está no modelo nem no prompt de resposta: está em quantas vezes o sistema teve permissão para consultar.

### Exemplo 3 — O que o teto de buscas evita

Sem `MAX_BUSCAS`, uma pergunta ambígua produz consultas sucessivas sem convergência — o equivalente, na recuperação, ao progresso nulo da Aula 05 ([nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md), §6): as consultas variam, e o conjunto de trechos recuperados deixa de crescer.

O sintoma é reconhecível no log: a partir de certa volta, as buscas devolvem artigos já recuperados. O detector correspondente é o mesmo — assinatura sobre os artigos devolvidos, e não sobre a consulta emitida, porque consultas diferentes recuperando os mesmos trechos é precisamente o caso a detectar.

---

## Fontes e leituras

ES, S. et al. **RAGAS**: automated evaluation of retrieval augmented generation. arXiv:2309.15217, 2023.

BARNETT, S. et al. **Seven failure points when engineering a retrieval augmented generation system**. arXiv:2401.05856, 2024.

LEWIS, P. et al. **Retrieval-augmented generation for knowledge-intensive NLP tasks**. arXiv:2005.11401, 2020.

ANTHROPIC. **Building effective agents**. 2024. (Orquestrador-trabalhador, o padrão que a §4 identifica.)

**Material da disciplina.** Aula 05, [nota 01-4](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-4-orquestrador.md) e [01-6](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-6-qual-padrao-usar.md) — orquestrador-trabalhador e composição de padrões; [nota 02 §1, §4 e §6](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — orçamento, retorno de erro e progresso nulo; [nota 03 §1 e exemplo 4](../../aula-05-arquitetura-de-agentes/notas-de-aula/03-context-engineering-dinamica.md) — o custo quadrático e a despesa de Lisboa. Aula 06, [nota 03 §6](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md) — `recall@k`.
