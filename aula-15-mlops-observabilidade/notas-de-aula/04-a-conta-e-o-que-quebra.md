# IA Aplicada com LLMs — Aula 15: Produção — A conta, e o que quebra

---

## Objetivos de aprendizagem

Ao final desta nota, espera-se que o aluno seja capaz de:

- **Consolidar** as medições de consumo das aulas anteriores em uma conta única por execução.
- **Identificar** onde o custo mora, lendo o painel em vez de supor.
- **Aplicar** as alavancas de redução em ordem de retorno, medindo o efeito de cada uma.
- **Fechar** a conferência da promessa da Parte 1 do trabalho.
- **Listar** as dívidas que vencem em produção e o que cada uma exige, e aplicar a suíte construída na aula de *evals* às que dependem dela.

---

## Desenvolvimento teórico

### 1. A cobrança das medições em aberto

Três aulas mediram consumo e nenhuma converteu em dinheiro:

| Aula | O que foi medido | O que ficou em aberto |
|---|---|---|
| *embeddings* e RAG | a assimetria entre embutir o corpus e embutir a pergunta | quanto custa reconstruir o índice a cada reinício |
| *frameworks* e orquestração | as duas versões do sistema, sem colunas de token | o consumo comparado em execuções reais |
| MCP | a ocupação de janela das declarações | quanto essa ocupação custa por execução |

Este bloco fecha as três. E é o primeiro momento do curso em que se mede **quanto o próprio sistema custa por execução**.

### 2. Onde o custo mora

Lido no painel, e não suposto. Em sistemas de agente, duas fontes dominam quase sempre:

**O reenvio do histórico.** A aula de arquitetura de agentes fez a conta: a cada volta, a trajetória inteira é reenviada, e o acumulado cresce com o quadrado do número de passos. Trinta passos consomem mais de 250 mil tokens acumulados.

**As declarações de ferramenta.** A aula de MCP mediu: dezenas de milhares de tokens em toda volta, antes de qualquer mensagem, quando há vários servidores conectados.

A terceira fonte, frequente e menos óbvia, é o **retorno de ferramenta não encurtado** — a tática zero de arquitetura de agentes, que aquela aula afirmou valer mais que as quatro táticas de compressão juntas.

### 3. As alavancas, em ordem de retorno

**Roteamento de modelo.** Modelo barato para o que é fácil, forte para o que é difícil. Trabalhos de roteamento relatam preservar cerca de **95% da qualidade** do modelo forte enviando a ele entre **14% e 26%** das chamadas (Ong et al., 2024).

É a regra da aula de escolha de modelos — um modelo por trabalho — convertida em dinheiro. E ela tem um pré-requisito, que a aula de *evals* forneceu: **sem suíte, não há como saber se o modelo barato basta** para uma classe de casos. A suíte, rodada com o modelo barato e lida por fatia, decide quais classes podem ser roteadas; o custo é medido aqui, e a qualidade, lá.

**Processamento em lote.** Para o que não é interativo — reindexação, avaliação em massa, relatórios noturnos —, provedores oferecem descontos substanciais em troca de latência. A pergunta que decide é simples: *alguém está esperando por isto agora?*

**As táticas de arquitetura de agentes.** Encurtar o retorno das ferramentas, *tool clearing*, compaction. A tática zero continua sendo a de maior retorno, e continua sendo a menos aplicada.

| Alavanca | Redução típica | Preço |
|---|---|---|
| roteamento | 75–85% no tráfego roteado | a suíte de *evals* precisa validar o modelo barato por fatia |
| lote | grande, no que não é interativo | latência |
| tática zero | variável, frequentemente a maior | trabalho de projeto de ferramenta |

### 4. A conferência da promessa

Este é o fecho do arco do trabalho, e é a parte da aula com mais consequência fora da técnica.

A Parte 1 exigiu **a justificativa de negócio**: por que agente e não software comum, que ganho se espera — com linha de base medida, conta à vista e denominador. Naquele momento, o número era uma promessa.

Agora existe painel. A conferência consiste em pôr os dois lado a lado:

```
  prometido na Parte 1 ....... redução de 80% no tempo de análise
                              linha de base: 14 min por despesa
  medido na Parte 3 .......... 4,2 min por despesa   -> redução de 70%
                              custo por execução: R$ 0,041
                              execuções/mês: 1.900   -> R$ 78/mês
```

E a lição que a disciplina quer deixar:

> **Prometer 80% e entregar 30%, com a conta à vista, é resultado aceitável e honesto** — e é o que acontece na maioria dos projetos reais. **Prometer "mais eficiência" e não ter como conferir, não é.**

O erro de estimativa é tolerável; a promessa sem meio de conferência, não.

### 5. As dívidas que vencem em produção

São cinco, todas declaradas antes. As duas primeiras se resolvem com a suíte construída na aula de *evals*; as três seguintes, com o que esta aula já tem.

**Prompt registry** (*prompt engineering*). A nota sobre o prompt como código estabeleceu que mudar prompt sem *deploy* é a vantagem do registro e é a mesma propriedade que faz a mudança escapar da revisão. Em produção, o efeito é concreto: a imagem fica igual, a *tag* fica igual, o comportamento muda, e o painel da nota sobre observabilidade não mostra a diferença, porque o corte por versão não separa o antes do depois. A política que resolve — **o registro é fonte, e a mudança passa pela suíte antes de valer** — usa a suíte de *evals*, e a versão do prompt entra no carimbo ao lado da *tag* da imagem, para que o corte por versão volte a separar o antes do depois.

**Depreciação de modelo** (escolha de modelos). O provedor anuncia, dá prazo, e depois a chamada falha. Migrar exige comparar o modelo novo com o atual nos mesmos casos e decidir com número; **sem esse instrumento, migrar é apostar**, e o prazo não negocia. O instrumento veio da aula de *evals*: a suíte de regressão, rodada com o modelo novo e lida por fatia, com o mesmo limiar declarado, transforma a migração em operação de engenharia.

**Memória sem teto** (memória). O crescimento é monotônico, e alguém paga o armazenamento e a latência de consulta. Exige política de retenção — o que guardar, por quanto tempo, e o que se apaga. A aula de segurança acrescentará uma razão para que a operação de esquecimento exista.

**Versão do índice** (*embeddings* e RAG). Reindexar com outro modelo de embedding invalida a comparação com as medições anteriores. É o carimbo, agora com consequência operacional: um índice reconstruído com modelo diferente muda o `recall` sem que ninguém tenha mudado o código.

**Custo que muda sem aviso.** O provedor reajusta preços, e o modelo novo pode ser mais caro por token e mais barato por tarefa — ou o contrário. A conta precisa ser refeita, não herdada.

---

## Exemplos

### Exemplo 1 — As três contas, fechadas

```
  UMA EXECUÇÃO DO AGENTE (8 passos, 2 servidores MCP conectados)

    declarações de ferramenta (MCP) .......  6.400 tk    38%
    reenvio do histórico (arquitetura) ....  7.900 tk    47%
    retornos de ferramenta ................  2.100 tk    12%
    resposta final ...........................  400 tk     2%
    ────────────────────────────────────────────────────────
    total ................................. 16.800 tk
    custo ................................. R$ 0,041

  E as medições que estavam em aberto:
    reconstruir o índice (embeddings) ..... R$ 0,12 por reinício
```

### Exemplo 2 — A conferência, com a diferença explicada

```
  PROMETIDO (Parte 1)      redução de 80% no tempo de análise
  ENTREGUE  (Parte 3)      redução de 70%

  a diferença, explicada:
    - 12% das despesas caem em exceção e exigem revisão humana
      (não estavam na linha de base)
    - a partida a frio acrescenta 40 s às primeiras execuções do dia

  custo de operação: R$ 78/mês para 1.900 execuções
  economia estimada: 168 h/mês de analista
```

Um resultado de 70% com a conta explícita é preferível a um de 80% sem ela, porque só o primeiro é conferível.

---

## Fontes e leituras

ONG, I. et al. **RouteLLM**: learning to route LLMs with preference data. arXiv:2406.18665, 2024.

**Material da disciplina.** [Escolhendo um modelo](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — depreciação e plano de migração; [Custo, latência e a decisão](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/04-custo-latencia-e-decisao.md) — a conta por chamada. [O prompt como código](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) — o *prompt registry*. [*Context engineering* dinâmica](../../aula-05-arquitetura-de-agentes/notas-de-aula/03-context-engineering-dinamica.md), em arquitetura de agentes — o reenvio do histórico e a tática zero. [O preço do protocolo](../../aula-10-mcp/notas-de-aula/03-o-preco-do-protocolo.md), em MCP — a ocupação das declarações. [O juiz, e a suíte](../../aula-11-avaliacao-agentes/notas-de-aula/03-o-juiz-e-a-suite.md), em *evals* — a suíte, sem a qual não se migra nem se roteia com segurança.
