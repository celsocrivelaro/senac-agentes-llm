# IA Aplicada com LLMs — Aula 12: Governança de IA agêntica — Provar a decisão

---

## Objetivos de aprendizagem

Ao final desta nota, espera-se que o aluno seja capaz de:

- **Especificar** o registro de uma decisão do agente — entrada, contexto recuperado, versões de modelo, prompt e índice, ferramentas, política aplicada e sua versão, revisão humana e saída —, com identificador, prazo de guarda e o mínimo de dado pessoal que basta para reconstruí-la.
- **Distinguir** reconstruir uma decisão a partir do registro de reproduzi-la por nova execução, e **listar** o que precisa estar versionado em conjunto para que a reprodução seja possível.
- **Redigir** o *model card* de um agente, com a autoridade delegada entre os campos, e **situá-lo** entre os documentos de uma auditoria interna ao longo do ciclo de vida.
- **Aplicar** a minimização como critério de projeto do registro — referência onde a referência basta, conteúdo onde o conteúdo é a prova — e **fixar** o prazo de guarda pela finalidade de prova.
- **Redigir** a informação que o usuário recebe sobre o agente, separada da documentação dirigida ao auditor.
- **Responder** a uma contestação a partir do registro do sistema — incluindo um pedido de revisão de decisão automatizada e um de eliminação — e **apontar** as lacunas de governança onde o registro não responde.
- **Descrever** o ciclo de vida do agente até a desativação.

---

## Desenvolvimento teórico

A [nota sobre governar a ação](01-governar-a-acao.md) tratou do que o agente está autorizado a fazer e de quem aplica esse limite. Esta nota trata da segunda pergunta da aula: diante de uma decisão contestada, o que o sistema consegue provar sobre o que decidiu, com base em quê e sob a responsabilidade de quem. O exemplo é o auditor de despesas do curso, que emitiu o parecer `P-1001`, recusando a despesa `D-4471` do funcionário `F-088`; o funcionário contestou. As obrigações legais que o registro e a documentação atendem — LGPD, ECA Digital, AI Act e o PL 2338/2023 — estão na [nota sobre legislação](03-legislacao.md); aqui a lei aparece só onde decide um requisito de projeto, com remissão ao texto.

### 1. O *trace* como evidência

**Rastreabilidade** (*traceability*) de uma decisão automatizada é a propriedade de o sistema guardar, para cada decisão, registro suficiente para que um terceiro — auditor, titular dos dados, regulador — identifique o que foi decidido, a partir de que informações, por qual versão do sistema, sob quais regras e com que participação humana. O *trace* modelado em arquitetura de agentes ([nota sobre confiabilidade](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md), §9) nasceu como instrumento de depuração e, na aula de observabilidade, passou a alimentar métricas agregadas ([nota sobre observabilidade](../../aula-14-mlops-observabilidade/notas-de-aula/02-observabilidade.md)). Usado como **evidência**, o mesmo registro muda de leitor: deixa de ser lido pela equipe que o definiu e passa a ser lido por quem não conhece o sistema e precisa confiar no que lê.

Formalmente, o **registro de uma decisão** é a tupla

$$R = (\mathit{id},\ t,\ x,\ C,\ V,\ \Phi,\ P,\ H,\ y)$$

em que:

- $\mathit{id}$ é o identificador da decisão, único e estável, citado no próprio parecer entregue ao interessado;
- $t$ é o instante;
- $x$ é a entrada — o pedido e os dados da despesa;
- $C$ é o contexto recuperado: os trechos do regulamento, com o identificador de cada trecho no índice, e os dados históricos consultados;
- $V = (v_{\text{modelo}},\ v_{\text{prompt}},\ v_{\text{índice}},\ v_{\text{sistema}})$ são as versões — o carimbo da aula de *prompt engineering* ([nota sobre o prompt como código](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md)) acrescido da versão do índice;
- $\Phi$ é a sequência de chamadas de ferramenta, com argumentos e resultados;
- $P = (\text{regras avaliadas},\ v_{\text{política}},\ \text{resultado})$ é a política aplicada pelo ponto de aplicação descrito na [nota sobre governar a ação](01-governar-a-acao.md) (§6.1), com a versão da política vigente no instante $t$;
- $H$ é a revisão humana: quem revisou, quando, o que viu e o que decidiu — ou o registro explícito de que não houve revisão;
- $y$ é a saída: o parecer e a sua fundamentação.

```text
 id = P-1001        t = 2026-10-05T14:32:07Z       despesa D-4471
+------------------------------------------------------------------+
| ENTRADA x      pedido, valor, categoria, comprovante (ref.)       |
+------------------------------------------------------------------+
| CONTEXTO C     trechos do regulamento: reg#4.2, reg#4.7           |
|                histórico consultado: F-088 (ref. pseudonimizada)  |
+------------------------------------------------------------------+
| VERSÕES V      modelo pedido  mistral-small-latest                |
|                modelo servido (resposta do provedor)              |
|                prompt v3      índice idx-2026-09-30    sistema 1.5|
+------------------------------------------------------------------+
| FERRAMENTAS    buscar_despesa > consultar_politica >              |
|                buscar_regulamento > registrar_parecer             |
+------------------------------------------------------------------+
| POLÍTICA P     politica v7: limite_valor OK, limiar_humano NÃO    |
+------------------------------------------------------------------+
| REVISÃO H      nenhuma  (registrada como ausência)                |
+------------------------------------------------------------------+
| SAÍDA y        RECUSADA - fundamento: reg#4.7                     |
+------------------------------------------------------------------+
```

Três decisões de projeto determinam se esse registro serve de prova.

**O identificador.** Shavit et al. (2023) chamam de **atribuibilidade** (*attributability*) a possibilidade de ligar uma ação de agente ao sistema e à organização responsáveis; Chan et al. (2024) propõem identificadores de agente e registro de atividade como medidas de visibilidade. No nível da decisão, a consequência é que o identificador do registro tem de aparecer no artefato que chega ao interessado — o parecer `P-1001` cita o seu próprio registro —, porque a contestação chega pelo parecer, e não pelo *trace*.

**A ausência registrada.** Um campo $H$ vazio é ambíguo: pode significar que não houve revisão ou que a revisão não foi registrada. O registro precisa distinguir os dois casos, porque o direito de revisão do art. 20 da LGPD depende de a decisão ter sido tomada **unicamente** com base em tratamento automatizado ([nota sobre legislação](03-legislacao.md), §2.4).

**A imutabilidade.** Um registro que a equipe operadora pode editar prova pouco. A prática é gravá-lo em armazenamento só de acréscimo (*append-only*), com controle de acesso separado do da operação, e, quando o risco justificar, encadear os registros por *hash* — cada registro carrega o *hash* do anterior, e uma alteração posterior quebra a cadeia.

### 2. Guardar o suficiente, e não tudo

O registro tensiona dois critérios. A rastreabilidade pede guardar mais; a **minimização** pede guardar só o dado pessoal necessário à finalidade — critério de projeto que a LGPD impõe como princípio da necessidade (art. 6º, III; [nota sobre legislação](03-legislacao.md), §2.2). O *log* de um agente é, quase sempre, um repositório de dado pessoal: o prompt carrega o nome e o histórico do funcionário, a resposta do modelo os repete, e os argumentos das ferramentas contêm o identificador — o caso do *checkpoint* que escapou da remoção, na aula de memória ([nota sobre esquecer e proteger](../../aula-08-memoria/notas-de-aula/04-esquecer-e-proteger.md), §3.1).

A regra que resolve o tensionamento é guardar **referência** onde a referência basta e **conteúdo** apenas onde o conteúdo é a prova:

| Elemento | Guardar | Não guardar |
|---|---|---|
| Trechos do regulamento | identificador do trecho e versão do índice (o conteúdo é recuperável do índice versionado) | — |
| Dados da despesa | identificador e *hash* do documento | cópia integral do comprovante no *log* |
| Identificador do titular | pseudônimo, com a tabela de correspondência em repositório de acesso restrito | nome e dados cadastrais repetidos em cada *span* |
| Prompt montado | *hash* do texto e versão do *template* | texto integral, salvo quando a política de retenção o exigir |
| Saída | o parecer e sua fundamentação | — |

O registro é só um dos lugares onde o dado pessoal cai. Os outros três — o prompt enviado ao provedor, o índice e a memória — pedem a mesma regra, e a lista dos quatro sai de um inventário de onde o dado efetivamente cai, e não da arquitetura pretendida ([nota sobre legislação](03-legislacao.md), §2.2). O prompt tem uma particularidade: é enviado a um terceiro. Por isso o campo $V$ do registro inclui o provedor e a região de processamento, porque a pergunta "para onde foi o dado pessoal" se responde por ele; a base legal dessa transferência, a retenção e o uso para treino pelo provedor estão na [nota sobre legislação](03-legislacao.md) (§2.6).

**Por quanto tempo.** O prazo de guarda é fixado pela finalidade de prova: o registro precisa existir enquanto a decisão puder ser contestada. Na relação de trabalho, uma referência é o prazo prescricional do art. 7º, XXIX, da Constituição — cinco anos, até o limite de dois anos após a extinção do contrato; o AI Act fixa um piso de seis meses para sistemas de alto risco ([nota sobre legislação](03-legislacao.md), §4.2). Vencido o prazo, o registro é eliminado — a LGPD só autoriza a conservação enquanto houver finalidade que a justifique ([nota sobre legislação](03-legislacao.md), §2.3) —, e a eliminação também é registrada. Um prazo indefinido não é prudência: é tratamento sem finalidade.

### 3. Reproduzir a decisão de ontem

Há duas operações distintas sobre uma decisão passada:

- **reconstrução**: explicar a decisão a partir do registro $R$ — o que entrou, o que foi recuperado, que regra valeu, o que saiu. Exige apenas o registro;
- **reprodução**: executar de novo o sistema nas mesmas condições e obter saída equivalente. Exige que todas as versões de $V$ ainda existam e possam ser carregadas.

A reprodução é o que permite responder se o erro foi da decisão ou do sistema — se, com o regulamento corrigido, o mesmo agente teria aprovado a despesa. Ela exige que quatro artefatos estejam **versionados em conjunto**, de forma que uma versão do sistema identifique uma combinação única deles:

```text
  versão do sistema 1.5  (tag da imagem)
        |
        +--> modelo     mistral-small-2506   (versão datada, não alias)
        +--> prompt     v3                   (prompt registry)
        +--> índice     idx-2026-09-30       (hash do corpus + embeddings)
        +--> suíte      evals v12            (golden dataset + juiz)
```

A suíte entra no conjunto porque é ela que diz se a reprodução foi bem-sucedida: com temperatura positiva, uma nova execução não devolve o mesmo texto, e a comparação é feita pelo critério de correção da aula de *evals* ([nota sobre o juiz e a suíte](../../aula-11-evals/notas-de-aula/03-o-juiz-e-a-suite.md)), e não por igualdade de cadeias. É a linhagem da aula de MLOps ([nota sobre MLOps](../../aula-13-mlops-empacotamento/notas-de-aula/02-mlops.md), §8) aplicada ao agente: o registro de modelos de MLOps responde de onde veio o modelo; aqui, a pergunta é de onde veio a decisão.

O elo frágil é o modelo. O *alias* `mistral-small-latest` é um ponteiro que o provedor move sem aviso ao cliente ([nota sobre escolha de modelos](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md), §7.2), e um modelo datado é desligado na deprecação. Daí duas consequências. Primeiro, o registro grava o modelo **efetivamente servido**, devolvido pelo provedor na resposta, e não só o modelo pedido — nas convenções do OpenTelemetry, `gen_ai.response.model` ao lado de `gen_ai.request.model`. Segundo, a reprodução de uma decisão tem horizonte limitado pelo ciclo de vida do modelo no provedor; passado esse horizonte, resta a reconstrução, e é por isso que o registro precisa bastar sozinho.

O índice tem fragilidade análoga: reindexar o corpus com outro modelo de *embedding* ou outro tamanho de trecho muda o que é recuperado para a mesma pergunta. Guardar o *hash* do índice só serve à reprodução se o índice com aquele *hash* for preservado, ou se o *pipeline* que o produziu for determinístico sobre o corpus versionado.

### 4. O *model card* do agente

O ***model card*** (MITCHELL et al., 2019) é um documento curto que acompanha um modelo e declara detalhes do modelo, uso pretendido, fatores relevantes, métricas, dados de avaliação e de treino, análises quantitativas, considerações éticas e ressalvas — tratado como prática de governança na aula de MLOps ([nota sobre MLOps](../../aula-13-mlops-empacotamento/notas-de-aula/02-mlops.md), §14). Para um agente, o formato precisa de adaptação: o agente não treina modelo próprio, de modo que "dados de treino" perde o centro, e o risco principal deixa de ser só o da saída e passa a ser o da ação. O campo que falta no formato original é a **autoridade**.

Proposta de campos para o *model card* do agente:

| Campo | Conteúdo | Origem |
|---|---|---|
| Identificação | nome, versão do sistema, data, responsáveis nomeados | registro de versões |
| Finalidade | o que o agente decide ou executa; usos fora de escopo | especificação |
| **Autoridade** | escopo, ações proibidas, credencial e seus privilégios, limites de gasto, limiares de aprovação humana, quem pode suspender | [nota sobre governar a ação](01-governar-a-acao.md), §8 |
| Ferramentas e integrações | cada ferramenta e servidor MCP, com a permissão que concede (leitura, escrita, envio) | inventário de ferramentas |
| Modelo e provedor | versão datada, provedor, região de processamento, termos de retenção e de uso para treino | contrato com o provedor |
| Dados | fontes, dado pessoal tratado, base legal, onde cada dado fica e por quanto tempo | inventário de dados ([nota sobre legislação](03-legislacao.md), §§2.1–2.2) |
| Avaliação | suíte e versão, resultados agregados e por fatia, concordância do juiz | aula de *evals* |
| Limites conhecidos | modos de falha observados, casos em que o agente não deve ser usado | avaliação e incidentes |
| Riscos aceitos | cada risco aceito, com a razão e o que o limita | aula de segurança |
| Supervisão humana | o que pausa para aprovação, quem aprova, como se suspende o agente | [nota sobre governar a ação](01-governar-a-acao.md), §7 |
| Registro e retenção | o que se registra por decisão, prazo de guarda, quem acessa | §§1–2 |
| Histórico de mudanças | cada versão, o que mudou, quem aprovou | registro de versões |

Os riscos aceitos vêm do registro de risco da aula de segurança ([nota sobre cadeia, rede e o fecho](../../aula-15-seguranca/notas-de-aula/03-cadeia-rede-e-o-fecho.md), Exemplo 4). A classificação regulatória do sistema — se é de alto risco no AI Act, se é de acesso provável por crianças e adolescentes no ECA Digital —, com a justificativa, entra na finalidade ou nos limites conhecidos ([nota sobre legislação](03-legislacao.md), §§3.2 e 4.1). O critério de qualidade do documento é que um auditor sem acesso à equipe consiga, só com ele, decidir se uma decisão concreta estava dentro da autoridade do agente.

### 5. Auditoria interna ao longo do ciclo de vida

**Auditoria algorítmica interna** é a verificação, conduzida pela própria organização antes e durante o uso do sistema, de que as decisões de projeto e de operação estão alinhadas aos princípios e às obrigações que a organização declarou. Raji et al. (2020) propõem o *framework* SMACTR, em que a auditoria acompanha o ciclo de desenvolvimento em cinco etapas — *Scoping* (escopo), *Mapping* (mapeamento), *Artifact collection* (coleta de artefatos), *Testing* (testes) e *Reflection* (reflexão) — e cada etapa produz documentos que, reunidos, formam o relatório final de auditoria.

A tese do artigo é que a auditoria só tem o que ler se os documentos forem produzidos no momento em que as decisões são tomadas; reconstruí-los depois de um incidente produz justificativa, e não registro. Aplicado ao agente:

| Fase | Documento produzido |
|---|---|
| Escopo | finalidade, autoridade pretendida, avaliação de impacto inicial |
| Mapeamento | partes interessadas, titulares afetados, inventário de dados e de ferramentas |
| Coleta de artefatos | *model card*, especificação de autoridade, registro de risco, contrato com o provedor |
| Testes | resultados da suíte, testes dos limites de autoridade, teste do desligamento |
| Reflexão | riscos aceitos, plano de mitigação, decisão de implantar ou não |

Em operação, a auditoria passa a amostrar registros de decisão e a conferir se cada um está dentro da autoridade declarada no *model card* — o que só é possível porque o registro da §1 inclui a política aplicada e a sua versão.

### 6. Transparência para o usuário

O *model card* é escrito para o auditor; o usuário do agente precisa de outra informação, em outro lugar. O *Model AI Governance Framework for Agentic AI* de Singapura, na versão 1.5 (IMDA, 2026), dedica uma das quatro dimensões do marco a habilitar a responsabilidade do usuário final e recomenda informar ao usuário, entre outros pontos:

- as **responsabilidades do próprio usuário**, como conferir a informação fornecida pelo agente;
- que a interação é com um agente, declarado na interface no ponto de interação, e não só em documentação separada;
- o **conjunto de ações e decisões** que o agente está autorizado a executar e a tomar;
- como os dados do usuário são coletados, armazenados e usados pelo agente;
- os **contatos humanos responsáveis**, a quem recorrer quando o agente falhar ou quando o usuário discordar de uma decisão.

O último item é o canal pelo qual a contestação do funcionário chega à organização. No auditor de despesas, a consequência é concreta: o parecer entregue ao funcionário declara que foi emitido por um agente, cita o identificador do registro, indica o trecho do regulamento que o fundamenta e informa a quem pedir revisão. O AI Act exige transparência equivalente, no art. 26, §11, para os sistemas de alto risco do Anexo III que tomam ou auxiliam decisões sobre pessoas ([nota sobre legislação](03-legislacao.md), §4.4).

### 7. O caso do parecer contestado

O funcionário `F-088` contestou o parecer `P-1001`, que recusou a despesa `D-4471`. As perguntas que a contestação impõe à organização são respondidas, uma a uma, a partir do registro da §1.

| Pergunta | Onde o registro responde | Onde não responde |
|---|---|---|
| Versão de modelo, prompt e índice que produziu o parecer | $V$: prompt v3, índice `idx-2026-09-30`, sistema 1.5 | o modelo foi pedido como `mistral-small-latest`; se o modelo servido não foi gravado, não se sabe qual modelo decidiu |
| Trecho do regulamento recuperado | $C$: `reg#4.2` e `reg#4.7`; a saída cita `reg#4.7` | se o índice `idx-2026-09-30` não foi preservado, o conteúdo do trecho naquela data não pode ser exibido |
| Política aplicada | $P$: política v7, limiar humano não atingido | se a versão da política não foi gravada, não se sabe se o limiar era o atual |
| Dado pessoal no contexto, e destino | $C$ e $x$: pseudônimo e histórico do titular; provedor e região em $V$ | sem inventário de dados, não se sabe se o histórico também foi para a memória episódica |
| Revisão humana, e de quem | $H$: ausência registrada | se $H$ não distingue "não houve" de "não registrada", a pergunta não tem resposta |
| Atendimento ao pedido de revisão (LGPD, art. 20) | a ausência em $H$ caracteriza decisão unicamente automatizada; os critérios (regulamento e política versionados) e o procedimento ($\Phi$) atendem ao § 1º | se não há procedimento de revisão definido e responsável nomeado, o pedido não tem a quem ir |
| Atendimento ao pedido de eliminação | memória, fatos e *checkpoint* removidos e verificados; registro de decisão conservado pela finalidade de prova (LGPD, art. 16) | se o registro guarda o nome em texto livre em cada *span*, a conservação guarda mais do que a prova exige |

A tabela mostra o argumento central desta nota. Cada linha da coluna da direita é uma **lacuna de governança**: não um defeito do modelo, e sim um campo que o registro não guardou, um artefato que não foi versionado ou um procedimento que não foi definido. Todas elas são decisões de projeto tomadas — ou omitidas — antes da contestação; nenhuma pode ser corrigida depois dela.

O atendimento concreto do pedido de revisão segue do registro. Como $H$ registra a ausência de revisão humana, a decisão é unicamente automatizada e o funcionário tem direito à revisão prevista no art. 20 da LGPD; o texto do artigo, o que se entende por "unicamente" e o estado da regulamentação pela ANPD estão na [nota sobre legislação](03-legislacao.md) (§§2.4–2.5). A revisão é atribuída à pessoa nomeada no *model card* como responsável, que recebe o registro completo — entrada, trechos recuperados, política e saída — e decide; a decisão do revisor entra no registro como um novo $H$ ligado a `P-1001`, sem sobrescrever o original. Se a revisão aprova a despesa, a investigação seguinte é a reprodução (§3): executar o sistema 1.5 sobre a mesma entrada e verificar se o erro se repete, o que separa um erro do regulamento, um erro de recuperação e um erro do modelo.

### 8. O ciclo de vida até a desativação

A governança do agente acompanha o sistema da concepção ao desligamento, e cada fase deixa um documento ou um controle verificável:

```text
 DESENHO        finalidade, autoridade e grau de autonomia definidos
    |           antes de construir; classificação de risco
    v
 TESTE          suíte de evals; testes dos limites de autoridade;
    |           teste do desligamento
    v
 IMPLANTAÇÃO    responsáveis nomeados; model card publicado;
    |           usuários e trabalhadores informados
    v
 OPERAÇÃO       ponto de aplicação de política; registro de decisões;
    |           supervisão humana; monitoramento e avaliação online
    v
 REVISÃO        deriva de autoridade; mudança de modelo pelo provedor;
    |           auditoria amostral dos registros
    v
 DESATIVAÇÃO    credenciais revogadas; integrações desligadas;
                dados operacionais eliminados; registros preservados
```

A **desativação** é a fase menos planejada e a que mais deixa resíduo. Um agente desativado só está desativado quando:

- as **credenciais** de serviço do agente estão revogadas, e não apenas sem uso — uma chave válida de um agente parado é uma credencial órfã;
- as **integrações** estão desligadas: servidores MCP desconectados, *webhooks* removidos, tarefas agendadas canceladas, chaves de API do provedor de modelo revogadas;
- os **dados operacionais** — memória, índice com dado pessoal, *checkpoints* — estão eliminados, com a eliminação verificada por busca posterior;
- os **registros de decisão** estão preservados pelo prazo de guarda, com o *model card* da última versão e os artefatos versionados necessários para reconstruir as decisões, e acesso atribuído a um responsável que continue existindo depois do sistema.

O último item fecha o argumento desta nota: as decisões tomadas pelo agente continuam contestáveis depois que o agente deixa de existir, e a organização que o implantou continua respondendo por elas.

---

## Fontes e leituras

BRASIL. **Lei nº 13.709, de 14 de agosto de 2018**. Lei Geral de Proteção de Dados Pessoais (LGPD). Brasília, DF: Presidência da República, 2018. Disponível em: https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm. Acesso em: 16 out. 2026.

CHAN, A. et al. Visibility into AI agents. In: ACM CONFERENCE ON FAIRNESS, ACCOUNTABILITY, AND TRANSPARENCY (FAccT), 2024. **Proceedings** [...]. ACM, 2024.

INFOCOMM MEDIA DEVELOPMENT AUTHORITY (IMDA). **Model AI Governance Framework for Agentic AI**. Version 1.5. Singapura, 20 maio 2026 (atualizado em 5 jun. 2026). Disponível em: https://www.imda.gov.sg/-/media/imda/files/about/emerging-tech-and-research/artificial-intelligence/mgf-for-agentic-ai.pdf. Acesso em: 16 out. 2026.

KOLT, N. Governing AI agents. **Notre Dame Law Review**, 2025. Disponível em: https://arxiv.org/abs/2501.07913.

MITCHELL, M. et al. Model cards for model reporting. In: CONFERENCE ON FAIRNESS, ACCOUNTABILITY, AND TRANSPARENCY (FAT*), 2019. **Proceedings** [...]. ACM, 2019.

OPENTELEMETRY. **Semantic conventions for generative AI systems**. Disponível em: https://opentelemetry.io/docs/specs/semconv/gen-ai/.

RAJI, I. D. et al. Closing the AI accountability gap: defining an end-to-end framework for internal algorithmic auditing. In: CONFERENCE ON FAIRNESS, ACCOUNTABILITY, AND TRANSPARENCY (FAT*), 2020. **Proceedings** [...]. ACM, 2020. Disponível em: https://arxiv.org/abs/2001.00973.

SHAVIT, Y. et al. **Practices for governing agentic AI systems**. OpenAI, 2023. Disponível em: https://cdn.openai.com/papers/practices-for-governing-agentic-ai-systems.pdf.

UNIÃO EUROPEIA. **Regulamento (UE) 2024/1689** do Parlamento Europeu e do Conselho, de 13 de junho de 2024 (Regulamento da Inteligência Artificial). Jornal Oficial da União Europeia, 2024. Disponível em: https://eur-lex.europa.eu/eli/reg/2024/1689/oj.

A bibliografia legal e regulatória completa — LGPD, ECA Digital, AI Act, PL 2338/2023 e os documentos da ANPD — está na [nota sobre legislação](03-legislacao.md).

**Leitura complementar (textos de fornecedores).**

HARKAR, S. **Agentic AI governance playbook**. IBM Think, [2026]. Disponível em: https://www.ibm.com/think/insights/agentic-ai-governance-playbook. — Decisões antes da implantação e ciclo de vida.

PALO ALTO NETWORKS. **What is agentic AI governance?** Cyberpedia, [2026]. Disponível em: https://www.paloaltonetworks.com/cyberpedia/what-is-agentic-ai-governance. — Ciclo de vida do agente.

**Material da disciplina.** Nesta aula, [nota sobre governar a ação](01-governar-a-acao.md) — autoridade delegada, ponto de aplicação de política, supervisão humana e desligamento; [nota sobre legislação](03-legislacao.md) — LGPD, ECA Digital, AI Act e PL 2338/2023, e a correspondência entre obrigações e controles. Arquitetura de agentes, [nota sobre confiabilidade](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md) — o *trace* como subproduto (§9). Observabilidade, [nota sobre observabilidade](../../aula-14-mlops-observabilidade/notas-de-aula/02-observabilidade.md) — o *trace* em convenção aberta e o corte por versão. MLOps e empacotamento, [nota sobre MLOps](../../aula-13-mlops-empacotamento/notas-de-aula/02-mlops.md) — linhagem, registro de modelos e *model cards* (§§8 e 14). *Prompt engineering*, [nota sobre o prompt como código](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md) — o carimbo de versão. Memória, [nota sobre esquecer e proteger](../../aula-08-memoria/notas-de-aula/04-esquecer-e-proteger.md) — remoção sob solicitação e a estrutura que escapa (§3). Escolha e configuração de modelos, [nota sobre escolha de modelos](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — o *alias* sem versão, a deprecação e onde o modelo roda (§§7 e 8). *Evals*, [nota sobre o juiz e a suíte](../../aula-11-evals/notas-de-aula/03-o-juiz-e-a-suite.md) — o critério de correção que julga a reprodução. Segurança, [nota sobre cadeia, rede e o fecho](../../aula-15-seguranca/notas-de-aula/03-cadeia-rede-e-o-fecho.md) — o risco aceito, declarado.
