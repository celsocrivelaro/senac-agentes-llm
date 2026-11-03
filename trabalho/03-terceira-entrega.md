# Trabalho da disciplina — Parte 3: O sistema sob controle

> Leiam antes a **[visão geral](00-visao-geral.md)**, a **[Parte 1](01-primeira-entrega.md)** e a **[Parte 2](02-segunda-entrega.md)**: esta é a última das três entregas sobre o **mesmo** sistema, no **mesmo** repositório.

## Contexto

A Parte 1 avaliou **a escolha** do problema. A Parte 2 avaliou **a construção** do agente: o grafo, o estado, o RAG, a memória e o ponto em que ele para e espera uma pessoa. Esta parte avalia o que separa um protótipo de um sistema que outra pessoa aceitaria pôr em uso:

> **Como vocês sabem que o agente funciona, o que ele está autorizado a fazer, o que ele faz quando a entrada é hostil, e como se vê o que ele fez?**

São quatro frentes obrigatórias, e cada uma responde a uma parte dessa pergunta:

| Frente | A pergunta que ela responde |
|---|---|
| **Avaliação** | o agente funciona, e uma mudança melhorou ou piorou? |
| **Governança** | o que o agente está autorizado a fazer, onde esse limite é aplicado, e como se prova o que ele decidiu? |
| **Segurança** | o que o agente faz quando a entrada é hostil? |
| **Observabilidade** | o que se vê do agente enquanto ele roda? |

E três frentes **opcionais** — multiagente, MCP e multimídia. Nenhuma é exigida, e não usá-las não tira nota. Quem as usar documenta como estão construídas (§5).

**O foco desta entrega é implementação e documentação.** Cada frente pede código que roda e um documento em `docs/` que explica o que o código faz. Um documento que descreve um controle que não existe no código vale menos que nenhum documento, porque afirma o que ninguém verificou.

---

## O que entregar

0. **O sistema da Parte 2, rodando** — com as correções que a Parte 2 apontou
1. **Avaliação** — a suíte implementada, e o que ela mede
2. **Governança** — o *model card*, a autoridade e a decisão reconstruída a partir do *trace*
3. **Segurança** — os riscos mapeados, os controles implementados e os dados pessoais
4. **Observabilidade** — o plano do case e a instrumentação que o cumpre
5. **As frentes opcionais**, se usadas — multiagente, MCP, multimídia
6. **O projeto como entrega**

> **Comecem pelo §1.** A suíte é o instrumento das outras frentes: é com ela que se decide se uma mudança sobe, e é dela que saem as execuções que o §4 observa.

---

## 0. O sistema da Parte 2, rodando

O ponto de partida é o sistema entregue na Parte 2: o grafo, o estado, o RAG, a memória e o *human-in-the-loop*. Nada disso é reescrito aqui.

Se a correção da Parte 2 apontou defeitos, corrijam-nos e registrem em `docs/mudancas.md`: o que estava errado, o que mudou, e o commit da correção. Esse arquivo é também o **histórico de mudanças** que o *model card* do §2 cobra.

---

## 1. Avaliação

A pergunta da frente é a que a visão geral formulou: *como vocês sabem que uma mudança melhorou em vez de piorar?* A resposta é uma suíte que roda com um comando e devolve um número por fatia.

O código fica em `evals/`; a explicação, em `docs/evals.md`.

### 1.1 O *golden dataset*

No mínimo **vinte casos**, em arquivo versionado (JSON, JSONL ou CSV), cada um com:

- a **entrada** e o **esperado** — a saída exata, um rótulo, ou o critério que a resposta deve satisfazer;
- **`origem`**: `producao`, `desenvolvimento` ou `sintetico`;
- **`classe`**: o agrupamento que permite ler o resultado por fatia;
- **`por_que_existe`**: o que este caso pega que os outros não pegam.

Exigências de composição, que valem mais que o tamanho:

- **casos de recusa** — perguntas fora do escopo ou sem resposta na base;
- **ao menos três classes**, com mais de um caso em cada;
- **sintéticos revisados**, com quem revisou declarado.

Os casos demonstrados no §9 da Parte 2 entram neste conjunto.

### 1.2 Os níveis de avaliação

Para cada propriedade que o sistema precisa cumprir, o nível em que ela é avaliada:

| Propriedade | Nível | Como se confere | Por que não um nível mais barato |
|---|---|---|---|

Os quatro níveis, do mais barato ao mais caro: **determinístico** (igualdade, expressão regular, validação de *schema*), **trajetória** (as ferramentas chamadas, em que ordem, quantas vezes, lidas no *trace*), **resultado** (o estado do mundo antes e depois da execução) e **juiz LLM**.

Toda propriedade avaliada por juiz justifica, na última coluna, por que um nível mais barato não resolve.

**Se o sistema age sobre o mundo**, a avaliação de **resultado** é obrigatória: ela captura o agente que relata ter feito o que não fez, e o que fez duas vezes.

### 1.3 O juiz, validado

Um juiz para **uma** propriedade sem gabarito: rubrica escrita, saída estruturada, critério **binário**, um exemplo de cada lado.

A validação:

- **vinte saídas** rotuladas à mão, com uma nota escrita por caso;
- o **limiar de adoção**, declarado antes de medir;
- a **matriz de concordância** juiz × humano, e não só o percentual;
- os **desacordos lidos um a um**, cada um classificado como defeito do juiz, da rubrica ou do rótulo humano.

### 1.4 A suíte rodando, e o resultado

A suíte roda **o sistema entregue**, pelo mesmo ponto de entrada que o `README` documenta, e não uma cópia simplificada dele. A organização do código é do grupo; o que se pede são dois comandos, documentados no `README` — por exemplo:

```bash
python -m evals.rodar --versao 1.4.0
python -m evals.comparar evals/resultados/<rodada-A> evals/resultados/<rodada-B>
```

O primeiro roda o conjunto e grava o resultado da rodada. O segundo compara duas rodadas e imprime:

```
DATASET v<x.y.z>   <n> casos   (producao <n> · desenvolvimento <n> · sintetico <n>)
JUIZ rubrica v<x.y.z>   concordância <n>%   limiar <n>%

classe            versão A   versão B   delta
<classe>            <n>%       <n>%     <+/-n>
MÉDIA               <n>%       <n>%     <+/-n>
```

**Os resultados das rodadas são commitados.** Eles são a evidência dos números de `docs/evals.md` e do campo Avaliação do *model card*: todo número citado aponta para a rodada que o produziu. Duas rodadas, no mínimo: a **versão anterior** e a **versão nova**, com uma mudança que o grupo de fato fez nesta entrega — como um dos controles do §3.2 —, e o que a comparação mostrou.

---

## 2. Governança

A governança responde a duas perguntas diferentes: **o que o agente pode fazer**, e **como provar a um terceiro o que ele fez** numa execução específica. A primeira se especifica aqui e se implementa nos controles do §3.2; a segunda se prova pelo *trace* do §4.

O documento central é `docs/model-card.md`.

### 2.1 O *model card*

Os doze campos da [nota sobre provar a decisão](../aula-12-governanca/notas-de-aula/02-provar-a-decisao.md) (§4): identificação, finalidade, autoridade, ferramentas e integrações, modelo e provedor, dados, avaliação, limites conhecidos, riscos aceitos, supervisão humana, registro e retenção, histórico de mudanças.

Cada campo traz **a afirmação** — o que o sistema é ou faz. Em quatro campos — **Autoridade**, **Avaliação**, **Riscos aceitos** e **Registro e retenção** — a afirmação vem com **a evidência**: o caminho no repositório que a sustenta (arquivo e linha, teste, *trace* de uma execução, saída de comando), ou a marca **LACUNA** com o que falta para sustentá-la.

O campo **Avaliação** aponta para os números do §1. O campo **Riscos aceitos**, para o §3.3. Uma seção final, **Lacunas**, reúne todas as marcas do documento, cada uma com a correção proposta.

### 2.2 A autoridade, implementada

Especifiquem a autoridade do agente como a [nota sobre governar a ação](../aula-12-governanca/notas-de-aula/01-governar-a-acao.md) (§8) faz com o auditor de despesas: escopo, ações proibidas, credencial, limite por execução, limiar de aprovação humana, comportamento quando um controle falha, e quem pode suspender o agente.

Para cada linha, **onde o limite é aplicado**:

| Onde | O que vale como evidência |
|---|---|
| ausência de ferramenta | o agente não tem a ferramenta que executaria a ação |
| código | a linha que bloqueia, **e um teste que a exercita** |
| credencial | o privilégio da chave ou do usuário de serviço, e o que ela não alcança |
| plataforma | a configuração (limite de gasto do provedor, permissão do banco) |
| prompt | a instrução no *prompt* de sistema |

Limite aplicado só no prompt é declarado como tal e conta como lacuna: nenhum controle fora do modelo o garante. Os limites aplicados em código entram nos controles do §3.2, com teste.

### 2.3 A decisão reconstruída a partir do *trace*

O registro de cada execução é o *trace* do §4.2 — não há um log separado. Para que alguém de fora responda *o que o agente decidiu, com base em quê, e sob qual versão?*, o *trace* guarda, nos atributos ou eventos dos *spans*, no mínimo:

- a entrada;
- o contexto recuperado (o identificador dos trechos, e não necessariamente o texto inteiro);
- as versões: do prompt, do modelo, do sistema;
- as ferramentas chamadas, com argumentos e resultado;
- a saída e o motivo de término;
- a aprovação humana, quando houve: quem, quando, e o que foi aprovado.

**A demonstração**: escolham uma decisão de uma execução passada e, **só a partir do *trace***, reconstruam por que o agente decidiu aquilo. O texto da reconstrução vai em `docs/model-card.md`, no campo Registro e retenção.

O *trace* também tem prazo e acesso: declarem **por quanto tempo** ele é guardado e **quem** pode lê-lo. Um *trace* que guarda dado pessoal herda a classificação desse dado.

---

## 3. Segurança

> ### Regra de escopo
>
> Os ataques desta frente valem **exclusivamente contra o sistema do próprio grupo, em ambiente local**. Aplicar o mesmo procedimento a sistema de terceiro sem autorização é ilícito, e este trabalho não o cobre em hipótese alguma.

A pergunta da frente é a da visão geral: *o que o sistema faria se a entrada fosse hostil?* O documento é `docs/seguranca.md`.

### 3.1 Os riscos do case

O catálogo do OWASP para aplicações agênticas lista dez riscos. A lista abaixo é um **repertório para reconhecer**, e não uma lista a preencher inteira: um agente sem memória não sofre envenenamento de memória, e um agente que só lê não move dinheiro.

| | Risco |
|---|---|
| ASI01 | Sequestro de objetivo do agente |
| ASI02 | Uso indevido e exploração de ferramenta |
| ASI03 | Abuso de identidade e privilégio |
| ASI04 | Cadeia de suprimentos agêntica |
| ASI05 | Execução inesperada de código |
| ASI06 | Envenenamento de memória e de contexto |
| ASI07 | Comunicação insegura entre agentes |
| ASI08 | Falhas em cascata |
| ASI09 | Exploração da confiança humano-agente |
| ASI10 | Agentes desgovernados |

Entreguem **os riscos que existem no sistema de vocês** — aqueles cuja capacidade correspondente está no código. Para cada um:

| Campo | O que se escreve |
|---|---|
| **Onde** | o componente, com **arquivo e linha** |
| **Cenário de ataque** | concreto: quem é o atacante, por qual canal o texto ou a ação dele entra, e o que o agente faria |
| **Impacto** | de 1 a 3, pelos critérios do exercício da Aula 14 |
| **Mitigação** | a medida implementada, e onde ela entra no sistema — ou a referência ao risco aceito do §3.3 |

Antes dos quadros, listem os **canais por onde conteúdo não confiável entra** no sistema: a entrada do usuário, o documento recuperado, o retorno de ferramenta, a memória — e, se houver, a mídia do §5.3 e o agente subordinado do §5.1. Os canais são o ponto de partida: cada um costuma levar a mais de um risco, e um risco que nenhum canal alcança dificilmente existe no case.

Ordenem os quadros por impacto. Os dois primeiros são os que o §3.2 mitiga.

### 3.2 Os controles, implementados

Ao menos **três controles aplicados fora do prompt** — no código, na credencial ou na arquitetura —, cada um com um teste automatizado que tenta a ação indevida e verifica que foi bloqueada. **Dois deles mitigam os dois riscos de maior impacto** do §3.1; o terceiro pode mitigar outro risco ou aplicar um limite da autoridade do §2.2.

Valem as medidas vistas nas Aulas 12 e 14: menor privilégio por ferramenta, limite por execução, confirmação humana que o prompt não contorna, minimização de contexto, validação da saída do modelo antes de usá-la, um dos seis padrões de defesa da [nota sobre defesa por arquitetura](../aula-14-mlops-seguranca/notas-de-aula/02-defesa-por-arquitetura.md) (§2).

**Instrução no *prompt* de sistema não é controle**: o texto injetado chega pelo mesmo canal.

Para cada controle, documentem:

- **o que ele impede**, e o teste que o demonstra;
- **o que o sistema deixou de conseguir fazer** por causa dele — a lista, explícita. Uma defesa apresentada sem essa lista parece gratuita, e não é;
- se ele bloqueia ação, se é ***fail-closed*** (na falha do controle, a ação não acontece) ou ***fail-open*** (acontece, com registro), e por quê.

### 3.3 O risco aceito

Ao menos um risco que o grupo **decidiu não mitigar**, declarado: qual é, por que foi aceito, o que o limita hoje, e que evento faria reavaliá-lo. Ele aparece também no campo Riscos aceitos do *model card*.

### 3.4 Os dados pessoais e o que a memória não guarda

Mapeiem onde o dado pessoal **entra**, onde **fica** e para onde **vai**: prompt, índice, memória, *trace* e provedor externo. Para cada lugar: o dado, a finalidade, a base legal (LGPD, art. 7º) e o prazo de guarda. Se o case não trata dado pessoal, digam isso e digam por quê — é uma propriedade do sistema, e o *model card* a registra no campo Dados.

A Parte 2 decidiu o que a memória guarda. Esta entrega declara, **com o ponto do código que impede a gravação**, o que ela não guarda:

- **dado sensível** — identificador pessoal, credencial, valor que não precisa persistir;
- **conteúdo vindo de fora sem verificação** — uma memória que aceita conteúdo não verificado é injeção indireta com persistência.

Se o case não tem memória de longo prazo, esta parte se reduz a dizer isso.

---

## 4. Observabilidade

A frente tem duas peças: **o plano**, que declara o que se precisa saber sobre o sistema enquanto ele roda, e **a instrumentação**, que produz os sinais que o plano pede. O plano vem antes, porque é ele que decide o que instrumentar.

### 4.1 O plano de observabilidade

Em `docs/observabilidade.md`, o plano **do seu case**, com as seções do [modelo da Aula 15](../aula-15-mlops-observabilidade/exercicios/modelo-plano-de-observabilidade.md):

1. **As perguntas** — quatro, uma por interessado (o negócio, a operação, quem paga, quem responde pelo sistema), cada uma com a decisão que a resposta muda;
2. **O *trace* de uma execução** — a árvore de *spans* de uma execução típica e os atributos próprios da raiz;
3. **Os modos de falha** — ao menos quatro, **duas silenciosas**, cada um com o sinal que o torna visível;
4. **O objetivo** — o SLO de qualidade, detalhado no §4.4.

### 4.2 A instrumentação

O sistema emite *traces* segundo as **convenções semânticas GenAI do OpenTelemetry**:

| *Span* | Quando é aberto |
|---|---|
| `invoke_agent <agente>` | uma vez por execução, como raiz |
| `chat <modelo>` | a cada chamada ao modelo, com modelo e tokens de entrada e saída |
| `execute_tool <ferramenta>` | a cada ferramenta executada |

Na raiz, os atributos próprios que o plano declarou. Dois são obrigatórios:

- **o motivo de término** — a taxonomia da Aula 05 (respondeu, orçamento esgotado, aguardando humano, erro, interrompido…). Ele não está na convenção, e é o que separa a execução que terminou bem da que parou no meio;
- **a versão do sistema** — sem ela, nenhum número se compara entre versões.

O *trace* é também o registro da decisão: os atributos que o §2.3 pede entram aqui.

Os *traces* são exportados para um **destino consultável** — uma plataforma de observabilidade, um *collector* local com interface, ou arquivo — e o `README` diz como vê-los.

### 4.3 As métricas, sobre execuções reais

Sobre ao menos **trinta execuções** — as da suíte do §1 valem —, reportem:

- **latência p50 e p95**, e não a média;
- **custo por execução**;
- **a distribuição dos motivos de término**;
- **a taxa de erro por ferramenta**.

Com **corte por versão**: um número que mistura versões soma sistemas diferentes.

E respondam, com esses números, **as quatro perguntas do §4.1**. Uma pergunta que os sinais implementados não respondem é lacuna do plano ou da instrumentação, e se declara como tal.

### 4.4 O objetivo: SLI e SLO

Dois termos, um apoiado no outro:

- **SLI** (*service level indicator*) — a **medida**: uma razão entre eventos bons e eventos válidos, ou um percentil, calculada a partir dos sinais do §4.2. Ex.: *fração das execuções com término `respondeu`*;
- **SLO** (*service level objective*) — o **alvo interno** para um SLI, numa janela. Ex.: *≥ 90% das execuções terminam com `respondeu`, em 7 dias*. A diferença entre o alvo e 100% é o **orçamento de erro**: quanto o sistema pode falhar na janela sem violar o objetivo.

Em `docs/observabilidade.md`, **um SLO de qualidade**, com o SLI definido com exatidão — que eventos contam como bons, quais como válidos, e de que atributo do *trace* eles saem —, o objetivo, a janela, o valor medido nas ≥30 execuções e o orçamento de erro consumido.

É o SLO de qualidade que cobre a **degradação silenciosa**: execuções que terminam por orçamento esgotado, sem erro, com resposta pior. Elas não movem a taxa de erro, e só um objetivo sobre o motivo de término as torna visíveis.

Se o SLO estiver violado nas execuções medidas, digam isso e o que fariam: um objetivo violado e reportado é resultado; um objetivo ajustado ao valor medido, para passar, não é objetivo.

---

## 5. As frentes opcionais

As três frentes abaixo **não são obrigatórias**, e não usá-las não tira nota. Quem as usar documenta como estão construídas, em um arquivo próprio em `docs/` — e o sistema que elas produzem é o que as frentes obrigatórias avaliam, protegem e observam.

### 5.1 Multiagente

Se o sistema tem mais de um agente, `docs/multiagente.md` explica **a arquitetura**:

- **o diagrama** — os agentes, quem chama quem, e onde estão as ferramentas de cada um;
- **a topologia** — supervisor, *swarm*, *pipeline* ou agente-como-ferramenta —, e **o que no case a justifica**: o critério da Aula 16 que se aplica (isolamento de contexto, paralelismo, especialização), com número. Uma *pipeline* de caminho fixo é *workflow*, e se chama assim;
- **o contrato de *handoff***, como *schema* no código: objetivo, decisões e razões, escritas executadas com o identificador que cada uma devolveu, erros cometidos, fatos numéricos e a **chave de idempotência gerada pelo coordenador**;
- **o que acontece quando um agente falha** no meio da execução: o que fica escrito no mundo, e quem decide abortar.

E as frentes obrigatórias passam a valer para cada agente: um `invoke_agent` por agente no *trace* (§4.2), a avaliação de trajetória cobrindo os *handoffs* (§1.2), e ASI07 e ASI08 avaliados no §3.1.

### 5.2 MCP

Se o sistema usa MCP, `docs/mcp.md` explica **como ele é usado**:

- **os servidores** — os próprios e os de terceiros, cada um com a versão;
- **o que cada servidor expõe**, separando **ferramenta** (a operação que o modelo decide chamar) de **recurso** (o conteúdo que a aplicação decide incluir no contexto), com a razão de cada escolha;
- **a descrição de cada ferramenta** — o que ela faz, o formato dos argumentos, o que ela não faz;
- **como subir e conectar**: o comando que sobe o servidor, o transporte, e qual cliente o agente usa;
- **quem consome** cada servidor.

A descrição de ferramenta de terceiro é texto que entra no contexto do modelo sem que nenhum cliente o inspecione: o servidor entra no §3.1 como ASI04, cadeia de suprimentos.

### 5.3 Multimídia

Se o sistema recebe ou produz imagem, áudio, documento escaneado ou vídeo, `docs/multimidia.md` explica **a entrada e a saída**:

- **a entrada** — que mídia entra, por onde, em que formato e com que limite de tamanho;
- **onde ela vira texto**, se vira — modelo multimodal nativo, ou *pipeline* (OCR, transcrição, legenda) antes do modelo de texto —, e o modelo ou serviço usado;
- **a saída** — se o sistema produz mídia, que mídia, por qual ferramenta ou modelo, e em que formato chega ao usuário;
- **o custo** — os tokens ou o preço de uma entrada típica.

A mídia é também um canal de entrada: entra na lista de canais do §3.1, e o *golden dataset* do §1 inclui casos com mídia.

---

## 6. O projeto como entrega

**`README.md`** — **como rodar** (do zero, por quem nunca viu o projeto) e **como usar**, com um exemplo de entrada e saída copiado de uma execução real. E, nesta entrega, dois comandos a mais:

- como **rodar a suíte** e **comparar duas rodadas** (§1.4), e os testes dos controles do §3.2;
- como **ver os *traces*** do §4.

**`requirements.txt`** com todas as versões fixadas — inclusive as do OpenTelemetry e, se houver, do SDK de MCP.

**`.env.example`** com os nomes das variáveis e nenhum valor. Chave de API **nunca** no repositório, nem no histórico:

```bash
git log --all -- .env
git log --all -S "OPENAI_API_KEY=" --oneline
```

A saída dos dois comandos, vazia, vai em `docs/seguranca.md`. Se não vier vazia, a correção é **revogar a chave** no provedor — reescrever o histórico não alcança os clones existentes. Digam o que fizeram.

**`docs/`** — toda a documentação, em Markdown, versionada.

---

## Entrega

No repositório do grupo — **o mesmo desde a Parte 1**:

```
README.md          como rodar · como usar · como rodar a suíte
                   como ver os traces
requirements.txt   tudo fixado
.env.example       os nomes das variáveis, sem nenhum valor

docs/
  (os documentos das Partes 1 e 2)
  mudancas.md        o §0: o que mudou desde a Parte 2, com commit
  evals.md           o §1: dataset, níveis, juiz e matriz,
                     o relatório da comparação
  model-card.md      o §2: os doze campos, a autoridade,
                     a decisão reconstruída do trace, as LACUNAS
  seguranca.md       o §3: canais, os riscos do case, os controles e o que custaram,
                     o RISCO ACEITO, os dados pessoais, o que a memória não guarda,
                     o histórico do .env verificado
  observabilidade.md o §4: o plano, as métricas por versão,
                     as respostas às quatro perguntas, o SLO
  multiagente.md     o §5.1, se houver
  mcp.md             o §5.2, se houver
  multimidia.md      o §5.3, se houver

src/               o sistema
  prompts/         os prompts, versionados
evals/             dataset · juiz · a suíte · os resultados das rodadas
  resultados/      uma pasta por rodada, commitada — a evidência dos números
tests/             os testes dos controles do §3.2
mcp/               o servidor, se houver
dados/             corpus e dados simulados
logs/              os traces citados nos documentos
```

Markdown, sempre — nada de `.docx` nem `.pdf`, para que o `git diff` funcione.

---

## Como esta parte será avaliada

Em ordem de peso:

| O que se avalia | O que se espera |
|---|---|
| **Segurança** | os riscos do case, com **arquivo e linha** e cenário concreto · três controles **fora do prompt**, com teste, dois deles nos riscos de maior impacto · **o que cada controle custou** · risco aceito · os dados pessoais e o que a memória não guarda |
| **Avaliação** | ≥20 casos com origem, classe e `por_que_existe` · casos de recusa · níveis com justificativa · juiz validado com matriz e desacordos lidos · a suíte rodando o sistema entregue · resultados commitados por rodada · uma comparação real entre duas versões |
| **Observabilidade** | o plano do case · *trace* nas convenções GenAI, com **motivo de término** e **versão** · as quatro métricas sobre ≥30 execuções, por versão · as perguntas do plano respondidas com os números · um SLO de qualidade com SLI exato e valor medido |
| **Governança** | *model card* com evidência nos quatro campos que a pedem, ou LACUNA · a autoridade com o ponto de aplicação de cada limite · uma decisão reconstruída a partir do *trace* |
| **Frentes opcionais** | quando usadas: a documentação explica a construção e o código corresponde a ela |
| **A entrega como projeto** | roda do zero · o `README` diz como usar, rodar a suíte e ver os *traces* · `docs/` completo · nenhuma chave no repositório **nem no histórico** |

E o que **não** conta: número de agentes, número de servidores, número de defesas, tamanho do painel, sofisticação visual.

**Resultados negativos medidos contam a favor**: um juiz que não atingiu o limiar e não foi adotado, com a matriz; um SLO violado nas execuções medidas, reportado como tal. O que perde nota é a afirmação sem evidência.

---

## Política de IA generativa

O uso de assistentes de IA na escrita do código e da documentação é permitido. O grupo responde por tudo o que entrega, e qualquer integrante deve conseguir explicar qualquer parte do sistema.

Um ponto específico desta entrega: **documentação gerada por IA a partir do código costuma afirmar mais do que o sistema faz**. Um *model card* que declara um controle inexistente, ou um mapa de riscos com mitigação que não está no código, é tratado como afirmação sem evidência. Confiram cada afirmação contra o código antes de entregar.

---

## Dicas

- **O §1 primeiro.** Sem a suíte, a comparação do §1.4 não tem instrumento, e o §4.3 não tem execuções para medir.
- **Escolham os controles pelo cenário do §3.1.** Um controle escolhido sem um cenário de ataque concreto costuma proteger o flanco errado.
- **Comecem o *model card* por uma execução real.** Abram o *trace* de uma execução e vejam o que ele guarda: isso responde metade dos campos e mostra o que falta.
- **"O modelo foi instruído a não fazer X" descreve o prompt, não o sistema.** Se é só isso que existe, o campo é lacuna.
- **Instrumentem cedo.** Com o *trace* ligado, a depuração das outras frentes fica mais rápida, e as trinta execuções do §4.3 se acumulam sozinhas.
- **Para achar falhas silenciosas**, percorram cada ferramenta e o modelo perguntando: *como isto pode devolver um resultado pior sem dar erro?*
- **Frente opcional é escolha do case, não da nota.** Multiagente, MCP e multimídia entram quando o problema pede — e, quando entram, multiplicam o que as frentes obrigatórias têm de cobrir.
