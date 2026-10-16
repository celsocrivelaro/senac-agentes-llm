# IA Aplicada com LLMs — Aula 12: Governança de IA agêntica — Governar a ação

---

## Objetivos de aprendizagem

Ao final desta nota, espera-se que o aluno seja capaz de:

- **Definir** governança de IA agêntica e **distinguir** o risco de saída, próprio de modelos que produzem um resultado revisado por alguém, do risco de ação, próprio de sistemas que executam operações com autoridade delegada.
- **Explicar** o agente de IA como problema de agência — assimetria de informação, autoridade discricionária e lealdade — e por que a responsabilidade pelo agente permanece com a organização que o implanta (KOLT, 2025; SHAVIT et al., 2023).
- **Situar** os controles de um agente nas funções *Govern*, *Map*, *Measure* e *Manage* do NIST AI RMF e **distinguir** esse marco da ISO/IEC 42001, que é um sistema de gestão da organização.
- **Especificar** a autoridade de um agente — escopo, ações proibidas, identidade própria, privilégio mínimo, limites de gasto e limiares de aprovação humana — e **reconhecer** a credencial herdada, a confiança entre agentes e a deriva de autoridade como vias de ampliação dessa autoridade sem decisão explícita.
- **Localizar** onde cada limite é aplicado — no código do agente, na plataforma ou num ponto de aplicação comum — e **justificar** por que ação consequente exige limite determinístico ou aprovação humana, e não detecção de conteúdo.
- **Projetar** a supervisão humana de um agente — *human-in-the-loop* ou *human-on-the-loop*, limiar de escalonamento, procedimento de interrupção e matriz de responsabilidade.

---

## Desenvolvimento teórico

### 1. Definição

**Governança de IA agêntica** é o conjunto de políticas, controles técnicos e atribuições de responsabilidade com que uma organização gere a **autoridade delegada** a sistemas que planejam e executam ações em seu nome: o que esses sistemas podem acessar e fazer, sob que supervisão, e quem responde pelo que eles fazem. A definição tem três componentes, e cada um corresponde a uma pergunta que precisa ter resposta antes de o agente operar:

- **limite** — o que o agente está autorizado a fazer, com que dados e até que valor;
- **supervisão** — quem acompanha a operação, quando intervém e como a interrompe;
- **responsabilização** — quem responde por cada classe de falha, e com que registro se demonstra o que aconteceu.

A palavra central é **autoridade**. Um agente, no sentido construído na aula de arquitetura de agentes, é um laço em que o modelo escolhe a próxima ferramenta a chamar; cada ferramenta de escrita — registrar um parecer, enviar uma mensagem, efetuar um pagamento — é uma capacidade de alterar o estado do mundo que a organização concedeu ao sistema. Governar o agente é governar essa concessão.

Esta nota trata da primeira das duas perguntas que organizam o assunto: **o que o agente está autorizado a fazer, e quem aplica esse limite**, antes e durante a ação. A segunda — diante de uma decisão contestada, o sistema consegue dizer o que decidiu, com base em quê e quem responde por ela — é objeto da [nota sobre provar a decisão](02-provar-a-decisao.md), que trata de rastreabilidade e documentação. O texto legal — LGPD, ECA Digital, AI Act e PL 2338/2023 — e a correspondência entre cada obrigação e o controle que a atende estão na [nota sobre legislação](03-legislacao.md).

### 2. Risco de saída × risco de ação

**Risco de saída** é o risco de um sistema de IA produzir um resultado incorreto, enviesado ou inexplicável — uma classificação errada, um texto falso, uma previsão mal calibrada — que uma pessoa ou outro sistema consome e sobre o qual decide. **Risco de ação** é o risco de um sistema de IA **executar** uma operação indevida: alterar um registro, mover dinheiro, enviar uma comunicação, conceder um acesso.

A governança de modelos preditivos concentra-se no risco de saída: acurácia, viés, explicabilidade, deriva de dados. Entre a saída e a consequência há, em geral, uma decisão humana ou uma regra de negócio. Num agente, essa mediação desaparece no ponto em que o modelo escolhe uma ferramenta de escrita: o erro de raciocínio deixa de ser uma saída a revisar e passa a ser um efeito já ocorrido. A aula de segurança registrou a mesma diferença sob outro ângulo — um *chatbot* que alucina produz uma frase errada; um agente que alucina registra um parecer.

Shavit et al. (2023) tratam sistemas agênticos como objeto próprio de governança e ilustram o risco de ação com um assistente encarregado de comprar ingredientes para um *cheesecake* japonês que, em vez de comprar os ingredientes localmente, compra uma passagem aérea cara para o Japão, e o usuário só percebe quando o reembolso já não é possível. A análise do caso é o ponto relevante: vários participantes poderiam ter evitado o dano — o desenvolvedor do modelo, melhorando a confiabilidade; quem implantou o sistema, exigindo aprovação explícita antes de compras; o usuário, não delegando autoridade de compra a um sistema sabidamente pouco confiável; a companhia aérea, exigindo consentimento humano para a compra. Quando todos poderiam ter evitado, cada um pode atribuir a culpa aos demais, e a governança existe para que essa atribuição esteja feita antes do dano.

| | Risco de saída | Risco de ação |
|---|---|---|
| Objeto | o resultado produzido | a operação executada |
| Mediação até a consequência | decisão humana ou regra de negócio | nenhuma, se a ferramenta de escrita está disponível ao modelo |
| Reversão | descartar a saída | desfazer a operação, quando possível |
| Controle típico | avaliação do modelo, revisão da saída | limite de autoridade, aprovação, interrupção |

O risco de ação não substitui o de saída; soma-se a ele. O parecer do auditor de despesas pode estar errado (saída) e, se o agente tiver a ferramenta que efetiva o reembolso, o erro se converte em pagamento indevido (ação).

### 3. O agente como problema de agência

Na teoria econômica e no direito, um **problema de agência** (*principal-agent problem*) surge quando um **principal** delega a um **agente** a autoridade para agir em seu nome, e o agente dispõe de informação e de margem de decisão que o principal não controla integralmente. Kolt (2025) aplica o direito e a teoria da agência aos agentes de IA e identifica nesse quadro três problemas:

- **assimetria de informação** — o agente observa o ambiente, o conteúdo recuperado e o resultado das ferramentas; o principal vê, no máximo, o que o agente registra ou relata;
- **autoridade discricionária** — a tarefa delegada não especifica cada passo; o agente escolhe entre caminhos que o principal não enumerou, e é precisamente essa margem que justifica usar um agente em vez de um *workflow*;
- **lealdade** — o agente pode perseguir um objetivo diferente do pretendido pelo principal, por especificação incompleta, por deriva ao longo de uma trajetória longa ou por instrução injetada por terceiro.

Os três problemas já apareceram no curso com outros nomes: a assimetria é o motivo do *trace* da aula de arquitetura de agentes; a autoridade discricionária é o eixo do espectro de autonomia; a lealdade comprometida é a deriva de objetivo e, quando há atacante, o sequestro de objetivo catalogado na aula de segurança. Kolt (2025) argumenta ainda que os instrumentos convencionais com que se governa um agente humano — desenho de incentivos, monitoramento e sanção — podem ser insuficientes para agentes de IA, que operam com opacidade, em velocidade e em escala que nenhum monitoramento humano acompanha, e propõe que a infraestrutura técnica e jurídica atenda a três princípios: inclusividade, visibilidade e responsabilização (*liability*).

A consequência para a governança é que **a responsabilidade não se delega junto com a autoridade**. Um agente de IA não é pessoa jurídica nem patrimônio que responda por dano; responde quem o desenvolveu, quem o implantou ou quem o usou. Shavit et al. (2023) formulam o requisito de forma operacional: para todo dano direto causado por um sistema agêntico deve haver ao menos uma entidade humana — pessoa, empresa ou outra entidade jurídica, e não apenas o sistema — responsável por ele. A autonomia muda como o sistema opera; não muda quem responde por ele.

### 4. A moldura: NIST AI RMF e ISO/IEC 42001

O **NIST AI Risk Management Framework** (AI RMF 1.0, NIST AI 100-1) é um marco voluntário para a gestão de riscos de sistemas de IA, publicado pelo National Institute of Standards and Technology em janeiro de 2023 (NIST, 2023). O núcleo do marco organiza a gestão de risco em quatro funções, cada uma dividida em categorias e subcategorias:

| Função | Finalidade | Exemplo no agente |
|---|---|---|
| ***Govern*** | estabelecer cultura, políticas, papéis e responsabilização pela gestão de risco; função transversal, que permeia as outras três | quem aprova a autoridade do agente; quem pode desligá-lo |
| ***Map*** | estabelecer o contexto: finalidade, ambiente de implantação, partes afetadas e impactos possíveis | que despesas o agente analisa, quem é afetado por um parecer |
| ***Measure*** | analisar e acompanhar riscos e confiabilidade com testes, métricas e avaliações | a suíte da aula de *evals*, a suíte hostil da aula de segurança, o painel da aula de observabilidade |
| ***Manage*** | priorizar os riscos mapeados e medidos e decidir a resposta: mitigar, transferir, aceitar, e recuperar-se de incidentes | limites, aprovação humana, procedimento de interrupção, registro de risco aceito |

O próprio NIST descreve *Govern* como função transversal (*cross-cutting*): não é uma etapa anterior às outras, e sim a condição que dá a elas responsável, critério e registro. Em 2024, o NIST publicou o **perfil de IA generativa** (NIST AI 600-1), que aplica o AI RMF a sistemas generativos: lista doze riscos próprios ou agravados pela IA generativa — entre eles confabulação, privacidade de dados, segurança da informação, configuração humano-IA (que inclui o viés de automação e a dependência excessiva) e integridade da cadeia de valor e de componentes — e propõe ações sugeridas organizadas pelas funções e subcategorias do marco (NIST, 2024). O valor do AI RMF para esta aula é o vocabulário comum: cada controle discutido adiante tem lugar numa das quatro funções, e a ausência de controles numa delas é uma lacuna visível.

A **ISO/IEC 42001:2023** é de outra natureza. É uma norma de **sistema de gestão** (*management system*): especifica requisitos para estabelecer, implementar, manter e melhorar continuamente um sistema de gestão de IA no contexto de uma organização, na mesma família estrutural de normas como a ISO/IEC 27001 de segurança da informação, com ciclo planejar-executar-verificar-agir e um anexo de controles de referência (ISO; IEC, 2023). A norma é certificável, e o que se certifica é **a organização** — seus processos de definição de política, avaliação de risco, operação e melhoria —, e não um modelo nem um agente específico. Um certificado ISO/IEC 42001 atesta que a organização tem processo para governar seus sistemas de IA; não atesta que um dado agente esteja corretamente limitado.

| | NIST AI RMF | ISO/IEC 42001 |
|---|---|---|
| Natureza | marco voluntário de gestão de risco | norma de requisitos para sistema de gestão |
| Objeto | riscos de sistemas de IA ao longo do ciclo de vida | o sistema de gestão de IA da organização |
| Certificação | não há | certificável por terceira parte |
| Uso nesta aula | vocabulário para situar cada controle | referência de que a governança é processo contínuo da organização |

### 5. Autoridade delegada

#### 5.1 Escopo e ações proibidas, declarados

A **especificação de autoridade** de um agente é o documento que declara, antes da implantação, o que o agente faz, o que nunca faz e com que dados. Tem três partes:

- **escopo** — a finalidade, as ferramentas disponíveis e, para cada uma, se é de leitura ou de escrita e se a escrita é reversível;
- **ações proibidas** — as operações que o agente não executa em nenhuma circunstância, ainda que tecnicamente possíveis;
- **dados** — as fontes que o agente lê e as categorias de dado pessoal que podem entrar no contexto.

A lista de ações proibidas não é redundante com o escopo. O escopo diz quais ferramentas existem; a proibição restringe o uso de uma ferramenta existente — o auditor de despesas tem a ferramenta `registrar_parecer`, e a proibição de aprovar acima de um valor restringe um dos argumentos dela. E uma proibição só tem efeito se for aplicada fora do modelo: escrita no prompt, é instrução concorrente com qualquer outra instrução que chegue ao contexto, e a aula de segurança mostrou que prompt não é mecanismo de segurança. Formalmente, seja $\mathcal{A}$ o conjunto de ações que as ferramentas do agente permitem executar, $E \subseteq \mathcal{A}$ o escopo declarado e $P \subseteq \mathcal{A}$ o conjunto de ações proibidas; o conjunto autorizado é

$$\mathcal{A}_{\text{aut}} = E \setminus P$$

e o controle consiste em garantir, em código, que toda ação executada pertença a $\mathcal{A}_{\text{aut}}$ — e não em pedir ao modelo que se mantenha nele.

#### 5.2 Identidade própria e privilégio mínimo

O agente opera com uma **identidade própria**: uma credencial de serviço emitida para ele, distinta da credencial do usuário que o aciona e da do desenvolvedor que o construiu. Sobre essa identidade aplica-se o **princípio do privilégio mínimo**: a credencial concede exatamente as permissões necessárias às ferramentas do escopo, e nenhuma outra.

A identidade própria resolve dois problemas. O primeiro é de **atribuição**: se o agente usa a credencial de uma pessoa, o registro do sistema atribui à pessoa as ações do agente, e não há como distinguir no *log* o que um fez do que o outro fez. O segundo é de **alcance**: a credencial de um desenvolvedor ou de um usuário com perfil amplo concede ao agente tudo o que essa pessoa pode fazer, e qualquer desvio do agente — erro de raciocínio, instrução injetada — opera com esse alcance. O catálogo OWASP Top 10 for Agentic Applications 2026 descreve exatamente essa situação em **ASI03 — Identity and Privilege Abuse**: sem uma identidade própria e governada, o agente opera numa lacuna de atribuição que torna impossível aplicar de fato o privilégio mínimo (OWASP, 2025).

O privilégio mínimo se aplica por ferramenta, e não só por agente. A ferramenta de leitura de despesas não precisa de permissão de escrita; a de registro de parecer escreve numa única tabela; nenhuma das duas precisa de acesso à folha de pagamento. A aula de segurança classificou essa medida como a defesa mais barata disponível, e aqui ela reaparece como requisito de governança: uma permissão não justificada pelo escopo é autoridade concedida sem decisão.

#### 5.3 Credencial herdada e confiança entre agentes

Duas vias ampliam a autoridade de um agente sem que ninguém a tenha concedido explicitamente. O catálogo OWASP as registra entre os exemplos de ASI03 (OWASP, 2025):

- **Credencial herdada** — o agente recebe, por conveniência ou limitação de arquitetura, o contexto de acesso integral de quem o aciona: um agente gerente com privilégio alto delega uma subtarefa a um agente trabalhador e repassa todas as suas permissões, e o trabalhador, que precisava de uma consulta, passa a poder tudo. Inclui a credencial guardada em memória ou em cache e reutilizada numa sessão posterior, por outro usuário.
- **Confiança entre agentes** — em sistemas multiagente, um agente aceita pedidos de outro agente interno como se fossem legítimos por virem de dentro. Um agente de baixo privilégio comprometido retransmite instruções de aparência válida a um agente de alto privilégio, que as executa sem verificar a intenção do usuário original — o problema clássico do **representante confuso** (*confused deputy*). O catálogo descreve o caso de uma mensagem forjada que instrui um agente de triagem de correio a pedir a um agente financeiro uma transferência; o agente financeiro, confiando no agente interno, executa o pagamento.

O mesmo catálogo registra uma terceira via, temporal, que ilustra bem o caso de um agente de compras com limite: a autorização é verificada **no início** de uma sequência de compra; horas depois, o limite de gasto do usuário é reduzido, mas o *workflow* prossegue com o *token* de autorização antigo e conclui uma transação que já não estava autorizada. É a condição de corrida conhecida como *time-of-check to time-of-use* (TOCTOU), e a correção é verificar a autorização no momento da ação, e não no início da trajetória.

A regra comum às três vias é que **a autoridade de um agente não pode ser maior do que a concedida a ele**, independentemente de quem o acionou. Na delegação, o agente delegado recebe uma credencial restrita à subtarefa — a interseção, e não a união, das permissões. Na comunicação entre agentes, a mensagem de outro agente é entrada não confiável, e o pedido que ela carrega é verificado contra a autoridade do usuário original. O endereço dessa fronteira no curso é o *handoff* e o *Agent Card* do protocolo A2A, tratados na [nota sobre contratos e protocolos](../../aula-16-sistemas-multiagente/notas-de-aula/02-do-contrato-ao-protocolo.md) da aula de sistemas multiagente; a comunicação insegura entre agentes é a categoria **ASI07 — Insecure Inter-Agent Communication** do mesmo catálogo.

#### 5.4 Deriva de autoridade

**Deriva de autoridade** é o crescimento da autoridade efetiva de um agente sem decisão explícita de ampliá-la. Ocorre por acréscimos que, isoladamente, parecem mudanças de configuração:

- uma **ferramenta nova** acrescentada ao agente para atender a um caso específico;
- um **servidor MCP novo** conectado, que expõe várias ferramentas de uma vez, inclusive de escrita;
- uma **atualização de um servidor MCP já aprovado**, cuja descrição ou conjunto de ferramentas muda depois da aprovação — o *rug pull* da [nota sobre o preço do protocolo](../../aula-10-mcp/notas-de-aula/03-o-preco-do-protocolo.md), da aula de MCP;
- uma **permissão ampliada** na credencial de serviço para resolver um erro de acesso;
- um **prompt novo** que muda o comportamento do agente sem mudar nenhuma ferramenta.

A formalização é a da §5.1, agora com o tempo. Seja $\mathcal{A}_{\text{dec}}$ a autoridade declarada na especificação e $\mathcal{A}_{\text{ef}}(t)$ a autoridade efetiva no instante $t$ — o conjunto de ações que as ferramentas conectadas, com as credenciais concedidas, permitem executar. A especificação está cumprida enquanto

$$\mathcal{A}_{\text{ef}}(t) \subseteq \mathcal{A}_{\text{dec}}$$

e cada acréscimo da lista acima aumenta $\mathcal{A}_{\text{ef}}$ sem alterar $\mathcal{A}_{\text{dec}}$. A deriva é a diferença $\mathcal{A}_{\text{ef}}(t) \setminus \mathcal{A}_{\text{dec}}$, que cresce de forma monotônica enquanto ninguém a mede. O controle tem duas partes: toda mudança que altere ferramentas, servidores conectados ou permissões passa pela mesma revisão que aprovou a especificação; e, periodicamente, a autoridade efetiva é recalculada a partir do que está de fato conectado e comparada com a declarada. É o mesmo problema do desvio de configuração da infraestrutura como código, discutido na aula de empacotamento, aplicado a permissões.

#### 5.5 Visibilidade como pré-requisito

Nenhum dos controles anteriores pode ser verificado sem que se saiba que agentes existem, o que fazem e em nome de quem. Chan et al. (2024) definem **visibilidade** sobre agentes de IA como o conhecimento sobre onde, por que, como e por quem os agentes são usados, e examinam três categorias de medidas, de intrusividade e informatividade crescentes:

| Medida | Definição | No agente do curso |
|---|---|---|
| **identificador do agente** | marca que indica, a quem interage com o agente, que se trata de um agente e permite associá-lo a quem o implantou | a credencial de serviço própria e o identificador do agente em cada chamada e em cada registro |
| **monitoramento em tempo real** | supervisão da atividade do agente enquanto ela ocorre, com possibilidade de sinalizar ou bloquear ações | o painel e o alerta da aula de observabilidade; o ponto de aplicação de política da §6 |
| **registro de atividade** | guarda das entradas e saídas do agente para análise posterior | o *trace* persistido em todos os caminhos de término, da aula de arquitetura de agentes |

Os autores analisam essas medidas em implantações centralizadas e descentralizadas e discutem os custos de privacidade e de concentração de poder que elas implicam (CHAN et al., 2024). Para a governança dentro de uma organização, a conclusão operacional é que **um agente sem identificador, sem monitoramento e sem registro não é governável**: não há como saber se ele cumpre a especificação, nem reconstruir o que fez quando não cumpriu. O registro de atividade como evidência — o que guardar para reconstruir uma decisão, e por quanto tempo — é tratado na [nota sobre provar a decisão](02-provar-a-decisao.md).

### 6. Controle em tempo de execução

#### 6.1 Política declarada × política aplicada

Uma **política declarada** é uma regra escrita — na especificação de autoridade, num documento de governança, num prompt. Uma **política aplicada** é uma regra executada por um componente que intercepta a operação e a permite, nega ou suspende. O componente que aplica a regra é o **ponto de aplicação de política** (*policy enforcement point*). A governança escreve a regra; o ponto de aplicação a executa, e uma regra sem ponto de aplicação é intenção, não controle.

Num agente, as operações que atravessam uma fronteira são quatro, e cada uma precisa de ponto de aplicação:

- **chamada de modelo** — que modelo pode ser chamado, com que dados no contexto, dentro de que orçamento;
- **chamada de ferramenta** — que ferramenta, com que argumentos, com que credencial;
- **chamada a servidor MCP** — que servidores estão aprovados, em que versão, com que ferramentas expostas;
- **chamada a outro agente** (A2A) — a quem se delega, com que autoridade, e o que se aceita de volta.

```text
                      +------------------------------------+
 agente ------------> |   ponto de aplicação de política   |
 (identidade          |------------------------------------|
  própria)            | identidade: quem chama             |
                      | escopo: pode chamar isto           |
 humano <-- pausa --- | orçamento: cabe na janela          |
 (aprovador)          | conteúdo: guardrail de entrada     |
                      | limiar: exige aprovação humana     |
                      | registro: decisão e versão         |
                      +------------------------------------+
                          |          |          |          |
                          v          v          v          v
                       modelo   ferramenta  servidor    outro
                      aprovado   interna       MCP      agente
                                                        (A2A)

       o retorno de cada destino volta pelo mesmo ponto
       e é entrada não confiável para o agente
```

A decisão do ponto de aplicação é uma função determinística da ação proposta, da identidade de quem a propõe e do estado acumulado. Em esboço:

```python
def decidir(acao, agente, estado):
    if acao.ferramenta not in agente.escopo:
        return "negar", "fora do escopo"
    if agente.proibe(acao):
        return "negar", "ação proibida"
    if estado.gasto_janela + acao.custo_estimado > agente.limite_janela:
        return "negar", "limite de gasto da janela"
    if acao.impacto() >= agente.limiar_aprovacao:
        return "pausar", "aguarda aprovação humana"
    return "permitir", None
```

Três propriedades decorrem do esboço. O modelo não participa da decisão: os argumentos vêm da chamada de ferramenta que o modelo propôs, mas a regra é código, e nenhum texto no contexto a altera — é a confirmação "inescapável por prompt" da [nota sobre defesa por arquitetura](../../aula-15-seguranca/notas-de-aula/02-defesa-por-arquitetura.md), da aula de segurança. A função devolve o **motivo**, pelo mesmo argumento do orçamento da [nota sobre confiabilidade](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md), da aula de arquitetura de agentes: negar por escopo e negar por gasto exigem correções opostas. E a pausa é um resultado próprio, que corresponde ao término "aguardando humano" da taxonomia de término daquela aula.

O ponto de aplicação pode estar em três lugares: **no código do agente** (o `Orcamento` e a confirmação construídos em arquitetura de agentes), **na plataforma** que executa as ferramentas (a permissão da credencial de serviço no banco de dados, que nega a escrita independentemente do que o agente tente), ou **num plano de controle comum** a vários agentes. Os três não se excluem; a permissão na plataforma é a última linha, porque vale mesmo quando o código do agente falha.

#### 6.2 O *gateway* como desenho possível

Um **LLM *gateway*** é um serviço interposto entre as aplicações e os provedores de modelo: a aplicação aponta para o *gateway* em vez de apontar para o provedor, e o *gateway* encaminha a chamada. Quando concentra também as chamadas de ferramenta, MCP e A2A, funciona como **plano de controle comum**: as políticas, os segredos (as chaves dos provedores ficam nele, e não em cada aplicação) e a telemetria ficam num só lugar, e uma regra alterada passa a valer para todos os agentes sem reimplantação.

Janicki (2026), em texto de fornecedor indicado aqui como leitura complementar, descreve esse desenho: a governança define as regras, o *gateway* as aplica em cada requisição de modelo e registra que decisão foi tomada. O *gateway* não é a definição de governança nem condição necessária para ela; é um desenho que resolve um problema de escala — muitos agentes, muitas equipes, regras que precisam ser iguais para todos. Um único agente pode ter os mesmos controles no próprio código e nas permissões da plataforma. E o *gateway* tem custo próprio: fica no caminho crítico de toda chamada, e sua falha é a falha de todos os agentes que dependem dele (§6.5).

#### 6.3 Custo como política

O custo de um agente é uma quantidade que a governança limita, e não apenas uma métrica que a observabilidade mede. A [nota sobre a conta da aula de observabilidade](../../aula-14-mlops-observabilidade/notas-de-aula/03-a-conta-e-o-que-quebra.md) mediu o custo por execução e identificou onde ele mora — o reenvio do histórico e as declarações de ferramenta; aqui o custo passa a ser **limite com responsável**.

O `Orcamento` da aula de arquitetura de agentes limita **uma execução**: no máximo $T_{\max}$ tokens ou $R_{\max}$ reais por trajetória. Esse limite não limita o agregado. Se $n$ execuções ocorrem numa janela, o gasto da janela é limitado apenas por $n \cdot R_{\max}$, e $n$ não está sob controle do agente: uma fila reprocessada, um cliente que reenvia em laço ou um agente sequestrado que dispara execuções multiplicam o gasto sem que nenhuma execução exceda o próprio teto. O limite por janela fecha essa lacuna. Para cada chave $k$ e cada janela $w \in \{\text{dia}, \text{semana}, \text{mês}\}$:

$$G_w(k) = \sum_{e \,\in\, \text{execuções de } k \text{ em } w} c(e) \;\le\; L_w(k)$$

com um alerta disparado quando $G_w(k) \ge \alpha \, L_w(k)$, para $\alpha < 1$ (por exemplo, $\alpha = 0{,}8$), de modo que alguém seja avisado antes de o limite interromper o serviço. Os componentes da política são:

- **limites por janela**, sobrepostos — diário, semanal e mensal —, porque um limite só mensal permite consumir o mês num dia;
- **limites por chave**, com uma chave por carga de trabalho, e não uma chave compartilhada por todos os agentes: a chave é a unidade de atribuição e de corte, e uma chave compartilhada torna impossível dizer qual agente gastou nem desligar só ele;
- **limites por equipe ou por usuário**, quando o agente atende a vários;
- **roteamento só entre modelos aprovados** — o roteamento de modelo da aula de observabilidade reduz custo enviando o fácil ao modelo barato, e a política restringe esse roteamento à lista de modelos cuja política de dados, região de processamento e avaliação foram aprovadas;
- **o contexto como alavanca** — o custo de uma chamada é proporcional ao contexto enviado, e o que reduz o contexto (encurtar o retorno das ferramentas, *prompt caching* com a fronteira correta) é também o que reduz a exposição de dado ao provedor.

Um efeito colateral do limite é útil à governança: um aumento súbito de gasto, ou de chamadas negadas por política, é sinal de mau funcionamento ou de comprometimento do agente. A aula de segurança já havia relido o teto de gasto como **contenção de dano** — ele limita quanto dano cabe antes de alguém perceber.

#### 6.4 *Guardrails* reduzem, não eliminam

Um ***guardrail*** é um verificador de conteúdo aplicado à entrada ou à saída de uma chamada de modelo, que bloqueia, mascara ou sinaliza o conteúdo que viola uma regra. Há duas famílias:

- **detecção por padrão** — expressões regulares e validadores para dados estruturados: CPF, número de cartão, chave de API, endereço de *e-mail*. É precisa no que tem formato fixo e cega ao que não tem;
- **detecção por modelo** — um classificador para conteúdo não estruturado: nome de pessoa, condição de saúde descrita em prosa, tentativa de *jailbreak* ou de injeção.

As duas falham em parte dos casos, nos dois sentidos: falso positivo, que bloqueia uma operação legítima, e falso negativo, que deixa passar o que deveria ter sido bloqueado. Para a detecção por modelo, a [nota sobre o mapa de riscos e a injeção](../../aula-15-seguranca/notas-de-aula/01-o-mapa-e-a-injecao.md), da aula de segurança, mostrou por que o falso negativo é estrutural no caso da injeção: a instrução maliciosa não precisa parecer maliciosa, basta parecer instrução.

A consequência é uma regra de projeto: **ação consequente depende de limite determinístico ou de aprovação humana, e não de detecção de conteúdo**. Seja $p$ a taxa de falso negativo do *guardrail* e $N$ o número de tentativas de ação indevida numa janela; o número esperado de ações indevidas que atravessam o *guardrail* é $pN$, que é positivo para qualquer $p > 0$ e cresce com a exposição. Um limite determinístico — "nenhuma aprovação acima de R\$ 1.000 sem humano" — não tem taxa de falso negativo para a regra que expressa: a ação acima do valor é suspensa, qualquer que seja o texto que a motivou. O *guardrail* continua útil para reduzir o volume do que chega ao limite e para mascarar dado pessoal antes de ele seguir para o provedor; não é o controle sobre a ação. Janicki (2026) registra a mesma conclusão.

#### 6.5 *Fail-open* × *fail-closed*

Todo controle no caminho de uma operação pode falhar — o *gateway* fica indisponível, o serviço de *guardrail* excede o tempo limite, a consulta ao limite de gasto não responde. O comportamento do sistema nessa situação é uma decisão de projeto, e precisa ser declarada:

- ***fail-open*** — se o controle não responde, a operação prossegue sem ele. Preserva a disponibilidade e perde o controle durante a falha;
- ***fail-closed*** — se o controle não responde, a operação é negada. Preserva o controle e perde a disponibilidade durante a falha.

A escolha segue o risco da operação, e não uma preferência geral. Se o controle tem disponibilidade $d$, uma operação *fail-closed* tem disponibilidade no máximo $d$ — o controle passa a ser dependência crítica; uma operação *fail-open* fica sem controle numa fração $1-d$ do tempo. Para leitura de baixo risco, *fail-open* com registro da falha é defensável. Para escrita consequente — registrar parecer, pagar, enviar —, a regra é *fail-closed*: a operação aguarda o controle voltar, ou segue para aprovação humana. Uma configuração *fail-open* em operação consequente equivale a declarar que o limite vale apenas enquanto o controle está no ar.

O mesmo raciocínio se aplica ao **modelo de reserva** (*fallback*): quando o provedor principal falha, a chamada é desviada para outro modelo. O desvio só é admissível se o modelo de reserva cumprir **as mesmas políticas** do principal — mesma política de retenção e de uso para treino, mesma região de processamento, mesma avaliação na suíte da aula de *evals* (JANICKI, 2026). Um *fallback* para um provedor não aprovado é um *fail-open* da política de dados: a disponibilidade é preservada enviando o dado para onde a política não permitia.

### 7. Supervisão humana e desligamento

#### 7.1 *Human-in-the-loop* × *human-on-the-loop*

Há duas formas de supervisão humana de um agente, e elas diferem no momento em que a pessoa intervém:

- ***human-in-the-loop*** — uma pessoa **aprova** a ação antes de ela ocorrer; a execução fica suspensa até a decisão. É a confirmação humana da aula de arquitetura de agentes e o término "aguardando humano";
- ***human-on-the-loop*** — o agente age sem aprovação prévia, e uma pessoa **monitora** a operação com poder de intervir: suspender, reverter, corrigir.

Shavit et al. (2023) tratam a exigência de aprovação como forma corrente de limitar falhas graves — certas decisões são importantes demais para delegar, como iniciar sozinho uma transação financeira grande e irreversível — e apontam o limite dessa forma: a pessoa precisa de contexto suficiente para entender o que aprova, e quem aprova muitas decisões precisa decidir cada uma depressa, o que reduz a capacidade de examiná-las. A aprovação em excesso degenera em carimbo. O AI Act registra o mesmo fenômeno ao exigir que quem supervisiona um sistema de alto risco permaneça consciente da tendência de confiar automaticamente na saída do sistema — o **viés de automação** — (UNIÃO EUROPEIA, 2024, art. 14, n. 4, b; ver a [nota sobre legislação](03-legislacao.md), §4.3).

A escolha entre as duas formas não é global: o mesmo agente usa *in-the-loop* para as ações de alto impacto e *on-the-loop* para as demais, e o critério que separa as duas classes é o limiar de escalonamento.

#### 7.2 Limiar de escalonamento

O **limiar de escalonamento** é a regra que determina quais ações suspendem a execução e seguem para aprovação humana, e para quem. É formulado sobre propriedades verificáveis da ação, e não sobre a confiança declarada pelo modelo, que é texto gerado e pode ser manipulada. Três propriedades cobrem a maior parte dos casos:

- **valor** — o montante envolvido, comparado com um limite numérico;
- **irreversibilidade** — se a operação pode ser desfeita, e a que custo; um parecer registrado e ainda não efetivado é reversível, um pagamento executado não é;
- **dado sensível** — se a operação envolve dado pessoal sensível (saúde, por exemplo) ou afeta direitos de uma pessoa.

O limiar define também **o destino** da pausa: o papel que aprova (o analista financeiro, o gestor da área), o prazo de resposta e o que ocorre se o prazo vencer — e, pela §6.5, o vencimento do prazo numa ação consequente resulta em não executar, e não em executar por omissão. A calibragem segue a advertência da nota sobre confiabilidade: confirmação em excesso suprime a autonomia que justificava o agente, e confirmação de menos deixa passar a ação que justificava o controle. O limiar é parâmetro revisado com dados — a fração de execuções que pausam, o tempo de espera, a fração de aprovações que o humano altera.

#### 7.3 Interromper o sistema

**Interruptibilidade** é a capacidade de suspender um agente em operação, de forma controlada, a qualquer momento. Shavit et al. (2023) a consideram uma salvaguarda rudimentar e crítica, e distinguem dois níveis: interromper uma categoria específica de ação — por exemplo, revogar o acesso a credenciais financeiras — e encerrar a operação do agente como um todo. Apontam ainda três requisitos: o desligamento deve ser gracioso, com procedimento de contingência preparado para o caso de a interrupção ocorrer no meio de uma sequência de ações; o agente não pode ser capaz de impedir ou adulterar a tentativa de desligá-lo; e a interrupção deve se estender aos subagentes que o agente iniciou.

O AI Act incorpora a interrupção como requisito de projeto para sistemas de alto risco: entre as capacidades que a supervisão humana precisa ter, está a de **intervir no funcionamento ou interromper o sistema** por meio de um botão de parada ou procedimento semelhante que o leve a um estado seguro (UNIÃO EUROPEIA, 2024, art. 14, n. 4, e). O texto do art. 14, a classificação de um sistema como de alto risco e as datas de aplicação dessas obrigações estão na [nota sobre legislação](03-legislacao.md) (§§4.1, 4.3 e 4.5); o requisito técnico de interrupção vale como boa prática independentemente da classificação.

A especificação de interrupção responde a quatro perguntas:

| Pergunta | Exemplo de resposta |
|---|---|
| quem pode interromper | o operador de plantão e o responsável pelo produto |
| como | desativar a chave do agente no ponto de aplicação; revogar a credencial de serviço na plataforma |
| em que estado o sistema fica | execuções em curso terminam como interrompidas, sem escrita parcial; o *trace* é persistido |
| como se testa | exercício periódico de desligamento em homologação, com tempo medido até a última ação |

O teste é a parte que costuma faltar. Um procedimento de interrupção nunca executado é uma hipótese: a chave desativada pode não ser a que o agente usa, a credencial revogada pode estar em cache, uma execução em segundo plano pode continuar. Os dois mecanismos da tabela são complementares, pela mesma razão da §6.1: a desativação no ponto de aplicação é rápida e seletiva; a revogação na plataforma vale mesmo que o agente contorne o ponto de aplicação.

#### 7.4 Matriz de responsabilidade

A **matriz de responsabilidade** atribui, a cada classe de falha, o papel que responde por ela e os papéis que participam da resposta. É a forma operacional do princípio da §3: a responsabilidade não se delega ao agente, e por isso precisa estar distribuída entre pessoas antes do incidente. Os papéis típicos são quatro — **produto** (define finalidade, escopo e limiar), **desenvolvimento** (implementa ferramentas, limites e registro), **operação** (monitora, responde a alertas, executa a interrupção) e **risco** (aprova a especificação, aceita riscos residuais, revisa a deriva).

| Falha | Responde | Participa |
|---|---|---|
| ação fora do escopo executada | desenvolvimento (o limite não foi aplicado) | risco (a especificação foi aprovada assim) |
| escopo inadequado à finalidade | produto | risco |
| limite de gasto excedido sem alerta | operação | desenvolvimento |
| agente não interrompido a tempo | operação | produto (quem decide interromper) |
| permissão acumulada sem revisão | risco | desenvolvimento |
| aprovação humana dada sem exame | produto (desenho do limiar e da tela de aprovação) | operação |

A coluna "responde" tem um único papel por linha; a ambiguidade nessa coluna reproduz a situação descrita por Shavit et al. (2023), em que cada participante pode atribuir a falha aos demais.

### 8. A autoridade do auditor de despesas, especificada

O auditor de despesas do curso analisa despesas submetidas por funcionários contra o regulamento interno e registra um parecer. As ferramentas são as construídas nas aulas de arquitetura de agentes e de MCP: `consultar_despesa`, `consultar_politica` e `buscar_regulamento`, de leitura, e `registrar_parecer`, de escrita, idempotente por chave derivada do conteúdo. A especificação abaixo aplica as seções anteriores a esse agente; os valores numéricos são ilustrativos e seriam fixados pelo responsável pelo produto com base na política da organização e nas medições de custo da aula de observabilidade.

| Item | Especificação | Onde é aplicado |
|---|---|---|
| **Finalidade** | analisar despesas submetidas e registrar parecer (aprovar, recusar ou encaminhar), com justificativa que cite o trecho do regulamento | especificação; prompt de sistema |
| **Ferramentas (escopo)** | `consultar_despesa`, `consultar_politica`, `buscar_regulamento` (leitura); `registrar_parecer` (escrita reversível: o parecer não efetiva pagamento) | lista de ferramentas do agente; ponto de aplicação |
| **Servidores MCP** | apenas o servidor da política, em versão fixada; servidor novo ou nova versão passa por revisão | ponto de aplicação; revisão de deriva |
| **Dados** | a despesa analisada e seus comprovantes; o regulamento. Não acessa folha de pagamento nem despesas de outros funcionários | permissões da credencial na plataforma |
| **Ações proibidas** | alterar a política ou o regulamento; efetivar reembolso ou pagamento; aprovar despesa acima de R\$ 1.000 sem aprovação humana; registrar parecer sem trecho citado | ausência da ferramenta (alterar, pagar); regra no ponto de aplicação (valor, citação) |
| **Credencial** | conta de serviço própria do auditor: leitura em despesas e regulamento, escrita apenas na tabela de pareceres; nunca a credencial do funcionário nem a do desenvolvedor; segredo fora do repositório, com rotação | plataforma (banco de dados, cofre de segredos) |
| **Delegação** | o auditor não delega a outro agente; se vier a delegar, o delegado recebe credencial restrita à subtarefa | especificação; revisão de deriva |
| **Limite por execução** | 60.000 tokens e R\$ 0,50 por execução, 12 passos, 120 s | `Orcamento` no código do agente |
| **Limite por janela** | chave exclusiva do auditor; R\$ 120 por mês (cerca de 1,5 vez o custo mensal medido), R\$ 10 por dia; alerta ao atingir 80% de cada limite | ponto de aplicação (*gateway* ou código) |
| **Modelos** | apenas os da lista aprovada; reserva somente com a mesma política de dados, região e resultado na suíte de *evals* | ponto de aplicação |
| **Limiar de aprovação humana** | pausa para o analista financeiro: toda recusa; toda aprovação acima de R\$ 1.000; toda despesa com dado de saúde. Prazo de 2 dias úteis; vencido o prazo, o parecer não é registrado | ponto de aplicação; término "aguardando humano" |
| **Falha de controle** | *fail-closed* para `registrar_parecer`; *fail-open* com registro para as ferramentas de leitura | ponto de aplicação |
| **Interrupção** | o operador de plantão desativa a chave do auditor e, se necessário, revoga a credencial de serviço; a retomada é decidida pelo responsável pelo produto; exercício de desligamento a cada trimestre | ponto de aplicação; plataforma |
| **Revisão de autoridade** | a cada ferramenta, servidor ou permissão nova, e a cada trimestre: autoridade efetiva comparada com a declarada | processo (*Govern*), responsável: risco |
| **Visibilidade** | identificador do agente em toda chamada e registro; painel e alerta de gasto e de negações; *trace* persistido em todo término | observabilidade |

Três decisões da tabela merecem justificativa. A **ausência de ferramenta** é o controle mais forte das ações proibidas: o auditor não pode efetivar reembolso porque não existe ferramenta para isso, e nenhuma instrução no contexto cria uma. A **recusa vai sempre a humano** porque é a decisão que afeta um funcionário de forma adversa e a mais provável de ser contestada; o tratamento da contestação — o que o registro precisa conter para responder a ela, e o pedido de revisão de decisão automatizada previsto na LGPD — é o caso que a [nota sobre provar a decisão](02-provar-a-decisao.md) percorre (§7), e o texto do art. 20 está na [nota sobre legislação](03-legislacao.md) (§2.4). E o **limite por janela** é derivado da medição: o exemplo da nota sobre a conta, da aula de observabilidade, chega a cerca de R\$ 78 por mês para 1.900 execuções a R\$ 0,041 cada, e o limite mensal com folga de cerca de 50% sobre esse valor interrompe o anômalo sem interromper o crescimento normal; um limite fixado sem medição seria arbitrário nos dois sentidos.

A especificação também torna visível o que ainda não está coberto. Ela diz o que o agente pode fazer; não diz, sozinha, como demonstrar a um terceiro — o funcionário, um auditor, a autoridade de proteção de dados — o que o agente fez numa execução específica. Essa é a outra metade da governança.

---

## Fontes e leituras

CHAN, A.; EZELL, C.; KAUFMANN, M.; WEI, K.; HAMMOND, L.; BRADLEY, H.; BLUEMKE, E.; RAJKUMAR, N.; KRUEGER, D.; KOLT, N.; HEIM, L.; ANDERLJUNG, M. Visibility into AI agents. In: ACM CONFERENCE ON FAIRNESS, ACCOUNTABILITY, AND TRANSPARENCY (FAccT), 2024, Rio de Janeiro. **Proceedings** [...]. New York: ACM, 2024. Disponível em: https://arxiv.org/abs/2401.13138.

INTERNATIONAL ORGANIZATION FOR STANDARDIZATION; INTERNATIONAL ELECTROTECHNICAL COMMISSION. **ISO/IEC 42001:2023**: information technology — artificial intelligence — management system. Genebra: ISO, 2023.

KOLT, N. Governing AI agents. **Notre Dame Law Review**, 2025. Disponível em: https://arxiv.org/abs/2501.07913. (Conferir volume e paginação.)

NATIONAL INSTITUTE OF STANDARDS AND TECHNOLOGY (NIST). **Artificial Intelligence Risk Management Framework (AI RMF 1.0)**. NIST AI 100-1. Gaithersburg: NIST, 2023. Disponível em: https://doi.org/10.6028/NIST.AI.100-1.

NATIONAL INSTITUTE OF STANDARDS AND TECHNOLOGY (NIST). **Artificial Intelligence Risk Management Framework**: generative artificial intelligence profile. NIST AI 600-1. Gaithersburg: NIST, 2024. Disponível em: https://doi.org/10.6028/NIST.AI.600-1.

OWASP GENAI SECURITY PROJECT. **OWASP Top 10 for Agentic Applications 2026**. Versão 2026. [S. l.]: OWASP, dez. 2025. Disponível em: https://genai.owasp.org/resource/owasp-top-10-for-agentic-applications-for-2026/. Ver ASI03 (Identity and Privilege Abuse) e ASI07 (Insecure Inter-Agent Communication).

SHAVIT, Y.; AGARWAL, S.; BRUNDAGE, M.; ADLER, S.; O'KEEFE, C.; CAMPBELL, R.; LEE, T.; MISHKIN, P.; ELOUNDOU, T.; HICKEY, A.; SLAMA, K.; AHMAD, L.; McMILLAN, P.; BEUTEL, A.; PASSOS, A.; ROBINSON, D. G. **Practices for governing agentic AI systems**. San Francisco: OpenAI, 2023. Disponível em: https://cdn.openai.com/papers/practices-for-governing-agentic-ai-systems.pdf. Ver seções 4.2 (Constraining the action-space and requiring approval) e 4.7 (Interruptibility and maintaining control).

UNIÃO EUROPEIA. Regulamento (UE) 2024/1689 do Parlamento Europeu e do Conselho, de 13 de junho de 2024, que cria regras harmonizadas em matéria de inteligência artificial (Regulamento da Inteligência Artificial). **Jornal Oficial da União Europeia**, L, 12 jul. 2024. Disponível em: https://eur-lex.europa.eu/eli/reg/2024/1689/oj. Art. 14 (Supervisão humana).

**Leitura complementar** (textos de fornecedores, sem uso como fonte de definição ou de números de mercado)

JANICKI, M. **Building governed agents**: a framework for cost, control, and compliance. LangChain, 20 jul. 2026. Disponível em: https://www.langchain.com/blog/building-governed-agents-a-framework-for-cost-control-and-compliance.

PALO ALTO NETWORKS. **A complete guide to agentic AI governance**. Cyberpedia, [2026]. Disponível em: https://www.paloaltonetworks.com/cyberpedia/what-is-agentic-ai-governance.

**Material da disciplina.** [Confiabilidade](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — orçamento, taxonomia de término e confirmação humana, da aula de arquitetura de agentes. [O preço do protocolo](../../aula-10-mcp/notas-de-aula/03-o-preco-do-protocolo.md) — a superfície criada por servidores MCP, da aula de MCP. [A conta, e o que quebra](../../aula-14-mlops-observabilidade/notas-de-aula/03-a-conta-e-o-que-quebra.md) — custo por execução e alavancas, da aula de observabilidade. [O mapa e a injeção](../../aula-15-seguranca/notas-de-aula/01-o-mapa-e-a-injecao.md) e [Defesa por arquitetura](../../aula-15-seguranca/notas-de-aula/02-defesa-por-arquitetura.md) — riscos agênticos e controles, da aula de segurança. [Do contrato ao protocolo](../../aula-16-sistemas-multiagente/notas-de-aula/02-do-contrato-ao-protocolo.md) — A2A e *Agent Card*, da aula de sistemas multiagente.
