# IA Aplicada com LLMs — Aula 11: Evals — O juiz, e a suíte

> **Pré-requisitos:** notas 01 e 02 desta aula · Aula 02, [nota 03](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/03-controle-da-saida.md) — saída estruturada · Aula 04, [nota 01](../../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/01-o-que-e-um-agente.md) — o padrão avaliador-otimizador.
>
> **Código:** [`aula11-evals/`](https://github.com/celsocrivelaro/senac-llm-code/tree/main/aula11-evals) — os scripts `05-human-review.py`, `06-llm-as-judge.py`, `07-rubric-scoring.py`, `08-regression-suite.py` e `09-agentevals-openevals.py` correspondem a esta nota.

---

## Objetivos de aprendizagem

Ao final desta nota, o aluno deve ser capaz de:

- **Escrever** um juiz LLM com rubrica explícita e saída estruturada.
- **Nomear** os vieses documentados de juízes automáticos e a mitigação de cada um.
- **Medir** a concordância entre o juiz e rótulos humanos, e decidir a adoção por limiar declarado.
- **Executar** uma suíte de regressão comparando duas versões, com repetição.
- **Decidir** se uma diferença observada justifica promover a mudança.

---

## Desenvolvimento teórico

### 1. Como se escreve um juiz

Quatro elementos, e nenhum é opcional.

**Rubrica explícita e curta.** O critério precisa estar escrito, não subentendido. "Avalie a qualidade da resposta" não é rubrica — é delegação de critério ao modelo, que então inventa um, diferente a cada chamada.

**Saída estruturada** (Aulas 02 e 03). O juiz devolve um objeto validável, e não prosa a ser interpretada por expressão regular.

**Critério binário antes de escala.** Um juiz que devolve nota de 1 a 10 sem âncoras devolve ruído com aparência de precisão — a diferença entre 6 e 7 não é reprodutível nem entre humanos. Comece com aprovado/reprovado; introduza escala só quando o binário se mostrar insuficiente, e com cada ponto ancorado em descrição.

**Exemplos dos dois lados.** Um caso que passa e um que não passa, com a razão. É *few-shot* da Aula 03, aplicado ao avaliador.

```python
RUBRICA_FIDELIDADE = """Você avalia se uma resposta é FIEL ao contexto fornecido.

FIEL: toda afirmação factual da resposta pode ser derivada do contexto.
NÃO FIEL: a resposta contém ao menos uma afirmação factual que o contexto
          não sustenta — ainda que a afirmação seja verdadeira no mundo.

Exemplo FIEL:
  contexto: "Art. 7º ... teto de R$ 120,00 por refeição."
  resposta: "O teto é de R$ 120,00, conforme o art. 7º."

Exemplo NÃO FIEL:
  contexto: "Art. 7º ... teto de R$ 120,00 por refeição."
  resposta: "O teto é de R$ 120,00, e valores acima exigem aprovação do
             diretor."                       <- não está no contexto

Responda no schema: {"fiel": bool, "afirmacao_nao_sustentada": str|null}"""
```

O critério de "não fiel" vale **ainda que a afirmação seja verdadeira no mundo**. Fidelidade mede aderência ao contexto, não veracidade — e essa distinção precisa estar na rubrica, ou o juiz avalia outra coisa.

### 2. Os vieses documentados

A literatura sobre juízes automáticos identificou vieses reprodutíveis (Zheng et al., 2023; Li et al., 2024). Três importam na prática, e cada um tem mitigação distinta.

| Viés | O que é | Mitigação |
|---|---|---|
| **Posição** | em comparação de pares, a ordem altera o veredito | avaliar nas duas ordens e considerar só o acordo |
| **Verbosidade** | respostas longas parecem melhores | rubrica que pontua completude, não extensão |
| **Auto-preferência** | o modelo prefere texto do próprio modelo | juiz de família diferente da do sistema |

O terceiro é o mais desconfortável para esta disciplina, em que o mesmo provedor serve o sistema e o juiz. É uma limitação real do laboratório, e vale declará-la em aula em vez de escondê-la — a mitigação, quando o orçamento permite, é trocar a família do juiz.

### 3. A tese: validar o juiz

Um juiz é um instrumento de medida. Instrumento de medida se **calibra** contra um padrão antes de ser usado, e o padrão aqui é o julgamento humano.

O procedimento:

1. **Rotular à mão** ao menos cinquenta casos, com critério binário e uma crítica escrita por caso.
2. **Executar o juiz** sobre os mesmos casos.
3. **Medir a concordância** — a fração em que juiz e humano coincidem.
4. **Adotar só acima do limiar declarado.** A literatura trabalha na faixa de 75% a 90%; o valor exato é decisão do projeto, e precisa ser escrito.
5. **Ler os desacordos um a um.** Eles dizem se o problema é do juiz ou da rubrica — e, com frequência, dizem que o critério humano não estava tão claro quanto se supunha.

> A matriz de concordância é mais informativa que o número agregado. Um juiz que **nunca reprova** pode ter 80% de concordância num conjunto em que 80% dos casos passam, e não está medindo nada.

```
                  humano: passa   humano: falha
  juiz: passa            41              9      <- falsos positivos
  juiz: falha             2             48

  concordância ..... 89 / 100
  falsos positivos ... 9    <- o juiz aprova o que o humano reprova
  falsos negativos ... 2
```

Nove falsos positivos num juiz de fidelidade significa que **nove respostas com afirmação não sustentada passariam**. Se o custo desse erro for alto, 89% não basta, por mais que o número pareça bom.

### 4. O mesmo mecanismo, em outro papel

O padrão **avaliador-otimizador** da Aula 04 é um juiz LLM **dentro** do sistema: um modelo critica a saída de outro, em produção, para melhorá-la. O juiz desta aula está **fora**, na bancada, e serve para decidir se o sistema pode mudar.

Mesmo mecanismo, papéis opostos — e uma consequência: **o juiz de fora precisa ser mais confiável que o de dentro**. Um avaliador em produção que erra às vezes degrada a saída; um juiz de bancada que erra sistematicamente autoriza a promoção de mudanças ruins, e o erro se acumula em todas as versões seguintes.

### 5. A suíte de regressão

A suíte é o teste de regressão da Aula 03, crescido de uma dimensão para todas as que decidem o comportamento do sistema: prompt, modelo, parâmetros, índice, memória, framework, servidor, conjunto de casos e rubrica do juiz.

**Quando roda:** antes de toda mudança de prompt, modelo, parâmetro, índice, memória, framework ou versão de servidor. Ou seja: a cada alteração de qualquer campo do carimbo.

**O que produz:** não um "passou/falhou", mas um relatório por fatia — porque, como a nota 01 estabeleceu, a média esconde a classe que quebrou.

```
                        versão A    versão B     delta
  simples (72)             97%         97%         0
  exceção intl. (8)        25%         62%       +37   <- a mudança visava isto
  recusa (12)              33%         17%       -16   <- e quebrou isto
  ──────────────────────────────────────────────────────
  MÉDIA (92)               83%         87%        +4
```

A média subiu quatro pontos, e a mudança piorou a recusa pela metade. Promover com base na média é o erro que o relatório por fatia impede.

### 6. A pergunta estatística

Uma diferença de 82% para 85% em quarenta casos **não é afirmável**. Três acertos a mais em quarenta é uma diferença de três casos, e a saída é não determinística: repetir a mesma versão duas vezes produziria variação da mesma ordem.

Duas exigências decorrem disso, e ambas são baratas:

**Repetir.** Cada caso executado *k* vezes, e a taxa reportada com a dispersão. É o mesmo argumento do `pass^k` da nota 01, aplicado à comparação entre versões.

**Declarar um limiar de promoção.** Uma regra escrita antes de olhar o resultado — por exemplo: *não se promove mudança cuja diferença não exceda a variação observada entre duas execuções da mesma versão*. A regra precisa ser anterior ao resultado, porque decidir depois é escolher o critério que confirma o que se queria fazer.

> O procedimento mínimo, e ele não exige estatística avançada: rode a **versão A duas vezes**. A diferença entre essas duas execuções é o seu ruído de base. Só diferenças maiores que ele merecem interpretação.

### 7. Offline e online

O que esta aula constrói é avaliação **offline**: conjunto fixo, executado na bancada, antes do *deploy*. Ela barra a promoção de mudanças piores.

O que ela não cobre é a **avaliação online**: amostrar o tráfego real e executar o juiz sobre a amostra, com o sistema no ar. É o que pega o que o *dataset* não previu — e o que devolve casos novos para o *dataset*, fechando o ciclo da nota 01, §6.

A avaliação online exige sistema em produção, painel e política de amostragem. **É entregue à Aula 12.**

### 8. O que a suíte destrava

Vale fechar dizendo o que este trabalho torna possível, porque a lista é maior do que parece:

- **migrar de modelo depreciado** (Aula 02) sem apostar;
- **atualizar o framework** (Aula 09) sabendo o que quebrou;
- **decidir se o multiagente compensa** (Aula 14) — e é essa a razão de esta aula vir antes daquela;
- **medir o custo de uma defesa de segurança** (Aula 13), que é o dado que falta em quase toda discussão da área.

A suíte existe **antes** do multiagente, e é ela que vai julgá-lo.

---

## Exemplos

### Exemplo 1 — O juiz, e o mesmo caso sem rubrica

```
SEM RUBRICA — "avalie a qualidade da resposta, de 0 a 10"
  execução 1: 8    "resposta clara e bem estruturada"
  execução 2: 6    "faltou detalhar a base legal"
  execução 3: 9    "correta e objetiva"
  -> três critérios diferentes, inventados na hora

COM RUBRICA BINÁRIA DE FIDELIDADE
  execução 1: fiel=false, afirmacao="exige aprovação do diretor"
  execução 2: fiel=false, afirmacao="exige aprovação do diretor"
  execução 3: fiel=false, afirmacao="exige aprovação do diretor"
  -> mesmo veredito, e a razão é inspecionável
```

### Exemplo 2 — A validação, com os desacordos lidos

```
concordância: 89/100

DESACORDOS (11), lidos um a um:
  7 casos  o juiz reprovou paráfrase fiel do contexto
           -> defeito da RUBRICA: não dizia que paráfrase é aceitável
  3 casos  o juiz aprovou resposta com número não presente no contexto
           -> defeito do JUIZ: não verifica valores numéricos
  1 caso   o rótulo humano estava errado
           -> defeito do CONJUNTO: corrigido

apos corrigir a rubrica: 96/100
```

Os onze desacordos valeram mais que os oitenta e nove acordos. É por isso que a etapa 5 do procedimento — ler os desacordos — não é opcional.

### Exemplo 3 — O ruído de base, medido antes de comparar

```
versão A, execução 1 .... 83.7%
versão A, execução 2 .... 81.5%
                          ------
ruído de base ........... 2.2 pontos

versão B ................ 85.1%
delta contra A .......... +1.4 pontos    <- MENOR que o ruído

VEREDITO: não promover. A diferença não excede a variação da própria
          versão A consigo mesma.
```

Duas execuções a mais custaram alguns minutos e evitaram uma conclusão errada.

---

## Fontes e leituras

ZHENG, L. et al. **Judging LLM-as-a-judge with MT-Bench and Chatbot Arena**. arXiv:2306.05685, 2023.

LI, D. et al. **From generation to judgment**: opportunities and challenges of LLM-as-a-judge. arXiv:2411.16594, 2024.

YAO, S. et al. **τ-bench**: a benchmark for tool-agent-user interaction in real-world domains. arXiv:2406.12045, 2024.

ES, S. et al. **RAGAS**: automated evaluation of retrieval augmented generation. arXiv:2309.15217, 2023.

**Material da disciplina.** Aula 02, [nota 03](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/03-controle-da-saida.md) — saída estruturada, que o juiz usa. Aula 03, [nota 03](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) — o teste de regressão que aqui cresce. Aula 04, [nota 01](../../aula-04-casos-de-uso-e-escolha-do-projeto/notas-de-aula/01-o-que-e-um-agente.md) — o avaliador-otimizador, o mesmo mecanismo dentro do sistema.
