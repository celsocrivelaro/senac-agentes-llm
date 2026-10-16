# IA Aplicada com LLMs — Aula 12: Governança de IA agêntica — Legislação

---

## Objetivos de aprendizagem

Ao final desta nota, espera-se que o aluno seja capaz de:

- **Descrever** como a regulação de IA se compõe no Brasil — uma lei geral de dados, leis setoriais e um projeto de lei em tramitação — e **situar** o AI Act como referência estrangeira de alcance condicionado.
- **Identificar** as obrigações da LGPD que alcançam o agente — base legal, minimização no prompt, no índice, na memória e no *log*, direitos do titular, revisão de decisão automatizada (art. 20) — e as decorrentes do uso de provedor externo.
- **Determinar** se um sistema agêntico está sujeito ao ECA Digital pelo critério de acesso provável por crianças e adolescentes e, quando estiver, **derivar** dos deveres da lei os controles de projeto: aferição de idade, padrão mais protetivo, vedação de perfilamento comercial, supervisão parental e registro.
- **Situar** o sistema no AI Act e no PL 2338/2023, com as datas e a situação de tramitação verificadas.
- **Associar** cada obrigação legal ao controle técnico que a atende, nas notas sobre governar a ação e sobre provar a decisão.

---

## Desenvolvimento teórico

A [nota sobre governar a ação](01-governar-a-acao.md) tratou dos limites do agente e de quem os aplica; a [nota sobre provar a decisão](02-provar-a-decisao.md), do registro, da documentação e da resposta a uma contestação. As duas usam a lei como critério de projeto em vários pontos; esta nota reúne o texto legal, por regime — LGPD, ECA Digital, AI Act e PL 2338/2023 —, e fecha com a correspondência entre cada obrigação e o controle técnico que a atende. O exemplo continua sendo o auditor de despesas do curso e o parecer `P-1001`, contestado pelo funcionário `F-088`.

Toda afirmação sobre lei nesta nota é datada: a situação descrita é a verificada em out. 2026. Os trechos marcados "(conferir)" não foram confirmados no texto oficial e precisam sê-lo antes da aula.

### 1. Como ler a regulação de IA no Brasil

Em out. 2026, não há no Brasil lei específica de inteligência artificial. A regulação que alcança um sistema agêntico é a soma de três camadas:

- **lei geral de dados** — a LGPD (Lei nº 13.709/2018) alcança todo tratamento de dado pessoal, qualquer que seja a técnica; é por ela que a decisão automatizada já tem regra própria (art. 20);
- **leis setoriais** — leis que regulam um domínio ou um público e alcançam o agente quando ele opera nesse domínio. O ECA Digital (Lei nº 15.211/2025) é a de efeito mais direto sobre o desenho de produtos digitais, porque impõe deveres que valem "desde a concepção" e "por padrão" a serviços que crianças e adolescentes provavelmente acessam; outras, como a de defesa do consumidor, alcançam o agente conforme o domínio em que opera;
- **projeto em tramitação** — o PL 2338/2023, que proporia uma lei geral de IA por risco; enquanto não aprovado, não cria obrigação.

O AI Act, regulamento da União Europeia, entra como quarta referência: obriga quem implanta ou fornece sistemas cujo resultado é usado na União, e, fora desse caso, serve de modelo de desenho e de antecipação do que o PL 2338/2023 propõe.

```text
  camada                  instrumento          alcança o agente quando
  ----------------------  -------------------  ----------------------------
  lei geral de dados      LGPD                 trata dado pessoal (sempre,
                                               no auditor de despesas)
  lei setorial            ECA Digital          é direcionado a crianças e
                                               adolescentes ou de acesso
                                               provável por eles
  projeto em tramitação   PL 2338/2023         nunca, enquanto não é lei
  referência estrangeira  AI Act (UE)          o resultado é usado na UE

  autoridade comum às duas leis vigentes: ANPD
```

A **ANPD** é o ponto comum. Criada pela LGPD como autoridade de proteção de dados, foi designada pelo Decreto nº 12.622/2025 como a autoridade administrativa autônoma prevista no ECA Digital, e a Lei nº 15.352/2026 a transformou em agência reguladora, com o nome de Agência Nacional de Proteção de Dados (ANPD, [2026a]; BRASIL, 2026a). Para o projeto, a consequência prática é que a mesma autoridade fiscaliza o tratamento de dado pessoal pelo agente e, quando o agente é de acesso provável por menores, os deveres de proteção de crianças e adolescentes.

### 2. LGPD no agente

A Lei nº 13.709/2018, a LGPD, alcança o agente sempre que ele trata dado pessoal — e o auditor de despesas trata dado pessoal em todas as execuções: identificador, despesas e histórico de um funcionário identificável.

#### 2.1 Base legal

Todo tratamento de dado pessoal precisa de uma das hipóteses do art. 7º. Para o auditor de despesas, as candidatas são o cumprimento de obrigação legal ou regulatória (inciso II), a execução de contrato do qual o titular é parte (inciso V) e o legítimo interesse do controlador (inciso IX). O consentimento é inadequado na relação de emprego, em que a assimetria entre as partes compromete a liberdade de recusar. A escolha da base legal é decisão jurídica da organização, registrada no *model card* ([nota sobre provar a decisão](02-provar-a-decisao.md), §4); o que cabe ao projeto é que cada finalidade de tratamento tenha uma base declarada, e que um uso novo do mesmo dado — treinar um classificador com os pareceres, por exemplo — seja tratado como finalidade nova.

#### 2.2 Minimização nos quatro lugares onde o dado cai

O princípio da necessidade (art. 6º, III) limita o tratamento ao mínimo necessário para a finalidade e se aplica a cada estrutura em que o dado pessoal entra, e não ao sistema em abstrato. No agente, são quatro:

| Lugar | Como o dado entra | Medida de minimização |
|---|---|---|
| **Prompt** | o contexto montado a cada chamada, enviado ao provedor | enviar o pseudônimo, e não o nome; enviar só os campos que a decisão usa |
| **Índice** | documentos indexados que contêm dado pessoal | indexar o regulamento, e não pareceres nominais; quando indexar, gravar o titular como metadado |
| **Memória** | episódios e fatos persistidos entre execuções | memória por titular, com metadado que permita a remoção (aula de memória) |
| ***Log*** | *trace*, *checkpoint*, registro de decisão | referência em lugar de conteúdo ([nota sobre provar a decisão](02-provar-a-decisao.md), §2); acesso restrito; prazo de guarda |

A lição da aula de memória vale para os quatro: a lista de lugares não se deriva da arquitetura pretendida, e sim de um **inventário de onde o dado efetivamente cai**, mantido a cada estrutura acrescentada ao sistema ([nota sobre esquecer e proteger](../../aula-08-memoria/notas-de-aula/04-esquecer-e-proteger.md), §3.1).

#### 2.3 Direitos do titular e eliminação

O art. 18 garante ao titular, entre outros, o acesso aos dados, a correção, e a anonimização, o bloqueio ou a eliminação de dados desnecessários, excessivos ou tratados em desconformidade com a lei (inciso IV). A eliminação tem um limite que o projeto precisa conhecer: o art. 16 autoriza a conservação após o término do tratamento para, entre outras finalidades, o cumprimento de obrigação legal ou regulatória pelo controlador (inciso I).

Daí o conflito, característico do agente, entre eliminar e provar. Um pedido de eliminação do funcionário `F-088` alcança a memória episódica, os fatos e o *checkpoint* — e não alcança, necessariamente, o registro do parecer `P-1001` enquanto ele for necessário para responder à contestação ou a uma obrigação de guarda. A resposta de projeto é separar as estruturas por finalidade: a memória, que serve à operação, é eliminável a pedido; o registro de decisão, que serve à prova, é conservado pelo prazo justificado, com acesso restrito, e eliminado ao fim dele. A remoção é verificada por busca posterior, como na aula de memória, e a resposta ao titular declara o que foi eliminado e o que foi conservado, com a razão.

#### 2.4 Art. 20: revisão de decisão automatizada

O art. 20 da LGPD, na redação dada pela Lei nº 13.853/2019, dispõe:

> "Art. 20. O titular dos dados tem direito a solicitar a revisão de decisões tomadas unicamente com base em tratamento automatizado de dados pessoais que afetem seus interesses, incluídas as decisões destinadas a definir o seu perfil pessoal, profissional, de consumo e de crédito ou os aspectos de sua personalidade.
>
> § 1º O controlador deverá fornecer, sempre que solicitadas, informações claras e adequadas a respeito dos critérios e dos procedimentos utilizados para a decisão automatizada, observados os segredos comercial e industrial." (BRASIL, 2018)

O § 2º permite à autoridade nacional realizar auditoria para verificar aspectos discriminatórios quando o controlador deixar de fornecer as informações do § 1º alegando segredo comercial e industrial. O texto vigente não exige que a revisão seja feita por pessoa natural: o § 3º que trazia essa exigência, incluído pelo Congresso na Lei nº 13.853/2019, foi vetado, e o veto foi mantido.

Três elementos do caput decidem se o artigo alcança o parecer recusado:

- **"unicamente"**: se nenhum humano participou da decisão, ela é unicamente automatizada. As contribuições consolidadas pela ANPD registram a posição de que a participação humana precisa ser **significativa** — um revisor que apenas referenda o resultado do sistema não afasta o caráter automatizado (ANPD, 2025). O campo $H$ do registro de decisão ([nota sobre provar a decisão](02-provar-a-decisao.md), §1) é o que permite responder a esse ponto;
- **"que afetem seus interesses"**: a recusa de reembolso afeta o interesse patrimonial do funcionário;
- **"perfil [...] profissional"**: um agente que acumula pareceres por funcionário e passa a usá-los como histórico em decisões futuras aproxima-se da definição de perfil profissional.

O § 1º tem tradução direta em requisito de projeto: o controlador precisa ser capaz de informar os critérios e os procedimentos da decisão. Para o auditor de despesas, os critérios são o regulamento e a política versionada; o procedimento é a sequência registrada em $\Phi$. Um agente cujos critérios só existem implícitos no prompt e no modelo não consegue atender ao § 1º.

#### 2.5 O estado da regulamentação

A ANPD incluiu a inteligência artificial, com atenção especial ao art. 20, na Agenda Regulatória 2025–2026. Realizou tomada de subsídios sobre inteligência artificial e revisão de decisões automatizadas entre 6 de novembro de 2024 e 24 de janeiro de 2025, e consolidou as contribuições na Nota Técnica nº 12/2025/CON1/CGN/ANPD, assinada em 3 de abril de 2025, que serve de insumo ao projeto regulatório e não representa posicionamento final da autoridade (ANPD, 2025). O balanço do primeiro semestre de 2026 da agenda registra que a minuta da solução regulatória estava em elaboração na área técnica (ANPD, 2026b). Em out. 2026, não há guia nem regulamento publicado sobre o art. 20; o alcance de "unicamente", de "revisão" e do segredo comercial continua matéria de interpretação.

#### 2.6 O provedor externo

Cada chamada a uma API de modelo envia ao provedor o prompt montado — e, com ele, o dado pessoal que estiver no contexto. A escolha de provedor, tratada na aula de escolha e configuração de modelos como decisão de custo, latência e privacidade ([nota sobre escolha de modelos](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md), §8), é também **decisão de conformidade**, por três razões:

- **transferência internacional**: quando o provedor processa os dados fora do Brasil, há transferência internacional, permitida apenas nas hipóteses do art. 33 da LGPD — entre elas, país com grau de proteção adequado (inciso I) ou garantias como cláusulas contratuais. A regulamentação da ANPD sobre transferência internacional (Resolução CD/ANPD nº 19/2024, conferir) define as cláusulas-padrão;
- **retenção**: o provedor pode guardar prompts e respostas por um período, para monitoramento de abuso ou por padrão; o prazo e as condições estão nos termos de serviço e no contrato vigentes, e não em documentação de marketing;
- **uso para treino**: os termos definem se o conteúdo enviado pode ser usado para treinar modelos do provedor, e se a exclusão depende de configuração da conta ou de plano contratado.

O provedor atua, em relação aos dados enviados, como operador da organização, e a organização continua controladora: a responsabilidade pelo que foi enviado não passa ao provedor. No registro de decisão, a consequência é que o campo de versões inclui o provedor e a região, porque a pergunta "para onde foi o dado pessoal" se responde por ele. O curso usa a API da Mistral; os termos de retenção e de uso para treino aplicáveis a uma implantação real devem ser conferidos no contrato vigente.

### 3. ECA Digital

#### 3.1 O que é, e desde quando vale

O **Estatuto Digital da Criança e do Adolescente** (ECA Digital) é a Lei nº 15.211, de 17 de setembro de 2025, que dispõe sobre a proteção de crianças e adolescentes em ambientes digitais (BRASIL, 2025a). É lei setorial: não regula a IA como técnica, e sim produtos e serviços digitais em função de quem os usa; um agente é alcançado por ela da mesma forma que uma rede social ou um jogo, se cumprir o critério de aplicação da §3.2.

A vigência teve três atos. O texto aprovado previa entrada em vigor um ano após a publicação, e esse dispositivo foi vetado na sanção. No mesmo dia, a Medida Provisória nº 1.319, de 17 de setembro de 2025, publicada em 18 de setembro, fixou a vigência em seis meses (BRASIL, 2025b). A MP não chegou a ser votada: foi revogada na íntegra pela Lei nº 15.352, publicada em 25 de fevereiro de 2026 — a mesma que transformou a ANPD em agência —, que fixou o início da vigência do ECA Digital em 17 de março de 2026 (CONGRESSO NACIONAL, 2026; BRASIL, 2026a; dispositivo da Lei nº 15.352/2026 que altera a Lei nº 15.211/2025: conferir). A lei está **em vigor desde 17 de março de 2026**. No dia seguinte, o Decreto nº 12.880, de 18 de março de 2026, regulamentou-a e instituiu a política nacional de proteção dos direitos de crianças e adolescentes no ambiente digital (BRASIL, 2026b).

#### 3.2 A quem se aplica: o critério de acesso provável

O art. 1º define o alcance:

> "Art. 1º Esta Lei dispõe sobre a proteção de crianças e de adolescentes em ambientes digitais e aplica-se a todo produto ou serviço de tecnologia da informação direcionado a crianças e a adolescentes no País ou de acesso provável por eles, independentemente de sua localização, desenvolvimento, fabricação, oferta, comercialização e operação." (BRASIL, 2025a)

**Produto ou serviço de tecnologia da informação** é, pelo art. 2º, I, o "produto ou serviço fornecido a distância, por meio eletrônico e provido em virtude de requisição individual", o que inclui aplicações de internet, *software*, sistemas operacionais, lojas de aplicações e jogos conectados. Um assistente ou agente oferecido como aplicação está dentro da definição. O ponto decisivo é a segunda hipótese do art. 1º, que dispensa a intenção do fornecedor. O parágrafo único a define:

> "Parágrafo único. Para os fins desta Lei, considera-se acesso provável por crianças e adolescentes as seguintes situações:
> I - suficiente probabilidade de uso e atratividade do produto ou serviço de tecnologia da informação por crianças e adolescentes;
> II - considerável facilidade ao acesso e utilização do produto ou serviço de tecnologia da informação por crianças e adolescentes; e
> III - significativo grau de risco à privacidade, à segurança ou ao desenvolvimento biopsicossocial de crianças e de adolescentes, especialmente no caso de produtos ou serviços que tenham por finalidade permitir a interação social e o compartilhamento de informações em larga escala entre usuários em ambiente digital." (BRASIL, 2025a)

O critério é, portanto, uma avaliação de fato, e não uma declaração de público-alvo: um assistente de uso geral, aberto ao público, com cadastro simples, que conversa sobre qualquer assunto e lembra o que o usuário disse, atende aos três incisos ainda que os termos de uso exijam maioridade. A ANPD tratou a categoria como alcançada: em ago. 2026, iniciou ações de monitoramento das principais plataformas digitais, lojas de aplicativos e ferramentas de inteligência artificial generativa para verificar o cumprimento da lei (ANPD, 2026c).

#### 3.3 As obrigações

O núcleo da lei é um conjunto de deveres de desenho. Os principais, com o dispositivo correspondente:

| Obrigação | Conteúdo | Dispositivo |
|---|---|---|
| **Dever de cuidado e segurança desde a concepção** | deveres de prevenção, proteção, informação e segurança; medidas razoáveis, desde a concepção, para prevenir e mitigar a exposição a exploração e abuso sexual, violência e assédio, instigação à automutilação e ao suicídio, jogos de azar e apostas, publicidade predatória e conteúdo pornográfico | arts. 5º e 6º |
| **Configuração mais protetiva por padrão** | o padrão é o modelo mais protetivo disponível em privacidade e proteção de dados; a configuração menos protetiva é escolha informada | art. 7º |
| **Verificação de idade** | para conteúdo, produto ou serviço impróprio a menores de 18 anos, mecanismos confiáveis de verificação a cada acesso, vedada a autodeclaração; lojas de aplicações e sistemas operacionais aferem a idade ou a faixa etária | arts. 9º e 12 |
| **Finalidade única do dado de idade** | o dado coletado para verificar a idade só serve a isso | art. 13 |
| **Supervisão parental** | ferramentas acessíveis, aviso de que estão ativas, limite de tempo; padrão no nível mais alto de proteção, incluindo controle e desativação de sistemas de recomendação personalizados e revisão regular das ferramentas de IA; poderes do responsável, como restringir compras | arts. 17 e 18 |
| **Caixas de recompensa** | vedadas em jogos direcionados a crianças e adolescentes ou de acesso provável por eles | art. 20 |
| **Vedação de perfilamento comercial** | vedado o perfilamento para direcionar publicidade comercial a crianças e adolescentes, e o uso de análise emocional e de realidade aumentada, estendida ou virtual para esse fim; nas redes sociais, vedada a construção de perfis comportamentais para esse fim | arts. 22 e 26 |
| **Conta vinculada** | contas de usuários de até 16 anos vinculadas à conta de um responsável legal | art. 24 |
| **Remoção e comunicação de conteúdo** | remoção e comunicação às autoridades de conteúdo de aparente exploração, abuso sexual, sequestro e aliciamento; remoção de conteúdo violador mediante notificação, sem ordem judicial, com direito de recurso de quem publicou | arts. 27 a 30 |
| **Relatórios de transparência** | semestrais, em português, para provedores com mais de um milhão de usuários na faixa etária, sobre denúncias, moderação e medidas adotadas | art. 31 |
| **Representante legal no País** | fornecedor estrangeiro mantém representante com poderes para receber citações e responder a autoridades | art. 40 |

A proibição de perfilamento tem redação que alcança diretamente sistemas que inferem preferências:

> "Art. 22. Para além das demais disposições desta Lei, é vedada a utilização de técnicas de perfilamento para direcionamento de publicidade comercial a crianças e a adolescentes, bem como o emprego de análise emocional, de realidade aumentada, de realidade estendida e de realidade virtual para esse fim." (BRASIL, 2025a)

**Perfilamento**, pelo art. 2º, V, é "qualquer forma de tratamento de dados pessoais, automatizada ou não, para avaliar certos aspectos de uma pessoa natural", com o objetivo de classificá-la e de inferir comportamentos, interesses e outras características. E o art. 13 fecha o uso do dado de idade:

> "Art. 13. Os dados coletados para a verificação de idade de crianças e de adolescentes poderão ser utilizados unicamente para essa finalidade, vedado seu tratamento para qualquer outro propósito." (BRASIL, 2025a)

As obrigações são moduladas: o art. 39 prevê que sejam aplicadas conforme as funcionalidades do serviço, o número de usuários e o porte do fornecedor (conferir a redação). O art. 17, § 4º, VIII, é o único dispositivo que nomeia a técnica: entre as configurações-padrão da supervisão parental, exige a "revisão regular das ferramentas de inteligência artificial" com base em critérios técnicos que assegurem a segurança e a adequação ao uso por crianças e adolescentes, garantida a possibilidade de desabilitar funcionalidades não essenciais (BRASIL, 2025a).

#### 3.4 Autoridade e sanções

A lei atribui a fiscalização a uma "autoridade administrativa autônoma" (art. 2º, X, e art. 34), sem nomeá-la. O Decreto nº 12.622, de 17 de setembro de 2025, designou a ANPD para essa função, e o Decreto nº 12.880/2026 reafirmou que a regulamentação e a fiscalização da lei cabem a ela, sem prejuízo da competência de outros órgãos (ANPD, [2026a]; BRASIL, 2026b). Crimes contra crianças e adolescentes não são objeto da ANPD: as denúncias de conteúdo criminoso recebidas pelos fornecedores seguem para o Centro Nacional de Proteção à Criança e ao Adolescente, operado pela Polícia Federal (ANPD; MJSP; SECOM, 2026).

As sanções estão no art. 35: **advertência**, com prazo de até 30 dias para correção; **multa simples** de até 10% do faturamento do grupo econômico no Brasil no último exercício ou, sem faturamento, de R\$ 10 a R\$ 1.000 por usuário cadastrado, limitada a R\$ 50 milhões por infração; **suspensão temporária** das atividades; e **proibição** do exercício das atividades. A autoridade aplica a advertência e a multa; a suspensão e a proibição dependem de decisão judicial (art. 35, § 5º, conferir) (BRASIL, 2025a).

A ANPD publicou orientações preliminares sobre aferição de idade em mar. 2026 e um cronograma de implantação em três etapas, com ações de fiscalização previstas a partir de jan. 2027; em out. 2026, o guia definitivo e o regulamento de fiscalização e de processo sancionador ainda não estavam publicados — este em consulta pública até 26 de outubro de 2026 (ANPD, [2026a]; ANPD, 2026c; conferir as datas na página da ANPD).

#### 3.5 Impactos para um sistema agêntico

Seja um assistente ou agente de uso geral, acessível a adolescentes pelo critério da §3.2, com memória entre sessões e ferramentas de escrita — compra, envio de mensagem, agendamento. Os deveres da lei se traduzem em controles de projeto, e quase todos têm lugar em estruturas já construídas no curso.

**Aferição de idade antes de agir.** A idade é um atributo da identidade de quem aciona o agente e entra na decisão do ponto de aplicação de política ([nota sobre governar a ação](01-governar-a-acao.md), §6.1) como qualquer outro atributo: uma chamada de ferramenta que adquire, contrata ou exibe algo impróprio a menores de 18 anos é negada quando a faixa etária do usuário não está aferida como adulta. O art. 9º, § 1º, veda a autodeclaração, de modo que o sinal de idade não pode vir do que o usuário diz na conversa — texto no contexto é entrada não confiável, e um "tenho 18 anos" no prompt é instrução como qualquer outra. O sinal vem de um mecanismo de aferição externo ao modelo, e, pela §6.5 da nota sobre governar a ação, a ausência do sinal é tratada como *fail-closed*: sem idade aferida, a ação imprópria a menores não é executada. O FAQ da ANPD dá exemplos concretos dessa classe de ação — a compra de bebida alcoólica, de cigarro ou de aposta por aplicativo de entrega ou *marketplace* exige conferência da idade antes da compra (ANPD; MJSP; SECOM, 2026).

**Padrão mais protetivo.** Pelo art. 7º, a configuração de partida é a mais protetiva, e a menos protetiva é escolha informada. No agente, isso decide os valores-padrão de parâmetros que, num produto para adultos, seriam escolhas de produto: memória persistente desligada ou restrita, prazo de retenção curto, personalização desligada, ferramentas de escrita sujeitas a confirmação. Um limiar de escalonamento ([nota sobre governar a ação](01-governar-a-acao.md), §7.2) calibrado para adultos não serve de padrão para uma conta de adolescente.

**Proibição de perfilamento comercial.** A memória de longo prazo de um agente — fatos e preferências extraídos das conversas — é, na definição do art. 2º, V, um perfil: tratamento de dados pessoais para inferir interesses e características. O art. 22 não proíbe a memória; proíbe usá-la para direcionar publicidade comercial a menores, e proíbe a análise emocional para esse fim. A consequência de projeto é separar a memória por finalidade: a que serve à tarefa do usuário pode existir, com padrão protetivo; nenhuma leitura dela alimenta recomendação de produto patrocinado, seleção de anúncio ou tom persuasivo calibrado pelo estado emocional inferido, quando o usuário é ou pode ser menor. Um agente de compras que sugere produtos não é proibido; um agente que escolhe o que sugerir a um adolescente porque um anunciante pagou pela sugestão e a memória indicou o interesse, é.

**Memória e dados de menores.** O esquecimento construído na aula de memória — remoção sob solicitação, verificada por busca posterior, alcançando a estrutura que escapa ([nota sobre esquecer e proteger](../../aula-08-memoria/notas-de-aula/04-esquecer-e-proteger.md), §3) — é o mecanismo com que o agente atende, para o titular adolescente, à eliminação da LGPD (§2.3), cujo art. 14 já exige que o tratamento de dados de crianças e adolescentes se faça em seu melhor interesse. O ECA Digital acrescenta duas restrições. O dado de aferição de idade (o documento, a imagem do rosto, a estimativa) segue a finalidade única do art. 13: não entra no prompt, na memória nem no *log* — o registro guarda o resultado da aferição (faixa etária, método, instante), e não o insumo. E o padrão protetivo do art. 7º desloca a questão: em vez de remover a pedido o que foi lembrado, o agente lembra menos por padrão.

**Supervisão parental como supervisão humana.** O art. 18 dá ao responsável poderes como restringir compras, e o art. 24 exige que contas de até 16 anos estejam vinculadas à conta de um responsável legal — o FAQ da ANPD apresenta a vinculação no contexto das redes sociais, e o alcance do art. 24 sobre outros serviços deve ser conferido. Para o agente, o responsável é um **aprovador** no sentido da [nota sobre governar a ação](01-governar-a-acao.md) (§7.1): a compra iniciada pelo agente numa conta de adolescente é uma ação *human-in-the-loop* cujo destino de pausa é o responsável, e não o próprio usuário. Os mesmos cuidados valem: o responsável precisa ver o que aprova, com contexto suficiente, e o vencimento do prazo de aprovação resulta em não executar. A diferença em relação à supervisão corporativa é que o aprovador é externo à organização, de modo que o canal de aprovação é parte do produto.

**Registro como evidência de conformidade.** A fiscalização pela ANPD pergunta o que o sistema fez para cumprir a lei, e a resposta vem do registro, com a mesma lógica da [nota sobre provar a decisão](02-provar-a-decisao.md): cada ação negada por falta de idade aferida, cada pausa para o responsável e cada configuração alterada a partir do padrão é um evento registrado, com a versão da política que o produziu. O registro guarda a decisão, e não o dado de aferição (art. 13). Para provedores com mais de um milhão de usuários na faixa etária, os relatórios semestrais do art. 31 são agregados desses mesmos eventos.

**O auditor de despesas.** O auditor de despesas do curso, provavelmente, não está sujeito ao ECA Digital. Não é direcionado a crianças e adolescentes, e os três incisos do art. 1º, parágrafo único, apontam para fora: não há atratividade para esse público (inciso I), o acesso depende de credencial corporativa emitida pela organização a funcionários (inciso II), e o sistema não permite interação social nem compartilhamento entre usuários (inciso III). O critério é o acesso provável **ao serviço**, e não a idade de um usuário em particular: se a organização emprega aprendizes adolescentes e algum deles submete despesas, os dados desse funcionário seguem a LGPD, inclusive o art. 14, mas a presença de um usuário menor numa ferramenta interna não torna, por si, o serviço de acesso provável no sentido da lei (conferir a interpretação da ANPD). A conclusão muda com o desenho: o mesmo agente oferecido como aplicativo aberto de controle de gastos pessoais, com cadastro livre, passaria a atender ao inciso II e precisaria da análise desta seção.

### 4. AI Act

#### 4.1 Classificação por risco e alcance

O Regulamento (UE) 2024/1689, o AI Act, classifica sistemas de IA por risco: práticas **proibidas** (art. 5º); sistemas de **alto risco**, definidos pelo art. 6º com remissão aos Anexos I (componentes de segurança de produtos regulados) e III (áreas listadas); sistemas sujeitos a **obrigações de transparência** (art. 50); e os demais. O ponto 4 do Anexo III inclui, na área de emprego e gestão de trabalhadores, sistemas destinados a tomar decisões que afetam as condições das relações de trabalho e a monitorar e avaliar o desempenho e o comportamento de pessoas nessas relações (UNIÃO EUROPEIA, 2024).

Um auditor de despesas que avalia a conduta de funcionários está próximo dessa descrição. O art. 6º, §3º, prevê que um sistema do Anexo III não seja considerado de alto risco quando não representar risco significativo — por exemplo, quando executa tarefa processual restrita ou tarefa preparatória de uma avaliação humana —, mas a exceção não se aplica quando o sistema faz perfilamento de pessoas. A classificação do auditor depende, portanto, do desenho: um agente que prepara o parecer para decisão humana e não constrói perfil tem argumento para a exceção; um agente que decide sozinho e usa o histórico do funcionário, não. A classificação é registrada no *model card*, com a justificativa.

O regulamento se aplica a quem implanta sistemas de IA na União e a fornecedores e responsáveis pela implantação estabelecidos fora dela quando o resultado do sistema é usado na União (art. 2º). Uma organização que opera só no Brasil não está sujeita a ele; o regulamento serve, nesse caso, como referência de desenho e como antecipação do modelo adotado pelo PL 2338/2023 (§5).

#### 4.2 O registro como obrigação: arts. 12 e 26

No AI Act, o registro deixa de ser boa prática e passa a ser requisito para sistemas de alto risco. O art. 12, §1º, determina que esses sistemas permitam tecnicamente o registro automático de eventos (*logs*) ao longo de toda a vida útil do sistema; o §2º associa esse registro à rastreabilidade do funcionamento, em especial à identificação de situações de risco, ao monitoramento pós-comercialização e ao monitoramento da operação por quem implanta (UNIÃO EUROPEIA, 2024).

A obrigação se divide entre os dois papéis que o regulamento distingue. O **fornecedor** (*provider*) constrói a capacidade de registro; o **responsável pela implantação** (*deployer*) a usa e guarda o resultado. O art. 26, §6º, obriga quem implanta a conservar os *logs* gerados automaticamente, na medida em que estejam sob o seu controle, por período adequado à finalidade e de **pelo menos seis meses**, salvo disposição diversa do direito da União ou nacional (UNIÃO EUROPEIA, 2024). O mesmo artigo impõe a quem implanta atribuir a supervisão humana a pessoas com competência e autoridade (§2º), monitorar o funcionamento (§5º), informar os trabalhadores e seus representantes antes de usar o sistema no local de trabalho (§7º) e informar as pessoas sujeitas a decisões tomadas ou auxiliadas por sistemas do Anexo III (§11).

Para o agente do curso, a organização que o implanta acumula os dois papéis: quem constrói o agente sobre um modelo de terceiro é fornecedor do sistema de IA, e quem o opera é responsável pela implantação. O modelo de propósito geral, que vem do provedor, tem regime próprio no regulamento e não transfere ao provedor a obrigação de registrar as decisões do agente.

#### 4.3 Supervisão humana: art. 14

O AI Act incorpora a supervisão humana e a interrupção como requisitos de projeto para sistemas de alto risco. O art. 14 exige que esses sistemas sejam projetados para ser efetivamente supervisionados por pessoas durante o uso, e o n. 4 enumera as capacidades que a supervisão precisa ter: compreender as capacidades e os limites do sistema e monitorar seu funcionamento; manter-se consciente da tendência de confiar automaticamente na saída do sistema — o **viés de automação**; interpretar corretamente a saída; decidir não usar o sistema, ou desconsiderar, anular ou reverter sua saída; e **intervir no funcionamento ou interromper o sistema** por meio de um botão de parada ou procedimento semelhante que o leve a um estado seguro (UNIÃO EUROPEIA, 2024, art. 14, n. 4, a–e). Os controles correspondentes — limiar de escalonamento, aprovação que não degenera em carimbo, procedimento de interrupção testado — estão na [nota sobre governar a ação](01-governar-a-acao.md) (§7) e valem como boa prática independentemente da classificação.

#### 4.4 Deveres de quem implanta e direito à explicação

Para quem implanta um sistema de alto risco, os deveres principais estão no art. 26 (§4.2): usar conforme as instruções do fornecedor, atribuir supervisão humana competente, monitorar, guardar os *logs* por pelo menos seis meses, informar trabalhadores e pessoas afetadas. O art. 86 acrescenta o **direito à explicação**: a pessoa afetada por decisão tomada com base na saída de sistema de alto risco do Anexo III, que produza efeitos jurídicos ou a afete de modo significativo, pode obter de quem implanta explicações claras e significativas sobre o papel do sistema no processo decisório e sobre os principais elementos da decisão (UNIÃO EUROPEIA, 2024). A obrigação tem a mesma tradução em projeto que o § 1º do art. 20 da LGPD (§2.4): só se explica o papel do sistema se o papel estiver registrado.

#### 4.5 Datas

As proibições se aplicam desde 2 fev. 2025, e as obrigações de modelos de propósito geral, desde 2 ago. 2025. As obrigações de alto risco, previstas originalmente para 2 ago. 2026, foram adiadas pelo Regulamento (UE) 2026/1744, o *Digital Omnibus* sobre IA: para os sistemas do Anexo III, 2 dez. 2027; para os do Anexo I, 2 ago. 2028 (conferir no EUR-Lex). As obrigações de transparência do art. 50 não foram adiadas pelo *Digital Omnibus*, segundo as análises consultadas em out. 2026 (conferir no EUR-Lex).

### 5. Brasil: o PL 2338/2023

O Projeto de Lei nº 2338/2023 dispõe sobre o desenvolvimento e o uso de sistemas de inteligência artificial no Brasil. Na versão aprovada pelo Senado Federal em dez. 2024, adota classificação por risco, com sistemas de risco excessivo, vedados, e de alto risco, sujeitos a obrigações de governança e a avaliação de impacto algorítmico; prevê direitos das pessoas afetadas, como informação, explicação e contestação de decisões; e organiza a regulação num sistema nacional coordenado pela ANPD (conferir o texto aprovado).

Na Câmara dos Deputados, o projeto foi recebido para revisão em 17 de março de 2025; a comissão especial que o examina foi constituída em 29 de abril de 2025, com relatoria designada em 20 de maio de 2025. Em out. 2026, a ficha de tramitação registra a situação "Aguardando Parecer do(a) Relator(a) na Comissão Especial". **O PL 2338/2023 não é lei**: não cria obrigação, e o texto ainda pode mudar na Câmara e, se alterado, voltar ao Senado. A obrigação vigente sobre decisão automatizada no Brasil é a do art. 20 da LGPD.

### 6. Das obrigações aos controles

Cada obrigação desta nota só é cumprida por um controle técnico ou organizacional que existe no sistema. A tabela liga as duas pontas; a última coluna indica onde o controle está desenvolvido.

| Obrigação | Controle técnico | Onde está |
|---|---|---|
| LGPD, art. 6º, III — necessidade | referência em vez de conteúdo no registro; pseudônimo no prompt; inventário de onde o dado cai | [provar a decisão](02-provar-a-decisao.md), §2; §2.2 desta nota |
| LGPD, art. 7º — base legal por finalidade | campo "Dados" do *model card*, com base legal e finalidade | [provar a decisão](02-provar-a-decisao.md), §4 |
| LGPD, arts. 16 e 18 — eliminação e conservação | memória eliminável a pedido, com remoção verificada; registro de decisão conservado pelo prazo de prova | [provar a decisão](02-provar-a-decisao.md), §§2 e 7; [esquecer e proteger](../../aula-08-memoria/notas-de-aula/04-esquecer-e-proteger.md), §3 |
| LGPD, art. 20, caput — revisão de decisão unicamente automatizada | campo $H$ que distingue "não houve" de "não registrada"; recusa enviada a humano; revisão registrada como novo $H$ | [provar a decisão](02-provar-a-decisao.md), §§1 e 7; [governar a ação](01-governar-a-acao.md), §8 |
| LGPD, art. 20, § 1º — critérios e procedimentos | regulamento e política versionados; sequência de ferramentas $\Phi$ no registro | [provar a decisão](02-provar-a-decisao.md), §§1 e 3 |
| LGPD, art. 33 — transferência internacional | provedor e região no registro; roteamento só entre modelos aprovados; reserva com a mesma política de dados | [provar a decisão](02-provar-a-decisao.md), §1; [governar a ação](01-governar-a-acao.md), §§6.3 e 6.5 |
| ECA Digital, arts. 9º e 12 — verificação de idade | faixa etária aferida como atributo da identidade, avaliada no ponto de aplicação; *fail-closed* sem aferição | [governar a ação](01-governar-a-acao.md), §§6.1 e 6.5; §3.5 desta nota |
| ECA Digital, art. 7º — padrão mais protetivo | valores-padrão de memória, retenção, personalização e confirmação definidos para o menor | [governar a ação](01-governar-a-acao.md), §7.2; §3.5 desta nota |
| ECA Digital, art. 13 — finalidade única do dado de idade | insumo da aferição fora do prompt, da memória e do *log*; registro só do resultado | [provar a decisão](02-provar-a-decisao.md), §2; §3.5 desta nota |
| ECA Digital, arts. 22 e 26 — perfilamento comercial | memória separada por finalidade; personalização comercial desligada para menor | §3.5 desta nota; [esquecer e proteger](../../aula-08-memoria/notas-de-aula/04-esquecer-e-proteger.md) |
| ECA Digital, arts. 17, 18 e 24 — supervisão parental | limiar de escalonamento cujo destino é o responsável legal; prazo vencido resulta em não executar | [governar a ação](01-governar-a-acao.md), §§7.1 e 7.2 |
| AI Act, art. 12 — registro automático | registro de decisão $R$, em armazenamento só de acréscimo | [provar a decisão](02-provar-a-decisao.md), §1 |
| AI Act, art. 26, §6º — guarda por seis meses ou mais | prazo de guarda fixado pela finalidade de prova | [provar a decisão](02-provar-a-decisao.md), §2 |
| AI Act, art. 14 — supervisão e interrupção | aprovação com contexto; procedimento de interrupção testado | [governar a ação](01-governar-a-acao.md), §§7.1 a 7.3 |
| AI Act, arts. 26, §11, e 86 — informação e explicação | parecer que declara o agente, cita o registro e indica a quem pedir revisão | [provar a decisão](02-provar-a-decisao.md), §6 |
| AI Act, art. 6º, §3º — classificação | classificação e justificativa no *model card* | [provar a decisão](02-provar-a-decisao.md), §4 |
| PL 2338/2023 — avaliação de impacto (proposta) | nenhuma obrigação vigente; a auditoria interna ao longo do ciclo de vida produz os documentos que uma avaliação de impacto pediria | [provar a decisão](02-provar-a-decisao.md), §5 |

A leitura da tabela de cima para baixo mostra que as leis diferem no que exigem, mas convergem nos mecanismos: quase toda obrigação recai sobre o ponto de aplicação de política, sobre o registro de decisão ou sobre o *model card*. Um sistema sem essas três estruturas não atende a nenhuma das leis, e um sistema com elas atende a uma obrigação nova, em geral, ajustando uma regra ou um campo, e não refazendo a arquitetura.

---

## Fontes e leituras

AGÊNCIA NACIONAL DE PROTEÇÃO DE DADOS (ANPD). **Eca Digital**. Brasília, DF, [2026a]. Disponível em: https://www.gov.br/anpd/pt-br/assuntos/eca-digital. Acesso em: 16 out. 2026.

AGÊNCIA NACIONAL DE PROTEÇÃO DE DADOS (ANPD). **Balanço da Agenda Regulatória 2025–2026**: primeiro semestre de 2026. Brasília, DF, 2026b. Disponível em: https://www.gov.br/anpd/pt-br/assuntos/regulacao/agenda-regulatoria-1/agenda-regulatoria-2025-2026/balanco-agenda-regulatoria-2025-2026-01-2026.pdf. Acesso em: 16 out. 2026.

AGÊNCIA NACIONAL DE PROTEÇÃO DE DADOS (ANPD). **ECA Digital completa um ano e é marco na proteção de crianças e adolescentes na internet**. Brasília, DF, 16 set. 2026c. Disponível em: https://www.gov.br/anpd/pt-br/assuntos/noticias/eca-digital-completa-um-ano-e-e-marco-na-protecao-de-criancas-e-adolescentes-na-internet. Acesso em: 16 out. 2026.

AGÊNCIA NACIONAL DE PROTEÇÃO DE DADOS (ANPD); MINISTÉRIO DA JUSTIÇA E SEGURANÇA PÚBLICA (MJSP); SECRETARIA DE COMUNICAÇÃO SOCIAL DA PRESIDÊNCIA DA REPÚBLICA (SECOM). **Perguntas e respostas sobre o ECA Digital**. Brasília, DF, atualizado em 30 jul. 2026. Disponível em: https://www.gov.br/anpd/pt-br/assuntos/eca-digital/perguntas_respostas_eca_digital_30072026.pdf. Acesso em: 16 out. 2026.

AUTORIDADE NACIONAL DE PROTEÇÃO DE DADOS (ANPD). **Nota Técnica nº 12/2025/CON1/CGN/ANPD**: consolidação das contribuições recebidas na Tomada de Subsídios sobre inteligência artificial e revisão de decisões automatizadas. Brasília, DF, 3 abr. 2025. Disponível em: https://www.gov.br/anpd/pt-br/acesso-a-informacao/participacao-social/outras-acoes/documentos/ts-06-2024-nt-12-2025-consolidacao-das-contribuicoes.pdf. Acesso em: 16 out. 2026.

AUTORIDADE NACIONAL DE PROTEÇÃO DE DADOS (ANPD). **Resolução CD/ANPD nº 19, de 23 de agosto de 2024**. Aprova o Regulamento de Transferência Internacional de Dados e o conteúdo das cláusulas-padrão contratuais. Brasília, DF, 2024. (Conferir número e data.)

BRASIL. **Lei nº 13.709, de 14 de agosto de 2018**. Lei Geral de Proteção de Dados Pessoais (LGPD). Brasília, DF: Presidência da República, 2018. Disponível em: https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm. Acesso em: 16 out. 2026.

BRASIL. **Lei nº 13.853, de 8 de julho de 2019**. Altera a Lei nº 13.709, de 14 de agosto de 2018. Brasília, DF: Presidência da República, 2019.

BRASIL. **Lei nº 15.211, de 17 de setembro de 2025**. Dispõe sobre a proteção de crianças e de adolescentes em ambientes digitais (Estatuto Digital da Criança e do Adolescente). Brasília, DF: Presidência da República, 2025a. Disponível em: https://www.planalto.gov.br/ccivil_03/_ato2023-2026/2025/lei/l15211.htm. Acesso em: 16 out. 2026. Texto da publicação original consultado em: https://www2.camara.leg.br/legin/fed/lei/2025/lei-15211-17-setembro-2025-797997-publicacaooriginal-176498-pl.html.

BRASIL. **Medida Provisória nº 1.319, de 17 de setembro de 2025**. Altera a Lei nº 15.211, de 17 de setembro de 2025, para dispor sobre a vigência do Estatuto Digital da Criança e do Adolescente. Brasília, DF: Presidência da República, 2025b. Disponível em: https://www2.camara.leg.br/legin/fed/medpro/2025/medidaprovisoria-1319-17-setembro-2025-797989-norma-pe.html. Acesso em: 16 out. 2026.

BRASIL. **Lei nº 15.352, de 2026**. Transforma a Autoridade Nacional de Proteção de Dados em Agência Nacional de Proteção de Dados. Brasília, DF: Presidência da República, 2026a. Publicada no DOU de 25 fev. 2026. (Conferir data de assinatura e o dispositivo que altera a vigência da Lei nº 15.211/2025.)

BRASIL. **Decreto nº 12.880, de 18 de março de 2026**. Regulamenta a Lei nº 15.211, de 17 de setembro de 2025, e institui a Política Nacional de Proteção dos Direitos da Criança e do Adolescente no Ambiente Digital. Brasília, DF: Presidência da República, 2026b. Disponível em: https://www.planalto.gov.br/ccivil_03/_ato2023-2026/2026/decreto/d12880.htm. (Conferir a ementa.)

BRASIL. Câmara dos Deputados. **Projeto de Lei nº 2338, de 2023**. Dispõe sobre o desenvolvimento, o fomento e o uso ético e responsável da inteligência artificial. Ficha de tramitação. Disponível em: https://www.camara.leg.br/proposicoesWeb/fichadetramitacao?idProposicao=2487262. Acesso em: 16 out. 2026.

CONGRESSO NACIONAL. **Medida Provisória nº 1319, de 2025**: ficha de tramitação. Brasília, DF, 2026. Disponível em: https://www.congressonacional.leg.br/materias/medidas-provisorias/-/mpv/170522. Acesso em: 16 out. 2026.

UNIÃO EUROPEIA. Regulamento (UE) 2024/1689 do Parlamento Europeu e do Conselho, de 13 de junho de 2024, que cria regras harmonizadas em matéria de inteligência artificial (Regulamento da Inteligência Artificial). **Jornal Oficial da União Europeia**, L, 12 jul. 2024. Disponível em: https://eur-lex.europa.eu/eli/reg/2024/1689/oj. Arts. 2º, 5º, 6º, 12, 14, 26, 50 e 86; Anexo III.

UNIÃO EUROPEIA. **Regulamento (UE) 2026/1744** (*Digital Omnibus* sobre IA). Jornal Oficial da União Europeia, 2026. (Conferir no EUR-Lex.)

**Material da disciplina.** Nesta aula, [nota sobre governar a ação](01-governar-a-acao.md) — ponto de aplicação de política, *fail-closed*, limiar de escalonamento, supervisão humana e interrupção; [nota sobre provar a decisão](02-provar-a-decisao.md) — registro de decisão, guarda, *model card*, transparência e o caso do parecer contestado. Memória, [nota sobre esquecer e proteger](../../aula-08-memoria/notas-de-aula/04-esquecer-e-proteger.md) — remoção sob solicitação e a estrutura que escapa (§3). Escolha e configuração de modelos, [nota sobre escolha de modelos](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — onde o modelo roda (§8).
