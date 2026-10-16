# IA Aplicada com LLMs — Aula 13: MLOps e empacotamento — DevOps

---

## Objetivos de aprendizagem

Ao final desta nota, espera-se que o aluno seja capaz de:

- **Explicar** o problema que originou o DevOps — a separação organizacional entre desenvolvimento e operação, os ciclos longos de entrega e as falhas de implantação — e **definir** DevOps como conjunto de cultura, práticas e medição, e não como cargo, equipe ou ferramenta.
- **Formalizar** o ciclo vicioso da implantação rara: o tamanho do lote, a probabilidade de falha e o custo de diagnóstico como funções do intervalo entre implantações.
- **Enunciar** os princípios do DevOps pelo acrônimo CALMS e pelas três maneiras, e **distinguir** o que cada quadro descreve.
- **Relacionar** cada prática — controle de versão, integração contínua, entrega e implantação contínuas, infraestrutura como código, conteinerização, monitoração e observabilidade, pós-incidente sem culpa — ao princípio que ela realiza, com o mecanismo e o limite de cada uma.
- **Definir** SRE, SLI, SLO, orçamento de erro e *toil*, e **calcular** o orçamento de erro de um serviço a partir do SLO.

---

## Desenvolvimento teórico

### 1. Definição

**DevOps** é o conjunto de práticas culturais, técnicas e de medição que reduz o tempo entre a decisão de mudar um sistema e a chegada dessa mudança à produção, mantendo ou aumentando a estabilidade do sistema em produção (KIM et al., 2016). A definição tem três componentes, e nenhum deles basta sozinho:

- **cultura** — responsabilidade compartilhada pelo sistema em produção entre quem o escreve e quem o opera;
- **práticas técnicas** — automação do caminho entre o repositório e a produção, e do retorno de informação da produção para o repositório;
- **medição** — o fluxo de entrega e o comportamento em produção observados com números, e não com impressões.

DevOps não é cargo, equipe nem ferramenta. O título "engenheiro DevOps" e a "equipe de DevOps" são usos de mercado, e uma equipe de DevOps interposta entre desenvolvimento e operação reproduz a separação que o termo pretende eliminar, com um terceiro grupo no meio. Da mesma forma, adotar um servidor de integração contínua ou um orquestrador de *containers* não constitui DevOps: a ferramenta automatiza uma prática, e a prática só tem efeito se a organização em volta dela mudar.

O termo surgiu em 2009. Naquele ano, John Allspaw e Paul Hammond apresentaram na conferência Velocity a forma como a Flickr fazia mais de dez implantações por dia com cooperação entre as equipes de desenvolvimento e de operação (ALLSPAW; HAMMOND, 2009), e Patrick Debois organizou em Gante, na Bélgica, o primeiro DevOpsDays, evento que deu nome ao movimento.

### 2. O muro entre desenvolvimento e operação

A organização clássica de TI separa duas funções. O **desenvolvimento** escreve o software e é cobrado por entregar mudança: funcionalidades novas, correções, prazos. A **operação** mantém o software em produção e é cobrada por estabilidade: disponibilidade, ausência de incidentes. Os incentivos são opostos — a mudança é o que o desenvolvimento produz e o que a operação tem razão para evitar, porque a maior parte dos incidentes em produção começa numa mudança (KIM et al., 2016).

```text
+--------------------+                   ||    +--------------------+
|  Desenvolvimento   |  pacote,          ||    |      Operação      |
|                    |  instruções,      ||    |                    |
|  cobrado por       |  data de          ||    |  cobrada por       |
|  entregar mudança  |  implantação      ||    |  estabilidade      |
|                    | ----------------> || -> |                    |
+--------------------+                   ||    +--------------------+
          ^                             muro              |
          |                                               |
          +-- incidente, sem contexto de quem escreveu ---+
```

O efeito documentado dessa separação é o código "jogado por cima do muro": o desenvolvimento entrega um pacote com instruções de instalação, a operação o implanta manualmente em ambiente que o desenvolvimento não conhece, e o conhecimento necessário para diagnosticar uma falha fica dividido entre dois grupos que se comunicam por chamado. A implantação torna-se manual, rara e arriscada, e cada lado tem incentivo para atribuir a falha ao outro.

### 3. O ciclo vicioso da implantação rara

A raridade da implantação não é só consequência do risco; é também a sua causa. O mecanismo é o seguinte:

1. a implantação é arriscada, e por isso é feita raramente e cercada de aprovação manual;
2. entre duas implantações raras, acumulam-se muitas mudanças, e o lote implantado é grande;
3. um lote grande falha com mais probabilidade e é mais difícil de diagnosticar, porque a falha pode estar em qualquer uma das mudanças e na interação entre elas;
4. a falha confirma o risco, e a resposta organizacional é implantar ainda menos e acrescentar mais aprovação — o que volta ao passo 1 com um lote maior.

```text
+--------------------------+       +--------------------------+
| implantação rara e com   | ----> | lote grande de           |
| aprovação manual         |       | mudanças                 |
+--------------------------+       +--------------------------+
              ^                                  |
              |                                  v
+--------------------------+       +--------------------------+
| percepção de risco       | <---- | falha mais provável e    |
| aumenta                  |       | diagnóstico mais difícil |
+--------------------------+       +--------------------------+
```

Uma formalização simples torna o mecanismo quantitativo. Sejam $\lambda$ a taxa de mudanças produzidas (mudanças por dia), $T$ o intervalo entre implantações (dias) e $p$ a probabilidade de uma mudança isolada conter um defeito que se manifesta em produção, com defeitos independentes entre mudanças. O tamanho do lote é

$$n = \lambda T$$

e a probabilidade de a implantação conter ao menos um defeito é

$$P_{\text{falha}}(T) = 1 - (1 - p)^{\lambda T}$$

que cresce com $T$ e tende a 1. Quando a falha ocorre, o número de mudanças candidatas a causa é $n$; uma busca binária sobre o lote — reverter metade, testar, repetir — exige da ordem de $\lceil \log_2 n \rceil$ rodadas de implantação e verificação, e isso supondo um único defeito e mudanças separáveis, o que lotes grandes raramente garantem. A hipótese de independência é otimista: mudanças acumuladas também interagem entre si, e uma interação não pertence a nenhuma mudança isolada. Com implantação frequente e lote unitário, a falha aponta uma única mudança, e a reversão dela é a correção imediata.

A conclusão é que o custo e o risco de cada implantação crescem com o intervalo entre elas. A resposta do DevOps inverte o ciclo: reduzir o custo da implantação por automação, para que ela possa ser frequente, o que reduz o lote, o que reduz o risco de cada implantação.

### 4. Velocidade e estabilidade não se trocam

A intuição organizacional por trás do muro é a de uma troca: mais mudança implica menos estabilidade, e a estabilidade se compra com menos mudança. A pesquisa empírica de Forsgren, Humble e Kim (2018), conduzida por questionários anuais com milhares de profissionais ao longo de quatro anos, não sustenta essa troca: as organizações de alto desempenho implantam com mais frequência e mais rapidamente **e**, ao mesmo tempo, falham menos e se recuperam mais depressa das falhas. Velocidade e estabilidade aparecem positivamente correlacionadas, e não em oposição.

O resultado é consistente com a §3: a frequência de implantação e a estabilidade são efeitos da mesma causa — lote pequeno, automação e retorno rápido de informação —, e não grandezas que disputam o mesmo recurso. A forma como essa pesquisa mede o desempenho de entrega não é tratada nesta nota.

### 5. Princípios: CALMS

**CALMS** é um acrônimo que enumera as dimensões que uma organização precisa ter para praticar DevOps. A forma original, **CAMS**, é atribuída a John Willis e Damon Edwards (WILLIS, 2010); o **L**, de *lean*, foi acrescentado por Jez Humble.

| Letra | Dimensão | Definição | Prática que a realiza |
|---|---|---|---|
| **C** | Cultura (*Culture*) | responsabilidade compartilhada pelo sistema em produção, sem a separação entre quem muda e quem estabiliza | pós-incidente sem culpa; desenvolvimento participa do plantão |
| **A** | Automação (*Automation*) | todo passo repetível do caminho até a produção executado por máquina | integração contínua; *pipeline* de implantação; infraestrutura como código |
| **L** | *Lean* | trabalho em lotes pequenos, com limite de trabalho em andamento e eliminação de espera e de retrabalho | implantações pequenas e frequentes; integração diária ao ramo principal |
| **M** | Medição (*Measurement*) | decisões sobre o processo e sobre o sistema tomadas sobre números coletados continuamente | monitoração e observabilidade; SLI e SLO |
| **S** | Compartilhamento (*Sharing*) | conhecimento, ferramentas e responsabilidade difundidos entre equipes, e não retidos por uma | relatório de pós-incidente publicado; repositório e *pipeline* comuns |

### 6. Princípios: as três maneiras

**As três maneiras** (*the three ways*) são os princípios dos quais Kim et al. (2016) derivam todas as práticas de DevOps. Cada uma descreve uma direção em que o trabalho ou a informação circula ao longo do **fluxo de valor** (*value stream*) — a sequência de atividades entre a decisão de mudar o sistema e a mudança em produção.

1. **Fluxo** — o trabalho circula da esquerda para a direita, do desenvolvimento para a produção, o mais rápido possível. Realiza-se com lotes pequenos, limite de trabalho em andamento (*work in progress*, WIP), gargalos tornados visíveis e eliminação de espera entre etapas. A formalização da §3 é um argumento de fluxo: lote menor, risco menor.
2. ***Feedback*** — a informação circula da direita para a esquerda, da produção de volta ao desenvolvimento, o mais cedo possível. Realiza-se com telemetria, testes automáticos, detecção precoce de problemas e com a regra de que um defeito detectado **para a linha**: a integração seguinte não prossegue enquanto o ramo principal estiver quebrado.
3. **Aprendizado contínuo e experimentação** — o próprio processo é objeto de melhoria. Realiza-se com análise de incidentes que produz mudança no sistema, tempo reservado para melhorar o trabalho diário e tolerância ao risco calculado de experimentar.

```text
+-----------------+   1ª maneira: fluxo    +-----------------+
|                 | ---------------------> |                 |
| desenvolvimento |                        |    produção     |
|                 | <--------------------- |                 |
+-----------------+  2ª maneira: feedback  +-----------------+
         |                                          |
         +--------------------+---------------------+
                              |
         3ª maneira: aprendizado sobre o próprio processo
```

A terceira maneira depende das duas primeiras: sem fluxo, um experimento leva meses para chegar à produção; sem *feedback*, o resultado do experimento não volta.

### 7. A relação entre os dois quadros

Os dois quadros não competem; descrevem aspectos diferentes da mesma organização.

| | CALMS | As três maneiras |
|---|---|---|
| Descreve | **o que** a organização tem | **em que direção** o trabalho e a informação circulam |
| Natureza | inventário de dimensões | princípios dinâmicos de onde as práticas são derivadas |
| Uso típico | diagnosticar o que falta numa organização | justificar por que uma prática existe |
| Exemplo | "não há medição contínua" (falta o M) | "o defeito leva semanas para ser conhecido" (falha a 2ª maneira) |

As correspondências são diretas: Automação e *Lean* servem sobretudo ao fluxo; Medição serve ao *feedback*; Cultura e Compartilhamento são a condição do aprendizado.

### 8. Práticas

Cada prática abaixo é apresentada pelo conceito, pelo mecanismo e pelo limite, e ligada ao princípio que realiza.

#### 8.1 Controle de versão de tudo

**Conceito.** Todo artefato necessário para reconstruir o sistema em produção é mantido em controle de versão: código da aplicação, configuração, definição da infraestrutura, *scripts* de migração de banco de dados, definição do *pipeline* de implantação. O repositório é a **fonte única de verdade** (*single source of truth*) do que está em produção (HUMBLE; FARLEY, 2010).

**Mecanismo.** Cada estado do sistema corresponde a um identificador de revisão. Reproduzir o estado de produção de qualquer instante passado reduz-se a obter a revisão correspondente e aplicar o processo automatizado; responder "o que mudou entre ontem e hoje" reduz-se a uma diferença entre duas revisões.

**Limite.** O que fica fora do repositório — uma configuração alterada à mão num servidor, uma variável definida no painel de um provedor — não é reproduzível nem auditável. Segredos são a exceção necessária: credenciais não vão para o repositório, que guarda apenas a referência a elas num cofre de segredos.

**Princípio.** Pré-condição de todas as outras práticas; realiza o fluxo (o que não está versionado não pode ser implantado automaticamente) e o compartilhamento.

#### 8.2 Integração contínua

**Conceito.** **Integração contínua** (*continuous integration*, CI) é a prática de integrar as mudanças de cada desenvolvedor ao ramo principal com frequência — ao menos diariamente —, verificando cada integração com *build* e testes automáticos, de modo que o ramo principal permaneça sempre em estado implantável (FOWLER, 2006).

**Mecanismo.** A cada integração, um servidor de CI obtém a revisão, compila, executa a suíte de testes automáticos e publica o resultado. Uma falha torna-se a prioridade da equipe: o ramo quebrado é corrigido ou a mudança é revertida antes de qualquer outra integração. Ramos de vida curta mantêm pequena a diferença entre o trabalho individual e o ramo principal, e com ela o custo de integração — o mesmo argumento de lote da §3, aplicado à integração em vez da implantação.

**Limite.** A integração contínua garante apenas o que os testes verificam: uma suíte fraca produz um ramo "verde" e defeituoso. E ramos de longa duração com um servidor de CI configurado não constituem integração contínua: o servidor existe, mas a integração não é contínua.

**Princípio.** Fluxo (lote de integração pequeno) e *feedback* (o defeito é conhecido minutos depois da mudança).

#### 8.3 Entrega contínua e implantação contínua

**Conceito.** Na **entrega contínua** (*continuous delivery*), toda mudança que passa pelo *pipeline* automatizado resulta num artefato que **pode** ser implantado em produção a qualquer momento; a implantação é uma decisão de negócio, tomada por uma pessoa e executada com um comando. Na **implantação contínua** (*continuous deployment*), toda mudança que passa pelo *pipeline* **vai** para produção, sem intervenção humana (HUMBLE; FARLEY, 2010). A implantação contínua pressupõe a entrega contínua; a recíproca não vale.

**Mecanismo.** O ***pipeline* de implantação** (*deployment pipeline*) é a sequência automatizada de estágios que leva uma revisão do repositório à produção, cada estágio funcionando como um **portão** (*gate*): a revisão só avança se o estágio passar.

```text
[ commit ]
    |
    v
[ build: artefato gerado uma vez ]
    |
    v
[ testes unitários ]
    |
    v
[ testes de aceitação em ambiente de homologação ]
    |
    v
< portão final >
    |
    +-- entrega contínua: decisão humana ----+
    |                                        +--> [ produção ]
    +-- implantação contínua: automático ----+
```

Duas regras sustentam o *pipeline*. O artefato é construído **uma única vez**, no primeiro estágio, e o mesmo artefato binário é promovido de um estágio a outro; reconstruí-lo em cada ambiente abriria a possibilidade de testar uma coisa e implantar outra. E os estágios são ordenados do mais rápido e barato ao mais lento e caro, para que a maior parte dos defeitos seja rejeitada cedo.

**Limite.** O *pipeline* automatiza a decisão de promover, e por isso só é tão confiável quanto os seus portões. A implantação contínua exige ainda a capacidade de reverter rapidamente ou de liberar a mudança de forma gradual, porque nenhum conjunto de testes antecipa todas as condições de produção.

**Princípio.** Fluxo (o caminho até a produção é automatizado e barato) e automação.

#### 8.4 Infraestrutura como código

**Conceito.** **Infraestrutura como código** (*infrastructure as code*, IaC) é a definição do ambiente de execução — servidores, redes, balanceadores, bancos de dados, permissões — em arquivos versionados, a partir dos quais uma ferramenta cria e atualiza o ambiente real (MORRIS, 2016).

**Mecanismo.** Nas ferramentas declarativas, o arquivo descreve o **estado desejado** $E_d$, e a ferramenta consulta o **estado real** $E_r$, calcula a diferença $\Delta = E_d \ominus E_r$ e aplica apenas as operações que levam $E_r$ a $E_d$. A operação é **idempotente**: aplicar duas vezes o mesmo arquivo produz o mesmo ambiente que aplicar uma vez, porque na segunda aplicação $\Delta = \varnothing$. A mudança de infraestrutura passa a ter o mesmo ciclo do código — revisão, histórico, integração e reversão.

A consequência operacional é tratar servidores como **substituíveis**, e não como mantidos à mão: um servidor com defeito é destruído e recriado a partir da definição, em vez de ser consertado. A analogia corrente na área contrapõe o servidor-animal de estimação (*pet*), com nome e cuidados individuais, ao servidor-rebanho (*cattle*), numerado e substituível.

**Limite.** Toda alteração manual feita fora da ferramenta cria **desvio de configuração** (*configuration drift*): o estado real deixa de corresponder ao arquivo, e a próxima aplicação desfaz a alteração ou falha. Dados persistentes não são recriáveis a partir de uma definição: o banco de dados é declarado como código, mas o conteúdo dele exige cópia de segurança e migração.

**Princípio.** Automação e controle de versão aplicados ao ambiente; realiza o fluxo (ambientes criados sob demanda) e o compartilhamento (o ambiente deixa de ser conhecimento de uma pessoa).

#### 8.5 Conteinerização

**Conceito.** Uma **imagem de *container*** é um artefato imutável que empacota a aplicação com as suas dependências de sistema e de linguagem, identificado por um *digest* criptográfico do seu conteúdo. Um ***container*** é um processo executado a partir de uma imagem, isolado do sistema hospedeiro.

**Mecanismo.** A imagem é construída uma vez, no estágio de *build* do *pipeline*, e o mesmo *digest* é promovido de homologação à produção — a regra do artefato único da §8.3 levada ao ambiente de execução inteiro. A diferença entre ambientes reduz-se à configuração injetada no início do processo (variáveis de ambiente, segredos), e não ao conteúdo do artefato. A mecânica de imagem, *container* e Compose está na [nota de Docker](../../aula-00-sobre-o-curso/notas-de-aula/02-docker.md) da aula sobre o curso; a publicação de uma imagem como serviço é tratada na aula de empacotamento.

**Limite.** O *container* compartilha o núcleo do sistema operacional do hospedeiro e não oferece o isolamento de uma máquina virtual. A imutabilidade cobre o artefato, não o estado: o que o processo grava no sistema de arquivos do *container* se perde quando ele é substituído, e o estado persistente precisa viver fora dele.

**Princípio.** Fluxo (o mesmo artefato percorre todos os ambientes) e automação.

#### 8.6 Monitoração e observabilidade

**Monitoração** é a coleta contínua de dados quantitativos sobre o sistema em produção; **observabilidade** é a propriedade do sistema de permitir investigar o seu estado interno a partir da telemetria que ele emite. As duas são a segunda maneira posta em prática: sem elas, a informação sobre a produção não volta ao desenvolvimento. O conteúdo — sinais de telemetria, SLI e SLO na prática, percentis e instrumentação — é tratado na aula de observabilidade.

**Princípio.** *Feedback* e medição.

#### 8.7 Pós-incidente sem culpa

**Conceito.** O **pós-incidente** (*postmortem*) é a análise escrita de um incidente depois de resolvido: o que aconteceu, qual foi o impacto, que condições o permitiram, como foi detectado e mitigado, e que ações impedem a recorrência. O pós-incidente **sem culpa** (*blameless postmortem*) procura as condições sistêmicas que permitiram a falha, e não o indivíduo responsável por ela (ALLSPAW, 2012; BEYER et al., 2016).

**Mecanismo.** A justificativa é informacional. Se a análise de incidentes resulta em punição, quem esteve envolvido deixa de relatar o que fez e por quê, e a organização perde exatamente a informação necessária para corrigir a condição que permitiu a falha. Partindo da premissa de que a pessoa agiu de forma razoável com a informação que tinha naquele momento, a pergunta deixa de ser "quem errou" e passa a ser "o que no sistema tornou esse erro possível, ou provável" — um comando destrutivo sem confirmação, um painel que não mostrava o dado relevante, um procedimento desatualizado. A ação corretiva muda o sistema, e não a pessoa (ALLSPAW, 2012). O relatório é publicado para toda a organização, e é isso que transforma um incidente local em aprendizado compartilhado.

**Limite.** "Sem culpa" não significa sem responsabilidade: as ações corretivas têm responsável e prazo. E um pós-incidente sem ação corretiva acompanhada até o fim é um documento, não aprendizado. Em sistemas complexos, a busca por uma única "causa raiz" também é uma simplificação: incidentes resultam em geral da combinação de vários fatores contribuintes.

**Princípio.** Aprendizado contínuo (terceira maneira), cultura e compartilhamento.

#### 8.8 Síntese das práticas e princípios

| Prática | Princípio (três maneiras) | Dimensão (CALMS) |
|---|---|---|
| Controle de versão de tudo | fluxo | Automação, Compartilhamento |
| Integração contínua | fluxo, *feedback* | Automação, *Lean* |
| Entrega e implantação contínuas | fluxo | Automação, *Lean* |
| Infraestrutura como código | fluxo | Automação, Compartilhamento |
| Conteinerização | fluxo | Automação |
| Monitoração e observabilidade | *feedback* | Medição |
| Pós-incidente sem culpa | aprendizado contínuo | Cultura, Compartilhamento |

### 9. SRE como implementação

**Site Reliability Engineering** (SRE) é a forma como o Google opera os seus sistemas de produção: a operação tratada como problema de engenharia de software, executada por engenheiros que escrevem código para automatizar a operação e que têm a confiabilidade do serviço como responsabilidade (BEYER et al., 2016). Em relação ao DevOps, o SRE é uma implementação concreta: os princípios das §§5 e 6 aparecem nele como papéis, metas numéricas e regras de decisão.

Três conceitos formam o núcleo do SRE:

- **SLI** (*service level indicator*) — uma medida quantitativa de um aspecto do serviço, em geral expressa como fração de eventos bons sobre eventos válidos: por exemplo, a fração de requisições respondidas com sucesso em menos de 300 ms.
- **SLO** (*service level objective*) — o valor-alvo de um SLI num intervalo de tempo: por exemplo, 99,9% das requisições bem-sucedidas, medidos numa janela de 30 dias.
- **Orçamento de erro** (*error budget*) — a quantidade de falha que o SLO admite:

$$B = 1 - \text{SLO}$$

Sobre $N$ eventos válidos na janela, o número de eventos ruins admitidos é $B \cdot N$; sobre uma janela de duração $\Delta t$, para um SLO de disponibilidade, a indisponibilidade admitida é $B \cdot \Delta t$.

O orçamento de erro é a formalização do conflito da §2. O SLO não é 100%, porque nenhum usuário distingue 99,999% de 100% através das demais falhas do caminho até ele, e porque cada fração de disponibilidade acima do necessário custa velocidade de mudança. A diferença entre 100% e o SLO é, portanto, um recurso a ser gasto, e a regra de decisão é explícita: **enquanto há orçamento, implanta-se**; **esgotado o orçamento, a prioridade passa a ser a estabilidade** — as implantações de funcionalidade são suspensas até que a janela recupere o orçamento (BEYER et al., 2016). O desenvolvimento e a operação deixam de negociar caso a caso, porque passam a compartilhar a mesma meta numérica.

O SRE limita também o trabalho de operação. ***Toil*** é o trabalho operacional manual, repetitivo, automatizável, tático, sem valor duradouro e que cresce linearmente com o tamanho do serviço — reiniciar um processo à mão a cada alerta, aplicar a mesma alteração em cada servidor (BEYER et al., 2016). O Google estabelece como meta que o *toil* ocupe menos de 50% do tempo de cada engenheiro de SRE; o restante é dedicado a engenharia que reduz o *toil* futuro. A regra é a terceira maneira em forma de orçamento de tempo: sem ela, a operação cresce em proporção ao serviço e não sobra tempo para melhorar o próprio processo.

**Limite.** O SRE pressupõe SLIs que reflitam a experiência do usuário: um SLO cumprido sobre um indicador mal escolhido autoriza implantações enquanto os usuários percebem degradação. E a regra do orçamento só tem efeito se a organização a cumprir quando o orçamento se esgota.

---

## Fontes e leituras

ALLSPAW, J. **Blameless PostMortems and a Just Culture**. Etsy Code as Craft, 2012. Disponível em: https://www.etsy.com/codeascraft/blameless-postmortems/.

ALLSPAW, J.; HAMMOND, P. **10+ deploys per day**: Dev and Ops cooperation at Flickr. Apresentação na O'Reilly Velocity Conference, San Jose, 2009.

BEYER, B.; JONES, C.; PETOFF, J.; MURPHY, N. R. (org.). **Site reliability engineering**: how Google runs production systems. Sebastopol: O'Reilly Media, 2016. Cap. 3: Embracing risk; cap. 4: Service level objectives; cap. 5: Eliminating toil; cap. 15: Postmortem culture.

FORSGREN, N.; HUMBLE, J.; KIM, G. **Accelerate**: the science of lean software and DevOps. Portland: IT Revolution, 2018.

FOWLER, M. **Continuous integration**. 2006. Disponível em: https://martinfowler.com/articles/continuousIntegration.html.

HUMBLE, J.; FARLEY, D. **Continuous delivery**: reliable software releases through build, test, and deployment automation. Boston: Addison-Wesley, 2010.

KIM, G.; HUMBLE, J.; DEBOIS, P.; WILLIS, J. **The DevOps handbook**: how to create world-class agility, reliability, and security in technology organizations. Portland: IT Revolution, 2016.

MORRIS, K. **Infrastructure as code**: managing servers in the cloud. Sebastopol: O'Reilly Media, 2016.

WILLIS, J. **What DevOps means to me**. Chef Blog, 2010.

**Material da disciplina.** [Docker](../../aula-00-sobre-o-curso/notas-de-aula/02-docker.md) — imagem, *container* e Compose, da aula sobre o curso.
