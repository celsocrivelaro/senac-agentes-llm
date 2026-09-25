# IA Aplicada com LLMs — Aula 07: RAG e Documentos — O pipeline e os quatro modos de falha

## Introdução

A Aula 06 entregou um buscador medido. Ele devolve trechos e não responde nada. Esta nota fecha o circuito — e constata que fechá-lo cria problemas que a busca isolada não tinha.

O assunto desta aula não é *como indexar*: isso foi resolvido. O assunto é o que fazer com cinco trechos recuperados dos quais três são irrelevantes, e o modelo citou o errado.

A nota retoma em uma seção o pipeline montado na Aula 06 e gasta o resto provocando os quatro modos de falha, que é o conteúdo real do encontro: o pipeline acerta as perguntas fáceis, que são a maioria das que se testa manualmente.

> **Pré-requisitos:** Aula 06 integralmente, em especial o pipeline ([nota 04](../../aula-06-embeddings-e-rag/notas-de-aula/05-introducao-ao-rag.md)) e o `recall@k` ([nota 02](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md)) · Aula 05, [nota 01-1](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-1-sequencial.md) · Aula 02, [nota 01 §4.3](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md).
>
> **Código:** [`04-pipeline-demo.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula07-rag-e-documentos/04-pipeline-demo.py) e [`05-modos-de-falha.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula07-rag-e-documentos/05-modos-de-falha.py).

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Montar** o pipeline de geração aumentada por recuperação e **classificá-lo** na taxonomia de padrões da Aula 05.
- **Implementar o portão** entre recuperação e geração, e justificar o limiar adotado.
- **Provocar e diagnosticar** os quatro modos de falha: recuperação incorreta, informação recuperada e ignorada, ausência de resposta no corpus e trechos contraditórios.
- **Distinguir** falha de índice de falha de geração a partir do sintoma observado.

---

## Desenvolvimento teórico

### 1. Onde este pipeline foi montado

O pipeline não é novidade desta nota. Ele foi construído no fim da Aula 06 ([nota 04](../../aula-06-embeddings-e-rag/notas-de-aula/05-introducao-ao-rag.md)), e de lá vêm quatro coisas que esta aula usa sem reapresentar:

| | |
|---|---|
| **a arquitetura** | `pergunta → busca → portão → contexto + LLM → resposta` |
| **o nome** | RAG, de LEWIS et al. (2020) |
| **a classificação** | *prompt chaining* com portão — e **não** um agente |
| **o índice** | um banco de vetores, adotado por persistência e não por desempenho |

E de lá vem também o problema que abre esta nota. O portão daquele laboratório tem limiar de 0,6 de distância, e a pergunta fora de domínio passou por ele a 0,2586 — porque o limiar é mais frouxo que o piso de cosseno medido na [nota 01](../../aula-06-embeddings-e-rag/notas-de-aula/01-o-vetor-e-a-similaridade.md) da Aula 06.

> **O pipeline da Aula 06 roda, e não confere nada do que produz.** Esta aula é sobre o que ele deixa passar.

---

### 2. Os quatro modos não são da mesma natureza

Antes de provocá-los, uma distinção que decide onde se mexe quando cada um aparece:

| Modo | Natureza | A causa tem nome desde a Aula 06 |
|---|---|---|
| **1** — recuperou o trecho errado | **falha de recuperação** | polaridade: o corpus trata do assunto apenas para **excluí-lo** ([cegueira de negação](02-o-que-o-embedding-nao-ve.md)) |
| **2** — recuperou o certo e ignorou | **falha de geração** | — o trecho certo estava lá |
| **3** — não havia resposta, e inventou | **falha de geração** | — não havia o que recuperar |
| **4** — trechos contraditórios | **falha de recuperação** | magnitude e ordem: as duas versões do mesmo artigo diferem só no número, e nada diz qual é a vigente |

**Os modos 1 e 4 já existiam antes desta aula.** São as cegueiras do vetor, e a Aula 06 as demonstrou em cosseno puro — inclusive num router, onde o erro ocorre sem nenhum modelo escrever uma palavra. O que o RAG acrescenta não é a falha: é a **plausibilidade**. O trecho errado volta a ser trecho errado, mas agora vem embrulhado numa resposta bem redigida, com fonte citada.

**Os modos 2 e 3 são novos**, e só existem porque há geração. O trecho certo voltou e o modelo não o usou; ou não havia trecho e o modelo respondeu assim mesmo. Nenhum ajuste de índice corrige qualquer um dos dois.

A distinção não é taxonômica — é operacional, e é o embrião da tabela de diagnóstico da [nota 03](03-medir-o-rag-e-a-recuperacao-como-ferramenta.md): falha de recuperação se conserta no índice, no corte e no `k`; falha de geração se conserta no *prompt* e no contrato de saída. Trocar um pelo outro é o erro que faz alguém passar uma semana ajustando *chunking* para consertar um defeito que está no *prompt*.

---

### 3. Modo de falha 1 — recuperou o trecho errado

**Sintoma.** Resposta bem redigida, confiante e sobre outro assunto.

**Como se provoca.** Uma pergunta cujo assunto o corpus trata apenas para excluí-lo. No regulamento do laboratório, o Art. 7º §2º menciona equipamento de informática somente para determinar que ele *não* se enquadra na categoria material e segue processo de compra. Perguntado sobre o teto de reembolso de um notebook, o vetor aproxima "notebook" de "material de escritório" e recupera o Art. 7º §1º, que estabelece R$ 50,00 por item.

**Por que ninguém percebe.** A resposta cita um artigo real, com um valor real, sobre uma categoria adjacente. Sem inspecionar a lista de trechos recuperados, o erro é indistinguível de um acerto.

**O que ele demonstra.** É o argumento de existir o campo `fontes`, que a [nota 05](05-a-resposta-que-cita-e-que-recusa.md) transforma em requisito. Uma resposta sem fonte não é auditável, e uma fonte que não pode ser conferida contra o corpus não vale mais que nenhuma.

---

### 4. Modo de falha 2 — recuperou o certo e ignorou

**Sintoma.** O trecho correto está entre os recuperados, e a resposta não o utiliza.

**Como se provoca.** A mesma pergunta, os mesmos trechos, em duas **ordens** distintas. O `05-modos-de-falha.py` recupera cinco trechos e responde duas vezes: uma com o trecho correto em primeiro lugar, outra com ele deslocado para o meio da lista.

```python
trechos = indice.buscar(ORDEM["pergunta"], k=5)
for rotulo, ordenados in (("certo em PRIMEIRO", trechos),
                          ("certo no MEIO", trechos[1:3] + trechos[:1] + trechos[3:])):
    r = responder(ORDEM["pergunta"], ordenados)
```

**A explicação.** É o viés em U documentado por LIU et al. (2023) e já apresentado na Aula 02 (nota 01, §4.3): modelos recuperam melhor a informação situada no início e no fim do contexto, e pior a situada no meio. Aqui o fenômeno deixa de ser uma observação sobre janelas longas e passa a ter preço numa janela curta.

**O que ele demonstra.** Se as duas respostas diferem, **nenhuma linha do índice precisa mudar**. O `recall` é idêntico nas duas execuções, por construção. O defeito está na montagem do contexto e no prompt — e é essa distinção que a [nota 03](03-medir-o-rag-e-a-recuperacao-como-ferramenta.md) converte em tabela de diagnóstico.

O tratamento imediato é ordenar os trechos deliberadamente em vez de aceitar a ordem da busca: posicionar o de maior escore no fim do contexto, mais próximo da pergunta, é uma alteração de duas linhas.

---

### 5. Modo de falha 3 — não havia resposta, e o modelo inventou

**Sintoma.** Resposta completa, plausível e sem lastro no corpus.

**Como se provoca.** Perguntas cuja resposta o regulamento não contém. O laboratório usa três:

- *"Qual o teto de reembolso para curso de idiomas?"* — categoria inexistente na política;
- *"Em quantos dias o reembolso é depositado?"* — o Art. 8º trata do prazo de **submissão**, não de pagamento;
- *"Quem aprova a compra de um notebook?"* — o Art. 7º §2º remete a outro processo, sem descrevê-lo.

Nos três casos a busca recupera trechos — sempre recupera, porque devolve os `k` mais próximos independentemente de quão próximos sejam —, o portão frouxo os aprova, e o modelo produz uma resposta.

**Por que é o modo de falha mais caro.** Os outros três produzem respostas erradas sobre assuntos que existem, e uma revisão atenta os detecta. Este produz uma resposta sobre um assunto que **não existe na política**, com a aparência exata de uma resposta correta.

**Como se fecha.** Com uma linha de schema. É o assunto da [nota 05](05-a-resposta-que-cita-e-que-recusa.md), e o mecanismo é conhecido: o campo `suficiente` é a rota `nenhuma` do roteador da Aula 05 (nota 02, §2). A lição é a mesma nas duas aulas — sem rota de escape, o modelo é obrigado a escolher entre opções que não se aplicam, e escolhe.

**Em sala.** Convém apresentar a resposta inventada e solicitar julgamento da turma **antes** de informar que a resposta não estava no corpus. A aceitação é majoritária, e é essa constatação que justifica o requisito.

---

### 6. Modo de falha 4 — trechos contraditórios

**Sintoma.** Resposta correta segundo um dos trechos recuperados, sem menção à existência do outro.

**Como se provoca.** O corpus do laboratório contém duas versões do mesmo dispositivo:

| Versão | Texto | Situação |
|---|---|---|
| 2026 | Art. 4º §1º — limitado a **R$ 120,00** por pessoa | vigente |
| 2024 | Art. 4º §1º — limitado a **R$ 90,00** por pessoa | **revogada** |

A situação não é artificial. Revisões antigas permanecem no repositório de documentos, e alguém deixou de removê-las do índice — que é a origem mais comum de contradição em sistemas de recuperação em produção.

**Por que o vetor não desempata.** Por duas cegueiras simultâneas da [nota 02](02-o-que-o-embedding-nao-ve.md) desta aula: os dois textos diferem apenas em **magnitude numérica** (§3), e a informação de qual é posterior é **temporal** (§5). Nenhuma das duas é representada.

**Por que nenhuma métrica captura.** Este é o ponto que torna o modo de falha 4 distinto dos anteriores. Na execução em que ele ocorre:

- o `recall` está alto — o trecho correto foi recuperado;
- a fidelidade está alta — a resposta se sustenta integralmente nos trechos.

As duas métricas da [nota 03](03-medir-o-rag-e-a-recuperacao-como-ferramenta.md) aprovam a execução, e a resposta pode estar errada. O defeito não está no índice nem no prompt: está no **corpus**.

**O que fica em aberto.** O tratamento exige carimbo de tempo no metadado e desempate em código, e é exatamente o experimento central da Aula 08. Não por acaso: quando o objeto indexado passa a ser a memória do próprio agente, a contradição deixa de ser uma anomalia do corpus e passa a ser o funcionamento normal — fatos mudam ao longo do tempo, e todos permanecem gravados.

---

## Exemplos

### Exemplo 1 — O pipeline nas perguntas fáceis

```
  P: Qual o teto de reembolso para uma refeição em viagem?
  R: O reembolso de refeições em viagem a serviço é limitado a R$ 120,00
     por pessoa por refeição, mediante apresentação de nota fiscal.
     fontes=['Art. 4º §1º']  esperado=Art. 4º §1º  suficiente=True
```

Funciona. E é essa a armadilha da aula: as perguntas que se testa manualmente são as fáceis, e o sistema acerta todas. Os quatro modos de falha não aparecem em nenhuma delas.

### Exemplo 2 — A ordem mudando a resposta

```
  [certo em PRIMEIRO] ordem=['Art. 4º §2º', 'Art. 4º §1º', 'Art. 6º §2º']
     -> Em viagem internacional o limite é de R$ 260,00 por pessoa.
        fontes=['Art. 4º §2º']

  [certo no MEIO]     ordem=['Art. 4º §1º', 'Art. 6º §2º', 'Art. 4º §2º']
     -> O limite para refeições em viagem é de R$ 120,00 por pessoa.
        fontes=['Art. 4º §1º']
```

*(Saída ilustrativa; a execução produz a do modelo em uso.)*

Os mesmos três trechos nas duas execuções. O `recall@3` é 100% nas duas. A resposta muda, e a segunda está errada — o Art. 4º §2º estava disponível e foi ignorado.

Se a execução do laboratório não reproduzir a divergência, o resultado ainda é informativo: significa que, com três trechos curtos, este modelo não apresenta o viés de posição de forma mensurável. A conclusão a registrar passa a ser sobre a **escala** em que o fenômeno aparece, não sobre sua inexistência.

### Exemplo 3 — A resposta sem lastro

```
  P: Em quantos dias o reembolso é depositado na conta do colaborador?
  R: O reembolso é depositado em até 30 dias corridos, conforme o prazo
     estabelecido na política.
```

O Art. 8º §1º estabelece 30 dias para a **submissão do pedido** pelo colaborador. O modelo transportou o prazo para o depósito, que a política não menciona. A resposta é gramaticalmente correta, cita um número que existe no documento, e está errada.

Nenhum ajuste de índice corrige isso: o trecho recuperado é o único que trata de prazos, e é o mais próximo que existe. O conserto é permitir que a resposta seja *"a política não trata do prazo de depósito"* — e a próxima nota trata de como se obtém isso.

---

## Fontes e leituras

BARNETT, S. et al. **Seven failure points when engineering a retrieval augmented generation system**. arXiv:2401.05856, 2024.

LEWIS, P. et al. **Retrieval-augmented generation for knowledge-intensive NLP tasks**. arXiv:2005.11401, 2020.

LIU, N. F. et al. **Lost in the middle**: how language models use long contexts. arXiv:2307.03172, 2023.

ANTHROPIC. **Building effective agents**. 2024. (A taxonomia que classifica o pipeline como *prompt chaining*.)

**Material da disciplina.** Aula 05, [nota 01-1](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-1-sequencial.md) e [01-2](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-2-router.md) — *chaining* com portão e a rota `nenhuma`. Desta aula, [nota 02](02-o-que-o-embedding-nao-ve.md) — as cegueiras de número e tempo, que explicam o modo de falha 4. Aula 04, [nota 02](../../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/02-os-casos-que-falharam.md) — *agent washing*.
