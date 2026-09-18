# IA Aplicada com LLMs — Aula 06: Embeddings e RAG — Introdução ao RAG

## Introdução

As quatro notas anteriores construíram um buscador inteiro sem que nada escrevesse uma palavra. O vetor, o cosseno, o corte em chunks, a medida em `recall@k`, um router que classifica sem gerar texto e um banco onde tudo isso persiste: é tudo recuperação, e a recuperação para nos trechos.

Esta nota fecha a volta. Os trechos recuperados viram **resposta**, a arquitetura ganha nome, e aparece a etapa que quase ninguém escreve — o portão entre a busca e a geração.

O que ela não faz é conferir o resultado. Se a resposta usou os trechos que voltaram, se a fonte citada existe, se o modelo deveria ter recusado: são quatro perguntas, e as quatro são a Aula 07.

> **Pré-requisitos:** notas [01](01-o-vetor-e-a-similaridade.md), [02](02-chunking-e-a-medida-da-busca.md), [03](03-o-router-por-embedding.md) e [04](04-bancos-vetoriais.md) desta aula · Aula 05, [nota 01-1](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-1-sequencial.md) — *prompt chaining* com portão.
>
> **Código:** [`06-rag-simples.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula06-embeddings-e-rag/06-rag-simples.py). A função `rag_simples` mora no próprio script — é usada só ali. O índice em banco de vetores está no `indice_chroma.py`, e a chamada de geração no `geracao.py`.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Montar** o pipeline de geração aumentada por recuperação e **classificá-lo** na taxonomia de padrões da Aula 05.
- **Reconhecer** que o banco de vetores nunca recusa uma consulta, e que a recusa é responsabilidade do código que o chama.
- **Implementar o portão** entre recuperação e geração, e **explicar por que um limiar escolhido no chute não funciona**, usando o piso de cosseno medido na nota 01.
- **Distinguir** distância de similaridade, e reconhecer qual das duas o banco devolve.

---

## Desenvolvimento teórico

### 1. O pipeline, e o nome dele

A arquitetura tem quatro etapas:

```
   pergunta ──> [ busca ] ──> portão ──> [ contexto + LLM ] ──> resposta
                    │                            │
              notas 01 a 03                    esta nota
```

O nome consagrado é **RAG** — *retrieval-augmented generation* —, proposto por LEWIS et al. (2020). O termo importa por duas razões: é o que o mercado utiliza, e é o que a Parte 2 do trabalho cobra.

Importa igualmente **classificar** a arquitetura. Pela taxonomia da Aula 05 (nota 01-1), o pipeline acima é *prompt chaining* com portão: o caminho está no código, o número de chamadas é conhecido antes da execução, e nenhuma etapa depende de descoberta feita durante a execução.

> **Um pipeline RAG não é um agente.** Chamá-lo de agente é precisamente o *agent washing* que a Aula 04 (nota 02) identificou no mercado. A distinção não é terminológica: um workflow é testável etapa por etapa e tem custo previsível, e desistir dessas duas propriedades sem necessidade é o erro que a Aula 05 inteira combate.

---

### 2. O índice, que já está pronto

A [nota 04](04-bancos-vetoriais.md) trocou a matriz de numpy por um banco de vetores e mediu o preço da troca. Três coisas de lá entram aqui sem reapresentação:

- **o argumento**: a troca é por persistência, não por desempenho — nesta escala o banco não traz ganho de velocidade nenhum;
- **o formato do retorno**: o banco devolve **distância**, e baixo é bom;
- **o limite**: o banco devolve os `k` mais próximos, sempre, e **nunca diz "não tenho isso"**.

A terceira é a que importa para esta nota. Se o banco sempre responde, alguém precisa decidir quando a resposta dele não serve — e esse alguém é a próxima seção.

---

### 3. O portão

A etapa que quase nunca é implementada:

```python
if not trechos or trechos[0]["distancia"] > piso_distancia:
    return {"resposta": "A busca não recuperou trecho suficientemente "
                        "próximo. Encaminhado para revisão.",
            "trechos": trechos, "barrado_no_portao": True}
```

Se a busca não trouxe nada próximo, **a etapa de geração não roda**. É o mesmo portão do *prompt chaining* da Aula 05: código determinístico entre duas chamadas, interrompendo a cadeia antes que uma etapa opere sobre entrada inválida.

O portão custa zero chamadas e evita uma. É a tática zero da Aula 05 num lugar novo: o caminho mais barato é o que não envolve o modelo.

#### 3.1 E o limiar, que não funciona

O laboratório usa `0,6` de distância. O script roda três perguntas, e a terceira — *"qual o índice de reajuste da tabela de fretes marítimos?"* — **não tem resposta no regulamento**. O portão deveria barrá-la.

Ele não barra:

```
  'qual o teto de refeição em viagem?'        [Art. 4º §1º]  d = 0,2359
  'e em viagem internacional?'                [Art. 2º §3º]  d = 0,1831
  'reajuste da tabela de fretes marítimos?'   [Art. 5º §1º]  d = 0,2586   <- passa
```

A pergunta fora de domínio veio a **0,2586** de distância, ou **0,7414** de cosseno. E esse número já era conhecido: é o **piso** que a nota 01 (§5) mediu entre textos sem relação nenhuma, onde o campeonato de futebol marcou 0,7487 contra o regulamento.

Um limiar de 0,6 de distância equivale a 0,4 de cosseno — muito abaixo do piso. Ele só barra índice vazio.

> **O portão está no lugar certo. O limiar é que foi chutado.** É a mesma conclusão da nota 01 — limiar absoluto escolhido por intuição não funciona —, agora com consequência: sem calibragem, a etapa de geração roda sobre trechos irrelevantes e responde mesmo assim.

Calibrar exige o que a nota 02 construiu: o conjunto de perguntas com resposta conhecida, acrescido de perguntas **sem** resposta no corpus. O exercício pede essa calibragem.

---

### 4. O que este pipeline não confere

O `06-rag-simples.py` monta um RAG que funciona e **não verifica nada do que produz**:

| Pergunta que ele não faz | Onde ela é respondida |
|---|---|
| a resposta usou os trechos que voltaram? | Aula 07 — fidelidade |
| a fonte citada existe entre os trechos? | Aula 07 — citação verificável |
| o modelo deveria ter recusado? | Aula 07 — o contrato de saída |
| qual limiar barra o que precisa ser barrado? | Aula 07 — e o exercício desta aula |

A distância entre *"o pipeline roda"* e *"dá para confiar no que ele devolve"* é a Aula 07 inteira.

---

## Exemplos

### Exemplo 1 — A pergunta fácil, ponta a ponta

> *"qual o teto de refeição em viagem?"*

A busca devolve `Art. 4º §1º` a 0,2359 de distância. O portão deixa passar — corretamente, desta vez. Os três trechos viram contexto rotulado, e o modelo responde com o valor do artigo.

Uma chamada de embedding para a pergunta, uma de geração para a resposta. O índice já estava construído.

### Exemplo 2 — A pergunta que o corpus não responde

> *"qual o índice de reajuste da tabela de fretes marítimos?"*

O trecho mais próximo é o `Art. 5º §1º`, que trata de transporte — assunto adjacente, resposta nenhuma. A distância de 0,2586 passa pelo portão, e o modelo recebe três trechos irrelevantes com a instrução de responder a partir deles.

O prompt do laboratório pede que ele diga quando não há informação. **Pedir não é garantir**, e a Aula 07 mede a diferença.

### Exemplo 3 — O custo de reconstruir

Construir o índice: uma chamada de embedding, com os 28 chunks juntos. Consultar: uma chamada, com a pergunta.

Com o índice em memória, toda execução paga a primeira. Com o banco em disco, ela é paga uma vez — e é esse, e só esse, o argumento da troca.

---

## Fontes e leituras

LEWIS, P. et al. **Retrieval-augmented generation for knowledge-intensive NLP tasks**. arXiv:2005.11401, 2020. (O artigo que nomeia a arquitetura.)

ANTHROPIC. **Building effective agents**. 2024. (A taxonomia que classifica o pipeline como *prompt chaining*, e não como agente.)

CHROMA. **Documentação**: métricas de distância e coleções persistentes.

**Material da disciplina.** Aula 05, [nota 01-1](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-1-sequencial.md) — *chaining* com portão. Aula 04, [nota 02](../../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/02-os-casos-que-falharam.md) — *agent washing*. Nota [01](01-o-vetor-e-a-similaridade.md) desta aula, §5 — o piso de cosseno que explica por que o limiar de 0,6 não barra nada.
