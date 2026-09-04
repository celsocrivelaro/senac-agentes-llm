# IA Aplicada com LLMs — Aula 04B: Casos de uso de agentes — Os casos que funcionam

## Introdução

A [nota anterior](01-o-mapa-da-adocao.md) foi sobre desconfiar de números. Esta é sobre o que sobra depois da desconfiança: os casos concretos, com empresa nomeada e resultado divulgado.

Cada caso aqui vem no mesmo gabarito, e o terceiro item é o que diferencia esta nota de uma matéria de revista:

1. **o que a empresa fez**;
2. **o número que ela divulgou**;
3. **que padrão da Aula 04 provavelmente está por trás**;
4. **por que funcionou** — e o que isso ensina para o seu case.

O item 3 exige um aviso. Empresa nenhuma publica o diagrama de arquitetura junto com o *case study*; o que se publica é o destinatário e o resultado. A leitura de arquitetura é **inferência**, e às vezes vai estar errada. Fazê-la mesmo assim é o exercício: você tem o vocabulário da Aula 04, e usá-lo em descrição de marketing é exatamente a habilidade que vai precisar quando alguém te trouxer um requisito começando com *"queria um agente que…"*.

E há um padrão que aparece cedo e se repete até o fim:

> **Quase nenhum "agente" de sucesso é um agente.** A maioria é roteador com workflow, e a autonomia fica confinada a um caminho estreito. Isso não é decepcionante — é a Aula 04, nota 01, acontecendo no mundo.

> **Pré-requisitos:** a [nota 01](01-o-mapa-da-adocao.md) desta aula. Da Aula 04, a [nota 01](../../aula-04-arquitetura-de-agentes/notas-de-aula/01-padroes-de-arquitetura.md) inteira (os cinco padrões e a tabela de decisão) e a [nota 02 §1](../../aula-04-arquitetura-de-agentes/notas-de-aula/02-o-agente-e-o-estado.md) (as três condições do agente).

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Citar** casos reais de agentes em produção nos seis tipos, com o número divulgado e a fonte.
- **Inferir** a arquitetura provável de um caso a partir da descrição pública dele.
- **Explicar** por que *code agents* é o tipo mais maduro, usando a terceira condição do agente.
- **Enumerar** as seis características comuns aos casos que funcionam.
- **Reconhecer** a ausência dessas características num caso novo — inclusive no seu.

---

## Desenvolvimento teórico

### 1. Code agents — o caso mais maduro, e o porquê importa mais que o quanto

Comece pelo número, que é o mais sólido desta aula porque vem da maior amostra: a *Developer Ecosystem Survey* da JetBrains, com mais de **15.000 desenvolvedores profissionais** (coleta entre maio e julho de 2026, oito idiomas, cotas regionais):

- **90%** usam agentes de codificação no trabalho **pelo menos semanalmente**;
- **68%** usam **diariamente**.

Nenhum outro tipo de agente chega perto disso. E o mercado de ferramentas se reorganizou rápido no período: Claude Code em 39% de adoção global (47% nos EUA), GitHub Copilot caindo de 29% para 21%, Codex saltando de 3% para 16%, Cursor recuando de 18% para 12%.

A pergunta interessante não é *quanto*. É **por que aqui primeiro?**

A resposta você já tem, e é a Aula 04, nota 02 §1 — a terceira condição do agente:

> **O ambiente devolve feedback verificável.**

Em software, o teste passa ou falha. O compilador aceita ou recusa. O *linter* aponta a linha. O programa roda ou estoura. É um ambiente que **contradiz o modelo em segundos, de graça, sem ambiguidade** — e um agente só se corrige onde existe algo capaz de dizer que ele errou.

Compare com "escrever um texto persuasivo": nenhuma ferramenta consegue dizer se convenceu. O laço não tem do que se corrigir; ele só acumula passos com a confiança intacta.

> **A lição para o seu case, e é a mais importante desta nota:** procure o verificador **antes** de procurar o problema. Se o domínio que você escolheu tem um "teste que passa ou falha", você escolheu um domínio onde agentes funcionam hoje. Se não tem, você vai passar o semestre sem saber se o seu sistema está certo — e o professor também não vai saber.

*Arquitetura provável:* **agente de verdade**, com laço longo. É um dos poucos tipos onde a autonomia se justifica, e não por acaso é o tipo em que ela funciona.

---

### 2. Customer agents — volume alto, autonomia baixa

O tipo mais visível, e o mais mal compreendido.

**Commerzbank** opera cerca de **2 milhões de conversas**, com **70%** resolvidas sem intervenção humana.
**NoBroker** processa cerca de 10.000 horas de gravação por dia, com agentes cobrindo **25% a 40%** das chamadas.
**LUXGEN** reduziu em **30%** a carga de trabalho dos atendentes.

Repare no formato dos três números: nenhum é 100%, e nenhum pretende ser. **Todos descrevem uma fatia.**

*Arquitetura provável:* **roteador + workflow**, com o agente reservado ao caso difícil. É literalmente o desenho do laboratório da Aula 04 — a triagem manda a maioria para um caminho barato e previsível, e a autonomia cara fica para a minoria que precisa dela.

E aqui vale recuperar a conta do `00-roteador.py`: a rota mais valiosa é a que **não chama o modelo**. Quando um banco diz que resolve 70% das conversas automaticamente, boa parte desses 70% é consulta de saldo respondida por regra — não é um agente raciocinando sobre finanças pessoais.

**O cuidado com a definição.** "Resolvida sem humano" é régua da própria empresa. Costuma significar *a conversa terminou sem transbordo para atendente* — o que inclui o cliente que desistiu e foi embora. Ao ler qualquer número de atendimento, procure a definição de "resolvido"; se ela não estiver publicada, o número vale menos.

---

### 3. Employee agents — assistência, não substituição

**United Wholesale Mortgage** relata ter **mais que dobrado** a produtividade de analistas de crédito em nove meses.
**Uber** usa sumarização de comunicações com usuários como ferramenta interna.
**Intuit**, no TurboTax, preenche automaticamente os dez formulários fiscais mais comuns dos EUA.

O padrão comum aos três é o que **não** aparece na manchete: **o humano continua decidindo**. O analista de crédito não foi substituído — ele passou a receber o caso pré-analisado. O formulário preenchido é conferido antes de ser enviado.

*Arquitetura provável:* **workflow com ferramentas**, ou *chaining* com portão. Em quase nenhum deles o modelo escolhe livremente a sequência.

> **Por que este tipo dá certo com tanta frequência:** o humano no fim da linha é um **verificador**, e um verificador com autoridade. O erro do sistema custa alguns segundos de conferência, e não um prejuízo. É a arquitetura mais tolerante a falha que existe — e por isso a mais fácil de colocar em produção.

Se o seu case couber aqui, ele é mais fácil de defender e mais fácil de terminar.

---

### 4. Data agents — o território do orquestrador

**Geotab** analisa dados de **4,7 milhões de veículos**, com bilhões de pontos por dia.
**Moglix** relata **4×** de melhoria em eficiência de *sourcing*.
**Domina** cita 80% de melhoria em acesso a dados e 15% em efetividade.
**BMW** usa milhares de simulações em gêmeos digitais para otimizar distribuição.

*Arquitetura provável:* **orquestrador-trabalhador**. É o tipo em que a decomposição realmente depende do conteúdo — quantas consultas, contra que fontes e em que ordem só se sabe depois de ver o primeiro resultado.

E é também o tipo que mais precisa do **teto** da Aula 04, nota 01 §4.4: um orquestrador solto sobre um banco grande é a forma mais rápida conhecida de transformar uma pergunta em uma fatura.

**A ressalva de leitura.** Números como "4× de eficiência" e "+80% de acesso" são **autodeclarados, sem linha de base publicada**. 4× em relação a quê? Medido como? Aplique as quatro perguntas da [nota 01](01-o-mapa-da-adocao.md): eles servem para saber que o tipo funciona, não para prometer o mesmo ganho a ninguém.

---

### 5. Creative e security — os dois extremos da tolerância a erro

Vale colocá-los lado a lado, porque juntos delimitam o espectro.

**Creative agents** — a **AdVon Commerce** processou um catálogo de 93.673 produtos em menos de um mês, com ganhos relatados de 30% em posições de topo de busca e 67% em vendas médias diárias para um cliente.
Aqui, errar é barato: um texto de produto ruim é corrigido, e ninguém é prejudicado. A revisão humana é o padrão, e o volume é o que justifica.

**Security agents** — o outro extremo. O falso positivo custa o tempo de um analista escasso; o falso negativo custa um incidente. É o tipo com a **menor** tolerância a erro dos seis, e por isso o mais conservador em autonomia: quase sempre triagem e enriquecimento, com a ação de contenção exigindo aprovação.

> **A variável que os separa não é a tecnologia. É o custo do erro** — que é a terceira pergunta da triagem da Aula 04 (nota 01 §2). Quanto mais caro o erro, menor a autonomia que se aceita, independentemente do que o modelo é capaz de fazer.

---

### 6. O que os casos que funcionam têm em comum

Olhando os seis tipos de uma vez, o padrão é consistente. Esta tabela é o que você leva desta nota para a [nota 04](04-o-brasil-e-a-escolha-do-case.md), onde ela vira critério de escolha:

| Característica | Por que é necessária | Onde aparece |
|---|---|---|
| **Tarefa repetitiva, volume alto** | é onde a economia existe; tarefa rara não paga a engenharia | 2 milhões de conversas, 93 mil produtos, 10 mil horas/dia |
| **Feedback verificável** | sem ele o agente não se corrige (Aula 04, nota 02 §1) | o teste que passa ou falha, nos *code agents* |
| **Tolerância a erro pequeno** | erro vai acontecer numa fração relevante | texto de produto sim; contenção de incidente não |
| **Ação reversível — ou confirmação humana** | Aula 04, nota 02 §7 | o analista que confere antes de aprovar |
| **Dado acessível por API** | *"o difícil não é a inteligência, é o acesso confiável aos sistemas de produção"* | todos, sem exceção |
| **Um número que define sucesso** | sem critério, não dá para saber se funcionou | 70% resolvidas, 4× sourcing, −30% de carga |

E o que **não** aparece na lista, apesar de dominar a conversa pública: qual modelo foi usado, qual framework, quantos parâmetros. Nenhum *case study* atribui o resultado ao modelo. Todos atribuem à **integração** — que é exatamente o que o MIT apontou como causa dos 95% ([nota 01](01-o-mapa-da-adocao.md)).

---

### 7. A frase que resume a nota

Levantamentos de mercado convergem para a mesma formulação, e ela vale ser guardada:

> **O mais difícil hoje não é a inteligência do modelo. É o acesso seguro e confiável aos sistemas de produção.**

Isso deveria mudar a sua estimativa de esforço no projeto final. A parte que parece o projeto — o prompt, o laço, a escolha do modelo — você já sabe fazer, e ela é a menor. A parte que decide se funciona é ferramenta confiável, dado acessível e salvaguarda. Que é, não por acaso, o assunto da Aula 04 inteira.

---

## Exemplos

### Exemplo 1 — Lendo a arquitetura por trás da descrição

> *"Nosso agente de atendimento resolve 70% das conversas sem intervenção humana."*

O que dá para inferir, sem ter visto o código:

| Pista na frase | Inferência |
|---|---|
| "70%", e não 100% | existe **triagem**: alguém decide o que o sistema atende e o que passa adiante → **roteador** |
| "sem intervenção humana" | existe **transbordo** para humano → o roteador tem uma rota de escape |
| volume de milhões | a maior parte é pergunta repetida → boa parte é **regra ou template**, não modelo |
| atendimento a cliente final | ação de escrita (abrir chamado, alterar cadastro) quase certamente exige **confirmação** |

Arquitetura provável, desenhada com o vocabulário da Aula 04:

```
   conversa ──> [ ROTEADOR ] ──┬──> FAQ / regra          (a maior fatia)
                               ├──> [ workflow c/ tools ] (consulta, status)
                               ├──> [ AGENTE ]           (o caso difícil, pouco)
                               └──> humano               (transbordo)
```

Não é o diagrama da empresa. É uma **hipótese defensável**, construída a partir de quatro pistas — e é exatamente o tipo de raciocínio que o exercício desta aula pede.

### Exemplo 2 — Dois casos, e por que só um vira agente

| | *"Agente que corrige bugs a partir das issues do repositório"* | *"Agente que escreve o texto de marketing dos produtos"* |
|---|---|---|
| Tarefa repetitiva e volumosa | sim | sim |
| **Feedback verificável** | **sim** — o teste passa ou falha | **não** — nenhuma ferramenta diz se convenceu |
| Erro é barato | sim, o PR não é mergeado | sim, o texto é reescrito |
| Ação reversível | sim | sim |
| Critério de sucesso | % de issues resolvidas com teste verde | ...? |

Os dois são casos de uso legítimos e economicamente reais — a AdVon é o segundo, em escala. Mas só o primeiro é **agente**: no segundo, o laço não tem como se corrigir, e a arquitetura certa é geração em lote com **revisão humana** ou com avaliador-otimizador de critério escrito (Aula 04, nota 01 §4.5).

Repare que a linha decisiva é a segunda, sempre.

---

## Exercícios resolvidos

### 1. Que padrão é este?

> *"Nosso sistema lê o relatório trimestral, identifica quais indicadores merecem investigação, consulta as bases correspondentes e monta um resumo executivo."*

**Orquestrador-trabalhador.** A pista está em *identifica quais indicadores merecem investigação*: as subtarefas são decididas em execução, a partir do conteúdo do relatório. Você não consegue escrever `for indicador in indicadores` antes de chamar o modelo — que é o teste da Aula 04, nota 01, exercício 2.

E, como todo orquestrador, este precisa de teto: se o modelo decidir que 60 indicadores merecem investigação, alguém tem que impedir.

Se a frase fosse *"consulta os cinco indicadores do painel"*, seria ***sectioning*** — e mais barato.

### 2. Este case tem verificador?

> *"Um agente que triagem chamados de TI e sugere a categoria e a prioridade."*

**Tem, e é bom** — mas só se você o construir de propósito.

O verificador não vem de graça do ambiente, como no caso do teste unitário. Ele vem de um **conjunto rotulado**: chamados com a categoria e a prioridade que um humano atribuiu. Com isso, "está certo?" vira uma métrica.

É o mesmo conjunto de teste que você montou na Aula 03 para medir *few-shot* — e é o embrião do *golden dataset* da aula de evals.

**A lição para a escolha do case:** verificador **construído** conta. O que não conta é não ter nenhum. Se a resposta para "como eu sei que está certo?" for "dá para ver que está bom", o case não passa.

---

## Síntese

- **Code agents** é o tipo mais maduro (90% de uso semanal entre devs) porque o ambiente devolve **feedback verificável** — a terceira condição do agente.
- **Customer agents** operam em volume alto com autonomia baixa: são **roteador + workflow**, e a fatia resolvida "sem humano" tem definição da própria empresa.
- **Employee agents** funcionam por manter o humano como verificador com autoridade — a arquitetura mais tolerante a falha.
- **Data agents** são o território do **orquestrador-trabalhador**, e o que mais precisa de teto.
- **Creative** e **security** delimitam o espectro pela variável que decide autonomia: **o custo do erro**.
- Seis características comuns aos casos que funcionam — volume, verificador, tolerância a erro, reversibilidade, dado acessível por API e um número de sucesso.
- Nenhum *case study* atribui o resultado ao modelo. Todos atribuem à **integração**.
- **Quase nenhum "agente" de sucesso é um agente**: a autonomia fica confinada ao caminho estreito onde ela paga.

---

## Fontes e leituras

- **JetBrains — Developer Ecosystem Survey 2026** ([adoção de agentes de codificação](https://blog.jetbrains.com/research/2026/08/ai-coding-agent-adoption-2026/)) — >15.000 desenvolvedores; a melhor amostra desta aula.
- **Google Cloud — catálogo de casos reais** ([1.302 casos, 11 setores × 6 tipos](https://cloud.google.com/transform/101-real-world-generative-ai-use-cases-from-industry-leaders)) — origem dos casos nomeados desta nota, e o material que você vai navegar para escolher o seu.
- **McKinsey — The State of AI** ([2026](https://www.mckinsey.com/capabilities/tech-and-ai/our-insights/tech-forward/state-of-ai-trust-in-2026-shifting-to-the-agentic-era)) — onde a adoção se concentra por função.
- Aula 04, [nota 01](../../aula-04-arquitetura-de-agentes/notas-de-aula/01-padroes-de-arquitetura.md) — os padrões usados para ler cada caso; [nota 02 §1](../../aula-04-arquitetura-de-agentes/notas-de-aula/02-o-agente-e-o-estado.md) — as três condições do agente.
- [nota 03](03-os-casos-que-falharam.md) desta aula — a outra metade da história, e a mais instrutiva.
