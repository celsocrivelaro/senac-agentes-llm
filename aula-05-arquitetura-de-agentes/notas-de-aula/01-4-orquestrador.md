# IA Aplicada com LLMs — Aula 05: Padrões de arquitetura — Orquestrador-trabalhador

> **Onde este padrão se encaixa.** **O modelo decide quantas subtarefas existirão** — e por isso este é o primeiro padrão da série que exige teto. O custo deixa de ser calculável antes de rodar.
>
> Parte da série de padrões da [nota 01](01-0-padroes-de-arquitetura.md). Anterior: [paralelização](01-3-paralelizacao.md). Próximo: [avaliador-otimizador](01-5-avaliador-otimizador.md).

---

## O padrão

**Definição.** Um modelo **determina quais são as subtarefas**; **sub-agentes** trabalhadores as executam; um sintetizador consolida os resultados.

O padrão é a combinação dos dois anteriores, com a decisão transferida para o modelo:

| | Quem decide o fluxo | Quantos caminhos executam |
|---|---|---|
| ***Router*** | o modelo **classifica**, entre rotas que você escreveu | **um**; os outros são descartados |
| **Paralelização** | ninguém — as seções estão no código | **N fixo**, conhecido antes de rodar |
| **Orquestrador** | **o modelo, em tempo de execução** | **N desconhecido** até o plano existir |

### As três fases

```
   ┌─ 1. DECOMPOSIÇÃO ─────────────────────────────────────────────┐
   │   entrada ──> [ orquestrador ]  decide QUANTAS e QUAIS        │
   └───────────────────────┬───────────────────────────────────────┘
                           │  plano: N subtarefas  (N só existe agora)
   ┌─ 2. DELEGAÇÃO ────────┼───────────────────────────────────────┐
   │            ┌──> [ sub-agente 1 ] ──┐                          │
   │            ├──> [ sub-agente 2 ] ──┤   paralelo OU sequencial │
   │            └──> [ sub-agente N ] ──┘   (ver §1.2)             │
   └───────────────────────┬───────────────────────────────────────┘
   ┌─ 3. SÍNTESE ──────────┼───────────────────────────────────────┐
   │            [ sintetizador ] ──> saída                         │
   │            consolida, e resolve contradições entre eles       │
   └───────────────────────────────────────────────────────────────┘
```

A terceira fase é a que se subestima: o sintetizador **não concatena** — ele recebe N respostas que podem se contradizer, e alguém precisa decidir qual vale. É o mesmo problema do *voting* (nota 02, §3b), com a diferença de que ali as respostas eram da mesma pergunta e aqui são de perguntas diferentes.

### Paralelo ou sequencial, e o critério

A fase 2 admite as duas formas, e a escolha não é de desempenho — é de **dependência**:

| Se as subtarefas | Execução | Porque |
|---|---|---|
| são independentes | **paralela** | o tempo de parede cai para o da mais lenta |
| uma consome a saída da outra | **sequencial** | não há como começar a segunda antes da primeira |

O esqueleto abaixo executa em sequência, que é o caso mais simples e o do laboratório. Paralelizar é trocar a compreensão de lista por execução concorrente — e só é lícito depois de verificar a primeira linha da tabela.

```python
def processar_lote(lote: list) -> dict:
    plano = chamar(PROMPT_ORQUESTRADOR, lote, schema=SCHEMA_PLANO)
    # plano["subtarefas"] só existe agora — não estava no código

    if len(plano["subtarefas"]) > MAX_SUBTAREFAS:      # orçamento, já aqui
        raise PlanoGrandeDemais(len(plano["subtarefas"]))

    resultados = [executar_subtarefa(s) for s in plano["subtarefas"]]
    return chamar(PROMPT_SINTESE, resultados, schema=SCHEMA_PARECER)
```

A distinção em relação ao *sectioning* é única e decisiva: lá as seções constam do código; aqui são determinadas em execução, e sua quantidade é desconhecida de antemão. Daí a verificação explícita: este é o primeiro padrão da nota com autonomia real e, por consequência, o primeiro que exige teto. **Um orquestrador sem teto é uma conta aberta assinada por um modelo.**

### Por que sub-agentes, e não um agente só com tudo

O ganho que justifica o padrão não é de organização — é de **contexto**.

Cada sub-agente recebe um *system prompt* curto, focado na própria especialidade, e **apenas as ferramentas que ela exige**. Um agente único que fizesse as N subtarefas carregaria, em toda volta, as declarações de todas as ferramentas e o histórico de todas as análises anteriores — e a nota 06 (§1) mostra o que isso custa: o consumo acumulado cresce com o quadrado dos passos, e a qualidade cai antes de a janela acabar.

> **É a mesma ideia que a nota 06 (§3) trata como *sub-agente é isolamento de contexto*, vista do outro lado.** Lá o foco é o que se ganha: o sub-agente gasta a própria janela e devolve um resumo curto. Aqui o foco é quem decide quantos existirão.
>
> E o preço é o mesmo, dito lá sem eufemismo: **o que o sub-agente não incluiu no resultado, o orquestrador nunca saberá.** A perda é irrecuperável, e não aparece como erro — aparece como conclusão confiante sobre informação incompleta.

Duas consequências práticas:

**A especialização é por requisito de contexto, não por cargo.** Dividir em "pesquisador / analista / redator" só se justifica se cada um precisar de um conjunto de informação diferente. É o critério que a **Aula 12** desenvolve, e o erro que ela nomeia: organograma não é arquitetura.

**Acrescentar um sub-agente é barato; acrescentar um domínio não é.** Um especialista novo entra sem reescrever o orquestrador — mas cada um que entra é mais uma chance de o resultado voltar incompleto, e mais uma linha na conta do teto.

| Usar quando | **Não** usar quando |
|---|---|
| a decomposição depende do conteúdo: alteração que afeta número imprevisível de arquivos, pesquisa com número imprevisível de consultas | as subtarefas são enumeráveis — nesse caso trata-se de *sectioning*, mais barato e sem surpresa |

**A forma que o mercado reconhece** é a criação de conteúdo: *"monte a campanha do produto X"* dispara um redator, um especialista em imagem e um de redes sociais, e o sintetizador entrega o plano. É a mesma estrutura do laboratório — lá o orquestrador decide quais análises a carteira de pedidos daquele dia comporta —, e a diferença entre os dois exemplos é só o domínio.

Implementação em [`02-orquestrador-trabalhador.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/02-orquestrador-trabalhador.py).

---


---

## Exemplo

### Sectioning ou orquestrador: o teste do `for`

```python
# SECTIONING — as partes existem antes de qualquer chamada
partes = dividir_por_capitulo(documento)              # código puro
resumos = [resumir(p) for p in partes]                # N conhecido

# ORQUESTRADOR — as partes só existem após uma chamada
plano = chamar(PROMPT_ORQUESTRADOR, documento)        # o modelo decide
for s in plano["subtarefas"]:                         # N desconhecido
    ...                                               # <- exige teto
```

O teste operacional: **é possível escrever o `for` sem chamar o modelo antes?** Em caso afirmativo, trata-se de *sectioning* — a opção preferida sempre que aplicável.

---

