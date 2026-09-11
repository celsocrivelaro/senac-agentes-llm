# IA Aplicada com LLMs — Aula 05: Padrões de arquitetura — Avaliador-otimizador (reflexão)

> **Onde este padrão se encaixa.** Quem decide o número de chamadas é **o avaliador**, a cada rodada. Segundo e último padrão que exige teto.
>
> Parte da série de padrões da [nota 01](01-0-padroes-de-arquitetura.md). Anterior: [orquestrador](01-4-orquestrador.md). Fecho: [qual padrão usar](01-6-qual-padrao-usar.md).

---

## O padrão

**Definição.** Laço curto de crítica: gerar → avaliar contra critério explícito → revisar.

```
   entrada ──> [ gerador ] ──> candidato ──> [ avaliador ]
                    ↑                             │
                    └───── crítica ───────────────┤ reprovou
                                                  └─ aprovou ──> saída
```

```python
def gerar_com_revisao(entrada, max_rodadas=3):
    critica = None
    for rodada in range(max_rodadas):
        candidato = chamar(PROMPT_GERADOR, entrada, critica=critica)
        aval = chamar(PROMPT_AVALIADOR, candidato, schema=SCHEMA_AVALIACAO)
        if aval["aprovado"]:
            return candidato, "aprovado", rodada + 1
        critica = aval["o_que_corrigir"]
    return candidato, "teto_de_rodadas", max_rodadas    # melhor esforço, sinalizado
```

Duas armadilhas ocorrem sistematicamente:

- **Sem critério escrito, o avaliador produz elogio.** O critério precisa ser uma lista verificável — *cita a regra aplicada? menciona o valor exato? conclui?* — e o schema da avaliação deve forçar resposta item a item. Um retorno `{"nota": 8}` não é acionável; `{"cita_regra": false, "o_que_corrigir": "..."}` é.
- **Sem teto de rodadas, o par oscila.** O avaliador exige A, o gerador entrega A e perde B, o avaliador exige B. Três rodadas costumam bastar; o essencial é que o retorno **declare** a saída por teto. Melhor esforço apresentado como aprovação é pior que reprovação explícita.

### Quem avalia: três modos, em ordem de preferência

O esqueleto acima usa um LLM como avaliador, e isso é o caso **intermediário** — não o padrão. Há três modos, e a ordem importa:

| Modo | Como | Custo | Quando |
|---|---|---|---|
| **Determinístico** | teste, *linter*, validação de schema, regra em código | **zero chamadas** | o critério é **executável** |
| **LLM como juiz** | rubrica explícita, saída item a item | 1 chamada por rodada | o critério existe, mas não é executável |
| **Humano no laço** | uma pessoa aprova, rejeita ou corrige | trabalho humano | o erro é caro ou irreversível |

> **É a tática zero deste padrão, e a aula já a aplicou duas vezes.** No *Router*, a rota de maior valor é a que não chama o modelo. Na context engineering, encurta-se o retorno antes de comprimir. Aqui: **se o critério é executável, o avaliador é `pytest`, não um LLM.**

O caso mais claro é o agente de programação:

```
   [ gerador ] ──> código ──> $ pytest
                                 │
                                 ├─ passou ──────────────> saída
                                 └─ falhou
                                      │
                                      └─> a saída do terminal volta
                                          ao gerador COMO CRÍTICA
```

Nenhuma chamada de modelo foi gasta na avaliação, e o *feedback* é exato em vez de aproximado — é o erro real, com a linha e a exceção. É a mesma lição da [nota 03](03-confiabilidade.md) §4: **o retorno de erro é prompt**, e um erro bem escrito ensina o modelo a se corrigir.

**O terceiro modo já tem nome nesta disciplina.** O humano que aprova não é um caso à parte: é o `Termino.HUMANO` da [nota 03](03-confiabilidade.md) §2 — uma das quatro formas de terminar, com o estado gravado e a decisão devolvida ao chamador. A **Aula 13** converte isso de conveniência em requisito, porque a aprovação pode chegar horas depois e o sistema precisa sobreviver à espera.

### O que o laço NÃO faz

Atribui-se com frequência a este padrão a redução de alucinação. **A atribuição é imprecisa, e a imprecisão é cara.**

Um avaliador sem acesso a fonte externa não verifica fato algum: ele verifica se a resposta é **internamente consistente** e se atende ao formato pedido. Um candidato que inventa um artigo de política inexistente, e o cita de modo coerente em três rodadas, é **aprovado** — e sai mais convincente do que entrou.

> O que reduz alucinação é o **critério verificável**, não o laço. `cita_regra` só vale alguma coisa se alguém conferir a regra citada **contra o regulamento**; caso contrário, mede se o texto menciona algo que pareça uma regra.

É o mesmo defeito que a Aula 07 vai tratar sob outro nome — citação plausível que não está na fonte — e a razão de a recuperação existir.

A formalização do padrão é o Reflexion (SHINN et al., 2023), que acrescenta a persistência da crítica em memória para realimentar as tentativas seguintes — função do parâmetro `critica` no esqueleto acima.

| Usar quando | **Não** usar quando |
|---|---|
| existe critério de qualidade explícito; a revisão melhora o resultado; 2–3 chamadas adicionais se justificam | o critério **é executável** — nesse caso o avaliador é código, e o laço não precisa de modelo nenhum (§2.1) |
| texto sujeito a requisitos formais, tradução com glossário obrigatório | o critério não é escrevível — o avaliador adequado é uma pessoa |
| a verificação depende de julgamento que só um modelo faz | tarefa subjetiva sem lastro verificável |

Implementação em [`03-avaliador-otimizador.py`](https://github.com/celsocrivelaro/senac-llm-code/blob/main/aula05-agentes/03-avaliador-otimizador.py), que executa também **sem** critério, para evidenciar a degeneração do avaliador em elogio.

---


---

## Exemplo

### O avaliador sem critério

```python
PROMPT_A = "Avalie se o parecer abaixo está bom. Responda aprovado ou reprovado."
# -> "Aprovado. O parecer está claro, bem estruturado e cobre os pontos principais."

PROMPT_B = """Verifique item a item e responda no schema:
- cita_regra: cita o artigo específico da política?
- cita_valor: menciona o valor exato da despesa?
- conclui: termina com APROVADO ou REPROVADO explícito?
- o_que_corrigir: se algum item é falso, o que exatamente falta."""
# -> {"cita_regra": false, "cita_valor": true, "conclui": true,
#     "o_que_corrigir": "não cita qual artigo da política foi violado"}
```

O primeiro avaliador aprovou um parecer que não cita a regra. A causa não é deficiência do modelo, e sim ausência de especificação do que constitui qualidade. A qualidade de um avaliador é a qualidade do critério redigido; o modelo apenas o executa.


---

## Fontes e leituras

SHINN, N. et al. **Reflexion**: language agents with verbal reinforcement learning. arXiv:2303.11366, 2023.
