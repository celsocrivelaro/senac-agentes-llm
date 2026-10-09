# IA Aplicada com LLMs — Aula 11: Evals — Os quatro níveis de avaliação

> **Pré-requisitos:** nota 01 desta aula · Aula 05, [nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — o *trace* · Aula 06, [nota 02](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md) — `recall@k` · Aula 07, [nota 03](../../aula-07-rag-e-documentos/notas-de-aula/03-medir-o-rag-e-a-recuperacao-como-ferramenta.md) — `recall` × fidelidade.
>
> **Código:** [`aula11-evals/`](https://github.com/celsocrivelaro/senac-llm-code/tree/main/aula11-evals) — os scripts `01-tool-unit-tests.py` e `04-trajectory-eval.py` correspondem a esta nota.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Classificar** uma propriedade a medir no nível de avaliação mais barato que a responde.
- **Distinguir** avaliação de trajetória de avaliação de resultado, e dizer o que cada uma revela.
- **Construir** um avaliador de trajetória a partir do *trace* da Aula 05.
- **Reconhecer** o caso do agente que acerta a resposta pelo caminho errado, e explicar por que ele é perigoso.
- **Justificar** a avaliação do estado do mundo em lugar do texto, quando o sistema tem efeito.

---

## Desenvolvimento teórico

### 1. Os quatro níveis

| Nível | O que mede | Como | Custo |
|---|---|---|---|
| **Determinístico** | o que tem gabarito | `==`, expressão regular, validação de schema, `recall@k` | grátis |
| **Trajetória** | o caminho: ferramenta certa, argumento certo, ordem | comparação do *trace* com o esperado | barato |
| **Resultado** | o estado final do mundo | comparação do estado persistido com o alvo | barato |
| **Juiz LLM** | o que não tem gabarito: fidelidade, tom, completude | modelo com rubrica | caro, e precisa ser validado |

A diferença de custo entre a primeira e a última linha é de ordens de grandeza, e não apenas em dinheiro: o nível determinístico é **exato**, e o juiz é **estimado**.

> **A regra que organiza o bloco:** use o nível mais barato que responde à pergunta. Um sistema avaliado inteiramente por juiz é caro, lento e menos confiável do que precisaria ser.

### 2. Determinístico: mais coisas do que parece

A primeira reação de quem avalia sistemas com LLM é supor que quase nada tem gabarito. Na prática, boa parte tem — desde que a pergunta seja bem escolhida.

```python
# O veredito é enumerado: `==` basta.
assert saida["veredito"] in {"aprovado", "reprovado", "pendente"}
assert saida["veredito"] == caso["esperado"]["veredito"]

# A citação é verificável: o artigo existe, e foi de fato recuperado.
assert saida["cita"] in contexto_recuperado

# O formato é contrato (Aula 03): valida-se com schema.
Parecer.model_validate(saida)

# A busca tem gabarito: `recall@k` da Aula 06.
recall = sum(caso["artigo"] in recuperados[:k] for caso in conjunto) / len(conjunto)
```

O trabalho aqui não é técnico — é de **formulação**. "A resposta é boa?" não tem gabarito. "A resposta cita o artigo correto?" tem, e responde à mesma preocupação em quase todos os casos que importam.

### 3. Trajetória: o que o *trace* permite

A Aula 05 (nota 02, §10) construiu o *trace* — a trajetória registrada com ferramenta, argumentos, tokens, custo e motivo de término — e afirmou que ele valia mais que o produto. Esta é a aula que cobra a afirmação.

Com o *trace*, avalia-se **o percurso**, e não apenas a resposta:

```python
esperado = ["consultar_despesa", "consultar_politica", "buscar_regulamento"]
obtido   = [p.ferramenta for p in trace.passos]

cobertura  = len(set(esperado) & set(obtido)) / len(esperado)
supérfluas = [f for f in obtido if f not in esperado]
ordem_ok   = subsequencia(esperado, obtido)
```

Três medidas distintas saem daí, e confundi-las é comum. **Cobertura** diz se o agente consultou o que precisava. **Chamadas supérfluas** dizem se ele gastou à toa. **Ordem** importa quando há dependência — não adianta registrar o parecer antes de consultar a política.

E há a avaliação de **argumento**, que é onde mora a maior parte dos defeitos silenciosos: o agente chamou a ferramenta certa com o identificador errado, obteve um erro, e contornou de um jeito que produziu resposta plausível.

### 4. Resultado: avalie o mundo, não a prosa

Quando o sistema tem efeito — registra, envia, movimenta —, o gabarito não é o texto da resposta. É **o estado do mundo depois da execução**.

O τ-bench (Yao et al., 2024) formaliza isso: ao final da conversa, o estado do banco de dados é comparado com o estado-alvo anotado. Não se avalia o que o agente disse ter feito; avalia-se o que ficou feito.

```python
# ANTES da execução
antes = dict(PARECERES)

rodar(caso["entrada"])

# DEPOIS: o mundo mudou como deveria?
novos = {k: v for k, v in PARECERES.items() if k not in antes}
assert len(novos) == caso["esperado"]["pareceres_criados"]
assert novos and list(novos.values())[0]["veredito"] == caso["esperado"]["veredito"]
```

Essa forma de avaliação captura uma classe de defeito que nenhuma outra captura: **o agente que relata ter feito o que não fez**. A resposta é convincente, a trajetória parece razoável, e o registro não existe.

Ela também captura o oposto, e este é o defeito que a Aula 05 tratou com idempotência: **o agente que fez duas vezes**. Um sistema que cria dois pareceres para a mesma despesa passa em qualquer avaliação de texto.

### 5. Trajetória × resultado: o que cada uma diz

A formulação útil é curta:

> **Resultado diz se funcionou. Trajetória diz por quê.**

Produção precisa das duas, e por razões diferentes. Sem resultado, não se sabe se o sistema serve. Sem trajetória, quando ele deixar de servir, não se sabe onde olhar — e a Aula 12 mostra que essa é exatamente a diferença entre um incidente de duas horas e um de dois dias.

**E há um caso que só a combinação das duas revela: o agente que acerta a resposta pelo caminho errado.**

```
caso: "A despesa D-4613 está dentro da política? Foi em Lisboa."
esperado: aprovado (art. 19 — teto acrescido de 60% em viagem internacional)

  resultado ..... aprovado          CORRETO
  trajetória .... consultar_despesa
                  (não consultou política nem regulamento)

  o agente acertou porque R$ 590 lhe "pareceu" razoável para hospedagem.
```

Este sistema passa na avaliação de resultado e é uma bomba-relógio: funciona no caso avaliado e falha no vizinho, porque não estava consultando nada. Avaliar só o resultado premia o palpite bem-sucedido.

### 6. O que sobra para o juiz

Depois de esgotar os três primeiros níveis, sobram propriedades que genuinamente não têm gabarito:

- **fidelidade** — a resposta é sustentada pelo contexto recuperado, ou o modelo acrescentou? (Aula 07)
- **completude** — respondeu tudo o que foi perguntado?
- **tom e forma** — adequado ao destinatário?
- **qualidade da justificativa** — a razão dada sustenta o veredito?

Estas são reais e importam. O erro comum não é usar juiz para elas — é usar juiz **para tudo**, inclusive para o que `==` resolveria de graça e com exatidão.

### 7. O caso do curso, classificado

| Propriedade | Nível | De qual aula vem |
|---|---|---|
| o artigo certo está entre os `k` recuperados | determinístico (`recall@k`) | 06 |
| o veredito é o esperado | determinístico (`==`) | 05 |
| a citação existe no contexto | determinístico (pertinência) | 07 |
| a saída obedece ao schema | determinístico (validação) | 03 |
| consultou a política antes de decidir | trajetória | 05 |
| o parecer foi registrado uma única vez | resultado | 05 |
| a resposta é fiel ao contexto | juiz | 07 |
| a justificativa sustenta o veredito | juiz | — |

Seis das oito propriedades que importam neste sistema são medidas sem chamar modelo nenhum. **Esse é o resultado prático da regra.**

---

## Exemplos

### Exemplo 1 — A mesma pergunta, nos quatro níveis

```
PERGUNTA: "o sistema está respondendo bem sobre viagem internacional?"

  determinístico ... o art. 19 apareceu entre os 4 recuperados?      12/12
  trajetória ....... buscou o regulamento antes de concluir?          7/12
  resultado ........ o parecer registrado tem o veredito certo?      10/12
  juiz ............. a justificativa cita a exceção e a aplica?       6/12
```

Quatro números, quatro diagnósticos. O primeiro isenta o índice; o segundo acusa o prompt; o quarto mostra que, mesmo quando acerta, o sistema frequentemente não explica por quê.

### Exemplo 2 — Trajetória, com as três medidas separadas

```python
def avaliar_trajetoria(trace, esperado: list[str]) -> dict:
    obtido = [p.ferramenta for p in trace.passos]
    return {
        "cobertura":   len(set(esperado) & set(obtido)) / len(esperado),
        "superfluas":  [f for f in obtido if f not in esperado],
        "ordem_ok":    e_subsequencia(esperado, obtido),
        "argumentos_invalidos": [p for p in trace.passos if p.erro],
    }
```

A última linha é a que costuma revelar mais: chamadas que devolveram erro de domínio e foram contornadas produzem respostas plausíveis com fundamento inexistente.

### Exemplo 3 — Resultado, e a escrita dupla

```
caso C-021 — "Registre o parecer da despesa D-4612."

  texto da resposta ..... "Parecer P-1000 registrado com sucesso."   OK
  estado do mundo ....... PARECERES contém P-1000 e P-1001
                          ambos para D-4612

  VEREDITO: falha. A avaliação de texto não veria isto.
```

O defeito é o da Aula 05 — escrita repetida por retry ou por laço —, e a chave de idempotência é a defesa. A avaliação de resultado é o que **detecta** quando a defesa não está lá.

---

## Fontes e leituras

YAO, S. et al. **τ-bench**: a benchmark for tool-agent-user interaction in real-world domains. arXiv:2406.12045, 2024.

ES, S. et al. **RAGAS**: automated evaluation of retrieval augmented generation. arXiv:2309.15217, 2023.

**Material da disciplina.** Aula 05, [nota 02](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — o *trace* e a idempotência. Aula 06, [nota 02](../../aula-06-embeddings-e-rag/notas-de-aula/02-chunking-e-a-medida-da-busca.md) — `recall@k`. Aula 07, [nota 03](../../aula-07-rag-e-documentos/notas-de-aula/03-medir-o-rag-e-a-recuperacao-como-ferramenta.md) — a separação entre recuperação e geração.
