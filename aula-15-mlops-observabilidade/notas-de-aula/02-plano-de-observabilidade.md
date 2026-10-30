# IA Aplicada com LLMs — Aula 15: Produção — O plano de observabilidade

---

## Objetivos de aprendizagem

Ao final desta nota, espera-se que o aluno seja capaz de:

- **Definir** plano de observabilidade e explicar por que ele precede a instrumentação.
- **Derivar** as perguntas operacionais de um sistema de agentes a partir dos seus interessados, e descartar as que não informam decisão.
- **Modelar** o sistema como fluxo de trabalho: a unidade de trabalho, as fronteiras de processo e as dependências externas.
- **Analisar** os modos de falha de um agente, classificando-os em ruidosos ou silenciosos, abruptos ou de deriva, locais ou sistêmicos.
- **Formular** SLIs e SLOs operacionais e de qualidade para um agente, ligados ao ganho que o sistema prometeu.
- **Construir** a matriz de rastreabilidade entre perguntas, sinais, atributos e consumidores, e declarar a amostragem, a retenção, os alertas e o que fica fora da telemetria.

---

## Desenvolvimento teórico

### 1. O plano de observabilidade

Um **plano de observabilidade** é o documento que declara, antes da instrumentação, que perguntas o sistema precisa responder enquanto opera, que falhas ele precisa tornar visíveis, que objetivos ele precisa cumprir, e, como consequência, que sinais ele emite, com que atributos, por quanto tempo são guardados e quem age sobre eles. A instrumentação implementa o plano; o plano não descreve a instrumentação.

A ordem importa porque cada sinal tem custo — de emissão, de armazenamento e, o mais escasso, de atenção de quem lê o painel — e porque o sinal decisivo num agente raramente é um dos que a plataforma oferece pronta. O motivo de término, que a nota sobre observabilidade trata como a métrica que quase nenhum painel comercial tem, só existe porque foi **projetado** como dado na aula de arquitetura de agentes. Nenhuma instrumentação automática o produziria.

O plano encadeia nove decisões, cada uma alimentada pelas anteriores:

```text
  perguntas ──► fluxo de trabalho ──► modos de falha ──► objetivos (SLO)
                                                              │
       ┌──────────────────────────────────────────────────────┘
       ▼
  sinais e cortes ──► amostragem e retenção ──► qualidade ──► ação
       │
       └──► o que fica fora

  matriz de rastreabilidade: liga cada pergunta ao sinal que a responde
```

A consequência mais direta do encadeamento é que a lista de métricas é **saída** do plano, e não entrada. As quatro métricas de operação da nota sobre observabilidade resultam deste método aplicado ao auditor de despesas (Exemplo 1); outra arquitetura, com outros modos de falha, chega a outra lista (Exemplo 3).

### 2. As perguntas, e quem as faz

Uma **pergunta operacional** é um enunciado que se responde por consulta à telemetria e que tem um **interessado** e uma **decisão** associada: a resposta muda o que alguém faz. Observabilidade é a capacidade de responder perguntas sobre o estado interno a partir do que o sistema emite (MAJORS; FONG-JONES; MIRANDA, 2022); o plano começa, por isso, pelas perguntas, e não pelos sinais.

Num sistema de agentes, quatro interessados formulam perguntas de natureza distinta:

| Interessado | Natureza da pergunta | Exemplo, no auditor de despesas |
|---|---|---|
| **o negócio** | o sistema entrega o ganho prometido? | a análise automática reduziu o tempo de conferência na proporção prometida? |
| **a operação** | o sistema funciona agora, e onde quebrou? | quantas análises terminaram sem veredito na última hora? |
| **quem paga** | quanto custa, e por quê? | qual é o custo por despesa analisada, e qual componente o domina? |
| **quem responde pelo sistema** | ele agiu dentro do que lhe foi delegado? | alguma análise consultou dado fora da despesa pedida? |

O critério de corte é a decisão. Uma pergunta cuja resposta não altera nenhuma ação é removida do plano: o sinal que a responderia seria emitido, guardado e nunca usado. *Quantas vezes por dia a ferramenta `consultar_politica` é chamada?* sai do plano do auditor, a menos que alguém dimensione capacidade a partir dela; *qual a taxa de erro de `buscar_regulamento`?* fica, porque decide se a integração com o regulamento precisa de correção.

A quarta linha da tabela liga o plano às aulas de governança e de segurança: a rastreabilidade de uma decisão automatizada e o uso anômalo de ferramenta são perguntas que só a telemetria responde, e precisam constar do plano para que o sinal exista quando forem feitas.

### 3. O sistema como fluxo de trabalho

A **unidade de trabalho** é a porção do processamento que o sistema trata como um todo, do pedido ao resultado: aquilo que o usuário reconhece como *uma* análise, *uma* resposta, *um* caso. Ela define o que é um *trace*, e com isso o denominador de todas as métricas por execução — custo por execução, latência por execução, proporção de cada motivo de término.

A escolha não é trivial em três arranjos recorrentes:

| Arranjo | Unidade de trabalho | Consequência para o *trace* |
|---|---|---|
| pedido único, como o auditor | a execução | um *trace* por execução, com `invoke_agent` na raiz |
| conversa de vários turnos | o turno, agrupado pela conversa | um *trace* por turno, com o identificador da conversa como atributo (`gen_ai.conversation.id`); métricas por turno e por conversa são distintas |
| execução com confirmação humana | o caso, que pode durar dias | um *trace* até a pausa e outro na retomada, ligados por *span link*; a espera humana fica fora da latência do agente |

O terceiro caso merece atenção. Se a pausa da confirmação humana, tratada nas aulas de arquitetura de agentes e de *frameworks*, ficar dentro do *trace*, o p95 da latência passa a medir o tempo de resposta do aprovador, e não o do agente. A espera é dado relevante — mede o gargalo humano —, mas é outra métrica.

Definida a unidade, o fluxo é descrito por três elementos:

- **fronteiras de processo**: cada ponto em que o trabalho atravessa de um processo a outro — o servidor MCP, o serviço que recebe o pedido, outro agente. Em cada fronteira, o contexto do *trace* precisa ser propagado (nota sobre monitoração e telemetria, §9), ou a árvore se parte em *traces* desconexos;
- **componentes**, separados em **não determinísticos** — o modelo, o juiz — e **determinísticos** — ferramenta, banco, índice. Os primeiros falham por qualidade, e os segundos, por exceção;
- **dependências externas**: o provedor do modelo, APIs de terceiros. Falham por motivos fora do controle da aplicação e mudam sem aviso.

```text
  pedido ──► [ serviço ] ──► invoke_agent (unidade de trabalho)
                               │
                 ┌─────────────┼──────────────────┐
                 ▼             ▼                  ▼
              chat ×n     execute_tool ×m     recuperação
            (provedor:    (consultar_*:       (regulamento:
             externo,      determinístico)     determinístico,
             não determ.)                      versão do índice)
                               │
                               ▼
                    término + veredito  ◄── dado da aplicação
```

### 4. Modos de falha

A **análise de modos e efeitos de falha** (FMEA, *failure mode and effects analysis*) é um método da engenharia de confiabilidade que, para cada componente de um sistema, enumera as formas como ele pode falhar, o efeito de cada falha sobre o sistema e o meio pelo qual ela é detectada (IEC, 2018). Adaptada à observabilidade, a análise produz, para cada componente $c$ e cada modo de falha $m$ de $c$, a quádrupla

$$(c, m) \mapsto (\text{sintoma}, \text{sinal}, \text{ponto de emissão}, \text{classe})$$

em que o **sintoma** é como a falha aparece por fora, o **sinal** é o dado que a captura e o **ponto de emissão** é onde, no fluxo da §3, esse dado nasce. Uma linha com o sinal vazio não é erro da análise: é um achado, e vira requisito de projeto.

A classe combina três eixos:

| Eixo | Polos | Consequência para a detecção |
|---|---|---|
| **manifestação** | ruidosa — exceção, código de erro — ou **silenciosa** — `200` com resultado pior | a ruidosa aparece na taxa de erro; a silenciosa, não |
| **evolução** | abrupta — quebra num instante — ou **deriva** — degradação gradual | a abrupta se vê num limiar; a deriva, só por comparação entre janelas ou versões |
| **alcance** | local — uma ferramenta — ou sistêmica — a execução inteira | a local exige o corte por componente para ser localizada |

Os modos típicos de um agente vêm do que as aulas anteriores construíram:

| Componente | Modo de falha | Classe | Sintoma |
|---|---|---|---|
| laço do agente | esgota o orçamento de passos | silenciosa, deriva, sistêmica | resposta pior, devolvida com sucesso |
| laço do agente | repete a mesma chamada | silenciosa, abrupta, sistêmica | custo e latência altos, término por laço |
| modelo | escolhe a ferramenta errada ou passa argumento inválido | silenciosa, abrupta, local | erro de domínio devolvido ao modelo como observação |
| modelo | responde fora do formato pedido | silenciosa, abrupta, sistêmica | resposta sem veredito |
| modelo | conclui errado | silenciosa, deriva, sistêmica | nenhum sintoma operacional |
| recuperação | devolve trechos vazios ou irrelevantes | silenciosa, deriva, local | resposta genérica ou recusa |
| provedor | indisponibilidade ou cota | ruidosa, abrupta, sistêmica | erro de API |
| provedor | troca o modelo por trás do mesmo nome | silenciosa, deriva, sistêmica | mudança de acerto sem mudança de versão |
| entrada | injeção de prompt | silenciosa, abrupta, local | ferramenta chamada fora do padrão |
| histórico | cresce sem compressão | silenciosa, deriva, sistêmica | custo por execução em alta |

A tabela sustenta a tese desta nota: **num agente, os modos de falha relevantes são, em maioria, silenciosos**, e um sinal técnico genérico — taxa de erro, latência — não os captura. Eles só se tornam visíveis por **sinais semânticos que a aplicação produz**: o motivo de término, o veredito extraído da resposta, o acerto amostrado. Nenhuma convenção nem plataforma fornece esses sinais, porque eles dependem do que *funcionar* significa naquele domínio. Projetá-los é tarefa da arquitetura do sistema, e é o que distingue o plano de observabilidade de um agente do de um serviço convencional.

### 5. Objetivos: SLIs e SLOs do agente

SLI, SLO e orçamento de erro estão definidos na nota sobre monitoração e telemetria (§1). No agente, os indicadores formam duas famílias:

- **operacionais**: latência por execução, proporção de execuções com término normal, custo por execução. Medem *como* o sistema respondeu;
- **de qualidade**: proporção de execuções corretas, estimada pela avaliação online sobre uma amostra. Mede *se* respondeu certo.

Um SLO de qualidade só é conferível porque a suíte da aula de *evals* fornece um juiz validado; sem ele, a família de qualidade fica vazia e o plano deixa de cobrir a última linha da tabela da §4 — o modelo que conclui errado.

Os objetivos também amarram o plano ao negócio. O ganho prometido na Parte 1 do trabalho, com linha de base e denominador, é um SLO declarado antes da existência do sistema; a conferência da promessa, tratada na nota sobre a conta, é a medida desse SLO sobre execuções reais. Um ganho prometido que não se exprime como SLI mensurável é, pela mesma razão, uma promessa que não se pode conferir.

### 6. Sinais e cortes

Cada pergunta é respondida pelo sinal adequado à sua forma (nota sobre monitoração e telemetria, §4):

| Forma da pergunta | Sinal | Uso |
|---|---|---|
| quanto, com que frequência, em agregado | métrica | painel e alerta |
| o que aconteceu nesta execução, e em que ordem | *trace* | diagnóstico |
| com que conteúdo, em que detalhe | *log* ou evento | investigação pontual |
| estava certo | avaliação | qualidade |

Os **cortes** — as dimensões pelas quais uma métrica é decomposta — derivam do que **varia** no sistema. O critério é o mesmo da §2: um corte entra no plano se separa populações que exigem decisões diferentes. Num agente, quatro fontes de variação justificam corte: a ferramenta, o agente, o tipo de tarefa (as fatias da suíte de *evals*) e a versão.

A versão de um agente é um **carimbo composto**: código, prompt, modelo e índice podem mudar independentemente, e qualquer um deles altera o comportamento. O *prompt registry* visto em *prompt engineering* muda o prompt sem mudar o código; o provedor muda o modelo sem mudar o nome; a reindexação muda o índice sem mudar nenhum dos dois. Um corte por versão que registra só a versão do código não separa o antes do depois de nenhuma dessas mudanças.

Todo atributo de corte em métrica tem **domínio fechado** e pequeno, pelo custo de cardinalidade (nota sobre monitoração e telemetria, §7): o motivo de término é uma enumeração; o identificador da despesa vai no *span*, nunca na métrica.

A **matriz de rastreabilidade** registra o resultado: uma linha por pergunta, com o sinal que a responde, os atributos de que depende e o consumidor — painel, alerta ou consulta. Ela é lida nas duas direções. Uma pergunta sem sinal é uma lacuna de instrumentação; um sinal sem pergunta é custo sem uso, e sai.

### 7. Amostragem e retenção

As estratégias de amostragem estão descritas na nota sobre monitoração e telemetria (§10). O que o agente acrescenta é que o desfecho de uma execução — o motivo de término, o veredito — só é conhecido no **fim** dela. Uma amostragem na cabeça descarta, na mesma proporção, execuções normais e execuções que terminaram mal; por isso, a regra para agentes é guardar **todas** as execuções com término anormal, erro ou avaliação reprovada, e uma fração das normais, o que exige amostragem na cauda ou uma decisão tomada pela própria aplicação ao fim da execução.

O volume é estimado antes de escolher a fração:

$$V = N \cdot \bar{s} \cdot \bar{b}$$

com $N$ execuções por dia, $\bar{s}$ *spans* por execução e $\bar{b}$ bytes por *span*. O resultado é confrontado com o limite do plano da plataforma e com o custo de armazenamento.

A retenção tem dois limites opostos. O inferior vem do diagnóstico e da governança: guarda-se o suficiente para reconstruir uma decisão contestada, como trata a nota sobre provar a decisão, na aula de governança. O superior vem da lei: dado pessoal é guardado pelo tempo necessário à finalidade, e não mais.

### 8. A camada de qualidade

A família de qualidade da §5 depende de **rótulos** — a indicação, por execução, de que o resultado estava certo ou errado. Há duas fontes:

- **rótulos explícitos**: o juiz validado na aula de *evals*, executado sobre uma amostra das execuções, como descreve a nota sobre observabilidade (§7). Têm custo por execução avaliada, e valem enquanto a concordância medida na validação valer;
- **rótulos implícitos**: comportamentos que revelam avaliação sem que ninguém a tenha pedido — o aprovador que rejeita ou edita a proposta do agente na confirmação humana, o usuário que reformula a pergunta logo em seguida, a decisão contestada. Não custam nada além de registrá-los, mas são enviesados: só existem para as execuções que alguém viu.

A camada de qualidade fecha o ciclo da aula de *evals*: cada falha encontrada, por qualquer das fontes, volta ao *golden dataset* como caso, com a origem declarada.

### 9. Da observação à ação

Um **alerta** é uma condição sobre um sinal que, satisfeita, notifica uma pessoa para que ela aja. A recomendação de alertar por sintoma, e não por causa, está na nota sobre monitoração e telemetria (§1). O plano acrescenta duas exigências.

A primeira é que **todo alerta declara destinatário e procedimento**: quem é notificado e o que faz ao ser notificado. Um alerta sem procedimento produz notificação sem ação; repetido, ensina a ignorar todos os alertas, inclusive os legítimos.

A segunda é que o limiar deriva do SLO. Em lugar de um limiar arbitrário sobre a métrica, alerta-se quando o **orçamento de erro** está sendo consumido depressa demais para durar até o fim da janela — a taxa de consumo (*burn rate*) (BEYER et al., 2018). Para o SLO de término normal do auditor, o alerta não dispara a cada execução por orçamento esgotado, e sim quando a proporção delas, mantida, esgotaria o orçamento do mês em poucos dias.

O **painel** tem papel distinto: não avisa, e sim permite diagnosticar, descendo pelos cortes da §6 — da métrica agregada para a ferramenta, a versão ou a fatia em que a piora se concentra.

### 10. O que fica fora

O plano declara também o que **não** é emitido. Prompts e respostas carregam o conteúdo do pedido, e com ele dado pessoal; o padrão é mantê-los fora dos *spans*. A plataforma de observabilidade é mais um armazenamento, com outro controle de acesso e outra retenção, e cada atributo emitido é uma cópia a mais do dado — a mesma pergunta que a aula de segurança e a de memória fazem sobre o que o sistema guarda.

Quando a camada de qualidade precisa do conteúdo, como o juiz precisa da resposta para julgá-la, o conteúdo vai para um armazenamento próprio, ligado ao *trace* pelo `trace_id`, com acesso e retenção declarados no plano. O *Collector* permite remover atributos sensíveis antes de a telemetria sair da rede (nota sobre monitoração e telemetria, §10).

### 11. O plano como hipótese

Cada linha da tabela de modos de falha afirma que uma falha será detectada por um sinal. A afirmação só se confirma quando a falha é **provocada** e o sinal a registra; até lá, o plano é hipótese. O caso mais informativo é o do modo de falha silencioso que, provocado, não move nenhum sinal: ele aponta a lacuna do plano antes que o usuário a encontre.

O plano também envelhece com a arquitetura. Um segundo agente cria fronteiras de processo novas e modos de falha novos — o subordinado que devolve resultado errado ao coordenador, a delegação em laço —, e o corte por agente, até então trivial, passa a separar populações distintas. O plano é revisto a cada mudança de arquitetura, como o *golden dataset* é revisto a cada mudança de domínio.

---

## Exemplos

### Exemplo 1 — O plano do auditor de despesas, condensado

```text
  PERGUNTAS (interessado → decisão)
    negócio    a conferência ficou mais rápida?      → manter ou rever o case
    operação   quantas análises saem sem veredito?   → investigar a versão
    operação   qual integração está falhando?        → corrigir a ferramenta
    custo      quanto custa uma análise?             → aplicar alavanca
    governança alguma análise usou dado de outra?    → revisar permissões

  UNIDADE DE TRABALHO   uma análise de despesa = um trace (invoke_agent)

  MODOS DE FALHA → SINAL
    orçamento esgotado        → agente.termino = orcamento_esgotado
    resposta sem veredito     → agente.veredito = ausente
    regulamento indisponível  → execute_tool com status ERROR, por ferramenta
    conclusão errada          → acerto amostrado (juiz da aula de evals)
    histórico sem compressão  → gen_ai.usage.input_tokens por execução

  SLOs (janela de 28 dias)
    p95 da execução ≤ 30 s
    ≥ 95% das execuções com término normal e veredito
    acerto amostrado ≥ 90%
    custo médio ≤ R$ 0,05 por análise
```

As quatro métricas de operação da nota sobre observabilidade aparecem aqui como derivação: a latência p95 e o custo vêm de SLOs; a distribuição dos motivos de término e a taxa de erro por ferramenta vêm de modos de falha. O veredito, que aquela nota não lista entre as quatro, entra pelo modo *resposta sem veredito*, e é o motivo de o `agente.py` gravá-lo.

### Exemplo 2 — A matriz de rastreabilidade do auditor

| Pergunta | Sinal | Atributos | Consumidor |
|---|---|---|---|
| quantas análises saem sem veredito? | contagem de `invoke_agent` | `agente.veredito`, `service.version` | alerta de taxa de consumo |
| a piora começou com qual mudança? | métricas por execução | versão do código, do prompt, do modelo e do índice | painel, corte por versão |
| qual integração está falhando? | contagem de `execute_tool` com `ERROR` | `gen_ai.tool.name` | painel, corte por ferramenta |
| quanto custa uma análise? | soma de tokens por `trace` × preço | `gen_ai.usage.*`, `gen_ai.request.model` | painel e alerta de custo |
| a análise estava certa? | avaliação de uma amostra | `trace_id`, versão da rubrica | painel, ao lado das métricas |
| alguma análise usou dado de outra? | argumentos de `execute_tool` | identificador da despesa pedida e da consultada, no *span* | consulta sob demanda |

A última linha não tem painel nem alerta: é uma pergunta de governança, feita quando uma decisão é contestada. Mesmo assim, entra no plano, porque o atributo precisa estar no *span* antes da contestação.

### Exemplo 3 — Outra arquitetura, outra lista

Um assistente de atendimento que consulta uma base de conhecimento por RAG e propõe respostas que um atendente aprova antes do envio produz, pelo mesmo método, uma lista diferente:

```text
  UNIDADE DE TRABALHO   o caso, com pausa para aprovação
                        → dois traces ligados por span link

  MODOS DE FALHA PRÓPRIOS → SINAL
    recuperação vazia            → proporção de consultas com 0 trechos
    trecho de versão antiga      → versão do índice no span de recuperação
    proposta rejeitada           → taxa de rejeição pelo atendente (rótulo implícito)
    proposta editada             → taxa de edição, por fatia de assunto
    atendente demora a aprovar   → tempo de espera, FORA da latência do agente
```

A taxa de rejeição pelo atendente substitui, em boa parte, o juiz amostrado: é um rótulo implícito disponível em todas as execuções, e não numa amostra. A latência do agente e o tempo de espera humana são métricas separadas — somadas, o p95 mediria o atendente.

---

## Fontes e leituras

BEYER, B.; JONES, C.; PETOFF, J.; MURPHY, N. R. (org.). **Site reliability engineering**: how Google runs production systems. Sebastopol: O'Reilly Media, 2016. Cap. 4: Service level objectives; cap. 6: Monitoring distributed systems.

BEYER, B.; MURPHY, N. R.; RENSIN, D. K.; KAWAHARA, K.; THORNE, S. (org.). **The site reliability workbook**: practical ways to implement SRE. Sebastopol: O'Reilly Media, 2018. Cap. 2: Implementing SLOs; cap. 5: Alerting on SLOs.

INTERNATIONAL ELECTROTECHNICAL COMMISSION. **IEC 60812**: failure modes and effects analysis (FMEA and FMECA). 3. ed. Genebra: IEC, 2018.

MAJORS, C.; FONG-JONES, L.; MIRANDA, G. **Observability engineering**: achieving production excellence. Sebastopol: O'Reilly Media, 2022.

OPENTELEMETRY. **Semantic conventions for generative AI systems**. Disponível em: https://opentelemetry.io/docs/specs/semconv/gen-ai/.

OPENTELEMETRY. **Traces**: span links. Disponível em: https://opentelemetry.io/docs/concepts/signals/traces/.

**Material da disciplina.** Nesta aula, [Monitoração, telemetria e OpenTelemetry](01-monitoracao-e-telemetria.md) — SLI, SLO, alerta por sintoma, cardinalidade e amostragem; [Observabilidade](03-observabilidade.md) — a implementação do plano do auditor; [A conta, e o que quebra](04-a-conta-e-o-que-quebra.md) — a conferência da promessa como medida de um SLO. [Confiabilidade](../../aula-05-arquitetura-de-agentes/notas-de-aula/02-confiabilidade.md), em arquitetura de agentes — o término modelado como dado. [O prompt como código](../../aula-03-prompt-engineering/notas-de-aula/03-prompt-como-codigo.md), em *prompt engineering* — o carimbo de versão. [O juiz, e a suíte](../../aula-11-avaliacao-agentes/notas-de-aula/03-o-juiz-e-a-suite.md), em *evals* — o juiz validado e o *golden dataset*. [Provar a decisão](../../aula-12-governanca/notas-de-aula/02-provar-a-decisao.md), em governança — o *trace* como evidência e o que guardar.
