# IA Aplicada com LLMs — Aula 13: MLOps e empacotamento — MLOps

---

## Objetivos de aprendizagem

Ao final desta nota, espera-se que o aluno seja capaz de:

- **Justificar** por que sistemas de aprendizado de máquina exigem mais que DevOps, a partir dos três eixos de mudança — código, dados e modelo — e da degradação sem mudança de código.
- **Explicar** a dívida técnica oculta de sistemas de ML pelos conceitos de Sculley et al. (2015): o código de ML como fração pequena do sistema, o princípio CACE, os ciclos de *feedback* ocultos, o código de cola e as *pipeline jungles*.
- **Definir** MLOps como extensão dos princípios de DevOps aos artefatos próprios de ML.
- **Descrever** os dez tópicos de MLOps — do ciclo de vida do modelo à operação e à organização em volta dele — com o mecanismo de cada um e as ferramentas como exemplos de categoria.
- **Localizar** onde o curso já tratou ou tratará cada tópico, reconhecendo as práticas já executadas sob outros nomes.
- **Classificar** um sistema nos níveis de maturidade 0, 1 e 2 propostos pelo Google.

---

## Desenvolvimento teórico

### 1. Três eixos de mudança

Um **eixo de mudança** é uma dimensão do sistema cuja alteração, isolada, altera o comportamento observado em produção. Em software convencional há um eixo só: o comportamento muda quando o código muda, e o controle de versão do código — com a configuração e a infraestrutura declaradas em arquivo, como trata a [nota sobre DevOps](01-devops.md) — identifica tudo o que está em produção.

Em um sistema de aprendizado de máquina há três eixos:

- **código**: o programa de preparação de dados, de treino, de avaliação e de inferência;
- **dados**: o conjunto usado para treinar e avaliar, e a distribuição das entradas recebidas em produção;
- **modelo**: os parâmetros aprendidos, que são o resultado de executar o código de treino sobre os dados.

O terceiro eixo não é independente dos outros dois. Escrito como função, o modelo é

$$M = \text{treino}(C_{\text{treino}},\ D,\ h,\ s)$$

em que $C_{\text{treino}}$ é o código de treino, $D$ é o conjunto de dados, $h$ são os hiperparâmetros e $s$ é a semente aleatória. O artefato implantável é o par formado pelo código de inferência e por $M$; a sua identidade é, portanto, a tupla de versões $(v_C, v_D, v_M)$, com $v_M$ determinado por $(v_C, v_D, h, s)$ e pelo ambiente de execução. **Reprodutibilidade**, em ML, é a capacidade de reconstruir $M$ a partir dessa tupla, e exige fixar os três eixos — o código com o *commit*, os dados com uma versão própria, o ambiente com uma imagem — e não apenas o primeiro.

```text
+--------------+
| Código  v_C  |--------+----------------------------------------+
+--------------+        |                                        |
                        v                                        v
+--------------+    +--------+    +-------------+    +-----------------------+
| Dados   v_D  |--->| Treino |--->| Modelo  v_M |--->| Artefato implantável  |
+--------------+    +--------+    +-------------+    | (v_C, v_D, v_M)       |
                                                     +-----------------------+
                                                                 |
                                                                 v
              +--------------------------------+     +-----------------------+
              | Distribuição das entradas      |---->| Comportamento         |
              | em produção                    |     | em produção           |
              +--------------------------------+     +-----------------------+
```

O controle de versão de código, sozinho, identifica um dos três eixos. Dois modelos treinados a partir do mesmo *commit* com *datasets* diferentes têm o mesmo identificador no repositório e comportamentos distintos.

### 2. Degradação sem mudança de código

O quarto nó do diagrama — a distribuição das entradas em produção — não é versionado por ninguém, porque não pertence ao sistema: é o mundo. Um modelo treinado aprende uma aproximação da distribuição conjunta $P(X, Y)$ das entradas $X$ e dos rótulos $Y$ observados no treino. Escrita como

$$P(X, Y) = P(Y \mid X)\, P(X)$$

ela pode mudar de duas maneiras depois do treino:

- **desvio de dados** (***data drift***, também *covariate shift*): $P(X)$ muda e $P(Y \mid X)$ se mantém. As entradas passam a vir de regiões do espaço pouco representadas no treino — um modelo de crédito treinado antes de uma mudança de público passa a receber outra faixa de renda;
- **desvio de conceito** (***concept drift***): $P(Y \mid X)$ muda. A mesma entrada passa a ter outro rótulo correto — o padrão de transação que era legítimo passa a ser o padrão de uma fraude nova.

Nos dois casos o sistema piora **sem nenhum *commit***. A implantação não mudou, os testes de código continuam passando, as métricas de serviço — latência, taxa de erro, saturação — continuam normais, porque o modelo responde no mesmo tempo e sem exceção; responde errado. Nenhuma métrica de DevOps detecta essa classe de falha, e é por isso que o monitoramento de ML tem uma camada própria (§11).

### 3. Definição de MLOps

**MLOps** (*machine learning operations*) é o conjunto de práticas, papéis e ferramentas que torna o ciclo de vida de modelos de aprendizado de máquina — dados, treino, avaliação, implantação e monitoramento — automatizado, reprodutível e observável, estendendo os princípios de DevOps aos artefatos próprios de ML (KREUZBERGER; KÜHL; HIRSCHL, 2023). A definição de Kreuzberger, Kühl e Hirschl (2023), obtida por revisão de literatura e entrevistas com praticantes, trata MLOps como paradigma — práticas, conceitos e cultura de desenvolvimento — e não como ferramenta, pela mesma razão que a nota sobre DevOps aplica a DevOps.

A relação com DevOps é de extensão, e não de substituição. Os princípios — fluxo em lotes pequenos, *feedback* por telemetria, aprendizado com falhas, automação do caminho até a produção — permanecem; o que muda é o conjunto de artefatos sobre os quais eles operam e o conjunto de papéis que precisam colaborar (IBM, s.d.; AMAZON WEB SERVICES, s.d.):

| | DevOps | MLOps |
|---|---|---|
| Eixos de mudança | código | código, dados e modelo |
| Artefato versionado | código, configuração, infraestrutura | os mesmos, mais *datasets*, *features*, modelos e metadados de execução |
| Integração contínua | testa código | testa código, valida dados e valida modelo |
| Entrega contínua | implanta a aplicação | implanta o *pipeline* de treino, que produz e implanta o modelo |
| Gatilho de mudança | *commit* | *commit*, chegada de dados novos, alerta de *drift* |
| Monitoramento | saúde do serviço | saúde do serviço e qualidade das predições |
| Papéis | desenvolvimento e operação | desenvolvimento, operação, ciência de dados, engenharia de dados e de ML |

A Amazon Web Services (s.d.) organiza a extensão sob o rótulo de **contínuo X**: integração contínua, entrega contínua, **treino contínuo** e **monitoramento contínuo**, os dois últimos sem equivalente em DevOps.

Kreuzberger, Kühl e Hirschl (2023) derivam nove princípios da literatura: automação de CI/CD, orquestração de fluxos de trabalho, reprodutibilidade, versionamento de dados, modelo e código, colaboração, treino e avaliação contínuos, rastreamento de metadados de ML, monitoramento contínuo e ciclos de *feedback*. Os dez tópicos das §§6 a 15 são a organização dessas práticas adotada nesta aula.

### 4. A dívida técnica oculta

**Dívida técnica** é o custo futuro de manutenção acumulado por decisões que aceleram o presente. Sculley et al. (2015) sustentam que sistemas de ML acumulam dívida técnica com mais facilidade que software convencional, porque, além de toda a dívida de código, têm uma dívida própria que fica no **nível do sistema**, fora do código do modelo, e por isso não aparece em revisão de código.

**O código de ML é uma fração pequena do sistema.** A observação central do artigo é que, em um sistema de ML real, o código que define e treina o modelo ocupa uma parte pequena do total; o restante é coleta de dados, verificação de dados, extração de *features*, configuração, gestão de recursos, infraestrutura de *serving*, monitoramento e ferramentas de análise e de gestão de processos (SCULLEY et al., 2015).

```text
+--------------------------------------------------------------------+
|                     Sistema de ML em produção                      |
|                                                                    |
|   Configuração      Coleta de dados      Verificação de dados      |
|   Extração de features      Gestão de recursos                     |
|                                                                    |
|                         +--------------+                           |
|                         | Código de ML |                           |
|                         +--------------+                           |
|                                                                    |
|   Ferramentas de análise      Infraestrutura de serving            |
|   Monitoramento      Gestão de processos                           |
+--------------------------------------------------------------------+
```

Três conceitos do artigo organizam o resto da dívida.

**CACE** (*Changing Anything Changes Everything*). Um modelo entrelaça as suas entradas: o peso aprendido para uma *feature* depende das demais. Se a distribuição de uma *feature* $x_1$ muda, ou se uma *feature* $x_{k+1}$ é acrescentada ou retirada, os pesos de todas as outras mudam no retreino. Não há, portanto, mudança isolada: o princípio de encapsulamento, que em software permite alterar um módulo sem revalidar os demais, não vale para as entradas de um modelo. O mesmo entrelaçamento alcança hiperparâmetros, parâmetros de amostragem e limiares de decisão (SCULLEY et al., 2015).

**Ciclos de *feedback* ocultos.** Quando a saída do modelo influencia os dados que alimentarão o próximo treino, o sistema passa a aprender parcialmente consigo mesmo. No caso direto, o modelo escolhe o que será observado — um sistema de recomendação só coleta cliques sobre o que ele mesmo recomendou. No caso **oculto**, a influência passa por outro sistema: dois modelos de empresas diferentes que agem no mesmo mercado alteram, um, os dados do outro. O efeito não aparece em nenhum teste isolado e se manifesta como deriva lenta do comportamento (SCULLEY et al., 2015).

**Código de cola e *pipeline jungles*.** Pacotes genéricos de ML exigem código de adaptação entre o formato dos dados do sistema e o formato que o pacote espera — o **código de cola** (*glue code*), que, segundo os autores, pode chegar a ser a maior parte do código de um sistema maduro e congela o sistema em torno do pacote. A *pipeline jungle* é o caso análogo na preparação de dados: uma sequência de raspagens, junções e passos intermediários acrescentada por incrementos, sem projeto, que só se testa ponta a ponta e que ninguém consegue alterar com segurança (SCULLEY et al., 2015).

O artigo lista outras formas de dívida que reaparecem nos tópicos seguintes: **dependências de dados** mais caras que dependências de código, porque não são declaradas nem verificadas por compilador; **consumidores não declarados** da saída do modelo; e **dívida de configuração**, quando a configuração de um sistema de ML é maior e menos testada que o próprio código.

### 5. Consequência: o *pipeline* precisa de portões novos

Da §1 à §4 decorre o que muda no *pipeline* de implantação. Em DevOps, o *pipeline* é uma sequência de portões sobre código: *build*, testes unitários, testes de integração, implantação. Em ML, a mudança pode entrar por dados ou por modelo, e o *pipeline* precisa também:

- **validar dados** antes do treino — *schema*, faixas, ausentes, distribuição contra a de referência;
- **avaliar o modelo contra uma linha de base** — o modelo candidato só avança se não for pior que o que está em produção, no mesmo conjunto de avaliação;
- **decidir a promoção** — registrar o candidato, com a linhagem de dados e código que o produziu, e movê-lo entre estágios.

O ciclo de vida que resulta é o seguinte; os números indicam os tópicos das seções seguintes.

```text
            +---------------------------------------+
     +----->| 1. Dados: ingestão, validação, versão |<.....
     |      +---------------------------------------+     :
     |                         |                          :
     |                         v                          :
     |      +---------------------------------------+     :
     |      | 2. Experimentação e treino            |<....:
     |      +---------------------------------------+     :   +----------------+
     |                         |                          :   | 4. Pipeline    |
     |                         v                          :...| automatizado   |
     |      +---------------------------------------+     :   | CI/CD/CT       |
     |      | 3. Registro: versão e linhagem        |<....:   +----------------+
     |      +---------------------------------------+     :
     |                         |                          :
     |                         v                          :
     |      +---------------------------------------+     :
     |      | 5. Deploy e serving                   |<....:
     |      +---------------------------------------+
     |                         |
     |                         v
     |      +---------------------------------------+
     +------| 6. Monitoramento: serviço e modelo    |
            +---------------------------------------+
     laço à esquerda: o alerta de drift é o gatilho de retreino
```

Os tópicos 7 a 10 — testes, infraestrutura, governança e cultura — atravessam o ciclo inteiro e não ocupam um nó dele.

### 6. Tópico 1 — Gestão de dados

**Gestão de dados**, em MLOps, é o conjunto de práticas que trata o *dataset* como artefato de engenharia: com origem conhecida, com qualidade verificada e com identidade versionada, como o código. Compreende quatro atividades:

- **ingestão**: a coleta das fontes, com registro da origem e do instante de cada lote;
- **validação**: a verificação automática de *schema* (tipos, colunas obrigatórias), de faixas (idade entre 0 e 120), de ausentes e de distribuição em relação a uma referência;
- **limpeza**: a correção ou a exclusão de registros inválidos, como transformação versionada e não como edição manual;
- **versionamento de dados**: a atribuição de um identificador imutável a cada estado do *dataset*, em geral um *hash* do conteúdo, para que o modelo possa referir a versão exata com que foi treinado. Arquivos grandes não cabem no controle de versão de código; ferramentas como DVC e lakeFS versionam os dados fora do repositório e guardam nele apenas o ponteiro.

Uma ***feature store*** (exemplo: Feast) é um repositório que define cada *feature* uma única vez — a transformação que a calcula a partir dos dados brutos — e a serve tanto ao treino, em lote e com valores históricos, quanto à inferência, com baixa latência. O problema que ela ataca é a **divergência treino-produção** (***training-serving skew***): quando a mesma *feature* é calculada por dois códigos diferentes — um *script* de treino e um serviço de inferência —, qualquer diferença entre eles faz o modelo receber em produção uma entrada diferente da que viu no treino, sem erro nenhum.

**No sistema do curso.** Não há treino, e os dados que mudam o comportamento são outros: o *golden dataset* revisado e versionado construído na aula de *evals*, que é o conjunto de avaliação com identidade, e o índice da aula de *embeddings* e RAG, que muda as respostas do agente sem mudança de código quando os documentos mudam.

### 7. Tópico 2 — Experimentação e desenvolvimento de modelos

**Rastreamento de experimentos** (*experiment tracking*) é o registro sistemático, a cada execução de treino ou de avaliação, de tudo o que a determina e de tudo o que ela produz: parâmetros e hiperparâmetros, versões de código e de dados, ambiente, métricas e artefatos gerados. Ferramentas como MLflow e Weights & Biases guardam esses registros e permitem comparar execuções.

O mecanismo responde à pergunta que a experimentação sem registro não responde: qual combinação produziu o melhor resultado, e se ela pode ser reproduzida. A reprodutibilidade exige fixar a semente, as versões dos dados e do ambiente, e registrar os três junto com o resultado (§1).

O desenvolvimento de modelos começa, em geral, em *notebook*: um meio adequado à exploração e inadequado à produção, porque a ordem de execução das células é livre, o estado fica oculto na sessão e o código não é testável por partes. A prática de MLOps é a **passagem do *notebook* exploratório para código de treino organizado** — funções, módulos, configuração em arquivo e testes —, que pode ser executado sem intervenção por um *pipeline*.

**No sistema do curso.** O equivalente de uma execução rastreada é a rodada da suíte de *evals* sobre uma combinação de prompt, modelo e parâmetros, com o resultado registrado ao lado do carimbo de versão; a comparação entre duas versões de prompt, repetida para separar diferença de ruído, é um experimento.

### 8. Tópico 3 — Versionamento e registro de modelos

Um **registro de modelos** (***model registry***) é um catálogo de versões de modelos em que cada versão carrega metadados, linhagem e estágio:

- **metadados**: métricas de avaliação, hiperparâmetros, autor, data, formato do artefato;
- **linhagem** (*lineage*): a versão de dados e a versão de código que produziram o modelo, e a execução de treino que o gerou;
- **estágio**: a posição do modelo no ciclo — tipicamente *staging*, produção e arquivado —, com **promoção controlada** entre eles, por aprovação ou por critério automático.

A linhagem é o que permite responder, diante de um incidente, de onde veio o modelo em produção: com ela, a cadeia "predição errada → versão do modelo → execução de treino → versão do *dataset* → lote de ingestão" é uma sequência de consultas, e não uma reconstrução por memória. Sem ela, um modelo em produção é um binário de origem desconhecida.

**No sistema do curso.** Duas práticas já executadas são versões deste tópico. O versionamento do prompt e o *prompt registry*, tratados na aula de *prompt engineering* ([nota sobre o prompt como código](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md)), dão identidade à combinação de prompt, modelo e parâmetros. O **carimbo de versão** registrado em cada execução é, sem esse nome, o registro de linhagem do sistema: liga cada saída à versão que a produziu.

### 9. Tópico 4 — Automação de *pipelines* e CI/CD/CT

Um ***pipeline* de ML** é a sequência de etapas que vai dos dados ao modelo avaliado — extração, validação, preparação, treino, avaliação, registro — expressa como um **grafo acíclico dirigido de tarefas**, em que cada aresta é uma dependência de dados entre etapas. Orquestradores como Airflow, Kubeflow Pipelines e Prefect executam esse grafo, retentam etapas que falham e registram cada execução.

A automação estende as práticas de DevOps em três pontos (GOOGLE CLOUD, 2020):

- **CI estendida**: além de testar código, testa e valida dados, *schemas* e modelos;
- **CD estendida**: implanta o *pipeline* de treino, que por sua vez produz e implanta o modelo — o artefato entregue deixa de ser um modelo e passa a ser o sistema que gera modelos;
- **treino contínuo** (*continuous training*, CT): o retreino executado automaticamente pelo *pipeline* implantado, disparado por **agenda** (todo dia), por **chegada de dados** (um lote novo disponível) ou por **alerta de *drift*** (§11), sempre seguido de avaliação automática contra a linha de base antes da promoção.

A consequência do CT é que o modelo em produção pode mudar sem nenhum *commit* e sem nenhuma pessoa no caminho. Isso só é seguro se o portão de avaliação do *pipeline* for confiável: o CT desloca o controle de qualidade de uma revisão humana para um teste automático.

**No sistema do curso.** O sistema não treina, e não há CT. O portão de integração contínua existe e é a suíte de regressão da aula de *evals*, rodada antes de cada mudança de prompt, de modelo ou de parâmetro — a forma que a integração contínua toma num sistema com LLM.

### 10. Tópico 5 — *Deploy* e *serving*

***Serving*** é a disponibilização do modelo para inferência. Há quatro modos, escolhidos pela latência exigida, pelo volume e por onde está o dado:

| Modo | Mecanismo | Quando se aplica |
|---|---|---|
| **API em tempo real** | uma requisição, uma predição, com latência de milissegundos a segundos | decisão no momento da interação: crédito no *checkout*, resposta ao usuário |
| ***Batch*** | predições em lote sobre um conjunto, em horário agendado | quando a predição pode ser pré-calculada: pontuação noturna de toda a base |
| ***Streaming*** | predição sobre eventos à medida que chegam a uma fila | fluxo contínuo de eventos com latência de segundos: detecção de anomalia em transações |
| ***Edge*** | o modelo executa no dispositivo | dado que não pode sair do dispositivo, ou rede ausente ou lenta |

A implantação usa as mesmas práticas descritas na nota sobre DevOps — conteinerização com Docker, orquestração com Kubernetes —, acrescidas de **servidores de modelo** (exemplos: BentoML, Seldon, KServe), que padronizam o carregamento do artefato, a interface de inferência, o agrupamento de requisições em lote e as métricas.

Como o comportamento de um modelo novo só se conhece com tráfego real, a liberação usa **estratégias de liberação** que limitam a exposição:

- ***canary***: a versão nova recebe uma fração crescente do tráfego — 1%, 10%, 50%, 100% —, e a progressão é interrompida se as métricas pioram;
- ***shadow***: a versão nova recebe uma cópia do tráfego real e suas saídas são registradas sem serem entregues ao usuário; permite comparar as duas versões sobre as mesmas entradas, sem risco, ao custo de processar o tráfego duas vezes;
- **teste A/B**: duas versões são servidas a populações distintas, atribuídas aleatoriamente, e comparadas por uma métrica de negócio, com teste estatístico sobre a diferença.

***Canary*** e *shadow* respondem se a versão nova é segura; o teste A/B responde se ela é melhor para o negócio.

**No sistema do curso.** A [nota sobre de *script* a imagem](03-de-script-a-imagem.md), nesta aula, aplica este tópico ao agente: a imagem Docker como artefato imutável, introduzida na [nota sobre Docker](../../aula-00-sobre-o-curso/notas-de-aula/02-docker.md), e a execução do agente em container.

### 11. Tópico 6 — Monitoramento e observabilidade

O monitoramento de um sistema de ML tem **duas camadas**:

- **camada de serviço**: latência, taxa de erro, tráfego, saturação de recursos — as mesmas métricas de DevOps;
- **camada de modelo**: desempenho preditivo (acurácia, erro, calibração), distribuição das entradas (*data drift*) e relação entre entrada e rótulo (*concept drift*).

A segunda camada enfrenta o **problema do rótulo atrasado**: o desempenho real de uma predição só é conhecido quando o rótulo verdadeiro chega — o empréstimo é pago ou não meses depois; a transação é contestada semanas depois. Durante esse intervalo, a acurácia não pode ser medida. O sinal disponível é a distribuição da entrada: $P(X)$ é observável na hora, e um desvio em $P(X)$ antecipa a provável queda de desempenho. Medidas usuais são testes de duas amostras, como o de Kolmogorov-Smirnov, e o **índice de estabilidade populacional** (PSI), que compara a fração $e_i$ de observações em cada faixa $i$ da referência com a fração $a_i$ em produção:

$$\text{PSI} = \sum_{i=1}^{k} (a_i - e_i)\,\ln\frac{a_i}{e_i}$$

O PSI é zero quando as distribuições coincidem e cresce com a divergência. O limite do sinal antecipado é que ele não detecta *concept drift* puro: se $P(X)$ não muda e $P(Y \mid X)$ muda, só o rótulo revela a piora.

Os alertas da camada de modelo são o **gatilho de retreino**: ao disparar, iniciam o *pipeline* do tópico 4, e o ciclo da §5 se fecha.

**No sistema do curso.** A aula de observabilidade aplica este tópico ao agente: telemetria em convenção aberta, painel, alerta, custo, e a **avaliação online** — o juiz da aula de *evals* executado sobre uma amostra do tráfego real —, que é o monitoramento da camada de modelo num sistema com LLM. A separação entre avaliação *offline*, antes do *deploy*, e *online*, em produção, foi estabelecida na aula de *evals* ([nota sobre o juiz e a suíte](../../aula-11-evals/notas-de-aula/03-o-juiz-e-a-suite.md)). O *drift* tem duas formas no sistema do curso: a das perguntas recebidas, que mudam de distribuição, e a do próprio modelo, quando o provedor atualiza o modelo apontado por um *alias* sem versão ou o deprecia, como tratado na aula de escolha e configuração de modelos ([nota sobre escolha de modelos](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md), §7).

### 12. Tópico 7 — Testes e qualidade

Os testes de um sistema de ML se organizam em quatro categorias:

- **testes de dados**: *schema*, distribuição de cada *feature* contra a referência, proporção de ausentes, ausência de vazamento do rótulo nas *features*;
- **testes de código**: unitários sobre as transformações de dados e sobre as etapas do *pipeline*;
- **testes de modelo**: desempenho contra a linha de base no mesmo conjunto; **robustez** a perturbações pequenas da entrada; e **viés por subgrupo**, isto é, desempenho medido separadamente em cada fatia relevante da população, já que uma métrica agregada boa pode esconder uma fatia com desempenho ruim;
- **testes de integração**: o *pipeline* executado de ponta a ponta, dos dados brutos à predição servida.

O ***ML Test Score*** (BRECK et al., 2017) é uma rubrica de prontidão para produção organizada em quatro seções — testes de *features* e dados, de desenvolvimento do modelo, de infraestrutura de ML e de monitoramento —, com sete testes cada. Cada teste vale meio ponto se é executado manualmente, com resultado documentado, e um ponto se é automatizado e executado repetidamente. A pontuação final é o **mínimo** entre as quatro seções:

$$\text{Score} = \min_{j \in \{\text{dados},\ \text{modelo},\ \text{infra},\ \text{monitoramento}\}} \sum_{t \in j} p_t, \qquad p_t \in \{0,\ 0{,}5,\ 1\}$$

O mínimo, e não a soma, expressa a tese da rubrica: um sistema com testes de modelo exaustivos e nenhum monitoramento não está pronto, e excelência numa seção não compensa ausência em outra.

**No sistema do curso.** Os testes de modelo são o objeto inteiro da aula de *evals*: os quatro níveis de avaliação — determinístico, trajetória, resultado e juiz — e o juiz validado contra rótulo humano ([nota sobre os quatro níveis](../../aula-11-evals/notas-de-aula/02-os-quatro-niveis.md)).

### 13. Tópico 8 — Infraestrutura e escalabilidade

A infraestrutura de ML difere da de software convencional pelo perfil de carga: o treino é intensivo, longo e intermitente; a inferência é contínua e sensível a latência. Os pontos de decisão são:

- **nuvem ou *on-premise***: elasticidade e custo por uso contra controle, residência do dado e custo fixo;
- **GPU**: necessária para o treino de redes profundas e, conforme o tamanho do modelo e a latência exigida, para a inferência;
- **infraestrutura como código** (exemplo: Terraform): a mesma prática da nota sobre DevOps, aplicada a *clusters* de treino e a serviços de inferência;
- **otimização de custo**: instâncias interruptíveis (*spot*) para o treino, que tolera interrupção quando há *checkpoint*; **autoescala** para a inferência, que acompanha o tráfego; e **desligamento de recurso ocioso**, já que uma GPU parada custa o mesmo que uma GPU em uso.

**No sistema do curso.** O modelo é hospedado por terceiro ou executado localmente, decisão tratada na aula de escolha e configuração de modelos; a infraestrutura própria se reduz ao processo que chama o modelo. A nota sobre de *script* a imagem, nesta aula, trata a parte deste tópico que se aplica ao agente: a imagem, o segredo e o estado no container efêmero.

### 14. Tópico 9 — Governança, segurança e conformidade

**Governança de modelos** é o conjunto de controles que torna verificável quem fez o quê com cada modelo e com cada dado, e sob que regra. Compreende:

- **auditoria**: o registro de quem treinou, aprovou e implantou cada versão — a linhagem do tópico 3 usada como prova;
- **explicabilidade**: a capacidade de indicar quais fatores levaram a uma predição, exigida quando a decisão afeta uma pessoa;
- ***fairness***: a verificação de que o desempenho e as taxas de decisão não diferem de forma injustificada entre grupos;
- **privacidade de dados**: no Brasil, o regime aplicável é a LGPD (BRASIL, 2018), que exige base legal para o tratamento de dados pessoais e garante, no art. 20, o direito do titular de solicitar a revisão de decisões tomadas unicamente com base em tratamento automatizado que afetem os seus interesses;
- **controle de acesso** a dados e modelos: quem pode ler o *dataset*, quem pode promover um modelo;
- **documentação**: o ***model card*** (MITCHELL et al., 2019) é um documento curto que acompanha o modelo e declara detalhes do modelo, uso pretendido e usos fora de escopo, fatores relevantes, métricas, dados de avaliação e de treino, análises quantitativas por subgrupo, considerações éticas e ressalvas.

A Amazon Web Services (s.d.) inclui a governança entre os princípios de MLOps, ao lado do versionamento e da automação: sem ela, a automação do tópico 4 promove modelos que ninguém consegue auditar.

**No sistema do curso.** A aula de segurança desenvolve este tópico sobre o sistema empacotado.

### 15. Tópico 10 — Cultura, processos e maturidade

MLOps envolve papéis que DevOps não tem. Kreuzberger, Kühl e Hirschl (2023) identificam, entre outros:

| Papel | Responsabilidade |
|---|---|
| **Cientista de dados** | formular o problema como problema de ML, explorar dados, desenvolver e avaliar modelos |
| **Engenheiro de dados** | construir e operar a ingestão e a preparação dos dados |
| **Engenheiro de ML** | levar o modelo à produção: *pipelines*, *serving*, monitoramento |
| **DevOps / plataforma** | a infraestrutura, a CI/CD e a operação geral |
| **Negócio** | definir o objetivo e a métrica de sucesso, e decidir a adoção |

O atrito entre esses papéis é o muro entre desenvolvimento e operação, descrito na nota sobre DevOps, com mais um lado: o cientista de dados é avaliado pela métrica do modelo no *notebook*, o engenheiro pela estabilidade do serviço, e o modelo é "jogado por cima do muro" entre os dois.

**Níveis de maturidade.** O Google Cloud (2020) descreve três níveis de automação de MLOps:

| | **Nível 0** — processo manual | **Nível 1** — automação do *pipeline* de ML | **Nível 2** — CI/CD do *pipeline* |
|---|---|---|---|
| Treino | manual, em *notebook*, executado por pessoa | automático, pelo *pipeline* implantado (CT) | automático, pelo *pipeline* implantado (CT) |
| O que se implanta | o modelo, entregue à equipe de operação | o *pipeline* de treino | o *pipeline*, ele próprio construído e testado por CI/CD |
| Frequência de liberação | rara — poucas vezes por ano | modelo novo a cada execução do CT | *pipeline* novo a cada mudança de código do *pipeline* |
| Validação | manual | automática, de dados e de modelo | automática, de dados, de modelo e do código do *pipeline* |
| Monitoramento de modelo | ausente | presente, com gatilho de retreino | presente, com gatilho de retreino |
| Componentes | — | *feature store*, metadados de execução, registro de modelos | os do nível 1, mais sistema de CI/CD do *pipeline* |
| Separação entre papéis | total: cientista entrega o artefato | menor: o *pipeline* é compartilhado | mínima: experimentação e produção usam o mesmo código |

A passagem do nível 0 ao 1 automatiza o **treino**; a do 1 ao 2 automatiza a **mudança do *pipeline***. Um sistema no nível 1 retreina sozinho, mas uma alteração na forma de treinar ainda é implantada manualmente.

Há outros modelos de maturidade; o da Microsoft (s.d.) tem cinco níveis, de 0 (sem MLOps) a 4 (operações de MLOps totalmente automatizadas). A escala do Google é a usada nesta aula por ser a mais citada.

### 16. Onde o curso trata cada tópico

O sistema construído no curso não treina modelo próprio. Várias práticas de MLOps foram executadas nele sob outros nomes, adaptadas a esse caso, e as restantes são aplicadas nas aulas de produção e de segurança.

| Tópico | Onde | O que se fez ou se fará |
|---|---|---|
| 1. Gestão de dados | aula de *evals*; aula de *embeddings* e RAG | o *golden dataset* revisado e versionado; o índice como dado que muda o comportamento sem mudança de código |
| 2. Experimentação | aula de *prompt engineering*; aula de *evals* | a comparação entre versões de prompt pela suíte, com o carimbo registrando a combinação avaliada |
| 3. Registro e linhagem | aula de *prompt engineering*; o carimbo de versão ao longo do curso | o versionamento do prompt e o *prompt registry*; o carimbo como registro de linhagem |
| 4. *Pipelines* e CI/CD/CT | aula de *evals* | a suíte de regressão como portão antes de cada mudança; não há CT, porque não há treino |
| 5. *Deploy* e *serving* | nesta aula, a nota sobre de *script* a imagem | a imagem Docker, a execução em container, o segredo e o estado no container efêmero |
| 6. Monitoramento | aula de observabilidade | telemetria em convenção aberta, painel, alerta, avaliação online e custo |
| 7. Testes e qualidade | aula de *evals* | os quatro níveis de avaliação e o juiz validado |
| 8. Infraestrutura | aula de escolha e configuração de modelos; nesta aula, a nota sobre de *script* a imagem | modelo hospedado ou local; a infraestrutura mínima do agente em container |
| 9. Governança e segurança | aula de segurança | o tópico desenvolvido sobre o sistema empacotado |
| 10. Cultura e maturidade | esta nota | os papéis e os níveis 0, 1 e 2 |

**Já visto, na aula de *evals*:** os testes e a qualidade de modelo (os quatro níveis de avaliação e o juiz validado), o *golden dataset* versionado como gestão de dados, e a suíte de regressão como portão antes de cada mudança — a forma que a integração contínua toma num sistema com LLM.

**Já visto em outras aulas:** o versionamento do prompt e o *prompt registry*, na aula de *prompt engineering*; o índice como dado que muda o comportamento sem mudança de código, na aula de *embeddings* e RAG; a depreciação de modelo pelo provedor, na aula de escolha e configuração de modelos.

**Nesta aula e nas de observabilidade e segurança:** *deploy*, *serving* e conteinerização, na nota sobre de *script* a imagem; monitoramento e observabilidade, na aula de observabilidade; governança e segurança, na aula de segurança.

---

## Fontes e leituras

AMAZON WEB SERVICES. **O que é MLOps?** Disponível em: https://aws.amazon.com/pt/what-is/mlops/. Acesso em: 14 out. 2026.

BRASIL. **Lei nº 13.709, de 14 de agosto de 2018**. Lei Geral de Proteção de Dados Pessoais (LGPD). Brasília, DF: Presidência da República, 2018.

BRECK, E. et al. The ML Test Score: a rubric for ML production readiness and technical debt reduction. In: IEEE INTERNATIONAL CONFERENCE ON BIG DATA, 2017. **Proceedings** [...]. IEEE, 2017.

GOOGLE CLOUD. **MLOps**: continuous delivery and automation pipelines in machine learning. Cloud Architecture Center, 2020. Disponível em: https://cloud.google.com/architecture/mlops-continuous-delivery-and-automation-pipelines-in-machine-learning.

IBM. **O que é MLOps?** IBM Think. Disponível em: https://www.ibm.com/br-pt/think/topics/mlops. Acesso em: 14 out. 2026.

KREUZBERGER, D.; KÜHL, N.; HIRSCHL, S. Machine Learning Operations (MLOps): overview, definition, and architecture. **IEEE Access**, v. 11, 2023.

MICROSOFT. **Machine learning operations maturity model**. Azure Architecture Center. Disponível em: https://learn.microsoft.com/azure/architecture/ai-ml/guide/mlops-maturity-model.

MITCHELL, M. et al. Model cards for model reporting. In: CONFERENCE ON FAIRNESS, ACCOUNTABILITY, AND TRANSPARENCY (FAT*), 2019. **Proceedings** [...]. ACM, 2019.

SCULLEY, D. et al. Hidden technical debt in machine learning systems. In: ADVANCES IN NEURAL INFORMATION PROCESSING SYSTEMS 28 (NeurIPS), 2015.

**Material da disciplina.** [Nota sobre DevOps](01-devops.md) — o muro, os princípios e as práticas que MLOps estende. *Prompt engineering*, [nota sobre o prompt como código](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) — versionamento do prompt e *prompt registry*. *Evals*, [nota sobre por que avaliar é diferente](../../aula-11-evals/notas-de-aula/01-por-que-e-diferente.md), [nota sobre os quatro níveis](../../aula-11-evals/notas-de-aula/02-os-quatro-niveis.md) e [nota sobre o juiz e a suíte](../../aula-11-evals/notas-de-aula/03-o-juiz-e-a-suite.md) — *golden dataset*, testes de modelo, suíte de regressão e avaliação *offline* e *online*. Escolha e configuração de modelos, [nota sobre escolha de modelos](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — o *alias* sem versão e a depreciação. Observabilidade, [nota sobre monitoração e telemetria](../../aula-14-mlops-observabilidade/notas-de-aula/01-monitoracao-e-telemetria.md). Nesta aula, [nota sobre de *script* a imagem](03-de-script-a-imagem.md).
