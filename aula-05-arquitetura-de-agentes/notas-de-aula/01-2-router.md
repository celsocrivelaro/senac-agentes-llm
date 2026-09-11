# IA Aplicada com LLMs — Aula 05: Padrões de arquitetura — Router (Roteador)

> **Onde este padrão se encaixa.** O modelo **classifica**; o código **despacha**. Uma chamada de classificação mais a rota escolhida — custo calculável antes de rodar.
>
> Parte da série de padrões da [nota 01](01-0-padroes-de-arquitetura.md). Anterior: [prompt chaining](01-1-sequencial.md). Próximo: [paralelização](01-3-paralelizacao.md).

---

## O padrão

**Definição.** O ***Router*** é o componente que **analisa a entrada e decide qual caminho, modelo ou ferramenta processa aquela requisição**. Em uma frase operacional: **o modelo classifica; o código despacha.**

É um ponto de triagem. Em vez de mandar toda pergunta para um modelo grande, ou de tentar fazer um único prompt resolver tudo, o *Router* direciona cada entrada para o recurso adequado a ela.

```
                    ┌──> rota A: código puro (sem LLM)
   entrada ──> [ classificador ] ──> rota B: outro prompt / outro modelo
                    │              └──> rota C: humano
                    └──> nenhuma das anteriores ──> fila de revisão
```

**Por que ele é fundamental na arquitetura.** Três razões, e a terceira só se paga por inteiro nas aulas seguintes:

| Razão | O que muda |
|---|---|
| **Eficiência** | a entrada simples vai para o modelo pequeno — ou para código, sem modelo nenhum; só a difícil chega ao modelo grande. É a comparação que a Aula 02 fez entre `mistral-small` e `mistral-large`, agora decidida em tempo de execução |
| **Especialização** | com mais de um agente especializado, é o *Router* que decide qual deles assume. É o embrião da coordenação da Aula 12 |
| **Uso de ferramenta** | identifica se a requisição precisa de consulta a banco, de busca, ou de nenhuma das duas — o que a Aula 09 retoma quando as ferramentas passam a vir de terceiros |

### As três formas de rotear

O classificador do desenho acima não precisa ser um LLM. São três implementações, em ordem crescente de custo:

| Forma | Como decide | Preço |
|---|---|---|
| **Regra / heurística** | palavra-chave, expressão regular, metadado do contexto | zero chamadas. Não alucina, e não generaliza |
| **Embedding / similaridade** | converte a entrada em vetor e compara com exemplos de cada categoria | uma chamada barata, sem geração de texto — **é a Aula 06** |
| **Semântico (LLM)** | um modelo leve devolve a classificação, como `{"rota": "reclamacao"}` | uma chamada de geração. Lida com o caso que as duas anteriores não previram |

**Elas se combinam, e é assim que o laboratório desta aula funciona:** a regra resolve a consulta de status pura, e só o que sobra vai ao modelo. A forma do meio ainda não está disponível ao aluno — ela depende de embeddings, que é a aula seguinte, e é por isso que aparece aqui nomeada e não implementada.

Tecnicamente, a terceira forma é **saída estruturada com enumeração** — o mecanismo da Aula 02 (nota 02, §7) determinando o fluxo do programa:

```python
ROTAS = ["dentro_da_politica", "ambiguo", "acima_da_alcada", "nenhuma"]

def processar(item):
    r = classificar(item, schema_com_enum(ROTAS))   # 1 chamada, barata
    if r["rota"] == "dentro_da_politica":
        return aprovar_por_regra(item)              # ZERO chamadas de LLM
    if r["rota"] == "ambiguo":
        return agente_analisa(item)                 # caro, e raro
    if r["rota"] == "acima_da_alcada":
        return encaminhar_humano(item, r["justificativa"])
    return fila_de_revisao(item)                    # "nenhuma": não inferir
```

### O mesmo *Router*, três entradas do domínio da aula

As três mensagens abaixo são do lote que o laboratório processa, e mostram as três faixas de tratamento:

```
 "Qual o status do pedido 48219?"
    -> intenção de consulta + número de pedido
    -> REGRA, em código. Zero chamadas de LLM, zero risco de alucinação

 "A caixa do 55870 chegou rasgada e o produto está trincado"
    -> reclamação: exige interpretação, e abre chamado
    -> MODELO, com a rota `reclamacao`

 "Consta que o 77310 foi entregue mas eu não recebi nada.
  Falei com o porteiro e ele também não viu."
    -> divergência entre o que o cliente diz e o que o sistema registra
    -> MODELO, e é o caso que justifica o agente da nota 02

 "meu pedido não chegou"
    -> falta o número. Não há rota aplicável
    -> `nenhuma` -> fila de revisão
```

A primeira e a última são as instrutivas. A primeira é a maioria do lote e **não deveria consumir modelo nenhum**; a última é a que revela se o *Router* tem rota de escape.

Dois pontos de engenharia distinguem um *Router* adequado de um inadequado:

- **A rota `nenhuma` é obrigatória.** Sem ela, o modelo é forçado a escolher entre opções que não se aplicam, e escolhe com alta confiança declarada. A rota de escape converte erro silencioso em fila de revisão.
- **A rota de maior valor é a que não chama o modelo.** Se 70% dos itens são resolvidos por `aprovar_por_regra`, o custo do sistema cai 70% sem perda de qualidade. É a otimização de maior retorno desta aula.

| Usar quando | **Não** usar quando |
|---|---|
| entradas heterogêneas exigem tratamentos distintos; triagem | as rotas executam quase a mesma operação — nesse caso, unificá-las num prompt |
| convém selecionar modelo pequeno para o caso fácil e grande para o difícil | a classificação é tão confiável em `if` quanto no modelo |

Implementação em [`01-router.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/01-router.py), que reporta quantos itens do lote foram resolvidos sem chamada ao modelo.

---


---

## Exemplo

### A rota de escape, presente e ausente

Item: *"Uber R\$ 340,00 — funcionário de outra filial, sem centro de custo."* Nenhuma das três rotas previstas se aplica.

```python
# Sem escape: enum = ["dentro_da_politica", "ambiguo", "acima_da_alcada"]
# -> {"rota": "ambiguo", "justificativa": "requer análise"}   # foi para o agente
#    o agente consumiu 6 passos e concluiu "não foi possível determinar"

# Com escape: enum = [..., "nenhuma"]
# -> {"rota": "nenhuma", "justificativa": "centro de custo ausente"}
#    foi para a fila de revisão em 1 chamada
```

A mesma incerteza produz destinos distintos. Sem a rota de escape, a incerteza converte-se em **gasto**; com ela, converte-se em **fila**. O modelo não errou nas duas execuções: na primeira, não dispunha de alternativa correta.

