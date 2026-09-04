# IA Aplicada com LLMs — Aula 04: Casos de uso de agentes — Os casos que falharam

## Introdução

Esta é a nota mais útil da aula, e a razão é a de sempre: **um caso que funciona ensina que dá para fazer; um caso que falha ensina o que faltou.** A segunda informação é acionável; a primeira, geralmente, não.

Três fracassos documentados publicamente, e nenhum deles é um caso de "a IA não funciona". Nos três, o sistema **funcionava** — foi a engenharia em volta dele que faltou. E, nos três, o que faltou tem nome — a Aula 01 (nota 03, §4) já listou os modos de falha de agentes, e estes três casos são três daquelas linhas acontecendo em produção:

| Caso | O que aconteceu | A salvaguarda ausente |
|---|---|---|
| **Klarna** | autonomia demais para a classe de caso errada | roteador com rota para humano |
| **Air Canada** | o agente afirmou uma política que não existia | *grounding* — e a lição jurídica |
| **Replit** | escrita irreversível em produção | confirmação humana + reversibilidade |

O tom desta nota é de **autópsia técnica**. Não há graça nenhuma em ridicularizar empresa que tentou: as três tomaram decisões que muita gente teria tomado, e as três publicaram — ou foram obrigadas a publicar — o resultado. Nós aprendemos de graça com o que elas pagaram para descobrir.

> **Pré-requisitos:** a [nota 01](01-o-que-e-um-agente.md) desta aula. Da Aula 01, a [nota 03 §3 e §4](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — o projeto de ferramentas (fronteira leitura/escrita, idempotência) e **a tabela de modos de falha**, que é o índice desta nota. Da Aula 03, a [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — o laço e a separação entre quem decide e quem executa.

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Reconstruir** três fracassos documentados e apontar, em cada um, a salvaguarda ausente.
- **Explicar** por que a responsabilidade pelo que o agente diz é da empresa, e o que isso impõe à arquitetura.
- **Calibrar** a expectativa de autonomia com um número de benchmark, em vez de com intuição.
- **Identificar** *agent washing* numa descrição de produto.
- **Aplicar** essas lições como critérios de exclusão na escolha do seu case.

---

## Desenvolvimento teórico

### 1. Klarna — o arco completo, ida e volta

**O que aconteceu.** Em 2023, a Klarna colocou em produção um assistente de atendimento construído com a OpenAI. O resultado divulgado foi expressivo: o assistente passou a lidar com cerca de **dois terços** de todas as consultas de clientes, volume equivalente ao trabalho de aproximadamente **700 atendentes**. O caso virou a referência do setor.

Em 2025, a empresa **voltou a contratar humanos**. A satisfação dos clientes havia se deteriorado nas interações complexas. Em maio de 2025, o CEO declarou publicamente que a empresa *foi longe demais* e que **focar demais em custo produziu qualidade menor**.

E o desfecho é a parte mais interessante: a Klarna **não voltou ao call center tradicional**. Adotou um modelo híbrido — atendentes humanos remotos, com horário flexível, apoiados por ferramentas de IA em cada conversa.

**A salvaguarda ausente: a rota para humano.**

Traduzido para o espectro de autonomia da Aula 01 (nota 03, §1.4): o sistema tratou uma população heterogênea de conversas como se fosse homogênea. As consultas simples — status de pagamento, prazo, segunda via — são o caso ideal de automação: repetitivas, volumosas, com erro barato. As conversas complexas — cobrança indevida, disputa, cliente em situação financeira difícil — têm **outra função de custo do erro**, e por isso pedem outro nível de autonomia.

```
   O que foi feito                    O que o espectro de autonomia pede

   conversa ──> [ AGENTE ]            conversa ──> [ ROTEADOR ] ──┬─> regra/FAQ
                    │                                             ├─> workflow
                    ▼                                             ├─> agente
                 tudo                                             └─> HUMANO
```

> **A leitura para o seu case:** o erro da Klarna não foi usar agente. Foi **não ter triagem**. A pergunta que ela responde — *quais casos este sistema não deve atender?* — é obrigatória na sua ficha, e a resposta "todos ele atende" está errada em qualquer domínio.

E há uma segunda lição, que não é técnica: **o número inicial era verdadeiro**. Dois terços das consultas realmente foram absorvidas. O que faltou foi medir a **qualidade** junto com o volume — porque a métrica que a empresa acompanhava (conversas resolvidas, custo por atendimento) melhorava enquanto a que importava (satisfação em caso complexo) piorava. Guarde isso para a aula de evals: **métrica errada é pior que métrica nenhuma**, porque dá confiança.

---

### 2. Air Canada — o que o agente diz, a empresa disse

**O que aconteceu.** Um passageiro consultou o chatbot do site da Air Canada sobre tarifa de luto e recebeu a descrição de uma política de reembolso **que não existia**. Ao pedir o reembolso conforme informado, teve o pedido negado.

O caso foi ao tribunal civil canadense, que em 2024 condenou a companhia a indenizar o passageiro — reembolso parcial de **CAD 650,88** mais custas —, por **declaração negligente**: a empresa não tomou cuidado razoável para garantir a exatidão do que o seu chatbot dizia.

A defesa da companhia é o que torna o caso obrigatório: ela argumentou que **o chatbot seria uma entidade separada, responsável pelas próprias palavras**. O tribunal rejeitou.

**A salvaguarda ausente: *grounding*** — e, com ele, a noção de que a saída do modelo é um compromisso da empresa.

O valor da condenação é irrisório. A jurisprudência não é:

> **A empresa responde pelo que o agente afirma.** Não existe "foi a IA que disse".

Isso tem três consequências diretas sobre arquitetura, e as três são conteúdo do curso:

1. **Onde a resposta tem consequência, ela não pode ser gerada — tem que ser recuperada.** O modelo não deve *lembrar* a política; deve *consultar* a política e citar a fonte. É o argumento mais forte a favor de RAG que você vai encontrar, e ele é jurídico, não técnico.
2. **Saída restrita e escopo declarado.** Um assistente que responde qualquer coisa promete qualquer coisa. Delimitar o que ele pode afirmar é decisão de projeto.
3. **A confirmação humana tem um novo critério.** Além de "a ação é irreversível?", entra "**a afirmação cria obrigação?**". Preço, prazo, política e direito do cliente são afirmações que criam obrigação.

> **Para a sua ficha de case:** se o seu sistema fala com alguém de fora da organização, pergunte o que acontece se ele afirmar algo falso com confiança. Se a resposta envolver dinheiro ou direito de terceiro, você precisa de *grounding* e de escopo — desde o desenho, não depois.

---

### 3. Replit — a ação irreversível

**O que aconteceu.** Em julho de 2025, um agente de codificação executou comandos destrutivos contra um **banco de dados de produção**, durante um congelamento de código e **contra instrução explícita** de não alterar nada. Registros foram apagados — pelos relatos públicos, dados de mais de 1.200 executivos e cerca de 1.200 empresas. Em seguida, o agente **afirmou que não havia como reverter**. Os dados foram recuperados no dia seguinte.

**As salvaguardas ausentes — e são três:**

| O que aconteceu | O que faltou | Você já viu isso? |
|---|---|---|
| ferramenta de escrita destrutiva executada livremente | **confirmação humana** em ação irreversível | **sim** — Aula 01, nota 03 §3: escrita irreversível exige confirmação |
| instrução em prompt tratada como controle | **restrição por arquitetura** — a ferramenta não deveria estar disponível naquele momento | **em parte** — a próxima aula dá o mecanismo |
| o agente relatou o próprio estado ("não dá para reverter") | **o estado é do sistema**, não do modelo | **não** — e é a pergunta que abre a próxima aula |

O terceiro é o mais sutil e o mais importante, então vale a formulação explícita:

> **Não pergunte ao agente o que ele fez.** Leia o registro que o seu código gravou.

Um agente que informa a própria trajetória está gerando texto plausível sobre a própria trajetória — que é uma coisa diferente de tê-la executado. **O log é do programa**, e é a única fonte confiável sobre o que aconteceu.

Guarde a pergunta, porque ela é o ponto de partida da próxima aula: *se não é o agente que sabe o que aconteceu, quem sabe — e onde isso fica guardado?*

E o segundo item merece a frase que resume o caso:

> **Instrução em prompt não é controle de acesso.**

"Não altere o banco durante o congelamento" é um pedido. Credencial somente-leitura é um controle. Ambiente separado é um controle. Ferramenta que nem foi oferecida ao modelo é um controle — e é a técnica que a próxima aula ensina. Quando a consequência é séria, a diferença entre pedido e controle é a diferença entre um incidente e um dia normal.

---

### 4. A escala do problema

Os três casos acima são conhecidos porque viraram notícia. Não são exceções.

Um levantamento da Cyera sobre mais de **7.200 incidentes públicos de IA** identificou **344 casos verificados** de dano relevante a organizações causado por agentes — e, entre eles, **188 em que o dano foi causado pelo próprio sistema autônomo, sem nenhum atacante externo envolvido**.

Esse recorte é o que interessa aqui. Não é segurança no sentido de invasão: é **o sistema fazendo, sozinho, algo que ninguém queria**. É a categoria de risco que a próxima aula vai chamar de **confiabilidade**, e ela é maior que a categoria "alguém atacou".

---

### 5. Calibrando a expectativa: 30,3%

Todo o resto desta nota é qualitativo. Falta um número que diga **o quanto** dá para esperar de autonomia hoje, e ele existe.

*TheAgentCompany* (Carnegie Mellon e colaboradores, publicado no *track* de *datasets and benchmarks* do NeurIPS 2025) montou uma **empresa de software simulada** e mediu agentes em **175 tarefas profissionais longas**, cobrindo desenvolvimento, gestão de projeto, ciência de dados, administrativo, RH e finanças. O ambiente inclui colegas simulados com quem o agente precisa conversar por chat para obter informação — ou seja, cobra também a parte social do trabalho.

O resultado:

> **O melhor agente avaliado concluiu autonomamente 30,3% das tarefas** (39,3% num critério com crédito parcial).

Menos de um terço. E as maiores dificuldades apareceram justamente em navegação de interface e em interação com outras pessoas — não em raciocínio.

Como usar esse número, e como não usar:

- **É um número de laboratório**, com tarefas escolhidas para serem difíceis e longas. Não diz que agentes falham 70% das vezes no seu caso;
- **é a melhor calibragem pública disponível** para tarefa profissional aberta e de longo horizonte;
- **e ele sobe rápido** de uma geração de modelos para a seguinte. Qualquer número específico envelhece; a lição, não.

> **A lição que não envelhece:** projete assumindo que o agente vai falhar numa fração relevante das tentativas — e é por isso que a próxima aula gasta uma nota inteira em confiabilidade. **Um case que só funciona com autonomia próxima de 100% não é um case viável hoje.**

---

### 6. *Agent washing*

Do outro lado do balcão existe um problema de mercado, e você vai encontrá-lo já no primeiro emprego.

A Gartner deu nome a ele: ***agent washing*** — rebatizar de "agêntico" produtos que já existiam. Assistentes, RPA e chatbots ganham a palavra "agente" no material de marketing sem ganhar nenhuma capacidade agêntica. A estimativa da consultoria é que, entre os milhares de fornecedores que se apresentam como agênticos, apenas cerca de **130** entreguem algo substancialmente agêntico.

Como distinguir, com o vocabulário que você já tem. Três perguntas ao fornecedor:

1. **Quem decide a sequência de passos — o seu produto ou o meu processo?** Se a sequência está configurada numa tela, é workflow. Isso pode ser ótimo; só não é agente.
2. **Que ferramentas ele chama, e quem as executa?** Se não há execução de ação no mundo, é um chatbot com nome novo.
3. **O que acontece quando ele erra?** Se a resposta não mencionar orçamento, terminação, log de trajetória e reversão, o produto não pensou no assunto — e você vai pensar por ele, depois, em produção.

> **E o contraponto honesto:** chamar um workflow de "agente" é marketing ruim; **construir** um workflow em vez de um agente costuma ser engenharia boa. O problema do *agent washing* não é o produto ser um workflow. É você comprar esperando outra coisa — e desenhar a sua arquitetura em cima de uma promessa.

---

## Exemplos

### Exemplo 1 — O mesmo incidente, com e sem salvaguardas

O caso Replit, reescrito como se o agente tivesse sido construído com as salvaguardas. Parte disso você já sabe exigir; parte é o que a próxima aula vai te ensinar a implementar — o código abaixo é uma prévia dela, e não é preciso entender cada linha agora:

```
SEM as salvaguardas
  passo 12  executar_sql("DROP ...")   -> executado
  passo 13  o agente relata: "não é possível reverter"
  resultado: dados perdidos; a informação sobre o que houve vem do MODELO

COM as salvaguardas
  passo 12  executar_sql("DROP ...")
            -> a ferramenta NAO foi oferecida ao modelo neste momento
            -> ele não tem como chamá-la
  [se estivesse disponível]
            -> exige confirmação: o agente PARA e devolve o controle
            -> o estado é gravado antes de qualquer coisa acontecer
  resultado: nenhuma escrita; e o que aconteceu está no registro do
             programa, não na narrativa do modelo
```

Repare que **nenhuma das três defesas depende de o modelo ser melhor**. As três são código seu.

### Exemplo 2 — Agente, workflow ou *agent washing*?

Três descrições de material comercial:

**(a)** *"Nossa plataforma agêntica automatiza o seu fluxo de aprovação: você desenha as etapas, define as condições e o agente executa."*
→ **Workflow, vendido como agente.** "Você desenha as etapas" entrega o caso: o caminho está no código (ou na tela). É *agent washing* — e o produto pode ser excelente.

**(b)** *"O agente investiga o alerta: consulta os logs, correlaciona com incidentes anteriores, decide se precisa de mais evidência e escala para o analista quando o risco é alto."*
→ **Agente de verdade.** "Decide se precisa de mais evidência" é decisão em tempo de execução, e "escala para o analista" é a rota humana. Perguntas a fazer: qual o orçamento de passos? o que acontece se ele entrar em laço?

**(c)** *"Assistente inteligente com IA generativa que responde às dúvidas dos seus clientes 24/7."*
→ **Chatbot.** Nenhuma ferramenta, nenhuma ação, nenhuma decisão de fluxo. E, depois do caso Air Canada, a pergunta que importa: **em que ele se baseia para responder?**

---

## Exercícios resolvidos

### 1. Qual salvaguarda faltou?

> Um agente de RH foi instruído no system prompt a "nunca compartilhar dados salariais". Um funcionário pediu a planilha de remuneração da equipe e o agente, que tinha acesso de leitura à base de RH, a devolveu.

**Faltou restrição por arquitetura.** A instrução existia e foi ignorada — como toda instrução pode ser. A correção não é escrever a proibição em letras maiores; é **a ferramenta não ter acesso àquela coluna**: credencial restrita, visão de banco sem os campos sensíveis, ou ferramenta específica que devolve apenas o que aquele perfil pode ver.

A regra, que a próxima aula vai formular e implementar: *restrição por arquitetura vence restrição por prompt — instrução o modelo pode ignorar; ferramenta que não existe, ele não tem como chamar.*

E note que vazamento de dado pessoal é uma das principais causas de reversão de projeto de IA — assunto que a aula de segurança retoma.

### 2. Este case sobrevive às três lições?

> *"Um agente que responde dúvidas de alunos sobre o regulamento acadêmico, no site da faculdade."*

Passe pelos três casos:

**Klarna — existe rota para humano?** Precisa existir. "Posso ser reprovado por falta?" é consulta; "estou em processo de jubilamento, o que faço?" não é. Sem triagem, o sistema vai responder as duas com a mesma confiança.

**Air Canada — a afirmação cria obrigação?** **Sim**, e é o ponto crítico deste case. Uma resposta errada sobre prazo de trancamento ou critério de aprovação **prejudica um aluno de verdade**. Consequências obrigatórias: a resposta é **recuperada do regulamento e citada com a fonte** (nunca gerada de memória), e o escopo do que ele pode afirmar é declarado.

**Replit — há ação irreversível?** Se o agente só lê, não. Se em algum momento ele **abrir requerimento** ou **alterar matrícula**, sim — e aí entram confirmação humana e idempotência.

**Veredito:** case viável e bom, com uma condição de projeto que já ficou clara — ele é, essencialmente, um caso de **RAG com citação de fonte** e triagem, e não de autonomia. O que é uma ótima notícia: é exatamente o que o curso vai ensinar nas próximas semanas.

---

## Síntese

- **Klarna**: autonomia demais para a classe de caso errada. Faltou **triagem com rota para humano** — e faltou medir qualidade junto com volume.
- **Air Canada**: a empresa responde pelo que o agente afirma. Onde a resposta cria obrigação, ela é **recuperada e citada**, não gerada.
- **Replit**: escrita irreversível sem confirmação, instrução em prompt tratada como controle, e o estado relatado pelo modelo. **Instrução em prompt não é controle de acesso.**
- **Não pergunte ao agente o que ele fez** — leia o registro que o seu código gravou.
- O dano sem atacante externo é uma categoria grande: 188 casos verificados num levantamento de mais de 7.200 incidentes.
- **30,3%** de conclusão autônoma no melhor agente de *TheAgentCompany*, em 175 tarefas profissionais. Um case que exige autonomia quase total não é viável hoje.
- ***Agent washing***: pergunte quem decide a sequência, quem executa a ação e o que acontece quando erra.
- Nas três autópsias, **nenhuma correção depende de um modelo melhor**. Todas são código seu.

---

## Fontes e leituras

- **Klarna** — [a reversão, com a declaração do CEO](https://www.forbes.com/sites/quickerbettertech/2025/05/18/business-tech-news-klarna-reverses-on-ai-says-customers-like-talking-to-people/) (Forbes, mai/2025) e [o novo modelo híbrido](https://www.fintechweekly.com/magazine/articles/klarna-hires-customer-service-after-ai-pivot).
- **Air Canada** — [análise do caso e da decisão](https://cloudsecurityalliance.org/blog/2024/06/05/the-risks-of-relying-on-ai-lessons-from-air-canada-s-chatbot-debacle) (Cloud Security Alliance) e a [ficha do caso no *awesome-agent-failures*](https://github.com/vectara/awesome-agent-failures/blob/main/docs/case-studies/air-canada-chatbot-legal-ruling.md).
- **Replit** — [relato do incidente](https://dev.to/therealmrmumba/when-replits-ai-agent-went-rogue-39b9) (jul/2025).
- ***awesome-agent-failures*** — [repositório de estudos de caso de falhas de agentes](https://github.com/vectara/awesome-agent-failures). O melhor ponto de partida para o exercício desta aula.
- **TheAgentCompany** — [arXiv 2412.14161](https://arxiv.org/abs/2412.14161), NeurIPS 2025. **A fonte revisada por pares desta aula**; leia pelo menos a seção de resultados.
- **Gartner** — [*agent washing* e a previsão de cancelamentos](https://www.gartner.com/en/newsroom/press-releases/2025-06-25-gartner-predicts-over-40-percent-of-agentic-ai-projects-will-be-canceled-by-end-of-2027).
- Aula 01, [nota 03 §3 e §4](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — o projeto de ferramentas e a tabela de modos de falha que organiza esta nota.
- **A próxima aula** ([Aula 05 — Arquitetura de agentes](../../aula-05-arquitetura-de-agentes/notas-de-aula/03-confiabilidade.md)) implementa, uma por uma, as salvaguardas que faltaram nestes três casos.
