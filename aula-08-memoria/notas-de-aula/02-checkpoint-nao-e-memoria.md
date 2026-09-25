# IA Aplicada com LLMs — Aula 08: Memória — Checkpoint não é memória

## Introdução

A Aula 03 (nota 04, §9) encerrou com uma lista de pendências. Após a Aula 05, três permaneciam de pé: memória entre execuções, MCP e sistemas multiagente. Esta aula encerra a primeira.

A [nota 01](01-o-agente-e-o-estado.md) construiu o objeto de estado e o gravou em disco (§8). A fronteira que ela não trata é esta:

> Persistir estado **no interior de uma única execução**, para retomá-la, é *checkpoint*. Persistir estado **entre execuções distintas** — o agente recuperar o que ocorreu na semana anterior — é memória, e é o objeto desta nota.

A distinção parece sutil e não é. Os dois mecanismos gravam estado em disco, usam a mesma serialização e são lidos no início de uma execução. Divergem no que fazem com o que leram — e essa divergência determina duas arquiteturas incompatíveis.

Esta nota estabelece a fronteira, situa-a no eixo *short-term* / *long-term* com que a literatura organiza o assunto, e apresenta a taxonomia que estrutura o restante da aula.

> **Pré-requisitos:** a [nota 01](01-o-agente-e-o-estado.md) desta aula, §3 (o objeto de estado) e §8 (o *checkpoint*) · Aula 07 integralmente · Aula 07, [nota 02 §5](../../aula-07-rag-e-documentos/notas-de-aula/02-o-que-o-embedding-nao-ve.md).
>
> **Código:** [`01-checkpoint-vs-memoria.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula08-memoria/01-checkpoint-vs-memoria.py) e os quatro [`02-memoria-*.py`](https://github.com/celsocrivelaro/senac-llm-code/tree/main/aula08-memoria) — um por tipo de memória, mais o que monta os três sob orçamento de janela.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Distinguir** *checkpoint* de memória, e demonstrar que a serialização de todas as execuções não produz memória.
- **Situar** o *checkpoint* e as três memórias no eixo *short-term* / *long-term* da literatura, e justificar por que uma janela de contexto maior não substitui a segunda.
- **Classificar** uma informação em episódica, semântica ou procedural, e **selecionar a estrutura de armazenamento** correspondente.
- **Separar** *checkpoint*, memória e RAG pela origem do que cada um guarda, e reconhecer que só a memória não tem curadoria externa.
- **Justificar** por que buscar um fato semântico por similaridade é inadequado.
- **Identificar** a informação que não deve ser armazenada.

---

## Desenvolvimento teórico

### 1. A fronteira

|  | **Checkpoint** ([nota 01](01-o-agente-e-o-estado.md) §8) | **Memória** (daqui em diante) |
|---|---|---|
| Escopo | uma execução | todas as execuções |
| Propósito | **retomar** | **lembrar** |
| Forma | estado íntegro | fragmento selecionado |
| Leitura | uma vez, no início | por relevância, a cada volta |
| Crescimento | não cresce | cresce sem teto |
| Descarte | ao concluir a execução | é decisão de projeto (nota 03) |

A linha decisiva é a terceira. Um *checkpoint* é útil porque preserva o estado **íntegro**: retomar exige saber exatamente onde a execução parou, e qualquer omissão inviabiliza a retomada. Memória é útil porque preserva **fragmentos selecionados**: recuperar tudo o que já aconteceu é indistinguível de não recuperar nada.

#### 1.1 A questão que abre a aula

Examina-se a proposição:

> A serialização do `Estado` de **todas** as execuções produz memória.

A proposição é falsa: o que se obtém é arquivo morto. O laboratório torna a afirmação concreta. Dez execuções anteriores do agente de prestação de contas somam 36 passos e pouco mais de 21 mil tokens. Diante de uma tarefa nova — analisar uma refeição de R$ 245,00 em Lisboa —, o arquivo íntegro oferece duas opções, ambas inúteis:

- **incluir tudo no contexto**: cerca de 5 mil tokens de execuções majoritariamente irrelevantes, competindo por espaço com a trajetória corrente;
- **não incluir nada**: que é o que ocorre na prática quando ninguém lê o arquivo.

O que falta entre as duas é **recuperação**, e recuperar exige decidir o que é relevante agora. Que é a Aula 07 apontada para dentro: mesmo mecanismo, objeto diferente.

| | Aula 07 | Aula 08 |
|---|---|---|
| O que é indexado | documentos | trajetórias do próprio agente |
| Quem escreveu | terceiros | o agente, em execuções anteriores |
| Muda com o tempo | raramente | **continuamente** |
| Contradição é | anomalia do corpus | funcionamento normal |

A última linha antecipa a [nota 04](04-esquecer-e-proteger.md). Num corpus de documentos, duas versões do mesmo dispositivo indicam falha de curadoria. Numa memória, dois fatos verdadeiros em momentos diferentes são o resultado esperado de um sistema que acumula.

#### 1.2 As duas perguntas

A tabela da §1 **descreve** a fronteira. O que a demonstra é fazer a mesma pergunta às duas estruturas, duas vezes.

> **Pergunta 1 — *"retome a execução `exec-0a1f` exatamente onde ela parou".***
>
> O *checkpoint* responde: tem os passos com argumentos e resultados, as ferramentas ativas, o histórico e o término. O fragmento guardado na memória **não** responde.

> **Pergunta 2 — *"o que já se sabe que ajude na despesa de agora?".***
>
> A memória responde: fragmentos selecionados por relevância, algumas dezenas de tokens. O *checkpoint* **não** responde — não há por onde perguntar, e responder exigiria carregar as dez execuções íntegras no contexto.

Cada estrutura falha na pergunta da outra, e falha pela mesma razão: **cada uma guardou o que a sua própria pergunta exige**. Não há aqui uma estrutura melhor e outra pior. O *checkpoint* da nota anterior não é uma memória deficiente — é outra coisa, e é a partir desta nota que a memória é construída.

A distinção é visível no formato. A mesma execução, nas duas formas, tem um campo `passos` em ambas — e não é o mesmo campo:

| | `passos`, no *checkpoint* | `passos`, na memória |
|---|---|---|
| Conteúdo | a lista, com argumentos e resultados de cada um | o número `3` |
| Serve para | reconstruir onde a execução estava | dimensionar o que ela custou |

Um guarda **o que** aconteceu; o outro guarda **que** aconteceu. É a linha "estado íntegro × fragmento selecionado" da tabela, em campo concreto — e é por isso que uma varredura por nome de campo não distingue as duas estruturas, enquanto uma tentativa de retomada distingue na primeira linha.

A confusão entre as duas tem consequência prática e tardia: equipes que serializam o `Estado` e chamam isso de memória descobrem, meses depois, que mantêm um arquivo que ninguém lê e que ninguém sabe consultar.

#### 1.3 A terceira estrutura: o RAG

A §1.1 observou que recuperar memória é a Aula 07 apontada para dentro. A proximidade entre os dois mecanismos é real, e produz a confusão mais comum da aula. Ela se desfaz com uma terceira pergunta:

> **Pergunta 3 — *"o que está escrito que ajude na despesa de agora?".***
>
> O RAG responde: os trechos da política de despesas e do regulamento que tratam de refeição no exterior. Nem o *checkpoint* nem a memória respondem — nenhum dos dois contém aquilo que o agente **nunca viveu**.

As três estruturas guardam texto, são lidas antes de decidir e podem usar o mesmo índice vetorial. Divergem na origem do que guardam, e tudo o mais decorre disso:

| | ***Checkpoint*** | **Memória** | **RAG** |
|---|---|---|---|
| Pergunta | onde **esta** execução parou | o que o agente **já viveu** que ajude agora | o que **está escrito** que ajude agora |
| Conteúdo | o estado íntegro de uma execução | fragmentos das execuções anteriores | documentos de terceiros |
| Quem escreveu | o próprio agente, **agora** | o próprio agente, **antes** | **terceiros**, fora do sistema |
| Escopo | uma execução | todas as execuções do agente | o corpus, independente do agente |
| Muda | a cada passo | a cada execução | raramente, por curadoria |
| Contradição é | impossível — há uma execução só | **funcionamento normal** | falha de curadoria |
| Onde na disciplina | [nota 01](01-o-agente-e-o-estado.md) §8 | esta aula | Aula 07 |

A linha decisiva é "quem escreveu", e a consequência aparece duas linhas abaixo. Num corpus de RAG existe alguém responsável pela correção do que está lá: o documento foi redigido, revisado e publicado por uma área. **Numa memória, o autor é o próprio agente, e não há curadoria.** É a razão de existir a [nota 04](04-esquecer-e-proteger.md): o corpus da Aula 07 é mantido por terceiros, e o desta aula tem de se manter sozinho.

Os três coexistem numa execução real, e sem hierarquia entre eles: o *checkpoint* retoma de onde parou, o RAG traz a norma aplicável, a memória traz o precedente. O Exemplo 2 desta nota mostra a combinação.

---

### 2. Curto e longo prazo

A fronteira da §1 tem nome na literatura, e o nome é anterior a qualquer taxonomia. Antes de distinguir tipos de memória, os textos de referência distinguem **duração**:

| | *Short-term memory* (memória de trabalho) | *Long-term memory* |
|---|---|---|
| Alcance | a interação corrente | todas as sessões |
| Implementação típica | *rolling buffer* ou a própria janela de contexto | banco de dados, grafo de conhecimento, índice vetorial |
| Ao terminar a sessão | não sobrevive | persiste |
| Serve para | manter continuidade dentro da tarefa | personalizar e aprender ao longo do tempo |

A IBM é direta quanto ao limite da primeira: ela *"melhora a continuidade em interações curtas, mas não retém informação além da sessão, o que a torna inadequada para personalização ou aprendizado de longo prazo"* (IBM, 2025; tradução livre, como nas demais citações desta seção). É a mesma insuficiência que a §1 descreveu em outro vocabulário.

#### 2.1 O mapeamento

O eixo não introduz conceito novo nesta aula. Ele **nomeia** o que as quatro notas constroem, e a tabela abaixo é a tradução completa:

| Literatura | Nesta aula | Onde |
|---|---|---|
| ***short-term memory*** | a janela: *system prompt*, objetivo e trajetória corrente | [nota 01](01-o-agente-e-o-estado.md) §3 e §4 |
| — sua persistência | o ***checkpoint*** | [nota 01](01-o-agente-e-o-estado.md) §8 |
| — sua compressão | *tool clearing* e *compaction* | [nota 03](03-escrever-e-ler.md) §5 |
| ***long-term memory*** | as três memórias | §3 desta nota |
| — *extraction* | as três políticas de escrita | [nota 03](03-escrever-e-ler.md) §1 |
| — *consolidation* | contradição e decaimento | [nota 04](04-esquecer-e-proteger.md) §1 e §2 |
| — *retrieval* | o orçamento de contexto | [nota 03](03-escrever-e-ler.md) §4 |

A coluna do meio já existia. O que a coluna da esquerda acrescenta é a capacidade de ler a documentação de qualquer *framework* de agentes sem tradução: *thread-scoped* e *cross-session* são os adjetivos com que a distinção aparece em código.

A consequência imediata está na §3. Episódica, semântica e procedural são **subtipos da memória de longo prazo**, e não uma lista plana de três. A pergunta *"qual das três guarda a trajetória da execução corrente?"* não tem resposta: nenhuma guarda, porque a trajetória é curto prazo.

#### 2.2 O *checkpoint*, renomeado

O eixo colide com o título desta nota, e a colisão é instrutiva. O Redis descreve a implementação da memória de curto prazo nestes termos: *"são necessários mecanismos de checkpoint [...] para persistir o estado no nível da thread"* (REDIS, 2026). O que a [nota 01](01-o-agente-e-o-estado.md) §8 construiu é, na nomenclatura da literatura, **memória de curto prazo persistida**.

A tese da §1.2 permanece de pé, e agora com o escopo explícito:

> O *checkpoint* não é uma memória de **longo prazo** deficiente. Onde esta aula escreve "memória" sem qualificar, leia-se memória de longo prazo — que é o objeto desta nota em diante.

O vocabulário duplo não é acidente de tradução. A aula separa as duas estruturas porque elas têm **arquiteturas incompatíveis**; a literatura as agrupa sob o mesmo substantivo porque ambas guardam estado. As duas leituras são compatíveis desde que o escopo seja dito.

#### 2.3 Janela grande não substitui memória de longo prazo

A objeção previsível é que janelas de centenas de milhares de *tokens* tornam o assunto desnecessário. Ela falha por uma razão estrutural: **a janela é reiniciada a cada requisição**. Uma janela maior compra mais curto prazo, e nunca longo prazo.

Três necessidades permanecem independentemente do tamanho da janela:

| Necessidade | Por que a janela não atende |
|---|---|
| Persistência por dias e semanas | a sessão termina, e com ela o conteúdo da janela |
| Aprendizado entre sessões | o que foi aprendido na execução anterior não está na requisição atual |
| Acesso seletivo | o agente precisa do fragmento relevante, não de tudo o que já ocorreu |

A terceira é a da §1.1, com outro nome: recuperar tudo é indistinguível de não recuperar nada. E a Aula 05 ([nota 03](../../aula-05-arquitetura-de-agentes/notas-de-aula/03-context-engineering-dinamica.md), §1) acrescenta o argumento de custo — o consumo acumulado cresce com o quadrado do número de passos, e a qualidade cai antes de a janela acabar.

---

### 3. As três memórias

A memória de longo prazo da §2 não é um bloco único. A literatura sobre agentes a divide em três subtipos — a formulação corrente vem do CoALA — e a divisão só tem valor prático porque os três exigem **estruturas de armazenamento diferentes**. Reduzi-los a um único mecanismo é o erro que leva alguém a construir um índice vetorial para armazenar o nome de um usuário.

| Tipo | No domínio de prestação de contas | Estrutura | Custo de consulta |
|---|---|---|---|
| **Episódica** | *"em 12/03 a despesa D-4102 do F-088 foi aprovada por estar dentro do teto"* | índice vetorial | 1 chamada de embedding + busca |
| **Semântica** | *"F-088 viaja para Portugal com frequência"* | chave-valor | zero chamadas |
| **Procedural** | *"quando o recibo diverge do valor declarado, devolver antes de analisar o mérito"* | texto, no *system prompt* | zero chamadas |

#### 3.1 Episódica

O que aconteceu, e quando. É a única das três que se recupera por similaridade, porque é a única em que a pergunta relevante — *"já ocorreu algo parecido com isto?"* — é uma pergunta sobre **assunto**, que é precisamente o que o vetor representa (Aula 07, [nota 02](../../aula-07-rag-e-documentos/notas-de-aula/02-o-que-o-embedding-nao-ve.md), §6).

```python
class MemoriaEpisodica:
    def gravar(self, episodios: list[dict]) -> None:
        self.colecao.add(
            ids=[e["id"] for e in episodios],
            documents=[e["resumo"] for e in episodios],
            metadatas=[{"data": e["data"], "funcionario": e.get("funcionario", ""),
                        "veredito": e.get("veredito", "")} for e in episodios],
            embeddings=gerar_matrix_embbeddings(textos).tolist(),
        )
```

Três campos vão como **metadado**, e a escolha de cada um decorre diretamente das cegueiras da Aula 06:

| Metadado | Cegueira que ele contorna |
|---|---|
| `data` | tempo (§5) — o vetor não representa anterioridade |
| `funcionario` | entidade (§4) — identificador não é significado |
| `veredito` | negação (§2) — "aprovado" e "reprovado" são quase idênticos para o vetor |

E o método de recuperação filtra por eles **antes** de ordenar por similaridade:

```python
def recuperar(self, consulta: str, k: int = 3,
              funcionario: str | None = None,
              veredito: str | None = None) -> list[dict]:
    """Filtra por metadado ANTES de ordenar por similaridade."""
```

A consulta *"despesas reprovadas do F-088 por falta de nota"* combina os três mecanismos: dois filtros determinísticos e uma busca por assunto sobre o que sobrar. Nenhum ajuste de busca semântica produziria o mesmo resultado, pelos motivos que a Aula 06 estabeleceu.

#### 3.2 Semântica

Fato estável sobre uma entidade. É uma tabela, e se lê por chave:

```python
semantica.gravar("F-088", "destinos_frequentes", ["Lisboa", "Curitiba"], "2026-08-11")
semantica.ler("F-088", "destinos_frequentes")     # zero chamadas
```

Recuperar isso por similaridade seria caro, impreciso e desnecessário. A pergunta *"para onde o F-088 costuma viajar?"* não é sobre assunto: é uma consulta por chave primária composta.

> É a mesma decisão que a Aula 05 tomou ao separar `Estado` de `mensagens[]`, e que a Aula 06 tomou ao recomendar `==` para identificadores. **A estrutura de armazenamento decorre da forma da consulta**, não do fato de o sistema envolver um LLM.

#### 3.3 Procedural

Como agir. É texto, e entra no *system prompt*:

```python
def como_system_prompt(self) -> str:
    regras = self.regras()
    if not regras:
        return ""
    return "Procedimentos aprendidos em execuções anteriores:\n" + \
           "\n".join(f"- {r}" for r in regras)
```

É a mais valiosa das três e a menos utilizada: representa o agente alterando o próprio comportamento com base no que ocorreu antes.

É também a mais perigosa, e por uma razão estrutural. As outras duas são consultadas seletivamente — um episódio irrelevante simplesmente não é recuperado. Uma regra procedural incorreta aplica-se a **todas** as execuções seguintes, sem que nenhum mecanismo de relevância a filtre. É por isso que a [nota 03](03-escrever-e-ler.md) recomenda revisão humana especificamente para essa categoria.

---

### 4. O que não deve ser armazenado

A quarta categoria é a que a taxonomia usual omite, e a que mais afeta o resultado.

Considere as informações que uma única execução produz:

| Informação | Classificação |
|---|---|
| "a despesa D-4102 foi aprovada em 12/03 por estar dentro do teto" | episódica |
| "F-088 viajou a Lisboa em março e agosto" | semântica |
| "quando o recibo diverge, devolver antes de analisar o mérito" | procedural |
| "a consulta ao histórico levou 240 ms" | **não guardar** |
| "a resposta veio com 1.820 tokens de entrada" | **não guardar** |
| "o agente concluiu no terceiro passo" | **não guardar** |
| "o modelo tentou F-88 antes de acertar F-088" | **não guardar** |

Quatro das sete pertencem à última linha — e numa execução real a proporção é pior, não melhor.

A última entrada da tabela merece discussão, porque é a que gera divergência em sala. A tentativa fracassada de consultar `F-88` é ruído de **uma** execução; a generalização dela — *"identificadores de funcionário têm o formato F seguido de três dígitos"* — é memória procedural legítima. A distinção é entre registrar o incidente e extrair a regra.

> **Memória preenchida com ruído é pior que memória vazia.** O ruído não é apenas inútil: ele **compete por espaço na janela** com o que seria útil — e a [nota 03](03-escrever-e-ler.md), §4, quantifica esse espaço.

---

## Exemplos

### Exemplo 1 — O arquivo morto e a memória, lado a lado

```
  A — O CHECKPOINT: uma execução, inteira
      exec-0a1f, ~250 tokens: objetivo, os 3 passos com argumentos e
      resultados, ferramentas_ativas, termino, historico

      ... e existem 10 destes gravados
      10 execuções · 36 passos · 21.360 tokens
      contexto para incluir tudo: ~5.340 tokens

  B — A MEMÓRIA: as mesmas dez, indexadas
      a MESMA exec-0a1f, agora como fragmento: ~64 tokens
      a memória descartou ~75% do checkpoint, de propósito

      consulta: "refeição de R$ 245,00 por pessoa em Lisboa, com nota"
      [2026-03-28] exec-3b7c  Jantar de R$ 312,00 para três pessoas em
                              Lisboa. Art. 4º §2º combinado com o §3º.
      [2026-08-11] exec-e770  Refeição de R$ 240,00 por pessoa em Lisboa.
                              Art. 4º §2º.
      [2026-03-12] exec-0a1f  Refeição de R$ 84,00 em viagem a Curitiba.
      contexto necessário: ~80 tokens

  C — A MESMA PERGUNTA ÀS DUAS. DUAS VEZES.
      "retome a exec-0a1f onde parou"     checkpoint SIM · memória NÃO
      "o que ajuda na despesa de agora?"  checkpoint NÃO · memória SIM
```

*(Valores ilustrativos, exceto os totais das dez execuções.)*

As três execuções recuperadas são as duas de Lisboa e uma de refeição — exatamente as pertinentes à tarefa corrente. A diferença entre 5.340 e 80 tokens não decorre de compressão: decorre de **seleção**.

E a seção C é o que impede a leitura errada do parágrafo acima. Ela não mostra a memória vencendo: mostra as duas estruturas **falhando**, cada uma na pergunta da outra (§1.2).

### Exemplo 2 — A consulta que combina os três mecanismos

```python
episodica.recuperar("falta de justificativa em corrida",
                    funcionario="F-091", veredito="reprovado", k=3)
```

Três operações distintas, em ordem:

1. `funcionario="F-091"` — filtro por igualdade sobre metadado;
2. `veredito="reprovado"` — filtro por igualdade sobre metadado;
3. `"falta de justificativa em corrida"` — busca por similaridade sobre o que restou.

Sem os filtros, a etapa 3 recuperaria com igual facilidade os episódios de **aprovação** de outros funcionários — pela cegueira de negação e pela cegueira de entidade, simultaneamente. Com eles, a busca por similaridade opera sobre um conjunto já restrito ao que é comparável.

### Exemplo 3 — A regra procedural que veio de um erro

Execução real do agente:

```
passo 0  consultar_historico(F-88)   -> erro: funcionário inexistente
                                        (esperado: F seguido de três dígitos)
passo 1  consultar_historico(F-088)  -> ok
```

Duas informações distintas podem ser extraídas desse par:

| Extração | Classificação | Valor |
|---|---|---|
| "na execução exec-f2a0, o agente tentou F-88 antes de F-088" | episódica, e ruído | nenhum |
| "identificadores de funcionário têm o formato F + 3 dígitos" | **procedural** | evita o erro em toda execução futura |

A segunda é generalização da primeira, e é o único produto do incidente que merece armazenamento. Quem escreve essa generalização — o agente, o código ou um humano — é o assunto da [nota 03](03-escrever-e-ler.md).

---

## Fontes e leituras

SUMERS, T. et al. **Cognitive architectures for language agents** (CoALA). arXiv:2309.02427, 2023. (A origem da divisão em episódica, semântica e procedural adotada na §3.)

IBM. **What is AI agent memory?**. 2025. Disponível em: https://www.ibm.com/think/topics/ai-agent-memory. (A divisão *short-term* / *long-term* da §2, e as três como subtipos da segunda.)

REDIS. **AI agent memory**: building stateful AI systems. 2026. Disponível em: https://redis.io/blog/ai-agent-memory-stateful-systems/. (O *checkpoint* como persistência de estado *thread-scoped*, e o argumento da §2.3.)

PACKER, C. et al. **MemGPT**: towards LLMs as operating systems. arXiv:2310.08560, 2023.

PARK, J. S. et al. **Generative agents**: interactive simulacra of human behavior. arXiv:2304.03442, 2023. (A formulação de *memory stream* mais citada; a taxonomia da §3 dialoga com ela.)

ANTHROPIC. **Effective context engineering for AI agents**. 2025.

**Material da disciplina.** Desta aula, [nota 01](01-o-agente-e-o-estado.md) §3 — o objeto de estado — e §8 — o *checkpoint*. Aula 05, [nota 02 §1](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — o orçamento em quatro moedas; [nota 03 §3](../../aula-05-arquitetura-de-agentes/notas-de-aula/03-context-engineering-dinamica.md) — o que ocupa a janela. Aula 07, [nota 02](../../aula-07-rag-e-documentos/notas-de-aula/02-o-que-o-embedding-nao-ve.md) — as cegueiras que justificam os três metadados. Aula 07 — o mecanismo de indexação, aqui aplicado a outro objeto.
