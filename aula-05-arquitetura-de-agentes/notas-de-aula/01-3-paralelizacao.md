# IA Aplicada com LLMs — Aula 05: Padrões de arquitetura — Paralelização — sectioning e voting

> **Onde este padrão se encaixa.** Não é arquitetura concorrente do Router: é **padrão de execução**, aplicável dentro de outra arquitetura. O ganho é tempo, não dinheiro.
>
> Parte da série de padrões da [nota 01](01-0-padroes-de-arquitetura.md). Anterior: [Router](01-2-router.md). Próximo: [orquestrador](01-4-orquestrador.md).

---

## O padrão

**Antes das duas formas, uma correção de categoria.** A paralelização **não é uma arquitetura concorrente do *Router***. É um **padrão de execução**, aplicável *dentro* de uma arquitetura — inclusive dentro de uma que já tem um *Router* na frente.

A diferença é o que cada um faz com os caminhos disponíveis:

| | O que faz com os caminhos |
|---|---|
| ***Router*** | escolhe **um** e **descarta os outros**: se a mensagem é reclamação, a rota de consulta nem é executada |
| **Paralelização** | executa **vários ao mesmo tempo** e combina o que voltar |

Os dois se compõem naturalmente, e a composição mais comum é esta: um *Router* decide **se** o caso merece tratamento caro e, só nos que merecem, dispara três análises em paralelo. O §5 mostra uma composição parecida com os padrões do exercício.

**E o ganho principal da paralelização não é dinheiro — é tempo.** Três chamadas em paralelo custam os mesmos tokens que três em sequência; o que muda é o tempo de parede, que cai para o da mais lenta em vez da soma das três. Quem paraleliza esperando economizar tokens vai se decepcionar.

### Sectioning — divisão e conquista

Subtarefas independentes executadas simultaneamente, e um **agregador** que consolida. As seções são conhecidas: constam do código.

```
                 ┌──> [ LLM: seção 1 ] ──┐
   entrada ──────┼──> [ LLM: seção 2 ] ──┼──> [ AGREGADOR ] ──> saída
                 └──> [ LLM: seção 3 ] ──┘
```

O agregador é um componente, não um `"\n".join()`. No laboratório desta aula ele aparece com o nome de **sintetizador**, e é ele que transforma três respostas soltas num parecer só.

**O exemplo do laboratório**, sobre a carteira de pedidos da transportadora:

```
entrada: os 5 pedidos em aberto

  ├─ seção "atrasos"    -> quais já passaram da previsão, e há quantos dias
  ├─ seção "carteira"   -> quantos pedidos por transportadora
  └─ seção "risco"      -> quais vencem nos próximos 5 dias
                                          (as três ao mesmo tempo)
  AGREGADOR -> "A carteira tem 3 pedidos em trânsito, 2 deles vencidos.
                A RápidoLog concentra o atraso: 2 dos 3 casos."
```

As três seções **não se consultam**. Nenhuma precisa do resultado da outra para existir — e é exatamente esse teste que autoriza o padrão. Se a seção "risco" precisasse saber o que "atrasos" concluiu, elas seriam uma cadeia, não um paralelo.

A forma que o mercado reconhece é a **pesquisa de concorrente**: *"analise a presença digital da Empresa X"* dispara, em paralelo, um agente que busca notícias, outro que lê a tabela de preços e um terceiro que varre as redes sociais — com o agregador escrevendo o relatório. É a mesma estrutura, com ferramentas no lugar de seções de prompt.

### Voting — consenso

**Voting.** A mesma tarefa executada N vezes, com decisão por maioria. Corresponde à *self-consistency* (WANG et al., 2022), apresentada na Aula 03 como técnica de prompt e aqui promovida a padrão de arquitetura. A diferença está no local de controle: naquele caso, o parâmetro `n` era passado na chamada e o provedor gerava N amostras do mesmo prompt; aqui, as N chamadas são orquestradas pelo código, o que permite variar prompt, modelo ou temperatura entre elas.

```
                 ┌──> [ LLM ] ──┐
   entrada ──────┼──> [ LLM ] ──┼──> voto majoritário ──> saída
                 └──> [ LLM ] ──┘
```

```python
def votar(item, n=3):
    vereditos = [classificar(item) for _ in range(n)]      # ou em paralelo
    rota, votos = Counter(v["rota"] for v in vereditos).most_common(1)[0]
    if votos < n // 2 + 1:
        return {"rota": "nenhuma", "motivo": "sem maioria"}   # divergência é sinal
    return {"rota": rota, "confianca": votos / n}
```

O detalhe habitualmente descartado: **a divergência é informação**. Três vereditos distintos não indicam que se deva escolher um deles; indicam que o caso é difícil e, provavelmente, que cabe encaminhamento a humano.

**Um exemplo completo**, sobre a mensagem ambígua do lote da triagem — *"Consta que o 77310 foi entregue mas eu não recebi nada"*:

```
  chamada 1 (temp 0.0)  -> reclamacao       "cliente afirma não ter recebido"
  chamada 2 (temp 0.7)  -> reclamacao       "divergência com o registro"
  chamada 3 (modelo B)  -> consulta_status  "o cliente quer saber onde está"

  maioria: reclamacao, 2 de 3   ->  confiança 0,67
```

Duas leituras possíveis desse resultado, e a escolha entre elas é de projeto:

- **aceitar a maioria** e seguir como reclamação, registrando a confiança;
- **tratar 2/3 como empate técnico** e mandar para humano, porque um dos três leu a mensagem de um jeito radicalmente diferente.

Num domínio em que abrir chamado indevido custa pouco, a primeira. Numa auditoria de código, num parecer jurídico ou num apoio a diagnóstico — os casos em que o *voting* de fato se paga —, a segunda: ali o consenso não é desempate, é **evidência de que a resposta é segura**, e a falta dele é o achado mais valioso da execução.

Repare também no que varia entre as três chamadas: **temperatura e modelo**. É o que a orquestração em código permite e o parâmetro `n` do provedor não — lá as N amostras saem do mesmo prompt no mesmo modelo.

| Usar quando | **Não** usar quando |
|---|---|
| *sectioning*: a tarefa se decompõe e as partes são independentes; latência é requisito | *sectioning*: as partes dependem umas das outras — a concatenação produz trechos contraditórios |
| *voting*: o custo do erro supera o de 3 chamadas e a resposta é discreta (aprovar/reprovar, classificar) | *voting*: tarefa aberta ("redigir um resumo") — não há maioria entre três textos distintos, e o custo triplica sem contrapartida |

---

---

