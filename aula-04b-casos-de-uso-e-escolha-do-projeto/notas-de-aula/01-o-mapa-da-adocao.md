# IA Aplicada com LLMs — Aula 04B: Casos de uso de agentes — O mapa da adoção

## Introdução

Você sabe construir um agente. Sabe escolher a arquitetura, impor orçamento, tratar erro e comprimir contexto. O que você não sabe — e o que esta aula existe para dar — é **o que o mundo já tentou fazer com isso**.

Essa pergunta parece fácil de responder: é só procurar. O problema é o que você encontra quando procura. Três resultados, todos de fontes sérias, todos de 2025–2026:

> **McKinsey:** a fatia de organizações **escalando agentes** em pelo menos uma função subiu de 27% para 40% em um ano.

> **MIT, Projeto NANDA:** **95%** dos pilotos de IA generativa não produziram nenhum impacto mensurável no resultado financeiro.

> **Gartner:** **mais de 40%** dos projetos de IA agêntica serão **cancelados** até o fim de 2027.

Antes de continuar, pare e tente conciliá-las. Uma diz que a adoção dobrou. A outra diz que quase tudo falha. A terceira diz que metade do que existe vai morrer.

Quem está mentindo?

**Ninguém.** E entender por que ninguém está mentindo é a coisa mais útil desta nota — mais útil, inclusive, que qualquer um dos três números.

> **Pré-requisitos:** da Aula 02, a [nota 01 §7](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — *leaderboard não decide nada*. Esta nota é a mesma desconfiança, aplicada a *press release*. Da Aula 04, a [nota 01](../../aula-04-arquitetura-de-agentes/notas-de-aula/01-padroes-de-arquitetura.md) — o vocabulário de padrões com que você vai ler os casos da [nota 02](02-os-casos-que-funcionam.md).

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Explicar** por que três estudos sérios chegam a conclusões aparentemente opostas sobre o mesmo fenômeno.
- **Distinguir** as três perguntas que se confundem no debate: *usa?*, *deu resultado?* e *sobreviveu?*
- **Aplicar** quatro perguntas de leitura crítica a qualquer estatística de adoção de IA.
- **Distinguir** um número **medido** de um número **declarado**, e saber por que quase todos são declarados.
- **Usar** a taxonomia dos seis tipos de agente para navegar o mercado.
- **Dizer** onde a adoção de agentes está concentrada, e por quê.

---

## Desenvolvimento teórico

### 1. As três perguntas que ninguém separa

As três manchetes não se contradizem porque **não estão respondendo à mesma pergunta**. Repare:

| Estudo | A pergunta que ele responde | O que ele **não** diz |
|---|---|---|
| McKinsey | **Usa?** A organização tem agentes rodando em escala em alguma função? | se aquilo produz dinheiro |
| MIT / NANDA | **Deu resultado?** O piloto mudou a linha do resultado financeiro? | se a empresa continua usando |
| Gartner | **Sobreviveu?** O projeto vai continuar existindo em 2027? | se ele funcionava enquanto existiu |

Um único sistema pode, ao mesmo tempo: **estar em uso** (entra na conta da McKinsey), **não ter impacto medido no P&L** (entra nos 95% do MIT) e **ser cancelado no ano que vem** (entra nos 40% da Gartner). As três afirmações são verdadeiras sobre ele. Simultaneamente.

> **A confusão é sempre a mesma:** *adoção* não é *resultado*, e *resultado* não é *permanência*. Quem lê as três como se fossem a mesma coisa conclui que "os dados são contraditórios" e para de olhar — que é exatamente o erro que esta nota quer evitar.

E vale ver de perto o que cada um mediu, porque o detalhe muda a leitura.

**McKinsey.** O dado de 27% → 40% é sobre *escalar em ao menos uma função*. Na mesma pesquisa, quando se olha **uma função específica**, nenhuma passa de cerca de 10% das organizações escalando agentes. Ou seja: muita empresa tem agente em algum canto; quase nenhuma tem agente espalhado. E entre organizações menores a adoção ficou praticamente parada, em torno de 22%.

**MIT / Projeto NANDA.** O relatório *The GenAI Divide: State of AI in Business* (julho de 2025) é o mais citado e o de amostra mais frágil: 52 entrevistas com executivos, 153 respondentes de survey e a análise de 300 implantações públicas. O achado — 95% dos pilotos sem impacto mensurável no P&L, contra US$ 30–40 bilhões investidos — é forte; a base é pequena. E a conclusão do próprio relatório contraria a leitura preguiçosa que se faz dele: **a causa não é a qualidade dos modelos**, é a lacuna de aprendizado e de integração organizacional. Não é um relatório dizendo que a IA não funciona. É um relatório dizendo que **integrar** é o problema.

**Gartner.** A previsão dos 40% cancelados vem com as causas declaradas: custo crescente, valor de negócio pouco claro e controle de risco inadequado. Nenhuma delas é sobre capacidade do modelo — todas as três são sobre **engenharia e gestão**, que é o assunto da Aula 04.

Repare no que os três concordam, apesar de discordarem no título:

> **O gargalo não é o modelo.** É integração, custo e controle.

---

### 2. As quatro perguntas

Você vai ler dezenas desses números — na imprensa, em proposta comercial, em slide de fornecedor. Quatro perguntas dão conta de quase tudo.

#### 2.1 Quem pagou?

Boa parte da estatística de adoção de agentes é publicada por quem **vende** agentes: fornecedores de plataforma, consultorias que prestam serviço de implantação, provedores de nuvem. Isso não invalida o número — invalida **a leitura ingênua** dele.

O sinal de alerta prático: quando o número é bom demais para o negócio de quem o publicou, desconfie da definição, não da honestidade. Ninguém precisa mentir para produzir um número inflado; basta perguntar de um jeito generoso.

#### 2.2 Quem respondeu?

Três amostras muito diferentes, todas chamadas de "pesquisa":

| Tipo | Exemplo | O que vale |
|---|---|---|
| Survey grande com cotas | JetBrains, >15.000 desenvolvedores, cotas regionais | o mais confiável desta aula |
| Survey de executivos | McKinsey, Sinch | mede **percepção de quem decide**, não o sistema |
| Enquete de webinar | a enquete Gartner de jan/2025, 3.412 participantes | **auto-selecionada**: quem entra num webinar sobre agentes já se interessa por agentes |

A enquete de webinar não é lixo — ela é útil para o que é. Só não é uma amostra da economia.

#### 2.3 O que foi perguntado, exatamente?

Aqui mora a maior parte da divergência entre estudos. Estas quatro perguntas produzem números radicalmente diferentes e são publicadas com o mesmo título de "adoção de IA":

1. *A sua empresa usa IA em alguma função?* — quase nove em dez respondem sim.
2. *A sua empresa tem um agente em produção?* — cai para algo entre um quinto e um terço, dependendo do estudo.
3. *A sua empresa escalou agentes em alguma função?* — cerca de 40% (McKinsey).
4. *O investimento em IA teve impacto mensurável no resultado?* — despenca.

São quatro perguntas diferentes. Manchete não distingue.

#### 2.4 O número é medido ou declarado?

Esta é a pergunta que quase ninguém faz, e é a que mais separa.

**Declarado** é auto-relato: alguém respondeu um formulário sobre a própria empresa. Está sujeito a otimismo, a pressão do chefe, a não saber o que os outros times fazem e à tentação de parecer moderno. **A esmagadora maioria dos números desta aula é declarada.**

**Medido** é telemetria: o número sai do sistema, não da opinião. O **Anthropic Economic Index** é o exemplo raro nesta aula — ele analisa o uso real de uma plataforma. E é interessante justamente porque desmente a narrativa fácil dos dois lados: nos dados de novembro de 2025, a fatia de conversas classificadas como **aumento** (o modelo ajuda a pessoa) subiu 5 pontos, para 52%, enquanto a de **automação** (o modelo faz a tarefa) caiu 4 pontos, para 45%. Numa janela mais longa, porém, a automação vem subindo devagar.

Ou seja: nem substituição total, nem hype vazio. Um deslocamento lento, com idas e vindas — que é como as coisas costumam ser, e é péssimo para manchete.

> **A regra:** um número medido de amostra estreita costuma valer mais que um número declarado de amostra larga. E quando você só tem declarado — que é o caso normal —, trate-o como *ordem de grandeza*, nunca como medida.

---

### 3. O teste de cheiro

Antes das quatro perguntas, há um filtro mais rápido: **o número é plausível?**

Você vai encontrar, em português, manchetes do tipo *"96% das organizações já utilizam agentes de IA"* e *"98% das empresas no Brasil já levam projetos de IA para produção com sucesso"*. Compare com os outros números desta nota — 40% escalando, 95% dos pilotos sem impacto, mais de 40% a caminho do cancelamento. **Pelo menos um dos lados está errado**, e não é possível que os dois descrevam o mesmo mundo.

Quando duas fontes divergem tanto assim, quase sempre a explicação é a §2.3: perguntaram coisas diferentes e publicaram com o mesmo rótulo. "Utiliza agentes" pode significar *tem alguém no time usando uma ferramenta de IA*. "Leva projetos à produção com sucesso" pode significar *pelo menos um projeto, alguma vez*.

Você não precisa saber qual está certo. Precisa saber **que não dá para citar os dois**.

---

### 4. A taxonomia que você vai encontrar

Quando for pesquisar o seu case, vai esbarrar na classificação que o mercado adotou — a do catálogo público do Google Cloud, que reúne mais de mil casos reais organizados por **11 setores × 6 tipos de agente**:

```
   ┌─────────────────┬──────────────────────────────────────────────┐
   │ CUSTOMER agents │ falam com o cliente final                    │
   │ EMPLOYEE agents │ apoiam o funcionário por dentro              │
   │ CODE agents     │ escrevem, revisam e corrigem software        │
   │ DATA agents     │ consultam, cruzam e resumem dados            │
   │ CREATIVE agents │ produzem conteúdo, imagem, texto de marketing│
   │ SECURITY agents │ detectam, triam e remediam incidentes        │
   └─────────────────┴──────────────────────────────────────────────┘
```

Note que **é uma taxonomia por destinatário, não por arquitetura**. Ela diz *para quem* o agente trabalha, e não *como* ele é feito. Um *customer agent* pode ser um roteador simples ou um agente autônomo — e a [nota 02](02-os-casos-que-funcionam.md) mostra que, na prática, quase sempre é o primeiro.

Isso importa para você por um motivo bem concreto: **o material do mercado descreve o destinatário e omite a arquitetura**. Ler a arquitetura por trás da descrição é trabalho seu, e você já tem o vocabulário para fazê-lo — é a Aula 04.

---

### 5. Onde a adoção realmente está

Somando as fontes, o mapa é razoavelmente consistente:

**Por função** — a adoção se concentra em **TI, gestão de conhecimento e engenharia de software**. Não é coincidência que sejam as três funções onde quem decide a compra é também quem usa a ferramenta, e onde o resultado é verificável por quem está na sala.

**Por setor** — bancos e seguros à frente; saúde e governo bem atrás. Duas explicações se somam: dinheiro disponível e tolerância regulatória. Onde o erro é catastrófico ou o dado é protegido por lei, a adoção é mais lenta — e isso é racional, não atraso.

**Por tipo de empresa** — empresas de tecnologia usam agentes de forma muito mais disseminada que as demais. Organizações pequenas ficaram praticamente paradas. Agente ainda é, majoritariamente, coisa de empresa grande e de empresa de tecnologia.

**E a maturidade de governança não acompanhou:** apenas cerca de um terço das organizações relata maturidade razoável em estratégia e governança de IA agêntica. Guarde esta frase — ela volta, com número brasileiro e mais desconfortável, na [nota 04](04-o-brasil-e-a-escolha-do-case.md).

---

## Exemplos

### Exemplo 1 — As quatro perguntas aplicadas à McKinsey

> *"A fatia de organizações escalando agentes em ao menos uma função subiu de 27% para 40%."*

| Pergunta | Resposta |
|---|---|
| Quem pagou? | consultoria que vende serviço de transformação com IA — **interesse existe**, e a metodologia é publicada |
| Quem respondeu? | executivos, em survey global grande |
| O que foi perguntado? | *escalar em **ao menos uma** função* — o "ao menos uma" faz quase todo o trabalho |
| Medido ou declarado? | **declarado** |

**Leitura final:** o número é bom para dizer que agentes deixaram de ser experimento em uma fatia relevante de empresas grandes. É ruim para dizer que "40% das empresas rodam agentes" — que é como ele costuma ser citado.

### Exemplo 2 — As mesmas perguntas no MIT

> *"95% dos pilotos de IA generativa não geram impacto mensurável."*

| Pergunta | Resposta |
|---|---|
| Quem pagou? | iniciativa acadêmica — **sem interesse comercial direto** |
| Quem respondeu? | 52 entrevistas, 153 respondentes, 300 implantações públicas — **amostra pequena** |
| O que foi perguntado? | impacto **mensurável no P&L** — critério exigente, e muitos pilotos nem tentam medir |
| Medido ou declarado? | misto: implantações públicas analisadas + auto-relato |

**Leitura final:** o número é uma denúncia da **falta de medição**, tanto quanto da falta de resultado. E a causa apontada pelo próprio relatório — integração, não capacidade do modelo — é a que mais importa para você, porque é a parte que **você** vai construir.

---

## Exercícios resolvidos

### 1. Conciliar dois números

> Uma fonte diz que 76% das empresas de um país têm agentes em produção. Outra diz que mais de 40% dos projetos agênticos serão cancelados. O país é um caso perdido?

**Não — e as duas podem estar certas.** "Ter em produção" é um retrato de **agora**; "ser cancelado até 2027" é uma previsão sobre **permanência**. Um parque grande de agentes em produção com alta taxa de cancelamento futuro descreve exatamente um mercado que **entrou rápido e ainda não aprendeu a operar**.

E há uma leitura de carreira nisso: um mercado assim precisa de gente que saiba **manter e consertar** agentes, não só construí-los. É o assunto da [nota 04](04-o-brasil-e-a-escolha-do-case.md).

### 2. Qual pergunta este número responde?

Classifique cada afirmação em *usa*, *deu resultado* ou *sobreviveu*:

| Afirmação | Pergunta |
|---|---|
| "90% dos desenvolvedores usam agentes de codificação semanalmente" | **usa** |
| "A empresa X dobrou a produtividade dos analistas em nove meses" | **deu resultado** |
| "80% das organizações já reverteram alguma implementação" | **sobreviveu** (no negativo) |
| "70% das conversas são resolvidas sem humano" | **deu resultado** — mas cuidado: *resolvida* é definição da empresa, e ela escolheu a régua |

A última é a mais instrutiva. "Resolvida sem humano" costuma significar *a conversa terminou sem transbordo* — o que inclui o cliente que desistiu. Sempre que o número depender de uma definição, **procure a definição**.

---

## Síntese

- As três manchetes contraditórias respondem a três perguntas diferentes: **usa**, **deu resultado**, **sobreviveu**. Um mesmo sistema pode contar nas três.
- Onde os três estudos concordam: **o gargalo não é o modelo** — é integração, custo e controle de risco.
- Quatro perguntas resolvem quase toda leitura de estatística: **quem pagou · quem respondeu · o que foi perguntado exatamente · medido ou declarado**.
- Quase todo número de adoção é **declarado** (auto-relato). Trate-o como ordem de grandeza.
- Antes das quatro perguntas, o **teste de cheiro**: números muito acima dos demais quase sempre mudaram a definição, não a realidade.
- A taxonomia de mercado (6 tipos de agente) classifica **por destinatário**, não por arquitetura — ler a arquitetura é trabalho seu.
- A adoção se concentra em TI, conhecimento e engenharia de software; bancos e seguros à frente, saúde e governo atrás; empresas pequenas praticamente paradas.
- A governança não acompanhou a adoção — só cerca de um terço relata maturidade razoável.

---

## Fontes e leituras

- **McKinsey — The State of AI** ([tech-forward, 2026](https://www.mckinsey.com/capabilities/tech-and-ai/our-insights/tech-forward/state-of-ai-trust-in-2026-shifting-to-the-agentic-era)) — a fatia escalando agentes, por função e por porte.
- **MIT, Projeto NANDA — *The GenAI Divide: State of AI in Business*** (jul/2025) — os 95% e, mais importante, a causa apontada. [Cobertura com os dados de amostra](https://www.healthcareitnews.com/news/mit-95-enterprise-ai-pilots-fail-deliver-measurable-roi).
- **Gartner — *Over 40% of Agentic AI Projects Will Be Canceled by End of 2027*** ([jun/2025](https://www.gartner.com/en/newsroom/press-releases/2025-06-25-gartner-predicts-over-40-percent-of-agentic-ai-projects-will-be-canceled-by-end-of-2027)) — a previsão, as causas e o *agent washing*.
- **Anthropic Economic Index** ([relatório de jun/2026](https://www.anthropic.com/research/economic-index-june-2026-report)) — o contraponto **medido**: automação × aumento no uso real.
- **Google Cloud — catálogo de casos reais** ([1.302 casos, 11 setores × 6 tipos](https://cloud.google.com/transform/101-real-world-generative-ai-use-cases-from-industry-leaders)) — a taxonomia, e o material de navegação da [nota 04](04-o-brasil-e-a-escolha-do-case.md).
- Aula 02, [nota 01 §7](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — *leaderboard não decide nada*, o ancestral direto desta nota.
