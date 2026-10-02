# Trabalho da disciplina — Parte 2: O agente de verdade

> Leiam antes a **[visão geral do trabalho](00-visao-geral.md)** e a **[Parte 1](01-primeira-entrega.md)**: esta é a segunda de três entregas sobre o **mesmo** sistema, no **mesmo** repositório.

## Contexto

Esta entrega avalia **a qualidade de uma construção**, e cada frente acrescenta uma peça ao sistema **e cobra uma evidência por ela**. A decisão de corte, declarada com a razão. A taxa de recusa indevida do portão. As linhas que vocês tiveram de escrever porque o framework não cobria. O log das duas aprovações provando que a segunda não criou nada.

> **A regra desta parte:** nenhuma decisão de projeto é aceita como preferência. Toda escolha vem com a medida que a sustenta — e, quando a medida contraria a escolha óbvia, **é a medida que vale**.

O sistema que sai daqui é um agente completo: ele é um **grafo** de nós e arestas, com estado declarado, contrato de entrada e saída, pontos onde para e espera uma pessoa, conhecimento de domínio consultável e memória do que já aconteceu.

> **MCP saiu desta entrega.** Publicar a integração como servidor passou para a **Parte 3**, junto com multiagente — é lá que ela encontra o problema que a justifica, que é mais de um agente consumindo a mesma ferramenta. A integração com software tradicional continua valendo aqui, como ferramenta do agente.

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

**Um grupo que não os fez** entrega o mesmo, e o enunciado abaixo é autossuficiente. Só vai levar cinco vezes mais tempo.

---

## O que entregar

Doze itens. Os do meio são as frentes que constroem o agente:

0. **O complemento do case** — o que vocês aprenderam do domínio e que falta para alguém entender o sistema
1. **O plano de prompt engineering**, documentado e versionado
2. **A arquitetura do agente**, documentada e defendida
3. **RAG** como memória consultável — com a forma de recuperar **escolhida e justificada**
4. **Memória** entre execuções
5. **O grafo**, desenhado e implementado em LangGraph — com os padrões da Aula 05 nomeados
6. **O estado** que atravessa o grafo, com os reducers justificados
7. **Entrada e saída**, com contrato validado e exemplo real
8. **Human-in-the-loop** — onde o grafo para, e o que acontece se ninguém responder
9. **O carimbo** — os campos da combinação que produziu cada execução
10. **Os casos demonstrados**, com log
11. **O projeto como entrega**

---

## 0. O complemento do case

Em `docs/case.md`, junto do que já existe. Vocês passaram semanas **dentro** do domínio — lendo os documentos, conversando com quem faz o trabalho, descobrindo os casos que não cabiam na descrição original. **Nada disso costuma estar escrito**, e é o que falta para alguém de fora entender o sistema.

Esta seção é onde isso entra: o case **completado** com o que o domínio ensinou.

**O que vocês sabem do domínio, e que não é óbvio.** O vocabulário que ninguém de fora entende. A regra que todo mundo da área conhece e nenhum documento registra. A exceção que aparece em um caso a cada vinte e que decide a arquitetura. O passo do processo que parecia um e são três.

Escrevam para quem vai ler o repositório sem ter conversado com vocês. É o teste: se um colega de outro grupo não consegue dizer, depois de ler, **por que o sistema é assim**, falta domínio escrito.

**O escopo, como ele está hoje.** Qual é o problema, quem é o usuário principal, o que entra e o que fica de fora. Ajustar o escopo é esperado e bem-visto; o que não vale é ajustar sem registrar. O `git log` de `docs/` é a evidência.

**O verificador, como ele funciona.** Como vocês sabem que uma saída está certa — com o sistema rodando, não em hipótese. Se ele se mostrou impossível de construir, **digam agora**: a Parte 3 inteira se apoia nele, e é barato trocar hoje.

**A linha de base do ganho prometido.** O ganho que vocês anunciaram se apoia num número de partida — quanto a tarefa custa, demora ou erra sem o sistema. Declarem o número que vale hoje. Não é preciso medir nada de novo para esta entrega; se ele mudou, basta registrar o valor atual e por que ele é esse.

> **Esta é também a última janela para ajustar o rumo.** A visão geral permite refinar o tema até aqui, e a Parte 3 vai cobrar a conta do que foi prometido.
>
> E a primeira pergunta **não** admite resposta vazia: um grupo que passou semanas no domínio e não tem nada a acrescentar sobre ele ou não entrou no domínio, ou não percebeu que entrou.

---

## 1. O plano de prompt engineering

Um documento em `docs/prompts.md`, e os prompts em arquivo, em `src/prompts/`. São coisas diferentes e moram em lugares diferentes: o documento é **raciocínio sobre o projeto**, e os prompts são **fonte**.

### 1.1 O documento

Uma linha por etapa em que o modelo é chamado — e o sistema de vocês tem várias agora: a triagem, a resposta do RAG, a extração de fato para a memória, o que mais houver.

| Etapa | Técnica | Por que ela | Contrato de saída | Como é testada |
|---|---|---|---|---|
| *ex: análise do relato* | few-shot com 3 exemplos | a classe depende de convenção do domínio que não está no modelo | JSON com `classe`, `confianca`, `justificativa` | 12 casos rotulados, ≥10 acertos |

As quatro colunas depois do nome da etapa valem ponto separadamente, e a terceira é a que os grupos escrevem mal:

**A técnica, nomeada.** Zero-shot, few-shot, cadeia de pensamento, decomposição, autoconsistência. O nome importa porque ele carrega a razão: quem escreve "few-shot" precisa dizer por que exemplos resolvem, e quem escreve "cadeia de pensamento" precisa dizer que passos de inferência existem ali.

**O contrato de saída, exato.** Formato, campos obrigatórios, valores válidos quando enumeráveis, e **o que é proibido aparecer**. Um contrato que diz "JSON" não é contrato.

**Como a etapa é testada.** Cada etapa precisa de uma forma de saber que ela funciona, e essa forma tem denominador. "Testamos manualmente" não é teste. Estes testes são o embrião do conjunto de avaliação que a Aula 11 vai cobrar de verdade.

### 1.2 Os prompts em arquivo

Um arquivo por prompt, versionados. **Prompt embutido no meio do código não é aceito a partir desta entrega** — é a regra que a visão geral anuncia, e ela existe porque um prompt que só existe dentro de uma f-string não tem histórico, não tem versão e não pode ser comparado com a execução da semana passada.

Eles ficam em **`src/prompts/`**, junto do código, pela razão que os define: são **carregados em tempo de execução**. Um prompt é recurso do pacote, como um schema ou um arquivo de configuração — não é artefato de leitura, e não é documentação. Se hoje estiverem em outro lugar, mover é um `git mv`, e o histórico vai junto.

Cada execução registra, no início, a combinação em uso. É o item 7.

### 1.3 A regra da frase

Vale desde a Aula 03, e continua valendo:

> **Se vocês apagarem uma frase do prompt e não souberem dizer o que ela impedia, ela não estava fazendo nada.**

Façam o exercício em ao menos **um** prompt do sistema e reportem: qual frase saiu, o que aconteceu, e se ela voltou. Um prompt que encolheu sem perder comportamento é um resultado — e é o mais barato de todos, porque ele reduz custo em toda execução.

---

## 2. A arquitetura do agente, documentada

Um documento `docs/arquitetura.md`, com diagrama e texto.

### 2.1 O diagrama

ASCII é preferido, e a razão é a mesma de sempre: ele vive no `git diff`, e vocês vão alterá-lo. Precisa mostrar a entrada, as etapas, onde o **modelo** decide, onde o **código** decide, as ferramentas, o índice, a memória, **os pontos onde o sistema para e espera uma pessoa** e as condições de parada.


### 2.2 As seis perguntas

O texto responde, sobre o sistema como ele está hoje:

| | |
|---|---|
| **as ferramentas** | quais são, o que fazem, leitura ou escrita, reversível ou não |
| **o laço** | o que acontece a cada volta, e o que faz a volta seguinte existir |
| **o estado** | o que é um objeto explícito, e o que ficou na lista de mensagens |
| **o orçamento** | os tetos de **passos** e de **tempo**: o que impede uma execução de não acabar. A conta em tokens e em dinheiro é da Parte 3 |
| **o término** | a taxonomia completa dos motivos pelos quais uma execução acaba |
| **o humano** | onde ele entra, **qual perfil de pessoa** é chamado, e o que acontece se ele não responder |

A última linha é a que mais some das entregas. Um sistema que suspende esperando aprovação e não define o que fazer com o silêncio tem um estado sem saída.

### 2.3 A defesa do nível de autonomia

A regra da disciplina é **usar a menor autonomia que resolve**. Digam qual é o nível — workflow, roteador ou agente — e **por que o de baixo não servia**.

A defesa é feita **com o sistema rodando**, e não em hipótese. Aponte, **no diagrama**, a decisão concreta que só o modelo consegue tomar em tempo de execução. Se não houver nenhuma, o sistema é um workflow — o que é um resultado legítimo, desde que declarado, e desde que vocês digam onde a decisão entra na Parte 3.

---

## 3. RAG como memória consultável

Não é um chatbot sobre PDFs. É **o conhecimento de domínio que os agentes precisam para decidir** — e a diferença aparece no §3.6.

### 3.1 As formas de recuperar, e as que o seu case usa

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

**1. A classificação.** Para cada uma das dez perguntas do §3.3, qual forma a responde. É esse exercício que revela de quais formas o seu case precisa — e não o contrário.

**2. A escolha, com justificativa.** De posse da classificação, digam **quais formas o sistema implementa**, uma linha por forma usada, em `docs/rag.md`:

| Forma usada | Com o quê | O que ela responde no seu case |
|---|---|---|
| *ex: busca vetorial* | *qual banco vetorial, qual modelo de embedding, quantas dimensões* | *as seis perguntas de assunto* |

**Implementar uma só é resultado legítimo**, e é o mais comum.

A pergunta *"o seu RAG é vetorial ou é banco de dados?"* precisa ter resposta direta, e **"os dois" é a resposta mais comum nos sistemas que funcionam**.

> Se as dez perguntas caíram todas em "assunto", o conjunto foi escrito para o índice que já existia, e não para o que os usuários perguntam. Acrescentem perguntas vindas de quem usa o sistema e refaçam a classificação.

O tratamento completo das cinco formas — incluindo quando um banco de grafo se paga e quando um `JOIN` basta — está na [nota 01 da Aula 07](../aula-07-rag-e-documentos/notas-de-aula/01-as-formas-de-recuperar.md).

### 3.2 O corpus, e o que entra nele

Declarem: que documentos entram, de onde vêm, quem os mantém e **com que frequência eles mudam**. A última pergunta é a que decide se o índice pode ser construído uma vez ou precisa ser reconstruído.

**O documento revogado.** Ponham no corpus, de propósito, um documento obsoleto que responda a uma das perguntas do conjunto, e reportem o que acontece. Se o sistema não tem como saber que aquele documento não vale mais, isso é um achado — e a solução não é técnica de recuperação, é **curadoria**.

### 3.3 O corte, e as dez perguntas

**O conjunto de perguntas.** No mínimo **dez**, cada uma com o trecho do corpus que a responde identificado. Duas exigências de composição: ao menos duas perguntas cuja resposta depende de uma **exceção**, e ao menos uma cuja resposta **não está no corpus** — esta é a mais informativa das dez, e é ela que valida o portão do §3.5.

> Este conjunto é o primeiro *dataset* de avaliação do trabalho, e a Parte 3 o retoma sob esse nome. Escrevam-no como se fosse durar o semestre, porque vai.

**A estratégia de corte.** Por caracteres, com sobreposição, por estrutura do documento, ou outra. Declarem a escolhida e **por que ela serve ao seu corpus**: um regulamento com artigos numerados pede corte por estrutura; uma transcrição corrida, não.

**O `k`.** Quantos trechos vão para o contexto. Declarem o valor e a razão: `k` alto esconde defeito de índice e enche a janela; `k` baixo perde a resposta.

> **A medição fica para a Parte 3.** Aqui o que se cobra é a **decisão declarada** e o conjunto de perguntas escrito. Comparar estratégias de corte com `recall@k` é trabalho da suíte de avaliação, e é lá que ele tem onde se apoiar.

### 3.4 A citação verificável

A resposta cita a fonte, e a citação é **verificada em código**:

```
o identificador citado está entre os trechos que foram recuperados?
```

Reportem a **taxa de citações verificáveis**, e não a existência de citação. Uma citação que o modelo produziu mas que não está no contexto é alucinação com aparência de rigor — o modo de falha mais perigoso do RAG, porque ele parece o oposto de uma falha.

### 3.5 O portão, e os dois erros

Implementem o limiar abaixo do qual o sistema **não responde**. Duas decisões, ambas avaliadas:

**Onde fica o limiar** — e ele é medido, não escolhido no chute. Rodem com vários valores e olhem a curva dos dois erros.

**O que o sistema diz quando recusa.** "Não sei" é pior que "o regulamento não trata deste assunto; o mais próximo que encontrei foi o artigo 14".

E os dois erros, medidos separadamente:

| Erro | O que é | Custo |
|---|---|---|
| **falso positivo** | respondeu quando devia recusar | resposta errada com aparência de fundamentada |
| **falso negativo** | recusou quando podia responder | sistema inútil |

Um limiar que zera o primeiro e dispara o segundo produz um sistema que ninguém usa. **Declarem a troca escolhida**, e liguem-na ao custo do erro do seu domínio — se errar para um lado é muito pior, o limiar reflete isso.

### 3.6 Onde a recuperação entra no laço

**Este item é o que separa RAG de "colar o documento no prompt", e ele é específico desta entrega.**

Duas formas, e vocês escolhem uma e a defendem:

- **etapa fixa** — toda execução busca, sempre, antes de gerar;
- **ferramenta do agente** — o modelo decide quando buscar, como decide chamar qualquer outra ferramenta.

A segunda é o que a visão geral quer dizer com *memória consultável*: o conhecimento fica disponível **para a decisão**, e não para a redação. Ela custa uma volta a mais no laço quando o modelo decide buscar, e economiza a busca inteira quando ele decide que não precisa.

Reportem o número de chamadas de recuperação por execução na forma escolhida. Se escolheram etapa fixa, digam quantas dessas buscas foram inúteis.

### 3.7 A expansão de conhecimento, demonstrada

O RAG só se justifica se o agente **passa a saber algo que não sabia**. Demonstrem isso com um par de execuções sobre a mesma pergunta:

- **sem o corpus** — o que o sistema responde apoiado só no modelo e nas ferramentas;
- **com o corpus** — a mesma pergunta, com a recuperação ligada.

As duas saídas lado a lado, em `docs/rag.md`. Três resultados possíveis, e **os três são aceitáveis desde que medidos**:

| Resultado | O que significa |
|---|---|
| a resposta **melhorou** | o corpus traz conhecimento que o modelo não tem — é o caso esperado |
| a resposta **ficou igual** | ou o modelo já sabia, ou o corpus não cobre a pergunta; nos dois casos, o corpus precisa mudar |
| a resposta **piorou** | a recuperação trouxe trecho irrelevante e o modelo seguiu por ele — é o modo de falha 2 da Aula 07 |

O corpus precisa conter **conhecimento de domínio que o modelo não poderia ter**: norma interna, dado do período, decisão da organização, catálogo próprio. Um corpus feito de informação pública e estável é um RAG que não expande nada — e a demonstração acima vai mostrar isso.

---

## 4. Memória entre execuções

O sistema do §3 consulta documentos. Ele não aprende nada: a execução de amanhã começa exatamente onde a de hoje começou.

### 4.1 Os dois níveis, e a fronteira entre três estruturas

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

> **A conta em tokens é da Parte 3.** Aqui o que se cobra é a **decisão** —
> quem cede lugar a quem —, não a medida.

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

### 4.2 As três memórias

Cada uma na estrutura adequada, com a justificativa no código:

| Tipo | O que guarda | Estrutura |
|---|---|---|
| **episódica** | o que aconteceu em execuções passadas | índice por similaridade |
| **semântica** | fatos sobre entidades do domínio | chave-valor |
| **procedural** | como fazer algo | texto no *system prompt* |

Buscar um fato semântico por similaridade é caro e impreciso quando uma chave resolve.

### 4.3 A política de escrita

Três opções, e cada uma tem preço:

| Política | Preço |
|---|---|
| o **agente** decide o que guardar | guarda demais, e guarda errado |
| o **código** extrai por regra | perde o que a regra não previu |
| o **humano** corrige | não escala |

Escolham uma — ou uma combinação — e declarem o **volume de escrita por execução**: quantos registros, de que tamanho. O que esse volume custa em dinheiro e em armazenamento é assunto da Parte 3, e é bom que o número já exista.

**A escrita de memória é escrita.** Ela tem efeito no mundo, e portanto exige chave de idempotência derivada do conteúdo, com `ja_existia` no retorno. `uuid4()` a cada chamada não é chave de idempotência.

### 4.4 O que NÃO entra

**A parte mais importante desta frente, e a que vale mais nota.**

Listem explicitamente o que o sistema não guarda:

- **dado sensível** — identificador pessoal, credencial, valor que não precisa persistir. e a decisão de não guardar precisa ser explícita;
- **conteúdo que veio de fora sem verificação** — memória é durável, e **o que entra nela sai muitas vezes**;
- **o que é derivável** — se pode ser recalculado, guardar é dívida.

A segunda linha é a que a Parte 3 vai cobrar sob outro nome.

### 4.5 A contradição, resolvida em código

Introduzam no domínio **dois fatos verdadeiros em datas diferentes** que respondam à mesma pergunta — uma política que mudou, um valor reajustado, uma regra revogada. Qualquer domínio real tem um; procurem uma regra que mudou de valor.

Os dois são igualmente similares à pergunta, porque **o vetor não tem noção de anterioridade**.

Resolvam **em código**: carimbo de tempo obrigatório em todo fato, e regra de desempate determinística. Delegar o desempate ao modelo é o antipadrão, e ele decorre de um princípio que a Aula 06 estabeleceu: data, valor e identificador se comparam com `==`, não por similaridade.

### 4.6 Como o agente perde a memória

São **três causas distintas**, e elas diferem no momento em que a decisão ocorre e no que acontece com o dado:

| Causa | O que aconteceu | O dado é apagado? | Quando decide |
|---|---|---|---|
| **contradição** | o fato mudou | não | na leitura (§4.5) |
| **decaimento** | o fato envelheceu sem ser contradito | sim, ou é rebaixado | em rotina |
| **remoção** | o titular solicitou | sim, obrigatoriamente | sob demanda |

Especifiquem as três. Para o **decaimento**, o corte em dias e a justificativa dele no domínio — o valor adequado depende da taxa de mudança do que se guarda —, e se o efeito é remover ou apenas rebaixar na ordenação.

Para a **remoção**, o requisito é de **cobertura**: o dado precisa sair de todas as estruturas em que foi gravado, e não apenas das que vieram à lembrança. Listem essas estruturas. A lista tem mais itens que a taxonomia sugere, e três escapam com frequência:

- o **checkpoint**, que não é uma das três memórias e guarda os argumentos de cada passo;
- o **log**, se ele registra os argumentos das chamadas de ferramenta;
- a memória **procedural**, quando uma regra aprendida a partir do erro de um titular menciona esse titular.

**Demonstrem que funciona, e a verificação é independente da remoção**: gravem, verifiquem que o comportamento mudou, removam, verifiquem que voltou — e então varram as estruturas procurando o identificador. Uma remoção que se declara concluída sem varredura não cumpre a obrigação; apenas afirma tê-la cumprido.

As três razões para o esquecimento existir:

- **privacidade** — o titular pede a remoção;
- **custo** — o crescimento é monotônico;
- **recuperação de incidente** — sem ele, uma memória envenenada é permanente, e cada execução futura repete o comportamento injetado.

Se o esquecimento exige reconstruir o índice inteiro, declarem isso. Ele continua servindo às duas primeiras razões e deixa de servir à terceira: uma memória envenenada que só sai com reconstrução completa é permanente até a próxima janela de manutenção.

### 4.7 O preço: o sistema deixou de ser reprodutível

Executem **a mesma entrada duas vezes, com memórias diferentes**, e reportem a diferença.

Isto não é um defeito a corrigir — é **consequência de projeto**, e vocês a escolheram deliberadamente ao construir o §4. Registrem-na, porque a Parte 3 vai ter de conviver com ela: um conjunto de avaliação que roda sobre um sistema com memória mede duas coisas ao mesmo tempo, e separá-las é trabalho.

### 4.8 A conversa, e o que sobra dela

As quatro subseções acima tratam do que atravessa execuções. Falta o que acontece **dentro** de uma conversa que dura — e esta é a parte que o checkpointer resolve.

Quatro decisões, em `docs/memoria.md`:

**O que identifica uma conversa.** O `thread_id` é o endereço do checkpoint: mesma chave, mesma conversa; chave nova, histórico vazio. Digam o que compõe essa chave no seu case — usuário, sessão, caso, protocolo — e o que acontece quando a mesma pessoa abre duas conversas ao mesmo tempo.

**Onde o checkpoint é gravado.** Em memória, em arquivo ou em banco. Se for em memória, a conversa morre com o processo, e isso precisa estar declarado como limitação — não descoberto na apresentação.

**O que acontece quando a janela enche.** Uma conversa longa estoura o contexto, e alguém decide o que cortar. Declarem a política — truncar o início, sumarizar o trecho antigo, descartar os retornos de ferramenta já usados — e **o que se perde em cada uma**. Um sistema que não decide trunca no fim, que é onde está o mais recente.

**O que passa da conversa para a memória de longo prazo.** Quando a conversa acaba, o que dela sobrevive? Essa é a ponte entre esta subseção e o §4.3, e a resposta honesta para muitos cases é *"nada"* — desde que dita.

---

## 5. O grafo, desenhado e implementado

O sistema é um **grafo**: nós que fazem coisas, arestas que decidem o que vem depois, e um estado que atravessa tudo.

> **Qual framework.** A visão geral fala em *workflow com LangChain*; o que se usa é o **LangGraph**, do mesmo ecossistema, porque o modelo mental dele — grafo de estado — é literalmente o que a Aula 05 pediu que vocês desenhassem à mão. A Aula 09 ensina os dois.

### 5.1 O desenho vem antes do código

Em `docs/arquitetura.md`, o grafo desenhado **em ASCII**, pela mesma razão do §2.1: ele vive no `git diff`, e vocês vão alterá-lo. Três coisas precisam ficar legíveis:

- **cada nó**: o que ele faz, e se ele chama o modelo ou não;
- **cada aresta**: incondicional ou condicional, e **quem decide** a condição;
- **os términos**: por onde a execução pode acabar, e não só o caminho feliz.

Um grafo em que todas as arestas são incondicionais não é um grafo — é uma sequência, e provavelmente o case não precisava de framework.

### 5.2 Os padrões, nomeados e justificados

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

### 5.3 Quem decide cada bifurcação

Para **cada aresta condicional**, digam qual dos três degraus decide:

| Degrau | Quando basta | Custo |
|---|---|---|
| **regra em código** | a condição é verificável — um campo, um limiar, um `None` | zero |
| **embedding** | a condição é de assunto, e as rotas são poucas e estáveis | uma chamada barata |
| **modelo** | a condição exige julgamento sobre texto livre | uma chamada cara, **e que erra** |

A ordem é a da Aula 06, e o critério também: use o degrau mais barato que resolve. Um portão que procura um campo vazio não precisa de modelo.

**E quando for modelo, a saída é estruturada** — `Literal` com as rotas válidas, não texto livre. O schema garante que a resposta seja uma das rotas; ele **não** garante que seja a rota certa. Essas são duas coisas diferentes, e quem confunde descobre em produção.

### 5.4 A implementação, e a conferência contra o desenho

O sistema roda sobre `StateGraph`. O que precisa estar no código:

- os nós registrados e as arestas ligadas — condicionais com o mapa de destinos explícito;
- paralelismo, se houver, por arestas que saem do mesmo ponto ou por `Send`;
- o grafo compilado uma vez, e não remontado a cada execução.

E a conferência, que é item de avaliação. Um script curto — `conferir_grafo.py`, na raiz — que **importe o mesmo objeto de grafo que o `src/` executa** e imprima dele duas listas:

- os **nós**, por nome;
- as **arestas**, na forma `origem -> destino`, com as condicionais marcadas.

`grafo.get_graph()` dá as duas sem dependência nenhuma. A saída vai colada em `docs/arquitetura.md`, **ao lado** do desenho do §5.1, junto com o comando que a produziu.

Duas regras sobre essa conferência, e a segunda é a que importa:

- se as duas listas e o desenho divergirem, **o que vale é o código** — e o desenho estava errado;
- o script **importa o grafo do sistema entregue**, e não monta um grafo próprio. Um grafo de demonstração montado dentro do script produz a mesma saída bonita e não prova nada.

> **Sobre paralelismo, um aviso medido em sala.** Despachar nós em paralelo no grafo **não** produz paralelismo se o provedor atende uma requisição por vez. Se vocês afirmarem ganho de tempo, meçam: o total precisa tender ao **maior** dos ramos, e não à **soma** deles. Na Aula 09, com modelo local, deu a soma.

### 5.5 O que o framework não deu

Reimplantem sobre o grafo, e **contem as linhas que isso custou**:

- **o motivo de término** — o nó terminal diz que acabou, não diz por quê;
- **o detector de laço** — `recursion_limit` limita o dano e não detecta; um agente preso na primeira volta gasta o teto inteiro e devolve o diagnóstico errado;
- **os tetos** — dos quatro da Aula 05, verifiquem quais o framework cobre. Provavelmente só o de passos;
- **erro recuperável × fatal** — a classificação continua sendo de vocês.

Uma frente que o framework cobre inteira também conta, e com evidência: `arquivo:linha` mostrando que vocês **não** precisaram escrever aquilo.

### 5.6 A decisão que virou parâmetro

Encontrem, na implementação, **ao menos uma decisão de projeto que o framework converteu em parâmetro com valor padrão** — um limiar, um `k`, um limite de recursão, uma política de repetição, um reducer.

Descrevam: qual era a decisão, como ela estava justificada quando era explícita, e o que acontece com o sistema **se o padrão da biblioteca mudar numa atualização**. Depois declarem o parâmetro no código, mesmo que o valor coincida com o padrão.

Este item separa quem usou o framework de quem entendeu o que ele escondeu.

---

## 6. O estado que atravessa o grafo

Na Aula 05 vocês aprenderam que **a lista de mensagens não é o estado**. No grafo isso deixa de ser argumento e vira declaração de tipo.

### 6.1 Os campos, e quem mexe em cada um

Uma tabela em `docs/arquitetura.md`, com uma linha por campo:

| Campo | Tipo | Quem **escreve** | Quem **lê** | De onde veio |
|---|---|---|---|---|

A coluna "quem escreve" é a que revela defeito: um campo escrito por três nós diferentes, sem reducer, é uma corrida — e o último a terminar ganha.

### 6.2 Os reducers, e o que acontece sem eles

Para cada campo acumulado, digam **qual reducer** e **por quê**. E demonstrem, com uma execução, o que o sistema faz **sem** ele: num grafo com nós paralelos, um campo sem reducer é sobrescrito, e o trabalho dos outros ramos desaparece sem erro nenhum.

Essa demonstração é item de avaliação porque é o defeito mais silencioso do padrão: não levanta exceção, não aparece no log, e só se manifesta como resultado incompleto.

### 6.3 O que NÃO está no estado

Duas listas curtas, e a segunda é a que vale nota:

- **o que ficou na lista de mensagens** e por quê — texto conversacional costuma ficar; decisão, contador e identificador, não;
- **o que é derivado e não guardado** — porque recalcular é mais barato que manter sincronizado, ou porque guardar criaria duas fontes de verdade.

### 6.4 O estado entre agentes

Se o sistema já tem mais de um agente — e não precisa ter, isso é Parte 3 —, declarem **o que um passa ao outro**: o estado inteiro, um subconjunto, ou um objeto de contrato próprio.

Passar o estado inteiro é a escolha fácil e a que mais atrapalha depois: acopla os dois agentes a cada campo novo. Se foi essa a escolha, digam que é provisória e o que a Parte 3 vai ter de mudar.

---

## 7. Entrada e saída

O `README` diz **como usar**. Aqui isso vira contrato, verificável em código.

### 7.1 A entrada é **texto**

**O sistema recebe texto livre, em linguagem natural.** Não formulário com campos, não JSON montado por outro programa, não arquivo enviado no lugar da pergunta.

A razão está na visão geral, e vale repetir: *um sistema que recebe um formulário preenchido e devolve uma resposta não exercita quase nada*. É a entrada em texto que obriga o sistema a **descobrir** o que a pessoa quer — perguntar o que falta, lidar com informação contraditória, decidir quando já sabe o suficiente. Tudo o que o §7.4 cobra depende disso.

Isso não proíbe as outras formas; coloca cada uma no lugar certo:

| Forma | Onde ela entra |
|---|---|
| **texto livre** | **a entrada do sistema** |
| arquivo, planilha, imagem | conteúdo que o agente busca **por ferramenta**, ou corpus do §3 |
| formulário, evento de outro sistema | gatilho que **antecede** o sistema, e que o §7.3 documenta como tal |

Declarem o que acompanha o texto sem ser digitado por ninguém — identificador de usuário, de sessão, de caso —, porque é disso que o `thread_id` do §4.8 costuma ser feito.

### 7.2 O contrato de saída

A saída do sistema é um **objeto validado**, não texto solto — e o schema vive no código, em Pydantic ou equivalente. Três campos são obrigatórios, venham eles com os nomes que vocês quiserem:

- **o resultado** — a resposta, a decisão, o documento;
- **a fonte** — o que sustenta o resultado, conferível (é o §3.4 chegando aqui);
- **a suficiência** — se o sistema teve ou não base para responder. A recusa é parte do contrato, não exceção dele.

### 7.3 Os exemplos, na documentação

Não um exemplo: **um conjunto**, em `docs/io.md`, com o principal repetido no `README`. No mínimo três:

| Exemplo | O que ele mostra |
|---|---|
| **o caminho feliz** | o uso para o qual o sistema foi feito |
| **a recusa** | a saída com suficiência negativa, e o que o sistema diz no lugar da resposta |
| **uma das três do §7.4** | entrada incompleta, ambígua ou fora do escopo |

Cada exemplo traz **as três coisas**, e a terceira é a que falta na maioria das entregas:

1. **a entrada exata**, o texto como a pessoa digitou — com os erros de digitação, se houver;
2. **a saída completa**, o objeto inteiro e não um trecho — incluindo os campos vazios;
3. **uma linha dizendo o que ler ali**: o que naquela saída mostra que o sistema fez a coisa certa.

**Copiados de uma execução**, não escritos à mão. E cada exemplo vem com o **commit** e a **data** da execução que o produziu, e com o log correspondente em `logs/` — é essa amarra que distingue exemplo real de exemplo plausível, e não a aparência do texto.

### 7.4 A entrada que não dá para atender

Três situações, cada uma com um exemplo real de execução:

| Situação | O que o sistema faz |
|---|---|
| entrada **incompleta** | pergunta o que falta, ou recusa dizendo o que falta |
| entrada **ambígua** | desambigua perguntando, ou escolhe e **declara a escolha** |
| entrada **fora do escopo** | recusa, e diz o que ele faz |

Um sistema que responde confiantemente às três não está sendo robusto — está escondendo o problema.

---

## 8. Human-in-the-loop

O humano entra no grafo como **código que para e espera**.

### 8.1 Onde o grafo para, e por qual critério

Marquem no diagrama do §5.1 **todos** os pontos de parada, e para cada um digam o critério. Os três que costumam valer:

| Critério | Exemplo |
|---|---|
| **ação irreversível** | escreve em sistema de terceiro, envia mensagem, move dinheiro |
| **custo acima de um teto** | a ação passa de um valor que vocês declaram |
| **confiança baixa** | o sistema recuperou pouco, ou o avaliador reprovou duas vezes |

Um sistema sem nenhum ponto de parada precisa justificar: ou ele não faz nada irreversível — e aí digam qual é a ação mais perigosa dele —, ou ele deveria parar e não para.

### 8.2 O que a pessoa vê, e o que ela devolve

O que o sistema mostra no momento da pausa é prompt, e vale a regra do §1.3: **a ação exata**, com os argumentos, e a consequência dela. Uma pergunta do tipo *"confirma?"* sem dizer o que vai acontecer transfere a responsabilidade sem transferir a informação.

As respostas aceitas precisam incluir, no mínimo, **aprovar** e **rejeitar**. **Aprovar editando** — a pessoa corrige um argumento antes de deixar seguir — é o que separa aprovação de carimbo, e vale ponto.

### 8.3 A implementação, e por que ela depende do checkpoint

A pausa é um `interrupt` dentro do nó; a retomada é uma nova invocação com `Command(resume=...)`, e o valor devolvido vira o retorno do `interrupt` — a função continua da linha seguinte.

Isso **só funciona porque existe um checkpointer**: a pausa é um checkpoint gravado, endereçado por `thread_id`. Digam qual checkpointer usaram e onde ele grava — e se for em memória, o sistema perde a aprovação pendente quando o processo morre, o que precisa estar declarado como limitação.

### 8.4 O que acontece se ninguém responder

Um prazo, e o que ocorre quando ele vence: a execução expira, é escalada, ou segue por um caminho padrão declarado. **Seguir em frente silenciosamente não é opção** — é o ponto de parada virando enfeite.

### 8.5 Aprovar duas vezes

Demonstrem que a ação **não acontece duas vezes** se a aprovação chegar repetida — por chave de idempotência derivada do conteúdo, como a Aula 05 pede para qualquer escrita. Com log das duas chamadas e a prova de que a segunda não criou nada.

---

## 9. O carimbo

Toda execução registra, no início, a combinação que a produziu. São **dez campos** acumulados até aqui:

| Campo | Desde | Por que ele invalida a comparação |
|---|---|---|
| versão do prompt | 03 | outro texto, outro comportamento |
| modelo | 03 | outro motor |
| parâmetros | 03 | `temperature` e `top_p` mudam a distribuição |
| arquitetura da etapa | 05 | trocar *workflow* por agente é mudança de versão |
| estratégia de corte | 06 | trocar o corte muda o que é recuperado, e a resposta com ele |
| modelo de embedding | 06 | reindexar com outro modelo muda a recuperação **sem ninguém mudar o código** |
| `k` | 07 | mais ou menos contexto, outra resposta |
| versão do prompt de resposta | 07 | é um segundo prompt, com vida própria |
| estado da memória | 08 | duas execuções com memórias diferentes **não são comparáveis** |
| versão do framework | 09 | o framework converte decisão em padrão, e **o padrão muda numa atualização** |

A última linha é a que os grupos esquecem, e é a que o §5.6 explica: uma parte do comportamento do sistema passou a morar na biblioteca. Fixem a versão no `requirements.txt`, como qualquer dependência.

> **Dois campos a mais chegam na Parte 3**, com o MCP: a revisão da especificação e a versão do servidor consumido. Eles têm natureza diferente destes dez — aqui, quem muda o carimbo é quem roda o experimento; num servidor de terceiro, **quem muda é outra pessoa**.

Se algum campo não existir porque o case não usa aquela peça, **digam isso explicitamente** em vez de omitir a linha.

---

## 10. Os casos demonstrados

**Oito execuções, com log no repositório:**

| # | O caso | O que ele prova | Onde isto é exigido |
|---|---|---|---|
| 1 | o caso simples | o caminho feliz existe | §7.3 |
| 2 | a **divergência** | o sistema diz uma coisa, o usuário diz outra | §7.4 |
| 3 | o **registro inexistente** | erro de ferramenta que o modelo contorna | §2.2 |
| 4 | o caso que **não** deve disparar a ação principal | o sistema sabe não agir | §2.3 |
| 5 | a pergunta **fora do corpus** | o portão recusa, e recusa dizendo algo útil | §3.5 |
| 6 | a **contradição** entre dois fatos verdadeiros | o desempate por carimbo de tempo funciona | §4.5 |
| 7 | a mesma entrada com **memórias diferentes** | a não reprodutibilidade, registrada | §4.7 |
| 8 | a ação que **para e espera** | o grafo pausa, a pessoa aprova editando, e a segunda aprovação não duplica nada | §8.2, §8.5 |

Os casos 5 a 8 são os que os grupos esquecem, e são os que provam as frentes desta entrega. Um log de oito execuções em que as oito dão certo pelo caminho feliz não demonstra nada.

---

## 11. O projeto como entrega

Cinco exigências, e nenhuma delas é sobre o código em si.

**`README.md`** — duas perguntas, e a segunda é a que costuma faltar: **como rodar** (do zero, por quem nunca viu o projeto, em menos de cinco minutos) e **como usar**, com o exemplo principal do §7.3 — **o texto que a pessoa digita** e a saída inteira que volta — e **o que acontece quando o grafo para e espera alguém**.

**`requirements.txt`** com todas as versões fixadas — inclusive a do framework, pela razão do §9.

**`.env.example`** com os nomes das variáveis e nenhum valor. Chave de API **nunca** no repositório.

**`docs/`** — toda a pesquisa e documentação, em Markdown, versionada. O `case.md` **complementado**, o `prompts.md`, o `arquitetura.md` — que carrega o grafo, a tabela de padrões, a tabela de campos do estado e os pontos de parada — e os resultados de medição de cada frente. O histórico dessa pasta é o que mostra **quando o grupo mudou de ideia sobre o próprio case, e por quê**.

**`src/prompts/`** — os prompts em arquivo, versionados, junto do código que os carrega. Prompt embutido no código não é aceito.

---

## Como esta parte será avaliada

Em ordem de peso:

| O que se avalia | O que se espera |
|---|---|
| **RAG que funciona e que recusa** | as dez perguntas escritas e classificadas · **as formas usadas, cada uma com o que ela responde no case** · corpus declarado e curado · **o corte e o `k` declarados, com a razão** · citação **verificada em código** · o portão implementado, com a troca entre os dois erros declarada · **a expansão demonstrada, com e sem corpus** |
| **Memória, e o que ela não guarda** | a fronteira com checkpoint e RAG declarada no case · as três memórias nas estruturas certas · política de escrita com volume declarado · **a lista do que não entra** · a contradição resolvida em código · o esquecimento demonstrado |
| **O grafo, desenhado e defendido** | o desenho em ASCII, com nós, arestas e términos · os padrões da Aula 05 **nomeados e justificados** · o degrau de decisão de cada bifurcação justificado · **as listas de nós e arestas extraídas do grafo que o `src/` executa**, batendo com o desenho · as peças que o framework **não** deu · a decisão que virou parâmetro |
| **O estado, e os reducers** | a tabela de campos com quem escreve e quem lê · o reducer de cada campo acumulado, justificado · **a execução que demonstra o que acontece sem ele** · as duas listas do que não está no estado |
| **Human-in-the-loop** | os pontos de parada marcados no diagrama, com critério · o que a pessoa vê, com a ação e a consequência · **aprovar editando** funcionando · o prazo e o que acontece quando vence · **a segunda aprovação que não duplica nada** |
| **Entrada e saída** | **a entrada é texto livre** · contrato de saída validado, com fonte e suficiência · **os três exemplos documentados**, cada um com entrada exata, saída inteira e o que ler nela · cada exemplo amarrado a commit, data e log · as três entradas que não dão para atender |
| **O plano de prompt engineering** | uma linha por etapa, com técnica **nomeada e justificada** · contrato de saída exato · como cada etapa é testada, com denominador · prompts em arquivo, versionados · a regra da frase aplicada em ao menos um prompt |
| **A arquitetura documentada** | diagrama com quem decide onde · as seis perguntas respondidas · **o nível de autonomia defendido contra o de baixo, apontando a decisão no diagrama** |
| **O complemento do case** | **o domínio escrito** — vocabulário, regra tácita, exceção que decide arquitetura · o escopo declarado como está hoje · **o verificador descrito com o sistema rodando** · a linha de base do ganho prometido |
| **O carimbo** | dez campos registrados a cada execução · campos ausentes **declarados**, não omitidos |
| **A entrega como projeto** | roda do zero em <5 min · o `README` diz **como usar**, com exemplo real · `docs/` completo e versionado · nenhuma chave no repositório |
| **Os oito casos** | executados, com log · os casos 5 a 8 presentes e demonstrando o que devem |

E o que **não** conta: quantidade de código, número de ferramentas, número de documentos no corpus, sofisticação visual.

**Resultados negativos bem medidos contam a favor**, e vale repetir porque os grupos não acreditam: "o corte por estrutura não serviu ao nosso corpus, e aqui está por quê", "recusamos o padrão avaliador-otimizador, e aqui está por quê", e "o paralelismo do grafo não reduziu o tempo, porque o provedor serializa" são entregas boas. Medição que contraria a expectativa é o produto mais valioso de uma engenharia honesta.

---

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
  rag.md             formas usadas, perguntas, corte, k, limiar, o corpus revogado
  memoria.md         a fronteira, a política, O QUE NÃO ENTRA, a contradição
  grafo.md           o desenho, os padrões, o estado, o que o framework não deu
  humano.md          os pontos de parada, o critério, o prazo, a idempotência
  io.md              os schemas e os três exemplos, com commit, data e log
  fontes.md          tudo que foi consultado, com link

src/               o sistema
  prompts/         os prompts, versionados — junto do código que os carrega
dados/             o corpus e os dados simulados
logs/              as 8 execuções demonstradas
```

Markdown, sempre — nada de `.docx` nem `.pdf`, para que o `git diff` funcione.

---

## Dicas

- **Comecem pelo item 0.** Ele leva uma hora e pode poupar três semanas — e a primeira pergunta é a que mais rende: o que vocês aprenderam do domínio nas últimas semanas se perde se ninguém escrever. Se o verificador não sobreviveu, é agora que se troca.

- **A lista do §4.4 — o que a memória não guarda — antes de escrever a memória.** É mais fácil decidir o que guardar depois de ter a lista de exclusão, e essa lista é o item de maior peso da frente.

- **Escrevam as dez perguntas do §3.3 antes de indexar qualquer coisa.** Quem indexa primeiro escreve perguntas que o índice já responde. Se qualquer estratégia de corte responder a todas, o conjunto é fácil demais.

- **Desenhem o grafo do §5.1 no papel antes de abrir o editor.** O desenho leva vinte minutos e revela os términos que vocês não tinham pensado. Depois confiram contra as listas que o `conferir_grafo.py` extrai do código: se diferirem, o desenho estava errado.

- **Quebrem o reducer de propósito, uma vez.** A execução do §6.2 — o campo sobrescrito, sem erro nenhum — é a única forma de o grupo inteiro entender por que ele existe.

- **O ponto de parada do §8.1 vem da ação mais perigosa do sistema, não da mais incerta.** Parar para confirmar uma consulta é teatro; parar antes de escrever no mundo é projeto.

- **Os casos 5 a 8 do item 10 são os que provam as frentes novas.** Se o log tiver oito execuções bem-sucedidas pelo caminho feliz, ele não demonstrou nada.

- **A integração do §5.1 costuma virar servidor com pouco mais que um decorador.** O trabalho não é esse: é a descrição, o erro, a idempotência e a conta.

- **Guardem o número de tudo que medirem.** A Parte 3 vai comparar o sistema no ar com estes valores, e refazer a medição depois custa caro.
