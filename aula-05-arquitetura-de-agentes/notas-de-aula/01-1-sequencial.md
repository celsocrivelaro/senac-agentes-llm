# IA Aplicada com LLMs — Aula 05: Padrões de arquitetura — Sequencial (prompt chaining) com portão

> **Onde este padrão se encaixa.** O caminho está inteiro no código: as etapas são conhecidas, e o número de chamadas também.
>
> Parte da série de padrões da [nota 01](01-0-padroes-de-arquitetura.md).  Próximo: [Router](01-2-router.md).

---

## O padrão

**Definição.** Sequência fixa de chamadas em que a saída de uma alimenta a seguinte, com **validação determinística entre elas** — o *portão*.

```
   entrada ──> [ LLM 1 ] ──> portão ──> [ LLM 2 ] ──> saída
                              │
                              └── falhou ──> tratamento (não segue adiante)
```

```python
def extrair_e_formatar(texto: str) -> dict:
    bruto = chamar(PROMPT_EXTRACAO, texto, schema=SCHEMA_EXTRACAO)

    if not bruto.get("valor") or bruto["valor"] <= 0:   # O PORTÃO:
        raise ValorInvalido(bruto)                      # código, não modelo
    if bruto["data"] > date.today():
        raise DataFutura(bruto)

    return chamar(PROMPT_PARECER, bruto, schema=SCHEMA_PARECER)
```

O *chaining* foi apresentado na Aula 03 como técnica de prompt; o que se acrescenta aqui é o portão. Regra expressável em Python é implementada em Python: sai mais barata, é mais confiável e não alucina.

| Usar quando | **Não** usar quando |
|---|---|
| as etapas são conhecidas e cada uma produz melhor resultado isolada: extrair → validar → formatar, traduzir → revisar | as etapas não são independentes — cada elo é uma oportunidade de perda de informação |
| a falha intermediária deve interromper o processo | o número de etapas depende da entrada (nesse caso, trata-se de agente) |

---


## O portão em detalhe

As validações de um portão são de **duas naturezas**, e a segunda é a que justifica o padrão existir:

| Natureza | A pergunta | Exemplo |
|---|---|---|
| **Forma** | o campo tem o formato certo? | o número tem cinco dígitos? |
| **Fato** | o que o cliente disse é **verdade**? | esse pedido existe na base? a data já passou? |

O modelo consegue simular a primeira e **não tem como verificar a segunda**: ele não conhece a base de pedidos. É por isso que a validação de fato é código, e é ela que impede o modo de falha mais caro deste padrão — **a etapa 2 redigindo uma resposta impecável sobre um pedido que não existe.**

> **O portão não melhora a resposta. Ele impede que uma resposta errada seja produzida.** São coisas diferentes, e a segunda vale mais: o custo de parar no portão é uma chamada; o de não parar é duas chamadas e uma resposta errada entregue ao cliente.

E repare no que a etapa 2 recebe: **os dados do sistema, não os do cliente**. O que o cliente escreveu serviu para localizar o pedido, e só.

Implementação em [`00-sequencial.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/00-sequencial.py), que roda cinco mensagens — três passam, duas são barradas por motivos diferentes — e imprime, para cada barrada, **o que a etapa 2 teria escrito sem o portão**.
