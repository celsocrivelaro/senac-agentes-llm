# Trabalho da disciplina — Parte 1: Escolher e provar o terreno

> Leiam antes a **[visão geral do trabalho](00-visao-geral.md)**: esta é a primeira de três entregas sobre o **mesmo** sistema.

## Contexto

Vocês já sabem chamar um modelo, ajustar parâmetros, escrever prompt com contrato de saída, declarar ferramentas e montar o laço de um agente com orçamento e salvaguardas. E acabaram de ver o que o mercado fez com isso — o que funcionou, o que quebrou e por quê.

Agora vocês escolhem **o seu problema**.

Esta entrega tem um objetivo que não é o óbvio. Ela não existe para vocês mostrarem um agente impressionante — existe para vocês **descobrirem cedo se o tema é viável**, enquanto ainda dá tempo de ajustar. Um tema que se revela impossível na Parte 3 é um semestre perdido; na Parte 1, é uma tarde de conversa.

Por isso o agente desta parte é **deliberadamente simples**. O que se avalia aqui é a **qualidade da escolha** e a **honestidade da análise**, não o tamanho do código.

---

## O que entregar

Quatro coisas:

0. **O grupo** — até 4 alunos
1. **O tema** escolhido
2. **O detalhamento do tema**: o contexto, **os usuários e a interação**, **o workflow** do agente e **a justificativa de negócio** — por que agente, e que ganho se espera
3. **A análise de modelos** que justifica a escolha do modelo
4. **Um agente simples rodando**, com prompt engineering e arquitetura básica — em **Python**, com a biblioteca **`openai`**, com **instruções de uso** e com a pesquisa em **`docs/`**

---

## 0. O grupo

**Até 4 alunos.** Grupos menores são permitidos, com a mesma exigência de profundidade.

Criem o repositório Git do grupo e coloquem os nomes no `README.md`. Todos os integrantes devem commitar — o histórico é evidência de participação, e ele será olhado.

---

## 1 e 2. O tema e o contexto

Este é o item mais importante da entrega, e o que mais decide o semestre de vocês.

Entreguem um documento chamado `docs/case.md` respondendo aos campos abaixo. Ele é a evolução da ficha de case preenchida em aula — se vocês já a preencheram, comecem dela.

### 2.1 O problema

**O problema em uma frase.** Se não couber em uma frase, o tema está grande demais. O tamanho da frase é o teste.

**Quem sofre com ele hoje.** Um cargo, uma pessoa concreta — "o analista de suporte do plantão noturno", e não "as empresas" ou "os usuários". Um tema sem usuário identificável não tem como ter critério de sucesso.

**O contexto de onde o agente vai ser usado.** Esta é a parte que os grupos costumam escrever mal, então seja específico:

- **onde** o sistema roda: dentro de que processo, em que momento, acionado por quem;
- **o que existe antes e depois dele**: de onde vem a entrada, para onde vai a saída, quem consome o resultado;
- **o que acontece hoje sem ele**: como a pessoa resolve isso atualmente, e quanto tempo leva;
- **quais são as regras do domínio**: as políticas, prazos, limites e exceções que o sistema precisa respeitar;
- **o que dá errado hoje**: os casos difíceis, as divergências, as exceções que quebram o processo.

O último item é o mais valioso. **Os casos difíceis do domínio são o que vai definir o seu sistema** — e são o que vocês vão usar para testá-lo.

### 2.2 Os usuários, e como o agente conversa com eles

Um sistema tem mais de um usuário, e eles querem coisas diferentes. Definir isso agora evita a maior causa de retrabalho no meio do semestre: descobrir na Parte 2 que o sistema foi desenhado para a pessoa errada.

#### Quem são

Listem **todos** os perfis que tocam o sistema — normalmente entre dois e quatro. Para cada um:

| Perfil | O que ele quer | O que ele sabe | O que ele **pode** fazer |
|---|---|---|---|
| *ex: analista de plantão* | achar os urgentes em 200 chamados | conhece o domínio, não conhece o sistema | aprova, reprova, devolve |
| *ex: solicitante* | resolver o problema dele | não conhece o processo | descreve, responde pergunta |
| *ex: gestor* | saber o que o sistema decidiu | nenhum dos dois | lê relatório |

A última coluna é a que mais importa, e a que os grupos esquecem: **quem pode aprovar uma ação irreversível?** Se ninguém puder, o sistema não deveria ter ação irreversível.

E identifiquem o **usuário principal** — aquele para quem o sistema é desenhado quando os interesses conflitam.

#### Como é a interação

Para o usuário principal, descrevam concretamente:

- **por onde** — chat, formulário, terminal, e-mail, planilha, API. E por que esse canal;
- **quem começa** — o usuário procura o sistema, ou o sistema procura o usuário? (Se for o segundo, é um agente **proativo**, e alguém precisa decidir *quando* ele acorda);
- **quantas trocas**, em média, até resolver — e o que acontece na primeira;
- **o que o sistema devolve** — texto? decisão? documento? um número? E em que formato;
- **como termina** — o que o usuário vê quando dá certo, e quando o sistema não consegue resolver.

O jeito mais rápido de fazer isso é **escrever um diálogo de exemplo**, com as falas reais dos dois lados, do início ao fim. Vale mais que três parágrafos de descrição, e vocês vão reaproveitá-lo como caso de teste.

#### E a complexidade que a disciplina pede

A disciplina recomenda um caso com **complexidade real de interação**. Respondam:

- o que o usuário **não** informa de primeira, e que o sistema precisa descobrir;
- o que acontece quando o que o usuário diz **contradiz** o que o sistema encontra;
- como o sistema decide que já sabe o suficiente para agir;
- **quando o sistema para e chama um humano** — e qual dos perfis acima ele chama.

Se as respostas forem "o usuário informa tudo no formulário", reconsiderem o tema.

### 2.3 O workflow do agente

Antes de qualquer código, desenhem o **fluxo simples** do sistema: a sequência de passos do pedido até o resultado.

Não é a arquitetura ainda — é o **processo**. Cinco a oito passos, em texto ou em diagrama ASCII, respondendo em cada um: *o que acontece aqui, e quem decide?*

```
1. ENTRADA      o solicitante descreve o problema no chat
2. COLETA       o sistema pergunta o que falta       [decide: MODELO]
3. CONSULTA     busca o registro no sistema interno  [decide: CÓDIGO]
4. TRIAGEM      caso simples -> regra                [decide: CÓDIGO]
                caso ambíguo -> segue para 5
5. ANÁLISE      compara o relato com o registro      [decide: MODELO]
6. AÇÃO         registra o parecer                   [ESCRITA - confirma]
7. RETORNO      informa o solicitante e o analista
```

Duas regras para esse desenho, e as duas valem ponto:

**Marquem quem decide em cada passo** — o seu código ou o modelo. Essa marcação é o que revela o nível de autonomia real do sistema, e ela normalmente surpreende: a maioria dos passos é código.

**Marquem os passos de escrita**, e se são reversíveis. Todo passo marcado como escrita irreversível precisa de confirmação humana, e o perfil que confirma vem da §2.2.

> **O que este desenho vai provar ou reprovar:** se vocês conseguirem escrever os oito passos sem nenhum `[decide: MODELO]`, o sistema é um **workflow** — e o §2.5 vai perguntar por que ele precisava de um agente. Melhor descobrir isso agora, num desenho de dez minutos, do que na semana 12.

### 2.4 O sistema

**O que o sistema faz**, em 3 a 5 linhas.

**O nível de autonomia pretendido** — workflow, roteador ou agente — **e por que este e não o de baixo**. A segunda metade é o que vale: a regra da disciplina é *use a menor autonomia que resolve*, então vocês precisam explicar por que o nível anterior não resolvia.

**As ferramentas**, numa tabela:

| Ferramenta | O que faz | Leitura ou escrita? | Reversível? | Contra o que ela conversa |
|---|---|---|---|---|

A última coluna é requisito desta parte — ver o item 4.2.

### 2.5 A justificativa de negócio — a venda

Este é o campo novo, e é o que mais aproxima o trabalho da vida real. Vocês vão ter que **vender o sistema**.

Não é retórica: em qualquer emprego, a decisão de construir um agente é aprovada ou recusada por alguém que não vai ler o seu código. Essa pessoa quer três coisas, nesta ordem — **por que agente**, **quanto se ganha** e **como você vai saber**.

#### Por que um agente, e não software comum

A pergunta mais importante e a que mais reprova. Se o problema se resolve com um formulário, uma consulta SQL e três `if`, **a resposta certa é o formulário** — e um trabalho que constrói um agente para isso está resolvendo o problema errado, mais caro.

Responda em duas frases: o que exatamente na tarefa exige decisão em tempo de execução, e por que o nível de autonomia abaixo do seu não dava conta.

#### O ganho esperado, com número e com a conta à vista

Escolha **um ou dois eixos** de ganho — mais que isso é conversa fiada — e para cada um apresente:

1. **A linha de base**: quanto custa, demora ou erra hoje. **Meça, não estime.** Cronometre dez casos, conte os itens de uma fila real, pergunte a quem faz. Um baseline de 10 medições vale mais que uma opinião.
2. **O alvo**: onde você espera chegar, e por quê.
3. **A conta**, escrita: `de X para Y = Z% de ganho`, multiplicado pelo volume.
4. **A ressalva**: é uma **estimativa**, e você a declara como tal.

Os eixos, com a unidade de cada um (a nota 01 da aula de casos de uso tem a tabela completa e exemplos reais):

| Eixo | Unidade | Como soa uma boa afirmação |
|---|---|---|
| **Tempo por tarefa** | minutos por caso | *"triagem de 12 min para 2 min por chamado: −83%"* |
| **Velocidade de processo** | tempo do início ao fim | *"do pedido ao parecer, de 3 dias para 4 horas"* |
| **Produtividade** | itens por pessoa por dia | *"de 40 para 110 itens conferidos por analista/dia"* |
| **Cobertura** | % resolvido sem humano | *"60% dos chamados fechados sem atendente, com 100% dos urgentes ainda escalados"* |
| **Redução de carga** | fila que não chega à pessoa | *"−70% dos tickets de nível 1"* |
| **Erro e retrabalho** | taxa antes × depois | *"erro de classificação de 18% para 5%: −72% de retrabalho"* |
| **Custo por transação** | R$ por caso | *"de R$ 4,10 para R$ 0,45 por atendimento contido"* |
| **Receita** | conversão, ticket médio | *"+8% de conversão na recomendação"* |

**Duas dicas que separam uma venda boa de uma ruim.**

A primeira: **número específico e feio vence número redondo e bonito.** *"Reduz 83% do tempo de triagem, de 12 para 2 minutos, em 200 chamados/dia"* é crível porque tem denominador. *"Aumenta a eficiência em 50%"* não é crível porque não tem.

A segunda: **prometa o eixo que você consegue medir.** Se você não tem como medir erro, não prometa redução de erro — prometa tempo, que você cronometra. Promessa que não dá para conferir é a mesma coisa que não ter verificador, e reprova pelo mesmo motivo.

#### O ganho para o usuário, que não é o mesmo do negócio

Dois destinatários, dois ganhos, e eles às vezes **entram em conflito**:

- **para o negócio**: custo, capacidade, velocidade, risco;
- **para o usuário**: espera menor, menos repetição de informação, resposta na primeira tentativa, menos transferências, mais autonomia para resolver sozinho.

Diga os dois. E se houver tensão — automação que corta custo e piora a experiência do caso difícil —, **diga a tensão**. É o que aconteceu no caso Klarna, visto em aula: a métrica que a empresa acompanhava melhorava enquanto a que importava piorava.

#### O que não vale como justificativa

| Não vale | Por quê |
|---|---|
| *"moderniza o processo"* | não é um ganho, é um adjetivo |
| *"melhora a eficiência"* | qual eficiência, medida como? |
| *"reduz custos"*, sem número | custo de quê, quanto, sobre que volume? |
| *"usa IA de ponta"* | é o meio, não o fim |
| *"os concorrentes já usam"* | é medo, não é ganho |
| um número sem linha de base | *"−80%"* de quanto para quanto? |

#### E o outro lado da conta

Uma venda honesta declara o custo. Em três linhas: **quanto custa rodar** (a conta de tokens do item 3, por execução e por mês), **quanto custa construir** (o tempo de vocês) e **o que se perde** — o caso que o sistema vai errar, e quem paga por ele.

> **O arco do trabalho:** o que vocês prometerem aqui será **conferido na Parte 3**, quando o sistema estiver rodando e a gestão de custos entrar. Prometer 80% e entregar 30% com a conta à vista é um resultado aceitável e honesto. Prometer "mais eficiência" e não ter como conferir, não.

### 2.6 O verificador

> **Como vocês vão saber que a saída está certa?**

**Este campo reprova mais que todos os outros juntos.** Respostas que valem:

- um **conjunto rotulado à mão** — 40 casos com a resposta que um humano daria;
- uma **regra de negócio** que confere o resultado;
- um **teste** que passa ou falha;
- a comparação com uma **decisão humana registrada**.

Resposta que não vale: *"dá para ver que está certo"*.

Verificador **construído** conta. O que não conta é não ter nenhum.

### 2.7 O critério de sucesso

**Um número, com denominador.** "Acerta em 80%" não diz nada; "acerta a categoria em 32 de 40 casos rotulados" diz.

Se o custo do erro for **assimétrico** — se errar para um lado for muito pior que para o outro —, digam isso e usem duas métricas. Exemplo: *"acerta a categoria em ≥32/40 **e** não deixa passar nenhum caso urgente"*.

### 2.8 Dados

**De onde vêm:** reais, públicos ou simulados.

Se simulados — e é o caso mais comum, e é legítimo —, expliquem **como vocês preservam a dificuldade do problema**. Dado de mentira fácil demais produz um sistema que só funciona no seu conjunto de teste. Nomeiem explicitamente:

- qual é o caso de **divergência** (o sistema diz uma coisa, o usuário diz outra);
- qual é o **registro inexistente** (o identificador que não existe);
- qual é o caso que **não deve** disparar a ação principal.

Os dados dos laboratórios da disciplina foram montados exatamente assim. Façam igual.

### 2.9 Dado sensível

Que dado sensível este tema toca — pessoal, financeiro, de saúde, sigiloso? Se não toca nenhum, digam isso; é uma vantagem do tema, não uma omissão.

Se toca, a regra é: **dado sensível não entra no repositório nem no contexto do modelo.** Ele é simulado.

### 2.10 Espaço para o que ainda vem

Marquem e **escrevam o quê**:

- [ ] **RAG** (Parte 2) — que conhecimento de domínio os agentes vão consultar, e em que formato ele existe hoje?
- [ ] **MCP** (Parte 2) — qual integração vai virar servidor MCP?
- [ ] **LangChain** (Parte 2) — que parte da orquestração?
- [ ] **Multiagente** (Parte 3) — quais seriam os agentes, e por que mais de um?

Um `[x]` sem texto não conta. "RAG: o regulamento interno, 40 páginas em PDF" conta.

### 2.11 O maior risco

O risco de verdade, não o de fachada. *"Pode ser que o modelo erre"* não é risco — é a premissa. *"A nossa única fonte é um PDF escaneado e o OCR pode não funcionar"* é risco, e vem com plano B.

---

## 3. A análise de modelos

Um documento `docs/modelos.md`. O objetivo é **justificar uma escolha**, não catalogar o mercado.

### 3.1 Os candidatos

Escolham **três** modelos candidatos e compare-os nos eixos que importam **para o seu caso** — e diga por que esses eixos, e não outros:

| Eixo | Por que pode importar no seu caso |
|---|---|
| janela de contexto | o seu caso tem documento longo? histórico grande? |
| suporte a **tool calling** e a **saída estruturada** | é pré-requisito: sem isso não há agente |
| suporte a multimídia | só se o caso realmente pedir |
| capacidade de raciocínio | tarefa com passos de inferência, ou classificação direta? |
| custo por milhão de tokens (entrada e saída) | multiplicado pelo número de chamadas por execução |
| latência | há alguém esperando na frente da tela? |
| onde roda | API pública, nuvem contratada, local |
| política de dados | quem pode ver o que vocês enviam |

**Podem escolher um modelo diferente do Mistral usado nos laboratórios.** Se escolherem, esta seção é onde vocês defendem a decisão — e onde vocês assumem o custo dela.

### 3.2 A conta

Estimem o custo de **uma execução** do seu sistema:

```
tokens de entrada por chamada  × nº de chamadas por execução × preço de entrada
+ tokens de saída por chamada  × nº de chamadas por execução × preço de saída
= custo por execução
```

E depois: custo por 100 execuções, e custo estimado do semestre. Um sistema com 6 chamadas por execução custa seis vezes o que vocês imaginaram olhando o preço por token.

### 3.3 A verificação mínima

**Não confiem em leaderboard.** Rodem uma verificação própria, mesmo pequena: **cinco casos do seu domínio** nos três candidatos, com o mesmo prompt, e mostrem o resultado numa tabela.

Cinco casos não são uma medição estatística — e não é isso que se pede. O que se pede é que vocês tenham **olhado a saída dos três modelos no seu problema** antes de escolher. É o embrião do benchmark próprio que a Parte 3 vai cobrar de verdade.

### 3.4 A decisão

Qual modelo, e **em que condições vocês mudariam de ideia**. A segunda parte é o que separa uma escolha de um chute.

---

## 4. O agente simples

Aqui vocês escrevem código. **Pouco código.**

O objetivo é provar que o caminho existe: que as ferramentas são implementáveis, que o modelo escolhido dá conta, e que o problema responde ao tratamento. Não é para estar bonito nem completo.

### 4.1 A pilha, e o mínimo exigido

**Python**, com a biblioteca **`openai`**. É a mesma pilha dos laboratórios: a biblioteca é da OpenAI, e a `base_url` aponta para o provedor que vocês escolheram no item 3 — Mistral, outro serviço compatível, ou um modelo local. Se vocês escolheram um modelo diferente do usado em aula, o que muda são duas variáveis de ambiente, não o código.

```python
from openai import OpenAI

client = OpenAI(
    base_url=os.environ["LLM_BASE_URL"],   # o provedor que vocês escolheram
    api_key=os.environ["OPENAI_API_KEY"],
)
```

A chave **nunca** no repositório: `.env` fora do git, `.env.example` dentro, com os nomes das variáveis e nenhum valor. E um `requirements.txt` com as versões fixadas.

| Requisito | Detalhe |
|---|---|
| **2 a 4 ferramentas** | mais que isso é escopo da Parte 2 |
| **Ao menos uma ferramenta de escrita** | com confirmação, se a ação for irreversível |
| **Laço com estado explícito** | não guardem tudo na lista de mensagens |
| **Orçamento** | teto de passos, tokens e — se houver custo — de dinheiro |
| **Terminação registrada** | o programa diz **por que** parou |
| **Log da trajetória** | ferramenta, argumentos, resultado e erro de cada passo |
| **Prompts em arquivo** | em `prompts/`, versionados, com `prompt × modelo × parâmetros` carimbado |
| **Erro de ferramenta como dado** | retorno de erro que **ensina** o modelo a se corrigir, não exceção que derruba |

### 4.2 A integração com software tradicional

**Requisito desta parte:** pelo menos uma ferramenta do agente precisa conversar com **software tradicional** — não com o modelo, e não com um dicionário Python no meio do arquivo.

Vale qualquer um destes:

- uma **API HTTP** — sua, de terceiro, ou um servidor local que vocês subam;
- um **banco de dados** — SQLite serve, Postgres serve;
- um **serviço** próprio, mesmo trivial, rodando em outro processo;
- um **arquivo estruturado** lido por uma camada de acesso separada (o mais fraco dos quatro, aceito se a camada existir de verdade).

**Operação mock é permitida e esperada.** O ponto não é ter o sistema real — é o agente atravessar a fronteira do próprio processo e lidar com o que isso traz: erro de rede, dado ausente, formato inesperado, latência.

Na Parte 2 esta integração é **reescrita como servidor MCP**. Escolham algo que faça sentido reescrever.

### 4.3 O prompt engineering

Ainda não é o plano completo da Parte 2, mas já não é improviso. Para cada etapa em que o modelo é chamado, digam **no código, em comentário**:

- qual **técnica** está sendo usada (zero-shot, few-shot, CoT…) e **por que ela**;
- qual é o **contrato de saída** — formato, e o que é proibido;
- o que aquela etapa **impede** de dar errado.

E a regra que vale desde a aula de prompt engineering: **se vocês apagarem uma frase do prompt e não souberem dizer o que ela impedia, ela não estava fazendo nada.**

### 4.4 A arquitetura básica

Um diagrama — ASCII serve, e é preferido — mostrando: a entrada, as etapas, onde o modelo decide, onde o seu código decide, as ferramentas e as condições de parada.

E um parágrafo respondendo: **onde neste desenho está a decisão que justifica um agente em vez de um workflow?** Se não houver nenhuma, é um workflow — o que é aceitável na Parte 1, desde que vocês digam onde a decisão vai entrar na Parte 2.

### 4.5 A demonstração

O programa roda contra **pelo menos 4 casos** do seu domínio, escolhidos para serem diferentes:

1. o caso simples, que deve funcionar;
2. o caso de **divergência** (o sistema diz uma coisa, o usuário diz outra);
3. o caso do **registro inexistente** (erro de ferramenta que o modelo tem de contornar);
4. o caso que **não** deve disparar a ação principal.

O log das quatro execuções vai no repositório.

---

### 4.6 As instruções de uso

O `README.md` precisa responder **duas** perguntas, e a segunda é a que costuma faltar:

**Como rodar** — clonar, instalar, configurar o `.env`, o comando que sobe o sistema. Testado do zero, por alguém que nunca viu o projeto, em menos de cinco minutos.

**Como usar** — e aqui é onde se separa um projeto entregue de um projeto que só compila:

- **o que a pessoa digita** e em que formato;
- **o que o sistema faz com aquilo**, em duas ou três frases;
- **que saída ela recebe**, e como interpretá-la;
- **um exemplo completo**, com entrada e saída **reais**, copiadas de uma execução — não inventadas;
- **o que o sistema não faz**, e o que acontece quando ele não sabe responder.

> Um sistema que roda e ninguém sabe usar não está entregue.

---

## Como esta parte será avaliada

Em ordem de peso:

| O que se avalia | O que se espera |
|---|---|
| **A qualidade da escolha do tema** | problema em uma frase · usuário concreto · **verificador que existe** · critério de sucesso com denominador · espaço declarado para o que vem |
| **Usuários e interação** | os perfis estão listados com o que **podem** fazer · há um usuário principal · a interação tem canal, iniciativa e formato · **há um diálogo de exemplo** · está dito quando o sistema chama um humano |
| **O workflow** | 5 a 8 passos · cada passo diz **quem decide** (código ou modelo) · os passos de escrita estão marcados, com reversibilidade |
| **A justificativa de negócio** | responde **por que agente e não software comum** · a linha de base foi **medida**, não estimada · a conta está à vista, com denominador e volume · o ganho do usuário está separado do ganho do negócio · o custo de rodar e o que se perde estão declarados |
| **A honestidade da análise de modelos** | três candidatos comparados nos eixos **do caso** · a conta de custo feita · os cinco casos rodados de verdade · a condição de mudar de ideia |
| **O agente rodando** | Python + `openai` · atende os requisitos de 4.1 · a integração de 4.2 é real · os 4 casos executados, com log |
| **A entrega como projeto** | roda do zero em <5 min · o `README` diz **como usar**, com exemplo real · a pesquisa está em `docs/`, em Markdown · nenhuma chave no repositório |
| **A justificativa da arquitetura** | o nível de autonomia é o menor que resolve, **e vocês explicam por que o de baixo não servia** |
| **O rigor do prompt** | prompts em arquivo, versionados, com técnica e contrato declarados |

E o que **não** conta: quantidade de código, quantidade de ferramentas, sofisticação visual. Nada disso é o assunto desta entrega.

---

## Entrega

No repositório do grupo:

```
README.md          nomes do grupo · o problema em uma frase
                   COMO RODAR e COMO USAR (item 4.6)
requirements.txt   dependências com versão fixada
.env.example       os nomes das variáveis, sem nenhum valor
docs/              TODA a pesquisa e documentação do case
  case.md          os itens 1 e 2, incluindo a justificativa de negócio
  modelos.md       o item 3
  fontes.md        tudo que foi consultado, com link
  autopsia.md      a autópsia do caso real, se vocês a fizeram
prompts/           os prompts, versionados
src/               o agente
logs/              as 4 execuções demonstradas
dados/             os dados simulados (com os casos difíceis nomeados)
```

**Todo documento gerado sobre o case vive em `docs/`**, em Markdown e versionado. Nas Partes 2 e 3 essa pasta cresce — plano de prompt engineering, arquitetura, decisões, resultados — e o histórico dela mostra **quando o grupo mudou de ideia sobre o próprio case, e por quê**. É o mesmo princípio do versionamento de prompt cobrado desde a Aula 03, aplicado ao raciocínio.

Nada de `.docx` nem `.pdf`: Markdown, para que o `git diff` funcione.

---

## Dicas

- **Meçam a linha de base antes de escrever qualquer prompt.** Cronometrem dez casos, contem uma fila real. Sem esse número, a venda da §2.5 é opinião — e é o item que mais some das entregas.
- **Comecem pelo verificador.** Se vocês souberem como medir que a saída está certa, o resto do tema se organiza sozinho. Se não souberem, nenhum outro campo salva o trabalho.
- **Escrevam os dados difíceis antes do agente.** Os quatro casos de 4.5 são o seu conjunto de teste, e escrevê-los primeiro força vocês a entender o domínio antes de escrever prompt.
- **Não escolham o tema mais impressionante. Escolham o que vocês conseguem medir.** O impressionante que ninguém consegue avaliar vira, na apresentação final, uma demonstração que funciona uma vez.
- **Leiam os quatro anti-padrões da [visão geral](00-visao-geral.md) procurando o seu tema neles** — e não procurando motivo para ele não estar lá.
- **Se o tema é o do TCC**, aproveitem: descrevam o contexto do item 2 com a profundidade que vocês já têm. É a parte em que vocês largam na frente.
- E lembrem da calibragem vista em aula: o melhor agente medido em tarefas profissionais longas concluiu **cerca de um terço** delas sozinho. **Um tema que só funciona se o agente acertar quase sempre, sem ajuda, não é viável.**
