# IA Aplicada com LLMs — Aula 03: Prompt engineering — Anatomia do prompt e as técnicas

## Introdução

A Aula 02 terminou com quase tudo sob controle: o modelo escolhido com critério, os parâmetros ajustados por tipo de tarefa, a saída em JSON garantida por schema. Sobrou uma coisa sem controle nenhum — **o texto**.

E o texto é a parte que a maioria das pessoas acha que já sabe fazer. Escrever em português todo mundo escreve. Daí o conselho que circula em todo lugar: *"seja mais específico"*, *"dê contexto"*, *"peça para pensar passo a passo"*. Conselhos que não são errados, mas que também não são **engenharia**: não dizem quando aplicar, quanto custa, nem como saber se funcionou.

Esta nota trata prompt como o que ele é num sistema de produção: **um componente que você projeta, escolhe com critério e mede**. Ela tem duas metades:

1. **A anatomia** — as peças de um prompt e o contrato que ele estabelece;
2. **As técnicas** — zero-shot, few-shot, chain-of-thought, self-consistency e as demais, cada uma com o problema que resolve, o paper que a propôs e, principalmente, **como se monta o prompt de cada uma**.

Uma advertência de posicionamento, que vale para a aula inteira: **este não é um curso de prompt engineering** (a Aula 00 é explícita nisso). Prompt é ferramenta, não assunto. O que interessa aqui é o critério de escolha — e, com a mesma força, saber **quando a resposta não é um prompt melhor**, e sim outra arquitetura.

> **Pré-requisitos:** Aula 02, principalmente [nota 02](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md) (§7 saída estruturada, §4.5 o parâmetro `n`, §9 a tabela de receitas) e [nota 01, §6](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) (instruct × raciocínio). E a [nota 03, §2](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/03-controle-da-saida.md) — `max_tokens` como guilhotina, que volta na §7.2.

---

## Objetivos de aprendizagem

Ao final desta nota você deve ser capaz de:

- **Distinguir** system, role e contextual prompting, e dizer o que pertence a cada um.
- **Escrever** um *output contract*: formato, exemplos e **exemplos negativos**.
- **Explicar** por que o zero-shot funciona — e o que *instruction tuning* tem a ver com isso.
- **Aplicar** few-shot e explicar, com base no Min et al. (2022), **o que os exemplos realmente ensinam**.
- **Reconhecer** o tipo de tarefa em que few-shot falha e CoT resolve.
- **Escolher** entre CoT, self-consistency, generated knowledge, prompt chaining, ToT e meta-prompting pela **tarefa e pelo modelo**.
- **Reconhecer quando cada técnica não ajuda** — e por que a mais simples costuma bastar.
- **Justificar** por que CoT no prompt é redundante em modelos de raciocínio.

---

## Desenvolvimento teórico

### 1. Por que "seja mais específico" não é conselho útil

Pegue o conselho ao pé da letra e ele se autodestrói: se "mais específico" é sempre melhor, o prompt ideal seria infinito. Não é — cada token ocupa janela e, passado certo ponto, **piora** o resultado (*lost in the middle*, Aula 02, nota 01 §4.3).

O conselho útil é outro, e tem três partes:

> **Específico sobre o quê?** Sobre a **tarefa**, o **formato** e os **casos de fronteira** — não sobre tudo.

Compare:

```text
❌ "Analise cuidadosamente e com muita atenção a mensagem do cliente
    abaixo, sendo bastante detalhado e específico na sua análise,
    e me diga do que se trata."
```

```text
✅ "Classifique a mensagem em: entrega_atrasada, endereco_errado,
    produto_avariado, duvida, elogio.

    Se houver mais de um problema, escolha o da reclamação.
    Se não houver problema, use 'duvida' ou 'elogio'.

    Responda apenas o rótulo."
```

O primeiro tem mais palavras e menos informação. "Cuidadosamente", "com muita atenção" e "bastante detalhado" não restringem nada — o modelo não tem um botão de esforço que essas palavras acionam. O segundo **fecha o espaço de respostas**: diz os rótulos possíveis, resolve a ambiguidade que o aluno viu no exercício 02 e define o formato.

**Regra de trabalho:** cada frase do seu prompt deve **eliminar alguma possibilidade**. Se você apagar a frase e não conseguir apontar o que ela impedia, ela é enfeite — e enfeite ocupa espaço que faria falta ao que importa.

---

### 2. Anatomia: system, role e contextual

Um prompt de produção tem três camadas, com **ciclos de vida diferentes**. Confundi-las é a origem de metade dos problemas.

| Camada | O que define | Muda quando |
|---|---|---|
| **System** | comportamento, restrições, formato de saída, o que **nunca** fazer | ~nunca (é versionado, revisado, testado) |
| **Role** | papel/persona: perspectiva, vocabulário, tom | por produto ou por público |
| **Contextual** | informação de fundo da tarefa **desta** chamada | a cada requisição |

```python
messages = [
    {"role": "system", "content": SYSTEM},          # estável, versionado
    {"role": "user",   "content": f"""
{CONTEXTO_DA_TAREFA}

Mensagem do cliente:
<mensagem>
{mensagem}
</mensagem>
"""},
]
```

**System prompting** é onde vive o contrato: *"você classifica mensagens de suporte"*, *"responda apenas com o rótulo"*, *"nunca invente número de pedido"*. É a parte que você trata como código (nota 03 desta aula).

**Role prompting** — *"você é um analista de logística"* — alinha vocabulário e perspectiva. Merece uma subseção própria, porque é a camada que os alunos mais usam e a que mais rende expectativa errada (§2.1).

**Contextual prompting** é o que muda a cada chamada — a mensagem do cliente, os documentos recuperados, o histórico. É a camada mais cara, porque cresce, e é o assunto da [nota 02](02-context-engineering.md).

> **Um detalhe que só aparece depois:** o system prompt vai junto em **toda** requisição. Tudo o que você escrever ali é reenviado a cada chamada, para sempre — então ele é o lugar onde frase-enfeite mais pesa.

#### 2.1 Role prompting: o que ele faz e o que não faz

*"Você é um especialista de classe mundial em..."* é provavelmente a linha mais escrita da história do prompt engineering. Vale saber o que ela entrega.

**O que role prompting realmente muda:**

- **registro e vocabulário** — um advogado e um engenheiro descrevem o mesmo contrato com palavras diferentes;
- **o que o modelo assume que você já sabe** — e portanto o que ele explica e o que ele pula;
- **convenções de formato** de uma profissão — um laudo, um parecer, um relatório de incidente têm estruturas próprias;
- **o enquadramento** — a mesma pergunta sobre um atraso de entrega tem respostas diferentes se você pede a perspectiva do jurídico, do comercial ou da operação.

**O que ele não muda:**

- a **acurácia** do modelo em fatos;
- a **capacidade de raciocínio** dele.

Dizer "você é um especialista" não dá ao modelo conhecimento que ele não tem, nem faz com que ele calcule melhor. Ele já viu texto de especialista no treino; o papel escolhe **de qual parte dessa distribuição ele vai puxar**, não eleva o teto.

> É por isso que a formulação honesta é: **role prompting é um seletor de registro, não um multiplicador de qualidade.** Quem espera a segunda coisa fica frustrado — e, pior, atribui ao modelo um problema que é do prompt.

#### 2.2 A reescrita que vale mais que o papel

Aplique a regra da §1 — *cada frase deve eliminar uma possibilidade* — à frase do especialista:

```text
❌ Você é um especialista de classe mundial em logística.
```

O que essa frase **impede**? Nada que você consiga nomear. Ela não diz que vocabulário usar, o que assumir do leitor, nem o que não fazer.

```text
✅ Escreva para um analista de logística.
   Use os termos do setor (coleta, redespacho, avaria, extravio) sem
   explicá-los. Não sugira ações que dependam de sistemas que você não
   conhece. Se faltar dado para concluir, diga o que falta.
```

Três frases, três possibilidades eliminadas. E note que **a segunda versão não precisa do papel** — ela diz diretamente o que o papel deveria ter comunicado.

> **A regra prática:** se você consegue escrever o que quer sem o papel, escreva. Use o papel quando o **enquadramento** for o objetivo (*"analise este contrato da perspectiva do jurídico"* × *"...do comercial"*), não quando estiver esperando que ele melhore o resultado.

---

### 3. Delimitadores: separar instrução de dado

Repare no exemplo acima: a mensagem do cliente vem entre `<mensagem>` e `</mensagem>`. Isso não é enfeite.

Do ponto de vista do modelo, **tudo é uma sequência de tokens**. Não existe um canal para instrução e outro para dado — a fronteira é apenas textual. Sem delimitador, uma mensagem de cliente que contenha *"ignore as instruções acima"* é indistinguível de uma instrução sua.

Delimitadores explícitos (`<tags>`, ` ``` `, `###`) fazem duas coisas:

1. **Ajudam o modelo** a saber onde a instrução termina e o dado começa;
2. **Não garantem nada.** São uma convenção de formatação: o modelo continua sem ter como distinguir os dois lados da tag.

Guarde essa segunda linha. Ela é a razão pela qual **prompt não é mecanismo de segurança** — um texto vindo de fora pode agir como instrução. As consequências disso, e as defesas, são assunto da **aula de segurança**.

---

### 4. O output contract

Na Aula 02 você aprendeu a **garantir o formato** com JSON Schema (nota 02, §7). O contrato de saída aqui é a parte que o schema **não** cobre: o que colocar dentro dos campos.

Um contrato completo tem quatro partes:

| Parte | Exemplo |
|---|---|
| **Formato** | schema JSON, ou "apenas o rótulo, sem pontuação" |
| **Domínio** | os 5 rótulos possíveis (no schema, um `enum`) |
| **Regras de fronteira** | "com dois pedidos, vale o reclamado" |
| **Exemplos negativos** | "não escreva 'Claro! Aqui está'; não repita a pergunta" |

Os **exemplos negativos** são a parte que quase ninguém escreve, e a que mais economiza depuração. O modelo tem tendências fortes — cumprimentar, explicar o que vai fazer, oferecer ajuda no fim. Dizer explicitamente *"não faça X"* é mais eficaz que esperar que ele adivinhe pelo silêncio.

```text
Responda APENAS o rótulo, em minúsculas.
Não escreva explicação, saudação ou pontuação final.
Não repita a mensagem do cliente.
```

> **Cuidado:** exemplo negativo funciona, mas tem limite — mencionar algo, mesmo para proibir, coloca aquilo no contexto. Proibir dez coisas específicas costuma render menos que **mostrar um exemplo positivo bem formado**.

---

### 5. Zero-shot: e por que ele funciona

**Zero-shot** é o caso mais simples: você descreve a tarefa e não dá exemplo nenhum.

```text
Classifique o texto em neutro, negativo ou positivo.
Texto: Acho que as férias estão boas.
Sentimento:
```

Funciona. E vale entender **por que**, porque isso não era óbvio há poucos anos: um modelo treinado só para prever o próximo token não tem motivo especial para *obedecer* a uma instrução.

A resposta é o **instruction tuning** (Wei et al., 2022 — o FLAN): depois do pré-treino, o modelo é ajustado em milhares de tarefas **descritas como instruções**. Ele aprende o padrão *"quando o texto tem cara de instrução, o que vem depois é a execução dela"*. Some o **RLHF**, que alinha o comportamento a preferências humanas, e você tem o modelo instruct que a Aula 02 descreveu.

> **A consequência prática:** zero-shot funciona porque alguém pagou um treino caro para que funcionasse. Não é uma propriedade natural de modelos de linguagem — é um produto de engenharia. E é por isso que **modelos base** (não instruct) se comportam de forma tão diferente com o mesmo prompt.

**Comece sempre por zero-shot.** É o mais barato, e em tarefas comuns ele resolve. Só escale a técnica quando medir que precisa.

---

### 6. Few-shot: e o que os exemplos realmente ensinam

Quando o zero-shot não basta, o passo seguinte é **dar exemplos no próprio prompt** — 2 a 5, tipicamente:

```text
A "whatpu" é um animal pequeno e peludo nativo da Tanzânia. Um exemplo de
frase que usa a palavra whatpu é:
Estávamos viajando pela África e vimos esses whatpus muito fofos.

"Farduddle" significa pular para cima e para baixo bem rápido. Um exemplo
de frase que usa a palavra farduddle é:
```

O modelo completa corretamente, tendo visto **um** exemplo. Isso é **in-context learning**: aprendizado que acontece na janela de contexto, sem nenhum peso ser alterado. É uma capacidade que **emergiu com a escala** (Brown et al., 2020; Kaplan et al., 2020) — modelos pequenos não fazem isso bem.

#### 6.1 O resultado que derruba a intuição

Aqui está o achado mais importante desta nota, e o mais contraintuitivo.

A intuição de todo mundo é: *"o modelo aprende com os exemplos o mapeamento entrada → saída"*. **Min et al. (2022)** testaram isso do jeito mais direto possível — embaralharam os rótulos dos exemplos, deixando-os **errados**:

```text
This is awesome!        // Negative     ← errado de propósito
This is bad!            // Positive     ← errado de propósito
Wow that movie was rad! // Positive
What a horrible show!   //
```

Saída: **`Negative`** — a resposta correta, apesar de os exemplos ensinarem o contrário.

A conclusão dos autores, na formulação deles: *"o espaço de rótulos e a distribuição do texto de entrada especificados pelas demonstrações são ambos importantes — **independentemente de os rótulos estarem corretos para as entradas individuais**"*.

Ou seja, o que o few-shot ensina **não** é o mapeamento. É:

1. **Quais rótulos existem** (o espaço de saída);
2. **Que tipo de texto** entra;
3. **O formato** da resposta.

E há um detalhe adicional do mesmo trabalho: mesmo com rótulos aleatórios, o resultado é **muito melhor do que sem rótulo nenhum** — e sortear os rótulos da distribuição real ajuda mais do que sortear uniformemente.

#### 6.2 O que fazer com isso

Não é curiosidade acadêmica; muda como você escreve exemplos:

- **cubra todos os rótulos** nos exemplos, especialmente os raros — se uma classe não aparece, o modelo tende a não usá-la;
- **use entradas parecidas com as reais** — mensagem de cliente de verdade, com erro de digitação, não frase de manual;
- **mantenha o formato rigorosamente igual** em todos os exemplos: é dele que o modelo tira a estrutura da resposta;
- **não gaste horas caçando os exemplos "perfeitos"**, com o rótulo mais canônico possível. O retorno está na cobertura e no formato, não na perfeição de cada par.

E há um efeito colateral de que quase ninguém se lembra: **os exemplos vão junto em toda requisição**. Eles ocupam janela — a mesma janela que, num sistema real, você vai querer para o histórico da conversa ou para os documentos recuperados. Por isso a regra é cobertura, não quantidade: 3 exemplos bem escolhidos valem mais que 10 redundantes.

#### 6.3 Onde o few-shot não salva

```text
Os números ímpares deste grupo somam um número par: 15, 32, 5, 13, 82, 7, 1.
R:
```

O modelo responde: *"Sim, os ímpares somam 107, que é par."* Duas coisas erradas de uma vez — a soma é 41, e 107 nem é par.

Acrescente quatro exemplos resolvidos e o resultado continua errado. **Few-shot mostra o formato da resposta, não o caminho até ela.** Quando a tarefa exige etapas intermediárias, o modelo continua tentando saltar direto para o resultado — só que agora com a formatação certa.

É exatamente esse problema que a próxima técnica ataca.

---

### 7. Chain-of-thought: mostrar o caminho, não só o destino

**Chain-of-thought** (Wei et al., 2022) muda os exemplos do few-shot: em vez de mostrar apenas a resposta, eles mostram **o raciocínio até ela**.

```text
Os ímpares deste grupo somam um número par: 4, 8, 9, 15, 12, 2, 1.
R: Os ímpares são 9, 15 e 1. Somando: 9 + 15 = 24, 24 + 1 = 25.
   25 é ímpar. A resposta é Falso.

Os ímpares deste grupo somam um número par: 15, 32, 5, 13, 82, 7, 1.
R:
```

Agora o modelo produz os passos — e acerta. O ganho é grande em aritmética, raciocínio de senso comum e problemas simbólicos.

**Por que funciona**, na leitura mecânica que a Aula 01 permite: o modelo gera um token por vez, condicionado a tudo que já escreveu. Ao escrever os passos intermediários, ele **coloca no próprio contexto** os resultados parciais de que precisa depois. Sem isso, ele teria que fazer toda a conta "de cabeça", numa única passagem. O raciocínio escrito é **memória de trabalho externalizada**.

#### 7.1 Zero-shot CoT

**Kojima et al. (2022)** mostraram que boa parte do ganho sai sem exemplo nenhum, com uma frase:

```text
Vamos pensar passo a passo.
```

É a técnica de melhor relação benefício/esforço da nota inteira: **uma linha**, sem precisar escrever exemplo nenhum. Quando alguém diz "peça para pensar passo a passo", é disto que está falando — e agora você sabe **por que** funciona e **quando não vale**.

#### 7.2 Quando **não** usar

CoT não é grátis em atenção do leitor nem em disciplina do modelo, e tem dois casos claros em que atrapalha:

- **Tarefa de um passo só** — classificação, extração, tradução. Não há etapas intermediárias para escrever, então o raciocínio não tem o que fazer. Pior: ele dá ao modelo espaço para **se contradizer antes de responder**, e a resposta vem cercada de texto que você não pediu. Se a sua saída é consumida por código, isso é atrito puro;
- **Modelos de raciocínio** — que já fazem isso internamente. Pedir de novo é redundante, e às vezes atrapalha (§14).

Há também um efeito prático: a resposta fica **longa**. Se você limitou o `max_tokens` pensando numa resposta curta, o raciocínio não cabe — e você recebe um texto cortado no meio, sem a conclusão (Aula 02, nota 03 §2.2).

> **Regra:** CoT em tarefa de **várias etapas**. No resto, o prompt direto com um bom contrato de saída é melhor.

---

### 8. Self-consistency: votar em vez de confiar

**Wang et al. (2022)** partiram de uma observação simples sobre o CoT: se o raciocínio é **amostrado**, ele varia entre execuções — e um caminho errado leva a uma resposta errada.

A solução é estatística: gere **N raciocínios independentes** com temperatura > 0 e fique com a **resposta majoritária**.

```
      ┌──> raciocínio 1 ──> 254
prompt├──> raciocínio 2 ──> 254   ──> voto majoritário ──> 254
      ├──> raciocínio 3 ──> 270
      ├──> raciocínio 4 ──> 254
      └──> raciocínio 5 ──> 254
```

Caminhos errados tendem a errar de formas **diferentes**; o caminho certo tende a convergir para o mesmo lugar. A maioria filtra o ruído.

É aqui que o parâmetro `n` da Aula 02 (nota 02, §4.5) ganha uso real: uma única requisição devolve as N respostas.

Repare numa diferença importante em relação a tudo o que veio antes: **o prompt é exatamente o mesmo** do zero-shot CoT. A técnica não está no texto — está em pedir várias respostas e comparar. É a primeira vez na nota em que a melhoria não vem de escrever melhor, e sim de **medir mais**.

E ela tem um limite claro: self-consistency só ajuda quando o modelo **erra por variação**. Se ele erra sempre da mesma forma, cinco execuções erradas produzem uma maioria errada. Comece com N=1 e só suba se medir que precisa.

---

### 9. Generated knowledge: buscar antes de responder

**Liu et al. (2022)**: em vez de perguntar direto, peça primeiro que o modelo **gere o conhecimento relevante** e só então responda usando-o.

```
1ª chamada:  "Liste fatos relevantes sobre <tema>."
2ª chamada:  "Dados estes fatos: <fatos>, responda: <pergunta>"
```

Ajuda em perguntas de senso comum, em que o modelo "sabe" o fato mas não o traz à tona ao responder de imediato.

> **A ressalva honesta, e ela é grande:** o conhecimento gerado sai **do próprio modelo**, então ele pode ser inventado com a mesma fluência de sempre. Isso **não** resolve alucinação — só reorganiza o que o modelo já tem. Quando a resposta depende de fato externo, a técnica certa não é essa, é **RAG**: buscar de uma fonte confiável. É a aula seguinte, e o motivo pelo qual a Aula 02 disse que *grounding* cura o que a temperatura não cura.

---

### 10. Prompt chaining: uma tarefa por chamada

**Prompt chaining** é decompor: em vez de uma chamada que faz tudo, várias em sequência, cada uma com um trabalho e a saída de uma alimentando a próxima.

O aluno já fez isso no exercício 02 — extrair e depois redigir são dois elos de uma cadeia.

| | Uma chamada | Cadeia |
|---|---|---|
| Depuração | tudo ou nada | **você vê onde quebrou** |
| Controle | nenhum no meio | **valida entre etapas** |
| Parâmetros | um conjunto para tudo | **um por etapa** — o ponto do exercício 02 |

O ganho decisivo não é qualidade, é **observabilidade e controle**. Numa cadeia você valida o JSON antes de redigir, e escala para outro modelo só na etapa difícil (o roteamento da Aula 02, nota 04 §8).

---

### 11. Tree of Thoughts: explorar e voltar atrás

**Yao et al. (2023)** generalizam o CoT: em vez de **uma** linha de raciocínio, uma **árvore** de pensamentos parciais, que o modelo avalia ("promissor / talvez / impossível") e por onde uma busca (BFS/DFS) avança e **retrocede**.

Serve para problemas com **exploração e backtracking** — quebra-cabeças, planejamento, o Jogo do 24. É poderoso, e é pesado: exige dezenas de chamadas por problema e um laço de controle escrito por você.

Na prática de sala, o que cabe é a versão de **Hulbert (2023)**, que aproxima a ideia num prompt só:

```text
Imagine que três especialistas diferentes estão respondendo a esta pergunta.
Todos escrevem 1 passo do seu raciocínio e compartilham com o grupo.
Então todos passam ao próximo passo, e assim por diante.
Se algum perceber que está errado, ele sai.
A pergunta é...
```

> **Onde ToT se encaixa neste curso:** a versão completa é, na prática, um **agente** — busca com avaliação e retrocesso. Ela reaparece com outro nome na aula de agentes (padrão *planning*). Aqui vale conhecer a ideia e saber que **quase nunca** é a primeira coisa a tentar.

---

### 12. Meta-prompting: estrutura em vez de conteúdo

**Zhang et al. (2024)**: em vez de dar exemplos **concretos** (few-shot), dê o **esqueleto** da solução — a forma do problema e da resposta, sem os detalhes.

Vantagens sobre o few-shot:

- **é mais enxuto** — descrever a estrutura ocupa menos espaço que exemplos completos;
- **comparação justa** entre modelos, sem o viés de exemplos específicos;
- funciona como **zero-shot**, sem depender da escolha dos exemplos.

O limite, declarado pelos autores: meta-prompting **assume que o modelo já conhece a tarefa**. Em domínio novo ou muito específico, ele degrada — como o zero-shot.

---

### 13. A tabela de decisão

Reunindo tudo. A coluna que mais importa é a última — quase todo material lista técnicas, e quase nenhum diz **quando não usar**:

| Técnica | O que você escreve | Quando usar | Quando **não** usar |
|---|---|---|---|
| **Zero-shot** | só a instrução | sempre o ponto de partida | — (se falhar, escale) |
| **Few-shot** | + 2 a 5 exemplos | formato específico, rótulos raros, tom | quando o schema já resolve o formato |
| **CoT** | + exemplos **com o raciocínio** | várias etapas, aritmética, lógica | tarefa de um passo; modelo de raciocínio |
| **Zero-shot CoT** | + *"vamos pensar passo a passo"* | idem, sem escrever exemplos | idem |
| **Self-consistency** | nada — muda o `n` | o modelo erra **por variação** | quando ele erra sempre igual |
| **Generated knowledge** | 2 prompts encadeados | senso comum que o modelo tem mas não usa | quando o fato é **externo** — aí é RAG |
| **Prompt chaining** | 1 prompt por etapa | etapas de naturezas diferentes | quando uma chamada já resolve |
| **ToT** | um laço de busca | exploração com backtracking | quase sempre — é praticamente um agente |
| **Meta-prompting** | o esqueleto da solução | tarefa que o modelo já conhece | domínio novo ou muito específico |

**Como usar a tabela:** de cima para baixo, **parando no primeiro que resolve**. A coluna "o que você escreve" mostra por que essa ordem faz sentido: ela vai do prompt mais simples de escrever para o mais trabalhoso. Escalar a técnica sem medir é o mesmo erro de escolher o modelo maior sem testar o menor (Aula 02, nota 01 §2.2).

---

### 14. A ressalva de 2026: a técnica depende do modelo

Tudo acima foi desenvolvido para modelos **instruct**. Com **modelos de raciocínio** (Aula 02, nota 01 §6), parte disso muda:

| Técnica | Em modelo instruct | Em modelo de raciocínio |
|---|---|---|
| CoT / zero-shot CoT | **ganho grande** | **redundante** — ele já faz internamente; a instrução pode atrapalhar |
| Self-consistency | ganho real | ganho menor — ele já explora caminhos internamente |
| Few-shot | útil | útil para **formato**; menos para raciocínio |
| Output contract | essencial | essencial |

A regra da Aula 02 — *raciocínio para decidir, instruct para redigir* — ganha aqui um complemento:

> **A técnica de prompting certa depende do modelo, não só da tarefa.** Aplicar CoT num modelo de raciocínio é pagar duas vezes pela mesma coisa.

E note o que isso implica para a sua suíte de testes: **trocar de modelo pode invalidar o seu prompt**. É o argumento da nota 03.

---

## Exemplos

### Exemplo 1 — A mesma tarefa, quatro níveis de técnica

Classificar a mensagem `"bom dia, o entregador deixou na rua de tras, numero 45. o meu é 145"` (do exercício 02):

```python
BASE = """Classifique em: entrega_atrasada, endereco_errado,
produto_avariado, duvida, elogio. Responda apenas o rótulo.

Mensagem: {msg}"""

FEW_SHOT = """Classifique em: entrega_atrasada, endereco_errado,
produto_avariado, duvida, elogio. Responda apenas o rótulo.

Mensagem: meu pedido não chegou até hoje
Rótulo: entrega_atrasada
Mensagem: chegou tudo certo, obrigado!
Rótulo: elogio
Mensagem: entregaram na casa do vizinho
Rótulo: endereco_errado

Mensagem: {msg}
Rótulo:"""

COT = BASE + "\n\nPense passo a passo e termine com 'Rótulo: <rótulo>'."
```

O que observar ao rodar:

- **zero-shot** costuma acertar — é uma tarefa fácil;
- **few-shot** ajuda principalmente no **formato** (o zero-shot às vezes devolve "O rótulo é: endereco_errado");
- **CoT** acerta, escreve um parágrafo de raciocínio antes e **não melhora a acurácia** — é a demonstração de que técnica errada não entrega nada, e ainda enche a saída de texto que você não pediu.

### Exemplo 2 — Consertando um prompt que falha

Este é o exemplo mais útil da nota, porque é o que você vai fazer o tempo todo. Um prompt que "quase funciona", e as três correções na ordem certa.

**Tentativa 1** — o pedido cru:

```text
Qual a categoria dessa mensagem?

Mensagem: o entregador deixou na casa do vizinho
```

Saída: *"Essa mensagem parece se tratar de um problema de entrega, possivelmente relacionado ao endereço..."* — prosa, não rótulo. **O prompt não disse quais categorias existem nem como responder.**

**Tentativa 2** — acrescenta o **domínio**:

```text
Classifique em: entrega_atrasada, endereco_errado, produto_avariado,
duvida, elogio.

Mensagem: o entregador deixou na casa do vizinho
```

Saída: *"Categoria: endereco_errado"* — certo, mas com um prefixo que o seu `json.loads` ou a sua comparação de string não esperavam.

**Tentativa 3** — acrescenta o **formato** e um **exemplo negativo**:

```text
Classifique em: entrega_atrasada, endereco_errado, produto_avariado,
duvida, elogio.

Responda apenas o rótulo, em minúsculas. Não escreva explicação,
prefixo nem pontuação.

Mensagem: o entregador deixou na casa do vizinho
```

Saída: `endereco_errado`

**A ordem importa e não é acidente.** Repare no que cada correção acrescentou: primeiro o **domínio** (quais respostas existem), depois o **formato** (como responder), e o exemplo negativo só para fechar as saídas que o modelo insistia em produzir. É o output contract da §4 sendo construído peça por peça — e é assim que se depura prompt: **uma correção por vez, olhando o que sobrou de errado.**

> E note o que **não** foi preciso: nenhum exemplo, nenhum "pense passo a passo", nenhuma técnica. A maioria dos prompts ruins melhora com contrato de saída, não com técnica.

---

## Exercícios resolvidos

### 1. Qual técnica para cada tarefa?

**Enunciado.** Escolha a técnica para cada caso e justifique:

**(a)** Classificar 500 mil mensagens/dia em 5 categorias, com um modelo instruct pequeno.
**(b)** Decidir a melhor combinação de fretes entre 4 transportadoras, com prazo, preço e multa por atraso.
**(c)** Extrair 8 campos de um contrato em PDF, sempre no mesmo formato.
**(d)** Responder dúvidas sobre a política de devolução da empresa.

**Resolução.**

**(a) Zero-shot, com output contract e `enum`.** Classificação em 5 rótulos é tarefa de **um passo**: CoT não teria etapas para escrever e só daria ao modelo espaço para se contradizer. O formato — que é o que costuma falhar aqui — se resolve com `enum` no schema, não com exemplos. Se a acurácia não bastar, o próximo passo é **few-shot com 3 exemplos cobrindo as classes raras** (cobertura, não quantidade — §6.2), medindo se melhora; e só depois, trocar de modelo.

**(b) CoT — e possivelmente self-consistency.** Tarefa de várias etapas com comparação de alternativas (é a estrutura do problema das canetas, Aula 02, nota 01 §6.1): exige resultados intermediários, e sem eles o modelo tende a acertar o raciocínio geral e errar o fecho. Se ao rodar várias vezes ele der respostas **diferentes**, é exatamente o caso de self-consistency com N=3 a 5. **Alternativa melhor:** um modelo de raciocínio — e aí **sem** CoT no prompt, que seria redundante. Melhor ainda, e é a resposta de engenharia: **calcule os fretes em Python** e use o LLM só para explicar a escolha. Aritmética determinística não é trabalho para LLM.

**(c) Zero-shot + JSON Schema, e prompt chaining se o PDF for longo.** Formato é resolvido por schema, não por exemplos (Aula 02, nota 02 §7). Few-shot pode ajudar em campos com convenção específica ("data sempre em ISO"), mas um exemplo bem feito no contrato de saída costuma bastar. Se o contrato for longo, a cadeia é: **localizar o trecho relevante → extrair** — que é RAG começando a aparecer.

**(d) Nenhuma das técnicas desta nota — é RAG.** A resposta depende de um documento **da empresa**, que o modelo não viu no treino. Nenhum prompt faz o modelo saber a política de devolução de vocês: ou você coloca o texto no contexto, ou ele **inventa com fluência**. É o caso mais importante de reconhecer: *"prompt melhor" não é a solução; buscar a informação certa é.*

### 2. Lendo o resultado do Min et al.

**Enunciado.** Você roda um classificador few-shot com 4 exemplos e obtém 85% de acurácia. Embaralha os rótulos dos exemplos (deixando-os errados) e obtém 82%. Um colega conclui: *"os exemplos não servem para nada, vamos remover"*. Ele removeu e a acurácia caiu para 61%. Explique.

**Resolução.**

O colega interpretou o resultado ao contrário. Os três números contam a história completa:

| Configuração | Acurácia | O que o modelo tinha |
|---|---|---|
| Few-shot correto | 85% | espaço de rótulos + distribuição + formato + mapeamento |
| Few-shot **embaralhado** | 82% | espaço de rótulos + distribuição + formato |
| **Sem exemplos** | 61% | nada disso |

A queda de 85% para 82% mede a contribuição do **mapeamento correto** — pequena. A queda de 82% para 61% mede a contribuição de **tudo o mais** — enorme. É exatamente o achado do Min et al.: *o espaço de rótulos e a distribuição da entrada são importantes, independentemente de os rótulos individuais estarem corretos.*

**O que fazer com isso, na prática:**

- **manter os exemplos** — eles valem 21 pontos;
- **corrigir os rótulos assim mesmo**: 3 pontos são 3 pontos, e custam nada;
- **investir o esforço onde ele rende**: cobrir todos os rótulos (principalmente os raros), usar entradas parecidas com as reais e manter o formato idêntico — em vez de caçar os exemplos "mais representativos";
- **desconfiar da conclusão fácil.** "Os exemplos não servem para nada" era uma leitura plausível de dois números; bastou um terceiro para derrubá-la. É o mesmo hábito da Aula 02: **meça a alternativa antes de decidir.**

---

## Síntese

- *"Seja mais específico"* não é conselho útil. O útil é: específico sobre **tarefa, formato e casos de fronteira**. **Cada frase do prompt deve eliminar uma possibilidade** — se não elimina, é enfeite que você paga sempre.
- Três camadas com ciclos de vida diferentes: **system** (estável, versionado), **role** (tom e domínio — a mais superestimada), **contextual** (muda a cada chamada, e é a que cresce).
- **Delimitadores** ajudam o modelo, mas **não garantem nada**: para o modelo, instrução e dado são a mesma sequência de tokens. É por isso que prompt não é mecanismo de segurança — assunto da aula de segurança.
- O **output contract** tem quatro partes: formato, domínio, regras de fronteira e **exemplos negativos** — a parte que quase ninguém escreve.
- **Zero-shot funciona por instruction tuning** (FLAN) + RLHF, não por natureza. Comece sempre por ele.
- **Few-shot ensina o espaço de rótulos, a distribuição e o formato — não o mapeamento** (Min et al.). Rótulos embaralhados degradam pouco; **sem exemplos** degrada muito. Cubra as classes raras, use entradas realistas, mantenha o formato idêntico.
- Few-shot **mostra a forma da resposta, não o caminho** — por isso falha em tarefas de várias etapas.
- **CoT** externaliza a memória de trabalho: escrever os passos coloca no contexto os resultados parciais. **Zero-shot CoT** ("vamos pensar passo a passo") captura boa parte do ganho por uma linha.
- **Self-consistency** vota entre N raciocínios: caminhos errados divergem, o certo converge. Custa **N×** a saída.
- **Generated knowledge não resolve alucinação** — o conhecimento sai do próprio modelo. Fato externo pede **RAG**.
- **Prompt chaining** ganha em **controle e observabilidade**, não em qualidade: valida entre etapas e permite parâmetros por etapa.
- **ToT** é quase um agente; **meta-prompting** troca exemplos por estrutura, ao custo de assumir que o modelo já conhece a tarefa.
- **A tabela de decisão se lê de cima para baixo, parando no primeiro que resolve** — ela vai do prompt mais simples de escrever para o mais trabalhoso.
- **Depurar prompt é uma correção por vez:** domínio, depois formato, depois exemplo negativo. A maioria dos prompts ruins melhora com **contrato de saída**, não com técnica.
- **A técnica certa depende do modelo, não só da tarefa.** CoT em modelo de raciocínio é pagar duas vezes — e trocar de modelo pode invalidar o seu prompt.

---

## Fontes e leituras

**Papers** — todos disponíveis em `material_auxiliar/prompt_engineering/papers/`

- Wei, J. et al. — *Finetuned Language Models Are Zero-Shot Learners* (FLAN, 2022). [arxiv.org/abs/2109.01652](https://arxiv.org/abs/2109.01652) — por que o zero-shot funciona.
- Brown, T. et al. — *Language Models are Few-Shot Learners* (GPT-3, 2020). [arxiv.org/abs/2005.14165](https://arxiv.org/abs/2005.14165) — o *in-context learning*.
- **Min, S. et al. — *Rethinking the Role of Demonstrations* (2022). [arxiv.org/abs/2202.12837](https://arxiv.org/abs/2202.12837)** — o resultado dos rótulos embaralhados. **Leitura obrigatória desta aula.**
- Wei, J. et al. — *Chain-of-Thought Prompting Elicits Reasoning* (2022). [arxiv.org/abs/2201.11903](https://arxiv.org/abs/2201.11903)
- Kojima, T. et al. — *Large Language Models are Zero-Shot Reasoners* (2022). [arxiv.org/abs/2205.11916](https://arxiv.org/abs/2205.11916) — o "vamos pensar passo a passo".
- Wang, X. et al. — *Self-Consistency Improves Chain of Thought Reasoning* (2022). [arxiv.org/abs/2203.11171](https://arxiv.org/abs/2203.11171)
- Liu, J. et al. — *Generated Knowledge Prompting* (2022). [arxiv.org/abs/2110.08387](https://arxiv.org/abs/2110.08387)
- Yao, S. et al. — *Tree of Thoughts* (2023). [arxiv.org/abs/2305.10601](https://arxiv.org/abs/2305.10601)
- Zhang, Y. et al. — *Meta Prompting for AI Systems* (2024). [arxiv.org/abs/2311.11482](https://arxiv.org/abs/2311.11482)

**Guia**

- Prompt Engineering Guide, em português: [promptingguide.ai/pt](https://www.promptingguide.ai/pt) — a base das notas em `material_auxiliar/prompt_engineering/`.

**Nesta disciplina**

- [Aula 02 — nota 02](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/02-configuracoes-da-chamada.md) — saída estruturada (§7), o parâmetro `n` (§4.5), receitas por tarefa (§9).
- [Aula 02 — nota 01, §6](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/01-escolha-de-modelos.md) — instruct × raciocínio, que sustenta a §14 desta nota.
- [Aula 02 — nota 03](../../aula-02-escolha-e-configuracao-de-modelos/notas-de-aula/03-controle-da-saida.md) — `max_tokens` e `stop`: por que uma resposta com raciocínio precisa de teto maior.
- [Nota 02 desta aula](02-context-engineering.md) — o que entra na janela, e por quê.
- [Nota 03 desta aula](03-prompt-como-codigo.md) — versionar e testar o que você acabou de escrever.
- [Nota 04 desta aula](04-tool-calling.md) — quando o prompt vira schema.
