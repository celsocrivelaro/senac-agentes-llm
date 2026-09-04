# IA Aplicada com LLMs — Aula 04: O que é um agente, e onde eles estão

## Introdução

Você já construiu um laço de agente. Declarou ferramentas, leu a chamada devolvida pelo modelo, executou e devolveu o resultado. Sabe fazer a coisa funcionar.

O que você não tem é **repertório** — e é o que esta nota entrega. Ela responde a quatro perguntas, nesta ordem:

1. **O que é um agente**, afinal — e o que ele *não* é. Porque "bot", "assistente de IA" e "agente de IA" aparecem trocados entre si em quase todo material comercial, e a diferença entre eles é o que decide se um caso interessa a esta disciplina.
2. **Que tipos existem**, na classificação que o mercado usa — e que nomes de arquitetura você vai encontrar ao ler sobre eles.
3. **Para que serve** — quais são os eixos de ganho, e como cada um se mede. É o que você vai precisar para justificar o seu tema.
4. **Onde eles estão de verdade**, e quais funcionam — com empresa nomeada, número divulgado e a arquitetura provável por trás de cada caso.

A última parte é a maior, e vem com uma advertência de leitura. Empresa nenhuma publica o diagrama de arquitetura junto com o *case study*; o que se publica é o destinatário e o resultado. A leitura de arquitetura é **inferência**, e às vezes vai estar errada. Fazê-la mesmo assim é o exercício — usar o vocabulário que você tem sobre descrição de marketing é exatamente a habilidade de que vai precisar quando alguém te trouxer um requisito começando com *"queria um agente que…"*.

E há um padrão que aparece cedo e se repete até o fim:

> **Quase nenhum "agente" de sucesso é um agente.** A maioria é roteador com workflow, e a autonomia fica confinada a um caminho estreito. Isso não é decepcionante — é a régua do espectro de autonomia da Aula 01 acontecendo no mundo: **use a menor autonomia que resolve**.

> **Pré-requisitos:** da Aula 01, a [nota 03 §1 e §3](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — a definição de agente, o **espectro de autonomia** e o projeto de ferramentas. Da Aula 03, a [nota 04](../../aula-03-prompt-engineering/notas-de-aula/04-tool-calling.md) — tool calling e o laço, que é o mecanismo por baixo de todo caso que você vai ler.

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Distinguir** bot, assistente de IA e agente de IA, e dizer qual eixo separa os três.
- **Reconhecer** que a lista de características de um agente é de **capacidades**, não de requisitos.
- **Usar** a taxonomia dos seis tipos de agente para navegar o mercado, e nomear os padrões de arquitetura que vai encontrar.
- **Dizer** onde a adoção de agentes está concentrada, e por quê.
- **Nomear os eixos de ganho** de um agente — produtividade, personalização, decisão, velocidade de processo, erro — e dizer **como cada um se mede**.
- **Citar** casos reais em produção nos seis tipos, com o número divulgado e a fonte.
- **Inferir** a arquitetura provável de um caso a partir da descrição pública dele.
- **Explicar** por que *code agents* é o tipo mais maduro, usando a ideia de **feedback verificável**.
- **Enumerar** as seis características comuns aos casos que funcionam, e reconhecer a ausência delas num caso novo — inclusive no seu.

---

## Desenvolvimento teórico

### 1. Três palavras que o mercado usa como sinônimo

Antes de ler qualquer caso, vale acertar o vocabulário — porque **bot**, **assistente de IA** e **agente de IA** aparecem trocados entre si em quase todo material comercial, e a diferença entre eles é justamente o que decide se um caso interessa a esta disciplina.

A distinção mais usada é a do Google Cloud, e ela se organiza em três eixos:

| | **Bot** | **Assistente de IA** | **Agente de IA** |
|---|---|---|---|
| **Autonomia** | mínima — segue regras pré-programadas | média — **depende de comando** e direção do usuário | **máxima** — opera e decide por conta própria para atingir um objetivo |
| **Complexidade** | interações simples | tarefas simples | tarefas e **fluxos complexos** |
| **Aprendizado** | pouco ou nenhum | alguma capacidade | adapta e melhora ao longo do tempo |
| **Iniciativa** | reage a um gatilho | reage a um pedido | pode ser **proativo** — roda em segundo plano, orientado a evento |

O eixo que faz o trabalho é o primeiro. **Autonomia** é a mesma régua do espectro da Aula 01 (nota 03, §1.4) — e a definição de agente que você já tem, com objetivo, ferramentas e laço (§1.2), é a ponta de cima desta tabela.

O quarto eixo é novo para você e vale reter, porque muda o desenho do sistema: um agente **reativo** só age quando alguém pede; um agente **proativo** vive em segundo plano e é acionado por evento — o que significa que alguém precisa decidir *quando* ele acorda, e o custo passa a existir mesmo quando nenhum usuário está olhando.

#### 1.1 As características que a indústria atribui a um agente

A mesma fonte enumera o que caracteriza um agente:

| Característica | O que é | Onde você vê no curso |
|---|---|---|
| **Raciocínio e planejamento** | usar lógica e a informação disponível para concluir e decidir o próximo passo | Aula 03 (CoT) e o laço da nota 04 |
| **Memória** | manter contexto e aprender com as interações — inclusive **memória de longo prazo**, entre sessões distintas | Aula 03, nota 02 (contexto) e a aula de memória |
| **Uso de ferramentas** | interagir com sistemas e fontes de dados externas | Aula 03, nota 04 (tool calling) |
| **Autonomia** | decidir e agir sem comando a cada passo | Aula 01, nota 03 e a próxima aula |
| **Multimodalidade** | processar texto, voz, imagem, vídeo, código | recomendação do trabalho, quando o caso pedir |
| **Coordenação** | trabalhar com outros agentes em fluxos mais complexos | Parte 3 do trabalho |

Duas leituras desta tabela, e a segunda é mais útil que a primeira.

**A primeira:** ela é um bom mapa do curso. Cinco das seis características são assunto de uma aula específica, e você já viu três.

**A segunda:** ela é uma lista de **capacidades**, não de requisitos. Nenhum sistema real tem as seis, e **um sistema não precisa das seis para ser um agente** — a Aula 01 definiu agente com quatro componentes, e "memória de longo prazo" não está entre eles. Listas assim servem para conversar com o mercado, e não para decidir se o que você construiu conta.

> **Por que isso abre a aula:** porque a partir do próximo parágrafo você vai ler material de mercado, e nele os três termos são intercambiáveis. Quando um fornecedor chama de "agente" algo que segue regras pré-programadas, ele está usando a palavra do jeito que a tabela acima proíbe — e isso tem nome, ***agent washing***, tratado na [nota 03 §6](02-os-casos-que-falharam.md).

E uma ressalva de coerência, porque a página que acabamos de usar é de um fornecedor de nuvem: **explicador de tema serve para vocabulário, não serve para evidência.** Definição e taxonomia é exatamente o que este gênero de texto faz bem. O que ele não faz é provar que alguém implantou algo — e é essa diferença que o [§3.1](#31-o-outro-modo-de-falha-capacidade-apresentada-como-adocao) trata.

---

---

### 2. A taxonomia que você vai encontrar

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

Note que **é uma taxonomia por destinatário, não por arquitetura**. Ela diz *para quem* o agente trabalha, e não *como* ele é feito. Um *customer agent* pode ser um roteador simples ou um agente autônomo — e a §5 mostra que, na prática, quase sempre é o primeiro.

Isso importa para você por um motivo bem concreto: **o material do mercado descreve o destinatário e omite a arquitetura**. Ler a arquitetura por trás da descrição é trabalho seu — e o vocabulário para isso é o da próxima seção.

---

### 2.1 Os nomes de arquitetura que você vai encontrar

A Aula 01 (nota 03, §1.4) deu o **espectro de autonomia** em seis níveis: prompt único, *chain*, roteador, workflow com ferramentas, agente e multi-agente. Aquele espectro já resolve a maior parte da leitura de caso, e a régua dele é a que importa:

> **Workflow**: o caminho está no código. **Agente**: o caminho é decidido em tempo de execução pelo modelo.

O mercado, porém, usa mais três nomes que aquele espectro não cobriu, e você vai topar com eles. Ficam aqui em nível **descritivo**, o suficiente para ler um caso — a próxima aula abre cada um em código:

| Nome | O que é | Como reconhecer num caso |
|---|---|---|
| **Paralelização** | a mesma tarefa dividida em partes independentes (*sectioning*), ou repetida N vezes com voto majoritário (*voting*) | menção a processar em lote, ou a "consenso" entre respostas |
| **Orquestrador-trabalhador** | um modelo **decide quais são as subtarefas**, outros as executam, um terceiro sintetiza | a decomposição depende do conteúdo: ninguém sabia de antemão quantas partes seriam |
| **Avaliador-otimizador** | um laço curto de crítica: gerar → avaliar contra critério → revisar | menção a revisão automática, a "qualidade garantida", a segunda passada |

E a distinção que mais vale guardar, porque separa dois padrões que parecem iguais:

> **Paralelização (*sectioning*)**: as partes estão no **seu código**.
> **Orquestrador-trabalhador**: as partes são decididas pelo **modelo**, em execução.

Com o espectro da Aula 01 mais estes três nomes, você tem o suficiente para os §5 a §9 — onde cada caso real é lido pela arquitetura provável dele.

---

---

### 3. Onde a adoção realmente está

Somando as fontes, o mapa é razoavelmente consistente:

**Por função** — a adoção se concentra em **TI, gestão de conhecimento e engenharia de software**. Não é coincidência que sejam as três funções onde quem decide a compra é também quem usa a ferramenta, e onde o resultado é verificável por quem está na sala.

**Por setor** — bancos e seguros à frente; saúde e governo bem atrás. Duas explicações se somam: dinheiro disponível e tolerância regulatória. Onde o erro é catastrófico ou o dado é protegido por lei, a adoção é mais lenta — e isso é racional, não atraso.

**Por tipo de empresa** — empresas de tecnologia usam agentes de forma muito mais disseminada que as demais. Organizações pequenas ficaram praticamente paradas. Agente ainda é, majoritariamente, coisa de empresa grande e de empresa de tecnologia.

**E a maturidade de governança não acompanhou:** apenas cerca de um terço das organizações relata maturidade razoável em estratégia e governança de IA agêntica. Guarde esta frase: a governança imatura é a causa por trás dos três fracassos da [nota 02](02-os-casos-que-falharam.md).

---

---

### 4. Para que serve: os benefícios, e como eles aparecem

Antes de percorrer os casos, vale nomear **o que se espera ganhar**. Porque "agentes são o futuro" não é um benefício, e a pergunta que qualquer chefe vai fazer — *o que isso me dá?* — tem respostas concretas, em eixos distintos.

A classificação mais usada é a do Google Cloud, e ela tem três eixos:

**Produtividade.** É o eixo mais citado e o mais fácil de medir. Agentes de funcionário tiram do caminho a tarefa repetitiva: respondem à pergunta interna, preenchem o formulário, traduzem e revisam o comunicado. No *2025 ROI of AI Report* da própria Google Cloud, **74%** dos executivos relatam ROI no primeiro ano e — o número mais chamativo — entre os que relatam ganho de produtividade, **39% viram a produtividade pelo menos dobrar**.

**Personalização.** Agentes de cliente entendem a necessidade, respondem, resolvem e recomendam, no canal que o cliente escolheu. O ganho aparece como satisfação e conversão, não como hora economizada.

**Decisão.** Agentes que decidem em vez de sugerir, onde a decisão autônoma cria valor imediato: resolução de atendimento, otimização de estoque, priorização de fila.

E há dois eixos que a lista de fornecedor costuma diluir, mas que na prática vendem melhor:

**Velocidade de processo.** Não é fazer mais no mesmo tempo — é o mesmo trabalho terminando antes. A mesma fonte cita marketing **32% mais rápido** na edição e **46%** na criação de conteúdo, e operações de segurança com o tempo médio de resposta a ameaça caindo **pela metade**.

**Redução de erro e de retrabalho.** O menos anunciado e o mais valioso em processo regulado, onde o custo do erro não é o tempo de refazer — é a multa, a devolução ou o cliente perdido.

#### 4.1 O benefício que ninguém anuncia

Para construir um agente, você é obrigado a **escrever a regra**. Qual é o teto de reembolso, o que caracteriza atraso, quando se escala para o humano, o que é caso duvidoso.

Em boa parte dos casos, esse exercício descobre que a regra nunca esteve escrita em lugar nenhum — vivia na cabeça de três pessoas experientes, cada uma com uma versão. **Uma fração relevante do ganho vem de ter finalmente documentado o processo**, e não do modelo.

Isso tem uma consequência prática que você vai reencontrar no seu trabalho: **parte do benefício é obtida sem LLM nenhum.** A rota de regra de um roteador — o `if` que resolve o caso claro — costuma responder pela maioria do volume, e ela só existiu porque alguém precisou escrever a regra para o agente.

#### 4.2 Como cada eixo se mede

Esta tabela é o que você vai usar para justificar o seu tema, na Parte 1 do trabalho:

| Eixo de ganho | Como se mede | Exemplo com número, desta nota |
|---|---|---|
| **Produtividade** | itens tratados por pessoa por dia; horas devolvidas por semana | United Wholesale Mortgage: produtividade de analistas **mais que dobrada** em 9 meses |
| **Cobertura / contenção** | % dos casos resolvidos sem humano | Commerzbank: **70%** de ~2 milhões de conversas |
| **Redução de carga** | fila que deixou de chegar à pessoa | IBM AskHR: **−75%** de tickets desde 2016 · LUXGEN: **−30%** de carga nos atendentes |
| **Velocidade de processo** | tempo do início ao fim de um caso | AdVon: catálogo de 93.673 produtos em menos de um mês |
| **Eficiência de operação** | custo ou esforço por transação | Moglix: **4×** em eficiência de *sourcing* |
| **Receita** | conversão, ticket, posição de busca | AdVon: **+67%** em vendas médias diárias para um cliente |
| **Erro e retrabalho** | taxa de erro antes × depois; horas de correção | o eixo mais escasso em número público — e o mais convincente quando você tem |

Repare na última linha. **Quase nenhum caso publica redução de erro**, e não é porque não acontece: é porque medir exige ter medido *antes*, e quase ninguém mediu. Se o seu tema permitir esse número, você tem o argumento mais forte da mesa.

#### 4.3 A ressalva, em uma frase

Todos os números desta seção — e da tabela — são **auto-relatados** por quem vendeu ou comprou o sistema, quase sempre sem linha de base publicada. "4× de eficiência" em relação a quê? Medido como?

Eles servem para você saber **quais eixos de ganho existem**, e para calibrar ordem de grandeza. Não servem como promessa, e principalmente não servem como a *sua* estimativa: essa você vai ter que construir com o número da sua própria operação.

---

### 5. Code agents — o caso mais maduro, e o porquê importa mais que o quanto

Comece pelo número, que é o mais sólido desta aula porque vem da maior amostra: a *Developer Ecosystem Survey* da JetBrains, com mais de **15.000 desenvolvedores profissionais** (coleta entre maio e julho de 2026, oito idiomas, cotas regionais):

- **90%** usam agentes de codificação no trabalho **pelo menos semanalmente**;
- **68%** usam **diariamente**.

Nenhum outro tipo de agente chega perto disso. E o mercado de ferramentas se reorganizou rápido no período: Claude Code em 39% de adoção global (47% nos EUA), GitHub Copilot caindo de 29% para 21%, Codex saltando de 3% para 16%, Cursor recuando de 18% para 12%.

A pergunta interessante não é *quanto*. É **por que aqui primeiro?**

A resposta é a coisa mais importante desta aula, e ela não estava nos números — está na natureza do domínio:

> **O ambiente devolve feedback verificável.**

Em software, o teste passa ou falha. O compilador aceita ou recusa. O *linter* aponta a linha. O programa roda ou estoura. É um ambiente que **contradiz o modelo em segundos, de graça, sem ambiguidade** — e um agente só se corrige onde existe algo capaz de dizer que ele errou.

Compare com "escrever um texto persuasivo": nenhuma ferramenta consegue dizer se convenceu. O laço não tem do que se corrigir; ele só acumula passos com a confiança intacta.

Guarde esta ideia com o nome, porque a próxima aula a promove a **condição de projeto**: um agente só se justifica onde o ambiente é capaz de contradizê-lo. Aqui ela é uma constatação sobre o mercado; lá ela vira um critério para decidir se você deve construir um agente.

> **A lição para o seu case, e é a mais importante desta nota:** procure o verificador **antes** de procurar o problema. Se o domínio que você escolheu tem um "teste que passa ou falha", você escolheu um domínio onde agentes funcionam hoje. Se não tem, você vai passar o semestre sem saber se o seu sistema está certo — e o professor também não vai saber.

*Arquitetura provável:* **agente de verdade**, com laço longo. É um dos poucos tipos onde a autonomia se justifica, e não por acaso é o tipo em que ela funciona.

---

### 6. Customer agents — volume alto, autonomia baixa

O tipo mais visível, e o mais mal compreendido.

**Commerzbank** opera cerca de **2 milhões de conversas**, com **70%** resolvidas sem intervenção humana.
**NoBroker** processa cerca de 10.000 horas de gravação por dia, com agentes cobrindo **25% a 40%** das chamadas.
**LUXGEN** reduziu em **30%** a carga de trabalho dos atendentes.

Repare no formato dos três números: nenhum é 100%, e nenhum pretende ser. **Todos descrevem uma fatia.**

*Arquitetura provável:* **roteador + workflow**, com o agente reservado ao caso difícil. A triagem manda a maioria para um caminho barato e previsível, e a autonomia cara fica para a minoria que precisa dela — e é exatamente o desenho que vocês vão construir no laboratório da próxima aula.

E aqui vale antecipar a conta que a próxima aula vai medir com script: a rota mais valiosa de um roteador é a que **não chama o modelo**. Quando um banco diz que resolve 70% das conversas automaticamente, boa parte desses 70% é consulta de saldo respondida por regra — não é um agente raciocinando sobre finanças pessoais.

**O cuidado com a definição.** "Resolvida sem humano" é régua da própria empresa. Costuma significar *a conversa terminou sem transbordo para atendente* — o que inclui o cliente que desistiu e foi embora. Ao ler qualquer número de atendimento, procure a definição de "resolvido"; se ela não estiver publicada, o número vale menos.

---

### 7. Employee agents — assistência, não substituição

**IBM**, com o **AskHR**, é o caso mais bem documentado deste tipo: cerca de 90 automações de RH — cartas, férias, folha, alterações de remuneração — atendendo **mais de 11,5 milhões de interações em 2024** (e mais de 16 milhões de mensagens em 2025), com **94% de contenção**, mais de um milhão de transações concluídas, **75% menos tickets de suporte** desde 2016 e adoção de 99% entre gestores.
**United Wholesale Mortgage** relata ter **mais que dobrado** a produtividade de analistas de crédito em nove meses.
**Uber** usa sumarização de comunicações com usuários como ferramenta interna.
**Intuit**, no TurboTax, preenche automaticamente os dez formulários fiscais mais comuns dos EUA.

**E o AskHR merece a mesma desconfiança do §2**, por dois motivos. O primeiro: é a IBM falando da IBM — fornecedor de plataforma de agentes relatando o próprio uso. O segundo é mais instrutivo: **"94% de contenção"** é exatamente a métrica de régua própria contra a qual esta nota já avisou. *Contido* significa *a conversa não escalou para um humano* — e inclui o funcionário que desistiu e foi resolver por outro canal. O número é útil e não é o que parece.

O padrão comum aos quatro é o que **não** aparece na manchete: **o humano continua decidindo**. O analista de crédito não foi substituído — ele passou a receber o caso pré-analisado. O formulário preenchido é conferido antes de ser enviado.

*Arquitetura provável:* **workflow com ferramentas**, ou *chaining* com portão. Em quase nenhum deles o modelo escolhe livremente a sequência.

> **Por que este tipo dá certo com tanta frequência:** o humano no fim da linha é um **verificador**, e um verificador com autoridade. O erro do sistema custa alguns segundos de conferência, e não um prejuízo. É a arquitetura mais tolerante a falha que existe — e por isso a mais fácil de colocar em produção.

Se o seu case couber aqui, ele é mais fácil de defender e mais fácil de terminar.

---

### 8. Data agents — o território do orquestrador

**Geotab** analisa dados de **4,7 milhões de veículos**, com bilhões de pontos por dia.
**Moglix** relata **4×** de melhoria em eficiência de *sourcing*.
**Domina** cita 80% de melhoria em acesso a dados e 15% em efetividade.
**BMW** usa milhares de simulações em gêmeos digitais para otimizar distribuição.

*Arquitetura provável:* **orquestrador-trabalhador**. É o tipo em que a decomposição realmente depende do conteúdo — quantas consultas, contra que fontes e em que ordem só se sabe depois de ver o primeiro resultado.

E é também o tipo que mais precisa de **teto**: um orquestrador solto sobre um banco grande é a forma mais rápida conhecida de transformar uma pergunta em uma fatura. Quem decide quantas subtarefas existem é o modelo, e alguém precisa limitar — é uma das primeiras coisas que a próxima aula implementa.

**A ressalva de leitura.** Números como "4× de eficiência" e "+80% de acesso" são **autodeclarados, sem linha de base publicada**. 4× em relação a quê? Medido como? Eles servem para saber que o tipo funciona, não para prometer o mesmo ganho a ninguém.

---

### 9. Creative e security — os dois extremos da tolerância a erro

Vale colocá-los lado a lado, porque juntos delimitam o espectro.

**Creative agents** — a **AdVon Commerce** processou um catálogo de 93.673 produtos em menos de um mês, com ganhos relatados de 30% em posições de topo de busca e 67% em vendas médias diárias para um cliente.
Aqui, errar é barato: um texto de produto ruim é corrigido, e ninguém é prejudicado. A revisão humana é o padrão, e o volume é o que justifica.

**Security agents** — o outro extremo. O falso positivo custa o tempo de um analista escasso; o falso negativo custa um incidente. É o tipo com a **menor** tolerância a erro dos seis, e por isso o mais conservador em autonomia: quase sempre triagem e enriquecimento, com a ação de contenção exigindo aprovação.

> **A variável que os separa não é a tecnologia. É o custo do erro.** Quanto mais caro o erro, menor a autonomia que se aceita — independentemente do que o modelo é capaz de fazer. É a mesma régua do espectro de autonomia da Aula 01 (nota 03 §1.4), agora com a consequência do erro entrando na conta.

---

### 10. O que os casos que funcionam têm em comum

Olhando os seis tipos de uma vez, o padrão é consistente. Esta tabela é o que você leva desta nota para a escolha do tema do trabalho, onde ela vira critério:

| Característica | Por que é necessária | Onde aparece |
|---|---|---|
| **Tarefa repetitiva, volume alto** | é onde a economia existe; tarefa rara não paga a engenharia | 11,5 milhões de interações, 2 milhões de conversas, 93 mil produtos |
| **Feedback verificável** | sem ele o agente não se corrige (§5 desta nota) | o teste que passa ou falha, nos *code agents* |
| **Tolerância a erro pequeno** | erro vai acontecer numa fração relevante | texto de produto sim; contenção de incidente não |
| **Ação reversível — ou confirmação humana** | Aula 01, nota 03 §3 — a fronteira leitura/escrita | o analista que confere antes de aprovar |
| **Dado acessível por API** | *"o difícil não é a inteligência, é o acesso confiável aos sistemas de produção"* | todos, sem exceção |
| **Um número que define sucesso** | sem critério, não dá para saber se funcionou | 70% resolvidas, 4× sourcing, −30% de carga |

E o que **não** aparece na lista, apesar de dominar a conversa pública: qual modelo foi usado, qual framework, quantos parâmetros. Nenhum *case study* atribui o resultado ao modelo. Todos atribuem à **integração** — e é ela, não a capacidade do modelo, que os estudos de adoção apontam como o gargalo.

---

### 11. A frase que resume a nota

Levantamentos de mercado convergem para a mesma formulação, e ela vale ser guardada:

> **O mais difícil hoje não é a inteligência do modelo. É o acesso seguro e confiável aos sistemas de produção.**

Isso deveria mudar a sua estimativa de esforço no trabalho da disciplina. A parte que parece o trabalho — o prompt, o laço, a escolha do modelo — você já sabe fazer, e ela é a menor. A parte que decide se funciona é ferramenta confiável, dado acessível e salvaguarda. Que é, não por acaso, o assunto da próxima aula inteira.

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

Arquitetura provável, desenhada com o vocabulário da §2.1:

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

Os dois são casos de uso legítimos e economicamente reais — a AdVon é o segundo, em escala. Mas só o primeiro é **agente**: no segundo, o laço não tem como se corrigir, e a arquitetura certa é geração em lote com **revisão humana** ou com um **avaliador-otimizador** de critério escrito (§2.1).

Repare que a linha decisiva é a segunda, sempre.

---

## Exercícios resolvidos

### 1. Que padrão é este?

> *"Nosso sistema lê o relatório trimestral, identifica quais indicadores merecem investigação, consulta as bases correspondentes e monta um resumo executivo."*

**Orquestrador-trabalhador.** A pista está em *identifica quais indicadores merecem investigação*: as subtarefas são decididas em execução, a partir do conteúdo do relatório. Você não consegue escrever `for indicador in indicadores` antes de chamar o modelo — que é o teste do §2.1: no *sectioning* as partes estão no seu código; no orquestrador, quem as define é o modelo.

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

- **Bot × assistente × agente** se separam por **autonomia** (e, em segundo lugar, por complexidade, aprendizado e iniciativa). A lista de características de um agente é de **capacidades**, não de requisitos.
- A taxonomia de mercado (6 tipos) classifica **por destinatário**, não por arquitetura — ler a arquitetura é trabalho seu, com o espectro da Aula 01 mais os três nomes do §2.1.
- Os ganhos se organizam em eixos, e cada um tem uma **unidade de medida** própria: itens por pessoa, % de contenção, tempo de ciclo, custo por transação, taxa de erro. Escolher o eixo é escolher o número que você vai prometer.
- **Parte do benefício é obtida sem LLM nenhum** — vem de ter sido obrigado a escrever a regra que nunca esteve escrita.
- Número de benefício publicado é quase sempre **auto-relatado e sem linha de base**: serve para ordem de grandeza, não como promessa.
- A adoção se concentra em TI, conhecimento e engenharia de software; bancos e seguros à frente, saúde e governo atrás; empresas pequenas praticamente paradas. E a governança não acompanhou.
- **Code agents** é o tipo mais maduro (90% de uso semanal entre devs) porque o ambiente devolve **feedback verificável**.
- **Customer agents** operam em volume alto com autonomia baixa: são **roteador + workflow**, e a fatia resolvida "sem humano" tem definição da própria empresa.
- **Employee agents** funcionam por manter o humano como verificador com autoridade — a arquitetura mais tolerante a falha.
- **Data agents** são o território do **orquestrador-trabalhador**, e o que mais precisa de teto.
- **Creative** e **security** delimitam o espectro pela variável que decide autonomia: **o custo do erro**.
- Seis características comuns aos casos que funcionam — volume, verificador, tolerância a erro, reversibilidade, dado acessível por API e um número de sucesso.
- Nenhum *case study* atribui o resultado ao modelo. Todos atribuem à **integração**.
- **Quase nenhum "agente" de sucesso é um agente**: a autonomia fica confinada ao caminho estreito onde ela paga.

---

## Fontes e leituras

- **Google Cloud — *The ROI of AI: agents are delivering for business now*** ([cloud.google.com/transform/roi-of-ai-how-agents-help-business](https://cloud.google.com/transform/roi-of-ai-how-agents-help-business)) — os números de ganho do §4, do *2025 ROI of AI Report*. Fornecedor sobre o próprio mercado, e **sem metodologia publicada**: leia como ordem de grandeza.
- **Google Cloud — *What are AI agents?*** ([cloud.google.com/discover/what-are-ai-agents](https://cloud.google.com/discover/what-are-ai-agents)) — a distinção entre bot, assistente e agente e as características do §1.
- **Google Cloud — catálogo de casos reais** ([1.302 casos, 11 setores × 6 tipos](https://cloud.google.com/transform/101-real-world-generative-ai-use-cases-from-industry-leaders)) — a taxonomia do §2, a origem dos casos nomeados e o material que você vai navegar para escolher o seu tema.
- **JetBrains — Developer Ecosystem Survey 2026** ([adoção de agentes de codificação](https://blog.jetbrains.com/research/2026/08/ai-coding-agent-adoption-2026/)) — >15.000 desenvolvedores; a melhor amostra desta aula.
- **IBM — estudo de caso do AskHR** ([ibm.com/case-studies/ibm-askhr](https://www.ibm.com/case-studies/ibm-askhr)) — os números do §7.
- **McKinsey — The State of AI** ([2026](https://www.mckinsey.com/capabilities/tech-and-ai/our-insights/tech-forward/state-of-ai-trust-in-2026-shifting-to-the-agentic-era)) — onde a adoção se concentra, por função e por porte.
- Aula 01, [nota 03 §1 e §3](../../aula-01-llms-e-agentes/notas-de-aula/03-agentes-de-ia.md) — o espectro de autonomia com que cada caso é lido, e a fronteira leitura/escrita.
- [nota 02](02-os-casos-que-falharam.md) desta aula — a outra metade da história, e a mais instrutiva.
- **A próxima aula** ([Aula 05 — Arquitetura de agentes](../../aula-05-arquitetura-de-agentes/notas-de-aula/01-padroes-de-arquitetura.md)) abre em código cada padrão que esta nota usou para ler os casos, e transforma o *feedback verificável* do §5 em condição de projeto.
