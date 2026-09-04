# IA Aplicada com LLMs — Aula 04B: Casos de uso de agentes — O Brasil, e a escolha do seu case

## Introdução

As três notas anteriores foram sobre o mundo. Esta é sobre o lugar onde você vai trabalhar — e sobre a decisão que você toma hoje.

Ela tem duas metades, e elas se conectam mais do que parece. A primeira é o retrato da adoção de agentes no Brasil, que é surpreendente nos dois sentidos: melhor do que se imagina em um número, e bem pior em outro. A segunda é o conjunto de critérios com que você vai escolher o case do seu projeto final — e vários deles saem, diretamente, do que o primeiro número revela.

> **Pré-requisitos:** as notas [01](01-o-mapa-da-adocao.md), [02](02-os-casos-que-funcionam.md) e [03](03-os-casos-que-falharam.md) desta aula. Da Aula 04, a [nota 01 §2](../../aula-04-arquitetura-de-agentes/notas-de-aula/01-padroes-de-arquitetura.md) (a pergunta do fluxograma) e a [nota 02 §1](../../aula-04-arquitetura-de-agentes/notas-de-aula/02-o-agente-e-o-estado.md) (as três condições).

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Descrever** a posição do Brasil na adoção de agentes, com o número e com a ressalva metodológica.
- **Explicar** a tensão entre adoção alta e governança imatura, e o que ela significa para a sua carreira.
- **Avaliar** um case candidato contra duas listas de critérios: **valor** e **viabilidade**.
- **Reconhecer** os quatro anti-padrões de escolha de case.
- **Preencher e defender** a ficha do seu case — em especial o campo do **verificador**.

---

## Parte 1 — O Brasil

### 1. O número que surpreende

A pesquisa *The AI Production Paradox*, publicada em 2026 pela Sinch com **2.527 executivos seniores** em dez países e seis setores, traz o seguinte:

> **76%** das empresas brasileiras mantêm agentes de IA em produção — acima da média global (**62%**) e dos Estados Unidos (**67%**).

Esse dado costuma causar estranheza em sala, e a estranheza vem de uma suposição errada: a de que o Brasil adota tecnologia depois. Em algumas categorias, adota antes. Meios de pagamento e serviços bancários digitais são o precedente óbvio, e a explicação é parecida — mercado grande, custo de mão de obra em serviço, concentração bancária e disposição a experimentar.

Outros números do mesmo levantamento e da imprensa setorial completam o quadro: cerca de **40%** das empresas brasileiras já investem em agentes e outras **33%** pretendem começar nos próximos doze meses; agentes e IA generativa aparecem na agenda estratégica de 2026 para pouco mais da **metade** dos executivos ouvidos; **66%** pretendem aumentar o investimento em mais de 25%.

---

### 2. O número que vem logo depois

Da mesma pesquisa:

> **80%** das organizações brasileiras já **interromperam ou reverteram** alguma implementação de IA.
> Em **39%** desses casos houve **vazamento de dados ou de informações pessoais**.

E o recorte regional: na América Latina, a taxa de reversão associada a vazamento é de **41%**, contra **26%** na América do Norte.

Junte os dois números da mesma pesquisa e você tem a tese desta nota:

> **Adoção alta com governança imatura.**

Não é um paradoxo, apesar do nome do estudo. É o resultado esperado de colocar em produção mais rápido do que se aprende a controlar. E é literalmente o assunto da Aula 04: as reversões acontecem onde faltou a engenharia em volta do modelo — orçamento, salvaguarda, restrição de acesso, registro do que aconteceu.

Repare que a principal causa citada não é "o modelo alucinou". É **vazamento de dado**. O que falhou não foi a inteligência: foi o controle sobre **o que entrou no contexto do agente e para onde a resposta foi**.

---

### 3. A ressalva — aplicada contra o próprio argumento

A nota 01 desta aula ensinou quatro perguntas. Seria desonesto não aplicá-las agora, justamente no dado que mais favorece a narrativa desta seção.

| Pergunta | Resposta |
|---|---|
| **Quem pagou?** | Sinch, **fornecedora de plataforma de comunicação** — que vende para empresas que automatizam atendimento |
| **Quem respondeu?** | 2.527 executivos seniores, dez países, seis setores — amostra grande e bem distribuída |
| **O que foi perguntado?** | "manter agentes em produção" é uma definição elástica: um agente, em uma área, contando |
| **Medido ou declarado?** | **declarado** |

**O que fazer com isso.** Os dois números da pesquisa — 76% em produção e 80% de reversão — vêm da **mesma amostra, com o mesmo viés**. Isso é o que os torna comparáveis entre si, e é o que sustenta a tese: quem respondeu que tem agente em produção é, em larga medida, quem respondeu que já reverteu algum.

O que **não** dá para fazer é citar o 76% isolado, como manchete de pioneirismo. Foi assim que ele circulou; foi por isso que a ressalva está aqui.

---

### 4. O que isso significa para você

Um mercado com adoção alta e governança imatura tem uma consequência concreta de carreira, e ela é boa notícia para quem está terminando esta disciplina:

> **O Brasil precisa de gente que saiba consertar agente que já está rodando** — e não só de gente que saiba construir mais um.

Quem instrumenta, mede, limita e restringe acesso está resolvendo o problema que causou 80% das reversões. Quem só faz funcionar na demonstração está produzindo o próximo caso revertido.

Traduzido para o que você acabou de aprender: **orçamento, terminação registrada, erro classificado, detecção de laço, idempotência, restrição por arquitetura e trace** não são requintes acadêmicos. São a diferença entre os dois perfis.

---

### 5. LGPD: o que este bloco planta e não resolve

Se 39% das reversões brasileiras envolvem vazamento de dado pessoal, então o dado que entra na janela de contexto do agente é um problema **jurídico** antes de ser técnico.

Três fatos para guardar até a aula de segurança:

1. **Contexto é transferência.** Colocar um dado no prompt é enviá-lo para quem opera o modelo. Se o modelo é de terceiro, o dado saiu da sua organização — e isso é uma operação com dado pessoal, com base legal, finalidade e responsabilidade.
2. **Log é dado.** O trace que a Aula 04 mandou você gravar contém tudo o que passou pelo agente, e por isso está sujeito às mesmas regras que o dado original. Um sistema bem instrumentado cria uma nova superfície de exposição.
3. **Minimização é decisão de arquitetura.** A pergunta "o agente precisa mesmo deste campo para decidir?" é a mesma da §6 da nota 02 da Aula 04, aplicada a dado em vez de ferramenta.

Por isso, e é requisito da ficha: **todo case declara que dado sensível ele toca.** Se não toca nenhum, isso também se declara — e é uma vantagem do case, não uma omissão.

---

## Parte 2 — Escolher o seu case

### 6. As duas listas

Um case precisa passar nas **duas**. O erro clássico — e o mais caro, porque só aparece na semana 12 — é escolher pela primeira e descobrir a segunda tarde demais.

#### Valor — por que vale fazer

| Critério | Como saber que passou |
|---|---|
| **O problema é real** | você consegue nomear alguém que sofre com ele hoje |
| **O usuário é uma pessoa** | um cargo, não uma categoria. "O analista de suporte", não "as empresas" |
| **Cabe em uma frase** | se precisa de um parágrafo, o escopo está grande demais |
| **Existe um número de sucesso** | "funcionar bem" não é critério; "acertar a categoria em 80% de 40 chamados" é |

#### Viabilidade — por que dá para fazer neste semestre

| Critério | Como saber que passou |
|---|---|
| **Dados obtíveis ou simuláveis com honestidade** | ou você tem acesso, ou consegue inventar dados que preservem a **dificuldade** do problema |
| **Ferramentas implementáveis** | você consegue escrever as funções sem depender de um sistema a que não tem acesso |
| **Existe verificação** | você sabe dizer **como** julgar se a saída está certa |
| **Cabe no tempo** | o núcleo funciona em três semanas; o resto é refinamento |
| **Há espaço para o que ainda vem** | pelo menos dois entre RAG, memória, MCP e evals têm o que fazer aqui |

Sobre "simuláveis com honestidade", que é onde mais se erra: dado de mentira é legítimo — os laboratórios inteiros deste curso rodam sobre dados inventados. O que não é legítimo é dado de mentira **fácil demais**. Se todos os seus casos de teste são o caso feliz, você construiu um sistema que só funciona no seu conjunto de teste. Repare como os dados das aulas 03 e 04 foram montados: sempre com uma divergência, um registro inexistente e um caso que **não** deveria disparar a ação. Faça igual.

---

### 7. A pergunta de triagem

Antes das listas, a pergunta que vem da Aula 04 (nota 01 §2):

> **Você consegue desenhar o fluxograma do processo antes de rodar?**

Se consegue, o seu case é um **workflow** — e um workflow é, quase sempre, a engenharia certa. Mas é um **projeto final ruim para esta disciplina**, porque não exercita o que o curso ensina: você vai entregar um programa que chama o modelo três vezes em ordem fixa.

O seu case precisa de pelo menos **um ponto de decisão genuína** — um momento em que o próximo passo depende do que o passo anterior descobriu.

E o contrário também reprova: se **tudo** no seu case é decisão do modelo, você não vai conseguir testar nada, e vai passar o semestre depurando trajetória. O alvo é o meio da tabela do espectro de autonomia, e não a ponta.

---

### 8. Os quatro anti-padrões

Cada um com o sintoma pelo qual você o reconhece na sua própria ficha.

**1. O case sem verificador.**
*"Um agente que escreve posts criativos para redes sociais."*
**Sintoma:** você não consegue preencher o campo "como sei que está certo?" sem escrever "dá para ver que ficou bom".
**Por que reprova:** sem verificação, o laço não se corrige (Aula 04, nota 02 §1), você não consegue montar eval e o professor não consegue avaliar. É o anti-padrão mais comum e o mais fatal.
**Como consertar:** ou troque o domínio, ou **construa** o verificador — um conjunto rotulado, uma regra de negócio, uma checagem de formato.

**2. O case que depende de dado que você não tem.**
*"Um agente que analisa os chamados internos da empresa onde eu estagio."*
**Sintoma:** o plano contém a frase "vou pedir acesso".
**Por que reprova:** o acesso não vem, ou vem na semana 14. E dado corporativo real traz LGPD junto (§5).
**Como consertar:** simule o dado com fidelidade estrutural — mesmos campos, mesma bagunça, mesmos casos difíceis.

**3. O case grande demais.**
*"Um assistente que ajuda o aluno em tudo na faculdade."*
**Sintoma:** não cabe em uma frase; a lista de ferramentas passa de dez.
**Por que reprova:** você vai construir um pouco de tudo e nada inteiro, e não vai ter número nenhum para mostrar no fim.
**Como consertar:** escolha **uma** das coisas. A que tem verificador.

**4. O case que é produto de terceiro.**
*"Um clone do Cursor."* / *"Um ChatGPT da minha empresa."*
**Sintoma:** a maior parte do trabalho é interface, infraestrutura ou integração com editor.
**Por que reprova:** você passa o semestre reimplementando o que não é o assunto da disciplina, e a arquitetura de agente fica com 5% do esforço.
**Como consertar:** pegue **a parte agêntica** do produto que te interessa e faça só ela, em escala de laboratório.

---

### 9. Como usar o catálogo

Se você não tem ideia de por onde começar, o caminho mais rápido é o catálogo público do Google Cloud, com mais de mil casos reais organizados por **11 setores × 6 tipos de agente**.

O método, em quatro passos:

1. **escolha o setor** que te interessa — o que você já conhece por dentro vale mais que o que parece moderno;
2. **leia cinco casos** desse setor e identifique o **padrão da Aula 04** por trás de cada um (é o exercício da [nota 02](02-os-casos-que-funcionam.md));
3. **escolha um** e pergunte: *qual é a versão deste caso em escala de laboratório?* Mesmo problema, mil vezes menor, dados de mentira, uma ferramenta em vez de doze;
4. **passe essa versão pelas duas listas** da §6.

O passo 3 é o que separa uma boa proposta de uma fantasia. "Commerzbank atende 2 milhões de conversas" não é o seu projeto. "Um triador de 200 mensagens de suporte com quatro rotas, uma delas sem LLM, e acurácia medida contra 40 mensagens rotuladas à mão" é — e é o **mesmo problema**.

---

### 10. A ficha, campo a campo

Os campos que costumam ser preenchidos mal, e o que se espera de cada um:

**O problema em uma frase.** Se não couber, corte escopo até caber. O tamanho da frase é o teste.

**Quem sofre com ele hoje.** Um cargo, uma pessoa concreta. "As empresas" e "os usuários" não são ninguém, e um case sem usuário identificável não tem como ter critério de sucesso.

**Nível de autonomia pretendido — e por que este e não o de baixo.** A segunda metade é a que vale. A regra da Aula 04 é *use a menor autonomia que resolve*, então você precisa justificar **por que o nível anterior não resolvia**.

**As ferramentas, com leitura/escrita e reversibilidade.** Esta tabela é onde os riscos aparecem sozinhos: toda linha marcada "escrita, irreversível" é um ponto que vai precisar de confirmação humana e chave de idempotência (Aula 04, nota 03 §8).

**O verificador.** *"Como você vai saber que a saída está certa?"* — **este campo reprova mais que todos os outros juntos.** Respostas que valem: um conjunto rotulado à mão; uma regra de negócio que confere o resultado; um teste que passa ou falha; a comparação com uma decisão humana registrada. Resposta que não vale: "dá para ver".

**O critério de sucesso.** Um número, com denominador. "80% de acerto" não diz nada; "acerta a categoria em 32 de 40 chamados rotulados" diz.

**Que dado sensível este case toca.** Ver §5. "Nenhum" é uma resposta válida e vantajosa — desde que verdadeira.

**Espaço para o que ainda vem.** Marque pelo menos dois, e escreva **o quê**. "RAG: o regulamento em PDF, 40 páginas" é uma resposta; um `[x]` sem texto não é.

**O maior risco.** O risco de verdade, não o de fachada. "Pode ser que o modelo erre" não é risco, é a premissa. "A minha única fonte de dados é um PDF escaneado e o OCR pode não funcionar" é risco — e tem plano B.

---

### 11. O contrato do resto do semestre

Você está escolhendo com informação incompleta, e isso é deliberado. Ainda faltam RAG, memória, MCP, evals e produção. Escolher agora, e não na semana 15, muda a natureza do que vem: cada aula nova passa a terminar com a pergunta **"o que isto muda no seu case?"**.

A regra do ajuste, para que a informação nova possa ser usada sem que o projeto vire areia movediça:

> **O case pode ser refinado até a semana 8. Trocado, não.**

Refinar é mudar escopo, ferramenta, nível de autonomia ou fonte de dados. Trocar é começar de novo — e quem começa de novo na semana 9 entrega um protótipo.

E a ficha é **versionada**, como os prompts desde a Aula 03: cada mudança vira um commit com justificativa. A trilha de como o seu case mudou de ideia é parte da avaliação, porque é o registro do que você aprendeu.

---

## Exemplos

### Exemplo 1 — Um case ruim virando um case bom

**Versão 1 (reprova):** *"Um agente que ajuda o time de suporte da minha empresa."*

Diagnóstico: anti-padrão 3 (grande demais) + anti-padrão 2 (dado que não tem) + sem verificador. Três dos quatro.

**Versão 2 (passa):**

> *"Um triador de chamados de suporte que classifica categoria e urgência e decide se abre incidente."*

| Campo | Conteúdo |
|---|---|
| Usuário | o analista de plantão, que hoje lê 200 chamados por dia para achar os 10 urgentes |
| Autonomia | **roteador + agente** — o roteador resolve o chamado padrão por regra; o agente entra no ambíguo. Não é workflow porque o caso ambíguo exige consultar histórico e sistema antes de decidir |
| Ferramentas | `buscar_chamado` (leitura) · `consultar_historico` (leitura) · `abrir_incidente` (**escrita, irreversível → confirmação**) |
| **Verificador** | **200 chamados rotulados à mão** com categoria e urgência; acurácia por classe |
| Sucesso | acerta a categoria em ≥160/200 e **não perde nenhum urgente** (recall 100% na classe crítica) |
| Dado sensível | nome e e-mail do solicitante → anonimizados no conjunto simulado |
| Espaço | RAG (base de soluções conhecidas) · evals (os 200 rotulados **são** o golden dataset) |
| Risco | rotular 200 chamados à mão leva mais tempo que o previsto → começar por 60 e crescer |

Repare no critério de sucesso: **duas** métricas, e a segunda é assimétrica de propósito. Perder um chamado urgente custa muito mais que classificar mal um chamado banal — e essa assimetria é a terceira pergunta de triagem da Aula 04 (*o erro é caro?*) virando número.

### Exemplo 2 — O mesmo caso, em três níveis de autonomia

Vale ver como a escolha do nível muda o projeto inteiro:

| Nível | O que seria | Por que não / por que sim |
|---|---|---|
| **Workflow** | classificar → se urgente, abrir incidente | previsível e testável, mas **não exercita a disciplina** — dá para desenhar o fluxograma inteiro |
| **Roteador + agente** | regra resolve o padrão; agente investiga o ambíguo | **o alvo.** Tem decisão genuína no caminho estreito, e a maior parte do volume é barata |
| **Agente puro** | tudo pelo laço, o modelo decide sempre | mais caro, imprevisível, e a maioria dos chamados nem precisava de LLM |

A linha do meio é a resposta em quase todo case desta turma. Não por ser o meio-termo confortável — mas porque é o desenho que os casos reais da [nota 02](02-os-casos-que-funcionam.md) usam.

---

## Exercícios resolvidos

### 1. Consertar um case sem verificador

> *"Um agente que gera resumos das aulas a partir das transcrições."*

**Sem verificador.** Não existe resposta certa para um resumo, e "dá para ver que ficou bom" não é critério.

Três consertos possíveis, do mais fraco ao mais forte:

1. **Verificação por propriedade** — o resumo cita os termos técnicos que apareceram na aula? tem entre X e Y palavras? não introduz nome próprio que não estava na transcrição? É verificável por código, e é a técnica da suíte de regressão da Aula 03 (nota 03).
2. **Avaliador com critério escrito** — o padrão da Aula 04, nota 01 §4.5, com os itens definidos por você. Melhor que o anterior, e mais caro.
3. **Trocar a tarefa por uma verificável** — em vez de resumir, **responder perguntas sobre a aula**, com um conjunto de 30 perguntas e gabarito. Aí existe acurácia.

A terceira é a mais honesta: em vez de forçar um verificador sobre uma tarefa que não tem, escolha a tarefa vizinha que tem. E, de quebra, ela vira um caso de RAG.

### 2. Este case tem decisão genuína?

> *"Um sistema que recebe uma nota fiscal em PDF, extrai os campos, valida contra o pedido de compra e aprova ou recusa."*

**Não — e é um workflow bom.** Extrair → validar → decidir é uma sequência fixa, com número de passos conhecido. Você consegue desenhar o fluxograma antes de rodar. Como projeto de engenharia, é sólido; como projeto **desta disciplina**, não exercita quase nada do que o curso ensina.

**O que o torna um case:** acrescentar o caso em que a validação **não fecha**. Divergência de valor, item que não está no pedido, fornecedor com nome diferente. Aí o próximo passo depende do que foi descoberto — consultar o histórico do fornecedor? procurar um pedido alternativo? pedir esclarecimento? —, e o número de passos deixa de ser conhecido.

**O desenho final** é o mesmo do Exemplo 1: **regra para a maioria, agente para a divergência**. E note que ele é literalmente o exercício da Aula 04, num outro domínio — o que é uma boa notícia, porque significa que você já sabe fazê-lo.

---

## Síntese

**Sobre o Brasil:**

- **76%** das empresas brasileiras têm agentes em produção — acima da média global (62%) e dos EUA (67%).
- E **80%** já interromperam ou reverteram alguma implementação, com **39%** desses casos envolvendo vazamento de dado. Na América Latina, 41% contra 26% na América do Norte.
- A tese: **adoção alta com governança imatura**. A causa dominante não é alucinação — é controle sobre o dado.
- A pesquisa é **patrocinada por fornecedor** e **declarada**. Os dois números vêm da mesma amostra, e é isso que os torna comparáveis entre si.
- Consequência de carreira: o mercado precisa de quem saiba **consertar** agente em produção, e é exatamente o que a Aula 04 ensinou.
- LGPD: **contexto é transferência**, **log é dado**, **minimização é arquitetura**.

**Sobre a escolha:**

- Duas listas, e o case passa nas duas: **valor** (problema real, usuário concreto, uma frase, um número) e **viabilidade** (dados, ferramentas, **verificador**, tempo, espaço para o que vem).
- Triagem: se dá para desenhar o fluxograma antes de rodar, é workflow — e falta decisão genuína.
- Quatro anti-padrões: **sem verificador** · **dado que você não tem** · **grande demais** · **produto de terceiro**.
- O verificador é o campo que mais reprova. Verificador **construído** conta; nenhum não.
- Use o catálogo, mas o passo que importa é **traduzir o caso real para a escala de laboratório**.
- O case pode ser **refinado até a semana 8; trocado, não** — e a ficha é versionada.

---

## Fontes e leituras

- **Sinch — *The AI Production Paradox*** (2026, n=2.527 executivos, dez países) — [cobertura com os números do Brasil](https://olhardigital.com.br/2026/08/14/inteligencia-artificial/brasil-supera-eua-na-adocao-de-agentes-de-ia-pelas-empresas/). Leia junto com a ressalva da §3.
- [**Empresas brasileiras colocam agentes de IA entre prioridades para 2026**](https://canaltech.com.br/mercado/empresas-brasileiras-colocam-agentes-de-ia-entre-prioridades-para-2026/) — Canaltech; a agenda estratégica e os percentuais de investimento.
- **Google Cloud — catálogo de casos reais** ([11 setores × 6 tipos de agente](https://cloud.google.com/transform/101-real-world-generative-ai-use-cases-from-industry-leaders)) — o material de navegação da §9.
- **Lei nº 13.709/2018 (LGPD)** — leitura de referência para a §5; o tratamento é da aula de segurança.
- Aula 04, [nota 01 §2](../../aula-04-arquitetura-de-agentes/notas-de-aula/01-padroes-de-arquitetura.md) (a pergunta do fluxograma), [nota 02 §1 e §6](../../aula-04-arquitetura-de-agentes/notas-de-aula/02-o-agente-e-o-estado.md) (condições do agente e restrição por arquitetura) e [nota 03 §8](../../aula-04-arquitetura-de-agentes/notas-de-aula/03-confiabilidade.md) (idempotência).
- [nota 03](03-os-casos-que-falharam.md) desta aula — os três casos contra os quais você deve testar o seu.
