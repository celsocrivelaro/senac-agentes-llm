# Trabalho da disciplina — Parte 2: O agente de verdade

> Leiam antes a **[visão geral do trabalho](00-visao-geral.md)** e a **[Parte 1](01-primeira-entrega.md)**: esta é a segunda de três entregas sobre o **mesmo** sistema, no **mesmo** repositório.

## Contexto

Nesta entrega o agente vira um sistema completo: um **grafo** de nós e arestas, com estado declarado, contrato de entrada e saída, pontos onde para e espera uma pessoa, conhecimento de domínio consultável e memória do que já aconteceu.

Cada frente acrescenta uma dessas peças, e cada peça vem acompanhada da decisão de projeto que a explica: por que esta forma de recuperar, por que este corte, por que o grafo tem esta topologia, onde o sistema para e espera alguém.

---

## Esta entrega não começa do zero

Os exercícios das Aulas 06 a 08 foram feitos **no repositório deste trabalho**, sobre o **seu** case, e a Aula 09 trouxe o laboratório do framework. Eles são a construção; esta entrega é a **consolidação**.

| De onde vem | O que já está entregue | O que a Parte 2 acrescenta em cima |
|---|---|---|
| **06** — o buscador | conjunto de perguntas com resposta conhecida · as estratégias de corte comparadas · as quatro cegueiras no seu corpus | o corpus **do case**, curado e declarado · a decisão de corte defendida como decisão de projeto |
| **07** — a resposta que cita e recusa *(complementar)* | pipeline completo · citação verificada em código · o portão com limiar escolhido | a recuperação **integrada ao laço do agente**, e não um pipeline paralelo |
| **08** — a memória do seu agente | os dois níveis de memória · as três memórias nas estruturas certas · o que **não** entra · as três causas de esquecimento, com a cobertura da remoção | a **implementação** do que o projeto decidiu, e as medições que o sustentam |
| **08** — a memória do case *(complementar, de código)* | as três memórias nas estruturas certas · política de escrita · desempate por carimbo de tempo · esquecimento seletivo | a fronteira com o checkpoint **e com o RAG**, declarada no seu domínio |
| **09** — o laboratório do framework | os padrões da Aula 05 como grafo · estado com reducer · checkpointer e `interrupt` rodando | **o seu** grafo: os padrões que o seu sistema usa, o estado do seu domínio, e o ponto de parada no seu caso |

**Um grupo que fez os exercícios chega nesta entrega com a maior parte do trabalho pronta.** O que falta é o que nenhum exercício isolado cobra: a integração das peças em **um sistema só**, os dois documentos de projeto — o plano de prompt e a arquitetura —, e o complemento do case.

O enunciado abaixo é autossuficiente: quem não fez os exercícios encontra aqui tudo o que precisa.

---

## O que entregar

Onze itens. Os do meio são as frentes que constroem o agente:

0. **O complemento do case** — o que vocês aprenderam do domínio e que falta para alguém entender o sistema
1. **Entrada e saída** — a entrada é texto, a saída é um objeto validado, e há exemplos reais
2. **O plano de prompt engineering**, documentado e versionado
3. **A arquitetura do agente**, documentada e defendida
4. **RAG** como memória consultável — com a forma de recuperar **escolhida e justificada**
5. **Memória** entre execuções, e a gestão da conversa
6. **O grafo**, desenhado e implementado em LangGraph — com os padrões da Aula 05 nomeados
7. **O estado** que atravessa o grafo, com os reducers justificados
8. **Human-in-the-loop** — onde o grafo para, e o que acontece se ninguém responder
9. **Os casos demonstrados**, com log
10. **O projeto como entrega**

> **A entrada e a saída vêm primeiro de propósito.** É o contrato do sistema com quem o usa; tudo o que vem depois existe para cumpri-lo.

---

## 0. O complemento do case

Em `docs/case.md`, junto do que já existe. Vocês passaram semanas **dentro** do domínio — lendo os documentos, conversando com quem faz o trabalho, descobrindo os casos que não cabiam na descrição original. **Nada disso costuma estar escrito**, e é o que falta para alguém de fora entender o sistema.

Esta seção é onde isso entra: o case **completado** com o que o domínio ensinou.

**O que vocês sabem do domínio, e que não é óbvio.** O vocabulário que ninguém de fora entende. A regra que todo mundo da área conhece e nenhum documento registra. A exceção que aparece em um caso a cada vinte e que decide a arquitetura. O passo do processo que parecia um e são três.

---

## 1. Entrada e saída

O `README` diz **como usar**. Aqui isso vira contrato, verificável em código.

### 1.1 A entrada é **texto**

**O sistema recebe texto livre, em linguagem natural.** Não formulário com campos, não JSON montado por outro programa, não arquivo enviado no lugar da pergunta.

A razão está na visão geral, e vale repetir: *um sistema que recebe um formulário preenchido e devolve uma resposta não exercita quase nada*. É a entrada em texto que obriga o sistema a **descobrir** o que a pessoa quer — perguntar o que falta, lidar com informação contraditória, decidir quando já sabe o suficiente. Tudo o que o §1.4 cobra depende disso.

Isso não proíbe as outras formas; coloca cada uma no lugar certo:

| Forma | Onde ela entra |
|---|---|
| **texto livre** | **a entrada do sistema** |
| arquivo, planilha, imagem | conteúdo que o agente busca **por ferramenta**, ou corpus do §4 |
| formulário, evento de outro sistema | gatilho que **antecede** o sistema, e que o §1.3 documenta como tal |

Declarem o que acompanha o texto sem ser digitado por ninguém — identificador de usuário, de sessão, de caso —, porque é disso que o `thread_id` do §5.5 costuma ser feito.

### 1.2 O contrato de saída

A saída do sistema é um **objeto validado**, não texto solto — e o schema vive no código, em Pydantic ou equivalente. Três campos são obrigatórios, venham eles com os nomes que vocês quiserem:

- **o resultado** — a resposta, a decisão, o documento;
- **a fonte** — o que sustenta o resultado, conferível: o trecho do corpus, o registro, a ferramenta que respondeu;
- **a suficiência** — se o sistema teve ou não base para responder. A recusa é parte do contrato, não exceção dele.

### 1.3 Os exemplos, na documentação

Não um exemplo: **um conjunto**, em `docs/io.md`, com o principal repetido no `README`. No mínimo três:

| Exemplo | O que ele mostra |
|---|---|
| **o caminho feliz** | o uso para o qual o sistema foi feito |
| **a recusa** | a saída com suficiência negativa, e o que o sistema diz no lugar da resposta |
| **uma das três do §1.4** | entrada incompleta, ambígua ou fora do escopo |

Cada exemplo traz três coisas:

1. **a entrada exata**, o texto como a pessoa digitou — com os erros de digitação, se houver;
2. **a saída completa**, o objeto inteiro e não um trecho — incluindo os campos vazios;
3. **uma linha dizendo o que ler ali**: o que naquela saída mostra que o sistema fez a coisa certa.

**Copiados de uma execução**, não escritos à mão. E cada exemplo vem com o **commit** e a **data** da execução que o produziu, e com o log correspondente em `logs/` — é essa amarra que distingue exemplo real de exemplo plausível, e não a aparência do texto.

### 1.4 A entrada que não dá para atender

Três situações, cada uma com um exemplo real de execução:

| Situação | O que o sistema faz |
|---|---|
| entrada **incompleta** | pergunta o que falta, ou recusa dizendo o que falta |
| entrada **ambígua** | desambigua perguntando, ou escolhe e **declara a escolha** |
| entrada **fora do escopo** | recusa, e diz o que ele faz |

Nas três, responder como se a entrada estivesse completa produz uma saída que parece boa e não é — e é por isso que elas são tratadas no contrato, e não no improviso.

---

## 2. O plano de prompt engineering

Um documento em `docs/prompts.md`, e os prompts em arquivo, em `src/prompts/`. São coisas diferentes e moram em lugares diferentes: o documento é **raciocínio sobre o projeto**, e os prompts são **fonte**.

### 2.1 O documento

Uma linha por etapa em que o modelo é chamado — e o sistema de vocês tem várias agora: a triagem, a resposta do RAG, a extração de fato para a memória, o que mais houver.

| Etapa | Técnica | Por que ela | Contrato de saída | Como é testada |
|---|---|---|---|---|
| *ex: análise do relato* | few-shot com 3 exemplos | a classe depende de convenção do domínio que não está no modelo | JSON com `classe`, `confianca`, `justificativa` | 12 casos rotulados, ≥10 acertos |

As quatro colunas depois do nome da etapa:

**A técnica, nomeada.** Zero-shot, few-shot, cadeia de pensamento, decomposição, autoconsistência. O nome importa porque ele carrega a razão: quem escreve "few-shot" precisa dizer por que exemplos resolvem, e quem escreve "cadeia de pensamento" precisa dizer que passos de inferência existem ali.

**O contrato de saída, exato.** Formato, campos obrigatórios, valores válidos quando enumeráveis, e **o que é proibido aparecer**. Dizer apenas "JSON" deixa de fora tudo o que o consumidor precisa saber para tratar a resposta.

**Como a etapa é testada.** A forma de saber que aquela etapa funciona — quantos casos, com que critério de acerto.

### 2.2 Os prompts em arquivo

Um arquivo por prompt, versionados. A partir desta entrega os prompts vivem **em arquivo**, e não dentro do código: um prompt que só existe dentro de uma f-string não tem histórico, não tem versão, e não há como comparar o comportamento de hoje com o da semana passada.

Eles ficam em **`src/prompts/`**, junto do código, pela razão que os define: são **carregados em tempo de execução**. Um prompt é recurso do pacote, como um schema ou um arquivo de configuração — não é artefato de leitura, e não é documentação. Se hoje estiverem em outro lugar, mover é um `git mv`, e o histórico vai junto.

Cada execução registra, no início, a combinação em uso. É o item 7.

### 2.3 A regra da frase

Vale desde a Aula 03, e continua valendo:

> **Se vocês apagarem uma frase do prompt e não souberem dizer o que ela impedia, ela não estava fazendo nada.**

Façam o exercício em ao menos **um** prompt do sistema e reportem: qual frase saiu, o que aconteceu, e se ela voltou. Um prompt que encolheu sem perder comportamento é um resultado — e é o mais barato de todos, porque ele reduz custo em toda execução.

---

## 3. A arquitetura do agente, documentada

Um documento `docs/arquitetura.md`, com diagrama e texto.

### 3.1 O diagrama

ASCII é preferido, e a razão é a mesma de sempre: ele vive no `git diff`, e vocês vão alterá-lo. Precisa mostrar a entrada, as etapas, onde o **modelo** decide, onde o **código** decide, as ferramentas, o índice, a memória, **os pontos onde o sistema para e espera uma pessoa** e as condições de parada.


### 3.2 As seis perguntas

O texto responde, sobre o sistema como ele está hoje:

| | |
|---|---|
| **as ferramentas** | quais são, o que fazem, leitura ou escrita, reversível ou não |
| **o laço** | o que acontece a cada volta, e o que faz a volta seguinte existir |
| **o estado** | o que é um objeto explícito, e o que ficou na lista de mensagens |
| **o orçamento** | os tetos de **passos** e de **tempo**: o que impede uma execução de não acabar |
| **o término** | a taxonomia completa dos motivos pelos quais uma execução acaba |
| **o humano** | onde ele entra, **qual perfil de pessoa** é chamado, e o que acontece se ele não responder |

A última linha merece atenção: um sistema que suspende esperando aprovação e não define o que fazer com o silêncio fica num estado sem saída.

### 3.3 A defesa do nível de autonomia

A regra da disciplina é **usar a menor autonomia que resolve**. Digam qual é o nível — workflow, roteador ou agente — e **por que o de baixo não servia**.

A defesa é feita **com o sistema rodando**, e não em hipótese. Aponte, **no diagrama**, a decisão concreta que só o modelo consegue tomar em tempo de execução. Se não houver nenhuma, o sistema é um workflow — e isso é um resultado legítimo, desde que declarado.

---

## 4. RAG como memória consultável

Não é um chatbot sobre PDFs. É **o conhecimento de domínio que os agentes precisam para decidir** — e a diferença aparece no §4.4.

### 4.1 As formas de recuperar, e as que o seu case usa

**Nada em "RAG" diz que a recuperação é vetorial.** Existem cinco formas, e quatro delas não envolvem *embedding*. Escolher a errada produz um problema que nenhum ajuste de *chunking* conserta, porque o defeito nunca esteve no corte.

A tabela abaixo é um **cardápio, não uma lista de obrigações**. Ela existe para vocês reconhecerem o que cada pergunta do seu case pede:

| A resposta é um… | Forma adequada |
|---|---|
| **registro** — *"quem aprovou a D-4471?"* | consulta estruturada |
| **termo** — *"onde aparece 'força maior'?"* | busca textual |
| **assunto** — *"posso reembolsar almoço em viagem?"* | busca vetorial |
| **relação** — *"que artigos alteram o teto de refeição?"* | grafo |
| **agregação** — *"quais os temas recorrentes?"* | grafo, ou pré-cálculo |

**O que se entrega são duas coisas:**

**1. A classificação.** Para cada uma das dez perguntas do §4.3, qual forma a responde. É esse exercício que revela de quais formas o seu case precisa — e não o contrário.

**2. A escolha, com justificativa.** De posse da classificação, digam **quais formas o sistema implementa**, uma linha por forma usada, em `docs/rag.md`:

| Forma usada | Com o quê | O que ela responde no seu case |
|---|---|---|
| *ex: busca vetorial* | *qual banco vetorial, qual modelo de embedding, quantas dimensões* | *as seis perguntas de assunto* |

**Implementar uma só é resultado legítimo**, e é o mais comum.

A pergunta *"o seu RAG é vetorial ou é banco de dados?"* precisa ter resposta direta, e **"os dois" é a resposta mais comum nos sistemas que funcionam**.

> Vale conferir se a classificação concentrou tudo em "assunto". Quando isso acontece, costuma ser porque as perguntas foram escritas a partir do índice, e não a partir do que os usuários perguntam — e perguntas vindas de quem usa o sistema mudam o resultado.

O tratamento completo das cinco formas — incluindo quando um banco de grafo se paga e quando um `JOIN` basta — está na [nota 01 da Aula 07](../aula-07-rag-e-documentos/notas-de-aula/01-as-formas-de-recuperar.md).

### 4.2 O corpus, e o que entra nele

Declarem: que documentos entram, de onde vêm, quem os mantém e **com que frequência eles mudam**. A última pergunta é a que decide se o índice pode ser construído uma vez ou precisa ser reconstruído.

**O documento revogado.** Ponham no corpus, de propósito, um documento obsoleto que responda a uma das perguntas do conjunto, e reportem o que acontece. Se o sistema não tem como saber que aquele documento não vale mais, isso é um achado — e a solução não é técnica de recuperação, é **curadoria**.

### 4.3 As dez perguntas, e o corte

**O conjunto de perguntas** vale para qualquer forma de recuperação, e é o que alimenta a classificação do §4.1.

No mínimo **dez**, cada uma com o trecho do corpus que a responde identificado. Duas exigências de composição: ao menos duas perguntas cuja resposta depende de uma **exceção**, e ao menos uma cuja resposta **não está no corpus** — esta é a que exercita a recusa do contrato de saída (§1.2).

> Este conjunto é o primeiro *dataset* de avaliação do trabalho. Escrevam-no como se fosse durar o semestre, porque vai.

**O corte e o `k` valem apenas se o sistema usa busca vetorial.** As duas decisões existem por causa dela: o documento precisa ser partido em trechos para ser indexado, e o resultado da busca é um ranking do qual se tira um número fixo de itens. Se as formas escolhidas no §4.1 forem consulta estruturada, busca textual ou grafo, não há corte nem `k` — e basta dizer isso.

**A estratégia de corte.** Por caracteres, com sobreposição, por estrutura do documento, ou outra. Declarem a escolhida e **por que ela serve ao seu corpus**: um regulamento com artigos numerados pede corte por estrutura; uma transcrição corrida, não.

**O `k`.** Quantos trechos vão para o contexto. Declarem o valor e a razão: `k` alto esconde defeito de índice e enche a janela; `k` baixo perde a resposta.

### 4.4 Onde a recuperação entra no laço

**Este item é o que separa RAG de "colar o documento no prompt", e ele é específico desta entrega.**

Duas formas, e vocês escolhem uma e a defendem:

- **etapa fixa** — toda execução busca, sempre, antes de gerar;
- **ferramenta do agente** — o modelo decide quando buscar, como decide chamar qualquer outra ferramenta.

A segunda é o que a visão geral quer dizer com *memória consultável*: o conhecimento fica disponível **para a decisão**, e não para a redação. Ela custa uma volta a mais no laço quando o modelo decide buscar, e economiza a busca inteira quando ele decide que não precisa.

Reportem o número de chamadas de recuperação por execução na forma escolhida. Se escolheram etapa fixa, digam quantas dessas buscas foram inúteis.

---

## 5. Memória entre execuções

O sistema do §4 consulta documentos. Ele não aprende nada: a execução de amanhã começa exatamente onde a de hoje começou.

### 5.1 Os dois níveis, e a fronteira entre três estruturas

Antes da taxonomia fina, a divisão que organiza tudo o mais:

| | curto prazo | longo prazo |
|---|---|---|
| o que é | a janela da execução corrente | o que atravessa execuções |
| conteúdo | system prompt, objetivo, trajetória, trechos recuperados | episódica, semântica, procedural |
| persistido como | **checkpoint** — estado íntegro | índice, tabela e texto |
| ciclo de vida | morre com a execução | acumula |

Declarem o que ocupa cada célula **no seu case**. A coluna da esquerda é a
que costuma ser tratada como óbvia: digam o que exatamente precisa estar no
checkpoint para que a execução seja retomável sem reaplicar efeito colateral,
e declarem a **ordem de descarte** entre as fontes que disputam a janela: quando
o contexto não cabe, o que sai primeiro e o que nunca sai. Um sistema sem essa
decisão a toma sozinha, e trunca no fim, que é onde está o mais recente.

Feito isso, a fronteira entre as três estruturas que guardam informação.
Declarem, **no seu case**, a diferença entre elas. Elas se confundem porque as três guardam informação, e a confusão produz sistemas que serializam tudo e não lembram de nada:

| | checkpoint (Aula 05) | memória (Aula 08) | RAG (Aula 07) |
|---|---|---|---|
| escopo | uma execução | todas | nenhuma — é externo |
| propósito | **retomar** | **lembrar** | **consultar** |
| forma | estado íntegro | fragmento selecionado | documento de terceiro |
| leitura | uma vez, no início | por relevância, a cada volta | por relevância, quando pedido |
| quem escreve | o sistema | o sistema | ninguém — o corpus é curado |

Serializar o estado de todas as execuções **não produz memória**: produz arquivo morto. Digam, no seu case, o que a recuperação por relevância traz que a serialização integral não traria — e o que ela deixa para trás.

### 5.2 As três memórias

Cada uma na estrutura adequada, com a justificativa no código:

| Tipo | O que guarda | Estrutura |
|---|---|---|
| **episódica** | o que aconteceu em execuções passadas | índice por similaridade |
| **semântica** | fatos sobre entidades do domínio | chave-valor |
| **procedural** | como fazer algo | texto no *system prompt* |

Buscar um fato semântico por similaridade é caro e impreciso quando uma chave resolve.

### 5.3 A política de escrita

Três opções, e cada uma tem preço:

| Política | Preço |
|---|---|
| o **agente** decide o que guardar | guarda demais, e guarda errado |
| o **código** extrai por regra | perde o que a regra não previu |
| o **humano** corrige | não escala |

Escolham uma — ou uma combinação — e declarem o **volume de escrita por execução**: quantos registros, de que tamanho.

**A escrita de memória é escrita.** Ela tem efeito no mundo, e portanto exige chave de idempotência **derivada do conteúdo**, com `ja_existia` no retorno. É a derivação que permite reconhecer a repetição: o mesmo conteúdo produz a mesma chave.

### 5.4 Como o agente perde a memória

São **três causas distintas**, e elas diferem no momento em que a decisão ocorre e no que acontece com o dado:

| Causa | O que aconteceu | O dado é apagado? | Quando decide |
|---|---|---|---|
| **contradição** | o fato mudou | não | na leitura, pelo carimbo de tempo |
| **decaimento** | o fato envelheceu sem ser contradito | sim, ou é rebaixado | em rotina |
| **remoção** | o titular solicitou | sim, obrigatoriamente | sob demanda |

Especifiquem as três. Para o **decaimento**, o corte em dias e a justificativa dele no domínio — o valor adequado depende da taxa de mudança do que se guarda —, e se o efeito é remover ou apenas rebaixar na ordenação.

Para a **remoção**, o requisito é de **cobertura**: o dado precisa sair de todas as estruturas em que foi gravado, e não apenas das que vieram à lembrança. Listem essas estruturas. A lista tem mais itens que a taxonomia sugere, e três escapam com frequência:

- o **checkpoint**, que não é uma das três memórias e guarda os argumentos de cada passo;
- o **log**, se ele registra os argumentos das chamadas de ferramenta;
- a memória **procedural**, quando uma regra aprendida a partir do erro de um titular menciona esse titular.

A verificação é independente da remoção: gravem, verifiquem que o comportamento mudou, removam, verifiquem que voltou — e então varram as estruturas procurando o identificador.

As três razões para o esquecimento existir:

- **privacidade** — o titular pede a remoção;
- **custo** — o crescimento é monotônico;
- **recuperação de incidente** — sem ele, uma memória envenenada é permanente, e cada execução futura repete o comportamento injetado.

Se o esquecimento exige reconstruir o índice inteiro, declarem isso. Ele continua servindo às duas primeiras razões e deixa de servir à terceira: uma memória envenenada que só sai com reconstrução completa é permanente até a próxima janela de manutenção.

### 5.5 A conversa, e o que sobra dela

As quatro subseções acima tratam do que atravessa execuções. Falta o que acontece **dentro** de uma conversa que dura — e esta é a parte que o checkpointer resolve.

Quatro decisões, em `docs/memoria.md`:

**O que identifica uma conversa.** O `thread_id` é o endereço do checkpoint: mesma chave, mesma conversa; chave nova, histórico vazio. Digam o que compõe essa chave no seu case — usuário, sessão, caso, protocolo — e o que acontece quando a mesma pessoa abre duas conversas ao mesmo tempo.

**Onde o checkpoint é gravado.** Em memória, em arquivo ou em banco. Se for em memória, a conversa morre com o processo, e isso precisa estar declarado como limitação — não descoberto na apresentação.

**O que acontece quando a janela enche.** Uma conversa longa estoura o contexto, e alguém decide o que cortar. Declarem a política — truncar o início, sumarizar o trecho antigo, descartar os retornos de ferramenta já usados — e **o que se perde em cada uma**. Um sistema que não decide trunca no fim, que é onde está o mais recente.

**O que passa da conversa para a memória de longo prazo.** Quando a conversa acaba, o que dela sobrevive? Essa é a ponte entre esta subseção e o §5.3, e a resposta honesta para muitos cases é *"nada"* — desde que dita.

---

## 6. O grafo, desenhado e implementado

O sistema é um **grafo**: nós que fazem coisas, arestas que decidem o que vem depois, e um estado que atravessa tudo.

> **Qual framework.** A visão geral fala em *workflow com LangChain*; o que se usa é o **LangGraph**, do mesmo ecossistema, porque o modelo mental dele — grafo de estado — é literalmente o que a Aula 05 pediu que vocês desenhassem à mão. A Aula 09 ensina os dois.

### 6.1 O desenho vem antes do código

Em `docs/arquitetura.md`, o grafo desenhado **em ASCII**, pela mesma razão do §3.1: ele vive no `git diff`, e vocês vão alterá-lo. Três coisas precisam ficar legíveis:

- **cada nó**: o que ele faz, e se ele chama o modelo ou não;
- **cada aresta**: incondicional ou condicional, e **quem decide** a condição;
- **os términos**: por onde a execução pode acabar, e não só o caminho feliz.

São as arestas condicionais que distinguem um grafo de uma sequência: é nelas que a execução escolhe caminho.

### 6.2 Os padrões, nomeados e justificados

A Aula 05 deu cinco padrões de workflow mais o agente, e a Aula 09 mostrou como cada um vira grafo. **Para cada padrão que o sistema de vocês usa**, uma linha:

| Padrão | Onde, no grafo | Por que ele, e não o de baixo | Condição de não uso |
|---|---|---|---|
| sequencial com portão | | | |
| *router* | | | |
| paralelização (*sectioning* / *voting*) | | | |
| orquestrador-trabalhador | | | |
| avaliador-otimizador | | | |
| agente (laço) | | | |

Uma exigência sobre essa tabela:

- **O agente é o degrau mais caro.** Se o laço livre aparece na tabela, justifiquem por que o trabalho não cabia num workflow — e a Aula 05 é explícita: autonomia é recurso escasso, e o nível certo é o **mais baixo** que resolve.

### 6.3 Quem decide cada bifurcação

Para **cada aresta condicional**, digam qual dos três degraus decide:

| Degrau | Quando basta | Custo |
|---|---|---|
| **regra em código** | a condição é verificável — um campo, um limiar, um `None` | zero |
| **embedding** | a condição é de assunto, e as rotas são poucas e estáveis | uma chamada barata |
| **modelo** | a condição exige julgamento sobre texto livre | uma chamada cara, **e que erra** |

A ordem é a da Aula 06, e o critério também: use o degrau mais barato que resolve. Um portão que procura um campo vazio não precisa de modelo.

**E quando for modelo, a saída é estruturada** — `Literal` com as rotas válidas, não texto livre. O schema garante que a resposta seja uma das rotas; ele **não** garante que seja a rota certa. Essas são duas coisas diferentes, e quem confunde descobre em produção.

### 6.4 A implementação, e a conferência contra o desenho

O sistema roda sobre `StateGraph`. O que precisa estar no código:

- os nós registrados e as arestas ligadas — condicionais com o mapa de destinos explícito;
- paralelismo, se houver, por arestas que saem do mesmo ponto ou por `Send`;
- o grafo compilado uma vez, e não remontado a cada execução.

E a conferência, que é item de avaliação. Um script curto — `conferir_grafo.py`, na raiz — que **importe o mesmo objeto de grafo que o `src/` executa** e imprima dele duas listas:

- os **nós**, por nome;
- as **arestas**, na forma `origem -> destino`, com as condicionais marcadas.

`grafo.get_graph()` dá as duas sem dependência nenhuma. A saída vai colada em `docs/arquitetura.md`, **ao lado** do desenho do §6.1, junto com o comando que a produziu.

Duas regras sobre essa conferência, e a segunda é a que importa:

- se as duas listas e o desenho divergirem, **o que vale é o código** — e o desenho estava errado;
- o script **importa o grafo do sistema entregue**, e não monta um grafo próprio. Um grafo de demonstração montado dentro do script produz a mesma saída bonita e não prova nada.

> **Sobre paralelismo, um aviso medido em sala.** Despachar nós em paralelo no grafo **não** produz paralelismo se o provedor atende uma requisição por vez. Se vocês afirmarem ganho de tempo, meçam: o total precisa tender ao **maior** dos ramos, e não à **soma** deles. Na Aula 09, com modelo local, deu a soma.

### 6.5 O que o framework não deu

Reimplantem sobre o grafo, e **contem as linhas que isso custou**:

- **o motivo de término** — o nó terminal diz que acabou, não diz por quê;
- **o detector de laço** — `recursion_limit` limita o dano e não detecta; um agente preso na primeira volta gasta o teto inteiro e devolve o diagnóstico errado;
- **os tetos** — dos quatro da Aula 05, verifiquem quais o framework cobre. Provavelmente só o de passos;
- **erro recuperável × fatal** — a classificação continua sendo de vocês.

Uma frente que o framework cobre inteira também conta, e com evidência: `arquivo:linha` mostrando que vocês **não** precisaram escrever aquilo.

### 6.6 A decisão que virou parâmetro

Encontrem, na implementação, **ao menos uma decisão de projeto que o framework converteu em parâmetro com valor padrão** — um limiar, um `k`, um limite de recursão, uma política de repetição, um reducer.

Descrevam: qual era a decisão, como ela estava justificada quando era explícita, e o que acontece com o sistema **se o padrão da biblioteca mudar numa atualização**. Depois declarem o parâmetro no código, mesmo que o valor coincida com o padrão.

Este item separa quem usou o framework de quem entendeu o que ele escondeu.

---

## 7. O estado que atravessa o grafo

Na Aula 05 vocês aprenderam que **a lista de mensagens não é o estado**. No grafo isso deixa de ser argumento e vira declaração de tipo.

### 7.1 Os campos, e quem mexe em cada um

Uma tabela em `docs/arquitetura.md`, com uma linha por campo:

| Campo | Tipo | Quem **escreve** | Quem **lê** | De onde veio |
|---|---|---|---|---|

A coluna "quem escreve" é a que revela defeito: um campo escrito por três nós diferentes, sem reducer, é uma corrida — e o último a terminar ganha.

### 7.2 Os reducers, e o que acontece sem eles

Para cada campo acumulado, digam **qual reducer** e **por quê**. E demonstrem, com uma execução, o que o sistema faz **sem** ele: num grafo com nós paralelos, um campo sem reducer é sobrescrito, e o trabalho dos outros ramos desaparece sem erro nenhum.

Essa demonstração é item de avaliação porque é o defeito mais silencioso do padrão: não levanta exceção, não aparece no log, e só se manifesta como resultado incompleto.

### 7.3 O que NÃO está no estado

Duas listas curtas, e a segunda é a que vale nota:

- **o que ficou na lista de mensagens** e por quê — texto conversacional costuma ficar; decisão, contador e identificador, não;
- **o que é derivado e não guardado** — porque recalcular é mais barato que manter sincronizado, ou porque guardar criaria duas fontes de verdade.

### 7.4 O estado entre agentes

Se o sistema tiver mais de um agente — e ele não precisa ter —, declarem **o que um passa ao outro**: o estado inteiro, um subconjunto, ou um objeto de contrato próprio.

Passar o estado inteiro acopla os dois agentes a cada campo novo: qualquer alteração no estado alcança os dois. Se foi essa a escolha, digam se ela é definitiva ou provisória.

---

## 8. Human-in-the-loop

O humano entra no grafo como **código que para e espera**.

### 8.1 Onde o grafo para, e por qual critério

Marquem no diagrama do §6.1 **todos** os pontos de parada, e para cada um digam o critério. Os três que costumam valer:

| Critério | Exemplo |
|---|---|
| **ação irreversível** | escreve em sistema de terceiro, envia mensagem, move dinheiro |
| **custo acima de um teto** | a ação passa de um valor que vocês declaram |
| **confiança baixa** | o sistema recuperou pouco, ou o avaliador reprovou duas vezes |

Um sistema sem nenhum ponto de parada precisa justificar: ou ele não faz nada irreversível — e aí digam qual é a ação mais perigosa dele —, ou ele deveria parar e não para.

### 8.2 O que a pessoa vê, e o que ela devolve

O que o sistema mostra no momento da pausa é prompt, e vale a regra do §2.3: **a ação exata**, com os argumentos, e a consequência dela. Uma pergunta do tipo *"confirma?"* sem dizer o que vai acontecer transfere a responsabilidade sem transferir a informação.

As respostas aceitas precisam incluir, no mínimo, **aprovar** e **rejeitar**. **Aprovar editando** — a pessoa corrige um argumento antes de deixar seguir — é o que separa aprovação de carimbo, e vale ponto.

### 8.3 A implementação, e por que ela depende do checkpoint

A pausa é um `interrupt` dentro do nó; a retomada é uma nova invocação com `Command(resume=...)`, e o valor devolvido vira o retorno do `interrupt` — a função continua da linha seguinte.

Isso **só funciona porque existe um checkpointer**: a pausa é um checkpoint gravado, endereçado por `thread_id`. Digam qual checkpointer usaram e onde ele grava — e se for em memória, o sistema perde a aprovação pendente quando o processo morre, o que precisa estar declarado como limitação.

### 8.4 O que acontece se ninguém responder

Um prazo, e o que ocorre quando ele vence: a execução expira, é escalada, ou segue por um caminho padrão declarado. O caminho precisa estar escrito — um ponto de parada que segue adiante sozinho, em silêncio, deixa de cumprir a função que tinha.

### 8.5 Aprovar duas vezes

Demonstrem que a ação **não acontece duas vezes** se a aprovação chegar repetida — por chave de idempotência derivada do conteúdo, como a Aula 05 pede para qualquer escrita. Com log das duas chamadas e a prova de que a segunda não criou nada.

---

## 9. Os casos demonstrados

**Seis execuções, com log no repositório:**

| # | O caso | O que ele prova | Onde isto é exigido |
|---|---|---|---|
| 1 | o caso simples | o caminho feliz existe | §1.3 |
| 2 | a **divergência** | o sistema diz uma coisa, o usuário diz outra | §1.4 |
| 3 | o **registro inexistente** | erro de ferramenta que o modelo contorna | §3.2 |
| 4 | o caso que **não** deve disparar a ação principal | o sistema sabe não agir | §3.3 |
| 5 | a pergunta **fora do corpus** | o sistema recusa, e recusa dizendo algo útil | §1.2 |
| 6 | a ação que **para e espera** | o grafo pausa, a pessoa aprova editando, e a segunda aprovação não duplica nada | §8.2, §8.5 |

Os casos 5 e 6 mostram o sistema **fora do caminho feliz** — a recusa e a pausa para aprovação —, e é por isso que estão na lista.

---

## 10. O projeto como entrega

Cinco exigências, e nenhuma delas é sobre o código em si.

**`README.md`** — duas perguntas, e a segunda é a que costuma faltar: **como rodar** (do zero, por quem nunca viu o projeto, em menos de cinco minutos) e **como usar**, com o exemplo principal do §1.3 — **o texto que a pessoa digita** e a saída inteira que volta — e **o que acontece quando o grafo para e espera alguém**.

**`requirements.txt`** com todas as versões fixadas — inclusive a do framework, pela razão do §6.6: parte do comportamento do sistema passou a morar na biblioteca.

**`.env.example`** com os nomes das variáveis e nenhum valor. Chave de API **nunca** no repositório.

**`docs/`** — toda a pesquisa e documentação, em Markdown, versionada. O `case.md` **complementado**, o `prompts.md`, o `arquitetura.md` — que carrega o grafo, a tabela de padrões, a tabela de campos do estado e os pontos de parada — e os resultados de medição de cada frente. O histórico dessa pasta é o que mostra **quando o grupo mudou de ideia sobre o próprio case, e por quê**.

**`src/prompts/`** — os prompts em arquivo, versionados, junto do código que os carrega.

## Entrega

No repositório do grupo:

```
README.md          como rodar e como usar, com exemplo real
requirements.txt   tudo fixado: framework, embeddings
.env.example       os nomes das variáveis, sem nenhum valor

docs/
  case.md            COMPLEMENTADO — o item 0
  modelos.md         a análise de modelos
  prompts.md         o item 1
  arquitetura.md     o item 2
  rag.md             formas usadas, as dez perguntas, o corpus revogado
                     e, se houver busca vetorial, o corte e o k
  memoria.md         a fronteira, as três memórias, a política, a conversa
  grafo.md           o desenho, os padrões, o estado, o que o framework não deu
  humano.md          os pontos de parada, o critério, o prazo, a idempotência
  io.md              os schemas e os três exemplos, com commit, data e log
  fontes.md          tudo que foi consultado, com link

src/               o sistema
  prompts/         os prompts, versionados — junto do código que os carrega
dados/             o corpus e os dados simulados
logs/              as 6 execuções demonstradas
```

Markdown, sempre — nada de `.docx` nem `.pdf`, para que o `git diff` funcione.
